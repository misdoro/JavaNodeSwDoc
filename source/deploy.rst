VAMDC-TAP node deployment
==============================


Install Java application server
--------------------------------

Java implementation of VAMDC-TAP node software is intended to be run as a web application within Java application server like Apache Tomcat.
For installation instructions refer to the server documentation.


Deploy node software
----------------------

Deploying the Java implementation is a simple process that requires not that many steps.
If your database plugin is already throughly tested with the TAPValidator, everything should just work.

#.	Deploy a recent TAP-VAMDC-FRAMEWORK.war using your server default method.
	For Apache Tomcat it would require just to copy .war file into webapps directory with a desired name.

#.	It is unlikely that you would need to modify servlet configuration file, but just in case, it's located at:
	WEB-INF/web.xml

#.	Load the address http://$BASEURL/config to see the default configuration, adjust it as necessary and put 
	into tapservice.conf in WEB-INF/config. For the full description of all configuration parameters, consult :ref:`config`
	
#.	Copy your apache cayenne configuration xml to WEB-INF/config/cayenne/ (path can be adjusted in WEB-INF/web.xml)
	Database credentials can be changed in WEB-INF/config/cayenne/....driver.xml (filename is specific to your database)

#.	Copy your DAO jar and plugin jar (or it can be combined in single jar file) to WEB-INF/lib/ directory of your servlet.
	**!WARNING!** Don't try to copy it to webserver or java system library directory, 
	it won't work, you'll be getting strange ClassCast exceptions on every request.
	
#.	Restart application server or reload application to update the configuration.
	If you open the application root URL, you will get an index page with all relevant addresses.
	
#.	Check availability and capabilities if everything is fine.
	Try some test queries with TAPValidator
	
#.	**!WARNING!** Do a backup copy of all configuration files and .jar files that were put or adjusted somewhere into application folder, notably the node configuration files,
	they will be erased on every TAP-VAMDC-FRAMEWORK.war update
	