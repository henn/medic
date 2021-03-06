=====
medic
=====
-------------------------------------------------
a command-line tool to manage a mirror of MEDLINE
-------------------------------------------------

The Swiss Army knife to parse MEDLINE_ XML files or
download eUtils' PubMed_ XML records,
bootstrapping a local MEDLINE/PubMed database,
updating and/or deleting the records, and
writing the contents of selected PMIDs into flat-files.

Synopsis
========

::

  medic [options] CMD FILE|PMID...

  man medic
  medic --help
  medic --output /tmp parse baseline/medline*.xml.gz
  medic --update parse update/medline*.xml.gz
  medic --info delete delete.txt
  medic --url sqlite:///tmp.db insert pubmed.xml
  medic --pmid-lists update pmids_to_fetch_online.txt
  medic --all update medline13n1000.xml
  medic --format html write 1028734 1298474 > out.html
  medic --logfile log.txt write pmid_list.txt

Setup
=====

If you are **not** using ``pip install medic``, install all
dependencies/requirements::

  pip install sqlalchemy
  # only if using python3 < 3.2:
  pip install argparse 

Install the **DB driver** you prefer to use (supported are PostgreSQL
and SQLite, with the latter part of the Python StdLib)::

  pip install psycopg2 

Create the PostgreSQL database::

  createdb medline 

If you are fine working with SQLite, you only need to use the path to the
SQLite DB file in the URL option (that will implicitly "create" the DB)::

  medic insert --url sqlite:///tmp.db 123456

To create the tables in the DB, you can "try" to fetch a record: As the DB
is empty, this will not write anything, but SQL Alchemy will create the tables
for you in the DB::

  medic write 123 # for PostgreSQL
  medic --url sqlite:///tmp.db write 123 # for SQLite

Description
===========

``medic [options] COMMAND PMID|FILE...``

