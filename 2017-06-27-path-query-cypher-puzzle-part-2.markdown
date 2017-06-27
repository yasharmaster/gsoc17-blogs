---
layout: post
title:  "PathQuery To Cypher - Part 2"
date:   2017-06-26 08:00:00
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

In this post I'll present the approach that I am using for the converting Path Query to Cypher. The basics of Path Query to Cypher conversion have been discussed in my [previous post](/blog/2017/path-query-cypher-puzzle/).

## Observations

A Path Query consists of many *paths*. Each path represents some data in the database. For example consider the following Path Query.

{% highlight xml %}
<query model="genomic" view="Gene.length Gene.homologues.dataSets.publication.title" constraintLogic="(A and B)">
	<constraint path="Gene.homologues.evidence.publications.abstractText" value="a" op="CONTAINS" code="A"/>
	<constraint path="Gene.homologues.dataSets.name" value="a" op="CONTAINS" code="B"/>
</query>
{% endhighlight %}

It contains the following four paths.

- `Gene.length`
- `Gene.homologues.dataSets.name`
- `Gene.homologues.dataSets.publication.title`
- `Gene.homologues.evidence.publications.abstractText`

One observation that we can make by looking at these paths is that some prefixes are common in paths. For example, `Gene.homologues` is common in the last three, `Gene.homologues` is common is the second & third and `Gene` prefix is there in all of them.

Infact, while building queries in the query builder, we start from a model and add its attributes, references & collections to our query. Then we move on to any of the references/collections and then add its attributes, references & collections to our Path Query as required, and so on. Thus, all the paths *will* have some prefix in common and there is an *heirarchy* associated with different components of the path.

With a bit of thought, I came to a conclusion that [*Tree Data Structure*](https://en.wikipedia.org/wiki/Tree_(data_structure)) would be the perfect representation for all the paths of the Path Query. Tree is a popular hierarchical data structure which has a root, subtrees of children with a parent node.

![Tree Data Structure](/images/tree-data-structure.png)

## PathTree Representation

Let us generate a *Tree* using the four paths of the example Path Query shown above. Since it is a Tree made up of paths, we can call it a Path Tree.

![A Path Tree](/images/PathTree.png)

A path tree is made up of many *TreeNodes*. Each tree node represents a component of the path. For example, the path `Gene.homologues.dataSets.name` would be represented by four TreeNodes - Gene, homologues, dataSets & name. These components further represent the Nodes, Relationships & Properties in the InterMine Neo4j graph. These TreeNodes also represent Neo4j Graphical Entities (& properties). All the paths having common prefix have common ancestor TreeNodes in the PathTree. This way we can represent hierachy among the various components of the path and can avoid storing redundant information.

## Generating Cypher using Path Tree

In Cypher, we can assign Nodes, Relationships & even Paths to the variables. These variables can then be used in place of those Nodes/Relationships/Paths in the rest of the query. For example consider an example Cypher query.

{% highlight cypher %}
MATCH (n:Gene), (n)-[r:locatedOn]-(c:Chromosome)
WHERE n.length > 12345
RETURN c.symbol
{% endhighlight %}

In the example, we first matched all the *Genes* and assigned it to the variable `n`. Now, `n` is used to *MATCH* a relationship from *Genes* to *Chromosomes* and also in the *WHERE* clause it is used to compare the length of the Genes.

This way, while converting a Path Query to Cypher, we can assign a unique variable name to each *TreeNode* in the Path Tree and then use it in the remaining MATCH, WHERE, RETURN, ORDER BY & OPTIONAL MATCH clauses. The *high-level* approach, in an algorithmic form is presented as follows.

{% highlight c++ %}
1. Take a PathQuery object as input.
2. Retrieve the following information from the PathQuery
	1. Views
	2. Constraints
	3. Sort Order
3. Retrieve all the Paths from the Views, Constraints & Sort Order.
4. Using the Paths, create a PathTree (please refer image attached) such that 
	1. Each TreeNode represents a component of the path. For example, the path "Gene.pathways.identifier" forms three TreeNodes i.e. Gene, Pathways & Identifier.
	2. Paths with common prefix have the same common ancestor.
	3. Root TreeNode represents a Graph Node.
	4. All Leaves of the PathTree always represent Graph Properties.
	5. All other Internal TreeNodes can represent either Graph Nodes or Graph Relationships.
5. Assign a unique variable name to each Internal node of the PathTree.
	1. This variable name will be used for referring that TreeNode in the cypher query.
	2. For generating the variable name, we can separate each component of the path using underscores. For example, the variable name for "Gene.pathways.identifier" will be gene_pathways_identifier.
6. Use the PathTree to generate the cypher query
	1. For creating the Match Clause,
		1. Simply write all the edges of the tree separated by commas after "MATCH" keyword.
		2. Add variable name assigned in the previous step to each entity in cypher.
	2. For creating the WHERE clause,
		1. For each constraint, get its path, operator & value.
		2. Get the variableName of the last TreeNode of the path.
		3. Add "<variableName> <operator> <value>" after the "WHERE" keyword
		4. Add operators in between each constraint as per the constraint logic.
	3. For creating the RETURN clause
		1. For each view, get its path
		2. Get the variableName of the last TreeNode of the path
		3. Add variableNames separated by commas for each such variable
	4. For creating ORDER BY clause,
		1. For each Sort Order, get its path
		2. Get the variableName of the last TreeNode of the path
		3. Add variableName ASC/DESC separated by commas for each such variable
	5. For handling JOIN operations in the PathQuery,
		1. Add OPTIONAL MATCH clause in the query for corresponding paths.
7. Return the generated query
{% endhighlight %}

In the next post, I'll explain the generation of each clause - MATCH, RETURN, ORDER BY, WHERE & OPTIONAL MATCH separately. Meanwhile, you can have a look at the Path Query to Cypher conversion code at [org.intermine.neo4j.cypher](https://github.com/intermine/neo4j/tree/dev/src/org/intermine/neo4j/cypher) package. 