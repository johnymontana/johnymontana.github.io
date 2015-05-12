---
layout: post
title: Twizzard, A Tweet Recommender System Using Neo4j
introtext: A system for ranking tweets based on user affinity and time decay.
permalink: twizzard-a-tweet-recommender-system-using-neo4j
mainimage: /public/img/Screen_Shot_2014_03_10_at_10_00_10_PM.png
---

![Twizzard mobile screnshot](/public/img/Screen_Shot_2014_03_10_at_10_00_10_PM.png)

I spent this past weekend hunkered down in the basement of the local Elk's club, working on a project for a hackathon. The project was a tweet ranking web application. The idea was to build a web app that would allow users to login with their Twitter account and view a modified version of their Twitter timeline that shows them tweets ranked by importance. Spending hours every day scrolling through your timeline to keep up with what's happening in your Twitter network? No more, with [Twizzard](http://www.twizzardapp.com)!

## System structure
Here's a diagram of the system:

![Twizzard system diagram](/public/img/yelp_data_model__3_.png)

* Node.js web application (using Express framework)
* MongoDB database for storing basic user data
* Integration with Twitter API, allowing for Twitter authentication
* Python script for fetching Twitter data from Twitter API
* Neo4j graph database for storing Twitter network data
* Neo4j unmanaged server extension, providing additional REST endpoint for querying / retrieving ranked timelines per user

## Hosted platforms
Being able to spin up hosted instances of our tech stack simplifies the process of putting this project together.

* [GitHub](http://github.com) - It goes without saying that we used GitHub for version control, making collaboration with my [teammate](http://www.visionaryg.com) who handled all the design work quite pleasant.
* [Heroku](http://heroku.com) - Heroku's PaaS for web hosting is great. Free tier is perfect for getting small projects going.
* [MongoLab](http://mongolab.com) - This was my first time using MongoLab's hosted MongoDB service. No complaints, worked great! Also free tier was perfect for getting started.
* [GrapheneDB](http://grapheneDB.com) - GrapheneDB provides hosted Neo4j instances. These guys are awesome! Their service is rock solid. I can't overstate how impressed I am with what they provide (they even allow for running custom server extensions!)

## Getting started
I stumbled acros this node.js [hackathon starter template](https://github.com/sahat/hackathon-starter) a few months ago. I decided to give it a try for the first time this weekend. It puts together a stack that I'm familiar with: node.js, mongoDB, Express, passport for handling OAuth, Bootstrap and jade. It's a great starter template for, as the name implies, starting hackathon projects.


## Graph data model
Since this project deals with Twitter data, modeling that data as a graph seems intuitive. We're concerned with users, their tweets, and the interactions between users. The data model is pretty simple:
![Twizzard graph data model](/public/img/ok__1_.png)

## Inserting Twitter Data With py2neo
Once a user authenticates to our web application and grants us permission to access their Twitter data, we need to access the Twitter API and store the data in Neo4j. We accomplish this with the help of the Python [py2neo](https://github.com/nigelsmall/py2neo) package.
<script src="https://gist.github.com/johnymontana/31cec4c643ee764cfd29.js?file=py2neo.py"></script>

## Ranking tweets
How can we score Tweets to show users their most important Tweets? Users are more likely to be interested in tweets from users they are more similar to and from users they interact with the most. We can calculate metrics to represent these relationships between users, adding an inverse time decay function to ensure that the content at the top of their timeline stays fresh.

### Jaccard similarity index
The [Jaccard index](http://en.wikipedia.org/wiki/Jaccard_index) allows us to measure similarity between a pair of users. For our purposes this is defined as the intersection of their sets of followers divided by the union of their sets of followers. This results in a score between 0 and 1 representing how "similar" the two users are to each other.

$ J(A,B) = \frac{A \cup B}{A \cap B} $

Calculating this in Neo4j Cypher for all users in our database looks like this:

<script src="https://gist.github.com/johnymontana/512e5f4aeb8efbb270ff.js"></script>

### Interaction metric
Another important factor to take into account is how often Twitter users are interacting with each other. Users are more likely to be interested in tweets from users they interact with often. To quantify the strength of this relationship for a user pair `A,B`, we simply divide the number of `A,B` Twitter interactions by the total number of interactions for user `A`.

### Weighted average
The similarity score and interaction score are combined using a weighted average. We weight similarity slightly higher than interaction.

### Time decay
To ensure temporal relevence, an inverse time decay function is used to discount tweets according to the amount of time elapsed since the tweet was sent:

$ TweetScore = \frac{WeightedUserScore}{elapsedTime^2} $

By storing these data and relationships in our Neo4j instance, we simply need to query the database for the highest ranked tweets from our web application and display these to the user.


## Extending the Neo4j REST interface
Neo4j Server provides a REST interface that allows for querying of the database. Neo4j also allows for adding unmanaged server extensions that allow us to extend the built-in REST API and add our own endpoints. We can do this using JAX-RS in Java. In this case, we write a simple server-extension that adds the endpoint `/v1/timeline/{user_id}` that will execute a Cypher query to return the tweets for the specified user's timeline, ordered by the ranking metric we've defined above.

<script src="https://gist.github.com/johnymontana/31cec4c643ee764cfd29.js?file=neo4j_extension.java"></script>




## Twizzard
We had the site up and running by the time final presentations were scheduled. In fact, we even had time for a few iterations based on user feedback. We called our web app Twizzard, as in Your Twitter Wizard (but with two z's, like a Tweet Blizzard). It's running now at [twizzardapp.com](http://twizzardapp.com). Sign in with your Twitter account and check it out.

<a href="http://twizzardapp.com">![Twizzard screenshot](/public/img/Screen_Shot_2014_03_10_at_8_34_39_PM-1.png)</a>

Shout out to my Twizzard teammates [@kevshoe](http://twitter.com/kevshoe) and [@VisionaryG](http://twitter.com/visionaryg). It was great fun working with you guys at Startup Weekend Missoula 2014!

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@lyonwj">
<meta name="twitter:title" content="Building a tweet ranking web app using Neo4j">
<meta name="twitter:description" content="I spent this past weekend hunkered down in the basement of the local Elk's club putting together a project for a hackathon. The project was a tweet ranking web application. The idea was to build a web app that would allow users to login with their Twitter account and view a modified version of their Twitter timeline that shows them tweets ranked in order of what they are likely to be interested in. Spending hours every day scrolling through your timeline to keep up with what's happening in your Twitter network?">
<meta name="twitter:creator" content="@lyonwj">
<meta name="twitter:image:src" content="http://lyonwj.com/content/images/2014/Mar/Screen_Shot_2014_03_10_at_10_00_10_PM.png">
<meta name="twitter:domain" content="lyonwj.com">
