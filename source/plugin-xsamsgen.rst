.. _XSAMSGen:

XSAMS tree building
=========================

XSAMS is an XML schema, adopted within VAMDC for data exchange.

Java node software implementation uses JAXB library to establish 
the mapping between Java objects and XML elements and to read/write XSAMS documents

Each node plugin is responsible for building object trees corresponding to the XSAMS document branches and
for attaching them to the main tree, managed by the node software.
Once the node plugin has finished building the XSAMS document and returned from the buildXSAMS() method, 
the node software outputs the XML document to the client.

Java objects corresponding to every element of XSAMS schema 
are generated using JAXB library and are a part of *org.vamdc.xsams* package, included in **xsams** and **xsams-extra** libraries.

XSAMS objects can be constructed by extension of Jaxb mapping classes with convenience methods,
provided within **xsams-extra** library.
Constructors can receive Cayenne mapping objects as argument and initialize appropriate mapping XML fields.
Such an approach allows to instantly apply an arbitrary processing to any field/element value or their combinations.


Example constructor class
-------------------------

As an example let's look at the BASECOL Source element constructor::

	package org.vamdc.basecol.xsams;

	import org.vamdc.basecol.dao.RefsArticles;
	import org.vamdc.basecol.dao.RefsGroups;
	import org.vamdc.xsams.XSAMSManager;
	import org.vamdc.xsams.schema.SourceCategoryType;
	import org.vamdc.xsams.schema.SourceType;
	import org.vamdc.xsams.util.IDs;

	public class Source extends SourceType{

		public Source(RefsArticles article){

			setSourceID(IDs.getSourceID(article.getArticleID().intValue()));
			setCategory(SourceCategoryType.fromValue(article.getJournalRel().getCategory()));

			setSourceName(article.getJournalRel().getSmallName());

			//Year
			setYear(article.getYear().intValue());

			//Authors
			setAuthors(new Authors(article.getFlatAuthorRel()));				
			//Title
			setTitle(article.getTitle());	
			//URL
			setUniformResourceIdentifier(article.getUrl());
			//Volume
			setVolume(article.getVolume());
			//Pages
			String pagesbe = article.getPage();
			if (pagesbe!=null){

				if (pagesbe.contains("-")){
					fillPages(pagesbe,"-");
				}else if (pagesbe.contains("\\+")){
					fillPages(pagesbe,"+");
				}
			};

		}
	}

The full source is available in **org.vamdc.basecol.xsams.Source** class.

Here, RefsArticles is the BASECOL Cayenne mapping object identifying one source record in the database, 
and SourceType is a root element of XSAMS Sources branch. 
SourceType is defined by the class **org.vamdc.xsams.schema.SourceType**.

Collection of RefsArticles objects is retrieved automatically through the Cayenne model relation.
For each *Source* element we need to check if it is already attached to the XSAMS Document tree.
If no existent object is found, 
the aforementioned builder is called and the generated object is attached to the document tree.::


	public static List<SourceType> getSources(
			List<RefsGroups> referenceRel, 
			XSAMSManager document, 
			boolean filterSource) {
			
		ArrayList<SourceType> result = new ArrayList<SourceType>();

		/*always add the database self-reference*/
		result.add(document.getSource(IDs.getSourceID(0)));

		if (referenceRel==null)	return result;

		/*Add all sources that are stated as 'isSource'*/
		for (RefsGroups myref:referenceRel){
			RefsArticles article = myref.getArticleRel();
			if (article!=null && (myref.getIsSource() || !filterSource)){
				//Check if the source with this ID is already referenced:
				SourceType source = document.getSource(
					IDs.getSourceID(article.getArticleID().intValue())); 
				if (source == null){//If not, create and add it:
					source = new Source(article);
					document.addSource(source);
				}
				//Now, add source record to the list of source references
				result.add(source);
			}
		}
		return result;
	}


