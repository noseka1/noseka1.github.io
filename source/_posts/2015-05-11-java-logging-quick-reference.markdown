---
layout: post
title: "Java Logging Quick Reference"
date: 2015-05-11 00:13:09 -0700
comments: true
categories: development java
---

The Simple Logging Facade for Java (SLF4J) serves as a simple abstraction for various logging frameworks. Let's look at how to configure SLF4J to work with SLF4J Simple logger, JDK 1.4 logger, Log4j, Logback and Log4j2 framework.

<!-- more -->

Following is a Java code of our logging application `LogApp`. Note that it uses SLF4J API classes to do the logging. The jar file `slf4j-api-1.7.12.jar` is the only compile time dependency.
{% codeblock lang:java LogApp.java %}
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LogApp {

	private static final Logger log = LoggerFactory.getLogger(LogApp.class);

	public static void main(String[] args) {
		log.trace("Trace message");
		log.debug("Debug message");
		log.info("Info message");
		log.warn("Warning message");
		log.error("Error message");
	}
}
{% endcodeblock %}

You can compile the LogApp code with:

{% codeblock lang:sh %}
javac -cp slf4j-api-1.7.12.jar LogApp.java
{% endcodeblock %}

And run the LogApp with:

{% codeblock lang:sh %}
java -cp .:slf4j-api-1.7.12.jar LogApp
{% endcodeblock %}

The output shows that SLF4J is missing a logger implementation. In the following we'll demonstrate how to plug in different logging backends. The principle is always the same: include the logging framework implementation jars on the classpath, include the respective SLF4J adaptor jar on the classpath and provide a configuration file specific to the logging framework.
{% codeblock %}
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
{% endcodeblock %}


SLF4J Simple logger
-------------------

The SLF4J comes with a Simple logger implemenation. Simple logger provides only basic functionality. It read its configuration from a `simplelogger.properties` file that must be included on the classpath. There's no way to specify a different location of the configuration file.

{% codeblock lang:properties simplelogger.properties - Output to console %}
org.slf4j.simpleLogger.logFile=System.out
org.slf4j.simpleLogger.defaultLogLevel=debug
org.slf4j.simpleLogger.showDateTime=true
org.slf4j.simpleLogger.dateTimeFormat=HH:mm:ss.SSS
{% endcodeblock %}

{% codeblock lang:properties simplelogger.properties - Output to file %}
org.slf4j.simpleLogger.logFile=/tmp/logger.out
org.slf4j.simpleLogger.defaultLogLevel=debug
org.slf4j.simpleLogger.showDateTime=true
org.slf4j.simpleLogger.dateTimeFormat=HH:mm:ss.SSS
{% endcodeblock %}

Run the application with:

{% codeblock lang:sh %}
java -cp .:slf4j-api-1.7.12.jar:slf4j-simple-1.7.12.jar LogApp
{% endcodeblock %}

JDK 1.4 logger (java.util.logging)
------------------------------

The java.util.logging package was introduced in JDK 1.4. The default `logging.properties` configuration file that comes with JRE (`jre/lib/logging.properties`) specifies a ConsoleHandler that routes logging to System.err. There's no way how to configure the JDK 1.4 logger to log to standard output instead of standard error output unless you do the configuration programmaticaly. You can specify the location of your JDK 1.4 logging configuration file in `java.util.logging.config.file` Java property.

{% codeblock lang:properties jdk14.stderr.properties - Output to console %}
handlers=java.util.logging.ConsoleHandler
.level=FINE
java.util.logging.ConsoleHandler.level=FINE
java.util.logging.ConsoleHandler.formatter=java.util.logging.SimpleFormatter
# message pattern works since Java 7
java.util.logging.SimpleFormatter.format=%1$tT [%2$s] %4$s - %5$s %6$s%n
{% endcodeblock %}

{% codeblock lang:properties jdk14.file.properties - Output to file %}
handlers=java.util.logging.FileHandler
.level=FINE
java.util.logging.FileHandler.level=FINE
java.util.logging.FileHandler.pattern=/tmp/logger.out
java.util.logging.FileHandler.append=true
java.util.logging.FileHandler.formatter=java.util.logging.SimpleFormatter
# message pattern works since Java 7
java.util.logging.SimpleFormatter.format=%1$tT [%2$s] %4$s - %5$s %6$s%n
{% endcodeblock %}

Run the application with:

{% codeblock lang:sh %}
java -cp .:slf4j-api-1.7.12.jar:slf4j-jdk14-1.7.12.jar -Djava.util.logging.config.file=/tmp/jdk14.stderr.properties LogApp
{% endcodeblock %}

