# Cool Cypher Hacks

A collection of interesting Cypher tricks and hacks to use until Michael Hunger creates an APOC call to do it better.

## Useful Links


## JSON Output
Creating JSON output is really simple.  Key-Value pairs are requested in the RETURN statement.   Complex structures can be constructed using  COLLECT.

This example uses the built-in Movie graph.

```SCRIPT

MATCH  (p:Person)-[a:ACTED_IN]->(m:Movie {title:'The Matrix'})<-[:DIRECTED]-(director)
RETURN { title:m.title, year: m.released, directors: collect(DISTINCT director.name), actors:collect (distinct {actor:p.name, roles:a.roles})}
```

Returns this JSON:

```JSON
{
  "actors": [
    {
      "actor": "Keanu Reeves",
      "roles": [
        "Neo"
      ]
    },
    {
      "actor": "Emil Eifrem",
      "roles": [
        "Emil"
      ]
    },
    {
      "actor": "Carrie-Anne Moss",
      "roles": [
        "Trinity"
      ]
    },
    {
      "actor": "Hugo Weaving",
      "roles": [
        "Agent Smith"
      ]
    },
    {
      "actor": "Laurence Fishburne",
      "roles": [
        "Morpheus"
      ]
    }
  ],
  "year": 1999,
  "directors": [
    "Lilly Wachowski",
    "Lana Wachowski"
  ],
  "title": "The Matrix"
}
```

# Logic Hacks

## FOR-EACH Hack

Thanks to Michael Hunger for this hack.  
https://gist.github.com/jexp/caeb53acfe8a649fecade4417fb8876a#conditional-data-creation

Cypher does not really have an IF statement, so the following is a useful workaround.  

It uses a FOREACH combined with a CASE WHEN as a crude IF statement.  The value returned when the CASE WHEN condition is evaluated is either an array with 0 items or an array with 1 item.  The length of the array is then used in the FOREACH.  The FOREACH will execute either 0 or 1 times - voila a Boolean

```cypher

// conditionally load phone numbers from an input line of data
FOREACH(ignoreMe IN CASE WHEN ( line.TelephoneNumber is not null) THEN [1] ELSE [] END |
     MERGE (p:Phone {phone:line.TelephoneNumber})
     MERGE (lc)-[:HOME_PHONE]-(p)
      )
```

## Use Range to Quickly Generate Data
```cypher

MATCH (office:Office {location:'London'})
WITH   range (1,1000) as range, office
FOREACH (r in range |
   CREATE (p:Person {name:'Name-' + r})-[:BASED_IN]->(office)
)
```

## Set Operations
This is borrowed from Mark Needham's post https://markhneedham.com/blog/2014/02/20/neo4j-cypher-set-based-operations/     

In earlier times, the first version was 1-2 orders of magnitude slower than the second, but no longer.  

```cypher
MATCH (mark:Person {name: "Mark"})-[:BASED_IN]->(office {location: "London"})<-[:BASED_IN]-(colleague)
WHERE NOT ((mark)-[:FRIENDS_WITH]->(colleague))
RETURN COUNT(colleague) as potentialFriendCount

```
alternate approach
```cypher
MATCH (mark:Person {name: "Mark"})-[:FRIENDS_WITH]->(colleague)
 WITH mark, COLLECT(colleague) as marksColleagues
 MATCH (colleague)-[:BASED_IN]->(office {location: "London"})<-[:BASED_IN]-(mark)
  WHERE NOT (colleague IN marksColleagues)
 RETURN COUNT(colleague) as potentialFriendCount

```

## Pattern Comprehension
Creates a custom projection with path information AFTER the RETURN statement.  This allows expressions to be evaluated and data returned without impacting the iterator of the result set.   The projections are optional

```cypher

MATCH (mark:Person {name:'Mark'})
  return (mark)-[:FRIENDS_WITH]-(:Person)
```

The following returns a JSON structure of related items

