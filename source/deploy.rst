.. _deploy:

VAMDC-TAP node deployment
==============================


Install Java application server
--------------------------------

Java implementation of VAMDC-TAP node software is intended to be run as a web application within Java application server like Apache Tomcat.
For installation instructions refer to the server documentation.


Deploy node software
----------------------

Deploying the Java implementation is a simple process that requires not that many steps.
If your database plugin is already throughly tested with the VAMDC-TAP Validator, everything should just work.

#.	Deploy a recent **vamdctap-webservice** war using your server default method.
	For Apache Tomcat it would require just to copy .war file into webapps directory with a desired 
	webservice name.

#.	It is unlikely that you would need to modify servlet configuration file, but just in case, it's located at:
	WEB-INF/web.xml

#.	Load the address *http://$BASEURL/config* to see the default configuration, adjust it as necessary and put 
	into tapservice.conf in WEB-INF/config. For the full description of all configuration parameters, 
	consult :ref:`config` section of this document;
	
#.	Copy your Apache Cayenne configuration xml to WEB-INF/config/cayenne/ (path can be adjusted in WEB-INF/web.xml)
	Database credentials can be changed in WEB-INF/config/cayenne/....driver.xml (filename is specific to your database)

#.	Copy your DAO jar and plugin jar (or it can be combined in single jar file) to WEB-INF/lib/ directory of your servlet.
	**!WARNING!** Don't try to copy it to webserver or java system library directory, 
	it won't work, you'll be getting strange ClassCast exceptions on every request.
	
#.	Restart application server or reload application to update the configuration.
	If you open the application root URL, you will get an index page with all relevant addresses.
	
#.	Check availability and capabilities if everything is fine.
	Try some test queries with VAMDC-TAP Validator
	
#.	**!WARNING!** Do a backup copy of all configuration files and .jar files that were put or adjusted 
	somewhere in the deployed application folder, notably the node configuration files.
	All those files will be erased on every **vamdctap-webservice** war update

	
.. _config:
	
VAMDC-TAP service configuration file
----------------------------------------

Java VAMDC-TAP service is configured through a single file, containing a set of key-value pairs.

Once service is deployed, you may get configuration info through 
http://host.name:8080/tapservice/config URL, where host.name:8080/tapservice is the url to your web application deployment.
If service is not configured, only the default configuration set is presented.

Default configuration parameters
++++++++++++++++++++++++++++++++++++++++++

::

	force_sources=true
	limits_enable=false
	limit_states=-1
	selfcheck_interval=60
	dao_test_class=org.vamdc.database.dao.ClassName
	test_queries=select species where atomsymbol like '%';
	xsams_id_prefix=DBNAME
	baseurl=http://host.name:8080/tapservice
	database_plug_class=org.vamdc.database.builders.ClassName
	xml_prettyprint=false
	limit_processes=-1


Detailed parameters description
++++++++++++++++++++++++++++++++++++++++++

*	**limits_enable** = *(true|false)* enable output document element count limits

*	**limit_states** = N - limit maximum output states count in document to N, "-1" disables the limit.

*	**limit_processes** = N - limit maximum output processes count in document to N, "-1" disables the limit.


*	**selfcheck_interval** = N interval in seconds between service availability self checks.
	First check is initiated after the first request to /VOSI/availability endpoint.
	Background checks allow more accurate tracking of "upSince", "backAt" and "downAt" time attributes.


*	**database_plug_class** = *org.vamdc.database.builders.ClassName*
	Class name for builder implementing org.vamdc.tapservice.api.DatabasePlugin interface
	It is instantiated on tapservice startup and all communication between framework and builders go through it.

*	**dao_test_class** = *fully.qualified.apache.cayenne.dao.Object*
	class name for self availability checking,
	it must be the table with more than 10 records.
	May be omitted, but then availability endpoint won't work properly.

*	**xsams_id_prefix** = *DBNAME*
	Prefix for XSAMS library id generator org.vamdc.xsams.util.IDs, used to produce all XML 
	id/idRef references. Each node needs to have it's own prefix to maintain globall uniquiness
	of identifiers in XSAMS documents.

*	**baseurl** = *http://host.name:8080/tapservice*
	Base url used in VOSI/capabilities output, must contain globally accessible URL pointing to the VAMDC-TAP service
	web application root. Multiple addresses may be specified to indicate mirror nodes, separated by *#* symbol.

*	**xml_prettyprint** = *(true|false)*
	Produce pretty-printed XML documents or output everything in a single line of text with no linefeeds.
	Defaults to false, enabling it increases the size of output XML document by ~20%

	
*	**test_queries** = *select species where atomsymbol like '%';*
	semicolon-separated list of valid test queries for the node.
	It must contain only valid queries that demonstrate the full functionality of the node.
	On the other hand such queries must produce compact documents, since those queries would be used 
	for periodic node testing.
	


