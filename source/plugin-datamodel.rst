.. _datamodel:

Database Object Model
========================

Java VAMDC-TAP node software implementation suggests and supports the use of
Apache Cayenne Object-Relational Mapping (ORM) library.

Apache Cayenne supports various database engines, notably, MySQL, PostgreSQL, SQLite, Oracle, DB2, Microsoft SQL Server.

Creation of database objects is made simple thanks to the graphical modeler application,
provided as a part of Cayenne.

Process of creating and using a database model is well described in the project official documentation [CAYDOC]_
and there is no need to repeat it in this document. Reading the Cayenne documentation is a **MUST** for the understanding
and creating efficient query mapper routines and high performance database access classes.

Step-by-step guide
----------------------

Create maven project
+++++++++++++++++++++++

Import data model
+++++++++++++++++++++++

Adjust data model
+++++++++++++++++++++

Generate Java classes
++++++++++++++++++++++

Save cayenne project under src/main/resources

Notable Cayenne features used
-------------------------------

The goal of using ORM framework is not only to get object-oriented view of database, but rather to 
simplify and automate query translation.

Relations traversing
++++++++++++++++++++++

If properly defined, database model contains information about all tables relations through the foreign keys.
While constructing the query, those relations can be automatically traversed to form the correct query with desired
selection criterias. 

For example, we have a table **'artists'** with fields **'id'** and **'name'**
and a table **'albums'** with fields **'id'**, **'artistId'**, **'name'** and **'year'**
For the table **'albums'** we have one-to-one relation with the **'artists'** table, called **'albumArtist'**
and for artists the reverse one-to-many relationship **'artistAlbums'**

So, if we want to get all artists that released albums in 1980, we would create an **Expression** containing the path
from the **'artists'** table to the **'year'** field of **'albums'** table and the expression type **'match'**

::

	Expression exp = ExpressionFactory.matchExp("artistAlbums.year", 1980);
	SelectQuery query = new SelectQuery(Artists.class,exp);
	List<Artists> artists = context.performQuery(query);

To add another constraint on a query, we may redefine the Expression::

	exp = exp.andExp(ExpressionFactory.likeExp("name", "Thomas%"));
	
Here we are not traversing the relationship, but using the table field as a constraint directly.

Normally, none of the expressions would require 'manual' construction, 
they will be translated from the incoming queries. Query translation is described in a separate chapter :ref:`queryMap`


Path aliases
+++++++++++++++

Imagine that we have the scenario of many-to-many relation through a separate table.
For the previous example, let's add a table **'artistAlbums'** with three columns, **'id'**, **'artistId'** and **'albumId'**
Table **'albums'** doesn't any more contain the 'artistID' column, but both forward and reverse relations are still 
called **'albumArtists'** and **'artistAlbums'**

If we need to select artists that released albums both in 1980 and 1990,
joining expressions neither with exp.andExp nor exp.orExp would give us appropriate queries.

exp.andExp() would return no results,

exp.orExp() would return all artists that released albums either in 1980 or 1990.

For such a case, Cayenne provides aliases mechanism::

	Expression e1 = ExpressionFactory.match("artistAlbumsAlias1.year", 1980);
	Expression e2 = ExpressionFactory.match("artistAlbumsAlias2.year", 1990);
	Expression e = e1.andExp(e2);
	q = new SelectQuery(Artists.class, e);
	q.aliasPathSplits("artistAlbums", "artistAlbumsAlias1", "artistAlbumsAlias2");
	
That last command tells the select query how to interpret the alias. 
Because the aliases are different, the SQL generated will have two completely separate set of joins.
This is called a "split path".



