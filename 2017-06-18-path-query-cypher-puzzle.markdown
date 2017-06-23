---
layout: post
title:  "The PathQuery To Cypher Puzzle - Part 1"
date:   2017-06-18 08:00:00
comments: true
categories: gsoc
tags:
    - intermine
    - neo4j
    - PathQuery
    - gsoc
    - Cypher
    - conversion
    - database
---
The conversion of PathQuery To Cypher effectively & accurately is crucial for the Neo4j prototype of InterMine. This post provides a brief overview of Path Query & Cypher and gives you an insight on how we plan to query the InterMine Neo4j database. In upcoming posts, the conversion approach will be discussed.

![InterMine Neo4j Logo](/images/intermine-neo4j.png)

## Introduction

If you have read my previous post, you would have an idea of what [InterMine](http://intermine.org/) is, a data warehousing system which stores complex biological data. Now, one of the main features of InterMine is its fast & flexible querying ability. After all what is the point of storing the data if you can't retrieve it as per your requirements, fast enough! :)

So, there are two parts to an InterMine data warehouse, front-end and back-end. The new front-end of InterMine is called [BlueGenes](http://bluegenes.apps.intermine.org/) and is being developed in the [BlueGenes Repository](https://github.com/intermine/bluegenes). In order to retrieve any data from InterMine, users need to submit some information about the data to BlueGenes. For example, give me all the *Experiments* in which, the *Notch* gene in organism *Drosophila* participates. Now based on this information, [BlueGenes](http://bluegenes.apps.intermine.org/) generates a *Path Query*. The back-end part processes this *Path Query* and returns the required data. You should give a try to the [BlueGenes Query Builder](http://bluegenes.apps.intermine.org/#/querybuilder), it makes building *Path Queries* easier, plus the UI is cool.

## Path Query

If you would like a detailed explanation about Path Queries, you can find it at [The PathQuery API docs](http://intermine.readthedocs.io/en/latest/api/pathquery/). Here I am presenting a much shorter version. First, lets look at a simple PathQuery example.

{% highlight xml %}
<query model="genomic" view="Gene.symbol">
  <constraint path="Gene.length" op="&gt;" value="12345"/>
</query>
{% endhighlight %}

These three lines of XML make up a PathQuery which says, for all the *Genes* which have the *length* greater than *12345*, give me their *Symbols*. The PathQuery consists of the following parts.

* `Paths` - In the example, `Gene.symbol`, `Gene.length` are paths. Paths can be of any arbitrary length, e.g. `Protein.gene.homologues.homologue.alleles.alleleClass` is also a valid Path.

* `Views` - These are the attributes that we want to retrieve from the database. First line in the example shows the `view` part.

* `Constraints` - These are used restrict the matching values i.e. to filter the data. In the example, the second line is the constraint part which says `Gene.length > 12345`.

As I mentioned, I am not presenting other PathQuery parts line JOINSs, Sort Order etc to keep it simple. If you want to get in detail of PathQuery, you are welcome to checkout [The PathQuery API docs](http://intermine.readthedocs.io/en/latest/api/pathquery/).

## Querying Neo4j Database

The goal of the project this summer, is to build a prototype of InterMine which uses Neo4j database at the backend. Now, there are three possible ways to query a Neo4j database.

* [Core Java API](https://neo4j.com/docs/java-reference/current/javadocs/org/neo4j/graphdb/package-summary.html)

* [Traversal Framework](https://neo4j.com/docs/java-reference/current/javadocs/org/neo4j/graphdb/traversal/package-summary.html)

* [Cypher](https://neo4j.com/docs/developer-manual/current/cypher/)

The first two can only be used for creating embedded Neo4j applications i.e. they work when your application runs on same JVM as the Neo4j database instance. In other words, the developed Java application (based on Core Java API or Traversal) and the Neo4j database must reside on the same server.

On the other hand, Cypher runs on [Bolt Protocol](https://boltprotocol.org/) which is a client-server protocol designed for database applications. It allows a client to send statements, each consisting of a single string and a set of typed parameters. The server responds to each request with a result message and an optional stream of result records. The massive advantage of this protcol is that, the Neo4j database and the client application need not be there on the same machine. So, I can run the client side application on my machine and run the queries on a database running on any machine on the planet, if it is connected to the Internet. How awesome is that! ^_^

Although, Core Java API & Traversal framework are a bit faster than Cypher, we decided to go with Cypher because of this advantage. Now let's discuss a bit about Cypher before moving further.

## Cypher

The *Cypher Query Language* (CQL) aka Cypher is a declarative language that is used to query the Neo4j graph database. Again, if you wanna go in detail, checkout [Neo4j Developer Manual](https://neo4j.com/docs/developer-manual/current/cypher/). I am barely touching the surface here. So, here is a simple Cypher query example.

{% highlight cypher %}
MATCH (n:Gene)
WHERE n.length > 12345
RETURN n.symbol
{% endhighlight %}

This Cypher query asks Neo4j engine to match all the nodes which are labelled as *Genes*, and if their *length* is greater than *12345*, then return their *symbols*. Yeah, this is similar to what the PathQuery example above says. The only difference is that this is in the context of a Graph Database and that was in the context of InterMine which is based on a relational database.

## PathQuery To Cypher

Now, we know that the InterMine front-end creates PathQueries which are processed by the IM Path Query Service. Also, the InterMine Neo4j Graph is queried using Cypher. So, in order for BlueGenes to query Neo4j, we need to somehow convert the Path Query into Cypher. 

Since the *paths* in the path query can get arbitrarily big and many other components like JOINs, Sort Order, Constraints are there to add to the complexity, the conversion of Path Query to Cypher is not a trivial task. The converter should convert ANY path query of ANY complexity to cypher. Turns out that this is the hardest and the most time consuming part of the project.

In the next part, I am going to cover the approach that we are going to use for PathQuery to Cypher conversion. So stay tuned!
