GeoNode and GeoServer
=====================

GeoNode uses GeoServer to convert geographic data between various file formats,
as well as render styled map tiles.  While standard GeoServer components,
including its OGC-compliant web services and REST configuration API, provide
the bulk of the GeoServer functionality used in a GeoNode site, there are some
extensions to help GeoServer and the Django application interoperate more
better.

Geoprocessing with the GeoServer Process Extension
--------------------------------------------------

The Process extension to GeoServer helps to define custom geoprocessing
operations against data managed by GeoServer.  The operations are exposed via a GeoServer RESTlet endpoint.

Some processes are included with the Process extension.  Defining your own
processes is possible too, but first let's look at how to use the processes
that are already present.

There are two 'modes' of accessing these processes: *interactive* and *batch*.
The *interactive* mode is appropriate for fast-running processes and returns
the result of the operation as the HTTP response to the request that launches
the process.  The *batch* mode starts a thread on the server and supports
polling to find out information about progress until the process is completed.

Interactive Mode Processing
...........................

To use interactive-mode processing: 

  * send an HTTP POST request with the process parameters to
    /geoserver/rest/process/{process}/ .  The parameters must be formatted as a
    JSON document.

  * The result of the process will be returned, or an error will be reported
    via the HTTP status code (500 for a general error, 400 for a badly
    formatted request, etc.)  While the format of this response varies with the
    process, JSON should be preferred for structured, machine-accessible
    responses.

Batch Mode Processing
.....................

To use batch-mode processing:

  * send an HTTP POST with the process parameters to
    /geoserver/rest/process/{process}/launch .  The parameters must be
    formatted as a JSON document as described in the section on interactive
    mode.  The reponse will contain a document describing the process's
    progress, including an {id} string that identifies the process for future
    reference.

  * The process will initially be in a queue.  You can check on its progress by
    sending an HTTP GET request to /geoserver/rest/processes/{id}/status .
    When the status is DONE, you can send an HTTP GET request to
    /geoserver/rest/processes/{id}/result.

  * If you find a reason to cancel (for example, the user closes the JS widget
    that launched the process request), you can cancel by sending a DELETE
    request to /geoserver/rest/processes/{id} .  This will free up resources
    sooner, but in either case GeoServer will automatically delete cached
    process results after a set period, so you should ensure that you present
    the results to the user in a prompt fashion via some means or other.

Custom Processes
................

To create a custom process:

#. Write a Java class that extends the Process interface.  It can use GeoTools,
   and has access to the GeoServer catalog.  Let's call it
   ``com.example.custom.Process``.  Aditionally, subclass ``ProcessRestlet`` to
   create a Restlet that invokes your Process.

#. Include a Spring context XML file called ``applicationContext.xml`` that
   defines a bean using your process class, and a restlet mapping that attaches
   your process to a specific URL pattern.  An example would be:

   .. code-block:: xml

       <beans>
         <bean class="com.example.custom.ProcessRestlet" id="exampleRestlet"/>
         <bean class="org.geoserver.rest.RESTMapping" id="exampleMapping">
           <property name="routes">
             <map>
               <entry>
                 <key><value>/process/example</value></key>
                 <value>exampleRestlet</value>
               </entry>
             </map>
         </bean>
       </beans>

#. Make sure that the JAR containing your process is on the Java classpath when
   your application is running by including it in the ``WEB-INF/lib`` directory 
   of your GeoServer WAR file.

Authentication/Authorization
----------------------------

.. warning:: 

    This section describes features which have not yet been implemented.

GeoNode also provides an extension to GeoServer to have it respect GeoNode's
user database and permissions instead of its own independent system.  This
extension allows GeoServer to authenticate users by HTTP Basic auth (good for
general desktop GIS applications) or Django session cookies (good for users
accessing GeoServer from the Django site.)  The basic strategy is for GeoServer
to forward either type of credential to a Django web service which provides
GeoServer with up-to-date information about the user's permissions.  Consider a
request coming into some GeoServer service::

    GET /geoserver/ows?request=GetFeature&typename=top_secret:data

GeoServer will first inspect the request to identify whether it contains an
``Authorization:`` header or a cookie from the Django session system.
GeoServer then issues a request to Django::

    GET /user/permissions
   
Including the credentials it found on the initial request.  If the
``Authorization:`` header contains the username and password for a valid user,
or the session cookie corresponds to an active session for a logged-in user,
then Django responds with a document describing the permissions associated with
that user.  If the ``Authorization:`` header or cookie is found and determined
to be invalid, then Django sets a 401 status on the HTTP response.  Otherwise,
Django assumes an anonymous user and returns a document describing the
permissions associated with anonymous users.  The permissions document is a
JSON object that looks like this::

    {
        "rw": ["prefix:name", "prefix:name"],
        "ro": ["prefix:name", "prefix:name"]
    }

That is, a top-level object with two keys:

``rw``
    an array of prefixed layer names of layers which should be fully available
    (both read and write) to this user

``ro``
    an array of prefixed layer names of layers which should be displayed to this
    user, but which he/she should not be able to modify

All layers not named in this response will be presumed fully restricted, that
is, neither modifiable nor visible to the user in question.

Printing with the Mapfish Print Service
---------------------------------------

The GeoNode map composer can "print" maps to PDF documents using the `Mapfish
print service <http://www.mapfish.org/doc/print>`_.  The recommended way to run
this service is by using the printing extension to GeoServer (if you are using
the pre-built GeoNode package, this extension is already installed for you).
However, the print service includes restrictions on the servers that can
provide map tiles for printed maps.  These restrictions have a fairly strict
default, so you may want to loosen these constraints.

Adding servers by hostname
..........................

.. highlight:: yaml

The Mapfish printing module is configured through a `YAML <yaml>`_
configuration file, usually named :file:`print.yaml`.  If you are using the
GeoServer plugin to run the printing module, this configuration file can be
found at :file:`{GEOSERVER_DATA_DIR}/printing/config.yaml`.  The default
configuration should contain an entry like so::

    hosts:
      - !dnsMatch
        host: labs.metacarta.com
        port: 80
      - !dnsMatch
        host: terraservice.net
        port: 80

You can add host/port entries to this list to allow other servers.

.. seealso:: 
  
   The `Mapfish documentation
   <http://www.mapfish.org/doc/print/configuration.html>`_ on configuring the
   print module.

   The `GeoServer documentation
   <http://docs.geoserver.org/stable/en/user/community/printing/>`_ on
   configuring the print module.
