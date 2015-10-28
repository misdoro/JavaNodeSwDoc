


.. _webservice:

Web-service description
=========================

Java implementation of VAMDC-TAP node software is a web-service application implementing a RESTful interface according to VAMDC-TAP standard specification.

It is accessible using the HTTP protocol, with URLs like *http://node.name.tld/vamdc-tap/12_07/VOSI/capabilities* or 
*http://node.name.tld/vamdc-tap/12_07/TAP/sync?query=select%20species&lang=VSS2&format=XSAMS*.
Here *http://node.name.tld/vamdc-tap/12_07/* is the node base url, 
**/VOSI/capabilities** and **/TAP/sync** are endpoint addresses, and
*?query=select%20species&lang=VSS2&format=XSAMS* are HTTP request parameters.

.. _endpoints:

Web-service endpoints
----------------------

Java implementation of VAMDC-TAP node software implements the following webservice endpoints:


*	/VOSI/capabilities
	Virtual Observatory Support Interface - service capabilities endpoint.
	Reports the service capabilities and available IVOA and VAMDC-TAP addresses, as well as their mirrors.
	Accept no parameters.

*	/VOSI/availability
	Virtual Observatory Support Interface - service availability endpoint.
	Reports the service availability and internal checks status.
	Accepts no parameters.

*	/TAP/sync
	VAMDC-TAP synchronous query interface, accepting HEAD and GET requests.
	Mandatory parameters are **query**, **lang** and **request**. Refer to the VAMDC-TAP standard documentation for more details.

*	/config
	Returns the web-service configuration options

*	/clear_cache
	Forces the purge of Apache Cayenne object caches. May be called on the database update to force the node software to reload the data from the database.