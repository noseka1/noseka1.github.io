+++
title = "Five Basic Tips for Logging in Java"
date = "2015-05-18"
slug = "2015/05/18/five-basic-tips-for-logging-in-java"
Categories = []
+++
Where do you look when your server application doesn't work as expected? Application logs can provide invaluable information when the systems in production misbehave. That's why a great software practitioner desings the application logs carefully. Let's review a couple of basic tips for great logging in Java.

<!-- more -->

1) Use SLF4J logging facade
---------------------------
There are different Java logging frameworks out there: java.util.logging, Log4j, Log4j2 and Logback to name the most popular ones. [SLF4J](http://www.slf4j.org/ "Simple Logging Facade for Java (SLF4J)") is a thin facade that can talk to all of these logging frameworks. You can write your code using the SLF4J API and plugin in the desired logging framework at deployment time. Instead of tying yourself to Log4j framework like this code example does:

{% codeblock lang:java Log4jApp.java %}
import org.apache.log4j.Logger;

public class Log4jApp {

	private static final Logger log = Logger.getLogger(Log4jApp.class);

	public static void main(String[] args) {
		log.info("Application using Log4j directly");
	}
}
{% endcodeblock %}

Write your application using the SLF4J API:

{% codeblock lang:java Slf4jApp.java %}
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Slf4jApp {

	private static final Logger log = LoggerFactory.getLogger(LogApp.class);

	public static void main(String[] args) {
		log.info("Application using SLF4J facade");
	}

}
{% endcodeblock %}
In the second example you can see no `org.apache.log4j` package dependencies. At deployment time you can indeed plug-in Log4j framework as a logging backend but you can also opt for some of the other logging alternatives. You can refer to my [previous article](/blog/2015/05/11/java-logging-quick-reference/ "Java Logging Quick Reference") on how to configure SLF4J with different logging frameworks.

If you're writing a library and need to do logging you should definitely consider using SLF4J. Having your library depend on a particular logging framework will not make your users happy if they prefer to deploy a different logging framework. Even when writing a stand-alone application you might come to the point when you need to break it up into libraries when it gets bigger. Save your time and use SLF4J up front.

2) Keep the logging performance in mind
---------------------------------------
You can improve the performance of your logging code by embracing the following programming idioms.

Instead of concatenating strings to form the logging message let the logging framework do it for you. In the following example assume that the log level was set to `info`, therefore debug messages won't be logged at all. The code at line 5 is concatenating five string objects in order to create one temporary string object which is not used anyway. You should instead follow the example at line 8. The formatting of the log message is done by the logging framework which is optimized to do the formatting only if the resulting message will actually be logged. Besides that, the code at line 8 is more readable than the variant at line 5.
{% codeblock lang:java %}
String foo = "foo";
String bar = "bar";

// don't do this!
log.debug("Value of foo is " + foo + ". Value of bar is " + bar + ".");

// do this instead
log.debug("Value of foo is {}. Value of bar is {}.", foo, bar);
{% endcodeblock %}

If it is expensive to obtain the data to be logged make sure that the log level is high enough before you go to retrieve the data. The example below fetches the debug data from the database only if the current log level is `debug`:
{% codeblock lang:java %}
if (log.isDebugEnabled()) {
	String data = null;

	// ...
	// fetch the data from the database
	// ...

	log.debug("Data from the database: " + data);
}
{% endcodeblock %}

3) Log Java exceptions
----------------------
Even if you'd like to ignore Java exceptions at some places, do at least log it before you ignore it. Logging frameworks can log the stack trace of the Java exception nicely. The code in the following example catches and logs a `NullPointerException`:
{% codeblock lang:java LogException.java %}
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LogException {

	private static final Logger log = LoggerFactory.getLogger(LogException.class);

	private static void exceptionalMethod() {
		throw new NullPointerException("My exception to be logged");
	}
	public static void main(String[] args) {
		try {
			exceptionalMethod();
		}
		catch (Exception e) {
			log.error("Something went wrong", e);
		}
	}
{% endcodeblock %}
This produces the following output in your logs:
{% codeblock %}
22:16:58.677 [main] ERROR LogException - Something went wrong
java.lang.NullPointerException: My exception to be logged
	at LogException.exceptionalMethod(LogException.java:9) ~[bin/:na]
	at LogException.main(LogException.java:13) ~[bin/:na]
{% endcodeblock %}

4) Mark and filter your log messages
------------------------------------
Java logging frameworks allow you to filter log messages based on the logger name and the message log level. Logback and Log4j2 frameworks come with an additional filtering facility: Markers. You can tag your log messages with user-defined markers in order to filter them later on. In our example, we want to store the timing messages in a separate log file for later processing. In order to accomplish this we'll mark timing messages with the `time` marker. You can see the definition of the `time` marker at line 19 and logging of the marked message at line 20:

