
# Russian Trolls demo

## Summary

This is to provide a useful script to replicate the demo in https://www.youtube.com/watch?v=0-1A7f8993M
## Preparation

## Sandbox    
The data is preloaded on the trolls database at https://neo4j.com/sandbox-v2/

The sandbox starts with the pre-written scripts from
:play https://guides.neo4j.com/sandbox/twitter-trolls/index.html


## queries





// count troll retweets
MATCH (r1:Troll)-[:POSTED]->(:Tweet)<-[:RETWEETED]-(:Tweet)<-[:POSTED]-(r2:Troll)
WITH r1, r2, COUNT(*) as count
return r1.name, r2.name, count

// create troll retweet relationship
match (r1:Troll)-[:POSTED]->(:Tweet)<-[:RETWEETED]-(:Tweet)<-[:POSTED]-(r2:Troll)
with r1, r2, COUNT(*) as count
create (r2)-[r:RETWEETS]->(r1)
set r.count = count


// assign pagerank scores
CALL algo.pageRank('Troll','RETWEETS',{write: true, writeProperty: 'pagerank'})


// Community Detection
CALL algo.labelPropagation('Troll', 'RETWEETS', 'OUTGOING', {write:true, partitionProperty:'community', weightProperty:'count'})



## Sample Web page
 this is from https://github.com/neo4j-contrib/neovis.js/blob/master/examples/twitter-trolls.html

 ##
 ## PageRank
 call algo.pageRank ('Troll','RETWEETS', {write: true, writeProperty: 'pagerank'})
