technical documentation
---------------------------

rest-api
~~~~~~~~~~

We have a main api for interacting with the database.

Each object has its URL, which really identifies the object (e.g.: plant #1 for
accession 101 in year 2001)::

  /garden/accession/2001.0101/plant/1/

Removing the object's trailing identificator from the URL gives the class
URL (e.g.: the plants collection)::

  /garden/accession/2001.0101/plant/

The trailing slash is part of the URL, but the server will add it if it's
missing.

**collections**

We organized the objects in three sections: ``taxonomy``, ``collection``,
``garden``.  There might come some day a ``herbarium`` or ``seedbank``
section, or we may reorganize in fewer sections, we will see.  As of now, we
have these collections::

  /taxonomy/rank/
  /taxonomy/taxon/
  /collection/accession/
  /collection/contact/
  /collection/accession/<code>/verification/
  /garden/accession/<code>/plant/
  /garden/accession/<code>/plant/<code>/propagation/
  /garden/location/

Verifications and Plants only make sense in combination with an accession,
so their collections are behind an accession code.  Same for Propagations,
which only make sense in relation with the mother plant.

**individual objects**

Append a primary key to a collection URL, and you get the URL for an
individual within the collection.

As far as their URLs are concerned, ``rank``, ``taxon``, ``contact`` have a
primary key which is a sequential number, with no semantics.

Accessions have their own accession code, Plants have a sequential plant
code within the Accession they belong to, Verifications also have a unique
sequential number within the Accession they describe.  Propagations have a
sequential number within their mother Plant.

.. note::

   If we generalize the database to model more than one garden, we will need
   to associate accessions to gardens, we will probably identify gardens
   with a stub, and will prepend accession urls with a garden stub code.  As
   of now, we only deal with a single garden.

**GET and her sisters**

Collection URLs implement the ``GET`` and ``POST`` verbs, respectively for getting
the whole collection (or a selection thereof), and for adding an individual
object to the collection.  These URLs get a ``-list`` suffix in their Django
name.

Individual URLs implement the ``GET``, ``PUT`` and ``DELETE`` verbs, with
their obvious meanings, applying to the specific individual only.  These
URLs get a ``-detail`` suffix in their Django name.

**more URLs**

Collections also have an URL for the empty html form, to be populated by
the user and posted to the server.  The corresponding Django names have
suffix ``-post-form``.

Individual objects have more entry points, respectively for:

- The populated html form (django suffix ``-form``)
- A json data dictionary for the infobox (django suffix ``-infobox``)
- A dictionary with several representations for the same object (django suffix ``-markup``)
- A json data dictionary with *depending* objects, and the definition of the
  concept depends on the object.  A Location considers the plants located
  there as its depending objects, a Taxon its subtaxa **and** the accessions
  verified to it.  The result has the same shape as the dictionary returned
  by a search.  (django suffix ``-depending``)
- A rendered html page with object pictures (django suffix ``-carousel``)


**search API**

``filter/`` and ``get-filter-tokens/`` are the main query api entry point.
Both expect a ``q`` parameter, which they interprets according to several
search strategies.  Search strategies are described in some detail in the user
manual.

The result of a ``get-filter-tokens/`` request is a dictionary, where the keys
are the names of the collection in the result, and the values are *tokens*.
You get as many tokens as the non-empty collections matching your query.

The next step on the client side is to enter a loop to *cash* your *tokens*.
Each invocation of the ``cash-token/<token>/`` returns you a dictionary with
three entries:

- ``chunk`` holds the list of items.
- ``expect`` specifies the length of the expected complete set.  One possible
  use is to update a progress bar.
- ``done`` tells you whether this was the last chunk.

Attempting to cash a token which was already paid in full will provide the
empty result.  Same will happen if you attempt to cash an invalid token.  The
empty result is ``expect:0``, ``done:True``, ``chunk:[]``.

If you are somewhat too quick in cashing a new token, the ``expect`` value
could still be a large hard-coded value.  The correct value is computed in a
separate thread, so the server can provide all tokens as soon as possible.

Tokens will expire after some delay in cashing them.  This prevents queries to
stay active in the system while not any more relevant.

For queries where you expect a small result set (less than ~70 elements), you
can may prefer the ``filter/`` entry point.  ``filter`` short-circuits this
process, providing the concrete result at once, in a dictionary having the
same external structure as the ``get-filter-tokens`` result, one list of
objects per non-empty collection, and values as the above ``chunk`` lists.