{% codeblock lang:java MarkerApp.java %}
mport org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.Marker;
import org.slf4j.MarkerFactory;

public class MarkerApp {

	private static final Logger log = LoggerFactory.getLogger(MarkerApp.class);

	static void processRequest() {
		log.info("Log info message");
	}

	public static void main(String[] args) {
		long startTime = System.nanoTime();
		processRequest();
		long endTime = System.nanoTime();

		Marker timeMarker = MarkerFactory.getMarker("time");
		log.info(timeMarker, "Request processing took {} ms", (endTime - startTime)/1e6);
	}
}
{% endcodeblock %}

In the logging configuration file we'll create two log appenders. The console appender sends all logging messages to the console output. In addition, the file appender logs only the messages marked with the `time` marker into the `/tmp/logger.out` log file.

{% codeblock lang:xml log4j2.marker.xml %}
<Configuration>
    <Appenders>
        <Console name="console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss} [%t] %-5p %c{1}:%L - %m%n" />
        </Console>
        <File name="file" fileName="/tmp/logger.out" append="true">
            <PatternLayout pattern="%d{HH:mm:ss} [%t] %-5p %c{1}:%L - %m%n"/>
            <Filters>
                <MarkerFilter marker="time" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
        </File>
    </Appenders>
    <Loggers>
        <Root level="debug">
            <AppenderRef ref="console" />
            <AppenderRef ref="file" />
        </Root>
    </Loggers>
</Configuration>
{% endcodeblock %}

If you run the example you'll see the following output on your console:

{% codeblock %}
22:55:48 [main] INFO  MarkerApp:11 - Log info message
22:55:48 [main] INFO  MarkerApp:20 - Request processing took 12.09897 ms
{% endcodeblock %}

You will find only the marked timing message in the `/tmp/logger.out` log file:
{% codeblock %}
22:55:48 [main] INFO  MarkerApp:20 - Request processing took 12.09897 ms
{% endcodeblock %}

5) Leverage diagnostic context in multithreaded applications
------------------------------------------------------------
Most server applications need to handle multiple clients simultaneously. Typically, the server application allocates a separate thread to handle a single client request. In such a system different threads handle different client requests in parallel and the log messages written by the threads interleave. In order to differentiate log messages from different threads from each other a diagnostic context comes in handy. Diagnostic context is a map associated with a particular thread. Each thread maintains its own map. You can store arbitrary key-value pairs in the map and in turn lay out your log messages to include the values from the map.

In the following example we want to log the name of the user on behalf of which we're doing some processing. In order to accomplish this we store the name of the user in the map under the key `user` before we start the processing. After the processing is complete we clear the map to get it ready for the next user.

{% codeblock lang:java MdcApp.java %}
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

public class MdcApp {

	private static final Logger log = LoggerFactory.getLogger(MdcApp.class);

	static void process() {
		log.info("Processing 1");
		log.info("Processing 2");
		log.info("Processing 3");
	}

	public static void main(String[] args) {
		MDC.put("user", "Joe");
		process();
		MDC.clear();
		MDC.put("user", "Eleonora");
		process();
	}
}
{% endcodeblock %}

In the logging configuration we need to configure the `PatternLayout` to include the `user` value from the map. The `%X{user}` formatting sequence does exactly that.

{% codeblock lang:xml log4j2.mdc.xml %}
<Configuration>
    <Appenders>
        <Console name="console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss} [%t] [%X{user}] %p %m%n" />
        </Console>
    </Appenders>
    <Loggers>
        <Root level="debug">
            <AppenderRef ref="console" />
        </Root>
    </Loggers>
</Configuration>
{% endcodeblock %}

When you run the `MdcApp` application you'll see the following output on your console:
{% codeblock %}
00:01:30 [main] [Joe] INFO Processing 1
00:01:30 [main] [Joe] INFO Processing 2
00:01:30 [main] [Joe] INFO Processing 3
00:01:30 [main] [Eleonora] INFO Processing 1
00:01:30 [main] [Eleonora] INFO Processing 2
00:01:30 [main] [Eleonora] INFO Processing 3
{% endcodeblock %}
Now you can tell the name of the user for whom you did the processing.
