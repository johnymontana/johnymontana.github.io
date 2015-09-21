---
layout: post
title: Notes From GraphConnect 2014
introtext: GraphConnect is an annual conference for graph database enthusiasts. These are my notes from GraphConnect 2014 in San Francisco.
permalink: notes-from-graphconnect-2014
mainimage: /public/img/IMG_1437.png
---

![Graph connect stage photo](/public/img/IMG_1437.png)


# GraphConnect 2014 - San Francisco

GraphConnect is Neo Technology's annual Graph Database conference. Many of the talks highlight Neo4j use-cases, but many of the concepts are applicable to graph data modeling and graph databases in general.

## Advanced Cypher training class

The day before the conference I was able to take the Advanced Cypher training course. This course was a great in-depth dive into Cypher and I was happy that the course lived up to its name, "Advanced" Cypher. [Wes Freeman](https://twitter.com/wefreema) did a great job leading the course. Thanks, Wes!


## Graph Hack

The evening before GraphConnect we participated in Graph Hack, a hackathon focused on building Neo4j apps using a handful of open source datasets put together by [Nicole White](https://twitter.com/_nicolemargaret). The datasets consisted of flight and plane data, music playlists, and public transportation data. You can find them available [here](https://gist.github.com/nicolewhite/cc178bf2a761d7ac3a20) if you'd like to play around with the data. My team built a simple command line app that computed the probability of your flight being delayed given the origin, destination, airline, day of flying and type of aircraft.

**The following are my notes from the conference:**

## Keynote: Graphs Are Eating The World

### Emil Eifrem (CEO, Neo Technology)

![](/public/img/IMG_1439.png)


* The internet of **connected** things
* Graphs are eating the world
* Graph DBs are fastest growing NoSQL segment
* Example social query
	* average 50 friends, see if two random users are connected
	* 2ms Neo4j (1000 or 1000000) users
	* 2s with MySQL (just 1000 friends)
* Neo4j 2.2 - performance and scale
	* initial import performance - millions of inserts per second
	* cypher performance improvements
	* concurrent writes - ACID write to disk (currently sequential)
		* batch / concurrent writes is coming in 2.2
		* 100x faster for large writes

## The Business Graph

### Kurt Freytag (Head of Product, CrunchBase )

* CrunchBase started in 2007 with 2 crappy AWS servers (1 MySQL, 1 Rails)
* Power of crowd sourced data
	* Evidence that people want it
	* Entities, activities, connections, time
	* The business lifecycle
* Graph is a natural way of modeling data
* Added Events
	* announced at TCDisrupt London
	* built in 2 weeks (!) because of Neo4j. This would have taken months with a RDBMS
* Graph data model adapts easily to changing requirements
* Built in Business Intelligence
	* Don't know what questions to ask in advance (at data modeling time)
	* OLAP alongside graph database, graphDB for ad-hoc queries
* Directly maps to Object Oriented thinking (OGM)
	* Nodes = objects, etc.
* Why doesn't everybody use Neo4j?
	* Historical reasons
		* Databases are used as dumb data stores
		* Logic / analysis is in application layer, focus on logic not data
	* Not plug and play
		* not supported by many high level languages
		* have to think about data modeling
		* CrunchBase developed Deja, a Ruby ORM for Neo4j
* Graph Insights feature on CrunchBase would not be possible without a graph database

## Using Graphs for Next-Gen Master Data Management at Pitney Bowes

### Aaron Wallace (Principal Product Manager, Pitney Bowes)

Slides: http://www.slideshare.net/neo4j/using-graphs-for-nextgen-master-data-management-at-pitney-bowes

* Integrate the data silos
* Single view of individual and connections and interactions (transactions)
* Insights / predictive analytics are now critical for companies
* Visual schema managment
* Visual query builder
* Graph database are peaking on the Gartner "Hype Cycle"
* Pull data from data silos and building the knowledge graph
* Easily expose custom built queries as a web service (REST/SOAP)

## From Zero To Graph In 120: Query

### Nicole White (Data Scientist, Neo Technology)

* Flight data set
* How to use Cypher to query this data set
* For performance, filter by cardinality first
	* Example query went from 1500+ms to 198ms

## From Zero To Graph In 120: Scale

### Ian Robinson (Engineer, Neo Technology)

![](/public/img/IMG_1446.png)


Slides: http://www.slideshare.net/neo4j/scaling-neo4jv1

* Understand current needs
* Iterative and incremental development
* Unit testing
* Performance tests

### Scaling Reads - Latency

* Latency is a function of search area
* Search area is a function of of domain invariants
* Domain invariants
	* absolute - (constant query latency x=50)
	* relative - x% of data set
		* query latency increases as data grows
* How to reduce latency?
	* Change domain invariants (often not possible)
	* Improve Cypher query
		* Small queries
		* Separate using **WITH** statements
		* Filter lower cardinality first
	* Change model
		* Explore less of the graph (reduce search area)
		* Using inferred (implicit) relationships?
			* replace inferred relationships with actual relationships (edges in the graph)
			* Cheaper reads, but more expensive write
			* More data to store
			* When to update these? On insertion (same transaction)? Batch updates?
	* Use unmanaged server extensions (custom API)
		* Write Java code that runs in the server
		* Benefits
			* closer to the data
			* single request, many operations
			* multiple implementation options
				* Cypher, Traversal API, Graph Algorithms, Java API
			* control request / response format - compact, conserve bandwidth
			* control HTTP header - cache control

### Scaling Reads - Throughput

* Scale horizontally - cluster
* HAProxy as read load balancer
* Object cache data is evicted
	* Solved with cache sharing (consistent routing)
		* Add parameter in URI for consistent routing, HAProxy config

### Scaling Writes

* Contention for locks
	* Lock nodes at last moment
	* Batch writes
		* Introduce a queue layer. Can add queue inside server extension

## Panel: Visualizing Graphs

### Cambridge Intelligence, Tom Sawyer Software, Linkurious and GraphAlchemist

![](/public/img/IMG_1447.png)


* People don't realize graphs are powering stuff (Facebook, recommendations, etc)
* Find one dimension that holds data together (ex: maps and location), but with graphs can have unlimited dimensions. Find one dimension for visualization to make sense.
* Groups / clustering. Show how groups are related, not individuals
* Trend: specific products / implementations for specific use cases
* Alternative to circles and lines?
	* island approach. remove edges
	* reduce number of edges for best visuals
	* matrix representation
* Every aspect of the visualization can be bound to the data (color, width, etc)
* 3D visualizations?
	* technology is not quite ready yet
	* 2.5D shows some promise


## Betting The Company On A Graph Database - Part 2

### Aseem Kishore (Developer, FiftyThree)

Slides available [here](http://aseemk.com/talks/mix-neo4j).

![](/public/img/IMG_1448.png)


* Mix by 53
	* Platform for collaborative creation
* Relationship direction choice
	* Point to the thing that the other depends on
* Use linked list in-graph for better performance
* Streams - profile stream
* Pagination
	* Must deal with changing data
	* Instead of skip, use cursor time and filter based on that
* Home stream (with recommendations)
* Deduplication
	* **WHERE NOT** is expensive
	* Use **PROFILE** to profile query plan
* Logarithmic threshold (???)

## Neo4j At Scale Using Enterprise Integration Patterns

### Brad Nussbaum (CTO, MediaHound)

Slides: http://www.slideshare.net/neo4j/media-hound-graphconnectfinal

![](/public/img/IMG_1450.png)


* Mediahound is building the entertainment graph
* Collaborative filtering for recommendations
* Graph influencers
* Requirement: sustained user write load and handle batch updates
* Use transactional cypher endpoint
	* Best for batch updates
* Writing a single relationship is 33 byte write to disk
* As of 2.2 Neo4j will batch writes on server

### Enterprise integration patterns

* Splitters - break down larger messages
* Aggregators - batching
* Throttling - limit concurrent writes. Use Neo4j status codes
* Push factor - how many replications must be written to for a successful return
* Cypher is the standard
	* Check driver for transaction support
* **"USING INDEX"** hint
* Recommendation processing with AWS spot instances
* Server extensions
	* Cache abstraction with Google Guava. Large in-memory caching.

### AWS Sport Instance Grid Processing

* Spot instances are bid-based and can be killed at any time if out bid
* Used for concurrent job processing
* Use Cypher statements that are not dependent
* Compound statements for Node/Relationship creation for the subgraph

### ESB

* 400 to 2000 updates per transaction is optimal
* Insert nodes -> single lock
* Insert relationships -> 2 node lock, so do node insertions first

## Applying The GraphAware Framework

### Michal Bachman (Director, GraphAware)

Slides: http://www.slideshare.net/neo4j/graphconnect-2014-sf

@graph_aware

@bachmanm

* GraphAware combines common patterns into a framework
* [Open sourced](http://graphaware.com/)

### Advanced Use Cases

* Custom APIs
* Transaction driven behavior
* Async / background processing

### Custom APIs

* access to Java API for improved performance
* Locking - selective locking
* use-case too complex for Cypher
* Custom logic
* limit to read only
* custom I/O format
* code to data distance
* GraphAware
	* Spring MVC
	* testing with GraphUnit
* Demo
	* Time hierarchy with GA TimeTree
	* `/timetree/now?resolution=second` - creates hierarchy
	* `/timetree/range...` - query based on range


### Transaction Driven Behavior

* ACID
* Integration with other systems
* in-graph indexing
* additional modifications per transaction
* schema enforcement
* Demo
	* Track changes in graph
	* Assign UUIDs

### Async / Background computation

* Crawlers
* Neo4j is very good at OLTP
* Graph algorithms can be expensive in real-time, but many can be approximated
* Execute computation during quiet periods
	* Adapts as usage changes
* PageRank (random walk, incremental counter)
* Recommendation
* Similarities
* Centralities
* Statistics
* Demo
	* Random graph generation
	* run NodeRank in background
* Recommendation system is WIP for GraphAware


## Keynote: Impossible Is Nothing With Graphs

### Jim Webber (Chief Scientist, Neo Technology)


![](/public/img/IMG_1456.png)


* Graph databases revolutionize data
* Reliabiliity, availability, eventual consistency
* As Neo4j is becoming more aggressive, how is this impacting the technology?

### History of data

* 70s
	* databases reflected squareness of data
* 80s
	> Databases designed for boring payroll processing, not your geospatial, social, mating mashup.

* 90s
	* Data got big
	* The Web! Graph data at scale, but nobody realized it because they were too busy dealing with app servers and ORMs
	* "The Vietnam decade of computer science"
* 00s
	* Large web properties building their own databases
	* Scale - NoSQL to the rescue!
	* Most NoSQL DBs deal with aggregate (unconnected) data
	* MapReduce to find connections
	* With graph database connections are first class citizens
	* Richer model with connected data

### Graph Database Use-Cases

* Telenor - manage hierarchy of customers
* Beamly - second screen startup. Metadata about TV
* eBay Now - quick routing
	* Before Neo4j, had queries longer than their shortest deliveries
* Accenture - saved Christmas with improvements on parcel delivery routing
* CrunchBase - the Business Graph


* Neo4j is still a small (scrappy) startup


### Reliability vs Availability

* Lee-Anderson - Fault Tolerance
* Dependability
	* Tradeoff between availability (uptime) and reliability (probability to produce correct output)
	* Integrity
	* Safety
	* Confidentiality
* Should the system be available or reliable?
	* depends
	* Aggregate database -> agree on a single value
	* Graph database -> 2 nodes in relationship both have to agree
	* For Graphs: reliability over availability

### What's Next?

* Bigger data (trillions of nodes)
* Smarter Cypher, learn from previous queries
* More throughput
* How is this possible?
	* Native graph all the way down
		* Optimize everywhere

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@lyonwj">
<meta name="twitter:title" content="Notes From GraphConnect 2014">
<meta name="twitter:description" content="My notes from Neo Technology's GraphConnect 2014 conference in San Francisco.">
<meta name="twitter:creator" content="@lyonwj">
<meta name="twitter:image:src" content="http://lyonwj.com/content/images/2014/10/IMG_1437.png">
<meta name="twitter:domain" content="lyonwj.com">
