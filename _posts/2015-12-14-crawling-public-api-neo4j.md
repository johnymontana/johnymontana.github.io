---
layout: post
title: Crawling APIs Using Neo4j - Building The Wookieepedia Graph
introtext: A common task when working with data from APIs is crawling the API and inserting the results in a database. In this example we'll import data into Neo4j by crawling a public API, building a graph in Neo4j as we go along. We'll use Neo4j as a queuing mechanism to store URLs for placeholder resources waiting to be fetched.
mainimage: /public/img/swapi.png
---

 A common task when working with data from APIs is crawling the API and inserting the results in a database. In this example we'll import data into Neo4j by crawling a public API, building a graph in Neo4j as we go along. We'll use Neo4j as a queuing mechanism to store URLs for placeholder resources waiting to be fetched.

Specifically, we'll use [SWAPI.co](http://swapi.co) to build a graph of information from the Star Wars universe. SWAPI contains data about characters, planets, starships (and much more!) that appear in the Star Wars universe. Much of the data is collected from [Wookieepedia](http://starwars.wikia.com/wiki/Main_Page).

## The Data Model

![](/public/img/swapi.png)

The graph data model we'll use when building the graph.

This example will use Python and py2neo (a Python driver for Neo4j) to fetch data from [SWAPI.co](http://swapi.co) and is available in an [iPython Notebook](https://github.com/johnymontana/SWAPI-graph/blob/master/SWAPI.ipynb).

To see the full example please check out the iPython Notebook [here](http://nbviewer.ipython.org/github/johnymontana/SWAPI-graph/blob/master/SWAPI.ipynb).

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@lyonwj">
<meta name="twitter:title" content="Crawling APIs Using Neo4j - Building The Wookieepedia Graph">
<meta name="twitter:description" content="A common task when working with data from APIs is crawling the API and inserting the results in a database. In this example we'll import data into Neo4j by crawling a public API, building a graph in Neo4j as we go along. We'll use Neo4j as a queuing mechanism to store URLs for placeholder resources waiting to be fetched.">
<meta name="twitter:creator" content="@lyonwj">
<meta name="twitter:image:src" content="http://www.lyonwj.com/public/img/swapi.png">
<meta name="twitter:domain" content="lyonwj.com">
