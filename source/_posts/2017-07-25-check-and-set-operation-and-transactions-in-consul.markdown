---
layout: post
title: "Check-and-Set Operation and Transactions in Consul"
date: 2017-07-25 23:07:07 -0700
comments: true
categories: cloud devops
---

In the [previous](/blog/2017/07/15/first-look-at-the-key-value-store-in-consul/) blog post, we were checking out the basic functionality of the key-value store in Consul. In this article, we will explore two of the more advanced features of Consul's key-value store, namely: Check-and-Set operation and transactions.

<!-- more -->

{% img right /images/posts/consul_logo.png 200 300 %}

For our experimenting, let's start a one-node Consul cluster. The meaning of the individual command-line parameters was described in the [previous](/blog/2017/07/15/first-look-at-the-key-value-store-in-consul/) article:

{% codeblock lang:sh %}
$ ./consul agent -ui -server -data-dir mydata -advertise 127.0.0.1 -bootstrap-expect 1
{% endcodeblock %}

In a short moment, the one-node Consul cluster should be up and ready. In the following, we're going to leverage Consul's [HTTP API](https://www.consul.io/api/index.html) as not all the desired functionality is exposed via the command-line client. First, let's verify that the Consul cluster is working properly. For that, we'll ask it to provide us with a list of cluster nodes:

{% codeblock lang:sh %}
$ curl http://localhost:8500/v1/catalog/nodes?pretty
[
    {
        "ID": "be79786e-749d-758c-2b65-824c1e956788",
        "Node": "zihadlo",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "Meta": {},
        "CreateIndex": 5,
        "ModifyIndex": 6
    }
]
{% endcodeblock %}

The response from Consul contains information about the single node which is what we expected.

## Check-and-Set operation

The purpose of the Check-and-Set operation is to avoid lost updates when multiple clients are simultaneously trying to update a value of the same key. Check-and-Set operation allows the update to happen only if the value has not been changed since the client last read it. If the current value does not match what the client previously read, the client will receive a conflicting update error message and will have to retry the read-update cycle.

The Check-and-Set operation can be used to implement a shared counter, semaphore or a distributed lock. Let's demonstrate how to create a basic distributed lock using the Check-and-Set operation. We'll start with creating a key that will represent our lock:

{% codeblock lang:sh %}
$ curl --request PUT http://localhost:8500/v1/kv/mylock --data ""
true
{% endcodeblock %}

We created the `mylock` key holding an empty value. The empty value signalizes that the lock is not taken. Before trying to acquire the lock, each client has to check whether the lock is unlocked:

{% codeblock lang:sh %}
$ curl http://localhost:8500/v1/kv/mylock?pretty
[
    {
        "LockIndex": 0,
        "Key": "mylock",
        "Flags": 0,
        "Value": null,
        "CreateIndex": 5638,
        "ModifyIndex": 5638
    }
]
{% endcodeblock %}

The value of the key in the Consul's response is still empty (null) which indicates that nobody is holding the lock. The second important item in the Consul's response is the `ModifyIndex`. Each key in the key-value store has its own `ModifyIndex`. The `ModifyIndex` is incremented by Consul each time the respective key is modified.

After verifying that the lock is not taken, the client can try to acquire it:

{% codeblock lang:sh %}
$ curl --request PUT http://localhost:8500/v1/kv/mylock?cas=5638 --data "client1"
true
{% endcodeblock %}

The client is trying to update the value of the key `mylock`. The value of the `ModifyIndex` is passed along as the query parameter `cas=5638` (cas meaning Check-and-Set). Because the query parameter `cas=5638` is specified in the request, Consul will update the value of the `mylock` key only if the current `ModifyIndex` of the `mylock` key matches 5638. In other words, the key has not been updated since the client last read it. In our example, the update was successful and the client is now holding the lock. Note that an arbitrary non-empty value can be stored under the `mylock` key. We chose to use the identification of the client that has acquired the lock.

Let's pretend that at the same time a second client was competing for the lock. The `client2` was trying to acquire the lock by sending this request to Consul including the same query parameter `cas=5638`:

{% codeblock lang:sh %}
$ curl --request PUT http://localhost:8500/v1/kv/mylock?cas=5638 --data "client2"
false
{% endcodeblock %}

Consul's response sent to `client2` shows that Consul refused to update the `mylock` value as, in the meantime, this value has been modified. In order to check the current status of the lock, `client2` can follow up with a get request:

{% codeblock lang:sh %}
$ curl http://localhost:8500/v1/kv/mylock?pretty
[
    {
        "LockIndex": 0,
        "Key": "mylock",
        "Flags": 0,
        "Value": "Y2xpZW50MQ==",
        "CreateIndex": 5638,
        "ModifyIndex": 5801
    }
]
{% endcodeblock %}

In Consul's response we can see that the lock is currently being held by `client1`. Until `client1` hasn't released the lock, `client2` must not try to acquire it. It can only periodically check the status of the lock and wait until it is released. To release the lock, `clent1` will simply set its value to an empty-value:

{% codeblock lang:sh %}
curl --request PUT http://localhost:8500/v1/kv/mylock --data ""
true
{% endcodeblock %}

There are two more comments to add. First, the lock we implemented is purely advisory. All the clients working with the lock have to follow the same rules for the lock to function properly. Each client has to check that the lock was not acquired by somebody else before trying to acquire it. A misbehaved client can easily break the lock. Second, if the client holding the lock fails to release it (e.g. client crashes before releasing the lock), the lock will remain locked and no other client will be able to acquire it. More robust locks that are automatically released in the case of client failure can be implemented using the Consul's [sessions](https://www.consul.io/docs/internals/sessions.html) along with the acquire and release operations.

## Leveraging the parameter cas=0

In our lock implementation, we created an opened lock first and the lock acquisition comprised of two steps. In the first step, the client read the current `ModifyIndex` of the lock. In the second step, the client tried to update the lock while passing the `ModifyIndex` as a `cas` query parameter. When implementing the lock, we could have alternatively leveraged the fact that if the `cas` parameter is set to `0`, Consul will only create the key in the key-value store if it does not already exist. The state of our lock would then correspond to the existence or non-existence of the respective key in the key-value store. In order to acquire the lock, the client would send a request to create the key:

{% codeblock lang:sh %}
$ curl --request PUT localhost:8500/v1/kv/mykey2?cas=0 --data 'client1'
true
{% endcodeblock %}

And to release the lock, the client would simply remove the respective key from the key-value store:

{% codeblock lang:sh %}
$ curl --request DELETE localhost:8500/v1/kv/mykey2
true
{% endcodeblock %}

## Transactions

[Transactions](https://www.consul.io/api/txn.html) in Consul manage updates or selects of multiple keys within a single, atomic transaction. A list of operations that will be executed in the transaction is specified in the body of the HTTP request. First, let's create a list of operations and save it as a file `transaction1.txt`:

{% codeblock lang:sh transaction1.txt %}
[
  {
    "KV": {
      "Verb": "set",
      "Key": "foo",
      "Value": "MQ=="
    }
  },
  {
    "KV": {
      "Verb": "set",
      "Key": "bar",
      "Value": "Mg=="
    }
  }
]
{% endcodeblock %}

Our transaction doesn't do anything spectacular. It just creates two key-value pairs `foo=1` and `bar=2`. Let's submit the transaction to Consul:

{% codeblock lang:sh %}
curl --request PUT 'localhost:8500/v1/txn?pretty' --data @transaction1.txt
{
    "Results": [
        {
            "KV": {
                "LockIndex": 0,
                "Key": "foo",
                "Flags": 0,
                "Value": null,
                "CreateIndex": 7267,
                "ModifyIndex": 7267
            }
        },
        {
            "KV": {
                "LockIndex": 0,
                "Key": "bar",
                "Flags": 0,
                "Value": null,
                "CreateIndex": 7267,
                "ModifyIndex": 7267
            }
        }
    ],
    "Errors": null
}
{% endcodeblock %}

The transaction completed successfully. In the response from Consul, we can find the list of results. The order of results corresponds to the order of operations that we submitted in our request. The value of the `ModifyIndex` `7267` is the same for both keys `foo` and `bar` as they were updated in the same transaction.

Next, let's see what happens if one of the operations in the transaction fails. To demonstrate this, we'll create a transaction that consists of two operations. The first operation updates the key `foo` to value `10`. The second operation updates the key `bar` to value `20` but only if the `ModifyIndex` of `bar` matches 100. We know that this condition is not fulfilled and the update should fail.

{% codeblock lang:sh transaction2.txt %}
[
  {
    "KV": {
      "Verb": "set",
      "Key": "foo",
      "Value": "MTA="
    }
  },
  {
    "KV": {
      "Verb": "cas",
      "Index": 100,
      "Key": "bar",
      "Value": "MjA="
    }
  }
]
{% endcodeblock %}

Let's submit the transaction to Consul:

{% codeblock lang:sh %}
curl --request PUT 'localhost:8500/v1/txn?pretty' --data @transaction2.txt
{
    "Results": null,
    "Errors": [
        {
            "OpIndex": 1,
            "What": "failed to set key \"bar\", index is stale"
        }
    ]
}
{% endcodeblock %}

The transaction failed, indeed. The returned error list contains all errors that occurred during the transaction processing. The operations that failed are denoted by the `OpIndex` which starts from value 0. In the example output we can see that the second operation in our transaction failed because of the stale index. Let's check the values of the keys `foo` and `bar` after the failed transaction:

{% codeblock lang:sh %}
$ ./consul kv get foo
1
$ ./consul kv get bar
2
{% endcodeblock %}

As expected, due to the failed udpate the entire transaction has been rolled back. Keys `foo` and `bar` retained their original values `1` and `2`.

## Conclusion

In this blog post, we explored the Check-and-Set operation supported by Consul and used it to implement a simple distributed lock. In the second part of the article, we poked into the transaction capabilities of Consul.

And what about you? How is your experience with using Consul for distributed locking or leader election? Did you get a chance to use transactions? I would like to hear your experiences, feel free to use the comment section below.