```Cypher

// Pattern Comprehension
profile MATCH (q:Movie)
RETURN q{.title,
         directors : [(q)<-[:DIRECTED]-(u) | u.name],
         actors   : [(q)<-[:ACTED_IN]-(t) | t.name],
         writer: [(q)<-[:WROTE]-(w)-[:PRODUCED]->(q2) | w{ .name, movies: q2.title } ] }

```

```JSON
{
  "actors": [
    "James Cromwell",
    "Max von Sydow",
    "Rick Yune",
    "Ethan Hawke"
  ],
  "writer": [],
  "title": "Snow Falling on Cedars",
  "directors": [
    "Scott Hicks"
  ]
}
```

## Top N Matches Aggregation - Plus JSON Results

This technique was 'borrowed' from
https://markhneedham.com/blog/2014/09/26/neo4j-collecting-multiple-values-too-many-parameters-for-function-collect/

This can be run from the :play movies graph.  It will find for each director the year in which they worked with the largest number of distinct actors.


```cypher

// find the year each director worked with the most distinct actors.
MATCH (d:Person)-[:DIRECTED]->(m:Movie)<-[:ACTED_IN]-(a:Person)
WITH d.name AS director, m.released AS year, COUNT (DISTINCT a.name) as actorCounts
// sort the data by actorCounts
ORDER BY director,  actorCounts desc
// Now collect the actorCounts into a list,
// but only keep the first actorCount value (the highest one) per director
// and return the actorCount along with the year in a Map
WITH director, collect({year:year, actors:actorCounts})[0]  as busiestYear
RETURN director, busiestYear

```

## Parameters in Queries - in Neo4j Browser

The process to convert Cypher queries to executable code is generally pretty fast, but this process is avoidable for running subsequent queries by parameterizing the query.

The Neo drivers support passing a parameterized query and a key-value map of parameters.  So does the Neo4j browser - although the process to use parameterized queries in the browser.   

Note:  The older syntax for a parameter {param} has been deprecated - use the new syntax $param

https://neo4j.com/docs/cypher-manual/current/syntax/parameters/

```Cypher
// unwind batch parameter example
:param batch=>  [{name:"Mark",age:32},{name:"Abby",age:42}];

UNWIND $batch as row
CREATE (n:Label)
SET n += row
```

```Cypher
// parameter example
:param actorName=>  'Tom Hanks';

MATCH (a:Person) where a.name = $actorName return a;
```

```Cypher
// list current parameters
:params

{
  "batch": [
    {
      "name": "Mark",
      "age": 39
    },
    {
      "name": "Abby",
      "age": 29
    }
  ],
  "director": "Tom Hanks",
  "actorName": "Tom Hanks"
}
```

## Finding Myself
One common mistake when using variable length relationships is to use a lower bounds of zero - which will return the source node as well as the related nodes.

```Cypher
MATCH (movie:Movie { title: 'Unforgiven' })-[*0..1]-(x)
RETURN x
```
Returns the source node as well as the related nodes.

```Cypher
╒══════════════════════════════════════════════════════════════════════╕
│"x"                                                                   │
╞══════════════════════════════════════════════════════════════════════╡
│{"title":"Unforgiven","tagline":"It's a hell of a thing, killing a man│
│","released":1992}                                                    │
├──────────────────────────────────────────────────────────────────────┤
│{"name":"Gene Hackman","born":1930}                                   │
├──────────────────────────────────────────────────────────────────────┤
│{"name":"Clint Eastwood","born":1930}                                 │
├──────────────────────────────────────────────────────────────────────┤
│{"name":"Richard Harris","born":1930}                                 │
├──────────────────────────────────────────────────────────────────────┤
│{"name":"Clint Eastwood","born":1930}                                 │
├──────────────────────────────────────────────────────────────────────┤
│{"name":"Jessica Thompson"}                                           │
└──────────────────────────────────────────────────────────────────────┘
```

## Find Nulls in a CSV File

```Cypher

// Find Null Values in CSV file
LOAD CSV WITH HEADERS FROM $parms.fileURL AS line  FIELDTERMINATOR '|'
WITH line WHERE ANY (k in keys(line) WHERE line[k] IS NULL)
RETURN  line LIMIT $parms.recordLimit
```
