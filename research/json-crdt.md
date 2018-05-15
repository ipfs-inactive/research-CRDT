# JSON-CRDT

This document explains the internals of JSON-CRDTs based on the 
[Conflict-Free weplicated JSON Datatype](https://arxiv.org/pdf/1608.03960.pdf) 
paper.

From the paper's abstract:

> [...] an algorithm and formal semantics for a JSON data structure that
> automatically resolves concurrent modifications such that no updates are lost,
> and such that all replicas converge towards the same state (a conflict-free
> replicated datatype or CRDT). It supports arbitrarily nested list and map
> types, which can be modified by insertion, deletion and assignment. The
> algorithm performs all merging client-side and does not depend on ordering
> guarantees from the network, making it suitable for deployment on mobile
> devices with poor network connectivity, in peer-to-peer networks, and in
> messaging systems with end-to-end encryption.

Conflict-Free Replicated Datatypes (CRDTs) are a family of data structures that 
support concurrent modifications in the data structure by different replicas.
CRDTs ensure no conflict when replica states are merged together and guarantee
eventual convergence. A JSON-CRDT is a distributed JSON document which 
automatically resolves concurrent modifications in the JSON structure without 
losing any update. The JSON-CRDT provide strong eventual consistency and does 
not depend on ordering guarantees from the network.

## Document editing API

## Supported types

A [JSON](http://json.org/) document is composed of maps, lists and registers 
which can be embedded. The JSON CRDT as presented in the paper is a JSON data
type in which maps, lists and registers are CRDTs too.

A register is a multi-value CRDT which keeps a map between the operation ID and
the value that the operation has assigned.

## Editing the JSON document and replica local state

Each replica keeps its own state of the document. Changes in the document 
occur only locally. When the document is edited, it generates an `operation` 
which uniquely describes the mutation globally. The operations are distributed 
to all peers as a mechanism to reach state consistency across all peers.

Once a replica receives an external operation, it applies the operation locally. 
The merging algorithm ensures that applying external operations locally will 
not conflict with the current local state.

The state of the document at any point in time is a set of operations which were
applied locally.

## Operations

The paper defines the JSON CRDT as an operation-based CRDT, in which an 
operation - a tuple representing a mutation in the JSON document - can be
propagated to other replicas without sending the whole local document state. The
local state of the document is represented by a set of individual operations.

The operation-based representation has the advantage that replicas need only to
share and merge singular operations, instead of the entire local state. On the 
other hand, whenever a new node joins the network, it still needs to receive the
whole document state, which grows linearly with the number of operations applied
to the document. This problem can be addressed with, for example,
[Delta State CRDTs](https://github.com/ipfs/research-CRDT/issues/31).

### Operation Representation

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
dotted version vectors. 

The implementation decision for which type of identifier to use 
should take into consideration that the set `deps` may grow impracticable if
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

### Applying Operations

Operations are always applied the local state. The operations to apply on the
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
set of `deps` kept by each node traversed. This aims at keeping a log in
each node of which operations have relied on it. A node is considered 
deleted form the local state when its set of dependency operations is empty.
When traversing the document, if the next node does not exist, the CRDT creates
the it, which may be of type `map` or `list`.

Once the traversal is complete, the next step is to apply `op` mutation to the
node. The mutation can be one of `INSERT`, `ASSIGN` or `DELETE`.

#### Deleting and clearing state

The `DELETE` and `ASSIGN` operations are destructive mutations and the CRDT must
ensure that concurrent modifications are not affected by leaving the node in the
document, but updating the node set of dependent operations. A deleted node is
not removed from the document, but rather a node in which `deps` set is empty.
These nodes - called [*tombstones*](https://github.com/ipfs/research-CRDT/issues/30) 
- need to be kept in the document so that new nodes may join the CRDT.

A register maintains a map between the operation ID and the value assigned by
the operations. Deleting/clearing a register consists of removing all the
`operation: value` key-value from the map which are part of the operations being 
applied. A register is considered deleted when its dependency set is empty.

Maps and lists follow a similar rationale: the `deps` set must be updated in the
cleared node and in all its children.

## Concurrent editing examples

## Further reading
[Conflict-Free weplicated JSON Datatype](https://arxiv.org/pdf/1608.03960.pdf) 
[Delta State CRDTs](https://github.com/ipfs/research-CRDT/issues/31)
[Lamport Logical clocks](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)
[Garbage-collection in op-based CRDTs](https://github.com/ipfs/research-CRDT/issues/30)

