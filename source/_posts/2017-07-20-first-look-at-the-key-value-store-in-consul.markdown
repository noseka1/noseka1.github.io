---
layout: post
title: "First Look at the Key-Value Store in Consul"
date: 2017-07-15 20:38:38 -0700
comments: true
categories: devops, cloud
---

If you are developing a distributed application that consists of multiple services, you might be thinking about how to manage the ever growing application configuration data. Instead of maintaining individual configuration files for each service, you can store all your configuration data in a key-value store. In this blog post we'll check out the key-value store in Consul.

<!-- more -->

{% img right /images/posts/consul_logo.png 200 300 %}

Consul is an open-source product developed by [HashiCorp](https://www.hashicorp.com/) and licensed under the [MPL 2.0](https://github.com/hashicorp/consul/blob/master/LICENSE). While Consul uses an open core business model, it comes with a great deal of functionality in its free edition. The top two features of Consul would be the service discovery combined with health checking and the key-value store functionality that we are going to review in this article. They both come handy when building distributed applications.

## Getting started

HashiCorp products are known for its thorough documentation and the Consul's [documenation](https://www.consul.io/docs/index.html) is not an exception.

What I like about Consul is its installation. Written in the Go language, Consul is distributed as a single statically linked binary. Download [links](https://www.consul.io/downloads.html) for various platforms are provided. After unzipping the distribution archive you can directly run the `consul` executable.


Let's put togher a command-line to start the Consul cluster. Our test cluster consists of a single node (`-bootstrap-expect 1`). For a production deployment, you should be looking at a cluster of three or five Consul nodes that is able to survive node failures. We will make the Consul Web UI available at `http://localhost:8500` by appending the `-ui` parameter. Consul needs a location where it will persist its data. In our example, we instruct Consul to create a directory `mydata` and store all its data in this directory. After a bit of typing, the complete commmand-line to start the Consul cluster looks as follows:

{% codeblock lang:sh %}
./consul agent -ui -server -data-dir mydata -advertise 127.0.0.1 -bootstrap-expect 1
{% endcodeblock %}

In several seconds the one-node Consul cluster is up and running:

{% codeblock lang:sh %}
==> WARNING: BootstrapExpect Mode is specified as 1; this is the same as Bootstrap mode.
==> WARNING: Bootstrap mode enabled! Do not enable unless necessary
==> Starting Consul agent...
==> Consul agent running!
           Version: 'v0.8.5'
           Node ID: 'be79786e-749d-758c-2b65-824c1e956788'
         Node name: 'zihadlo'
        Datacenter: 'dc1'
            Server: true (bootstrap: true)
       Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
    Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2017/07/15 20:37:57 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:127.0.0.1:8300 Address:127.0.0.1:8300}]
    2017/07/15 20:37:57 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
    2017/07/15 20:37:57 [INFO] serf: EventMemberJoin: zihadlo 127.0.0.1
    2017/07/15 20:37:57 [INFO] consul: Adding LAN server zihadlo (Addr: tcp/127.0.0.1:8300) (DC: dc1)
    2017/07/15 20:37:57 [INFO] serf: EventMemberJoin: zihadlo.dc1 127.0.0.1
    2017/07/15 20:37:57 [INFO] consul: Handled member-join event for server "zihadlo.dc1" in area "wan"
    2017/07/15 20:37:57 [INFO] agent: Started DNS server 127.0.0.1:8600 (udp)
    2017/07/15 20:37:57 [INFO] agent: Started DNS server 127.0.0.1:8600 (tcp)
    2017/07/15 20:37:57 [INFO] agent: Started HTTP server on 127.0.0.1:8500
    2017/07/15 20:38:02 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2017/07/15 20:38:02 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
    2017/07/15 20:38:02 [INFO] raft: Election won. Tally: 1
    2017/07/15 20:38:02 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2017/07/15 20:38:02 [INFO] consul: cluster leadership acquired
    2017/07/15 20:38:02 [INFO] consul: New leader elected: zihadlo
    2017/07/15 20:38:02 [INFO] consul: member 'zihadlo' joined, marking health alive
    2017/07/15 20:38:02 [INFO] agent: Synced service 'consul'
{% endcodeblock %}

In the log output, Consul informs us that the Consul API is available at 127.0.0.1:8500. That's where the Consul client will connect to by default. In the following, you want to make sure that you're running the Consul commands on the same box as you started your Consul cluster.

The single `consul` binary provides the server as well as the client functionality. Let's list our current cluster members to verify that the client can connect to the cluster:

{% codeblock lang:sh %}
$ ./consul members
Node     Address         Status  Type    Build  Protocol  DC
zihadlo  127.0.0.1:8301  alive   server  0.8.5  2         dc1
{% endcodeblock %}

## Basic CRUD with Consul

In this section, we're going the exercise the basic Create, Read, Update and Delete functionality of the Consul key-store. First, let's store the value `12345` under the key `foo`:

{% codeblock lang:sh %}
$ ./consul kv put foo 12345
Success! Data written to: foo
{% endcodeblock %}

Great, the value is saved in the store. To retrieve the value under the key `foo` from Consul we can type:

{% codeblock lang:sh %}
$ ./consul kv get foo
12345
{% endcodeblock %}

