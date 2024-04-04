@title  RDF, Knowledge Graphs, Document Stores
@date   2024-04-02
@author beka
@tags   rdf, knowledge graph, graph database, document store, freebase, mongodb

# RDF, Knowledge Graphs, Document Stores

One particularly interesting realm of p2p technology is a place where it intersects the semantic web and related topics from the late 90s and early 2000s. In particular, p2p approaches to things like RDF, graph databases, document stores such as MongoDB, etc. are a challenging but potentially very useful thing to have.

This article aims to discuss some of the techniques existing in the literature, or latent in the possibility space, in order to try to convey some of the challenges, and some possible routes to explore for future systems.

For simplicity, we'll assume that the systems in question deal with semantic triples, as in RDF. This isn't required, but it's the smallest interesting case, and everything generalizes from there in various ways. A triple in this setting consists of three parts, a subject, a verb, and an object. An example of a triple might be something like `(dylan_hunt, has_rank, captain)`, which means that Dylan Hunt has the rank Captain, or `(trance_gemini, has_one, tail)`, which means that Trance Gemini has a tail. The subjects are `dylan_hunt` and `trance_gemini`, respectively, the verbs are `has_rank` and `has_one`, and the objects `captain` and `tail`. What the collection of valid subjects, verbs, and objects are depends on the system.

Notations here don't matter much, so this article will continue to use the mathematical triple notation, but there are many many options.

## Storing Triples and Basic Triple Queries

The first interesting problem, and probably the most basic, is how to store and query for individual triples. For instance, consider the triple `(trance_gemini, has_one, tail)`. How might this be stored in a p2p system, thereby asserting its truth? A few options exist depending on what kinds of queries we aim to do.

If our primary questions are only of the yes/no variety, then one simple choice is to just use a DHT. We can hash the triple and that tells us the key to store the triple under. This would basically treat the triple as a document.

To query for the presence of a triple to see if its asserted to be true, we simply look up its key, as in any DHT. If we get the triple back, then the answer is yes, otherwise no or maybe "signs hazy try again later" (depending on whether we want to use the open world assumption or not).

We might also want to query for triples that look like `(X, has_one, tail)` for any `X`, and maybe we expect to get back all of the valid `X`s, or all of the valid triples, or whatever. In a case like this -- where we want to specify some but not all of the triple's components -- we can modify the storage slightly. For every triple we wish to assert, we can store it like normal, storing `(trance_gemini, has_one, tail)` at its key, but we can _also_ substitute a dummy symbol, lets say `_`, in place of each component we wish to be able to omit. If we want to omit any component, leaving at least one component behind, then that gives us

```
(_, has_one, tail)
(trance_gemini, _, tail)
(trance_gemini, has_one, _)
(trance_gemini, _, _)
(_, has_one, _)
(_, _, tail)
```

We can then hash these six partial triples, and store the full triple under each. To query, we simply run the same hashing, and retrieve everything stored there. Since many possible full triples will hash to the same partial triple -- for instance `(_, has_one, tail)` will be used for any triple about someone having a tail -- we can retrieve all of the satisfying components. Efficiency might dictate that we don't actually send back full triples but only the components, given the amount of redundancy that's present.

If range queries are desired -- for instance, to be able to search for triples like `(X, has_age, Y) where Y > 20 and Y < 50` -- then we can use various multi-attribute range query systems. For simplicity's sake, let's assume every component of the triple can be treated as being orderable. For things like names and other text, the usual lexicographic ordering suffices, and for inherently unordered things like verbs, we can impose an ordering based on the canonical representation. Then a complete triple is a coordinate in a 3-dimensional space formed by the three ordered in question. We can then simply store and query using standard higher-dimensional range query systems.

## Compound Queries

In general, querying for individual triples is not sufficient. A typical query will actually involve many triples, because we wish to find out some more complicated things. For example, we might want to search a database of scifi shows that have space stations where the executive officer of the station is a woman. That might be represented by a conjunctive query

