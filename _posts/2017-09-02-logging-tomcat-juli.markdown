---
layout: post
title:  "Tomcat JULI Logging"
date:   2017-09-02 20:20:00 -0600
categories: tomcat logging
---
Last month I faced two service failures that involved me looking through log files for more information about what had gone wrong. 
Each problems occurred on a service running in a tomcat instance and each problem was obscured by how our logs were structured.
For both problems an exception was causing the respective service to fail and the exception was not being captured in the service specific log.

Fortunately the exceptions were being captured in catalina.out. Unfortunately we have more than one service logging into catalina.out and not all 
of these services log useful well formatted messages. After sifting through logs, I was able to find the exceptions that I needed and correct the causes. I also found a new mission
find out how to change the format of log messages in catalina.out and how to raise the default log level for java.util.Logger by logger instance by config file.

The first goal I had after my adventure was to change the logging format in catalina.out primarily the date of the event was being logged as a unix timestamp. A great format for 
a computer but a bad one for a human. After checking with my team lead and system admin to verify that nothing cared about the log format in question I set about trying to change the 
timestamp to be in a standard that I could easily read. After some preliminary reading ( [Tomcat docs](https://tomcat.apache.org/tomcat-8.0-doc/logging.html) ), 
I had more reading ( [Tomcat JavaDoc](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/juli/package-summary.html), [Java's logging JavaDoc](http://docs.oracle.com/javase/6/docs/api/java/util/logging/package-summary.html) notice this one is for java 6 and is not very complete) )
and the config file I needed to change ${catalina.base}/conf/logging.properties.

At this point I leaned that log formatting for catalina.out is being handled by java.util.logging.SimpleFormatter. It is at this point that I also realized that the javadocs tomcat's site links to are not completely helpful.
I would like to say my next step was to find newer javadocs, but it was to codegrep to see what SimpleFormatter was doing. It turns out that it uses a String.format with a format String specified by a property and the properties off of LogRecord
to generate the String to be logged. At this point I looked up more recent [javadocs](https://docs.oracle.com/javase/7/docs/api/java/util/logging/SimpleFormatter.html) to find out how to set the format String.
Now with more up to date documentation. At this point I only needed one more piece of [documentation](https://docs.oracle.com/javase/7/docs/api/java/util/Formatter.html) to understand the arcane text of the format String.

Inorder to get a more readable timestamp in catalina.out I needed to add line in ${catalina.base}/conf/logging.properties that looks something like.

	java.util.logging.SimpleFormatter.format=%1$Ft %1$tT %2$s [%4$s] %5$s %6$s%n
	
Combining some docs together to better understand that bit of arcane trickery.

	%[argument_index$][flags][width][.precision]conversion

|argument_index | property | Description |
|1 | date | a Date object representing event time of the log record. |
|2 | source | a string representing the caller, if available; otherwise, the logger's name. |
|3 | logger | the logger's name. |
|4 | level | the log level. |
|5 | message | the formatted log message returned from the Formatter.formatMessage(LogRecord) method. It uses java.text formatting and does not use the java.util.Formatter format argument. |
|6 | thrown | a string representing the throwable associated with the log record and its backtrace beginning with a newline character, if any; otherwise, an empty string. |

To break it down further 

* %1$Ft is date formatted as yyyy-MM-dd
* %1$tT is date formatted as HH:mm:ss
* %2$s is the source of the logged event
* %4$s is the log level
* %5$s is the log message
* %6$ is the exception thrown
* %n is a newline character

With that my first goal was completed my second goal was to suppress log events coming from a 3rd party library. These events were printing soap requests and response into the log.
So every few minutes catalina.out would get around twenty to a hundred line of xml soap requests and response added to it. The [Tomcat docs](https://tomcat.apache.org/tomcat-8.0-doc/logging.html) had
already answered my questions on how to do handle application level logging. I needed to add a file to the web application's directory at WEB-INF/classes/logging.properties. The only changes I needed to make to the example logging.properties
were to add the lines raising the log level on the noisy class and removing the ConsoleHandler.

	handlers = org.apache.juli.FileHandler

	org.apache.juli.FileHandler.level = FINE
	org.apache.juli.FileHandler.directory = ${catalina.base}/logs
	org.apache.juli.FileHandler.prefix = MyApplication.
	
	org.example.NoisyClass.level = ERROR

With that all java.util.logging events in that service were redirected to a log file of its very own and all of the xml was suppressed. My second goal was accomplished.

To wrap up, Tomcat offers web application specific logging configured by WEB-INF/classes/logging.properties. To modify catalina.out's java.util.logging format the property to set is java.util.logging.SimpleFormatter.format
 in ${catalina.base}/conf/logging.properties. The format of this is the same as String.format with the arguments listed in the above table.