Log4j
-----
The Log4j doesn't log a single message for you unless you provide it with a proper configuration:
{% codeblock %}
log4j:WARN No appenders could be found for logger (LogApp).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
{% endcodeblock %}

The following are the sample Log4j configuration files. Note that the `log4j.configuration` Java property that specifies the location of the configuration file must be a URL. In the example below the `/tmp/log4j.stdout.properties` location has to be prepended with `file:` to form a URL.

{% codeblock lang:properties log4j.stdout.properties - Output to console %}
log4j.rootLogger=DEBUG, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.Target=System.out
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d{HH:mm:ss} [%t] %-5p %c{1}:%L - %m%n
{% endcodeblock %}

{% codeblock lang:properties log4j.file.properties - Output to file %}
log4j.rootLogger=DEBUG, file
log4j.appender.file=org.apache.log4j.FileAppender
log4j.appender.file.File=/tmp/logger.out
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{HH:mm:ss} [%t] %-5p %c{1}:%L - %m%n
{% endcodeblock %}

Run the application with:

{% codeblock lang:sh %}
java -cp .:slf4j-api-1.7.12.jar:slf4j-log4j12-1.7.12.jar:log4j-1.2.17.jar -Dlog4j.configuration=file:/tmp/log4j.stdout.properties LogApp
{% endcodeblock %}

Logback
-------
With no configuration provided Logback defaults to printing all log messages on the console standard output. The `logback-classic` jar package that comes with Logback includes the `org.slf4j.impl.StaticLoggerBinder` class that serves as an adaptor to SLF4J framework. Therefore no extra SLF4J adaptor jar is needed on the runtime classpath. You can specify the location of your Logback configuration file in the `logback.configurationFile` Java property.

{% codeblock lang:xml logback.stdout.xml - Output to console %}
<configuration>
  <appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
    <Target>System.out</Target>
    <encoder>
      <pattern>%d{HH:mm:ss} [%t] %-5p %c{1}:%L - %m%n</pattern>
    </encoder>
  </appender>
  <root level="DEBUG">
    <appender-ref ref="stdout"/>
  </root>
</configuration>
{% endcodeblock %}

{% codeblock lang:xml logback.file.xml - Output to file %}
<configuration>
  <appender name="file" class="ch.qos.logback.core.FileAppender">
    <File>/tmp/logger.out</File>
    <encoder>
      <pattern>%d{HH:mm:ss} [%t] %-5p %c{1}:%L - %m%n</pattern>
    </encoder>
  </appender>
  <root level="DEBUG">
    <appender-ref ref="file"/>
  </root>
</configuration>
{% endcodeblock %}

Run the application with:

{% codeblock %}
java -cp .:slf4j-api-1.7.12.jar:logback-core-1.1.3.jar:logback-classic-1.1.3.jar -Dlogback.configurationFile=/tmp/logback.stdout.xml LogApp
{% endcodeblock %}

Log4j2
-------

With no configuration provided Log4j2 informs you that it logs only error messages to stdout. You can provide the location of your Log4j2 configuration file in the `log4j.configurationFile` Java property.

{% codeblock lang %}
ERROR StatusLogger No log4j2 configuration file found. Using default configuration: logging only errors to the console.
22:09:13.052 [main] ERROR LogApp - Error message
{% endcodeblock %}

{% codeblock lang:xml log4j2.stdout.xml - Output to console %}
<Configuration>
    <Appenders>
        <Console name="console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss} [%t] %-5p %c{1}:%L - %m%n" />
        </Console>
    </Appenders>
    <Loggers>
        <Root level="debug">
            <AppenderRef ref="console" />
        </Root>
    </Loggers>
</Configuration>
{% endcodeblock %}

{% codeblock lang:xml log4j2.file.xml - Output to file %}
<Configuration>
    <Appenders>
        <File name="file" fileName="/tmp/logger.out" append="true">
            <PatternLayout pattern="%d{HH:mm:ss} [%t] %-5p %c{1}:%L - %m%n"/>
        </File>
    </Appenders>
    <Loggers>
        <Root level="debug">
            <AppenderRef ref="file" />
        </Root>
    </Loggers>
</Configuration>
{% endcodeblock %}

Run the application with:

{%codeblock lang:sh %}
java -cp .:slf4j-api-1.7.12.jar:log4j-core-2.2.jar:log4j-api-2.2.jar:log4j-slf4j-impl-2.2.jar -Dlog4j.configurationFile=/tmp/log4j2.stdout.xml LogApp
{% endcodeblock %}
