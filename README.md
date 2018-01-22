# CRDT Research

> Discussions and Planning about getting CRDT implementation on top of IPFS & libp2p

# tl;dr;

CRDT, or Conflict-Free Replicated Data Types, is a type of specially-designed data structure used to achieve strong eventual consistency (SEC) and monotonicity (absence of rollbacks).

# Discussions

- Previous discussions on ipfs/notes
  - https://github.com/ipfs/notes/issues/40#issuecomment-194899389

# CRDT Research Review (papers, books, talks, lectures, etc)

### Papers

- [Conflict-free replicated data types](https://scholar.google.pt/citations?view_op=view_citation&hl=en&user=NAUDTpMAAAAJ&citation_for_view=NAUDTpMAAAAJ:M3ejUd6NZC8C)
- [A comprehensive study of Convergent and Commutative Replicated Data Types](http://hal.upmc.fr/inria-00555588/document)
- [Merging OT and CRDT Algorithms](http://dl.acm.org/citation.cfm?id=2596636)
- [CRDTs: Making δ-CRDTs Delta-Based](http://novasys.di.fct.unl.pt/~alinde/publications/a12-van_der_linde.pdf)
- [Key-CRDT Stores](https://run.unl.pt/bitstream/10362/7802/1/Sousa_2012.pdf)
- [TRVE Data: Placing a bit less trust in the cloud](https://www.cl.cam.ac.uk/research/dtg/trve/)
- [LSEQ: an Adaptive Structure for Sequences in Distributed Collaborative Editing](https://hal.archives-ouvertes.fr/hal-00921633/document)
- [A Conflict-Free Replicated JSON Datatype](https://arxiv.org/pdf/1608.03960.pdf)

### Books

### Lectures

### Talks

- ["CRDTs Illustrated" by Arnout Engelen](https://www.youtube.com/watch?v=9xFfOhasiOE)
- [Coding CRDT](https://www.youtube.com/playlist?list=PLzUeAPxtWcqxBXjUelmcm5ORVjEpbUlHH)
- [Dmitry Ivanov & Nami Naserazad - Practical Demystification of CRDT (Lambda Days 2016)](https://www.youtube.com/watch?v=PQzNW8uQ_Y4)
- [ElixirConf 2015 - CRDT: Datatype for the Apocalypse by Alexander Songe](https://www.youtube.com/watch?v=txD1tfyIIvY)
- [GOTO 2016 • Conflict Resolution for Eventual Consistency • Martin Kleppmann](https://www.youtube.com/watch?v=yCcWpzY8dIA)
- [CRDTs in IPFS](https://www.youtube.com/watch?v=2VOF-Z-nLnQ)

### Notes and blog posts

- [CRDT Tutorial for Beginners](https://github.com/ljwagerfield/crdt)
- [Conflict-Free Replicated Data Types (CRDTs), An Offline Camp passion talk](https://medium.com/offline-camp/conflict-free-replicated-data-types-crdts-2c6ae67ab9a4#.duh4g0r9k)
- [CRDT Notes by Paul Frazee](https://github.com/pfrazee/crdt_notes)
- [Towards a unified theory of Operational Transformation and CRDT by Raph Levien](https://medium.com/@raphlinus/towards-a-unified-theory-of-operational-transformation-and-crdt-70485876f72f)
- [A simple approach to building a real-time collaborative text editor](http://digitalfreepen.com/2017/10/06/simple-real-time-collaborative-text-editor.html)


### Available libraries and systems using CRDT

- http://y-js.org/
- http://swarmjs.github.io/
- http://gun.js.org/enterprise/
- http://scuttlebot.io/
- https://github.com/mafintosh/hyperlog
- https://github.com/orbitdb

### CRDT libraries using IPFS

- Yjs through [y-ipfs-connector](https://github.com/pgte/y-ipfs-connector)
- [ipfs-log](https://github.com/orbitdb/ipfs-log), append-only log CRDT used in [OrbitDB](https://github.com/orbitdb/orbit-db)

### Terms

There is an [IPFS Glossary](https://github.com/ipfs/glossary), a work in progress, which should have definitions for terms used in CRDT. If you are consistently running into terms that you do not know the meaning of, please open an issue on that repository and we can work on a definition that will help you (and others!) going forward.
