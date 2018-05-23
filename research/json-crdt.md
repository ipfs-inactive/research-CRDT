# JSON-CRDT

This document explains the internals of JSON-CRDTs based on the 
[Conflict-Free weplicated JSON Datatype](https://arxiv.org/pdf/1608.03960.pdf) 
paper.

Conflict-Free Replicated Datatypes (CRDTs) are a family of data structures that 
support concurrent local modifications in different replicas in a way that there
are no conflicts when replicas are merged. It also ensures that the state of
each replica will eventually converge. A JSON CRDT ensures that a data structure
consistent with the [JSON](https://json.org) object standard and in which its 
basic types - lists, maps and registers - are CRDTs and can be embedded. From 
the paper's abstract:

> [...] an algorithm and formal semantics for a JSON data structure that
> automatically resolves concurrent modifications such that no updates are lost,
> and such that all replicas converge towards the same state (a conflict-free
> replicated datatype or CRDT). It supports arbitrarily nested list and map
> types, which can be modified by insertion, deletion and assignment. The
> algorithm performs all merging client-side and does not depend on ordering
> guarantees from the network, making it suitable for deployment on mobile
> devices with poor network connectivity, in peer-to-peer networks, and in
> messaging systems with end-to-end encryption.

## Document editing API

The interaction with the local JSON document is done through an API, which 
enables the user to programmatically retrieve and modify the document. Read-only
API calls do not produce any side effect in the data structure, but
modifying the document produces `operations` which uniquely identify the 
modifications in the document.

Example on how to use the API programmatically to construct and edit a JSON
document as described by the paper:

```javascript
doc.get("shopping") = []
let head = doc.get("shopping").idx(0)
head.insertAfter("eggs")
let eggs = doc.get("shopping".idx(1)
head.insertAfter("cheese")
eggs.insertAfter("milk")

// current state:
// {"shopping": ["cheese", "eggs", "milk"]}

doc.get("bought") = {}
doc.get("shopping").idx(1).delete()
doc.get("bought").idx(0).insertAfter("eggs")

// current state:
// {"shopping": ["cheese", "milk"], "bought": ["eggs"]}

doc.yield() // performs network transactions (sends/receives operations)
```

## Supported types

A JSON document is composed of maps, lists and registers 
which can be embedded. The JSON CRDT as presented in the paper is a JSON data
type in which maps, lists and registers are CRDTs and that can be embedded as
expected by the [JSON specs](https://json.org).

A register as defined in the paper is represented as a multi-value CRDT register
which keeps a mapping between the register values and the operation ID that set
it.

## Diving deep 

In this section we'll describe in more detail the JSON CRDT data structure and
algorithm that ensure strong eventual consistency and no user input loss across 
all replicas.

### Editing the JSON document and replica local state

Each replica keeps its own state of the document. Changes in the document 
occur only locally. When the document is edited, it generates an `operation` 
which uniquely describes the mutation globally. The operations are distributed 
to all peers as a mechanism to reach state consistency across all peers.

Once a replica receives an external operation, it applies the operation locally. 
The merging algorithm ensures that applying external operations locally will 
not conflict with the current local state.

The state of the document at any point in time is a set of operations which were
applied locally.

### Operations

The paper defines the JSON CRDT as an operation-based CRDT, in which an 
operation - a tuple representing uniquely a mutation in the JSON document - can be
propagated to other replicas without sending the whole local document state. The
local state of the document is represented by a set of individual operations.

The operation-based representation has the advantage that replicas need only to
share and merge singular operations, instead of the entire local state. On the 
other hand, whenever a new node joins the network, it still needs to receive the
whole document state, which grows linearly with the number of operations applied
to the document. This problem can be addressed with, for example,
[Delta State CRDTs](https://github.com/ipfs/research-CRDT/issues/31).

#### Operation Representation

An operation is a tuple of the form

```
op (
  id: N x ReplicaID
  deps: P(N x ReplicaID)
  cur: cursor(<k1,...,kn-1>, kn)
  mut: insert(v) | delete | assign(v)   v: VAL
)
```

where the `id` is a Lamport timestamp which identifies the operation uniquely; 
`deps` is a set of casual dependencies of the operation; `cur` describes the 
position of the document to be modified; and `mut` describes the modification 
itself.

- **Id**

The operation `id` is a Lamport timestamp (see appendix), which guarantees that
every operation is uniquely identified globally and that the peers implicitly
agree on the partial casual order of when the operations occurred, without any 
other synchronization mechanism.

- **Cursor**

A cursor `cur` describes the path from the root to a branch or leaf of the 
document in which the mutation will be applied. The cursor must consist only of 
immutable keys and identifiers.

If we consider the following JSON document

```
{"shopping": ["cheese", "eggs", "milk"]}
```

the cursor

```
cursor(<MapT(doc), listT("shopping")>, id1)
```

is evaluated to `eggs`


**Dependencies**

The dependencies `deps` is a set of identifiers of all operation that the current
operation depends on. The purpose of the operation dependencies is to ensure 
that the partial order of the operations is respected. 

In order to maintain the partial ordering of operations, an operation is applied 
on a local replica only when the local state has applied all the 
operation dependencies. If an external operation has been received and the set 
of dependencies is not fulfilled, the received operation is buffered until the 
dependencies operations are applied to the document.

The dependencies set can be implemented in several ways. The requirements are 
that `deps` contains information about all the operations that the current 
operations depends on.

The operation identifiers which are part of the `deps` set can be represented
as, for example, a set of Lamport timestamps, version vectors, state vectors or 
dotted version vectors.  The implementation decision for which type of identifier
to use should take into consideration that the set `deps` may grow impracticable if
the number of elements in the set grows linearly with the number of
operations applied to the document.

**Mutation**

A mutation `mut` describes the modification to be applied to the node
pointed by the `cursor`. The mutation type may be one of `INSERT`, `DELETE`
and `ASSIGN`. Each document type supports different types of mutation:

|        | MapT | ListT | RegisterT |
|--------|------|-------|-----------|
| INSERT | Yes  | Yes   | No        |
| ASSIGN | Yes  | Yes   | Yes       |
| DELETE | Yes  | Yes   | Yes       |

*Table: Document type and support to type of mutation*

The `INSERT` mutation inserts a new element to a list of a new key-value pair to
a map. The `ASSIGN` assignes a new value to a register, map, which overwrites 
the current structure values. The `ASSING` mutation may also be applied to a
register. The `DELETE` mutation deletes a map, list or register.

#### Applying Operations

Operations are always applied locally. The operations to apply on the
local state may be generated locally or received from other replicas. The Figure
1 shows an overview of the algorithm to apply local and remote operations on the
JSON document in a way that guarantees conflict free state updates with eventual
consistency and no loss of data.

![Applying operations overview](applying-operations-overview.png?raw=true "Figure 1. Applying operations overview")

When an operation `op` is received from a remote replica (remote operation case) 
the CRDT first makes sure that the operation was not applied already by checking
whether the `op` ID is part of a set containing all the operation IDs
applied to the local state. If the operation was not applied yet, the next step
is to make sure that the local state has applied all the operations that `op` is
is dependent on. This can be verified by comparing the `op` dependency set with
the set containing all the operation IDs applied to the local state. If one or
more dependent operation is missing in the current local state, the operation is
buffered and applied only when all the dependencies have been satisfied.

The `apply_local` action performs the state update in the document. It starts
by traversing the document from the root until the node represented by the `op`
cursor and where the mutation should be applied. The document traversal is done by 
descending from the document's root until reaching the node representing by the
`op` cursor. While traversing the document tree, the `op` ID is added to the 
set of `deps` kept by each node traversed. The goal is to keep a log in
each node of which operations have relied on them.

A node is considered  deleted form the local state when its set of dependency 
operations is empty. When traversing the document, if the next node does not 
exist, the CRDT creates the node,  which may be of type `map` or `list`.

Once the traversal is complete, the next step is to apply `op` mutation to the
node. The mutation can be one of `INSERT`, `ASSIGN` or `DELETE`.

#### Deleting and clearing state

The `DELETE` and `ASSIGN` operations are destructive mutations. In the operation
based JSON CRDT as presented in the paper, a deleted node is
not removed from the document but instead it is represented by a node in which 
it `deps` set is empty. A document node which has its `deps` set empty is
considered deleted and its called
[*tombstones*](https://github.com/ipfs/research-CRDT/issues/30).

The process of clearing the state of the nodes differs from its type. A register
maintains a map between the operation ID and the value assigned by
the operations. Clearing a register consists of deleting the map entries in the 
register that key is part of the operation `deps` set.

Maps and lists follow a similar rationale: the `deps` set must be updated in the
in the same way on cleared node and in all its children.

**Tombstones and garbage-collection**

The tombstone nodes must be kept in the data
structure to avoid inconsistencies between replicas which delete nodes that are
concurrently edited or used by other replicas. As a consequence, the data
structure may keep unnecessary nodes once all replicas are in sync regarding
deleted nodes. Ideally, the tombstone nodes should be safely removed from the
data structure to avoid wasted bandwidth, wasted storage space and in order to
speed up new entities joining the network. A possible solution for this problem
is to snapshot the data structure at times and safely garbage-collection 
tombstone nodes.

For more details and discussion about CRDT snapshot and garbage collection 
in operation based CRDTs, check [here](https://github.com/ipfs/dynamic-data-and-capabilities/issues/2)
and [here](https://github.com/ipfs/dynamic-data-and-capabilities/issues/14).

### Data structure metadata

In order for the JSON CRDT data structure to ensure strong eventual consistency
between all replicas and no user input losses, the data structure needs to
maintain metadata describing the operations which have been applied previously
at a document, node and operation level. The metadata aims at supporting 
applying operations locally.

At a document level, the data structure needs to keep a list of operation ID 
which have been applied locally. 

Each operation must keep a unique network-wide ID and its dependency set. At an 
operation level, the set of dependencies represents the operations that were
applied locally before the operation was created.

At a node level, each node in the document must keep a list of operation ID 
which the node depends on. Every time a node is traversed or created, the 
ID from the operation should be added to the set. If a node is deleted, the
operation ID and its dependencies must be removed from the node's dependencies 
set.

## Further reading

[Conflict-Free weplicated JSON Datatype](https://arxiv.org/pdf/1608.03960.pdf) 

[Delta State CRDTs](https://github.com/ipfs/research-CRDT/issues/31)

[Lamport Logical clocks](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)

[Garbage-collection in op-based CRDTs](https://github.com/ipfs/research-CRDT/issues/30)

[CRDT snapshotting](https://github.com/ipfs/dynamic-data-and-capabilities/issues/14)
