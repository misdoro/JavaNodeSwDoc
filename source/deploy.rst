.. _deploy:

VAMDC-TAP node deployment
==============================


Install a Java application server
---------------------------------

The Java implementation of the VAMDC-TAP node software is provided as a web application archive (.war) 
and is intended to be run under a Java application server like Apache Tomcat.
For the server installation instructions refer to the server documentation.


Deploy the node software
-------------------------

Once the node plugin is tested with TAPValidator and working,
a few steps are needed to install(deploy) a VAMDC node using the Java Node software.

#.      Download the latest version of Java Node Software from the VAMDC site: http://www.vamdc.org/software/
	What you need is a web application archive vamdctap-webservice-xxx.war

#.	Deploy the downloaded .war file using your server default deploy method. 
	(see the server documentation).
	The contents of the .war archive is unpacked to *webapps/vamdctap.../* by the server.

#.	The Apache Cayenne configuration files need to be copied to WEB-INF/classes/
	subdirectory of the unpacked web application archive.
	The cayenne project file needs to be renamed into **cayenne-project.xml**
	Database credentials can be changed in WEB-INF/classes/cayenne-project.xml 
	or in dbcp.properties if you are using Apache DBCP for the database connection pooling.

#.	Load the address *http://$BASEURL/config* to see the default configuration.
	To alter the options, a key-value pairs should be saved in a 
	text file *WEB-INF/config/tapservice.conf*.
	For a full list of options and their desriptions please refer to the section :ref:`config`.
	
#.	The node plugin and DAO jar files should be copied to *WEB-INF/lib/* directory 
	of the deployed web application.
	**!WARNING!** Never try to copy those jars to the webserver or java system library paths.
	Such a configuration will never work, causing unexpected ClassCast exceptions.
	
#.	The application or server should be reloaded to update the configuration and load the libraries.
	
#.	Try to access the Capabilities and Availability endpoints of the service.
	They should be accessible through *http://$BASEURL/VOSI/capabilities* and 
	*http://$BASEURL/VOSI/availability*. Availability status should be
	**Service is currently available: true**	

#.	**!WARNING!** Do a backup copy of all configuration files and .jar files that were put or adjusted 
	in the deployed application folder, notably the node configuration files.
	All those files will be erased on every **vamdctap-webservice** war redeployment 
	and should be recopied in place every time the Java Node Software web application
	is updated or relocated.

#.	Once the deployment procedure is complete, the node operation 
	should be tested using the TAPValidator application.
	In the TAPValidator configuration the network mode should be chosen and the capabilities endpoint
	address of the node should be indicated.

	
.. _config:
	
service configuration file
----------------------------------------

Java VAMDC-TAP service is configured through a single file, containing a set of key-value pairs.

Once service is deployed, you may get the current configuration information 
by accessing the config endpoint:
*http://host.name:8080/tapservice/config*. Here *host.name:8080/tapservice*
is the url of the web application deployment.


Default configuration parameters
++++++++++++++++++++++++++++++++++++++++++

By default, the node parameters take the following values::

	limits_enable=false
	limit_states=-1
	limit_processes=-1
	selfcheck_interval=60
	database_plug_class=org.vamdc.database.tap.OutputBuilder
	dao_test_class=org.vamdc.database.dao.ClassName
	xsams_id_prefix=DBNAME
	baseurl=http://host.name:8080/tapservice#http://mirror.host/tapservice
	xml_prettyprint=false
	test_queries=select species where atomsymbol like '%';select * where inchikey='UGFAIRIUMAVXCW-UHFFFAOYSA-N'
	returnables=keyword1;keyword2;...
	processors=ivo://vamdc/processor_1#ivo://vamdc/processor2


Detailed parameters description
++++++++++++++++++++++++++++++++++++++++++

*	**limits_enable** = *(true|false)* enable output document element count limits

*	**limit_states** = N - limit maximum output states count in document to N, "-1" disables the limit.

*	**limit_processes** = N - limit maximum output processes count in document to N, "-1" disables the limit.


*	**selfcheck_interval** = N interval in seconds between the node availability self checks.
	First check is initiated upon the request to /VOSI/availability endpoint.
	Background checks allow the accurate tracking 
	of "upSince", "backAt" and "downAt" time attributes.


*	**database_plug_class** = *org.vamdc.database.tap.OutputBuilder*
	The name of the class implementing the *org.vamdc.tapservice.api*.**DatabasePlugin** interface.
	An instance of that class is created on the web application startup.
	Methods of this class are invoked to construct a response for the incoming request.

*	**dao_test_class** = *fully.qualified.apache.cayenne.dao.Object*
	The name of a Cayenne DAO class, used for the node availability checking.
	A table behind that class should have more than 10 records.
	This option may be omitted but in that case the availability endpoint will not work properly.

*	**xsams_id_prefix** = *DBNAME*
	Prefix for XSAMS library id generator org.vamdc.xsams.util.IDs, used to produce all XML 
	id/idRef references. Each node needs to have it's own prefix to maintain globall uniquiness
	of identifiers in XSAMS documents.

*	**baseurl** = *http://host.name:8080/tapservice*
	Base url used in VOSI/capabilities output, must contain globally accessible URL 
	pointing to the VAMDC-TAP service web application root. 
	Multiple addresses may be specified to indicate mirror nodes, separated by *#* symbol.
	The first address must point to the node itself.

*	**xml_prettyprint** = *(true|false)*
	Produce pretty-printed XML documents or output everything in a single line of text with no linefeeds.
	Defaults to false, enabling it increases the size of output XML document by ~20%

	
*	**test_queries** = *select species where atomsymbol like '%';*
	semicolon-separated list of valid test queries for the node.
	It must contain only valid queries that demonstrate the full functionality of the node.
	On the other hand such queries must produce compact documents, since those queries would be used 
	for periodic node testing.
	
*	**returnables** a semicolon-separated list of Returnable keywords that will be shown in the
	Capabilities endpoint output. May be left empty.

*	**processors** a list of IVOA identifiers of the preferred XSAMS processors.
	This list is used by the vamdc portal to suggest the processors.
	Processor identifiers may be obtained from the VAMDC registry.

Cayenne configuration using DBCP
------------------------------------

To avoid database connection time-out errors, Apache *commons-dbcp* library should be used in 
junction with Apache Cayenne. Configuration change for this case is simple and straight-forward.
**vamdctap-webservice** application archive already comes with bundled *commons-dbcp* jar.

* 	in the file cayenne-project.xml the *factory* attribute of the **node** element should be changed 
	to org.apache.cayenne.conf.DBCPDataSourceFactory.
	
* 	dbcp.properties configuration file should be put next to 
	the cayenne-project.xml in the classes directory.
	the following parameters should be put into the file::
	
	 cayenne.dbcp.driverClassName=com.mysql.jdbc.Driver
	 cayenne.dbcp.url=jdbc:mysql://hostname:port/databasename
	 cayenne.dbcp.username=databaseUserName
	 cayenne.dbcp.password=databasePassword

Other DBCP parameters may be adjusted, 
please consult http://commons.apache.org/dbcp/configuration.html for more information.



Database updates and cache
----------------------------

Java node software maintains its own database cache. 
If database fields are updated, this cache needs to be purged. 
To force node to purge its caches, request to $BASEURL/clear_cache resource needs to be sent.
The full URL will be http://host.name:8080/tapservice/clear_cache. 
Result document contains a single line of text, indicating the number 
of records that were contained in the cache.

This URL should be called every time the database contents is updated.
It may be accessed either manually, included as an iframe in the node 
database administration interface, or accessed by
a special script that tracks database modifications in some way.


Node mirroring
----------------------

The best way to set up node mirrors is to configure database replication and deploy
an instance of the node software on each mirror.

Deployment procedure of node software on master server and mirrors is the same.
The only difference will be the order of the addresses in the **baseurl** 
configuration parameter on each of the mirrors. 

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
	
	
Node registration
--------------------------

To update the node registration to include the mirror nodes,
two steps are needed:

*	In the configuration of the main node and all mirrors the
	**baseurl** parameter should be set up to include the node and mirrors, 
	separated with hash (#) symbol.
	On mirror nodes, the url of the mirror itself should be the first in that list.
	Java node web application should be reloaded to update the configuration.

*	Registry record of the node should be updated with new Capabilities including new mirror.


In any case for the registry update please contact the VAMDC registry 
maintainer via support@vamdc.org.


