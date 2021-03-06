:orphan:

.. _postgres-walkthrough:

=======================
Walkthrough: PostgreSQL
=======================

Follow along with the instructor as we work with PostgreSQL

Install Postgres App
--------------------

- Go `https://postgresapp.com/ <https://postgresapp.com/>`_
- Download and open the file
- Note: this is specifcally for Mac OS

Add postgres to your $PATH
**************************

* Open a terminal
* Open your ``.bash_profile`` in an editor like nano::

    $ nano ~/.bash_profile

* Paste in this text to your ``~/.bash_profile``::

    # add postgres and it's related commands to path
    export PATH="/Applications/Postgres.app/Contents/Versions/latest/bin:$PATH"

* Save your ``~/.bash_profile`` file
* Open a new terminal and run ``$ postgres --version``
* You will use the ``postgres`` command on the terminal later in the class


Let's Run Some Postgres Commands
--------------------------------

* Open Postgres.app on your mac
* In the postgres terminal we will run the following commands
* ``Create database sports;``
* ``\l`` shows list of databases
* ``create schema baseball;``
* ``Create table baseball.teams (name varchar(50), description varchar(100));``
* ``\d baseball.teams`` - show info about table

Now For Some SQL Queries
------------------------
* Insert
* Select
* delete
* Alter table

  * ALTER TABLE baseball.teams ADD COLUMN id integer PRIMARY KEY;

* Constraints

  * Not null
  * Primary key
  * Foreign key

* Create a Sequence

  * CREATE SEQUENCE baseball.teams_id_seq START 10;
  * DEFAULT nextval('baseball.teams_id_seq');

* Indexes

  * Created automatically with Primary Key and Unique constraints
  * You can also add an index manually