Cayenne configuration using DBCP
------------------------------------

To avoid database connection time-out errors, Apache *commons-dbcp* library should be used in 
junction with Apache Cayenne. Configuration change for this case is simple and straight-forward.
**vamdctap-webservice** application archive already comes with bundled *commons-dbcp* jar.

* First, in cayenne.xml **node** element *factory* attribute have to be changed for org.apache.cayenne.conf.DBCPDataSourceFactory
	and *datasource* attribute should indicate a file name that will contain key-value pairs of dbcp configuration;
	
* Second, dbcp configuration file should be put in *cayenne* configuration directory
	with the name same as specified in *datasource* attribute and the following parameters:
	
	- cayenne.dbcp.driverClassName=com.mysql.jdbc.Driver
	- cayenne.dbcp.url=jdbc:mysql://hostname:port/databasename
	- cayenne.dbcp.username=databaseUserName
	- cayenne.dbcp.password=databasePassword

	Other DBCP parameters also may be adjusted, see http://commons.apache.org/dbcp/configuration.html for more information.



Database updates and cache
----------------------------

Java node software maintains it's own database cache. If database fields are updated, this cache needs to be purged. 
To force node to purge it's caches, request to  /clear_cache resource needs to be sent.
The full URL will be http://host.name:8080/tapservice/clear_cache. 
As a result, single text line will be sent indicating the number of records that were contained in the cache.
This URL may be accessed either manually, included as a frame in node database administration panel or accessed by
a special script that tracks database modifications in some way.


Node mirroring
----------------------

The best way to set up node mirrors is to configure database replication on each mirror and deploy
multiple instances of node software on mirror servers, each of them using own mysql installation. 
Deployment procedure of node software on master server and mirrors is the same.

Mysql master server configuration
+++++++++++++++++++++++++++++++++++++

On master server we need to enable binary logging in my.cnf::

	[mysqld]:
	server-id = 1
	log-bin = /var/lib/mysql/mysql-bin
	replicate-do-db = databasename
	bind-address = 0.0.0.0
	
and add user with replication privileges in mysql console::
	
	mysql@master> GRANT replication slave ON "databasename".* TO "replication"@"mirror.ip.or.hostname" 
		IDENTIFIED BY "password";

mysql service needs to be restarted after that.

After server restart we need to create a database dump:

	mysql@master> FLUSH TABLES WITH READ LOCK;
	mysql@master> SET GLOBAL read_only = ON;
	mysql@master> SHOW MASTER STATUS\G
	
	#mysqldump -u root -p databasename  | bzip2 -9 -c - > database_dump.sql.bz2

	mysql@master> SET GLOBAL read_only = OFF;

Here we need to note **File** and **Position** values from the mysql command *Show Master Status*.



Mysql slave configuration
+++++++++++++++++++++++++++

On a slave (replica) server we need to set up the following things:


import mysql dump for database::

	#bzip2 -d -c database_dump.sql.bz2 | mysql -u root -p databasename

In my.cnf::

	[mysqld]:
	server-id = 2
	relay-log = /var/lib/mysql/mysql-relay-bin
	relay-log-index = /var/lib/mysql/mysql-relay-bin.index
	replicate-do-db = testdb
	
restart mysql service and start replication::
	
	mysql@replica> CHANGE MASTER TO MASTER_HOST = "master.ip.or.host", MASTER_USER = "replication", 
	MASTER_PASSWORD = "password", MASTER_LOG_FILE = "mysql-bin.000003", MASTER_LOG_POS = 98;
	mysql@replica> start slave;

where MASTER_LOG_FILE and MASTER_LOG_POS parameters we take from the *SHOW MASTER STATUS \G* 
mysql command on master server.

Then we may see the slave server status by issuing mysql command::
	
	mysql@replica> SHOW SLAVE STATUS\G
	
Following status parameters are important and indicated values show that replication is working properly:

*	**Slave_IO_State**: Waiting for master to send event
*	**Slave_IO_Running**: Yes
*	**Slave_SQL_Running**: Yes
*	**Seconds_Behind_Master**: 0

Changing **Read_Master_Log_Pos** parameter value may indicate that cache of node software on the mirror 
needs to be purged. A script can be set up to track this parameter, if no other mean of cache invalidation is
used.

In detail mysql replication is described in the official manual:
	http://dev.mysql.com/doc/refman/5.5/en/replication.html
	
	
Node mirror registration
++++++++++++++++++++++++++

Procedure for registering a new mirror is very simple:
In configuration of main node and all mirrors another url needs to be added to the end of 
**baseurl** parameter, separated with hash (#) symbol.

After reload of master node web application, registry needs to be updated with new Capabilities including new mirror.
For registry update, please contact VAMDC registry maintainer via support@vamdc.org.

