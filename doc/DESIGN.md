Design goal
===========

Preamble
--------
The main goal of this library is to provide some simple structures client-side
that are made persistent using a remote server.
All these structures could be done using standard relational database, and some
could be achieved using NoSQL-like databases.
However, achieving the secondary goals described below requires more involvement
when using these solutions.

For a more full-fledged, less restricted, but maybe slightly more complex
solution one could look into [typeorm](https://typeorm.io),
[sql.js](https://github.com/kripken/sql.js/) or
[PouchDB](https://pouchdb.com/).

Goals
-----

### Primary
The main goal is to provide a few structures organised in groups that can
reference each others.
In terms of a relational database it can be seen as specialized tables: instead
of having the ability to specify a model and add arbitrary rows, a few behavior
are enforced.
These behaviors controls how elements can reference each other and how conflicts
are merged.

Again, in terms of a relational database, this mainly mean that we allow only
a few options to handle primary keys and only one option for foreign key: using
the primary key.
This also borrows a bit from NoSQL because rows can have arbitrary content in
addition to a few restricted properties used to identify cross-references.

### Secondary
In addition to the actual data to handle, here are the other goals this library
aims to reach: transparent persistence, confidentiality from the remote server
and handling short offline periods.

Transparent persistence is achieved by using a remote server that simply
get a copy of each write operations and is able to provide remote instances with
all the data they missed (eventually from the start).

Confidentiality is done by providing hooks to transform data before they are
sent to the server and when they are retrieved.
Applying client-side encryption at this point provides a good level of
confidentiality.
Some data needs to remain visible, especially row identifiers and references so
that the system can operate but aside from that all the payloads can be
transformed.

Handling offline periods is done by keeping a log of activity on the client
before sending updates to the server.
If a remote client is stuck without connectivity to the server, this logs is
kept until connectivity is restored at which point it can be sent and discarded
locally.

The server itself does not need to know what each remote instances knows.
They just send the identifier of the last update they got and retrieve all data
from that point onward.

Available structures
--------------------
Each row can have an optional identifier that can be used to find a specific
value.
These identifiers are not required to be unique.

### Set (unordered list)
A set is a collection of elements in a group, with no particular requirement for
ordering.
It is possible to add, edit, and delete entries without hassle.
The actual key of each element is an integer defined by the local instance on
creation, which is updated by the server when it gets the update to their
"final" value.

### List (ordered by time of creation)
An ordered list where creation date defines the order of all rows.
It is possible to add, edit and delete entries which gets automatically sorted.
When pushing to the server, the new elements are re-merged by the server, and
their keys are updated.
If two rows have the exact same timestamp, previously known rows from the server
will have lower keys value.

### References
Each row in a Set or a List is able to reference other rows in other structures.
When such a reference exists, a deletion policy must be set.
Either we can delete the row that reference a deleted row, or we can remove the
reference.
These references are based on the rows identifiers, and there can be multiple
references to different rows in the same structure, as well as references to
rows in multiple structures.
The server keep tracks of these references and make sure they stay coherent.

Querying
--------
The query system is very basic: you can either directly reference a row via its
identifier, via its key in a structure, via a known reference or via arbitrary
data matching.

### Identifier
The simplest way to find a row is by using its defined identifier.
If multiple rows of the same structure share the same identifier, they should
all match.

### Key
The key is a relatively unstable value, but between updates from the server it
won't change much and can be used to retrieve a unique row.

### Known reference
It is possible to query for each row that reference a known row, or for each
rows that are referenced by a known row.

### Arbitrary data matching
Some basic test (equality, regex, number comparison) can be implemented on the
actual data from each row in a structure.

Deletion and unused space
-------------------------

### Deletion
Some rows can be set to delete all rows that reference them when they are
deleted (think of it as a cascade delete).
This is fine until two remote instances try to push conflicting changes on the
server.
Usually, if one instance delete a row, other instances will also receive order
to delete that row.

However, if one instance deletes a row, and another instance add a reference to
the now deleted row, an issue arise: the first remote instance don't have that
row anymore, the server have deleted the row, but the second remote instance
references it.
The server will not actually delete data when a row is deleted, but instead mark
it as deleted.
If a new reference arrive on a row that is marked as deleted, it is then
resurrected.

### Unused space
Keeping track of all deleted rows inevitably lead to lost space.
A basic "clean" operation can be triggered to trim all the deleted rows
definitely.
This would break remote clients that are offline at the time of the clean and
still reference deleted rows; however that should be a very rare case.
Should it happen, the server would simply refuse the update from the client and
return an appropriate error code.
