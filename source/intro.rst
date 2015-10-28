



.. _intro:


Introduction
=============

Java implementation of the VAMDC-TAP node software was created in parallel with the Python version to pursue several objectives:

*	Insure that standards are complete and contain no implementation-specific elements

*	Give a choice of an implementation to node maintainers

*	Employ the schema-based DOM XML generator to ensure the document validity.
	It presents an alternative approach to the keywords dictionary as used in Python/Django node software - see below the :ref:`diff` section


Used software
-----------------------------------------------

The following open-source libraries and components are used:

* JAXB Reference Implementation (RI) for XML schema mapping and XSAMS document output

* Apache Cayenne ORM framework for database access

* MySQL database (any relational database can be used via Apache Cayenne)

* ANTLR generated query parser with slightly modified SQLite syntax to transform the incoming query into the tree of objects

* Oracle Jersey JAX-RS implementation to support the VAMDC-TAP webservice interfaces

* Apache Tomcat application server to run the Java VAMDC-TAP web application


VAMDC common components
-----------------------------------------------

Additional libraries were developed within the VAMDC project are part of Java VAMDC-TAP node software.

* Dictionaries of standard keywords, 
	used in a query;

* Query parsing library, 
	providing object-oriented view on a query string;

* Query mapping library,
	providing basic support for mapping of incoming queries to the node database queries using Apache Cayenne objects.

* XSAMS helper library, 
	providing convenience methods for output XML generator implementation;

* Web application, 
	.war archive integrating all libraries

* TAPValidator,
	VAMDC-TAP service validation tool that may be used for node plugin testing on all phases of development.


Node-specific components
-----------------------------

In Java node software each node installation requires creating two libraries:

#. Database Access Objects using Apache Cayenne.
	This task is well described in the Apache Cayenne documentation [CAYDOC]_ . 
	A graphical tool is provided allowing to define the database structure and relations.
	Java classes corresponding to table entities are generated using this tool.
	
#. Node plugin,
	A piece of software that implements the node plugin interfaces and integrates with Java VAMDC Node Software application.
	Node plugin is responsible for a query translation into the database-specific queries and for building the appropriate XSAMS tree
	from the fetched database access objects.




Comparison with the python/django node software
----------------------------------------------------

The paragraph provides a comparison between the Java-Implementation and
the Python/Django node software

Common features
++++++++++++++++++

Both Java and Python node software implementations

* Work as a web application behind a web server

* Use object-relational mapping to access the database

* Provide the implementation of the parts that are common for all nodes

* Node-specific part works as a plugin, that means that no modification to the common components is needed during the node-specific part development.

.. _diff:

Differencies
++++++++++++++

* The main architectural difference
	between the the Java implementation and the Python/Django one is the XML generator.
	
	Java version uses Document Object Model (DOM) mapping of the XML, Java node plugin needs to build XSAMS blocks
	as trees of objects.
	
	Python/Django version provides the XML generator with a defined and limited set of loops and anchors("returnables").
	Node developer needs to study not only the XSAMS documentation, but also to look through a plane list of keywords to correctly pin the data
	to the XML document structure.
	
	
	The use of DOM XML mapping has some advantages and disadvantages:
	
	+	On a positive side, it gives more flexibility for the document generation. Any attribute of the output document can be accessed
and set without the XML generator modification.
	
	+	The output document is kept error-free because of the compile-time type checks and runtime XML special symbols filtering.
	
	+	XML DOM mapping provided is complete: even if node developer wishes to put the data in
		a rarely used element of XSAMS, he can do it without the need to output XML blocks as the plain text.
	
	-	On a bad side, the task of building a document tree requires an additional amount of node-specific code.
		Parts of XSAMS-extra library provide helper methods to simplify this task.

	-	Memory consumption is higher due to the need to keep the whole document tree in memory.

	-	Document output is started after the construction of the XML tree, an additional delay is introduced, compared to the immediate streaming
		of the output by the Python/Django node software.
	
	
	For the task of implementing XSAMS blocks builder, existing builders of KIDA, BASECOL and VALD may be used as examples.
	
	
* Java implementation does not support document streaming.
	The whole document tree is built in memory before producing the output XSAMS response.
	
	This approach allows to generate the document in the arbitrary order,
	i.e. export some species and states, then export processes, while exporting some more species and states.
	
	
* Java implementation does not provide any import tool from ASCII files into a relational database
	
	The node developer is himself responsible for creating and maintaining the node database structure and administration tool.

* Java implementation provides a sophisticated query parsing and mapping support
	

Node implementation
---------------------

Implementing a node with the Java Node Software would require the following steps:

*	The database model and classes should be created, as described in the :ref:`datamodel` section.

	After completing this step you will be able to access your database in a convenient way
	from any Java software you develop. For the details, see the Apache Cayenne documentation. [CAYDOC]_

*	The development environment for the plugin should be deployed. 
	The query process and the interaction of the node and the plugin should be understood.
	See the :ref:`plugin` section for the details.

*	XSAMS tree builders should be created, as described in the :ref:`XSAMSGen` section.
	This work should be performed in collaboration with a person responsible for the scientific content of the
	database to establish the correspondance of the XSAMS elements and the database content.

	At this step the node plugin may be tested according to the procedure described in the section :ref:`plugintest`.
	XSAMS generation should be verified and all the validation errors should be eliminated at this step.

*	The supported restrictables and corresponding mappings should be defined, as described in the :ref:`QueryHandling` section.

	Once this step is accomplished, it becomes possible to send different queries to the node.
	It should be checked if all the produced XSAMS documents are valid against the schema.
	
*	The last development step would be to implement the query metrics that are used to estimate quickly if the node has the data
	for a particular query or not. See the :ref:`metrics` section for the implementation details.
	
*	Once the node plugin is tested and working, it may be deployed on the web server, as described in the :ref:`deploy` section.
	The node should be tested again with the VAMDC-TAP Validator working in the network mode.
	