By the way, Consul doesn't impose any restrictions on what kind of data you may store. Only the size of the data is limited to 512KB of data per key. It's up to your application, what data format you choose to use. For example, you can decide to store numbers, strings, JSON-formatted data or arbitrary binary data. For instance, when designing a centralized configuration management solution for your application, you have the flexibility of storing individual configuration options as key-value pairs or decide to save entire configuration files as values in Consul.

To replace the value, simply put an new value in Consul under the existing key:

{% codeblock lang:sh %}
$ ./consul kv put foo bar
Success! Data written to: foo
{% endcodeblock %}

The value has been successfully updated as we can tell:

{% codeblock lang:sh %}
$ ./consul kv get foo
bar
{% endcodeblock %}

To remove the value from the key-value store you can use the `delete` command:

{% codeblock lang:sh %}
$ ./consul kv delete foo
Success! Deleted key: foo
{% endcodeblock %}

To verify that the value is really gone, try to retrieve it:

{% codeblock lang:sh %}
$ ./consul kv get foo
Error! No key exists at: foo
{% endcodeblock %}

## Hierarchical keys and prefix matching

Keys in Consul can be organized in a hierarchy where different levels of the hierarchy are separated by the slash character (`/`). For example, you can create a database that holds the population numbers in different continents and countries (in millions of inhabitants) like this:

{% codeblock lang:sh %}
$ ./consul kv put europe 743.1
$ ./consul kv put europe/germany 82.67
$ ./consul kv put europe/france 66.9
$ ./consul kv put asia 4436
$ ./consul kv put asia/india 1324
{% endcodeblock %}

Now that you organized your keys hierarchically, you can use the Consul's prefix matching to discover the keys on the single level of hierarchy. For example, to retrive the keys with the prefix `e`:

{% codeblock lang:sh %}
$ ./consul kv get -recurse -keys e
europe
europe/
{% endcodeblock %}

Prefix matching can be used to retrieve the values, too. For example, to retrieve the population numbers in Europe, you can type:

{% codeblock lang:sh %}
$ ./consul kv get -recurse e
europe:743.1
europe/france:66.9
europe/germany:82.67
{% endcodeblock %}

Note that when retrieving the keys recursively, only the keys on the single level of hierarchy were returned whereas when retrieving the values recursively, values on all the nested levels of hierarchy were returned.

To obtain the population numbers for the European countries, you can append a slash to the keys name (`europe/`):

{% codeblock lang:sh %}
$ ./consul kv get -recurse europe/
europe/france:66.9
europe/germany:82.67
{% endcodeblock %}

And if you are interested only in the European countries that start with letter `g`, you can use the prefix `europe/g`:

{% codeblock lang:sh %}
./consul kv get -recurse europe/g
europe/germany:82.67
{% endcodeblock %}

## Export/import of key-value pairs

Another useful feaure of the Consul's key-value store is the bulk export and import of key-value pairs. To export the entire key-value store database, you can type:

{% codeblock lang:sh %}
$ ./consul kv export
[
        {
                "key": "asia",
                "flags": 0,
                "value": "NDQzNg=="
        },
        {
                "key": "asia/india",
                "flags": 0,
                "value": "MTMyNA=="
        },
        {
                "key": "europe",
                "flags": 0,
                "value": "NzQzLjE="
        },
        {
                "key": "europe/france",
                "flags": 0,
                "value": "NjYuOQ=="
        },
        {
                "key": "europe/germany",
                "flags": 0,
                "value": "ODIuNjc="
        }
]
{% endcodeblock %}

Consul exports the key-value pairs into the JSON format which is currently the only supported format. In the sample output, you can see that all the values are [base64](https://en.wikipedia.org/wiki/Base64) encoded. The base64 encoding is commonly used in the text-based formats like JSON and XML to allow embedding of binary data.

You can export a subset of the key-value pairs by specifying the prefix. For instance, to export the data pertaining Europe, you can speficy the `europe` prefix:

{% codeblock lang:sh %}
$ ./consul kv export europe
[
        {
                "key": "europe",
                "flags": 0,
                "value": "NzQzLjE="
        },
        {
                "key": "europe/france",
                "flags": 0,
                "value": "NjYuOQ=="
        },
        {
                "key": "europe/germany",
                "flags": 0,
                "value": "ODIuNjc="
        }
]
{% endcodeblock %}

To import the JSON-formatted data back to the Consul key-value store, you can use the command `./consul kv import`.

## Web UI

Besides the commmand-line client, you can access Consul through its beautiful Web interface. Point your web browser to [http://localhost:8500](http://localhost:8500).

{% img right /images/posts/consul_ui.png %}

## Conclusion

In this blog post, we reviewed the basics of the key-value store in Consul. There are many other cool features of the key-value store that we didn't cover like atomic key updates using Check-and-Set operations, [transactions](https://www.consul.io/api/txn.html), [locks](https://www.consul.io/docs/commands/lock.html) or [watches](https://www.consul.io/docs/commands/watch.html). Also, I recommend to you to take a look at the great Consul's [RESTful API](https://www.consul.io/api/index.html) that allows you to interact with Consul programatically.

If you're looking for a key-value store that would enhance your distributed application, Consul is definitely a candidate to consider. Besides that, Consul will be ready when you later on realize that service discovery is what you need to address next.
