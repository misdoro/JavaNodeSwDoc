.. _plugin:

Node architecture
=========================

Java implementation of VAMDC-TAP node software is a web-service application implementing a RESTful interface according to VAMDC-TAP standard specification.

It is accessible using the HTTP protocol, with URLs like *http://node.name.tld/vamdc-tap/12_07/VOSI/capabilities* or 
*http://node.name.tld/vamdc-tap/12_07/TAP/sync?query=select%20species&lang=VSS2&format=XSAMS*.
Here *http://node.name.tld/vamdc-tap/12_07/* is the node base url, 
**/VOSI/capabilities** and **/TAP/sync** are endpoint addresses, and
*?query=select%20species&lang=VSS2&format=XSAMS* are HTTP request parameters.

For the complete list of endpoint addresses implemented please refer to section :ref:`endpoints`.


.. _requestflow:

Request processing
--------------------

During normal operation, node software receives *HTTP GET* and *HTTP HEAD* requests to the **/TAP/sync** 
endpoint. As a response to a request a valid XSAMS document and/or the HTTP response headers are sent. 

The figure below presents the request flow, *highlighting* the components involved in the request processing.

.. image:: img/queryProcess.png

During the request processing the few steps are performed.
Java VAMDC-TAP implementation (**Framework**) handles the steps that are common for all VAMDC-TAP nodes.
Node **Plugin** is responsible for the steps that are essentially node-specific.
To access the **database**, the Apache Cayenne object-relational mapping framework is used.

*	On the query reception, the **framework** asks the **plugin** for the list of supported **keywords** (Restrictables).

*	The **Framework** parses the incoming query. The validity of the query is checked. 
	A tree of objects is constructed, representing the logical structure of the query.
	For more details please refer to the :ref:`Query` section.

*	**Framework** asks **Plugin** to construct the document by calling the :ref:`databasePlug` 
    buildXSAMS(RequestInterface incomingRequest) method. All the query information is accessible via the 
    RequestInterface object passed as a parameter.

*	**Plugin** maps the incoming query logical tree into one or more **Database** queries, 
	as described in the :ref:`QueryMap` section.
	
*	**Plugin** queries the **database** and retreives **Apache Cayenne** data objects.
	**XSAMS** elements are built from those data objects and are attached to the XSAMS document tree. 
	See the :ref:`XSAMSGen` section for the details.
	
*	When the document tree is built, **Plugin** returns the control to the **Framework**.

*	The **Framework** does the final checks on the document tree and calculates the accurate metrics for the document.

*	The **Framework** converts the document tree into XML stream and sends it to the client.


.. _Interfaces:
Plugin and Framework Interfaces
---------------------------------

Interaction between the database plugin and the Java node software is performed through two interfaces:

*	:ref: `DatabasePlug`
	defined in the **org.vamdc.tapservice.api.DatabasePlugin** java interface and implemented by the node developer.
	Methods of this interface are called by the node software upon the arrival of a VAMDC-TAP request or during the initial startup and operation checks.

*	:ref: `RequestInterface`
	defined in the **org.vamdc.tapservice.api.RequestInterface** java interface and implemented in the web-service framework.
	This interface is available to the node plugin to get the request information and feed the XSAMS blocks constructed.


.. _DatabasePlug:

DatabasePlugin interface
++++++++++++++++++++++++++++

Each and every node plugin must implement the **org.vamdc.tapservice.api.DatabasePlugin** 
interface, defining the following methods:

*	**public abstract Collection<Restrictable> getRestrictables();**
	
	Must return a collection of **org.vamdc.dictionary.Restrictable** dictionary keys that are supported by the node.
	This method is called once per each request to the */TAP/sync* and */VOSI/capabilities* endpoints.
	
*	**public abstract void buildXSAMS (RequestInterface userRequest);**
	
	Build the XSAMS document tree corresponding to the incoming request. 
	Object implementing :ref:`RequestInterface`
	is passed as a parameter. No return is expected.
	This method is called every time the node software is receiving an *HTTP GET* request to the */TAP/sync?* endpoint.
	
	**WARNING!** Node plugin class is instantiated only once when the node is started,
	all calls to buildXSAMS should be thread-safe to handle concurrent requests correctly.
	
	Implementation details are covered in the :ref:`XSAMSGen` section.
	
