.. _config:

VAMDC-TAP service configuration file
=======================================

Java VAMDC-TAP service is configured through a single file, containing a set of key-value pairs.

Once service is deployed, you may get configuration info through 
http://host.name:8080/tapservice/config URL, where host.name:8080/tapservice is the url to your web application deployment.
If service is not configured, only the default configuration set is presented.

Default configuration parameters
-----------------------------------

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
------------------------------------

*	**force_sources** = *(true|false)* force tapservice to always output source information.
	If set to true, requestInterface.checkBranch is always true for Requestable.Sources,
	so when XSAMS builder checks for returnables it will be forced to always return sources 
	disregarding the user query.


*	**limits_enable** = *(true|false)* enable output document element count limits

*	**limit_states** = N - limit maximum output states count in document to N, "-1" disables the limit.

*	**limit_processes** = N - limit maximum output processes count in document to N, "-1" disables the limit.


*	**selfcheck_interval** = N interval in seconds between service availability self checks.
	First check is initiated after the first request to /VOSI/availability endpoint.
	Background checks allow more accurate tracking of "upSince", "backAt" and "downAt" time attributes.


*	**database_plug_class** = *org.vamdc.database.builders.ClassName*
	Class name for builder implementing org.vamdc.tapservice.api.DatabasePlug interface
	It is instantiated on tapservice startup and all communication between framework and builders go through it.

*	**dao_test_class** = *fully.qualified.apache.cayenne.dao.Object*
	class name for self availability checking,
	it must be the table with more than 10 records.
	May be omitted, but then availability endpoint won't work properly.

*	**xsams_id_prefix** = *DBNAME*
	Prefix for XSAMS library id generator org.vamdc.xsams.util.IDs
	Used for example to produce idRef's and stateId's
	Helps to create globally unique identifiers in XSAMS documents.

*	**baseurl** = *http://host.name:8080/tapservice*
	Base url used in VOSI/capabilities output, must contain globally accessible URL pointing at the VAMDC-TAP service
	web application root.


*	**xml_prettyprint** = *(true|false)*
	Produce pretty-printed XML documents or output everything in a single line of text with no linefeeds.
	Defaults to false, enabling it increases the size of output XML document by ~20%

	
*	**test_queries** = *select species where atomsymbol like '%';*
	semicolon-separated list of valid test queries for the node.
	It must contain only valid queries that demonstrate the full functionality of the node
	and on the other hand produce not too much output, since those queries would be used for periodic node testing.
	
	