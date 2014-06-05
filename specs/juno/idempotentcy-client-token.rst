..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Idempotency for OpenStack API
==========================================

https://blueprints.launchpad.net/nova/+spec/idempotentcy-client-token

In this blueprint, I suggest ClientToken that user can specify, and post
requests which idempotency is guaranteed by this ClientToken.


Problem description
===================

* ClientToken is implemented in the run_instance of AmazonEC2.
  It is also implemented in Openstack EC2 run_instance.

  - https://bugs.launchpad.net/nova/+bug/1188327

  - https://review.openstack.org/#/c/32060/

* There is no reason to ClientToken is limited to only the run_instance.
  I suggest implementation of ClientToken to other Openstack API.

* Use case for End User
#. User requested that boot own server to OpenStack.
#. Unfortunately, the client has gone down.
#. There is no way to know how was his server for user.
#. User thought booting was fauilure(even if it went well),
   so user booted a new server with same param as 1.

* It looks no problems at first glance, but money problems may occur
  with using OpenStack as a service.
  (There is a possibility that the overage charges for two servers.)

* So user need to way how is his server, how is his server status.
  In this case, idempotency client token is so useful.
  To specify the token itself by user, user can know status of server.
  Even if how many times user requests POST method, it is guaranteed that
  the state of the POST request which was same with return of
  user's first POST request.

* Now, I don't target this feature to Developer.

* For a major reworking of something existing it would describe the
  problems in that feature that are being addressed.


Proposed change
===============

In curl command, you add header of request 'X-Client-Token: foo'.
X-Client-Token should be uuid or any strings.
In this name, I already implemented python-novaclient implementation.

So, you can put request like below.
'nova --x-client-token boot --flavor 1 --image hoge ...'
If you have good name instead of 'X-Client-Token', please tell me.

After, if you retry same request parameter and same X-Client-Token,
now OpenStack returns GET response of instance.

You can correctly know instance's status and don't happen to boot
needless instance.

* Here, I target this to any POST API in Nova.
* In future, it is useful this specification is adopted to any POST API.

Alternatives
------------

* TaskAPI

  https://blueprints.launchpad.net/nova/+spec/instance-tasks-api

  First, I wanted to use TaskAPI.
  But it is needed to set instance_id as required parameter to
  use this TaskAPI.
  I imagene the case that client can't get instance_id, so I can't
  use this TaskAPI what I want to resolve.

* Instance tag

  https://blueprints.launchpad.net/nova/+spec/tag-instances

  This is add tag to instance not request.
  To prevent any needless request, it is needed to add tag for request.

Data model impact
-----------------

It is a decorator function.
I made PoC as decorator, so if you want to idempotent, you set this
decorator any APIs you want.

* Every data related to this blueprint stored in memcached as below.

  <client_token>: <[url, body, tenant_id]>

* Memcached data's life could be cahnged in 'memcache_expiration' configuration.

REST API impact
---------------

This idempotent feature decorator can be used with POST API.

* First POST request, it returns 202.

* Second POST request, with same 'X-Client-Token' and same
  request parameter, it also returns 202 but response content
  is same as GET response.

* Second POST request, with same 'X-Client-Token' and different
  request parameter, it returns 409 because that 'X-Client-Token' is
  already used.

* Now, second POST request returns GET result with schema for GET,
  in the near future, I want to return POST schema with
  GET response content.

See concrete example below of this link:

https://github.com/ntt-sic/nova/commit/c9bc157b122907d7bd7e98b364137b7ecd47bd0f


Security impact
---------------

This decorator judge by tenant_id scope.
So can specify same 'X-Client-Token' if tenant is differ.


Notifications impact
--------------------

None


Other end user impact
---------------------

If user don't specfiy this 'X-Client-Token', there is no effect to original
POST request.

In case of using python nova-client, need to modify it to
accept 'X-Client-Token'.

Performance Impact
------------------

None


Other deployer impact
---------------------

Now, this feature using memcached.

So you need to add your /etc/nova/nova.conf as below.

* memcached_servers=YourIP:11211

If memcached is not so appropriate, I re-implement with other way like DB.


Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  haruka tanizawa(h-tanizawa)

Other contributors:
  None

Work Items
----------

1. Now second request returns GET result.
   I want to return same schema as first POST request.
2. About flavor, decorated function and show function is in different
   class. So, I need to resolve this problem.
3. Consolidate decorator's resolver.
4. I implemented idempotent.py temporarily.
   Appropriate file path is need to be considered.
5. This feature is decorator. And I applied this decorator to 'Create Server'.
   In nova, I also applied it to create keypair.
   I am going to apply other nova POST method.


Dependencies
============

One of Heat blueprint depend on this blueprint.

* Support API retry function with Idempotency in creating/updating a stack
  https://blueprints.launchpad.net/heat/+spec/support-retry-with-idempotency


Testing
=======

Add decorator unittest.

Moreover I should tempest for testing multiple times request with same
parameter and same client token.

And combination of tenant, request URL, parameter, client token etc...


Documentation Impact
====================

There are some documentation impacts.

First, new request parameter is added.
User can use this if he wants.

Second, response of POST request is differ from how many times request.


References
==========

Mailing list discussions

- https://lists.launchpad.net/openstack/msg13082.html
- http://lists.openstack.org/pipermail/openstack-dev/2013-October/017691.html

Related specifications in EC2

- http://goo.gl/8gQX8X
- http://goo.gl/Awphn9
