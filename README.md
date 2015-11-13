IronMQ Node.js Client
-------------

The [full API documentation is here](http://dev.iron.io/mq/3/) and this client tries to stick to the API as
much as possible so if you see an option in the API docs, you can use it in the methods below.


## Getting Started

1\. Install the node module:

```
npm install iron_mq
```

2\. [Setup your Iron.io credentials](http://dev.iron.io/mq/reference/configuration/)

3\. Create an IronMQ Client object:

```javascript
var iron_mq = require('iron_mq');
var imq = new iron_mq.Client();
```

Or pass in credentials:

```javascript
var imq = new iron_mq.Client({token: "MY_TOKEN", project_id: "MY_PROJECT_ID", queue_name: "MY_QUEUE"});
```

If no `queue_name` is specified it defaults to `default`.

### Keystone Authentication

#### Via Configuration File

Add `keystone` section to your iron.json file:

```javascript
{
  "project_id": "57a7b7b35e8e331d45000001",
  "keystone": {
    "server": "http://your.keystone.host/v2.0/",
    "tenant": "some-group",
    "username": "name",
    "password": "password"
  }
}
```

#### In Code

```javascript
var keystone = {
    server: "http://your.keystone.host/v2.0/",
    tenant: "some-gorup",
    username: "name",
    password: "password"
}
var imq = new iron_mq.Client({project_id: "57a7b7b35e8e331d45000001", keystone: keystone});
```

## Usage

### Get Queues List

```javascript
imq.queues(options, function(error, body) {});
```

**Options:**

* `per_page` - number of elements in response. The default is 30, the maximum is 100.
* `previous` - this is the last queue on the previous page, it will start from the next one. If queue with specified 
               name doesn’t exist result will contain first per_page queues that lexicographically greater than previous
* `prefix` - an optional queue prefix to search on. e.g., `prefix=ca` could return queues `["cars", "cats", etc.]`

--

### Get a Queue Object

You can have as many queues as you want, each with their own unique set of messages.

```javascript
var queue = imq.queue("my_queue");
```

**Note:** if queue with desired name does not exist it returns fake queue.
Queue will be created automatically on post of first message or queue configuration update.

--

### Retrieve Queue Information

```javascript
queue.info(function(error, body) {});
```

--

### Post a Message on a Queue

Messages are placed on the queue in a FIFO arrangement.
If a queue does not exist, it will be created upon the first posting of a message.

```javascript
queue.post(messages, function(error, body) {});

// single message
queue.post("hello IronMQ!", function(error, body) {});
// with options
queue.post({body: "hello IronMQ", delay: 30}, function(error, body) {});
// or multiple messages
queue.post(["hello", "IronMQ"], function(error, body) {});
// messages with options
queue.post(
  [{body: "hello", delay: 35},
   {body: "IronMQ", delay: 30}],
  function(error, body) {}
);
```

**Required messages' parameters:**

* `body`: The message body as a string. This does not jsonify objects.

**Optional messages' parameters:**

* `delay`: The item will not be available on the queue until this many seconds have passed.
Default is 0 seconds. Maximum is 604,800 seconds (7 days).

--

### Reserve/Get Messages from a Queue

```javascript
queue.reserve(options, function(error, body) {});
```

**Options:**

* `n`: The maximum number of messages to get. Default is 1. Maximum is 100.

* `timeout`: After timeout (in seconds), item will be placed back onto queue.
You must delete the message from the queue to ensure it does not go back onto the queue.
If not set, value from POST is used. Default is 60 seconds. Minimum is 30 seconds.
Maximum is 86,400 seconds (24 hours).

In `reserve` function when `n` parameter is specified and greater than 1 method returns list of messages.
Otherwise, message's object will be returned. 

When you pop/reserve a message from the queue, it is no longer on the queue but it still exists within the system.
You have to explicitly delete the message or else it will go back onto the queue after the `timeout`.

--

### Get a Messages off a Queue by Message ID

```javascript
queue.msg_get(message_id, function(error, body) {});
```

--

### Peek Messages on a Queue

Peeking at a queue returns the next messages on the queue, but it does not reserve them.

```javascript
queue.peek(options, function(error, body) {});

queue.peek_n(options, function(error, body) {});
```

**Options:**

* `n`: The maximum number of messages to peek. Default is 1. Maximum is 100.

In `peek` function when `n` parameter is specified and greater than 1 method returns list of messages.
Otherwise, message's object will be returned. `peek_n` function returns `Array` of messages even if `n` option
is set to 1 or omitted.

--

### Touch a Message on a Queue

Touching a reserved message extends its timeout by the duration specified when the message was created, which is 60 seconds by default.

```javascript
var message_id = "xxxxxxx";
var reservation_id = "xxxxxxx";
queue.msg_touch(message_id, reservation_id, {timeout: 120}, function(error, body) {});
```

--

### Release Message

```javascript
queue.msg_release(message_id, reservation_id, {delay: 4600}, function(error, body) {});
```

**Options:**

* `delay`: The item will not be available on the queue until this many seconds have passed.
Default is 0 seconds. Maximum is 604,800 seconds (7 days).

--

### Delete a Message from a Queue

Be sure to delete a message from the queue when you're done with it.

```javascript
queue.del(message_id, {}, function(error, body) {});
```
--

To delete a reserved message `reservation_id` should be passed to the method.

```javascript
queue.del(message_id, {reservation_id: 'xxxxxxxxx'}, function(error, body) {});
```
--

Delete multiple messages from a Queue after reserving or posting

```javascript
queue.reserve({n:3}, function(error, messages) {
   queue.del_multiple({reservation_ids: messages}, function(error,body){});
});
queue.post(["hello", "IronMQ"], function(error, ids) {
   queue.del_multiple({ids: ids}, function(error,body){});
});
```

--

### Clear a Queue

```javascript
queue.clear(function(error, body) {});
```

--

### Delete a Message Queue

```javascript
queue.del_queue(function(error, body) {});
```

--

## Push Queues

IronMQ push queues allow you to setup a queue that will push to an endpoint, rather than having to poll the endpoint. 
[Here's the announcement for an overview](http://blog.iron.io/2013/01/ironmq-push-queues-reliable-message.html). 

### Create a Push Queue

```javascript
var options = {
    'message_timeout': 120,
    'message_expiration': 24 * 3600,
    'push': {
        'subscribers': [
            {
                'name': 'subscriber_name',
                'url': 'http://rest-test.iron.io/code/200?store=key1',
                'headers': {
                    'Content-Type': 'application/json'
                }
            }
        ],
        'retries': 4,
        'retries_delay': 30,
        'error_queue': 'error_queue_name'
    }
};
imq.create_queue("test_name", options, function(error, body) {})
```

**Options:**

* `type`: String or symbol. Queue type. `:pull`, `:multicast`, `:unicast`. Field required and static.
* `message_timeout`: Integer. Number of seconds before message back to queue if it will not be deleted or touched.
* `message_expiration`: Integer. Number of seconds between message post to queue and before message will be expired.

**Push queues only:**

* `push: subscribers`: An array of subscriber hashes containing a `name` and a `url` required fields,
and optional `headers` hash. `headers`'s keys are names and values are means of HTTP headers.
This set of subscribers will replace the existing subscribers.
To add or remove subscribers, see the add subscribers endpoint or the remove subscribers endpoint.
See below for example json.
* `push: retries`: How many times to retry on failure. Default is 3. Maximum is 100.
* `push: retries_delay`: Delay between each retry in seconds. Default is 60.
* `push: error_queue`: String. Queue name to post push errors to.


### Update a Message Queue

Same as create queue

```javascript
queue.update(options, function(error, body) {});
```

--

### Add/Remove Subscribers on a Queue

```javascript
var subscribers = [
    {
        'name': 'first',
        'url': 'http://first.endpoint.xx/process',
        'headers': {
            'Content-Type': 'application/json'
        }
    },
    {
        'name': 'second',
        'url': 'http://second.endpoint.xx/process',
        'headers': {
            'Content-Type': 'application/json'
        }
    }
];
queue.add_subscribers(subscribers, function(error, body) {});

queue.rm_subscribers({name: 'first'}, function(error, body) {});

queue.rm_subscribers(
  [{name: 'first'},
   {name: 'second'}],
  function(error, body) {}
);
```

### Replace Subscribers on a Queue

Sets list of subscribers to a queue. Older subscribers will be removed.

```javascript
var subscribers = [
    {
        "name": "the_only",
        "url": "http://my.over9k.host.com/push"
    }
];
queue.rpl_subscribers(subscribers, function(error, body) {});
```

--

### Add alerts to a queue. This is for Pull Queue only.

```javascript
queue.add_alerts(
              [
                  {
                      type: 'fixed',
                      direction: 'asc',
                      trigger: 201,
                      queue: 'my_queue_for_alerts',
                      snooze: 11
                  },{
                      type: 'fixed',
                      direction: 'desc',
                      trigger: 202,
                      queue: 'my_queue_for_alerts',
                      snooze: 12
                  }
              ],
              function(error, body) {}
          );
```

### Replace current queue alerts with a given list of alerts. This is for Pull Queue only.

```javascript
queue.update_alerts(
              [
                  {
                      type: 'fixed',
                      direction: 'asc',
                      trigger: 211,
                      queue: 'my_queue_for_alerts',
                      snooze: 20
                  },{
                      type: 'fixed',
                      direction: 'desc',
                      trigger: 212,
                      queue: 'my_queue_for_alerts',
                      snooze: 21
                  }
              ],
              function(error, body) {}
          );
```

### Remove alerts from a queue. This is for Pull Queue only.

```javascript
queue.delete_alerts(
          [
              {
                 id: 'xxxxxxxxxxxxxxxxxx1'
              },
              {
                  id: 'xxxxxxxxxxxxxxxxxx2'
              },
          ],
          function(error, body) {} );
```

### Remove alert from a queue by its ID. This is for Pull Queue only.

```javascript
queue.delete_alert_by_id(alert_id, function(error, body) {});
```

--

### Get Message Push Status

After pushing a message:

```javascript
queue.msg_push_statuses(message_id, function(error, body) {});
```

--

### Acknowledge / Delete Message Push Status

```javascript
queue.del_msg_push_status(
  message_id, reservation_id, subscriber_name, function(error, body) {});
```

--

### Revert Queue Back to Pull Queue

If you want to revert you queue just update `push_type` to `"pull"`.

```javascript
queue.update({push_type: "pull"}, function(error, body) {});
```

--

## Further Links

* [IronMQ Overview](http://dev.iron.io/mq/)
* [IronMQ REST/HTTP API](http://dev.iron.io/mq/reference/api/)
* [Push Queues](http://dev.iron.io/mq/reference/push_queues/)
* [Other Client Libraries](http://dev.iron.io/mq/libraries/)
* [Live Chat, Support & Fun](http://get.iron.io/chat)

-------------
© 2011 - 2013 Iron.io Inc. All Rights Reserved.
