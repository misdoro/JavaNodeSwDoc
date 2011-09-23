.. _metrics:

Query metrics support
================================

To enable the estimation of the amount of data the node will return for any specific query,
node software supports HEAD request queries, containing the same set of parameters as normal GET/POST queries.

As a response node software provides a set of HTTP headers indicating the estimate count of
each XSAMS block elements:

* *VAMDC-COUNT-SPECIES*
	Total count of the atomic **Ion** and **Molecule** records with distinct SpecieID attribute.
	
* *VAMDC-COUNT-ATOMS*
	Count of the atomic **Ion** records with distinct SpecieID attribute.
	
* *VAMDC-COUNT-MOLECULES*
	Count of the **Molecule** records with distinct SpecieID attribute.
	
* *VAMDC-COUNT-SOURCES*
	Count of distinct **Source** records
	
* *VAMDC-COUNT-STATES*
	Count of distinct **State** records, both **AtomicState** and **MolecularState** combined
	
* *VAMDC-COUNT-COLLISIONS*
	Count of the **CollisionalTransition** elements of the **Processes** branch of XSAMS.
	
* *VAMDC-COUNT-RADIATIVE*
	Count of the **RadiativeTransition** elements of the **Processes** branch of XSAMS.
	
* *VAMDC-COUNT-NONRADIATIVE*
	Count of the **NonRadiativeTransition** elements of the **Processes** branch of XSAMS.
	

getMetrics(...) method
---------------------
	
Producing such a response doesn't require to build a document tree, 
so a dedicated plugin method is called by the node software 
for each HEAD request.
	
::	
	
	public abstract Map<Dictionary.HeaderMetrics,Integer> getMetrics(RequestInterface userRequest);
	
**userRequest**	has the same interface as the parameter of buildXSAMS method,
but it doesn't expect to have XSAMS tree attached.

Typical logic of the method would be like that:

* translate the query using the same method as builders use,

* convert the query into Count() using org.vamdc.tapservice.query.CountQuery.count(DataContext,SelectQuery) static method

* add the resulting value to the map of headers

* iterate if more then one builder is normally used


Sample implementation
------------------------

Here follows the sample implementation from BASECOL database plugin

::

	@Override
	public Map<HeaderMetrics, Integer> getMetrics(RequestInterface request) {
		if (request.isValid() && checkRequest(request)){
			return Metrics.estimate(request);
		}
		return null;
	}

	
and Metrics.estimate has the following implementation::

	public static Map<HeaderMetrics, Integer> estimate (RequestInterface request){
		Map<HeaderMetrics, Integer> estimates = new HashMap<HeaderMetrics, Integer>();
		
		//Estimate collisions
		Expression colExpression = CollisionalTransitionBuilder.getExpression(request);
		SelectQuery query=new SelectQuery(Collisions.class,colExpression);
		Long collisions = CountQuery.count((DataContext) request.getCayenneContext(), query);
		
		if (collisions>0)
			estimates.put(HeaderMetrics.VAMDC_COUNT_COLLISIONS, collisions.intValue());
		
		//Estimate species
		Expression spExpression = ElementBuilder.getExpression(request);
		SelectQuery spQuery=new SelectQuery(EnergyTables.class,spExpression);
		Long etables = CountQuery.count((DataContext) request.getCayenneContext(), spQuery);
		
		if (etables>0)
			estimates.put(HeaderMetrics.VAMDC_COUNT_SPECIES, etables.intValue());
		
		return estimates;
		
	}

Here, getExpression(...) methods are the same translator methods as used in corresponding builders.
