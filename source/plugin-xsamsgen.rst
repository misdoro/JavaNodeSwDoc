.. _XSAMSGen:

XSAMS tree builder
=========================

XSAMS is an XML schema, adopted within VAMDC for data exchange.

Java node software implementation uses JAXB library for mapping between objects and XML elements.

Contrary to the Python/Django node software, Java version doesn't provide neither it's own XML generator, nor a plain
list of mapping keywords (Returnables as they are called in Python version).

Each node plugin is responsible by itself for building object trees corresponding to the document branches and
for attaching them to the main tree. Node software than outputs the built tree as XML XSAMS document.

Basically, the XSAMS builder solves the same problem as Python node software Returnables dictionary.
As input builder methods receive Apache Cayenne objects, and their job is to produce JAXB XSAMS objects 
with relevant attributes copied from the Cayenne objects. 
Such an approach allows to instantly apply an arbitrary processing to any field/element value or their combinations.

Example builder class
-------------------------

As an example let's look at the BASECOL Source element builder. 
The full source is available in **org.vamdc.basecol.builders.SourcesTypeBuilder** class.
Here follow some extracts from it.

The only public method available is::

	public static List<SourceType> getSources(
			List<RefsGroups> referenceRel, 
			RequestInterface myrequest, 
			boolean filterSource)

It accepts as parameters

*	the list of BASECOL model bibliography objects, as they are returned by relations
*	request, providing interface for XSAMS document tree and the incoming query tree
*	BASECOL-specific boolean switch that may change the behaviour of method, when needed.

and returns the list of SourceType objects that can be attached to any JAXB object corresponding to the element
extending XSAMS PrimaryType.

SourceType objects are built by a simple method, accepting Cayenne object as a parameter.
::

	private static SourceType getSource(RefsArticles thisarticle) {
		
		SourceType source = new SourceType();
		
		source.setSourceID(
				IDs.getSourceID(thisarticle.getArticleID().intValue()));
		source.setCategory(SourceCategoryType.fromValue(thisarticle.getJournalRel().getCategory()));
		source.setSourceName(thisarticle.getJournalRel().getSmallName());

		XMLGregorianCalendar cal = null;
		try{
			cal = DatatypeFactory.newInstance().newXMLGregorianCalendar();
			cal.setYear(thisarticle.getYear().intValue());
			cal.setTimezone(DatatypeConstants.FIELD_UNDEFINED);
		}catch (DatatypeConfigurationException e) {
			e.printStackTrace();
		}
		source.setYear(cal);

		//Authors
		source.setAuthors(buildAuthors(thisarticle.getFlatAuthorRel()));				
		//Title
		source.setTitle(thisarticle.getTitle());	
		//URL
		source.setUniformResourceIdentifier(thisarticle.getUrl());
		//Volume
		source.setVolume(thisarticle.getVolume());
		//Pages
		String pagesbe = thisarticle.getPage();
		if (pagesbe!=null){
			String[] pages = pagesbe.split("--");
			source.setPageBegin(pages[0]);
			if (pages.length>1)source.setPageEnd(pages[1]);
		};
		return source;
	}

Here, RefsArticles is a BASECOL Cayenne mapping object identifying one source record, 
and SourceType is a root element of XSAMS Sources branch. 
SourceType is defined by the class **org.vamdc.xsams.schema.SourceType**.

Collection of RefsArticles objects is retrieved automatically through the Cayenne model relation.
For each reference element we need to check if it is already attached to the XSAMS Document tree.
If not, then the mentioned above builder is called, and finally,
after the SourceType object is built, it needs to be attached to the document tree::


	public static List<SourceType> getSources(
			List<RefsGroups> referenceRel, RequestInterface myrequest, boolean filterSource) {
			
	//This array is returned from the method and can be passed to the 
	//addSources(Collection<SourceType>) method of all objects, requiring sources
	ArrayList<SourceType> newsources = new ArrayList<SourceType>();
	
	/*Add all sources that are stated as 'isSource'*/
	for (RefsGroups myref:referenceRel){
		RefsArticles thisarticle = myref.getArticleRel();
		if (thisarticle!=null && (myref.getIsSource() || !filterSource)){
			SourceType source = null;
			//Check if source with this ID already exists:
			if (myrequest!=null) source = myrequest.getXsamsroot().getSource(
					IDs.getSourceID(thisarticle.getArticleID().intValue())); 
			//If not, create it:
			if (source == null){
				source = getSource(thisarticle);
				if (myrequest!=null) myrequest.getXsamsroot().addSource(source);
			}
			//Now, add source record to the list of source references:
			newsources.add(source);
		}
	}
	
	//Return the sources collection for later use.
	return newsources;
	}


Later this list should be added to the element requiring source reference,
for example, we create a new DataType value and have references attached to it::

	DataType quantity = new DataType(table.value, table.units);
	quantity.addSources(SourcesBuilder.getSources(table.sourceRelation,request,true));
	
Here, "table" is an object of your database model, providing value and units fields plus the relation to the sources.
First, we need to create a quantity of the DataType, then we construct all related source elements, 
automatically adding them to the XSAMS document tree if necessary, and attach to the quantity element.

	
Attaching objects to XSAMS Document tree
------------------------------------------

**RequestInterface** provides access to XSAMS Document tree through **XSAMSData** interface, implementation of
which can be obtained by calling **getXsamsroot()** method of the request.

**org.vamdc.xsams.XSAMSData** interface provides a handful of methods to add different branches to the XSAMS tree,
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

	
	
Few value constructors were added for convenience:

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