This list of the SourceType objects should be passed as the argument to the
objects requiring the bibliographic references. 
For example, if a new DataType object is created, references may be attached to it in the following manner::

	DataType quantity = new DataType(table.value, table.units);
	quantity.addSources(Source.getSources(table.sourceRelation,request,true));
	
Here, "table" is an object of the database model, providing value and units fields plus the relation to the sources.

	
Attaching objects to the XSAMS Document tree
---------------------------------------------

The interface **RequestInterface** provides the access 
to the XSAMS Document tree through the **XSAMSManager** interface.
Node plugin may obtain the XSAMSManager by calling the **getXsamsManager()** method of the **RequestInterface**.

**org.vamdc.xsams.XSAMSManager** interface provides a handful of methods to add different branches to the XSAMS tree,
getting them by known ID or iterating through all of them. For a full list of methods,
consult the JavaDoc of the JAXB XSAMS library [XSAMSJavaDoc]_.

Most commonly used methods are:

*	String addSource(SourceType source);
	  returning the source identifier

*	String addElement(SpeciesInterface species);
	  returning the species identifier

*	int addStates(String speciesID,Collection<? extends StateInterface> states);
	  returning the number of added atomic or molecular states

*	boolean addProcess(Object process);
	  returning true if the process element was attached to the tree

for adding the sources, species, states and processes respectfully.

For each XSAMS block that has an identifier, a convenience method is implemented
allowing to fetch the root element of the block by its string identifier.
For example, for the sources block there exists a method **SourceType getSource(String sourceID)**.

A complete list of methods is available in the JavaDoc of the XSAMSManager interface [XSAMSJavaDoc]_.


Identifiers generation
-------------------------

Each major block of an XSAMS document has an unique identifier,
an arbitrary alphanumeric string starting with a block-specific symbol.

To assure VAMDC-wide uniquiness of those identifiers, a node-specific prefix is inserted into every identifier string.
For instance, that allows to merge the documents coming from the different databases.

In Java node software the identifiers are managed by a special class, **org.vamdc.xsams.IDs**. 
That class provides several static constants and methods:

*	String getID(char prefix, String suffix) 
		Most generic method, allowing to generate an arbitrary identifier.
		The allowed prefix values are enumerated as *public final static char* constants:
		
		-	IDs.SOURCE
		-	IDs.ENVIRONMENT
		-	IDs.SPECIE
		-	IDs.FUNCTION
		-	IDs.METHOD
		-	IDs.STATE
		-	IDs.MODE
		-	IDs.PROCESS

*	String getSourceID(int idSource) 
		;
*	String getEnvID(int idEnv) 
		for the environment identifiers
*	String getFunctionID(int idFunction)
		;
*	String getMethodID(int idMethod)
		;
*	String getStateID(int EnergyTable, int Level)
		;
*	String getModeID(int molecule, int mode)
		;
*	String getSpecieID(int idSpecies)
		;
*	String getProcessID(char group, int idProcess)
		;

All those ID generation methods automatically add the configured node-specific ID prefix.


XSAMS JAXB convenience extensions
-------------------------------------

For convenience, all the XSAMS object classes were extended and grouped 
by the schema block into different packages :


* org.vamdc.xsams.common
	for elements used all around the schema
* org.vamdc.xsams.environments
	for elements from the Environments branch
* org.vamdc.xsams.functions
	for elements from the Functions branch
* org.vamdc.xsams.methods
	for elements from the Methods branch
* org.vamdc.xsams.process
	for elements from the Processes (collisions,transitions) branch
* org.vamdc.xsams.sources
	for elements from the Sources branch
* org.vamdc.xsams.species
	for elements from the Species (atoms, molecules, particles, solids) branch

	
	
Few value constructors were added:

*	class **org.vamdc.xsams.species.molecules.ReferencedTextType**::

		public ReferencedTextType(String value);

	Creates a ReferencedTextType element with the defined value
	
