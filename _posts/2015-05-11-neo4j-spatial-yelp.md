![](/content/images/2015/05/screenshot.png)

A common use-case for database queries is to search for things that are close to other things or within some specified geospatial boundary. Geospatial indexes and queries are offered by NoSQL databases, such as [MongoDB](http://docs.mongodb.org/manual/core/geospatial-indexes/) and relational databases such as [PostgreSQL](http://postgis.net/). But what about graph databases? In this article, I show how to create a web application to search within a user-defined boundary powered by the Neo4j graph database.

## Searching for businesses within a user defined boundary

Our requirements for this web app are:

1. allow the user to draw a polygon on a map
1. allow the user to select the category of business they would like to search for
1. show the user businesses that match their selected category and within the bounds they have defined on the map

You can check it out [here](http://spatialcypherdemo.herokuapp.com/), but this is what it looks like:

![spatial query gif](/content/images/2015/05/out.gif)

In the rest of this post we'll look at how to build this project.

## The stack

Let's take a look at the components of this project.

#### Data

The data for this project comes from the [Yelp Academic Dataset](https://www.yelp.com/academic_dataset). At the time that I built this project I was a graduate student at the University of Montana so I'm pretty sure I was operating within the confines of Yelp's license for the use of this data. Needless to say, I can't redistribute the data so if you'd like to follow along you'll need to download it directly from Yelp.

The dataset contains information about over 100,000 businesses in the Phoenix, AZ area in the U.S. This includes Yelp user reviews and some anonymized information about the users. We will use the latitude / longitude data to map businesses, as well as the business category for filtering.

#### Neo4j Spatial

[Neo4j](http://neo4j.com) is a graph database that allows for modeling, storing and querying data as a graph. If you haven't been exposed to graph databases yet it's worth checking out as many use cases are naturally modeled as a graph. [Neo4j Spatial](https://github.com/neo4j-contrib/spatial) is a plugin for Neo4j that enables spatial operations: spatial indexing and spatial querying (such as find things within x distance of y or find things within a certain geometry.

Here we will use Neo4j Spatial's `SearchIntersects` query to perform an efficent spatial query to find businesses within a user defined polygon.

#### MapBox / Leaflet.js
[Leaflet.js](http://leafletjs.com/) is an open-source JavaScript maps library that makes it really easy to create maps in the browser. MapBox provides an extension to Leaflet, Mapbox.js, that adds some features and makes it easier to work with MapBox's tile servers.

Now that we've seen what tools we'll be working with, the first step is to load our data into Neo4j.

## Loading the data

The data consists of an array of JSON objects describing each business. Something like this:

{% highlight json %}
// from https://www.yelp.com/academic_dataset
{
  'type': 'business',
  'business_id': (a unique identifier for this business),
  'name': (the full business name),
  'neighborhoods': (a list of neighborhood names, might be empty),
  'full_address': (localized address),
  'city': (city),
  'state': (state),
  'latitude': (latitude),
  'longitude': (longitude),
  'stars': (star rating, rounded to half-stars),
  'review_count': (review count),
  'photo_url': (photo url),
  'categories': [(localized category names)]
  'open': (is the business still open for business?),
  'schools': (nearby universities),
  'url': (yelp url)
}
{% endhighlight %}

Lots of interesting data available here but we are only interested in `business_id`, `name`, `latitude`, `longitude`, and `categories`.

There are [several](http://neo4j.com/docs/stable/query-load-csv.html) [ways](http://neo4j.com/docs/stable/import-tool.html) to import data into Neo4j. The Neo4j Spatial plugin even provides functionality for importing from [shapefiles and from OSM data](https://github.com/neo4j-contrib/spatial#examples). I hadn't originally considered making use of the business category information so I ended up with a messy two step data import process:

1. Create a Business node for each business in the dataset and add it to the spatial index
1. Create relationships between this Business node and the appropriate Category nodes

Including the in-graph spatial index, the data model looks like this:

![](/content/images/2015/05/scdemo_datamodel.png)

##### Create Business nodes and add to spatial index

I chose to create an [unmanaged server extension](http://neo4j.com/docs/stable/server-unmanaged-extensions.html) for interacting with Spatial using the Java API (more on this in the next section), adding an endpoint for Business node creation. This endpoint takes a JSON object that represents the business to be created and added to the spatial index:

```
{
    "business_id": "lajsldkjfaslj48o",
    "name": "Bob's Burgers",
    "address": "123 Fake St. Notacity, MT 59601",
    "lat": 22.21323,
    "lon": 14.43211
}
```

I wrote a simple Python script to iterate through all business objects in the dataset and make a POST request to our new endpoint. The script is available [here](https://github.com/johnymontana/scdemo/blob/master/yelp_post.py). We should really be posting an array of business objects here, but we'll save that for v2 ;-) More in the next section about how this POST request is actually handled.

##### Add category relationships to Business nodes

I quickly realized that having business category information would make this demo a bit more interesting. Each business has an array of categories to which it belongs. So we now iterate through the dataset again, but this time execute a cypher query that will add a relationship from the Business node to the appropriate Category nodes for each business, creating our Category nodes along the way using a Cypher `MERGE` statement: 


{% highlight python %}
import json
from py2neo import neo4j

db = neo4j.GraphDatabaseService("http://server_addr_here:7474/db/data/")

merge_category_query = '''
    MATCH (b:Business {business_id: {business_id}})
    MERGE (c:Category {name: {category}})
    CREATE UNIQUE (c)<-[:IS_IN]-(b)
'''

print "Beginning category batch"
with open('data/yelp_academic_dataset_business.json', 'r') as f:
	category_batch = neo4j.WriteBatch(db)
	count = 0
	for b in (json.loads(l) for l in f):
		for c in b['categories']:
			category_batch.append_cypher(merge_category_query, {'business_id': b['business_id'], 'category': c})
			count += 1
			if count >= 10000:
				category_batch.run()
				category_batch.clear()
				print "Running batch"
				count = 0
	if count > 0:
		category_batch.run()
{% end highlight %}

This script uses the [py2neo](https://github.com/nigelsmall/py2neo) Python library written by [Nigel Small](https://twitter.com/neonige) that allows for easy Neo4j interaction from Python. 



## Neo4j Spatial queries / server extension

As I mentioned, we use a Neo4j server extension to add two endpoints to the Neo4j REST API. The first endpoint handles a POST request with the data for a business and then creates a corresponding Business node in the database and adds this node to our spatial index, allowing for later spatial query operations. The second endpoint will handle a GET request, with a [polygon WKT](http://en.wikipedia.org/wiki/Well-known_text) string and business category as parameters and return an array of all businesses found within the polygon that match the given category.





#### `POST .../scdemo/node`
The purpose of this endpoint is to add a given Business into the Spatial layer. Unmanaged extensions allow us to define arbitrary [JAX-RS](http://en.wikipedia.org/wiki/Java_API_for_RESTful_Web_Services) handlers to be executed when a request is sent to our new endpoints. We saw above that we made a series of POST requests, sending a JSON body with details for each business. Here we will look at the code for the handler for that endpoint.

Handler to create Business node and add to the Spatial index:

{% highlight java %}
@POST
@Path("/node")
public Response addNode(String nodeParamsJson, @Context GraphDatabaseService db) {
    Node businessNode;

    SpatialDatabaseService spatialDB = new SpatialDatabaseService(db);

    Gson gson = new Gson();     
    BusinessNode business = gson.fromJson(nodeParamsJson, BusinessNode.class);

    try ( Transaction tx = db.beginTx()) {

        businessNode = db.createNode();
        businessNode.addLabel(Labels.Business);
        businessNode.setProperty("business_id", business.getBusiness_id());
        businessNode.setProperty("name", business.getName());
        businessNode.setProperty("address", business.getAddresss());
        businessNode.setProperty("lat", business.getLat());
        businessNode.setProperty("lon", business.getLon());
        tx.success();
    }

        try (Transaction tx = db.beginTx()) {

        Layer businessLayer = spatialDB.getOrCreatePointLayer("business", "lat", "lon");

        businessLayer.add(businessNode);
        tx.success();
    }

    return Response.ok().build();

}
{% endhighlight %}

We first declare that this method will handle POST requests to `.../node` and that we expect a JSON body string. Next, we get a handle on the `GraphDatabaseServive` and instantiate a new `SpatialDatabaseService`. Really this should only be done once, and then cached for later calls, but I'm just trying to show each handler as a self-enclosed method. Next, we serialize the JSON body to an instance of `BusinessNode` using the [google-gson](https://github.com/google/gson) Java library. `BusinessNode` is a [POJO](http://en.wikipedia.org/wiki/Plain_Old_Java_Object) with instance vars for the properties of our business and the appropriate getters / setters to facilitate serialization with GSON. We then create a new Neo4j node called `businessNode` and set the appropriate properties on this node before committing the transaction. Finally, we get a handle of the Neo4j Spatial point layer and add this newly created `businessNode` to the Spatial layer. Spatial will take care of the appropriate in-graph R-Tree indexing and we can now make spatial query operations on the `businessNode`.

#### `GET .../scdemo/intersects?polygon=WKTPOLYGON&category=xxx`

The other endpoint we add in our server extension executes the spatial query. We send it a business category and a polygon and in return we get an array of all businesses within that polygon that match the business category.

Handler to query within polygon
```
@GET
@Path("/intersects/")
public Response getBusinessesInPolygon(@QueryParam("polygon") String polygon, @QueryParam("category") String category, @Context GraphDatabaseService db) throws IOException, ParseException{
    WKTReader wktreader = new WKTReader();

    ArrayList<Object> resultsArray = new ArrayList();

    SpatialDatabaseService spatialDB = new SpatialDatabaseService(db);
    Layer businessLayer = spatialDB.getOrCreatePointLayer("business", "lat", "lon");
    SpatialIndexReader spatialIndex = businessLayer.getIndex();

    SearchIntersect searchQuery = new SearchIntersect(businessLayer, wktreader.read(polygon));


    try (Transaction tx = db.beginTx()) {
        SearchResults results = spatialIndex.searchIndex(searchQuery);


        for (Node business : results) {
            for (Relationship catRel : business.getRelationships(RelTypes.IS_IN, Direction.BOTH)) {
                Node categoryNode = catRel.getOtherNode(business);
                if (categoryNode.getProperty("name").equals(category)) {
                    HashMap<String, Object> geojson = new HashMap<>();
                    geojson.put("lat", business.getProperty("lat"));
                    geojson.put("lon", business.getProperty("lon"));
                    geojson.put("name", business.getProperty("name"));
                    geojson.put("address", business.getProperty("address"));
                    resultsArray.add(geojson);
                }
            }

        }

        tx.success();
    }

    ObjectMapper objectMapper = new ObjectMapper();
    return Response.ok().entity(objectMapper.writeValueAsString(resultsArray)).build();
}
```

Here we define the handler of a GET request and specify two string query parameters: the polygon, as a [WKT string](http://en.wikipedia.org/wiki/Well-known_text), and the business category. We instantiate a new `SpatialDatabaseService` and get a handle on our Spatial Layer `businessLayer`. We then use Spatial's `SearchIntersect` query to perform a spatial query within our `businessLayer` to find any Nodes intersecting the polygon. We then iterate through the results and build up an array of `HashMap` objects with the business name, latitude and longitude to return as JSON.

## Mapbox / Leaflet frontend

Now that we've imported the data into Neo4j, set up our spatial index, and have an endpoint that will allow us to perform the spatial query that we need it's time to build a frontend for this application. We wil create a web app that renders a map, allows the user to draw an arbitrary polygon on the map and select a business category. Our web app will then need to send a request to our Neo4j server extension endpoint to perform the spatial query and render the results as pin annotations on the map.

##### Load the map

We will use the excellent [Mapbox.js](https://www.mapbox.com/mapbox.js/) JavaScript library to facilitate this. Mapbox.js is an extension to the [Lealet.js](http://leafletjs.com/) library that adds some functionality and makes working with Mapbox tile servers a breeze. 

First we'll need to display the map, enable polygon drawing and define which functions should be called once the polygon is drawn or an existing polygon edited:

```
L.mapbox.accessToken = 'MAPBOX_TOKEN_HERE';
var map = L.mapbox.map('map', 'MAP_ID_HERE')
    .setView([33.47870401153533, -112.0305061340332], 14);
var featureGroup = L.featureGroup().addTo(map);
var dataLayer = L.mapbox.featureLayer().addTo(map);
var current_poly;
var url = "http://NEO4j_SERVER_URL_HERE:7474/scdemo/scdemo/intersects";

// enable polygon drawing
var drawControl = new L.Control.Draw({
      edit: {
        featureGroup: featureGroup
      },
      draw: {
        polygon: true,
        polyline: false,
        rectangle: false,
        circle: false,
        marker: false
      }
    }).addTo(map);

// call showPolygonMarkers function when a polygon is created
map.on('draw:created', showPolygonMarkers);

// call showPolygonMarkersEdited when an existing polygon is edited
map.on('draw:edited', showPolygonMarkersEdited);

// if the category is changed and we have an existing polygon in the map, perform a new query to find businesses in the polygon matching the new category
$("#category").change(function() {
      if (current_poly){
        data = {polygon: current_poly, category: $("#category").val()}
        $.ajax({
          url: url,
          data: data,
          success: dataToMarkers
        });
      }
});
```

##### Handlers for polygon drawing / editing

Here we build a WKT Polygon string from the user defined polygon and make an AJAX request to our Neo4j server extension to perform the spatial query to find all businesses within our polygon / category:

```
function showPolygonMarkersEdited(e) {
      e.layers.eachLayer(function(layer) {
        showPolygonMarkers({ layer: layer });
      });
    }
    function showPolygonMarkers(e) {
      console.log(e);
      console.log($("#category").val());
      featureGroup.clearLayers();
      featureGroup.addLayer(e.layer);
  
      coords = e.layer['_latlngs']
      first = coords[0]['lat'] + " " + coords[0]['lng']+", "
      wkt = "POLYGON (("
      coords.forEach(getCoords)
      wkt = wkt + first.substring(0, first.length-2) + "))"
      console.log(wkt)
      
      current_poly = wkt
      data = {polygon: wkt, category: $("#category").val()}
      success = function(data){
        console.log(data)
      }
      
      $.ajax({
        url: url,
        data: data,
        success: dataToMarkers
        //dataType: dataType
      });
    }
    function getCoords(e) {
      group = e['lat'] + " " + e['lng']+", "
      console.log(group)
      wkt = wkt + group
    }
```

##### Render the markers on Ajax success

Now we just need to create markers for each business in the JSON array that comes back from our server extension and add that layer to the map:
```
function dataToMarkers(data) {
        data = JSON.parse(data)       
        var geojson = { type: 'FeatureCollection', features: [] };
        
        for (var i = 0; i < data.length; i++) {
        
        if (data[i].lon === null || data[i].lat === null) continue;
        geojson.features.push({
          type: 'Feature',
            geometry: {
              type: 'Point',
              coordinates: [ data[i].lon, data[i].lat]
            },
          properties: {
        'marker-size': "large",
        'marker-color': "#c091e6",
        'title': data[i].name
        }
        });
      }
      dataLayer.setGeoJSON([]);
      dataLayer.setGeoJSON(geojson);
      }
```

Well that's about it. You can see the demo live [here](http://spatialcypherdemo.herokuapp.com). The full code is available on my [GitHub](http://github.com/johnymontana) page. Code for the Neo4j server extension is available [here](https://github.com/johnymontana/scdemo-extension) and code for data import and the web app is available in [this repository](https://github.com/johnymontana/scdemo). 

If you'd like to be notified when I publish more posts like this, you can [follow me on Twitter](http://twitter.com/lyonwj).

As an aside, I was fortunate to receive a Google Summer of Code grant last summer to work with the folks at OSGeo and Neo Technology to build a prototype allowing for interacting with Neo4j Spatial directly from the Cypher query language. You can read more about that [here](https://github.com/johnymontana/neo4j/wiki/tutorial).

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@lyonwj">
<meta name="twitter:title" content="Using Neo4j Spatial with Mapbox to search for businesses by location">
<meta name="twitter:description" content="An example using Neo4j Spatial to perform spatial query operations on the Yelp Academic dataset. We build a map frontend web app using Mapbox.js and use Neo4j Spatial on the backend.">
<meta name="twitter:creator" content="@lyonwj">
<meta name="twitter:image:src" content="http://lyonwj.com/content/images/2015/05/screenshot.png">
<meta name="twitter:domain" content="lyonwj.com">
