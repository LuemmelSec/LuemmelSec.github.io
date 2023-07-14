---
layout: post
title: Fantastic BloodHound Queries and Where to Find Them
---

Hey, dog walker! C'mere. How would ya like to buy the letter O? What about an invisible ice cream cone?  
Or... psssht... some fu***ng fancy BloodHound queries?  

<img src="/images/2023-06-01/lefty_meme.png">  
 

<!--more-->
# Introduction  

I was not happy that since the AzureHound release there were no integrated queries shipped with BloodHound, nor could I find ones besides the stuff from [Hausec](https://hausec.com/2020/11/23/azurehound-cypher-cheatsheet/).   
During several pentests in the past I collected cloud stuff with AzureHound, but while only being able to get some general info like "which users are GA" etc., was never able to make much profit out of it.  
So once again it was up to good old me to get shit done. But ...  

I have no idea about Cypher queries  
I don't know what the Graph Theory is like   
I have no idea of what to search for   
I don't even have a deeper knowledge of all the Azure stuff  

<I AM DUMB MEME OR STH ALIKE>

So all in all the best prerequisits to start off with a new topic I guess.  
It so happened that at the same time I was doing the Xintra Azure Attack course [Attacking and Defending Azure & M365](https://training.xintra.org/attacking-and-defending-azure-m365) from Lina [@inversecos](https://twitter.com/inversecos), and for nearly each topic there came up several ideas about what to look for or what would be beneficial to search for to carry out certain attacks. So I partly used the course as a sort of guideline to gather some of the queries as well as all the gems I collected in my personal arsenal of Azure attacks. If you want to dig into attacking Azure, absolutely go for it.  
As sharing is caring I also opened PRs for AzureHound and BloodHound (more on this later).  
If you just happen to be here for the fancy stuff -> here are the standalone queries [Awesome BloodHound Queries](https://github.com/LuemmelSec/Custom-BloodHound-Queries) and here the [PR](https://github.com/BloodHoundAD/BloodHound/pull/670) for the tool itself.    


There also is [THE DOG WHISPERER'S HANDBOOK](https://ernw.de/download/BloodHoundWorkshop/ERNW_DogWhispererHandbook.pdf) from [Walter](https://twitter.com/SadProcessor), and I urge you to take a look into this masterpiece because it does a much better job than I ever could do.    
I just combined this with another approach here, because I am a lazy ass pentester.  

# Cypher what? 

First things first. What are we talking about in general?  

## Graph Theory

Well yeah, all the BloodHound stuff is based around the [Graph Theory](https://en.wikipedia.org/wiki/Graph_theory).

<img src="/images/2023-06-01/graph_meme.png"> 

It basically is all about relationships between stuff that is visualized in terms of nodes and edges, where nodes are like objects - let's say a ``computer`` - and edges define what that relationship is like - let's say ``owns``. So if ``Bob`` owns a computer object in AD we would have two nodes and one edge.

<img src="/images/2023-06-01/bob_owns_computa.png"> 

But Dan you said Graph Theory and not Node Theory or Edge Theory...  
That is true my fellow friend. The cool thing here is, that we can combine more nodes and more edges to draw what is called a graph. Let's now assume that in addition to the scenario described above we have a Domain Admin with a session on the ``computer`` node Bob ``owns``, and he is holding the key to the kingdom as we all know from fairy tales. Now if we would like to know how ``Bob`` would be able to enter the castle it would be like this:  

<img src="/images/2023-06-01/bob_kingdom.png">

I hope you understand that this is very simplified, but you should get the idea, right?  
And that's about it.

## Cypher  

All the stuff you let the doggo collect is pushed into a [Neo4j](https://en.wikipedia.org/wiki/Neo4j) database. The data in this database can be queried with a language called Cypher.
I can't explain it better then the legend [Rohan](https://twitter.com/CptJesus) himself, so please take your time to read this -> [Intro To Cypher](https://blog.cptjesus.com/posts/introtocypher/).  

In most cases what you want is that the GUI is drawing you a path from ``A`` to ``B`` or to stick with ``Bob`` and the ``Kingdom``.  
We start with a ``MATCH``, define nodes in parentheses ``()`` and edges in brackets ``[]`` and also provide it with a direction in which to search with ``-`` and ``<>``. We also define variables that get filled with what we are searching like ``Users``, ``Groups``, but also ``admin to``, ``member of`` etc. Lastly ``RETURN`` is used to present the results. If you would like to filter, let's say on the node ``User``, you can do so with a condition inside ``{}``.
Be aware that everything is **CASESENSITIVE**. So you can search e.g. for ``User`` but not for ``user``.  

Let us start with a more simple query and assume we would like to see the group memberships of ``Bob``. This could look like this:  


```
MATCH (u:User{name:"Bob"}), (g:Group), p=(u)-[:MemberOf]->(g) RETURN p
```

This translates to:  
Take the node ``User`` as variable ``u`` and ``Group`` as variable ``g``, where we filter the ``User`` node for where the ``name`` property matches ``Bob`` and draw a path ``p`` for every matching ``u`` where the edge is ``MemberOf`` ``g``.

The result would be a graph looking like this:  

<img src="/images/2023-06-01/bob_groups.png">

When searching for paths, we can also limit / specifiy the amount of nodes the path might have by adding something like ``*1..3`` - between 1-3 nodes, ``*1..`` - between one and infinite nodes, ``*..2`` - maximum 2 nodes etc, to our search parameters. The ``shortestPath`` function searches, well, for the shortest path between the given nodes it can find - magic.     
Like: 
```
MATCH (u:User{name:"Bob"}), (g:Group{name:"DOMAIN ADMINS@evilcorp.local"}), p=shortestPath((u)-[*1..3]->(m)) RETURN p
```
This would try to find the shortest path from ``Bob`` to the ``DOMAIN ADMINS@evilcorp.local`` group with a minimum of one node and a max of 3 nodes in the path.  



# ChatGPT to the rescue  

Enough theory. This blog should be more like a guideline on how you can write your own queries without actually knowing what you are doing. So let's hit the street and get some traction.  

<img src="/images/2023-06-01/dont_know_shit.png">

If you are not living behind a rock, you probably heard of all the new fancy AIs like ChatGPT and alike. And so do I, thinking about how I can leverage this to my advantage.  
To be honest it was and is not as simple as asking for "Write me a Cypher query that searches for X", but you can get a pretty good baseline out of it that you "just" need to tweak to make it work. Together with the already existing queries it becomes as easy as breathing. 

So how was my approach lately? Well, it all starts with the question what I would like to search for. Let's come up with some examples.  

<img src="/images/2023-06-01/chatgpt1.png">

Perfect, lets copy pasta end get the results.  
```
MATCH (u:User)-[:MEMBER_OF]->(g:Group)
WHERE g.value = "high"
RETURN u.name, g.name
```

<img src="/images/2023-06-01/nodata.png">

Well, fuck you ChatGPT. Why is this not working?  

Let's dig into it by checking what nodes are available at all and which properties they offer. This time ChatGPT for the win?

<img src="/images/2023-06-01/chatgpt2.png">

```
MATCH (n)
RETURN DISTINCT labels(n) AS node_label
```

<img src="/images/2023-06-01/nodata1.png">

The struggle is real, but this time it happens because the GUI can not handle the raw Cypher query (don't ask me why exactly). However, the Neo4j console can.  

<img src="/images/2023-06-01/neo4jconsole.png">  

To query all properties of a specific node we can do this like so (after a huge fight with ChatGPT again):  
```
MATCH (u:User)
RETURN properties(u) AS user_properties
LIMIT 1
```
<img src="/images/2023-06-01/nodeproperties.png">  

This means that we can e.g. directly filter the node ``User`` for the property ``samaccountname``.  

We can get a list of all available edges this way:  

```
MATCH ()-[r]->()
RETURN DISTINCT type(r) AS edge_label
```
<img src="/images/2023-06-01/edges.png">

If you followed along closely you might have noticed why our initial query to get members of high value groups in Azure didn't work out.  
On the one hand ChatGPT used the nodes ``User`` and ``Group`` rather than ``AZUser`` and ``AZGroup``.  
Additionally there is no ``value`` propertie available to neither ``Group`` nor ``AZGroup``. I think what it tried to do is query the ``highvalue`` property.  

<img src="/images/2023-06-01/highvalue.png">

Our final query can then go this way:  

```
MATCH (g:AZGroup) WHERE g.highvalue = true  RETURN g
or
MATCH (g:AZGroup{highvalue:true}) RETURN g
```

<img src="/images/2023-06-01/highvalue_success.png">

What the ... one group? What about GlobalAdmin maybe?  
In terms of Azure, or Entra, or whatever the name will be next time, GlobalAdmin is a role rather than a group. 

<img src="/images/2023-06-01/hacker.png">

We could therefore do it like this:  

```
MATCH p=(n)-[:AZHasRole|AZMemberOf*1..2]->(r:AZRole WHERE r.displayname =~ '(?i)Global Administrator|User Administrator|Cloud Application Administrator') RETURN p
```

In this case ``~ '(?i)Global Administrator|User Administrator|Cloud Application Administrator')`` specifies a regular expression searching for caseinsensitive names devided by the ``|`` as ``OR``.  

Perfekt.  
Next case.  

I wanted to find all Azure users that are synced from the onPrem AD, so I could check if it would be possible to jump from AD -> AAD.  
Looking inside AAD, we find that the user objects that are synced have a property called ``onpremisesyncenabled``.  
Cool let's search for it:  

```
MATCH (n:AZUser WHERE n.onpremisesyncenabled = true) RETURN n
```

<img src="/images/2023-06-01/nomeme.png">

But why? The BloodHound GUI even shows this field, but it is empty all the time.  
It turned out that the API AzureHound is using under the hood (It is the Graph API ``/v1.0/users``) has a default set of values that are fetched if you don't tell it to do otherwise by giving it a ``$select`` parameter with the additional values you want to collect. See [here](https://learn.microsoft.com/en-us/graph/api/resources/users?view=graph-rest-1.0#common-properties) and [here](https://learn.microsoft.com/en-us/graph/api/resources/user?view=graph-rest-1.0#properties) and [here](https://learn.microsoft.com/en-us/graph/api/user-list?view=graph-rest-1.0&tabs=http#optional-query-parameters).  

The only things you get with AzureHound out of the box is this:  

<img src="/images/2023-06-01/defaultvalues.png">  

So I was like  

<img src="/images/2023-06-01/holdmybeer.png"> 

The outcome is the following [PR](https://github.com/BloodHoundAD/AzureHound/pull/42) which fixes this and some other issues.  

One last example and then the rest is up to you guys to play around and create your own queries.  

I followed [Fabian Bader`s](https://twitter.com/fabian_bader) [blog](https://cloudbrothers.info/en/prem-global-admin-password-reset/) of pwning onPrem AADConnect servers to move and escalate to Azure. I was wondering if I could write a query that would help me identify AADC related  objects, no matter what they are, and ended up with this:  

```
MATCH (u) WHERE (u:User OR u:AZUser) AND (u.name =~ '(?i)^MSOL_|.*AADConnect.*' OR u.userprincipalname =~ '(?i)^sync_.*') OPTIONAL MATCH (u)-[:HasSession]->(s:Session) RETURN u, s
```

So we are searching for nodes of the type ``User`` or ``AZUser``. Their name should be (fuck the case) ``MSOL_something`` or contain ``AADConnect`` or ``sync_xxx``. Optional we are also looking if they have sessions, and if yes, return them.

# Wrap up

I think we are able to simplify tasks with the help of AI, while we should not blindly trust what it is telling us.  
Things that can help you with cool BloodHound queries:  

- Use existing queries as a starting point. From there adjust them to your needs.
- Ask ChatGPT. But please, do not just rely on the results and expect ne need to tweak them.
- Ask the community. I personally reached out to [Jonas] and [Andy] several times asking questions about why certain things won't work and how to work around. One advise though: Do not blindly trust them as well :)  
- Read the blogs. This one also provides some useful links. The Handbook has even more of them.  

# Acknowledgement FIX ME

Big shoutout to all you awesome people sharing knowledge and tools:  

[Andy](https://twitter.com/_wald0)     
[Will](https://twitter.com/harmj0y)  
[Rohan](https://twitter.com/CptJesus)  
[Lina]()
[Walter](https://twitter.com/SadProcessor)  
[Ryan](https://twitter.com/Haus3c)  
[Fabian](https://twitter.com/fabian_bader)  
[Jonas](https://twitter.com/Jonas_B_K)

Stay safe and happy Graphing.  
LuemmelSec
