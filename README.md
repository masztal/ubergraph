# Ubergraph

Ubergraph is a versatile, general-purpose graph data structure for Clojure.

## Features

* Ubergraph supports directed edges, undirected edges, weighted edges, node and edge attributes.
* Ubergraph implements all of Loom's protocols.
* Ubergraph goes beyond Loom's protocols, allowing a mixture of directed and undirected edges within a single graph,
multiple "parallel" edges between a given pair of nodes, multiple weights per edge, and changeable weights.

Ubergraph is a great choice for people who:

* Want to use Loom, but don't want to think about which specific Loom graph implementation to use.  (Hmmm, do I need directed edges or undirected edges?  Do I need weights or attributes for this project?  I'm not sure yet, so I'll just use ubergraph because it can do it all.)
* Need graph capabilities beyond what Loom provides.
* Are implementing algorithms for Loom, and want to test the algorithm against an alternative graph implementation to be certain you've properly programmed the algorithm against the necessary abstractions, rather than Loom's concrete representations.

## Quickstart

Add the following line to your leiningen dependencies:

	[ubergraph "0.1.0"]

Require ubergraph in your namespace header:

	(ns example.core
	  (:require [ubergraph.core :as uber]))

Ubergraph lists loom as a dependency, so when you add ubergraph to your project, leiningen will also download loom.  This means all of loom's namespaces are also available to you in your program.  Loom is currently organized in a way that its protocols are split across a couple different namespaces.  As a convenience, ubergraph.core provides access to all of loom's protocol functions through its own namespace.

For example, rather than calling `loom.graph/out-edges` and `loom.attr/add-attr`, you can just call `uber/out-edges` and `uber/add-attr`.

## Usage

There are four flavors of Ubergraphs: graph, digraph, multigraph, and multidigraph.  All share the same underlying representation, but have different default behaviors with respect to adding new edges and attributes.  Specifically:

Graphs and Digraphs do not allow duplicate or "parallel" edges, whereas Multigraphs and Multidigraphs do.

Digraphs and Multidigraphs, by default, treat new edge descriptions as directed/one-way edges, whereas Graphs and Multigraphs, by default, treat new edge descriptions as bidirectional.  It is possible, however, to override this default behavior and add directed edges to a an undirected graph, and undirected edges to a directed graph.

<table>
  <tr>
    <th></th>
    <th>Allows parallel edges<br></th>
    <th>No parallel edges<br></th>
  </tr>
  <tr>
    <td>Directed Edges (by default)<br></td>
    <td>Multidigraph</td>
    <td>Digraph</td>
  </tr>
  <tr>
    <td>Undirected Edges (by default)<br></td>
    <td>Multigraph</td>
    <td></td>
  </tr>
</table>

All ubergraph constructors are multiple arity functions that can take an arbitrary number of "inits" where an init is defined as one of:

+ Edge description:
++ [src dest]
++ [src dest weight]
++ [src dest attribute-map]
+ Adjacency map (e.g., {1 [2 3], 2 [3]} adds the edges 1->2, 1->3, and 2->3).
+ Weighted adjacency map (e.g., {:a {:b 2, :c 3}} creates an edge of weight 2 between :a and :b, etc.)
+ Another ubergraph
+ Node (anything that doesn't fit one of the other patterns is interpreted as a node)

### Graphs

Ubergraphs built with the graph constructor treat every edge as a bidirectional, undirected edge.

```clojure
(def graph1
  (uber/graph [:a :b] [:a :c] [:b :d]))

=> (uber/pprint graph1)
Graph
4 Nodes:
	 :d
	 :c
	 :b
	 :a
3 Edges:
	 :b <-> :d
	 :a <-> :c
	 :a <-> :b
```

Edge descriptions can include weights.

```clojure
(def graph2
  (uber/graph [:a :b 2] [:a :c 3] [:b :d 4]))

=> (uber/pprint graph2)
Graph
4 Nodes:
	 :d
	 :c
	 :b
	 :a
3 Edges:
	 :b <-> :d {:weight 4}
	 :a <-> :c {:weight 3}
	 :a <-> :b {:weight 2}
```

Note that ubergraph differs from Loom in the way that it handles weights.  *Ubergraph simply stores weights in the edge's attribute map, under the keyword :weight.*  In my opinion, this is a superior way to handle weights because it allows you to manipulate weights using the same interface that you use to alter other attributes.  Also, once weight is no longer a privileged field, it makes it easier to develop algorithms that take as an additional input the attribute to use as the edge weight.  Right now, Loom algorithms have a baked-in notion that we only want to do traversals based on weight, and this is a problem.  Ideally, we want it to be just as easy to search a graph for shortest traversals using other attributes such as :cost or :distance.  However, to be compatible with Loom's protocols, Ubergraphs support the protocol function `weight` which simply extracts the `:weight` attribute from the attribute map.

```clojure
(def graph3
  (uber/graph [:a :b {:weight 2 :cost 200 :distance 10}]
              [:a :c {:weight 3 :cost 300 :distance 20}]))

=> (uber/pprint graph3)
Graph
3 Nodes:
	 :c
	 :b
	 :a
2 Edges:
	 :a <-> :c {:weight 3, :cost 300, :distance 20}
	 :a <-> :b {:weight 2, :cost 200, :distance 10}
```

You can extend graphs with more "inits" by using `build-graph`, or you can use `add-nodes` or `add-edges` (which takes any legal edge descriptors).  `add-nodes*` and `add-edges*` are variants which take a sequence of nodes/edges rather than multiple args.

```clojure
(build-graph graph1 [:d :a] [:d :e])
(add-edges graph1 [:d :a] [:d :e])
(add-edges* graph1 [[:d :a] [:d :e]])
(add-nodes graph1 :e)
(add-nodes* graph1 [:e])
```

Adding nodes that already exist will do nothing.  Graphs do not permit "parallel edges", so adding edges that already exist will cause the new attribute map to be merged with the existing one.  However, if you really want to, you can add a directed edge to an undirected graph:

```clojure
=> (uber/pprint (uber/add-directed-edges graph1 [:a :d]))
Graph
4 Nodes:
	 :d
	 :c
	 :b
	 :a
4 Edges:
	 :b <-> :d
	 :a -> :d
	 :a <-> :c
	 :a <-> :b
```

Ubergraph supports all of Loom's protocols, so you can do all the things you'd expect to be able to do to graphs:
nodes, edges, has-node?, has-edge?, successors, out-degree, out-edges, predecessors, in-degree, in-edges, transpose, weight, add-nodes, add-nodes*, add-edges, add-edges*, remove-nodes, remove-nodes*, remove-edges, remove-edges*, and remove-all.

#### Edges

However, one way that Ubergraph improves upon Loom is that graph edges have a richer implementation and support additional abstractions.  This richer implementation is what allows directed and undirected edges to coexist, and allows for parallel edges in multigraphs.  So where Loom functions return [src dest] tuples or [src dest weight] tuples to represent edges, Ubergraph returns actual Edge or UndirectedEdge objects.  For example,

```clojure
=> (-> (uber/graph [:a :b])
     (uber/add-directed-edges [:a :c])
     uber/edges)

(#ubergraph.core.UndirectedEdge{:id #uuid "3969fb54-0645-46e4-bad8-89cc28cee7cb", :src :b, :dest :a, :mirror? true} #ubergraph.core.Edge{:id #uuid "32d688aa-53cd-469f-a71b-3fa035d0e500", :src :a, :dest :c} #ubergraph.core.UndirectedEdge{:id #uuid "3969fb54-0645-46e4-bad8-89cc28cee7cb", :src :a, :dest :b, :mirror? false})
```

The main thing to note here is that internally, all edges have a `:src` field, a `:dest` field, and a uuid, which you can think of as a pointer to the map of attributes for that edge.  The other thing to note is that undirected edges are stored internally as a pair of edge objects, one for each direction.  Both edges of the pair share the same attribute map and one of the edges is marked as a "mirror" edge.  This is critical because in some algorithms, we want to traverse over all edges in both directions, but in other algorithms we only want to traverse over unique edges.  Loom provides no mechanism for this, but Ubergraph makes this easy with the protocol function `mirror-edge?`, which returns true for the mirrored edge in an undirected pair of edges, and false for directed edges and the non-mirrored undirected edges.  So `(edges g)` gives you all the edges in a graph, and `(filter (complement mirror-edge?) (edges g))` would give you a sequence of unique edges, without listing both directions of the same undirected edge.

When writing algorithms over edges, it is strongly recommended that you access the edge endpoints through the edge abstraction, using the protocol functions `src` and `dest`.  

Ubergraph edges also support the following abstractions:

## Relationship to Loom

Loom is currently the most important graph library in the Clojure ecosystem.  Loom strives to achieve three goals:

1. Loom factors graph behavior into a series of Clojure protocols.
2. Loom provides default graph implementations for each of these protocols.
3. Loom provides a library of algorithms that can operate on any graph that implements the relevant abstractions for that algorithm.

Graphs are an important part of my work, and I was disappointed when I discovered that Loom couldn't do many of the things I wanted it to do.  For example, I couldn't mix directed and undirected edges, I couldn't have multiple edges between a given pair of nodes, and I couldn't update weights associated with a given edge or have multiple notions of "cost" associated with a given edge.

My first thought was, "Hey, I'll just make an implementation of multi-edge graphs and submit it to Loom."  But as I started looking carefully at Loom's protocols and algorithms, I realized things weren't quite that simple:

1. Many of Loom's protocols were hard-coded to the assumption that graphs had only a single edge between a pair of nodes.
2. Many of Loom's algorithms were hard-coded to the assumption that graphs had only a single edge between a pair of nodes.

In other words, the protocols didn't even support the behavior necessary to implement a multi-edge graph, and even if the protocols were enhanced, very few of the algorithms would actually work on multi-edge graphs -- too many assumptions were sprinkled throughout the code base that [src dest] vectors are enough to uniquely identify a given edge.

I decided to embark on a five-step plan to improve Loom to cover multi-edge and other complex graph needs.

### Step 1

Submit pull request to Loom to support an Edge abstraction, and to update the built-in protocol functions to accept and return things that support that abstraction.  Step completed, and pull request has been accepted to Loom (Thanks Aysylu!)

### Step 2

Rewrite Loom's algorithms to use this Edge abstraction.  If this is done, then the algorithms will work properly on both single-edge and multi-edge graphs.  Unfortnately, I can't keep up with this on my own -- people keep contributing new "broken" algorithms (i.e., algorithms which assume a single edge between nodes and ignore the new Edge abstraction) to Loom faster than I can possibly fix them.  I'm going to need help from the community for this, so...

### Step 3

Write ubergraph as a concrete implementation of all the graph behvior that Loom currently supports, as well as all the behavior I'd like it to support.  That's a big part of what this library is all about.  Ubergraph provides the authors of Loom's algorithms with a rich graph data structure that has a very different internal structure than Loom's -- offering a way to test the algorithms to ensure they really are programmed to the correct protocol abstractions, and not some underlying notion of Loom's concrete implementation.

### Step 4

Establish an ubergraph.alg namespace where the community can help "curate" Loom's algorithms, identifying algorithms from loom.alg that work out of the box on ubergraphs, modifying algorithms from loom.alg that need modification, and writing new algorithms that leverage ubergraph's added capabilities.

### Step 5

Ideally, I'm hoping that once all of Loom's algorithms have been successfully converted to work with multi-edge graphs, Ubergraph can be merged into Loom as the default multi-edge graph implementation.  If that turns out not to be practical, ubergraph will continue to exist independently as a kind of "Loom+", a graph data structure that works with all of Loom's protocols and algorithms, as well as supporting additional functionality.



## License

Copyright (C) 2014 Mark Engelberg (mark.engelberg@gmail.com)

Distributed under the [Eclipse Public License](http://opensource.org/licenses/eclipse-1.0.php), the same as Clojure.