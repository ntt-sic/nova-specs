..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add tag field to TaskAPI
==========================================

https://blueprints.launchpad.net/nova/+spec/tag-of-task-api

This is a new feature of TaskAPI.
With this specification, user can add tag to Nova API when user requests.
Tag is string or UUID which user can specify.

User can know task status of request API, even if user couldn't
get response from nova, on condition that user requested with 'tag'.

Problem description
===================

For end User, it is too hard to get task status without TaskAPI.
Even if TasAPI is implemented in Nova, it is still difficult if user could
not get response and instance-id or task-id from Nova.
This may happen in network disconnecting or client hang up.

For deployer, it is possible to specify of missing request and its resource.

Proposed change
===============

To solve these problems, it is effective to use tag with TaskAPI.
User request tag with Nova request API.
Even if user could not get task objects with any trouble, user can
know request API status as below.
User use TaskAPI with tag and get tag-id and get requested API status.

This tag feature is enable with any TaskAPI.
User can set this tag as request header.

 #. First, user request Nova API with tag header.
 #. Something wrong happen such as down of client side or
    network disconnection.
 #. User could not get task-id.
 #. So, user ask own request task status with TaskAPI using tag just now.
 #. Like http://<..>/v3/servers/<tag-id>/tasks.
 #. So user could get Task status directly.

Alternatives
------------

In the first place, I want to hear any ideas that workarounds of these trouble.
I say the case user couldn't get response from OpenStack.
Is it responsible for OpenStack or client and deployer?

In the second place, as before, I filed idempotentcy-client-token[1],
probably it is not appropriate to use word 'idempotent'.
But I just wanted to implement feature of API tracking.

In last, it is kindness if Nova itself has safety feature
such as TaskAPI and this tag.

[1] https://blueprints.launchpad.net/nova/+spec/idempotentcy-client-token

Data model impact
-----------------

About data, just add 'tag' column to TaskAPI table.
If user doesn't specify any tag, NULL remains as default value.


REST API impact
---------------

First, all of this tag implementaion is based on TaskAPI.
So, here, I write just diff from TaskAPI.
I have to implement this added diff feature.

* Specification for the method

  * Add this header "-H "Task-tag: 49fe58f1-a658-4232-b723-8f0b3efc1014".
    Using this header is limited to API which is enable TaskAPI.

  * As same as TaskAPI, tag effects creating an instance or acting
    on an instance.

  * Basically, normal and error http response code(s) will not change.

  * Qyery for TaskAPI using tag
    http://<..>/v3/servers/<tag-id>/tasks

    This returns result same as http://<..>/v3/servers/<server-id>/tasks
    from TaskAPI.
    And also response returns tag as header.

    response header = {
        'date': 'Thu, 22 May 2014 01:36:36 GMT',
        'content-length': '15',
        'content-type': 'application/json',
        'x-compute-request-id': 'req-3e26271a-c819-44af-bc37-a27863ffba5a',
        'task-tag': '49fe58f1-a658-4232-b723-8f0b3efc1014'
    }

    task = {
        'type': 'object',
        'properties': {
            'uuid': { 'type': 'string', 'format': 'uuid' },
            'task': { 'type': 'string' },
            'state': { 'type': 'string' },
            'resource': {
                'type': 'object',
                'patternProperties': {
                    'server|image|network|volume': {
                        'type': 'string',
                        'format': 'uuid'
                    }
                }
            },
            'request_id': { 'type': 'string' },
            'user_id': { 'type': 'string' },
            'project_id': { 'type': 'string' },
            'start_time': { 'type': 'string', 'format': 'date-time' },
            'last_updated': { 'type': 'string', 'format': 'date-time' }
        }
        'additionalProperties': True
    }


Security impact
---------------

Scope of tag fully depends on TaskAPI.

* If each user who belong to several tenants specify same tags,
  TaskAPI return only each tenant's information.
  People can see own tenant's task infoemation.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

If end user uses this feature, user set any string to tag.
Second request of same tag, same request URL and same parameter,
in this case, TaskAPI returns 'That request is already accepted'.

So, if you combine tag with any client such as Heat, it is successful that
this client can generate unique tag everytime.

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

 * Add tag header to Nova request.

 * Add tag header to Nova response.

 * Add tag field to TaskAPI table.

 * Add DBAPI which can find task from tag.

 * Add API which can find task from tag.


Dependencies
============

instance-tasks-api
https://blueprints.launchpad.net/nova/+spec/instance-tasks-api


Testing
=======

I have no idea how to pass tag from client in tempest.
And how to ensure that Nova returns(or doesn't return) Task objects.


Documentation Impact
====================

None


References
==========

https://etherpad.openstack.org/p/juno-nova-v3-api
