+++
title = "Openstack Nova Notifications Subscriber"
date = "2015-05-25"
slug = "2015/05/25/openstack-nova-notifications-subscriber"
Categories = [ "devops", "cloud" ]
+++

OpenStack components generate notifications that can provide useful insight into what is going on in OpenStack. Let's create a simple subcriber that dumps incoming notifications from OpenStack Nova to standard output.

<!--more-->

Configure Nova to generate notifications
----------------------------------------

First let's make sure that Nova is configured to send out notifications. You should find the following lines in your `/etc/nova/nova.conf` file:

{{< highlight ini "linenos=table" >}}
[DEFAULT]
notification_topics=notifications
notification_driver=messagingv2
notify_on_state_change=vm_state
{{< / highlight >}}

Property `notification_topics` determines the base name of the topic (routing key) where the notification messages are sent to. The full name of the topic where the info-level notifications are published is `notifications.info`.
The possible choices of notification driver are briefly described in [oslo.messaging FAQ](http://docs.openstack.org/developer/oslo.messaging/FAQ.html "Frequently Asked Questions"). The `messagingv2` option instructs Nova to send notifications using the 2.0 message format that wraps the messages into an oslo.messaging envelope. The `notify_on_state_change` property determines the kind of notifications Nova should send out. You can set its value to `vm_and_task_state` if you want to receive additional notifications. After you modified your `/etc/nova/nova.conf` restart the Nova components for changes to take effect:

{{< highlight shell "linenos=table" >}}
$ sudo systemctl restart openstack-nova-api
$ sudo systemctl restart openstack-nova-compute
$ sudo systemctl restart openstack-nova-conductor
$ sudo systemctl restart openstack-nova-scheduler
{{< / highlight >}}

Implement Nova notifications subscriber
------------------------------------
Internally, Nova uses [Kombu](https://kombu.readthedocs.org/en/latest/ "Kombu Documentation") messaging library to connect to the RabbitMQ message broker. Let's use this Python library in our notifications subscriber to avoid the need to install additional libraries.

Nova sends out notification messages to a *topic* exchange called `nova` with the routing key `notifications.info`. In order to receive notification messages our client application needs to create a queue and bind it to the `nova` exchange. The binding key used to bind the queue to the `nova` exchange must match the routing key used by Nova to send out notification messages. Whenever there's a new message in the queue the Kombu library will invoke the `on_message` callback on our client to handle the message. The complete code of our notifications subscriber looks as follows:

{{< highlight python "linenos=table" >}}
#!/usr/bin/env python
import sys
import logging as log
from kombu import BrokerConnection
from kombu import Exchange
from kombu import Queue
from kombu.mixins import ConsumerMixin

EXCHANGE_NAME="nova"
ROUTING_KEY="notifications.info"
QUEUE_NAME="nova_dump_queue"
BROKER_URI="amqp://guest:guest@localhost:5672//"

log.basicConfig(stream=sys.stdout, level=log.DEBUG)

class NotificationsDump(ConsumerMixin):

    def __init__(self, connection):
        self.connection = connection
        return

    def get_consumers(self, consumer, channel):
        exchange = Exchange(EXCHANGE_NAME, type="topic", durable=False)
        queue = Queue(QUEUE_NAME, exchange, routing_key = ROUTING_KEY, durable=False, auto_delete=True, no_ack=True)
        return [ consumer(queue, callbacks = [ self.on_message ]) ]

    def on_message(self, body, message):
        log.info('Body: %r' % body)
        log.info('---------------')

if __name__ == "__main__":
    log.info("Connecting to broker {}".format(BROKER_URI))
    with BrokerConnection(BROKER_URI) as connection:
        NotificationsDump(connection).run()
{{< / highlight >}}

After you run the notifications subscriber and if everything went fine you should see the output similar to:

{{< highlight plaintext "linenos=table" >}}
INFO:root:Connecting to broker amqp://guest:guest@localhost:5672//
DEBUG:amqp:Start from server, version: 0.9, properties: {u'information': u'Licensed under the MPL.  See http://www.rabbitmq.com/', u'product': u'RabbitMQ', u'copyright': u'Copyright (C) 2007-2014 GoPivotal, Inc.', u'capabilities': {u'exchange_exchange_bindings': True, u'connection.blocked': True, u'authentication_failure_close': True, u'basic.nack': True, u'per_consumer_qos': True, u'consumer_priorities': True, u'consumer_cancel_notify': True, u'publisher_confirms': True}, u'cluster_name': u'rabbit@rdo-controller', u'platform': u'Erlang/OTP', u'version': u'3.3.5'}, mechanisms: [u'AMQPLAIN', u'PLAIN'], locales: [u'en_US']
DEBUG:amqp:Open OK!
INFO:kombu.mixins:Connected to amqp://guest@127.0.0.1:5672//
DEBUG:amqp:using channel_id: 1
DEBUG:amqp:Channel open
{{< / highlight >}}

Whenever you create/delete an instance in OpenStack a host of notification messages should be rolling on your screen.

Troubleshooting
---------------
RabbitMQ comes with a `rabbitmqctl` command which can be used to inspect the state of the exchanges, queues and bindings in the running RabbitMQ instance. First let's check that the `nova` topic exchange exists:

{{< highlight shell "linenos=table" >}}
$ sudo rabbitmqctl list_exchanges | grep nova
nova    topic
{{< / highlight >}}

Next let's make sure that our consumer queue was successfully created:

{{< highlight shell "linenos=table" >}}
$ sudo rabbitmqctl list_queues | grep nova_dump_queue
nova_dump_queue 0
{{< / highlight >}}

As a last thing we want to double-check that our `nova_dump_queue` was bound with the `nova` exchange using the binding key `notifications.info`:

{{< highlight shell "linenos=table" >}}
$ sudo rabbitmqctl list_bindings | grep nova_dump_queue
        exchange        nova_dump_queue queue   nova_dump_queue []
nova    exchange        nova_dump_queue queue   notifications.info      []
{{< / highlight >}}

References
----------

You can find more detailed information on topic exchanges in the great RabbitMQ tutorial [here](https://www.rabbitmq.com/tutorials/tutorial-five-python.html "Topics"). A version of the Nova notifications subscriber implemented with Python [Pika](http://pika.readthedocs.org/en/latest/ "Introduction to Pika") library is described in [this](https://prosuncsedu.wordpress.com/2014/01/08/notification-of-actions-in-openstack-nova/ "Action Notifications in OpenStack nova") blogpost.
