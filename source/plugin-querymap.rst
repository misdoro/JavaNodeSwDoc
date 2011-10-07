.. _QueryMap:

VSS query recognition and mapping
=====================================

Obviously, each node has it's own database structure.
In order to fetch data requested by the NodeSoftware client,
incoming query needs to be mapped into one or multiple queries to the node database.
Java Node software tries to make the process of query mapping as simple and sophisticated as possible.


Query keywords tree
-----------------------

First, as Java node software receives a query, it validates it and parses into a tree of objects.
Intermediate nodes of that tree are representing boolean relations and leafs keep the information about
the query keywords, comparison operator and values.

For example, query::

	SELECT * WHERE reactant1.AtomSymbol = 'C' AND reactant2.AtomSymbol = 'O' 
	AND (CollisionCode='inel' or CollisionCode='elas')
	
would map into a tree

.. image:: img/queryTree.png


Tree objects
---------------------

Following objects are representing the query tree

Query
++++++++++++

Main interface of the query parser library,
provides access to the query tree and few utility methods.

*	**public LogicNode getRestrictsTree()**
	is the main method, returning the root of the query tree.

	Accompanying are two methods, **getFilteredTree** and **getPrefixedTree**, returning subsets of tree.

*	**public LogicNode getFilteredTree(Collection<Restrictable> allowedKeywords)**
	returns a subtree containing only keywords listed in collection passed as a parameter.

*	**public LogicNode getPrefixedTree(VSSPrefix prefix, int index)**
	returns a subtree containing only keywords having the defined prefix and index.
	If *null* is passed as a prefix, returned tree would only contain nodes without any prefix.
	
*	**public Collection<Prefix> getPrefixes()**
	returns a collection of prefixes present in the query

*	**public List<RestrictExpression> getRestrictsList()**
	is the most dummy method, returning a list of all keywords specified in the query.
	Using that list as a main source for query mapping is discourages since it leads to the loss of logic.
	





LogicNode
+++++++++++++++++

LogicNode interface represents a node of the query tree.
**getOperator()** method provides access to the node operator,
**getValues()** returns a collection of child nodes.

For certain operators like **NOT**, **getValue()** method also makes sense, returning a single
child element.


RestrictExpression
+++++++++++++++++++++

RestrictExpression elements are the leafs of the logic tree, representing the actual query restriction keywords.

Same as the LogicNode, RestrictExpression provides **getOperator()**, **getValues()** and **getValue()** methods,
plus

*	**public Prefix getPrefix()** method, returning this expression prefix

*	**public Restrictable getColumn()** method, returning a restriction keyword from the dictionary
	for this expression


Prefix
+++++++++++++

Prefix is a simple class, keeping **VSSPrefix** from dicrionary
and integer index of the prefix.

*	**int getIndex()** method provides access to index, and

*	**VSSPrefix getPrefix()** gives access to the prefix name.


Query mapping scenarios
-------------------------

To obtain Apache Cayenne Expression object, several mapping scenarios are provided, plus plugin developer 
is free to implement his own one.


