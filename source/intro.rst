.. _intro:

Introduction
=============

Creation of a separate implementation of the VAMDC-TAP node software in Java pursuied several goals:

* Insure that standards are complete and contain no implementation-specific elements

* Give end-users a choice of the implementation language

* Architectural disagreements with Python node software author


Used software
-----------------------------------------------

The following open-source libraries were used as Java node software components:

* JAXB RI for XML schema mapping and document output

* Apache Cayenne ORM framework for database access

* MySQL database (any relational database can be used)

* ANTLR generated query parser with slightly modified SQLite syntax

* Oracle Jersey JAX-RS implementation

* Apache Tomcat application server


VAMDC common components
-----------------------------------------------

Following components, developed within the VAMDC project are part of Java VAMDC node software implementation.

* Dictionaries of standard keywords, 
	used in a query;

* Query manipulation library, 
	providing object-oriented view on a query string;

* XSAMS helper library, 
	providing convenience methods for output XML generator implementation;

* Query mapper:
	reference implementation and examples of a mapper from VAMDC VSS2 queries to Cayenne queries


Node-specific components
-----------------------------

In Java node software each node installation requires creating two code blocks:

#. Database mapping classes with Apache Cayenne.
	This task is well described in Cayenne documentation [CAYENNE]. Fancy graphical tool is provided.
	
#. Node plugin, 
	responsible for query translation into internal queries 
	and building appropriate XSAMS tree from the database objects
	fetched using mapped queries.
	
This document is dedicated mostly to describe how to implement and deploy a node plugin.


Comparison with the python/django node software
----------------------------------------------------

This document won't be complete without the comparison with the Python/Django node software implementation.

Common features
++++++++++++++++++

* Work as a web application behind a web server

* Use object-relational mapping for database access

* Try to minimize the amount of node-specific code

* Node-specific part works as a plugin

Differencies
++++++++++++++

* Java implementation provides no XSAMS generator, node plugin needs to build XSAMS blocks.
	
	On a bad side, this results in a slightly bigger amount of node-specific code
	
	On a good side, it gives much more flexibility in document generation.
	
	For the task of implementing XSAMS generator, existing examples of KIDA, BASECOL and VALD can be used.
	
	Also, XSAMS schema itself is better documented than the "returnables" dictionary,
	and the tree structure of XSAMS is easier for understanding than a plain list of keywords, representing
	a subset of that tree.
	
* Java implementation doesn't (yet) support document streaming, 
	it requires to have a whole document tree to be built in memory
	and then streamed in a response.
	
	Thus increasing memory footprint, it allows to not strictly follow the document generation order,
	i.e. export some species and states, then export processes, while exporting some more species and states.
	
* Java implementation doesn't provide an import tool from ASCII files into relational database
	
	Well, there was no need for such a tool. You may use Python one.
	