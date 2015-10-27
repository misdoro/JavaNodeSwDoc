.. _metrics:

Query metrics support
================================

To give the estimation of the volume of data returned by the node for a query,
the node software supports HEAD request queries.
The parameter set of the HEAD request is the same as for the GET requests.

The VAMDC-specific HTTP headers that may be present in the response are the following:

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

* *Last-Modified* HTTP header can also be sent to client to indicate the time when the extracted 
	data was modified last time.

getMetrics(...) method
------------------------
	
HTTP HEAD response may be produced without building the XSAMS document tree.
For this purpose a dedicated plugin method is called by the node software 
for each incoming HEAD request.
	
::	
	public abstract Map<Dictionary.HeaderMetrics,Object> getMetrics(RequestInterface userRequest);
	
**userRequest** parameter has the same interface as 
the parameter of the plugin *buildXSAMS* method.
Since the HEAD response will not transmit the XSAMS document body,
there is no need to access the *XSAMSManager* object.

Typical logic of the method implementation would be the following:

* Translate the query using the same translation strategy as the builders use;

* Convert the query into the *SELECT Count(\*) WHERE ...* using the
  *org.vamdc.tapservice.query.QueryUtil.countQuery(DataContext,SelectQuery)* static method;

* Add the obtained count() value along with the corresponding HTTP header to the map of result headers;

* If the obtained count() value is greater than zero, set the value of the *Last-Modified* header
  by using the **RequestInterface** *setLastModified(...)* method and 
  *org.vamdc.tapservice.query.QueryUtil.lastTimestampQuery(...)* translator to obtain the 
  value from the database last-modified date column.

* iterate over all primary tables if more than one primary table is used to query the database.


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

	public static Map<HeaderMetrics, Object> estimate (RequestInterface request){
		Map<HeaderMetrics, Object> estimates = new HashMap<HeaderMetrics, Object>();
		
		//Estimate collisions
		Expression colExpression = CollisionalTransitionBuilder.getCayenneExpression(request);
		SelectQuery query=new SelectQuery(Collisions.class,colExpression);
		Long collisions = QueryUtil.countQuery((DataContext) request.getCayenneContext(), query);
		
		if (collisions>0){
			estimates.put(HeaderMetrics.VAMDC_COUNT_COLLISIONS, collisions.intValue());
			
			request.setLastModified(QueryUtil.lastTimestampQuery(
					(DataContext) request.getCayenneContext(), 
					query, 
					"modificationDate"));
		}
		
		//Estimate species
		Expression spExpression = ElementBuilder.getExpression(request);
		SelectQuery spQuery=new SelectQuery(EnergyTables.class,spExpression);
		Long etables = QueryUtil.countQuery((DataContext) request.getCayenneContext(), spQuery);
		
		if (etables>0){
			estimates.put(HeaderMetrics.VAMDC_COUNT_SPECIES, etables.intValue());
			
			request.setLastModified(QueryUtil.lastTimestampQuery(
					(DataContext) request.getCayenneContext(), 
					spQuery, 
					"modificationDate"));
		}
		
		return estimates;
		
	}

Here, the getExpression(...) methods are the same translator 
methods as used in corresponding XSAMS builders.
