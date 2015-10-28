Preface
========


This document describes the implementation and installation of a VAMDC-TAP node using the Java Node Software. It is organized to follow the node development path.
Each chapter corresponds to an individual development task.

In the :ref:`intro`, a brief description of the components employed in the java node software implementation is given.

:ref:`plugin` chapter presents the architecture of the Java Node Software and gives the details on the interaction interfaces between the node software parts.

Setting up the development environment with the TAPValidator program is described in the :ref:`plugintest` chapter.

The first development task is to access the data in the database. Apache Cayenne library is used to perform this task. The process is briefly described in the
chapter :ref:`datamodel`.
Once the access to the data is established, one may proceed to building the XSAMS elements from the database objects, as described in the chapter :ref:`XSAMSGen`.
The last development task is to define the mapping between the incoming VSS2 queries and the internal database queries. 
The chapter :ref:`QueryHandling` describes that mapping in detail.

By this moment, node plugin should be capable of producing the XSAMS documents in response to incoming queries.
Final development task is to implement special queries used to estimate if the database contains data for a given VSS2 request. 
It is described in the :ref:`metrics` chapter.

The last chapter and last task is to install the node web service (:ref:`deploy`).