.. _week8_project:

==================================================
Week 8 - Project Week: Zika Mission Control Part 4
==================================================

Overview
========

`Read Mission Briefing 4 <../../_static/images/zika_mission_briefing_4.pdf>`_


You will be adding GeoServer to your local and deployed application. This will require several steps, including setting up a GeoServer instance, connecting it to a PostGIS store, and creating a new layer from existing data.

After you have this new system fully set up, you'll be able to add additional layers in GeoServer and show them using OpenLayers.

Requirements
============

1. Use GeoServer to display the report features.
2. Incorporate additional layers using external data (e.g. elevation, temperature, natural earth).
3. Deploy your application to AWS, including a EC2 instance running GeoServer.

New Stuff
=========

There are a few differences in this project compared to ``week6-starter``.  You may want to copy over certain changes instead of merging.

* `Week 8 Starter <https://gitlab.com/LaunchCodeTraining/zika-cdc-dashboard/tree/week8-starter>`_
* `week8-starer / week6-starter diff <https://gitlab.com/LaunchCodeTraining/zika-cdc-dashboard/compare/week6-starter...week8-starter>`_

Specific Changes
----------------

1. Using three docker containers locally

   * GeoServer container
   * PostGIS container
   * Elasticsearch container

2. Partially normalized report data

   * Relationship between ``Report`` and ``Location`` via ``Report.state``
   * Setup by Hibernate relationship
   * Populated by making a ``POST`` request to endpoint ``/report/assignStates``

3. ``cloud/geoserver_userdata.sh`` is provided to create a GeoServer instance in AWS

4. Running Tomcat on a different port locally by adding ``server.port={SERVER_PORT}`` to ``application.properties``
   
   * Then set the environment variable ``SERVER_PORT=9090``

Setup Locally
=============

Before getting started, **be sure that your Boundless virtual machine is not running**. We will be using the same port as the VM with one of our Docker containers, so if it is running there will be conflicts. It is not enough to pause the VM; you must shut it down completely or select *Save State*.

Docker Commands
---------------

* ``docker ps`` see list of **running** containers
* ``docker ps -a`` see lisf of all containers, including ones that failed or were stopped
* ``docker start <container-name or id>`` starts the container
* ``docker stop <container-name or id>`` stops the container
* ``docker restart <container-name or id>`` restarts the container
* ``docker rm <container-name or id>`` removes the container
* ``docker images`` shows list of images that you have downloaded. containers are created from images
* ``docker image rm <image-name or id>`` removes an image
* For more info and more commands please see `the Docker CLI docs <https://docs.docker.com/engine/reference/commandline/docker/>`_

.. warning::

  If your containers start crashing with exit code 137, it's because they are out of memory. You need to incrase the memory given to Docker by going to Peferences, Advanced. See screen shot below.

.. figure:: /_static/images/docker-memory.png

   Docker Memory Configuration

Create PostGIS Container
------------------------

First create a file ``env.list`` in the root of your ``zika-cdc-dashboard`` project folder, with the same contents as `our envlist file <https://gist.github.com/chrisbay/d74442a8e8707111472a742832d76796>`_.

Next run this command to create a postgis container referencing ``env.list`` from above. We have to use a docker instance of Postgis so that our GeoServer docker instance and our local web application can both connect to Postgis. ::

  $ docker run --name "postgis" -p 5432:5432 -d -t --env-file ./env.list kartoza/postgis:9.4-2.1

.. warning::

  In ``env.list`` you'll see that the ``POSTGRES_DBNAME`` environment variable is set to ``zika``. This variable is supposed to set the name of our PostGIS-enabled database within the container to be ``zika``. However, a bug in the Dockerfile for this image ignores the name, creating a database named ``gis``.

Verify that the container is running by running ``docker ps``.

Create GeoServer Container
--------------------------

We are going to link the PostGIS and GeoServer containers. That tells docker that these containers need to be able to communicate.::

  $ docker run --name "geoserver" --link postgis:postgis -p 8080:8080 -d -t kartoza/geoserver

.. warning::

  If the ``postgis`` docker image is not running when starting the geoserver, the link will fail.

