.. _QueryHandling:

VSS query parsing and mapping
=====================================

Vamdc nodes are queried using a common SQL-like keyword-based query language.
A typical query looks like::
  
  SELECT * WHERE (keyword1=value or keyword2=value2) and keyword3='value3'

where the keywords correspond to the different parts of the XSAMS document
and are defined in the VAMDC Dictionary [VAMDCDict]_ as Restrictable keywords.
Since every VAMDC node has a specific database structure, 
the incoming query keywords need to be mapped to 
the database fields in order to fetch the data corresponding to the query.
Java Node software tries to make the process of query mapping as simple and sophisticated as possible.


Query keywords tree
-----------------------

As a first step, on query reception the Java Node Software parses the query into a tree of objects
using the **vamdctap-queryparser** library.
Leaf nodes of that tree of objects represent the individual keyword expressions.
The root node and the branches are reflecting the boolean relations between the leaves.
For instance, a query::

	SELECT * WHERE reactant1.AtomSymbol = 'C' AND reactant2.AtomSymbol = 'O' 
	AND (CollisionCode='inel' or CollisionCode='elas')
	
would map into the tree

.. image:: img/queryTree.png


QueryParser library interface
-----------------------------

The QueryParser libary provides an interface *org.vamdc.tapservice.vss2.* **Query**
providing several methods to access the incoming query elements.

.. _query:

Query
++++++++++++

The methods to access the incoming query elements are:


*	**public LogicNode getRestrictsTree()**
	is the main method, returning the root of the query tree.

	Accompanying are two methods, **getFilteredTree** and **getPrefixedTree**, returning the subsets.

*	**public LogicNode getFilteredTree(Collection<Restrictable> allowedKeywords)**
	returns a subtree containing only the keywords present in the collection 
	that is passed as a parameter.

*	**public LogicNode getPrefixedTree(VSSPrefix prefix, int index)**
	returns a subtree containing only the keywords having the prefix and index that are passed to the method.
	If *null* is passed as a prefix, returned tree would only contain nodes that have a *null* prefix.
	
*	**public Collection<Prefix> getPrefixes()**
	returns a collection of prefixes that are present in the query

*	**public List<RestrictExpression> getRestrictsList()**
	returns a list of all the keywords that are present in the query.
	The logic of relations between the elements is lost.

LogicNode
+++++++++++++++++

The *LogicNode* interface represents a node of the query tree.
**getOperator()** method provides access to the node operator (**OR**, **AND**, **NOT**),
**getValues()** returns a collection of child nodes.

For the single-value operators like **NOT**, the **getValue()** method may be used to obtain
the child element directly.


RestrictExpression
+++++++++++++++++++++

The leaf nodes of the logic tree are implementing the RestrictExpression interface, 
representing the query expressions.

The *RestrictExpression* interface is the extension of the *LogicNode* interface
providing the **getOperator()**, **getValues()** and **getValue()** methods, 
as well as few additional methods:

*	**public Prefix getPrefix()** a method returning the prefix used in the expression.

*	**public Restrictable getColumn()** a method returning the Restrictable keyword 
	used by the expression.


Prefix
+++++++++++++

Prefix is a simple class, keeping the **VSSPrefix** keyword of the dictionary, 
plus an integer index of the prefix.

*	**int getIndex()** method provides access to the index, and

*	**VSSPrefix getPrefix()** gives access to the prefix name.

.. raw:: latex

    \newpage

Filtering logic
----------------

The algorithm implemented for the **getFilteredTree()** and **getPrefixedTree()** methods of the
:ref:`query` interface works the following way:
It removes the irrelevant RestrictExpression objects from LogicNodes,
than removes the Nodes of the tree that have no child expression elements.

For example, in the case of filtering by the prefix:

*	Original query::

		select ALL where reactant1.AtomSymbol='C' and reactant1.AtomIonCharge=1 
		and reactant2.AtomSymbol='H' and reactant2.AtomIonCharge=-1 and EnvironmentTemperature > 100

*	Effective query for getPrefixedTree(VSSPrefix.REACTANT, 1)::

		select ALL where reactant1.AtomSymbol='C' and reactant1.AtomIonCharge=1

*	Effective query for getPrefixedTree(VSSPrefix.REACTANT, 2)::

		select ALL where reactant2.AtomSymbol='H' and reactant2.AtomIonCharge=-1
	
*	Effective query for getPrefixedTree(null, 0)::

		select ALL where EnvironmentTemperature > 100
	
*	Effective query for getFilteredTree(Collection<Restrictable>{AtomSymbol}) with a collection containing only the AtomSymbol element::

		select ALL where reactant1.AtomSymbol='C' and reactant2.AtomSymbol='H'