The ``--url URL`` option represents the DSN of the database and might
be needed (default: ``postgresql://localhost/medline``); For example:

PostgreSQL
  ``postgresql://host//dbname``
SQLite DB
  ``sqlite:////absolute/path/to/foo.db`` or
  ``sqlite:///relative/path/to/foo.db``

The five **COMMAND** arguments:

``insert``
  Create records in the DB by parsing MEDLINE XML files or
  by downloading PubMed XML from NCBI eUtils for a list of PMIDs.
  The insert fails if the record already exists in the DB.
``write`` *
  Write records as MEDLINE_ files to a directory, each file named as
  "<pmid>.txt". Alternatively, just the TIAB (title and abstract) plain-text
  can be output, and finally, a single file in TSV or HTML format can be
  generated (see option ``--format``).
  If the requested PMID does not exist in the DB, the command does not fail,
  but the relevant file, row, or element will not have been written.
``update``
  Insert or update records in the DB (instead of creating them); note that
  if a record exists, but is added with ``create``, this would throw an
  `IntegrityError`. If you are not sure if the records are in the DB or
  not, use ``update`` (N.B. that ``update`` is slower).
``delete`` *
  Delete records from the DB for a list of PMIDs (using ``--pmid-lists``)
``parse``
  Does not interact with the DB, but rather creates ".tab" files for each
  table that later can be used to load a database, particularly useful when
  bootstrapping a large collection.

\* Note that ``write`` and ``delete`` can only use PMID lists (option
``--pmid-lists``), so for these two commands, that option is always active
(implicitly).

For example, to download two PubMed records by PMID and update them in
the DB::

  medic update 100000 123456

Add a single MEDLINE or PubMed XML file to the database::

  medic insert pudmed.xml

Export a few records from the database as HTML (to STDOUT)::

  medic write --format html 292837491 128374 213487

Note that if the suffix ".gz" is present, the parser automatically
decompresses the XML file(s) first. This feature *only* works with
GNU-zipped files and the ".gz" suffix must be present.

Therefore, command line arguments are treated as follows:

integer values
  are always treated as PMIDs to download PubMed XML data
all other values
  are always treated as MEDLINE XML files to parse
  **unless** you use the option ``--pmid-lists``
files ending in ".gz"
  are treated as gzipped MEDLINE XML files

Requirements
============

- Python 3.2+
- SQL Alchemy 0.8+
- PostgreSQL 8.4+ or SQLite 3.7+

*Note* that while any DB supported by SQL Alchemy should work, all other DBs
are **untested**.

Loading MEDLINE
===============

Please be aware that the MEDLINE distribution **is not unique**, meaning that
it contains a few records multiple times (see the section about
**Version IDs**).

Parsing and loading the baseline into a PostgreSQL DB on the same machine::

  medic parse baseline/medline14n*.xml.gz

  for table in citations abstracts authors chemicals databases \
  descriptors identifiers keywords publication_types qualifiers sections; 
    do psql medline -c "COPY $table FROM '`pwd`/${table}.tab';";
  done

For the update files, you need to go *one-by-one*, adding each one *in order*,
and using the flag ``--update`` when parsing the XML. After parsing an XML file
and *before* loading the dump, run ``medic delete delete.txt`` to get rid of
all entities that will be updated or should be removed (PMIDs listed as
``DeleteCitation``\ s)::

  # parse a MEDLINE update file:
  medic --update parse medline14n1234.xml.gz

  # delete its updated and DeleteCitation records:
  medic delete delete.txt

  # load (COPY) all tables for that MEDLINE file:
  for table in citations abstracts authors chemicals databases \
  descriptors identifiers keywords publication_types qualifiers sections; 
    do psql medline -c "COPY $table FROM '`pwd`/${table}.tab';";
  done

Alternatively - simpler but slower - you can just ``update`` from the XML
directly::

  medic update medline14n1234.xml.gz

Version IDs
===========

MEDLINE has began to use versions to allow publishers to add multiple citations
for the same PMID. This only occurs with 71 articles from one journal,
"PLOS Curr", in the 2013 baseline, creating a total of 149 non-unique records.

As this is the only journal and as there may only be one record per PMID in the
database, alternative versions are currently being ignored. In other words, if
a MedlineCitation has a VersionID value other than "1", those records can be
skipped to avoid DB errors from non-unique records.

For example, in the 2013 baseline, PMID 20029614 is present ten times in the
baseline, each version at a different stage of revision. Because it is the
first entry (in the order they appear in the baseline files) without a
``VersionID`` or a version of "1" that is the relevant record, ``medic`` by
default filters citations with other versions than "1". If you do want to
process other versions of a citation, use the option ``--all``.

To summarize, *medic* by default **removes** alternate citations.

Database Tables
===============

Citation (citations)
  **pmid**:BIGINT, *status*:ENUM(state), *title*:TEXT, *journal*:VARCHAR(256),
  *pub_date*:VARCHAR(256), issue:VARCHAR(256), pagination:VARCHAR(256),
  *created*:DATE, completed:DATE, revised:DATE, modified:DATE

Abstract (abstracts)
  **pmid**:FK(Citation), **source**:ENUM(type), copyright:TEXT

Section (sections)
  **pmid**:FK(Medline), **source**:ENUM(type), **seq**:SMALLINT,
  *name*:ENUM(section), label:VARCHAR(256), *content*:TEXT

Author (authors)
  **pmid**:FK(Medline), **pos**:SMALLINT, *name*:TEXT,
  initials:VARCHAR(128), forename:VARCHAR(128), suffix:VARCHAR(128),

PublicationType (publication_types)
  **pmid**:FK(Medline), **value**:VARCHAR(256)

Descriptor (descriptors)
  **pmid**:FK(Medline), **num**:SMALLINT, major:BOOL, *name*:TEXT

Qualifier (qualifiers)
  **pmid**:FK(Descriptor), **num**:FK(Descriptor), **sub**:SMALLINT, major:BOOL, *name*:TEXT

Identifier (identifiers)
  **pmid**:FK(Medline), **namespace**:VARCHAR(32), *value*:VARCHAR(256)

Database (databases)
  **pmid**:FK(Medline), **name**:VARCHAR(32), **accession**:VARCHAR(256)

Chemical (chemicals)
  **pmid**:FK(Medline), **idx**:VARCHAR(32), uid:VARCHAR(256), *name*:VARCHAR(256)

Keyword (keywords)
  **pmid**:FK(Medline), **owner**:ENUM(owner), **cnt**:SMALLINT, major:BOOL, *value*:TEXT

- **bold** (Composite) Primary Key
- *italic* NOT NULL (Strings that may not be NULL are also never empty.)

Supported XML Elements
======================

Entities
--------

- MedlineCitation and ArticleTitle (``Medline`` and ``Identifier``)
- Abstract and OtherAbstract (``Abstract`` and ``Section``)
- Author (``Author``)
- Chemical (``Chemical``)
- DataBank (``Database``)
- Keyword (``Keyword``)
- MeshHeading (``Descriptor`` and ``Qualifier``)
- PublicationType (``PublicationType``)
- DeleteCitation (for deleting records when parsing updates)

Fields/Values
-------------

- Abstract (with "NLM" as ``Abstract.source``)
- AbstractText (``Section.name`` "Abstract" or the *NlmCategory*, ``Section.content`` with *Label* as ``Section.label``)
- AccessionNumber (``Database.accession``)
- ArticleId (``Identifier.value`` with *IdType* as ``Identifier.namesapce``; only available in online PubMed XML)
- ArticleTitle (``Citation.title``)
- CollectiveName (``Author.name``)
- CopyrightInformation (``Abstract.copyright``)
- DataBankName (``Database.name``)
- DateCompleted (``Medline.completed``)
- DateCreated (``Medline.created``)
- DateRevised (``Medline.revised``)
- DescriptorName (``Descriptor.name`` with *MajorTopicYN* as ``Descriptor.major``)
- ELocationID (``Identifier.value`` with *EIdType* as ``Identifier.namespace``)
- ForeName (``Author.forename``)
- Initials (``Author.initials``)
- Issue (``Medline.issue``)
- Keyword (``Keyword.value`` with *Owner* as ``Keyword.owner`` and *MajorTopicYN* as ``Keyword.major``)
- LastName (``Author.name``)
- MedlineCitation (with *Status* as ``Medline.status``)
- MedlineTA (``Medline.journal``)
- NameOfSubstance (``Chemical.name``)
- MedlinePgn (``Medline.pagination``)
- OtherAbstract (with *Type* as ``Abstract.source``)
- OtherID (``Identifier.value`` iff *Source* is "PMC" with ``Identifier.namespace`` as "pmc")
- PMID (``Medline.pmid``)
- PubDate (``Medline.pub_date``)
- PublicationType (``PublicationType.value``)
- QualifierName (``Qualifier.name`` with *MajorTopicYN* as ``Qualifier.major``)
- RegistryNumber (``Chemical.uid``)
- Suffix (``Author.suffix``)
- VernacularTitle (``Section.name`` "Vernacular", ``Section.content``)
- Volume (``Medline.issue``)

Version History
===============

2.1.2
  - fixed a bug where SQLite did not find the implicit FK->PK reference
    (thanks to Jason Hennessey for reporting the issue)
2.1.1
  - added SQLite temporary DB example URL to help output
  - refactored HTML output code
2.1.0
  - DB schema change from: ``records() -> sections(content)``
    to: ``citations(title) -> abstracts(copyright) -> sections(content)``
  - name change: the entity/table Medline/records is now called Citation/citations
  - title and copyright text is no longer stored in Section/sections
  - added a new Abstract/abstracts entity/table with a ``copyright`` attribute
    (formerly stored in ``sections.content`` with ``name`` = 'Copyright') 
  - added a new ``citations.title`` attribute
    (formerly stored in ``sections.content`` with ``name`` = 'Title') 
  - added a new ``source`` primary-key attribute to Section and Abstract
    (set to either 'NLM' for regular Abstract elements or to
    the value of the OtherAbstract Type attribute for other abstracts)
  - skipping "Abstract available from the publisher."-only abstracts
2.0.2
  - made the use of ``--pmid-lists`` for ``delete`` and ``write`` implicit
  - added instructions to bootstrap the tables in a PostgreSQL DB
  - minor improvements to this manual
  - fixed a bug when inserting/updating from MEDLINE XML files
2.0.1
  - fixed a bug that lead to skipping of abstracts
    (thanks to Chris Roeder for detecting the issue)
2.0.0
  - added Keywords and PublicationTypes
  - added MEDLINE publication date, volume, issue, and pagination support
  - added MEDLINE output format and made it the default
  - DB structure change: descriptors.major and qualifiers.major columns swapped
  - DB structure change: section.name is now an untyped varchar (OtherAbstract separation)
  - cleaned up the ORM test cases
1.1.1
  - code cleanup (PEP8, PyFlake)
  - fixed an issue where the parser would not leave the skipping state
1.1.0
  - ``--update parse`` now writes a file to use with ``--pmid-lists delete``
  - fixed a bug with CRUD manager
  - added a man page
1.0.2
  - fixes to make the PyPi version and ``pip install medic`` work
1.0.1
  - updates to the setup.py and README.rst files
1.0.0
  - initial release

Copyright and License
=====================

License: `GNU GPL v3`_\ .
Copyright 2012, 2013 Florian Leitner. All rights reserved.

.. _GNU GPL v3: http://www.gnu.org/licenses/gpl-3.0.html
.. _MEDLINE: http://www.nlm.nih.gov/bsd/mms/medlineelements.html
.. _PubMed: http://www.ncbi.nlm.nih.gov/pubmed
