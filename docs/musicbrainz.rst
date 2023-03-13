#######################
Coming from MusicBrainz
#######################

This page describes the key differences between the BookBrainz and MusicBrainz
schemas, for developers already familiar with the MusicBrainz schema.

Entities
========
There are 6 entity types in the BookBrainz world: Author, Work, Edition Group, Edition, Publisher and Series.
For a better description of each entity see :ref:`entities-description`.


Versioning
==========
Entities in BookBrainz are versioned, which means that the entire editing history
is stored in the database. To make this possible, we have a number of additional tables
in the database.
See a more complete description in :doc:`schema`

In short, each entity type has its own ``_header``, ``_revision`` and ``_data`` tables
(i.e ``author_header``, ``author_revision``, ``author_data``). The ``_header`` points to
the latest ``_revision``(each modification of one or more entities creates a new revision),
and the _revision points to the ``_data`` for that revision.
An additional revision_parent table allows us to keep a tree of modifications.

In additions to the versioning system, "deleting" an entity in BookBrainz is only ever a "soft delete",
meaning we mark the entity as deleted but keep all their editing history.
This allows us to resurrect 'deleted' entities if need be.

Aliases
=======

Each entity has an Alias Set containing one or more aliases. An entity's 'name' as well as other names
(authors' aliases, names in other languages and scripts, etc.) are all represented with the same ``alias`` table.
For convenience and ease of access, an entity will store its "default" alias ID to avoid having to fetch the Alias Set.

Sets
====
Inside that data, some elements like aliases (names), relationships and identifiers come in ``sets``.
A set contains one or more element; any change to the elements contained in the set (modification, addition or deletion)
creates a new set. That new set contains the unchanged elements as well as new elements for any modified item.

For example, let's imagine an Author has an Alias Set with ID 1 containing one alias with ID 123.
If we modify this alias, a new Alias Set with ID 2 will be created, containing a new alias with ID 124.
The new revision for this author will point to a new author_data row with AliasSet ID 2.

If we want to revert this change, we can simply create a new revision that either points to the previous author_data ID,
or a new author_data row with AliasSet ID 1.

Merging
=======
Merging in BookBrainz is very similar to regular edits: we create a new revision for each entity, one of which will contain
the new aggregated data from the merge. The new revision will be marked as being a merge operation, as well as each author_revision
(using Authors as an example) for each of the merge entity.

Let's say we're merging Author B and Author C into Author A. We select the correct value if there are any differences between entities,
and the sets are combined (we add together the aliases, relationships, and identifiers).
We create a new revision and three new author_revision, all marked as a merge operation.

Author A's author_revision entry will point to a new author_data ID with the combined data described above.
Author B and C's revisions will have their author_data ID set to NULL, the equivalent of a soft delete.
With the help of the revision_parent table, we can infer that Authors B and C previously existed but were merged into Author A.
The versioning system also gives us all the data we would need to revert this merge operation.