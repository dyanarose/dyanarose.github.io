---
layout: post
title: "Preventing Duplication when Creating Relationships in Neo4j"
date: 2014-07-08 20:50:10 +0100
comments: false
categories: [neo4j, cypher]
author: Dyana Rose
---
Creating relationships between known nodes using Cypher in Neo4j is simple.

    MATCH (p:Person), (e:Episode)
    CREATE (p) - [:INTERVIEWED_IN] -> (e)

But what it you don't know if one of the nodes exists? And further, what if you don't know if the relationship itself already exists?

-   If the node doesn't exist, I want it to be created.
-   If the relationship doesn't exist, I want it to be created.
-   If both node and relationship exist, then nothing should be changed

The simple scenario is of a set of Episode nodes and a set of Person nodes.
    CREATE (e:Episode {title:"foo", subtitle:"bar"})
    return e
    CREATE (p:Person {name:"Lynn Rose"})
    return p

The Episode nodes are known to exist. The Person nodes may or may not exist.

My first attempts at getting this right in my working database were tragic failures.
For example, the following, while working as intended in a new 2.0.1 database, fails and causes duplicate person nodes in my working database. (Adding a unique constraint on person.name causes the statement to throw an exception rather than create a duplicate)
    MATCH (e:Episode {title: "foo"})
    CREATE UNIQUE (e) <- [:INTERVIEWED_IN] - (p:Person {name:"Lynn Rose"})

As I said, the duplication doesn't happen in a clean 2.0.1 database, so the problem must be with my working database.

1.  The database was originally created on Neo4j 1.9.3 and is now running under Neo4j 2.0.1
2.  The nodes created before the release of 2.x use the old style indexes.

But those facts aside, I still need to stop the duplication! So back to the [Cypher documentation](http://docs.neo4j.org/chunked/stable/cypher-query-lang.html).

The Cypher documents for [CREATE UNIQUE](http://docs.neo4j.org/chunked/stable/query-create-unique.html)  specify the following in a call out box:
>   [MERGE](http://docs.neo4j.org/chunked/stable/query-merge.html) might be what you want to use instead of CREATE UNIQUE

It's MERGE that gives the ability to control what happens when a node is, or isn't, matched. It does this through the syntax of ON MATCH and ON CREATE.

Using MERGE and ON CREATE I can get a handle on an existing person node to be able to use in our relationship creation, thus preventing duplication of person nodes.

    MATCH (e:Episode {title: "foo"})
    MERGE (p:Person {name: "Lynn Rose"})
    ON CREATE SET p.firstname = "Lynn", p.surname = "Rose"
    MERGE (e) <- [:INTERVIEWED_IN] - (p)

But, we've still got an issue here. This doesn't necessarily solve the problem of duplicating relationships.

In that call out box in the CREATE UNIQUE documentation, it goes on to say:
>   Note however, that MERGE doesn't give as strong guarantees for relationships being unique.

So I take from this that I should use MERGE to prevent node duplication, but CREATE UNIQUE should be used to prevent relationship duplication.

    MATCH (e:Episode {title: "foo"})
    MERGE (p:Person {name: "Lynn Rose"})
    ON CREATE SET p.firstname = "Lynn", p.surname = "Rose"
    CREATE UNIQUE (e) <- [:INTERVIEWED_IN] - (p)

And here we are.

-   If the node doesn't exist, it is created using MERGE ON CREATE.
-   If the relationship doesn't exist, it is created using CREATE UNIQUE.
-   If both node and relationship exist, then nothing is changed.