```
(X, instance_of, tv_show) and
(X, has_genre, scifi) and
(X, has_setting, Y) and
(Y, instance_of, space_station) and
(Y, has_crewmember, Z) and
(Z, has_rank, executive_officer) and
(Z, instance_of_woman)
```

Running this query should return not just a set of satisfying triples, but rather a set of assignments of values to variables, which was one of the options mentioned above as an optimization. We would expect to get something like

```
X = babylon_5_tvshow,
Y = babylon_5_station,
Z = susan_ivanova.

X = star_trek_deep_space_9_tvshow,
Y = deep_space_9_station,
Z = kira_nerys.
```

This kind of query is significantly more complex. In a traditional relational database, we would need to perform a number of joins to do this. It's a definitively tricky sort of query.

In the p2p setting, it's even trickier. One option is to simply run each triple query separately, and do the joins ourselves, but that's obviously wasteful. Another option is to do individual queries in some order, propagating partial results around. For instance, maybe starting with `(X, instance_of, tv_show)`, we get a list of all TV shows, then for each one we substitute the show's identifier in for `X` in the remaining triples and continue querying with more information. This approach is acceptable, at least insofar as it will get us the right results and will involve less unnecessary work on our part.

We can improve this, of course, by starting with triples that are going to narrow down the variable assignments as much as possible. For instance, there are lots of TV shows, but hardly any space stations, even in science fiction, so probably we're better off starting with the triple `(Y, instance_of, space_station)`. To successfully prioritize things, we need good statistics, so we can also add some extra kinds of queries to the system: instead of simply querying for a partial triple's variable assignments, we can also query for things like how many distinct values there are for the different variables, or how many total variable assignments there are, or any other relevant statistic we care to ask about.

We can extend this to asking statistics about a particular set of candidate variable assignments. For instance, suppose we've done the `(Y, instance_of, space_station)` query, and gotten back a list of values for `Y`. We might then go and ask each for statistics for each of the `Y`-related queries, where the question is "how many distinct triples exist where the `Y` value comes from this chosen set?". That statistic would depend on the set we give it, not just the total triples, and so if we give it the results for `(Y, instance_of, space_station)`, we'll get a more precise estimate of the best choices to make. Probably we'll find that the `(X, has_setting, Y)` has the fewest triples, and is therefore the most informative, so we prioritize it in the next round of queries, and ask for all of the variable assignments wheere `Y` is chosen from the specified set of values. This would give us a set of candidate `X` values, and so on.

This process is also probably amenable to some of the constraint satisfaction techniques that involve arc consistency. No papers on this topic seem to exist, so it's probably an open research topic, but the literal on constraint satisfaction is extensive and can probably be adapted with little effort.

One possible drawback of this particular approach to compound queries is that the network traffic is potentially very high. An alternative is to pass the query on to other nodes in sequence. For instance, instead of asking for values for `(Y, instance_of, space_station)`, and then having them all transmitted across the network, we can pass the entire query to the node responsible for the pattern `(_, instance_of, space_station)`, and they can do the selection themself, and then pass the remainder of the query on further to the `(_, has_setting, _)` node, with the appropriate candidates for `Y` as part of the query. This way, the computational load is more or less the same, but the network load is less, possibly about half, at least for the core triple queries, ignoring the statistics queries.

## Document Store Queries

A related set of problems is document store databases such as MongoDB. In a setting like that, the basic kind of stored information is an entire document, and in MongoDB's case this is a JSON object. The simplest query is to simply retrieve a particular document, but the more interesting queries involve giving a _template_ for an object. For instance, we might have the document

```
{
    "instance_of": "tv_show",
    "name": "Babylon 5",
    "has_genre": "scifi",
    "setting": {
        "instance_of": "space_station",
        ...
    },
    ...
}
```

And we could query for such objects with

```
{
    "instance_of": "tv_show",
    "setting": {
        "instance_of": "space_station"
    }
}
```

which might find all TV shows set on a space station. Performing this kind of query in a database such as MongoDB, which is document-oriented, can be inefficient. The basic way to implement such a search is to simply iterate over all the documents in question and see if they match. This is obviously a bad choice for a single server, but for a p2p setting this is catastrophically bad, as it would require inspecting every single entry which is almost certainly infeasible.

