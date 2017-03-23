# CRDT-research

> Discussions and Planning about getting CRDT implementation on top of IPFS & libp2p

# tl;dr;

CRDT, or Conflict-Free Replicated Data Types, is a type of specially-designed data structure used to achieve strong eventual consistency (SEC) and monotonicity (absence of rollbacks).

# Discussions

- Previous discussions on ipfs/notes
  - https://github.com/ipfs/notes/issues/40#issuecomment-194899389

# CRDT Research Review (papers, books, talks, lectures, etc)

### Papers

- [Conflict-free replicated data types](https://scholar.google.pt/citations?view_op=view_citation&hl=en&user=NAUDTpMAAAAJ&citation_for_view=NAUDTpMAAAAJ:M3ejUd6NZC8C)
- [Merging OR and CRDT Algorithms](http://dl.acm.org/citation.cfm?id=2596636)

### Books

### Lectures

### Talks

- ["CRDTs Illustrated" by Arnout Engelen](https://www.youtube.com/watch?v=9xFfOhasiOE)
- [Coding CRDT](https://www.youtube.com/playlist?list=PLzUeAPxtWcqxBXjUelmcm5ORVjEpbUlHH)
- [Dmitry Ivanov & Nami Naserazad - Practical Demystification of CRDT (Lambda Days 2016)](https://www.youtube.com/watch?v=PQzNW8uQ_Y4)
- [ElixirConf 2015 - CRDT: Datatype for the Apocalypse by Alexander Songe](https://www.youtube.com/watch?v=txD1tfyIIvY)

### Notes and blog posts

- [Conflict-Free Replicated Data Types (CRDTs), An Offline Camp passion talk](https://medium.com/offline-camp/conflict-free-replicated-data-types-crdts-2c6ae67ab9a4#.duh4g0r9k)
- [CRDT Notes by Paul Frazee](https://github.com/pfrazee/crdt_notes)

### Available libraries and systems using CRDT

- http://y-js.org/
- http://swarmjs.github.io/
- http://gun.js.org/enterprise/
- http://scuttlebot.io/
- https://github.com/mafintosh/hyperlog
- https://github.com/orbitdb

### Terms

There is an [IPFS Glossary](https://github.com/ipfs/glossary), a work in progress, which should have definitions for terms used in CRDT. If you are consistently running into terms that you do not know the meaning of, please open an issue on that repository and we can work on a definition that will help you (and others!) going forward.
