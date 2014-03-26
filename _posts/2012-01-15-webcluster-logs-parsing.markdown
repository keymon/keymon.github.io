---
author: keymon
comments: true
date: 2012-01-15 04:30:56+00:00
layout: post
slug: webcluster-logs-parsing
title: 'New job assessment: Webcluster logs parsing'
wordpress_id: 309
categories:
- Misc
- monitoring
- script
- sysadmin
- Technical
---

These is one of the proposed solutions for the job assessment [commented in a previous post](http://keymon.wordpress.com/2012/01/15/some-posts-fro…job-assessment).


> _Provide a design which is able to parse Apache access-logs so we can generate an overview of which IP has visited a specific link. This design needs to be usable for a 500+ node webcluster. Please provide your configs/possible scripts and explain your choices and how scalable they are._


I will consider these requirements:



	
  * It is not critical to register all the log entries. It is no needed ensure that all the web hits are registered.

	
  * No control on duplicated log entries. It is not needed
to check that the log entries had been already loaded.

	
  * It is also needed to propose a mechanism to gather the logs from the webservers.

	
  * It must be scalable.

	
  * It is a plus to make it flexible to allow further different analysis.


The problems to be solved are log storage and log gathering, but the main bottleneck will be the storage.

One realizes that the best option is a noSQL database due to
the characteristics of the data to process (log entries):

	
  * Time ordered entries

	
  * no duplicates

	
  * need of fast insertion

	
  * fixed fields

	
  * no data relation or conceptual integrity

	
  * need to be rotated (old entries removed)

	
  * etc...


So, I will propose the usage of MongoDB [1] ([http://www.mongodb.org/](http://www.mongodb.org/)), that fits the requirements:



	
  * It is fast, both at inserting and querying.

	
  * Scales horizontally without disruption (is initially proper configured).

	
  * Supports replication and High Availability.

	
  * Well known solution. Commercial support if needed.

	
  * Python bindings (pyMongo)










[1]


**Note**: I will not enter in details of a MongoDB scalable HA architecture.
See [the quick start guide](http://www.mongodb.org/display/DOCS/Quickstart+Unix)
to setup a single node and the [documentation](http://www.mongodb.org/display/DOCS/Admin+Zone)
for architecture examples.




To parse the logs and store them in MongoDB, I will propose a
simple python script: [accesslog_parse_mongo.py](https://gist.github.com/1614315) that:



	
  * Setup a direct MongoDB connection.

	
  * Read the access log from standard input.

	
  * Parse the logs and store all the fields, including: client_ip, url, referer, status code, timestamp, timezone...

	
  * I do not set any indexes in the NoSQL db. Indexes could be
created on _url_ or _client_ip_ fields, but not having indexes allows faster
insertions, that is the objective. The reads are very uncommon and performed
in batch processes.

	
  * Notice that it should be improved to be more reliable. For instance, it
does not check for errors (DB failures, etc.). It could buffer entries in case of DB failure.

	
  * A second script called [example_query_accesslog.py](https://gist.github.com/1614319) queries the DB and prints the access. It gets an optional argument, the relative URL.


To feed the DB with the logs from the webservers, some solutions could be:

	
  * 


_Copy the log files with a scheduled task_ via SSH or similar, then process them with
accesslog_parse_mongo.py in a centralized server (or cluster of servers).




	
    * Pros: Logs are centralized. Only a set of servers access to MongoDB.
System can be stopped as needed.

	
    * Cons: Needs extra programming to get the logs.
No realtime data.




	
  * 


_Use a centralized syslog service_, like [syslog-ng](http://www.balabit.com/network-security/syslog-ng/)
(can be balanced and configured in HA),
and setup all the webservers to send the logs via syslog
(see [this article](http://oreilly.com/pub/a/sysadmin/2006/10/12/httpd-syslog.html)).


In the log server, we can process resulting files with a batch process or send all the messages to accesslog_parse_mongo.py. For instance, the configuration for syslog-ng:

    
    destination d_prog { program("/apath/accesslog_parse_mongo.py"
                                  template(“$MSGONLY\n”)
                                  template-escape(no)); };



	
    * Pros: Centralized logs. No extra programming. Realtime data.
Use of existent infrastructure (syslog). Only a set of servers access to MongoDB.

	
    * Cons: Some logs entries can be dropped. Can not be stopped, if not log entries will be lost.




	
  * 


_Pipe the webserver logs directly to the script_, accesslog_parse_mongo.py.
In Apache configuration:




    
    CustomLog "|/apath/accesslog_parse_mongo.py" combined



	
    * Pros: Easy to implement. No extra programming or infrastructure. Realtime data.

	
    * Cons: Some logs entries can be dropped. It can not be stopped or log entries will be lost.
The script should be improved to make it more reliable.





These is one of the proposed solutions for the job assessment [commented in a previous post](http://keymon.wordpress.com/2012/01/15/some-posts-fro…job-assessment).