Fortunately, some solutions exist. In MongoDB specifically, you can create indexes on individual keys, for instance the "instance_of" key, which makes it possible to very quickly find all the documents that represent TV shows. It's also possible in MongoDB to create multi-key indexes where an entire set of keys is being tracked, not just one key. In a p2p setting, the best solution is probably to proactively index all sets of keys up to size 3 or so, thereby providing lots of information to use to constrain the lookups/

The same kind of statistics techniques, and candidate-constrained queries that we did above for triples are applicable here as well, which shouldn't be too surprising since these are basically triples, just taken as aggregates rather separate units. Indeed, the single-key index is just a triple pattern as above. But this also then suggests that we can back-port the multi-key indexes to the triple technique above and use multi-triple indexes as well. Thus, instead of storing patterns like just `(_, instance_of, space_station)`, we would also store `(_, instance_of, space_station) and (_, has_crew, _)`, whenever the subjects are the same.

MongoDB also supports what it calls "multikey indexes". The specific nuances of MongoDB's multikey indexes aren't especially important, but they do translate somewhat straightforwardly to this setting. Namely, instead of just indexing documents based on sets of keys that they have, such as objects that have both the keys `instance_of` and `has_crew`, we can also think of indexing by deeper keys as well.

For example, the TV show document above has the key `has_setting`, which has a value which itself has the key `has_crew`. So then why not index based on that? To steal MongoDB's notation, the outer document for the TV show can be indexed also by the key `has_setting.has_crew`. This is itself a two-key index, but the keys aren't off the same object, they're chained together. Obviously this applies also to triples.

## Generic Graph Indexes

A more general pattern applies here: for any graph structure, whether it's a tree, a DAG, or a full blown graph with cycles, we can collect up a whole variety of different subgraphs and index by them. One simple enough option is to build an index of nodes based on all of the paths they're part of, up to some maximum size. Perhaps the index would store and return the full path, including its nodes, or perhaps it only stores which nodes participate in which position, leaving out the full path information, and relying on the querying node to fill that out. What we choose would depend on how this kind of index behaves, and research is required.

Another option, provided that the set of possible edge labels isn't too obnoxiously large, is to build up a kind of "signature" of a node in a graph, or an entire graph and querying for that. For instance, each node participates in only so many length 2 paths. And there are only so many length 2 paths that are possible: quadratically many as the number of edge labels, as there are two edges in the path. So then we can imagine building a signature by enumerating all of the length 2 paths in some canonical order, and counting up how many of each a given node participates in.

For instance, suppose we have the edge labels `foo`, `bar`, and `baz`, and the following graph:

```
a --foo--> b --foo--> c --bar--> d
           |
          baz
           |
           v
           e
```

The node `b` participates in 3 paths of length 2:

```
a --foo--> b --foo--> c
a --foo--> b --baz--> e
b --foo--> c --bar--> d
```

So then, we enumerate all of the paths, and list the counts for each kind that `b` participates in:

```
foo.foo: 1
foo.bar: 1
foo.baz: 1
bar.foo: 0
bar.bar: 0
bar.baz: 0
baz.foo: 0
baz.bar: 0
baz.baz: 0
```

This gives us a signature for `b`: `1,1,1,0,0,0,0,0,0`. If we now do this same thing for every node, we get:

```
a: 1,0,1,0,0,0,0,0,0
b: 1,1,1,0,0,0,0,0,0
c: 1,1,0,0,0,0,0,0,0
d: 0,1,0,0,0,0,0,0,0
e: 0,0,1,0,0,0,0,0,0
```

This forms a vector database, which is amenable to many different kinds of p2p structures, including higher dimensional range query structures. We could then convert our queries into sets of such vectors, and use a vector lookup mechanism to constrain candidate values.

Indeed, any mechanism by which we can create a kind of "signature" of a node or a subgraph, based on its structural properties within the large graph, would let us do this kind of query.

Such a mechanism would be especially valuable for distributed analogs of WikiData, or the now defunct Freebase, whose knowledge graph was acquired by Google to power their own knowledge graph system.