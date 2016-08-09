---
layout: post
title: Using Neo4j Spatial Procedures in legis-graph-spatial
introtext: Updating legis-graph-spatial to make use of the new spatial procedures in Neo4j 3.0 and the official Neo4j Javascript driver. Procedures provide a new API for interacting with the Neo4j spatial extension and are callable from Cypher.
mainimage: /public/img/spatial_query.png
---

Neo4j 3.0 introduced the concept of [user defined procedures](): code written in Java (or any JVM language) that is deployed to the database and callable from Cypher. User defined procedures are an alternative to unmanaged extensions, with the key difference that user defined procedures are callable from Cypher (instead of extending the http REST endpoints). This allows for an alternative API for [Neo4j's spatial extension](https://github.com/neo4j-contrib/spatial) - namely, exposing Neo4j Spatial's functionality through Cypher!

![](/public/img/spatial_query.png){: .center-image}

If you're not familar with [Neo4j Spatial](https://github.com/neo4j-contrib/spatial), it's an extension to Neo4j that implements spatial indexing that enables import, storage, and querying of spatial data in Neo4j. After the introduction of user defined procedures in Neo4j many procedures were added to spatial, making it much easier to use - instead of making separate http requests to the endpoints exposed by spatial or using the spatial Java API a user can now simply call spatial procedures directly from Cypher (and make use of the official language drivers and the efficient Bolt binary protocol).

Today I got around to updating [legis-graph-spatial](http://legis-graph.github.io/legis-graph-spatial/) to use these new spatial procedures, instead of the spatial REST API it was using previously. Here's a brief overview of the update:

* Simplified congressional district boundaries
* Update to Neo4j 3.04 and spatial 0.20
* Geospatial querying using spatial procedures from Cypher
* Update to official Neo4j Javascript driver using Bolt protocol

# Legis-graph-spatial

If you haven't seen it before, legis-graph-spatial provides a visual way to explore US Congress members topics of influence by district. The user clicks on a map to see who represents that area in the House, as well as information about the committees that representative serves on and over what issues they have influence in Congress. This is all powered by Neo4j. See [this post](http://www.lyonwj.com/2016/03/21/legis-graph-spatial-indexing/) for more info.

![](/public/img/ca_san_mateo.png)
*Legis-graph-spatial. Explore legislator topics of influence by district.*

# Simplified Congressional District Boundaries

Previously I was using a much higher resolution version of the congressional district boundaries then was necessary. I took this opportunity to replace the WKT boundaries for each district using this lower resolution data from [Govtrack.](https://gis.govtrack.us/map/demo/cd-2012/).

This Python script crawls the gis.govtrack.us API to fetch WKT format boundaries for each congressional district and saves it in a CSV file that we can later import into Neo4j easily as part of the [LOAD CSV import query for legis-graph](https://github.com/legis-graph/legis-graph/blob/master/quickstart/114/legis_graph_import_114.cypher):


{% highlight python %}

'''
Fetch simplified WKT boundaries for 2014 congressional districts and
save in CSV format:
    state,district,polygon

'''
import requests
import csv

BASE_URL = "https://gis.govtrack.us"
CD_2014_URL = "/boundaries/cd-2014/?limit=500"


# get meta boundary
r = requests.get(BASE_URL + CD_2014_URL)
j = r.json()

boundaries = j['objects']

with open('cb_2014_districts.csv', 'w') as f:
    writer = csv.writer(f)
    writer.writerow(['state', 'district', 'polygon'])

    for b in boundaries:
        p = str.split(b['name'], '-')
        r = requests.get(BASE_URL + b['url'] + 'simple_shape?format=wkt')
        wkt = r.text
        writer.writerow([p[0], p[1], wkt])

{% endhighlight %}

# Upgrading And Installing Neo4j Spatial

The server for legis-graph-spatial is running on a Digital Ocean vps instance, so I upgraded to Neo4j 3.0.4 using the Neo4j [Debian package](http://debian.neo4j.org/) which will install Neo4j as a service. Once I installed Neo4j I built and installed spatial:

{% highlight shell %}

git clone https://github.com/neo4j-contrib/spatial.git
cd spatial
mvn clean install -DskipTests
cp target/neo4j-spatial-0.20-neo4j-3.0.3-server-plugin.jar $NEO4J_HOME/plugins/.
$NEO4J_HOME/bin/neo4j restart

{% endhighlight %}

Then I used the [legis-graph import script](https://github.com/legis-graph/legis-graph/blob/master/quickstart/114/legis_graph_import_114.cypher) with the [LazyWebCypher tool for easily running multi statement Cypher scripts](http://www.lyonwj.com/LazyWebCypher/?file=https://raw.githubusercontent.com/legis-graph/legis-graph/master/quickstart/114/legis_graph_import_114.cypher) from the browser to load legis-graph data for the 114th Congress.

# Indexing Districts

Once the data is loaded we want to add the `District` nodes to a spatial index so we can do things like find the Congressional District that contains a certain latitude and longitude.

We'll use a spatial procedure, `spatial.addWKTLayer` to create a WKT layer:

{% highlight cypher %}
call spatial.addWKTLayer('geom', 'wkt')
{% endhighlight %}

Then we'll match on all `District` nodes and add them to the WKT layer we just created (again using a spatial procedure, in this case `spatial.addNodes`):

{% highlight cypher %}
MATCH (d:District)
WITH collect(d) AS districts
CALL spatial.addNodes('geom', districts) YIELD node
RETURN count(*)
{% endhighlight %}

# Querying

Now that the `District` nodes have been added to the spatial layer, we can perform an indexed geospatial query to find the Congressional district given a latitude and longitude. And since these `District` nodes are part of [legis-graph](https://github.com/legis-graph/legis-graph), it's just a simple graph traversal to find the Legislator that represents that District and their topics of influence:

{% highlight cypher %}

CALL spatial.withinDistance('geom',
  {latitude: 37.563440, longitude: -122.322265}, 1) YIELD node AS d
WITH d, d.wkt AS wkt, d.state AS state, d.district AS district LIMIT 1
MATCH (d)<-[:REPRESENTS]-(l:Legislator)
MATCH (l)-[:SERVES_ON]->(c:Committee)
MATCH (c)<-[:REFERRED_TO]-(b:Bill)
MATCH (b)-[:DEALS_WITH]->(s:Subject)
WITH wkt, state, district, l.govtrackID AS govtrackID, l.lastName AS lastName,
  l.firstName AS firstName, l.currentParty AS party, s.title AS subject,
  count(*) AS strength, collect(DISTINCT c.name) AS committees
ORDER BY strength DESC LIMIT 10
WITH wkt, state, district, {lastName: lastName, firstName: firstName,
  govtrackID: govtrackID, party: party, committees: committees} AS legislator,
  collect({subject: subject, strength: strength}) AS subjects
RETURN {legislator: legislator, subjects: subjects, state: state,
  district: district} AS r, wkt LIMIT 1

{% endhighlight %}

![](/public/img/spatial_query_table.png)
*Querying legis-graph using indexed geospatial query from Cypher*

# Using the official Neo4j Javascript Driver

Previously I was making an http request to the spatial endpoint to query the district and another to the Cypher transactional endpoint to query for topics of influence, but now we can combine this all into one query as seen above.

Also, we were using a jQuery AJAX POST request to the transaction Cypher HTTP endpoint to execute our Cypher query:

{% highlight javascript %}
 /**
     *  Run a Cypher query, call the callback with results
     */
    function makeCypherRequest(statements, callback) {

        var url = baseURI.replace(/\/db\/data.*/,"") + "/db/data/transaction/commit";

        $.ajax({
            type: 'POST',
            data: JSON.stringify({
                statements: statements
            }),
            contentType: 'application/json',
            url: url,
            error: function(xhr, statusText, errorThrown){
                callback("Error", null);
            },
            //headers: authHeader(),
            success: function(data) {
                console.log(data);
                callback(null, data);

            }
        });
    }
{% endhighlight %}



 With the release of Neo4j 3.0 we can instead make use of the new Bolt binary protocol to send less data over the wire and save time on serialization. For this we'll send our query using the [neo4j-javascript-driver](https://github.com/neo4j/neo4j-javascript-driver).

The neo4j-javascript-driver has a version for node.js based applications and a version for the browser. We can include the browser version in our simple app like this:

{% highlight html %}
<script src="/lib/neo4j-web.min.js"></script>
{% endhighlight %}

Then to execute the Cypher query we saw above that does our spatial lookup and parse the data:

{% highlight javascript %}
var driver = neo4j.v1.driver(neo4jURI);
var session = driver.session();

var districtParams = {
    "layer": "geom",
    "lon": latlng.lng,
    "lat": latlng.lat,
    "distanceInKm": distance
}

session
    .run(subjectsQuery, districtParams)
    .then(function(result){
        result.records.forEach(function(record) {
            console.log(record);

            var points = parseWKTPolygon(record.get("wkt"));
            var districtInfo = record.get("r");
            districtInfo["points"] = points;

            addDistrictToMap(districtInfo, latlng);

        });
});

{% endhighlight %}

As you can see we are using a promise for dealing with the asynchronous behavior of a database request. Alternatively we could act on the data as a stream, by subscribing to the data event:

{% highlight javascript %}
session
  .run(subjectsQuery, districtParams)
  .subscribe({
    onNext: function(record) {
     console.log(record);

     var points = parseWKTPolygon(record.get("wkt"));
     var districtInfo = record.get("r");
     districtInfo["points"] = points;
     addDistrictToMap(districtInfo, latlng);
    }
  });
{% endhighlight %}

However since we only expect to receive one row back we don't need to process as a stream.

You can give the new legis-graph-spatial version a try [here](http://legis-graph.github.io/legis-graph-spatial/) and as always the full code is available on [Github.](http://github.com/legis-graph/legis-graph-spatial)

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@lyonwj">
<meta name="twitter:title" content="Using Neo4j Spatial Procedures in legis-graph-spatial">
<meta name="twitter:description" content="Updating legis-graph-spatial to make use of the new spatial procedures in Neo4j 3.0 and the official Neo4j Javascript driver. Procedures provide a new API for interacting with the Neo4j spatial extension and are callable from Cypher.">
<meta name="twitter:creator" content="@lyonwj">
<meta name="twitter:image:src" content="http://www.lyonwj.com/public/img/spatial_query.png">
<meta name="twitter:domain" content="lyonwj.com">