*	**public abstract Map<Dictionary.HeaderMetrics,Integer> getMetrics(RequestInterface userRequest);**
	
	Get the metrics corresponding to a query. This method is called for every incoming HEAD request to the */TAP/sync?* endpoint.
	*RequestInterface userRequest* parameter is identical to the one passed to buildXSAMS method.
	This method should return a map of VAMDC-COUNT-* HTTP header names and their estimate values.
	For header names and meaning, see the [VAMDC-TAP]_ documentation section DataAccessProtocol.
	
	
*	**public abstract boolean isAvailable();**
	
	Do some really node-specific availability checks. This method is called
	periodically from the availability monitor. First call is initiated after the first request
	to the */VOSI/availability* service endpoint. Method may be used to temporary
	shutdown the node during the database maintenance, or to do some integrity checks on the database.
	Availability check interval may be set in the :ref:`config` option **selfcheck_interval**.
	
	**WARNING!** this method should not be used for doing periodic maintenance since it is never called before
	the first request to the */VOSI/availability* service endpoint.

	
.. _RequestInterface:

RequestInterface interface
+++++++++++++++++++++++++++++++

When the incoming request arrives to Java Node Software, methods of the :ref:`DatabasePlug` are called, 
as described earlier. Methods of that interface accept as a parameter an object
implementing the **org.vamdc.tapservice.api.RequestInterface**. That interface provides the access 
to the request information and the Java Node Software facilities.

Following methods are part of this interface:

*	**public abstract boolean isValid();**
	this method returns **true** if the incoming request is valid and should be processed.
	If the returned value is **false**, the request should be considered as invalid, no processing is required.
    Query string may be obtained by calling the getQueryString method and saved by the node plugin for the logging purposes.

*	**public abstract Query getQuery();**
	This method returns the base object of the QueryParser library, implementing the Query interface. 
    That interface is described in detail in the :ref:`query` section of this document. 

*	**public abstract LogicNode getQueryTree();**
    This is the shortcut method returning the root element of the logic tree corresponding to the incoming query.
	
*	**public abstract Collection<RestrictExpression> getQueryKeywords();**
	This is an another shortcut method, allowing to get a collection of the keywords used in the incoming query.
	The keywords relation logic is omitted.
	
	**WARNING!** This method should not be used as the main source of data for the query mapping since
	it completely looses the query relation logic. Imagine the query::
	
		SELECT * WHERE AtomSymbol='Ca' OR AtomSymbol='Fe'
		
	If this method is used for the query mapping, this query would produce the same result as the query::
	
		SELECT * WHERE AtomSymbol='Ca' AND AtomSymbol='Fe' 
		
	which is obviously incorrect.
	
	
*	**public abstract String getQueryString();**
	The shortcut method to get the incoming query string.

*	**public abstract boolean checkBranch(Requestable branch);**
	The shortcut method for the Query.checkBranch(),
	returns true if the result document is requested to contain a certain branch of XSAMS,
	specified by the **org.vamdc.dictionary.Requestable** name.
	
	This method should be called in all builders to verify if a certain branch should be built,
	before even executing or mapping the queries.
	
	The behaviour of the keywords is described in the VAMDC Dictionary documentation [VAMDCDict]_, 
	the section **Requestables**
	
*	**public abstract ObjectContext getCayenneContext();**
	Returns the Apache Cayenne object context. That is the main endpoint of the Cayenne ORM library.
	For more information on using the Apache Cayenne look in the sections :ref:`datamodel` and :ref:`QueryMap`.

	
*	**public abstract XSAMSManager getXsamsManager();**
	Get an instance of the XSAMS document manager. This manager contains several helper methods for building the XSAMS document.
	All XSAMS branches built by the node plugin should be attached to it. Check the :ref:`XSAMSGen` section for more details.

*	**public abstract void setLastModified(Date date);**
	
	Sets the last-modified header of the response. 
	This method may be called several times during the request processing.
	If called more than once, the last modification date is updated only if
	the subsequent date is newer than the value communicated before.


