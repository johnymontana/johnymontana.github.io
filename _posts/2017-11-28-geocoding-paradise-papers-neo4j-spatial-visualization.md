---
layout: post
title: Geocoding Paradise Papers Addresses In Neo4j To Build Interactive Geographical Data Visualizations
introtext: This post explores how to build spatial data visualizations using address data from the Paradise Papers leak of offshore corporations and the people connected to them. First, we geocode all addresses in the leaked data, then build a heatmap and interactive map for exploring the data of offshore legal entities.
mainimage: /public/img/ppviz/heatmap1.png
---
This post explores how to build spatial data visualizations using address data from the Paradise Papers leak of offshore corporations and the people connected to them. First, we geocode all addresses in the leaked data, then build a heatmap and interactive map for exploring the data of offshore legal entities.

![](/public/img/ppviz/heatmap1.png){: .center-image}

The [Paradise Papers](https://www.icij.org/investigations/paradise-papers/) is the latest data journalism investigation from the [International Consortium of Investigative Journalists (ICIJ)](https://www.icij.org). Like anyone working with complex and highly connected data (think of a chain of ownership of offshore legal entities) the ICIJ use [Neo4j](https://neo4j.com/) to find connections in the data. You can read more about how graph databases and Neo4j were used by the ICIJ for the Paradise Papers investigation [here](https://neo4j.com/blog/analyzing-paradise-papers-neo4j/) and [here](https://neo4j.com/blog/depth-graph-analysis-paradise-papers/).

## The Data

Anyone can download the Paradise Papers data as a Neo4j database from the [ICIJ's Offshore Leaks site](https://offshoreleaks.icij.org/pages/database).

Here's what the data model looks like:

![](/public/img/ppviz/datamodel.jpg)

As you can see each `Officer`, `Entity`, and `Intermediary` node is connected to an `Address` node through a `REGISTERED_ADDRESS` relationship. In the case of the Paradise Papers dataset, this address was found in the leaked documents or corporate registry.

I'm particularly interested in analyzing the data using this geographic component. We can use Cypher and Neo4j to ask questions of the data. For example, for Officers with an address in San Mateo, what are the most popular offshore jurisdictions for their legal entities?

{% highlight cypher %}
MATCH (a:Address)<-[:REGISTERED_ADDRESS]-(o:Officer)-[:CONNECTED_TO|OFFICER_OF]->(e:Entity)
WHERE a.name CONTAINS "San Mateo"
RETURN e.jurisdiction_description AS juris, COUNT(*) AS num
ORDER BY num DESC
{% endhighlight %}

~~~
╒════════════════╤═════╕
│"juris"         │"num"│
╞════════════════╪═════╡
│"Bermuda"       │90   │
├────────────────┼─────┤
│"Cayman Islands"│14   │
├────────────────┼─────┤
│"Mauritius"     │1    │
├────────────────┼─────┤
│"Seychelles"    │1    │
└────────────────┴─────┘
~~~

Since we have the addresses, we can use a process called geocoding to find the latitude and longitude for each address, enabling us to build spatial data visualizations for exploring the data.

## Geocoding Addresses

According to Wikipedia:

> Geocoding is the computational process of transforming a postal address description to a location on the Earth's surface (spatial representation in numerical coordinates)

There are several services that provide geocoding, including [Google's Geocoding API](https://developers.google.com/maps/documentation/geocoding/start) and [Open Street Maps' Nominatim](http://wiki.openstreetmap.org/wiki/Nominatim).

Thanks to Neo4j's spatial guru [Craig](https://twitter.com/craigtaverner) we can use both of these geocoding services directly from Cypher as part of the [APOC Cypher procedure library](
https://neo4j-contrib.github.io/neo4j-apoc-procedures/#_geocode).

Once you've installed APOC there are a few settings to add to neo4j.conf:


{% highlight shell %}
dbms.security.procedures.unrestricted=apoc.*


apoc.spatial.geocode.provider=google
apoc.spatial.geocode.osm.throttle=1000 # be sure to throttle if using the free OSM API
apoc.spatial.geocode.google.throttle=0 # no need to throttle if using the PAID Google API
apoc.spatial.geocode.google.key=INSERT_GOOGLE

{% endhighlight %}


You can geocode a single address using the `apoc.spatial.geocodeOnce` procedure:

{%  highlight cypher %}
CALL apoc.spatial.geocodeOnce('901 Trophy Hills Drive; Las Vegas; Nevada  89134; United States of America')
{% endhighlight %}


~~~
{
  "formatted_address": "901 Trophy Hills Dr, Las Vegas, NV 89134, USA",
  "geometry": {
    "viewport": {
      "southwest": {
        "lng": -115.2948439802915,
        "lat": 36.1811390197085
      },
      "northeast": {
        "lng": -115.2921460197085,
        "lat": 36.1838369802915
      }
    },
    "location": {
      "lng": -115.293495,
      "lat": 36.182488
    },
    "location_type": "ROOFTOP"
  },
  "types": [
    "street_address"
  ],
  "address_components": [
    {
      "long_name": "901",
      "short_name": "901",
      "types": [
        "street_number"
      ]
    },
    {
      "long_name": "Trophy Hills Drive",
      "short_name": "Trophy Hills Dr",
      "types": [
        "route"
      ]
    },
    {
      "long_name": "Summerlin",
      "short_name": "Summerlin",
      "types": [
        "neighborhood",
        "political"
      ]
    },
    {
      "long_name": "Las Vegas",
      "short_name": "Las Vegas",
      "types": [
        "locality",
        "political"
      ]
    },
    {
      "long_name": "Clark County",
      "short_name": "Clark County",
      "types": [
        "administrative_area_level_2",
        "political"
      ]
    },
    {
      "long_name": "Nevada",
      "short_name": "NV",
      "types": [
        "administrative_area_level_1",
        "political"
      ]
    },
    {
      "long_name": "United States",
      "short_name": "US",
      "types": [
        "country",
        "political"
      ]
    },
    {
      "long_name": "89134",
      "short_name": "89134",
      "types": [
        "postal_code"
      ]
    },
    {
      "long_name": "0567",
      "short_name": "0567",
      "types": [
        "postal_code_suffix"
      ]
    }
  ],
  "place_id": "ChIJ412KJ_K_yIARxCzkI0IN7Ho"
}
~~~


We're only interested in the latitude and longitude, which we can grab from the `location` column:

{% highlight cypher %}
CALL apoc.spatial.geocodeOnce('901 Trophy Hills Drive; Las Vegas; Nevada  89134; United States of America') YIELD location
{% endhighlight %}

~~~
{
  "description": "901 Trophy Hills Dr, Las Vegas, NV 89134, USA",
  "latitude": 36.182488,
  "longitude": -115.293495
}
~~~


So what we want to do is iterate through all the `Address` nodes in the Paradise Papers dataset, send the address text to the geocoding service and store three additional properties on the `Address` node: `latitude`, `longitude`, and `description`:

{% highlight cypher %}
MATCH (a:Address) WITH a LIMIT 1
CALL apoc.spatial.geocodeOnce(a.address) YIELD location
WITH a, location.latitude AS latitude, location.longitude AS longitude,
  location.description AS description
SET a.latitude = latitude,
    a.longitude = longitude,
    a.description = description
{% endhighlight %}

This works, but we have ~65k addresses to process and we're making one request at a time here. This results in about 1 request per second.

![](https://i.imgflip.com/2052xu.jpg){: .center-image}


If we are using the paid Google geocoding API then we have a rate limit of 50 requests/second, so let's take advantage of that!

We can use [`apoc.periodic.iterate`](https://neo4j-contrib.github.io/neo4j-apoc-procedures/#_apoc_periodic_iterate) to parallelize our requests. `periodic.iterate` takes two Cypher queries, runs the first query passing the results to the second query, which is then run in batches. Optionally, we can choose to run those batches in parallel.

**NOTE: This approach should only be used for the paid Google goeocoding API, otherwise you will exceed rate limits!**

{% highlight cypher %}
CALL apoc.periodic.iterate('MATCH (a:Address) RETURN a',
'CALL apoc.spatial.geocodeOnce(a.address) YIELD location
  WITH a, location.latitude AS latitude, location.longitude AS longitude,
  location.description AS description
SET a.latitude = latitude,
    a.longitude = longitude,
    a.description = description', {batchSize:100, parallel:true})
{% endhighlight %}


By executing our geocode procedure in parallel batches we can increase our throughput, significantly:

![](/public/img/ppviz/throughput1.png)


Once we've geocoded our addresses we can do things like calculate distance using Cypher.

*What are all the addresses in the dataset within 2km, and what offshore legal entities are they associated with?*

{% highlight cypher %}
MATCH (e:Entity)<-[:CONNECTED_TO|OFFICER_OF]-(o:Officer)-[:REGISTERED_ADDRESS]->(a:Address)
WHERE exists(a.latitude) AND exists(a.longitude)
AND DISTANCE(POINT(a), POINT({latitude: 37.5634399, longitude:-122.3244533})) < 2000
RETURN DISTINCT e.name AS offshore_entity, e.jurisdiction_description AS jurisdiction, a.name AS officer_address
{% endhighlight %}

~~~

╒══════════════════════════════╤════════════════╤══════════════════════════════╕
│"offshore_entity"             │"jurisdiction"  │"officer_address"             │
╞══════════════════════════════╪════════════════╪══════════════════════════════╡
│"AutoChina International Limit│"Cayman Islands"│"605 Hobart Ave; San Mateo; 94│
│ed"                           │                │402 California; United States │
│                              │                │of America"                   │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"Mill River Re Limited"       │"Bermuda"       │"160 Bovet Road; San Mateo; CA│
│                              │                │ 94402; United States of Ameri│
│                              │                │ca"                           │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"Indigo Reinsurance Ltd."     │"Bermuda"       │"160 Bovet Road; San Mateo; CA│
│                              │                │ 94402; United States of Ameri│
│                              │                │ca"                           │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"Asurion Japan Reinsurance Ltd│"Bermuda"       │"160 Bovet Road; San Mateo; CA│
│."                            │                │ 94402; United States of Ameri│
│                              │                │ca"                           │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"International Wireless Commun│"Bermuda"       │"400 S El Camino Real; Suite 5│
│ications Latin America Holding│                │80; San Mateo; CA 94402; Unite│
│s Ltd."                       │                │d States of America"          │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"Renters Reinsurance Company I│"Bermuda"       │"108 Chukker Court; San Mateo │
│I, Ltd."                      │                │CA  94403; United States of Am│
│                              │                │erica"                        │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"D&H Reinsurance Company, Ltd.│"Bermuda"       │"108 Chukker Court; San Mateo │
│"                             │                │CA  94403; United States of Am│
│                              │                │erica"                        │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"Farallon Reinsurance Company,│"Bermuda"       │"108 Chukker Court; San Mateo │
│ Ltd."                        │                │CA  94403; United States of Am│
│                              │                │erica"                        │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"H&F Rose Investors Ltd."     │"Bermuda"       │"133 Ridgeway Road; Hillsborou│
│                              │                │gh California  94010; United S│
│                              │                │tates of America"             │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"H&F International Rose Invest│"Bermuda"       │"133 Ridgeway Road; Hillsborou│
│ors Ltd."                     │                │gh California  94010; United S│
│                              │                │tates of America"             │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"H&F Corporate Investors IV (B│"Bermuda"       │"133 Ridgeway Road; Hillsborou│
│ermuda) Ltd."                 │                │gh California  94010; United S│
│                              │                │tates of America"             │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"Celtic Pharma Development Ser│"Bermuda"       │"109 Chukker Ct; San Mateo; CA│
│vices Bermuda Ltd."           │                │ 94403; United States of Ameri│
│                              │                │ca"                           │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"Mill River Re Limited"       │"Bermuda"       │"1700 South El Camino Real; Su│
│                              │                │ite 200; San Mateo; California│
│                              │                │ 94402; United States of Ameri│
│                              │                │ca"                           │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
│"Indigo Reinsurance Ltd."     │"Bermuda"       │"1700 South El Camino Real; Su│
│                              │                │ite 200; San Mateo; California│
│                              │                │ 94402; United States of Ameri│
│                              │                │ca"                           │
├──────────────────────────────┼────────────────┼──────────────────────────────┤
~~~


but we can also build really cool interactive map based visualizations with our data!

## Building An Interactive Geographical Data Visualization

Using Leaflet.js to build an interactive geographic visualization, we want to pull data from Neo4j and annotate a map. To do this we'll make use of Leaflet.js (and a couple of plugins!), and the Javascript driver for Neo4j.

### Leaflet.js

[Leaflet.js](http://leafletjs.com/) is an open-source JavaScript library for creating interactive maps. We can create a simple map like this:

{% highlight html %}

<html>
<head>
  <style>
    #map {height: 100%;}
  </style>

  <script src="http://cdn.leafletjs.com/leaflet-0.4.4/leaflet.js"></script>
  <link rel="stylesheet" href="http://cdn.leafletjs.com/leaflet-0.4.4/leaflet.css" />
</head>

  <body>
    <div id="map"></div>

    <script>
      var baseLayer = L.tileLayer('http://{s}.tile.osm.org/{z}/{x}/{y}.png',
        { attribution: 'OpenStreetMap', minZoom: 2, maxZoom: 18});

      var map = new L.Map('map', {center: new L.LatLng(37.5634399, -122.3244533), zoom: 17, layers: [baseLayer]});
    </script>

  </body>
</html>

{% endhighlight %}

This gives us a map:

![](/public/img/ppviz/map1.png)

now let's add some data to it! Leaflet allows us to create multiple layers and show/hide various layers. So we create a base map layer (our map tiles) and our data visualizations live in map layers that are plotted on top of the map tiles.

We'll make use of two types of geographical data visualizations - a heat map and marker clusters.


### Marker Clusters

Adding markers to our map is very simple in Leaflet, but if we add a marker for each address in the Paradise Papers dataset our map will soon look just like a pin cushion. Fortunately, the [MarkerCluster plugin for Leaflet](https://github.com/Leaflet/Leaflet.markercluster) includes logic for clustering markers together, depending on zoom level. Here's an example of how we can add a few markers to our Leaflet map:


{% highlight javascript %}

var markerLayer = new L.MarkerClusterGroup();

var marker1 = new L.Marker(new L.LatLng(37.5634399, -122.3222646));
marker1.bindPopup('Neo4j HQ');

var marker2 = new L.Marker(new L.LatLng(37.5669493, -122.3237035));
marker2.bindPopup('Philz Coffee');

var marker3 = new L.Marker(new L.LatLng(37.5622308, -122.3268279));
marker3.bindPopup('San Mateo Public Library');

markerLayer.addLayer(marker1);
markerLayer.addLayer(marker2);
markerLayer.addLayer(marker3);
map.addLayer(markerLayer);

{% endhighlight %}

Our markers are clustered together when zoomed out:

![](/public/img/ppviz/map2.png)

But once we zoom in we can see each individual marker:

![](/public/img/ppviz/map3.png)

### Heatmap

A heatmap is a data visualization (often imposed on a map, but could be done on a matrix as well) where colors are used to represent data values. In a geographical data visualization, where data points are often sparse, some form of interpolation is often used.

There are a [few different Leaflet heatmap plugins](http://leafletjs.com/plugins.html#heatmaps ) we can choose from but I like [heatcanvas](https://github.com/sunng87/heatcanvas) as its API provides several configuration options for configuring the formula used to determine the value of a pixel.

Here's how we can add a simple heatmap to our Leaflet map using heatcanvas:


{% highlight javascript %}

var heatmap = new L.TileLayer.HeatCanvas({}, {'step':0.3, 'degree': HeatCanvas.LINEAR, 'opacity': 0.7});
heatmap.pushData(37.5634399, -122.3222646, 10.0); // Neo4j HQ
heatmap.pushData(37.5669493, -122.3237035, 20.0); // Philz Coffee
heatmap.pushData(37.5622308, -122.3268279, 12.0); // Public Library
map.addLayer(heatmap);

{% endhighlight %}

![](/public/img/ppviz/map4.png)
*And our heatmap shows that I spend much more time at the coffeeshop and library than I do at work ;-)*

### Using Data From Neo4j

Now that we've seen how to create a map using Leaflet, adding markers, marker clusters, and a heatmap let's see how we can pull data out of Neo4j to populate our map. Since we previously added `latitude` and `longitude` properties to our `Address` nodes we'll use those to mark points on our map, but we'd also like to include information about the `Officer` connected to that address and the legal entities the officer is connected to. This Cypher query will find all registered addresses of Officers in the Paradise Papers dataset (that we were able to geocode), as well as all their connected offshore legal entities:


{% highlight cypher %}
MATCH (a:Address)<-[:REGISTERED_ADDRESS]-(o:Officer)-[:CONNECTED_TO|OFFICER_OF]-(e:Entity)
  WHERE exists(a.latitude) and exists(a.longitude)
RETURN a.name AS address, a.latitude AS latitude, a.longitude AS longitude,
  COLLECT(DISTINCT o.name) AS officers, COLLECT(DISTINCT e.name) AS entities,
  COLLECT(DISTINCT e.jurisdiction_description) AS jurisdictions, 1.0*COUNT(*) AS strength
{% endhighlight %}

We can make use of [the JavaScript driver for Neo4j](https://github.com/neo4j/neo4j-javascript-driver) to run this query, and add data to the map for each record returned:

{% highlight javascript %}

var driver = neo4j.v1.driver("bolt://localhost:7687", neo4j.v1.auth.basic("neo4j", "MYPASSWORD"));
var session = driver.session();
session
  .run(`MATCH (a:Address)<-[:REGISTERED_ADDRESS]-(o:Officer)-[:CONNECTED_TO|OFFICER_OF]-(e:Entity)
        WHERE exists(a.latitude) and exists(a.longitude)
        RETURN a.name AS address, a.latitude AS latitude, a.longitude AS longitude,
          COLLECT(DISTINCT o.name) AS officers, COLLECT(DISTINCT e.name) AS entities,
          COLLECT(DISTINCT e.jurisdiction_description) AS jurisdictions, 1.0*COUNT(*) AS strength`)
  .subscribe({
    onNext: function (record) {
      //console.log(record);
      var marker = new L.Marker(new L.LatLng(record.get('latitude'), record.get('longitude')));
      marker.bindPopup('<b>Address:</b> ' + record.get('address') + '<br>' + '<b>Officers: </b>' + record.get('officers').toString() + '<br><b>Entities:</b> ' + record.get('entities').toString() + '<br><b>Offshore Countries: </b> ' + record.get('jurisdictions').toString());
      markerLayers.addLayer(marker);
      heatmap.pushData(record.get('latitude'), record.get('longitude'), record.get('strength')*0.01);
    },
    onCompleted: function () {
      var overlayMaps = {'Markers': markerLayers, 'Heatmap': heatmap};
      var controls = L.control.layers(null, overlayMaps, {collapsed: false, autoZIndex: true});
      map = new L.Map('map', {center: new L.LatLng(51.505, -0.09), zoom: 3, layers: [baseLayer]});
      map.addLayer(markerLayers);
      map.addLayer(heatmap);
      controls.addTo(map);
      session.close();
    },
    onError: function (error) {
      console.log(error);
    }
  });


{% endhighlight %}


Now we have an interactive geographic visualization of addresses from the Paradise Papers dataset that we can explore.

Both the heatmap, to show us areas that have a high concentration of addresses associated with officers of offshore entities:

[![](/public/img/ppviz/heatmap1.png)](http://www.lyonwj.com/pp-viz/heatmap/)

And we can also explore the markers to find officers and connected offshore legal entities. Here we can see casino owner [Sheldon Adelson and his Bermuda-based company](https://projects.icij.org/paradise-papers/the-influencers/#/sheldon-adelson) used for registering private aircraft:

[![](/public/img/ppviz/marker.png)](http://www.lyonwj.com/pp-viz/heatmap/)


These type of geographical data visualizations are particularly useful for data journalists who are interested in finding stories relevant for a certain geographic area.


All code is available on [Github](https://github.com/johnymontana/pp-viz) and you can try the live map visualization online [here](http://www.lyonwj.com/pp-viz/heatmap/).

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@lyonwj">
<meta name="twitter:title" content="Geocoding Paradise Papers Addresses In Neo4j To Build Interactive Geographical Data Visualizations">
<meta name="twitter:description" content="This post explores how to build spatial data visualizations using address data from the Paradise Papers leak of offshore corporations and the people connected to them. First, we geocode all addresses in the leaked data, then build a heatmap and interactive map for exploring the data of offshore legal entities.">
<meta name="twitter:creator" content="@lyonwj">
<meta name="twitter:image:src" content="http://www.lyonwj.com/public/img/ppviz/heatmap1.png">
<meta name="twitter:domain" content="lyonwj.com">