*	class **org.vamdc.xsams.sources.AuthorsType**::

		public AuthorsType(Collection<String> authors)
		public AuthorsType(String concatAuthors, String separator)
	
	First constructor creates Authors elemen with all authors from the passed collection,
	second one splits the first argument using the separator from the second one and puts the
	resulting strings into distinct Author records.
	
*	class **org.vamdc.xsams.sources.AuthorType**::

        	public AuthorType(String name)
        
        Creates a single Author element with the name from the argument.
        
*	class **org.vamdc.xsams.common.TabulatedDataType**::

		public TabulatedDataType(String... CoordsUnits);
		public TabulatedDataType(Collection<String> columns);
		
	Constructors, defining multi-dimensional tables. Parameters passed define the units of axes,
	the last element of the collection or the last string define the units for Y (values).
	The *org.vamdc.xsams.common.TabulatedDataType* class contains a full set of methods for the
	XSAMS tables manipulation, so if you need to use them it is 
	worth reading the XSAMS library JavaDoc [XSAMSJavaDoc]_
	
*	class **org.vamdc.xsams.common.DataType**::
        
        	public DataType(Double value,String units, AccuracyType accuracy, String comments);
        	public DataType(Double value,String units);
        
        You will certainly use DataType objects, since almost any quantity in XSAMS is represented by them.
        Two constructors are provided, with parameter names speaking for themselves.
        Source references may be attached to created object later 
        by calling the *addSource()* or *addSources()* methods.
        
*	class **org.vamdc.xsams.common.ValueType**::

	        public ValueType(Double value, String units);
	        
	ValueType, used as often as the DataType, supports no source reference and is a simple extension 
	of the Double type, providing the *units* attribute. Convenience constructor is also provided for it.
	
*	class **org.vamdc.xsams.common.ChemicalElementType**::

		public ChemicalElementType(int charge, String symbol);
	
	Used in Atoms and Solids branches, ChemicalElementType has a convenience constructor consuming
	the atom nuclear charge and its chemical element symbol.



Case-By-Case generic builders
--------------------------------

Molecular state quantum numbers in XSAMS are represented as additional XML sub-schemas,
defining an element QNs with the ordered child quantum number elements.
Each case has its own separate namespace, that means that Java JAXB mapping 
of each case would be in a separate package and the user would either require a generic builder using
Java Reflection or have a builder for each case.

Since all cases are just combinations of roughly 30 quantum numbers,
it was chosen to create an intermediate structure to contain them, together with the case identifier. 
The class name is **org.vamdc.xsams.util.StateCore**.
It holds a collection of quantum numbers and other important state-related information.

Each quantum number is represented by the **org.vamdc.xsams.util.QuantumNumber** object.
That object contains the value, optional label and mode index, plus the mandatory quantum number type.

Each case package is complemented with its own builder.
The general case builder **org.vamdc.xsams.cases.CaseBuilder** accepts the **StateCore** object 
as a single parameter.
Depending on the caseID parameter value, a specific case builder is called, returning the built tree.
The identifier Case ID is the same as defined in the case-by-case documentation.

The following code illustrates the use of the **CaseBuilder**::

	StateCore statedata = new BasecolStateCore(myetable, level);
	MolecularStateType molecularState = new MolecularStateType();
	// filling in other MolecularStateType fields is omitted
	if (myrequest.checkBranch(Requestable.MoleculeQuantumNumbers))
		molecularState.getCases().add(CaseBuilder.buidCase(statedata));
			
Here, **BasecolStateCore** is a custom class that extends **StateCore**, providing a custom constructor
to fill the fields from the Basecol Cayenne objects.

MolecularStateType is the autogenerated XSAMS JAXB mapping class 
that should be fed direcly to the XSAMS library by calling the::
	
	RequestInterface.getXsamsroot().addState(speciesID, molecularState);

Please note that the element corresponding to the given speciesID should be attached to the XSAMS tree 
before attaching the state.