One more entry point in this group is ``count/``, it accepts the same
parameters as ``filter`` and ``get-filter-tokens``, and returns a dictionary
with same external structure.  The values in this case are the matching query
``count()``, plus a grand total under the key ``__total__``.  You can use this
to decide whether to use ``filter`` or the chunked approach
``get-filter-tokens``.

On the server side, executing a search corresponds to constructing one or more
queryset.  Each element in the queryset is subsequently converted into a
dictionary, with the structure:

:inline: The string shown in the result.  It may contain html tags.
:twolines: Three elements to be shown in different parts of the client.
:infobox_url: The url to get the corresponding infobox.

The ``inline`` and ``twolines`` entries are meant to be included in the
results box.  The ``infobox_url`` provides quick access to the URL where we
will get the infobox data, but you can just replace the trailing *infobox/*
part and replace with whatever other valid suffix.  at the moment of writing,
the URLs implemented are *form/*, *markup/*, *depending/*.

importing from ghini.desktop
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Please consider this work in progress, try out the instructions, and be
prepared to ask for help or to open an issue if the present instructions do
not work.

First of all: taxasoft-ghini is not complete, not yet.  The current goal is
to have it do something useful, and to be visible on-line, it does not (yet)
substitute ghini.desktop.  Not at all.  Expect things to be exciting, but do
not expect things to work out of the box.

Got this?  Good, now let's see how to copy your ghini.desktop collection
into taxasoft-ghini!

from ghini.desktop
.................................

#. open ghini-1.0

   #. export your (complete) data to csv.

#. close ghini

#. open ghini-1.0 again,

   #. create a new sqlite3 connection,
   #. let ghini create the database.
   #. import the data, this will again initialize the database.

#. close ghini

   the result of the above steps is an expendable sqlite3 database: this way
   whatever we do on it, it has zero impact on your original data.

#. remove all taxonomic information that is not used.  we do this straight
   on the expendable database::

     sqlite3 ghini.db
     delete from genus where id not in (select genus_id from species);
     delete from family where id not in (select family_id from genus);
     delete from genus_synonym where genus_id not in (select id from genus);
     delete from genus_synonym where synonym_id not in (select id from genus);

#. consider removing history too, it's not imported anyway::

     delete from history;

#. open ghini.desktop-1.0

   #. export your (reduced) data to csv.

      this will take a fraction of the time for the previous export.

#. close ghini

now to taxasoft-ghini
.................................

#. enter the directory of your check-out;
#. activate the virtual environment;
#. move any previous database out of the way;
#. create a new database and initialize it::

     ./manage.py migrate

#. consider whether you also want the intermediate taxa, between ranks
   familia and genus.  since importing this information takes rather long,
   it is not included in the 'migration' command.  if you want this data,
   you must request the import explicitly, with::

     ./manage.py import_genera_derivation

   have something else to do in the meanwhile, this will take no less than
   one full hour.  on my laptop, writing to a sqlite3 database, it lasts 2
   hours.

   if you're in a hurry, ask for a partial genus import, limiting to the
   genera in your trimmed database::

     ./manage.py import_genera_derivation --filter-genera <your genus.txt file>

   you can repeat the command without filtering, whenever you know you're
   not going to use the database for a couple of hours.

#. run the command::

     ./manage.py import_desktop <location of second export>

   this will output as many ``+`` as the objects it inserted, as many ``.`` as
   the objects it already found in place.  for species, a ``v`` is added if
   the related species is at lower rank.

   the genus list in particular, that should be just a sequence of dots.  if
   it is not, it's because you're importing genera that were not created
   during the previous steps.  that's clearly not good and you should review
   your data.

   the opposite goes for the species list: remember that with ghini reloaded
   fictive species are not any more needed.  A dot tells you that the
   corresponding taxon was found in the database, at some higher rank.

   it is normal that importing accessions takes longer: for each object we
   are creating not only the accession but also the verificaiton object that
   links the accession to the corresponding taxon.

#. create your superuser::

     ./manage.py createsuperuser

#. run your server::

     ./manage.py runserver

#. I'm sure there will be errors.  please open issues about them, and if you
   have a solution, propose it.