When its container is running, you can access this GeoServer instance the same way in which you previously accessed GeoServer locally when running the Boundless virtual machine. It will be running on port 8080 (try ``http://localhost:8080/geoserver``) with credientials **admin / geoserver**.


Create Elasticsearch Container
------------------------------

::

  $ docker run --name "es" -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node"  -e "xpack.security.enabled=false" docker.elastic.co/elasticsearch/elasticsearch:5.6.0

.. warning::

  If Docker has no more than 2G of memory allocated for container use, you may have issues with the ``elasticsearch`` container crashing due to lack of memory. If this happens, increase memorgy to at least 3G by going to *Docker > Preferences > Advanced*.

Enable CORS in GeoServer
------------------------

You'll be making requests to the GeoServer container from a port other than the one on which GeoServer is running, which means CORS will come into play. Let's enable cross-origin requests within GeoServer.

.. tip:: You may want to wait until you actually see a `CORS <https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS>`_ error in your browser's JavaScript console before performing these steps.

Open a shell within the Docker container and install a text editor (you can also install ``nano`` instead of ``vim`` if you want):::

  $ docker exec -it geoserver bash
  root@2992f761f41e:/usr/local/tomcat# apt-get update
  root@2992f761f41e:/usr/local/tomcat# apt-get install vim

Open the GeoServer ``web.xml`` for editing:::

  root@2992f761f41e:# vi /usr/local/tomcat/conf/web.xml

Add the following XML just within the opening ``<web-app>`` tag:

.. code-block:: xml

  <filter>
    <filter-name>CorsFilter</filter-name>
    <filter-class>org.apache.catalina.filters.CorsFilter</filter-class>
  </filter>
  <filter-mapping>
    <filter-name>CorsFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>

Save the file and exit. Then exit the docker container shell. ::

  root@2992f761f41e:# exit

Stop and start the ``geoserver`` container: ::

  $ docker stop geoserver
  $ docker start geoserver

Now ``XHR``requests from your local zika app running on ``http://localhost:9090`` will be accepted by our GeoServer instance. If you don't set that up, you will see ``CORS`` errors in the js console.

Populate Container PostGIS Database
-----------------------------------

We need to load the report and location data into the ``postgis`` docker container.  We will copy over the ``.csv`` files to the container and execute psql copy commands.

* First, let's change the paths referenced in the ``/src/main/resources/data.sql`` file to be ``'/tmp/locations.csv'`` and ``'/tmp/all_reports.csv'``
* Then copy the files to the ``postgis`` contianer:

::

  $ docker cp locations.csv postgis:/tmp
  $ docker cp all_reports.csv postgis:/tmp


Verify that the files made it:::

  $ docker exec -it postgis ls -l /tmp

Remember that ``data.sql`` makes use of the ``unaccent`` function, which is part of the ``unaccent`` Postgres extension. While our Docker image came with the PostGIS extension installed, the ``unaccent`` extention is **not** present. Let's fix that.

Also ``data.sql`` will not actually be executed by Spring Data. If you rename it to ``import.sql`` and edit property ``spring.jpa.hibernate.ddl-auto`` in ``application.properties``. If ``spring.jpa.hibernate.ddl-auto`` is either ``create`` or ``create-drop``, then ``import.sql`` will run. After you database has been initialized you can change the value to ``validate``. More here on `Spring Data - Database Initialization <https://docs.spring.io/spring-boot/docs/current/reference/html/howto-database-initialization.html>`_

.. warning::

  Stop all instances of Postgres on your local machine. Stop the Postgress App in the top bar and stop the service being managed by ``brew``. The only Postgres we want running is the one inside of the Docker container. If you get an error below that the ``gis`` database doesn't exist, then you are connected to the wrong Postgres instance.

Fire up ``psql``, note the password for ``zika_app_user`` is ``somethingsensible``: ::

  $ psql -h localhost -p 5432 -U zika_app_user -d gis

And then install the extension: ::

  # create extension unaccent;

Exit ``psql``.

Change Tomcat Port
------------------

Now, configure your ``zika-cdc-dashboard`` app so it can connect to the PostGIS datbase. This requires editing the environment variables in the ``Application`` run configuration. The only edit you should need to make is to set the ``APP_DB_NAME`` to ``gis`` (see the Warning above).

Before we can run our Spring app, we need to configure it to run on a port other than 8080. Recall that we set up the GeoServer container to bind to port 8080 on our localhost, so the default for Spring (which is also 8080) will not work. We can easily adjust the port that Spring will run on by adding ``server.port=9090`` to ``application.properties``.

.. note::

  You may also need to change the port referenced in ``script.js``. ``url: 'http://localhost:9090/api/es/report/?date=2016-03-05'``. Another solution for this is to use a relative path ``url: '/api/es/report/?date=2016-03-05'``

Start up your Spring app. Verify that the app started up cleanly, and that the ``locations`` and ``reports`` databases were built and populated properly.

.. tip::

  If your ``locations`` and ``reports`` databases aren't being populated, you can populat them manually by copying the ``data.sql`` file in ``src/main/resources/`` to the ``postgis`` container (see above) and running:
  ``$ docker exec -it postgis psql -h localhost -U zika_app_user -d gis -a -f /tmp/data.sql``

Add foreigh keys to reports
---------------------------

We want to set up explicit relationships between reports and locations in the database. To do this, we've created an endpoint that for each ``Report`` object, will look for a corresponding ``Location`` object and create a reference/foreign key relationship.

Start up your Spring app and hit the endpoint from the command line:::
  
  $ curl -XPOST http://localhost:9090/api/report/assignStates


This will take a few minutes to run. When the request is complete, all ``Report`` objects for which there is a corresponding ``Location`` will have the relationship stored as a foreign key in the ``report.state_id`` column.

Database and Layer Setup
------------------------

These views will allow us to create a layer in GeoServer that will allow us to query location geometries with case totals by date.

Using either ``psql`` or a Posgrest graphical client to connect to the PostGIS database running inside the Docker container (recall that it is accessible on port 5432 from your local environment). Create two views:

.. code-block:: sql

  CREATE view cases_by_state_and_date AS
    SELECT state_id,report_date,sum(value) AS cases FROM report
    GROUP BY state_id,report_date;


.. code-block:: sql

  CREATE view states_with_cases_by_date AS
    SELECT * FROM location INNER JOIN cases_by_state_and_date ON location.id=cases_by_state_and_date.state_id;

Create Data Store and Layers in GeoServer
-----------------------------------------

* Create a workspace in GeoServer (we recommend ``lc/https://launchcode.org``)
* Create a PostGIS data store

  * Use ``gis`` as the database name and ``postgis`` as the hostname

* Create a new layer from the ``states_with_cases_by_date`` table

  * Make sure Native and Declared SRS are set to **EPSG:4326**
  * For Native Bounding Box, click on **Compute from data**
  * For Lat/Lon Bounding Box, click on **Compute from native bounds**

Updating OpenLayers Code
------------------------

Following the `OpenLayers example <https://openlayers.org/en/latest/examples/vector-wfs-getfeature.html>`_ for querying ``GetFeature``, update your OpenLayers code to query GeoServer to get locations with report totals by date. You'll need to use the ``ol.format.filter.equalTo`` filter.

.. warning::

  For the geometries in your layer to be rendered properly on the map, the spatial reference systems (SRS) must match. You can control the SRS that is used to generate the returned features using the ``srsName`` parameter when create the request in OpenLayers.

Update Report POST Endpoint
---------------------------

There is a controller in ``ReportController`` called ``saveNewReport`` that saves creates a new report object and saves it in both data stores (Postgresql and Elasticsearch). Update this method so that it looks up and assigns the corresponding ``Location`` object (if one exists) for the given report.


Deploying to AWS
================

To deploy GeoServer on AWS you will be using a ``t2.small`` CentOS machine.

Paste the contents of shell script `geoserver_userdata.sh <https://gitlab.com/LaunchCodeTraining/zika-cdc-dashboard/blob/week8-starter/cloud/geoserver_userdata.sh>`_ into the "Advanced Details" details section of "Configure Instance" to create the instace.  The script installs Apache Tomcat, downloads the Boundless Suite WAR, and deploys the geoserver WAR the Apache Tomcat server.  The deployed geoserver can be reached on ``http://{your IP}:8080/geoserver``.

.. tip::

  Remember the default username for Geoserver is ``admin`` and the default password is ``geoserver``.

Bonus Mission
-------------

When you complete all of these instructions, check out the `ElasticGeo Plugin <https://github.com/ngageoint/elasticgeo>`_. It is an Elasticsearch plugin that allows you to integrate Elasticsearch into Geoserver. The great thing is that you can do Elasticsearch queries directly through Geoserver via WFS calls. Here are the setup instructions and instructions on how to make the calls: `ElasticGeo Instructions <https://github.com/ngageoint/elasticgeo/blob/master/gs-web-elasticsearch/doc/index.rst>`_
