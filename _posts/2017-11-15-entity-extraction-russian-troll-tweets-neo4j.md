---
layout: post
title: Applying NLP and Entity Extraction To The Russian Twitter Troll Tweets In Neo4j (and more Python!)
introtext: Natural language processing (NLP) techniques like entity extraction can be used to help make sense of a large text corpus. In this post we apply named entity resolution to the scraped Russian Twitter Troll tweets to try to get a better understanding of how these trolls were spreading fake news.
mainimage: /public/img/russian-twitter-trolls/louisiana.png
---

![](/public/img/russian-twitter-trolls/louisiana.png)

# Russian Twitter Troll Tweets

[Previously](http://www.lyonwj.com/2017/11/12/scraping-russian-twitter-trolls-python-neo4j/), we explored how to scrape tweets from Internet Archive that have been removed from Twitter.com and the Twitter API as a result of the US House Intelligence Committee's investigation into Russia's involvement in influencing the 2016 election through social media, largely by spreading fake news.

These accounts were identified by Twitter as connected to Russia's Internet Research Agency, a company believed to have been involved in spreading fake news in an attempt to influence the US election, however Twitter has removed all data related to these accounts.

Our previous post focused on scraping Internet Archive to retrieve the data and import into Neo4j. We also looked at some Cypher queries we could use to analyze the data. In this post we make use of a natural language processing technique called entity extraction to enrich our graph data model and help us explore the dataset of Russian Twitter Troll Tweets. For example, can we try to see what people, places, and organizations these accounts were tweeting about in the months leading up to the 2016 election?

# Named Entity Recognition

According to [Wikipedia](https://en.wikipedia.org/wiki/Named-entity_recognition):

> Named-entity recognition (NER) (also known as entity identification, entity chunking and entity extraction) is a subtask of information extraction that seeks to locate and classify named entities in text into pre-defined categories such as the names of persons, organizations, locations, expressions of times, quantities, monetary values, percentages, etc.

Entity extraction is an important task in natural language processing and information extraction and can be used to help make sense of large bodies of text.

## Entity Extraction With Polyglot

There are several NLP libraries and services available (such as [Stanford CoreNLP](https://stanfordnlp.github.io/CoreNLP/), [IBM's AlchemyAPI](https://www.ibm.com/watson/alchemy-api.html), [Thomson Reuters' Open Calais](http://www.opencalais.com/)) that offer entity extraction. One tool that I like is [Polyglot](https://github.com/aboSamoor/polyglot), a Python library and command line tool for NLP. Polyglot supports *many* different languages (up to 165 for some NLP tasks) and a wide range of NLP features including language detection, sentiment analysis, part of speech tagging, and word embeddings.

In this post we'll make use of Polyglot's entity extraction functionality to find entities mentioned in the tweets in our Russian Twitter Troll tweet data.

Other entity extraction tools use supervised learning approaches that require human labeled training data. Polyglot however uses link structure from Wikipedia and Freebase that results in a language agnostic technique for entity extraction. You can read more about their technique in this [paper.](https://sites.google.com/site/rmyeid/papers/polyglot-ner.pdf?attredirects=0&d=1)

## Installing Polyglot

From looking at some issues online it seems many folks struggle with installing Polyglot, especially on MacOS. These are the steps I used to install Polyglot with the Anaconda Python distribution on Ubuntu 16.04, including Jupyter, thanks to [this gist](https://gist.githubusercontent.com/linwoodc3/8704bbf6d1c6130dda02bbc28967a9e6/raw/786fc5074c811b3ec359135f2ce8d92d8413b33a/polyglotOnMacOSX35.sh).

~~~
conda create -n icutestenv python=3.5.2 --no-deps -y
source activate icutestenv
conda install -c ccordoba12 icu=54.1 -y
conda install ipython numpy scipy jupyter notebook -y
python -m ipykernel install --user
easy_install pyicu
pip install morfessor
pip install pycld2
pip install polyglot
~~~

# Applying Entity Extraction To The Russian Twitter Troll Dataset

Our approach will be to use the Python driver for Neo4j to retrieve all the tweet text in our dataset from Neo4j, run the Polyglot entity extraction algorithm on the text of each tweet, saving the tweet text, tweet id, and extracted entities to a JSON file that we can then use to update the Neo4j graph, extending the datamodel to connect `Tweet` nodes to new `Person`, `Location`, and `Organization` nodes as identified by Polyglot.

## Running Polyglot Entity Extraction

Polyglot entity extraction works by taking a piece of text (tweets in our case), and annotating (or tagging) the text where it recognizes named entities.

Polyglot can recognize three types of entities:

* **Locations** *(Tag: I-LOC)*
  * cities, countries, regions, continents, neighborhoods, etc.
* **Organizations** *(Tag: I-ORG)*:
  * companies, schools, newspapers, political parties, etc.
* **Persons** *(Tag: I-PER)*:
  * politicians, celebrities, authors, etc.

Here's a simple example:

{% highlight python %}
# simple Polyglot entity extraction example

from polyglot.text import Text
blob = Text('@realDonaldTrump "Hillary Clinton has zero record to run on - unless you call corruption positive.." - @IngrahamAngle')
blob.entities
{% endhighlight %}


And we can see that Polyglot recognized "Hillary Clinton" as a `Person` entity:
~~~
[I-PER(['Hillary', 'Clinton'])]
~~~

Note that Polyglot did not tag either of the two Twitter screen names, `@realDonaldTrump` or `@IngramAngle` as entities. Since our data has Twitter mentions we can infer these already.

## Retrieving Tweet Data From Neo4j

We start with the Neo4j database that we built from scraped tweets in the [previous post](http://www.lyonwj.com/2017/11/12/scraping-russian-twitter-trolls-python-neo4j/). We want to query that database for all tweets, retrieving the text of the tweet and the tweet id. We create a Python dict for each tweet that contains the id and text and then append each tweet dict to a list:

{% highlight python %}
from neo4j.v1 import GraphDatabase
driver = GraphDatabase.driver("bolt://localhost:7687")


with driver.session() as session:
    results = session.run("MATCH (t:Tweet) RETURN t.text AS text, t.tweet_id AS tweet_id")

tweetObjArr = []

for r in results:
    tweetObj = {}
    tweetObj['id'] = r['tweet_id']
    tweetObj['text'] = r['text']
    tweetObjArr.append(tweetObj)

{% endhighlight %}

Next, we iterate through each tweet dict in the array and run the Polyglot entity extaction algorithm for each tweet. We create a new dict, including any tagged entities and append those to a new array:


{% highlight python %}
entityArr = []


for t in tweetObjArr:
    try:
        parsedTweet = {}
        parsedTweet['id'] = t['id']
        parsedTweet['text'] = t['text']
        blob = Text(t['text'])
        entities = blob.entities
        parsedTweet['entities'] = []
        for e in entities:
            eobj = {}
            eobj['tag'] = e.tag
            eobj['entity'] = e
            parsedTweet['entities'].append(eobj)
        if len(parsedTweet['entities']) > 0:
            entityArr.append(parsedTweet)
    except:
        pass
{% endhighlight %}

*Note that we could write this in a more pythonic list comprehension, but I think this version is better for seeing what exactly is going on.*


Here's an example of what we have at this point:

{% highlight json %}
{
        "entities": [
            {
                "entity": [
                    "Republicans"
                ],
                "tag": "I-ORG"
            },
            {
                "entity": [
                    "FBI"
                ],
                "tag": "I-ORG"
            },
            {
                "entity": [
                    "Clinton"
                ],
                "tag": "I-PER"
            }
        ],
        "id": 784163886547230720,
        "text": "RT @OffGridInThePNW: Republicans blast FBI for 'astonishing' agreement to destroy Clinton aides' laptops |  https://t.co/cwEUtK9i9V"
    }
{% endhighlight %}

Next, we save this array of entities and tweets to a json file that we'll use in the next step to import into Neo4j:

{% highlight python %}
import json
with open('parsed_tweets.json', 'w') as f:
    json.dump(entityArr, f, ensure_ascii=False, sort_keys=True, indent=4)
{% endhighlight %}


{% highlight shell %}
{'entities': [{'entity': I-PER(['Hillary', 'Clinton']), 'tag': 'I-PER'}],
 'id': 773585101489922048,
 'text': '@realDonaldTrump "Hillary Clinton has zero record to run on - unless you call corruption positive.." - @IngrahamAngle'}
 {% endhighlight %}


## Importing Extracted Entities Into Neo4j

So far our datamodel looks like this:

![](/public/img/russian-twitter-trolls/datamodel_pre.png)

What we want to do is extend the data model to include `Person`, `Location`, and `Organization` nodes, connected to any tweets that contain references to these entities:

![](/public/img/russian-twitter-trolls/datamodel_post.png)

From the previous step, we have a json file of tweet text, tweet id, and extracted entities:

{% highlight shell %}
{'entities': [{'entity': ['California'], 'tag': 'I-LOC'},
              {'entity': ['Cleveland'], 'tag': 'I-LOC'},
              {'entity': ['Nicholas', 'Rowe'], 'tag': 'I-PER'}],
 'id': '669554555835682816',
 'text': 'California man killed in Cleveland was shot in the head and back: '
         'Nicholas Rowe was identified as the man found dead Nov. 20 b...  '
         '#crime'}
{% endhighlight %}

First, we execute three Cypher queries to add uniqueness constraints to our database. The uniqueness constaints allow us to assert that no duplicate nodes will be created. For example, we only want one node with the label `Location` and `name` property `California` to represent the state of California. Any tweets that contain "California" should all have a relationship to the canonical "California" node. Uniqueness constraints give us this guarantee. Uniqueness constraints also create an index, which will speed our import significantly.

{% highlight python %}
with driver.session() as session:
    session.run('CREATE CONSTRAINT ON (p:Person) ASSERT p.name IS UNIQUE;')
    session.run('CREATE CONSTRAINT ON (l:Location) ASSERT l.name IS UNIQUE;')
    session.run('CREATE CONSTRAINT ON (o:Organization) ASSERT o.name IS UNIQUE;')
{% endhighlight %}

Next, we load our json file that contains tweets and entities:

{% highlight python %}
with open("parsed_tweets.json") as f:
    parsed_tweets = json.load(f)
{% endhighlight %}

And define a Cypher query that will take this array of tweets and entities, iterate through them and update the graph, creating new `Person`, `Organization`, and `Location` nodes and creating relationships to any tweets that contains these entities:

{% highlight python %}
entity_import_query = '''
WITH $parsedTweets AS parsedTweets
UNWIND parsedTweets AS parsedTweet
MATCH (t:Tweet) WHERE t.tweet_id = parsedTweet.id


FOREACH(entity IN parsedTweet.entities |
    // Person
    FOREACH(_ IN CASE WHEN entity.tag = 'I-PER' THEN [1] ELSE [] END |
        MERGE (p:Person {name: reduce(s = "", x IN entity.entity | s + x + " ")}) //FIXME: trailing space
        MERGE (p)<-[:CONTAINS_ENTITY]-(t)
    )

    // Organization
    FOREACH(_ IN CASE WHEN entity.tag = 'I-ORG' THEN [1] ELSE [] END |
        MERGE (o:Organization {name: reduce(s = "", x IN entity.entity | s + x + " ")}) //FIXME: trailing space
        MERGE (o)<-[:CONTAINS_ENTITY]-(t)
    )

    // Location
    FOREACH(_ IN CASE WHEN entity.tag = 'I-LOC' THEN [1] ELSE [] END |
        MERGE (l:Location {name: reduce(s = "", x IN entity.entity | s + x + " ")}) // FIXME: trailing space
        MERGE (l)<-[:CONTAINS_ENTITY]-(t)
    )
)

'''
{% endhighlight %}

Note that we make use of the [`FOREACH( _ IN CASE WHEN ... THEN [1] ELSE [] END | ... )` trick](http://www.markhneedham.com/blog/2014/06/17/neo4j-load-csv-handling-conditionals/) for handling conditionals.

Finally, we use the Neo4j Python driver to run our import query, passing in the array of dicts as a parameter:

{% highlight python %}
with driver.session() as session:
    session.run(entity_import_query, parsedTweets=parsed_tweets)
{% endhighlight %}

## Making sense of our extracted entities

Now that we've enriched our graph model we can answer a few more interesting questions of our Russia Twitter Troll dataset.

### Most frequently mentioned entities

~~~
// Most frequently mentioned Person entities
MATCH (t:Tweet)-[:CONTAINS_ENTITY]->(p:Person)
RETURN p.name AS name, COUNT(*) AS num ORDER BY num DESC LIMIT 10

╒══════════════════╤═════╕
│"name"            │"num"│
╞══════════════════╪═════╡
│"Trump "          │22   │
├──────────────────┼─────┤
│"… "              │18   │
├──────────────────┼─────┤
│"Obama "          │15   │
├──────────────────┼─────┤
│"Clinton "        │12   │
├──────────────────┼─────┤
│"Hillary "        │12   │
├──────────────────┼─────┤
│"sanders "        │7    │
├──────────────────┼─────┤
│"Bernie Sanders " │6    │
├──────────────────┼─────┤
│"Donald Trump "   │6    │
├──────────────────┼─────┤
│"pic.twitter.com "│6    │
├──────────────────┼─────┤
│"Hillary Clinton "│6    │
└──────────────────┴─────┘
~~~


~~~
// Most frequently mentioned Location entities
MATCH (t:Tweet)-[:CONTAINS_ENTITY]->(l:Location)
RETURN l.name AS name, COUNT(*) AS num ORDER BY num DESC LIMIT 10

╒══════════════╤═════╕
│"name"        │"num"│
╞══════════════╪═════╡
│"Louisiana "  │13   │
├──────────────┼─────┤
│"Chicago "    │13   │
├──────────────┼─────┤
│"Cleveland "  │11   │
├──────────────┼─────┤
│"baltimore "  │9    │
├──────────────┼─────┤
│"Miami "      │8    │
├──────────────┼─────┤
│"Ohio "       │6    │
├──────────────┼─────┤
│"California " │6    │
├──────────────┼─────┤
│"Iraq "       │6    │
├──────────────┼─────┤
│"Westboro "   │5    │
├──────────────┼─────┤
│"New Orleans "│5    │
└──────────────┴─────┘

~~~


### Exploring Tweets Around Certain Entities


~~~
// Tweets, hashtags, and entities around Louisiana
MATCH (u:User)-[:POSTED]->(t:Tweet)-[:CONTAINS_ENTITY]->(l:Location {name: "Louisiana "}  )
OPTIONAL MATCH (t)-[:HAS_TAG]->(ht:Hashtag)
OPTIONAL MATCH (t)-[:CONTAINS_ENTITY]->(e)
RETURN *
~~~

![](/public/img/russian-twitter-trolls/louisiana.png)



You can find the code for this project on [Github](https://github.com/johnymontana/russian-twitter-trolls), in [this Jupyter notebook](https://github.com/johnymontana/russian-twitter-trolls/blob/master/import/polyglot.ipynb):

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@lyonwj">
<meta name="twitter:title" content="Applying NLP and Entity Extraction To The Russian Twitter Troll Tweets In Neo4j (and more Python!)">
<meta name="twitter:description" content="Natural language processing (NLP) techniques like entity extraction can be used to help make sense of a large text corpus. In this post we apply named entity resolution to the scraped Russian Twitter Troll tweets to try to get a better understanding of how these trolls were spreading fake news.">
<meta name="twitter:creator" content="@lyonwj">
<meta name="twitter:image:src" content="http://www.lyonwj.com/public/img/russian-twitter-trolls/louisiana.png">
<meta name="twitter:domain" content="lyonwj.com">