.. _QueryMap:

Query mapping scenarios
-------------------------

To perform queries on the actual database, the incoming VAMDC-TAP query 
needs to be adapted to the database structure. In the case of Apache Cayenne,
*org.apache.cayenne.exp*.**Expression** object needs to be constructed,
representing the query logic relative to the database primary table.

In this chapter the way to construct the query expressions from the incoming query is described.

Mapping of the LogicTree
+++++++++++++++++++++++++

Mapping of the LogicTree nodes is simple with one-to-one mapping of **AND**, **OR** and **NOT** operators to
Cayenne Expression.andExp(), Expression.orExp(), Expression.notExp(), see the Cayenne Javadoc [CAYJAVADOC]_.

Usable example of such mapper is provided in *org.vamdc.tapservice.query.QueryMapper* class (vamdctap-querymapper library),
that is bundled both with the TAPValidator and the node software.

Mapping of RestrictExpression elements
++++++++++++++++++++++++++++++++++++++++

Mapping of the leaf nodes of the query tree, represented by the RestrictExpression elements,
may be a bit more tricky. A lot of the information is contained in leaf nodes 
that should be mapped into the query logic:

*	Keyword prefix
*	Prefix index
*	VAMDC dictionary Restrictable keyword
*	comparison operator
*	value or a set of values

VAMDC Restrictable keyword may correspond to one or several database fields.
For example, let us have a database where all the species: 
particles, atoms and molecules are stored in a single table, 
where the inchikeys are defined for atoms and molecules, and there is a field indicating the species type.
In this case the **MoleculeInchiKey** keyword in a query will 
map to two internal **Expression** constraints: 

a constraint on the inchikey field of the table,

and a constraint indicating that the species is actually a molecule.

To correctly handle such a keyword we will need to join two Cayenne Expressions with the Expression.andExp 
operator, then add them to the mapped query tree.

Prefix and prefix index may also impose a check for a certain field, like if element 
is a reactant or product in a chemical reaction.

To handle the prefixes, it is possible to loop through the prefixes present in the query
by using **Query.getPrefixes()** method.
The second step would be to filter the incoming query tree 
by the prefix using the **Query.getPrefixedTree(...)** method, than map it to Cayenne Expressions,
and finally join the obtained expressions with the Expression.andExp() method.

Query Mapping Library
--------------------------

As a part of Java node software, a Query Mapper implementation is provided.
It is able to map incoming query trees into cayenne Expression objects.
Query Mapper implementation is a part of **vamdctap-querymapper** library,
represented by two interfaces and two generic implementations within a package
*org.vamdc.tapservice.querymapper*

	
*	**KeywordMapper** interface defining an interface of RestrictExpression mapper;
*	**KeywordMapperImpl** generic implementation, providing one-to-one mapping of
	Restrictable keywords to database fields without value transformation.
	In many cases node plugin may use extensions of this class, implementing value translation,
	to-many fields mapping or prefix-conditional mapping.

*	**QueryMapper** interface defining the library main interface;
*	**QueryMapperImpl** generic implementation, keeping references to KeywordMappers 
	and responsible for mapping parsed query trees to Cayenne Expressions. Boolean logic operations
	between nodes are translated one-to-one with Cayenne andExp, orExp and notExp, KeywordMappers are
	called for each RestrictExpression encountered.

Using QueryMapper library
++++++++++++++++++++++++++++++++++
From the plugin side work with the mapper library is performed the following way:

*	In some class we initialize a static variable QueryMapper,
	in constructor adding keyword mappers for each keyword supported by the node::


		public final static QueryMapper queryMapper= new QueryMapperImpl(){{
			this.addMapper(
					new KeywordMapperImpl(Restrictable.IonCharge)
					.addNewPath("symelementRel.elementRel.charge")
					.addNewPath("partyRel.elementRel.charge")
					);
		}};
	
	Here subsequent calls to **addNewPath** method define cayenne relations path
	originating from different primary tables, both used for mapping.
	The first call is for species query, the second for processes.

*	Own extensions of KeywordMapperImpl may be implemented to add the possibility to map 
	keywords to multiple fields, translate values from query units to database units, or
	add any other specific handling.
	
*	QueryMapper automatically keeps a list of Restrictable keywords supported by node,
	it can be fetched using **public Collection<Restrictable> getRestrictables();** method.
	
*	From XSAMS builder methods **mapAliasedTree(...)** or **mapTree(...)** methods are called to construct 
	Cayenne Expressions from incoming query trees or filtered subtrees.
	