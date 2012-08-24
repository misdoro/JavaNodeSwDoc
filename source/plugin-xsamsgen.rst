.. _XSAMSGen:

XSAMS tree building
=========================

XSAMS is an XML schema, adopted within VAMDC for data exchange.

Java node software implementation uses JAXB library for mapping between objects and XML elements.

Contrary to the Python/Django node software, Java version doesn't provide limited keyword-based XML generator.

Each node plugin is responsible by itself for building object trees corresponding to the document branches and
for attaching them to the main tree, managed by the node software.
Node software than outputs the built tree as XML XSAMS document.


XSAMS objects can be constructed by extension of Jaxb mapping classes with convenience methods,
provided within **xsams-extra** library.
Constructors can receive Cayenne mapping objects as argument and initialize appropriate mapping XML fields.
Such an approach allows to instantly apply an arbitrary processing to any field/element value or their combinations.


Example constructor class
-------------------------

As an example let's look at the BASECOL Source element constructor::

	package org.vamdc.basecol.xsams;

	import java.util.ArrayList;
	import java.util.List;

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
		//...
	}

The full source is available in **org.vamdc.basecol.xsams.Source** class.

Here, RefsArticles is a BASECOL Cayenne mapping object identifying one source record, 
and SourceType is a root element of XSAMS Sources branch. 
SourceType is defined by the class **org.vamdc.xsams.schema.SourceType**.

Collection of RefsArticles objects is retrieved automatically through the Cayenne model relation.
For each reference element we need to check if it is already attached to the XSAMS Document tree.
If not, then the mentioned above builder is called, and finally,
after the SourceType object is built, it needs to be attached to the document tree::


	public static List<SourceType> getSources(
			List<RefsGroups> referenceRel, 
			XSAMSManager document, 
			boolean filterSource) {
			
		//Here sources will be added
		ArrayList<SourceType> result = new ArrayList<SourceType>();

		/*always add database self-reference*/
		result.add(document.getSource(IDs.getSourceID(0)));

		if (referenceRel==null)
			return result;

		/*Add all sources that are stated as 'isSource'*/
		for (RefsGroups myref:referenceRel){
			RefsArticles article = myref.getArticleRel();
			if (article!=null && (myref.getIsSource() || !filterSource)){
				//Check if source with this ID was already referenced:
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


Later this list should be added to the element requiring source reference,
for example, we create a new DataType value and have references attached to it::

	DataType quantity = new DataType(table.value, table.units);
	quantity.addSources(Source.getSources(table.sourceRelation,request,true));
	
Here, "table" is an object of your database model, providing value and units fields plus the relation to the sources.
First, we need to create a quantity of the DataType, then we construct all related source elements, 
automatically adding them to the XSAMS document tree if necessary, and attach to the quantity element.

	
Attaching objects to XSAMS Document tree
------------------------------------------

**RequestInterface** provides access to XSAMS Document tree through **XSAMSManager** interface, implementation of
which can be obtained by calling **getXsamsManager()** method of the request.

**org.vamdc.xsams.XSAMSManager** interface provides a handful of methods to add different branches to the XSAMS tree,
getting them by known ID or iterating through all of them. For a full list of methods,
consult the JavaDoc of the JAXB XSAMS library [XSAMSJavaDoc]_.

Notable are:

*	public String addSource(SourceType source);

*	public String addElement(SpeciesInterface species);

*	public int addStates(String speciesID,Collection<? extends StateInterface> states);

*	public boolean addProcess(Object process);

for adding correspondingly sources, species, states and processes.



Identifiers generation
-------------------------

Each major block of XSAMS has it's own unique identifier,
which is a string starting with a block-specific character.

To assure VAMDC-wide uniquiness of those identifiers, permitting merging of documents,
NodeSoftware (both Python and Java implementations) have a mechanism for adding node-specific prefix.

For Java node software it is a special class, **org.vamdc.xsams.IDs**, providing several constants and methods.

*	public static String getID(char prefix, String suffix) 
		Most generic method, allowing to generate an arbitrary ID.
		All allowed prefix values are enumerated as *public final static char* constants:
		
		-	IDs.SOURCE
		-	IDs.ENVIRONMENT
		-	IDs.SPECIE
		-	IDs.FUNCTION
		-	IDs.METHOD
		-	IDs.STATE
		-	IDs.MODE
		-	IDs.PROCESS

*	public static String getSourceID(int idSource)
*	public static String getEnvID(int idEnv)
*	public static String getFunctionID(int idFunction)
*	public static String getMethodID(int idMethod)
*	public static String getStateID(int EnergyTable, int Level)
*	public static String getModeID(int molecule, int mode)
*	public static String getSpecieID(int idSpecies)
*	public static String getProcessID(char group, int idProcess)

All those ID generation methods automatically add the configured node-specific ID prefix.


XSAMS JAXB convenience extensions
-------------------------------------

For convenience, all XSAMS object classes were extended and grouped into packages
by the schema block they are appearing in:


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
	the atom nuclear charge and it's chemical element symbol.

So far, this is the full list of all convenience constructors created for the XSAMS library.
If you need more convenience constructors or methods to be added, 
contact the Java node software authors and those methods would be included in the next software release.


Case-By-Case generic builders
--------------------------------

Molecular state quantum numbers in XSAMS are represented as additional XML sub-schemas,
defining an element QNs with ordered child quantum number elements.
Each case has it's own separate namespace, that means that Java JAXB mapping 
of each case would be in a separate package and the user would either require a generic builder using
Java Reflection or have a builder for each case.

Since all cases are just combinations of roughly 30 quantum numbers,
the decision was taken to create an intermediate structure able to keep all of them plus
the case identifier. The class name is **org.vamdc.xsams.util.StateCore**.
It is able to contain a collection of quantum numbers and other important state-related information.

Each quantum number is represented by the **org.vamdc.xsams.util.QuantumNumber** object.
It contains the value, optional label and mode index plus the mandatory quantum number type, 
defining the place where in the case-by-case representation the value will go.

Each autogenerated case package is complemented with it's own builder.
The general case builder **org.vamdc.xsams.cases.CaseBuilder** accepts **StateCore** as a single parameter and
is calling case builders based on the integer case ID, returning the built tree.
Case ID is the same as it is defined in the case-by-case documentation.
The following code illustrates the use::

	StateCore statedata = new BasecolStateCore(myetable, level);
	MolecularStateType molecularState = new MolecularStateType();
	// filling in other MolecularStateType fields is omitted
	if (myrequest.checkBranch(Requestable.MoleculeQuantumNumbers))
		molecularState.getCases().add(CaseBuilder.buidCase(statedata));
			
Here, BasecolStateCore is a custom class that extends StateCore to automatically
fill in all the fields from the Basecol Cayenne model.

MolecularStateType is the autogenerated XSAMS JAXB mapping class 
that should be fed direcly to the XSAMS library by calling the::
	
	RequestInterface.getXsamsroot().addState(speciesID, molecularState);

Obviously, the element corresponding to the speciesID should already be there.

