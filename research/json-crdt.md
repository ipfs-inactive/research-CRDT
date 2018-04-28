# JSON-CRDT

This document explains the internals of JSON-CRDTs based on the 
[Conflict-Free weplicated JSON Datatype](https://arxiv.org/pdf/1608.03960.pdf) 
paper.

JSON-CRDT is a distributed data type which automatically resolves concurrent 
modifications in the JSON structure without losing any update. The JSON-CRDT
provide strong eventual consistency and does not depend on ordering guarantees
from the network.

These properties make the JSON-CRDT a good fit for p2p applications or any
state-synchronizing application.

## Types

### MapT
### ListT
### RegisterT

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

|        | MapT | ListT | RegisterT |   |
|--------|------|-------|-----------|---|
| INSERT | Yes  | Yes   | No        |   |
| ASSIGN | No   | Yes   | Yes       |   |
| DELETE | Yes  | Yes   | Yes       |   |

*Table: Document type and support to type of mutation*

### Generating Operations

### Applying Operations

## Document editing API

## Concurrent editing examples

## Appendix

### Lamport Timestamps 

1.[Conflict-Free weplicated JSON Datatype](https://arxiv.org/pdf/1608.03960.pdf) 

