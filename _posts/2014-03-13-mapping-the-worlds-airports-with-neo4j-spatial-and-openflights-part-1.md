---
layout: post
title: Mapping the World's Airports With Neo4j Spatial and Openflights
introtext: Loading every airport in the world into Neo4j Spatial for the purposes of route finding.
permalink: mapping-the-worlds-airports-with-neo4j-spatial-and-openflights-part-1
mainimage: /public/img/Screen_Shot_2014_03_13_at_8_53_30_AM.png
---

![Neo4j Spatial Browser](/public/img/Screen_Shot_2014_03_13_at_8_53_30_AM.png)

When I was taking flight lessons, I learned that the maximum range of a [Cessna-172](http://en.wikipedia.org/wiki/Cessna_172) is about 1000km. Knowing this, **can we find a path across the globe where each airport in the path is within the range of a Cessna 172?** Let's use Neo4j Spatial to try to answer this question! This blog post will focus on loading data from [OpenFlights.org](http://openflights.org) into Neo4j and how we can use Neo4j Spatial to help us represent this data in a Neo4j graph database. The next post will show how we can use the Neo4j Spatial Java API to traverse our graph and answer this question.

## Neo4j Spatial
[Neo4j Spatial](https://github.com/neo4j/spatial) is a Neo4j Server plugin that allows for spatial operations within a Neo4j database. Spatial makes use of an in-graph [R-tree](http://en.wikipedia.org/wiki/R-tree) to create spatial indexes allowing for geographic queries in the context of Neo4j. For example, Spatial allows for queries to find nodes within a specified geographic region or within a certain distance of a specified point. Neo4j Spatial can easily be installed as a server extension. Pre-packaged ZIPs containing all the JARs needed are provided on the [Spatial GitHub page](https://github.com/neo4j/spatial). Alternatively, [GrapheneDB](http://graphenedb.com) provides Neo4j Spatial as a plugin option. Getting Spatial running on a GrapheneDB instance is literally as easy as a few clicks.

## OpenFlights.org Data
OpenFlights.org provides a [dataset](http://openflights.org/data.html) containing information about almost every airport in the world. This dataset includes 6977 airports, in CSV format:

* Airport ID
* Name
* City
* Country
* **IATA/FAA code**
* ICAO code
* **Latitude**
* **Longitude**
* Altitude
* Timezone
* DST

We are most interested in columns 4 (airport FAA code) and 6/7 (latitude/longitude). Here's what the row for my home airport, MSO, looks like:

<script src="https://gist.github.com/johnymontana/d208de69249dbd947655.js?file=MSO.csv"></script>

## Loading The Data
Like most Neo4j projects, we have several API choices for how we want to interact with the Neo4j Server. In the next post in this series we'll look at using the Spatial Java API, but for now we will use the REST interface. To make use of Spatial we will follow three steps:

1. Create a Spatial index
1. Create nodes with lat/lon data as properties
1. Add these nodes to the Spatial index

We also will create an index in Neo4j (with the same name as our Spatial layer) to that we can query the database using Cypher.


## Neo4j Spatial REST API
This Python script follows the steps outlined above and uses the REST interface to add each airport to the database.

<script src="https://gist.github.com/johnymontana/d208de69249dbd947655.js?file=openflightsneo.py"></script>

First, the Spatial index is created, specifing the geometry type and properties that will contain lat/lon data on each node to be added to that index. The file `airports.dat` is our OpenFlights CSV file listing all airports. We iterate through each row, creating a node for each airport, adding that node to the Spatial index and then to the index we have created so we can later query using Cypher.

## Finding Airports
All airports should now be loaded into our database. Now we can start to query the database using some Spatial features to find airports.
### Using Cypher
The following Cypher query will search for an airport within 50km of my hometown, Missoula, MT.

<script src="https://gist.github.com/johnymontana/d208de69249dbd947655.js?file=MSO.cypher"></script>

Running that in Neo4j we see:
![MSO query](/public/img/Screen_Shot_2014_03_13_at_8_29_02_AM.png)

Alright, looks like Missoula has an airport! Great!

### Using the REST interface
We can also make use of the REST interface to find all airports within a certain georgraphic area, or within a certain distance of a specified point.

The endpoint `/db/data/ext/SpatialPlugin/graphdb/findGeometriesWithinDistance` will allow us to query for all airports within 175km of Missoula, MT:

<script src="https://gist.github.com/johnymontana/d208de69249dbd947655.js?file=response.js"></script>

Looking at the response we see that MSO (Missoula, MT), BTM (Butte, MT), HLN (Helena, MT) and FCA (Kalispell, MT) are airports within 175km of Missoula.

## Looking Forward
That was a quick introduction to Neo4j Spatial. The next post will show how we can use the Neo4j Spatial Java API for more advanced traversals and answer our original question: Is it possible to fly a Cessna 172 across the world and what is the route?

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@lyonwj">
<meta name="twitter:title" content="Mapping The World's Airports With Neo4j Spatial And OpenFlights - Part 1">
<meta name="twitter:description" content="When I was taking flight lessons, I learned that the maximum range of a Cessna-172 is about 1000km. Knowing this, can we find a path across the globe where each airport in the path is within the range of a Cessna 172? Let's use Neo4j Spatial to try to answer this question! This blog post will focus on loading data from OpenFlights.org into Neo4j and how we can use Neo4j Spatial to help us represent this data in a Neo4j graph database. The next post will show how we can use the Neo4j Spatial Java API to traverse our graph and answer this question.">
<meta name="twitter:creator" content="@lyonwj">
<meta name="twitter:image:src" content="http://lyonwj.com/content/images/2014/Mar/Screen_Shot_2014_03_13_at_8_53_30_AM.png">
<meta name="twitter:domain" content="lyonwj.com">
