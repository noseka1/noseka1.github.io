---
layout: post
title: "Troubleshooting the Performance of Vert.x Applications, Part II &mdash; Preventing Event Loop Delays"
date: 2019-07-22 11:15:42 -0700
comments: true
categories: development
---

In the [previous part](/blog/2019/06/30/troubleshooting-the-performance-of-vert-dot-x-applications-part-i-the-event-loop-model/) of the series, we took a closer look at the event loop model. In this article, we are going to discuss several techniques that help you to prevent event loop delays.

<!-- more -->

The causes of event loop delays can be divided into two categories. The first category contains event loop delays caused by a handler calling a blocking API. The second category covers delays caused by a handler code taking a great amount of CPU time to complete. Let's start with the first category and talk about blocking API calls.

## Working with blocking APIs

Calling a blocking API on the event loop thread is especially hurtful for the performance of your application and you should avoid it at all cost. When you call a blocking API from the event loop thread, the event loop thread will be put to sleep, i.e. it will relinquish the CPU. The duration of the sleep can be rather long in comparison to how much work the event loop thread could have accomplished if it would remain executing on the CPU. This is going to result in a serious decrease of the throughput of your application. In addition to impacting the throughput, the latency of your application is going to raise, too. Because as the event loop thread is sleeping no processing is taking place and so all the outstanding work is going to be pushed back by the duration of the sleep.

Common examples of blocking APIs that you should not call from the event loop thread are:

* "Old" Java  I/O APIs found in the `java.io` package
* JDBC APIs
* Locking APIs in the `java.util.concurrent.locks` package
* Using `synchronized ` keyword in your code
* Other blocking APIs

You should also check the various third-party libraries you may be using to ensure that their APIs are non-blocking. Sometimes the differences can be really subtle. For example, if you are using [Apache Log4j 2](https://logging.apache.org/log4j/2.x/) library for logging, you may want to configure it to use [asynchronous loggers](https://logging.apache.org/log4j/log4j-2.0/manual/async.html) when logging from the event loop.

There are situations where you cannot avoid using blocking APIs. A typical example is when a third-party library you want to use provides only blocking APIs. As there is no way how to execute a blocking API on the event loop thread without putting this thread to sleep, your only option in Vert.x is to offload the blocking calls to a worker thread. I am going to show you two techniques how you can accomplish this.

The first technique is straight forward. It leverages the `executeBlocking` method provided by Vert.x. In the following example, the event loop thread schedules a `blockingCodeHandler` to run on a worker thread by calling the `vertx.executeBlocking()` method. After the execution of the `blockingCodeHandler` is complete, the `resultHandler` will be executed  on the event loop thread that made the original `vertx.executeBlocking()` call:

{% codeblock lang:java %}
class ExecuteBlockingExample extends AbstractVerticle {

	@Override
	public void start() {
		// on the event loop thread
		System.out.println("Calling from " + Thread.currentThread().getName());

		Handler<Future<String>> blockingCodeHandler = future -> {
			// executed on a worker thread
			System.out.println("Work executed on " + Thread.currentThread().getName());
			future.complete("OK");
		};

		Handler<AsyncResult<String>> resultHandler = result -> {
			// back on the calling event loop thread
			System.out.println("Result '" + result.result() + "' received on " + Thread.currentThread().getName());
		};

		vertx.executeBlocking(blockingCodeHandler, resultHandler);
	}
}
{% endcodeblock %}

After running the example code, you will see the following output:

{% codeblock lang:java %}
Calling from vert.x-eventloop-thread-0
Work executed on vert.x-worker-thread-0
Result 'OK' received on vert.x-eventloop-thread-0
{% endcodeblock %}

The second technique for offloading the blocking API calls to a worker thread is a bit more involved. We are going to deploy a worker verticle and send it the work as a message using the event bus. After the worker thread completes the work it will reply sending the result back to us.

{% codeblock lang:java %}
vertx.deployVerticle(new AbstractVerticle() {
	@Override
	public void start(Future<Void> startFuture) {

		// work handler
		Handler<Message<String>> handler = message -> {
			System.out.println("Received message on " + Thread.currentThread().getName());

			// do work
			System.out.println("Working ...");

			message.reply("OK");
		};

		// wait for work
		vertx.eventBus().consumer("worker", handler).completionHandler(r -> {
			startFuture.complete();
		});
	}
}, new DeploymentOptions().setWorker(true));

vertx.deployVerticle(new AbstractVerticle() {
	@Override
	public void start() {

		// reply handler
		Handler<AsyncResult<Message<String>>> replyHandler = message -> {
			System.out.println(
				"Received reply '" + message.result().body() + "' on " + Thread.currentThread().getName());
		};

		// dispatch work
		vertx.eventBus().send("worker", "request", replyHandler);
	}
});
{% endcodeblock %}

After running the above code  you will receive the following output:

{% codeblock lang:sh %}
Received message on vert.x-worker-thread-1
Working ...
Received reply 'OK' on vert.x-eventloop-thread-0
{% endcodeblock %}

## Executing compute intensive tasks

What is a compute intensive task? It is a task that makes heavy use of CPU and memory. Common examples of compute intensive tasks are parsing, encryption, compression and others. Executing compute intensive task within the event loop handler doesn't affect the throughput of your application because the event loop thread is busy doing useful work which would need to be done anyway. However, as the event loop thread is kept busy, other handlers on the event loop will be processed with a delay. How can we improve the situation and allow other handlers to be processed in a timely fashion?

Let's assume that you are able to chunk up the compute intensive task into several chunks. Then instead of running the entire compute intensive task at once:

{% codeblock lang:java %}
workChunk1();
workChunk2();
workChunk3();
{% endcodeblock %}

You can distribute the execution of individual chunks in time allowing the event loop to process other handlers in between. In the following example, we are creating pauses of 100 milliseconds between the work chunks to allow the event loop to interleave other handlers:

{% codeblock lang:java %}
long delay = 100;

workChunk1();
vertx.setTimer(delay, timerId -> {
	workChunk2();
	vertx.setTimer(delay, timerId2 -> {
		workChunk3();
	});
});
{% endcodeblock %}

You may encounter scenarios where you won't be able to chunk up the compute intensive task. For example, the compute intensive task's code is contained within a third-party library and executes all at once. In this case, you will have to defer to running this task on a worker thread and incur the cost of context switching. The operating system scheduler will periodically preempt the compute intensive task to prevent it from hogging the CPU and giving your event loop threads a chance to run.

## Conclusion

In this article, we discussed how to work with blocking APIs in Vert.x. A blocking API call has to be made on a worker thread and not on an event loop thread. Futhermore, we described a technique that allows you to execute compute intensive tasks on the event loop without considerably delaying the processing of other tasks on the same event loop.

If you have any comments or questions please feel free to use the comment section below. In the final article in the series we will cover some techniques for troubleshooting event loop delays.
