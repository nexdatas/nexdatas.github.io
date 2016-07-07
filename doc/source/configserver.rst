Server clients
##############

Client code of NeXus Data Writer
================================

In order to use Nexus Data Server one has to write a client code. Some
simple client codes are in the nexdatas repository. In this section we
add some comments related to the client code.

To use the Tango Server we must import the ``PyTango`` module and create
``DeviceProxy`` for the server.

::

    import PyTango

    device = "p09/tdw/r228"
    dpx = PyTango.DeviceProxy(device)
    dpx.set_timeout_millis(10000)

    dpx.Init()

Here ``device`` corresponds to a name of our Nexus Data Server.
``Init()`` method resets the state of the server.

::


    dpx.FileName = "test.h5"
    dpx.OpenFile()

We set the name of the output HDF5 file and open it.

Now we are ready to pass the XML settings describing a structure of the
output file as well as defining a way of data storing. Examples of the
XMLSettings can be found in the XMLExamples directory.

::


    xml = open("test.xml", 'r').read()
    dpx.XMLSettings = xml

    dpx.JSONRecord = '{"data": {"parameterA":0.2},
                          "decoders":{"DESY2D":"desydecoders.desy2Ddec.desy2d"},
                          "datasources":{"MCLIENT":"sources.DataSources.LocalClientSource"}}'

    dpx.OpenEntry()

We read our XML settings settings from a file and pass them to the
server via the ``XMLSettings`` attribute. Then we open an entry group
related to the XML configuration. Optionally, we can also set
``JSONRecord``, i.e. an attribute which contains a global JSON string
with data needed to store during opening the entry and also other stages
of recording. If external decoder for ``DevEncoded`` data is need one
can registred it passing its packages and class names in ``JSONRecord``
e.g. "desy2d" class of "DESY2D" label in "desydecoders.desy2Ddec"
package. Similarly making use of "datasources" records of the JSON
string one can registred additional datasources. The ``OpenEntry``
method stores data defined in the XML string with ``strategy=INIT``. The
``JSONRecord`` attribute can be changed during recording our data.

After finalization of the configuration process we can start recording
the main experiment data in a ``STEP`` mode.

::


    dpx.Record('{"data": {"p09/counter/exp.01":0.1, "p09/counter/exp.02":1.1}}')

Every time we call the ``Record`` method all nexus fields defined with
``strategy=STEP`` are extended by one record unit and the assigned to
them data is stored. As the method argument we pass a local JSON string
with the client data. To record the client data one can also use the
global ``JSONRecord`` string. Contrary to the global JSON string the
local one is only valid during one record step.

::


    dpx.Record('{"data": {"emittance_x": 0.1},  "triggers":["trigger1", "trigger2"]  }')

If you denote in your XML configuration string some fields by additional
trigger attributes you may ask the server to store your data only in
specific record steps. This can be helpful if you want to store your
data in asynchronous mode. To this end you define in the local JSON
string a list of triggers which are used in the current record step.

::


    dpx.JSONRecord = '{"data": {"parameterB":0.3}}'
    dpx.CloseEntry()

After scanning experiment data in 'STEP' mode we close the entry. To
this end we call the ``CloseEntry`` method which also stores data
defined with ``strategy=FINAL``.

Since our HDF5 file can contains many entries we can again open the
entry and repeat our record procedure. If we define more than one entry
in one XML setting string the defined entries are recorded parallel with
the same steps.

Finally, we can close our output file by

::


    dpx.CloseFile()

Additionally, one can use asynchronous versions of ``OpenEntry``,
``Record``, ``CloseEntry``, i.e. ``OpenEntryAsynch``, ``RecordAsynch``,
``CloseEntryAsynch``. In this case data is stored in a background thread
and during this writing Tango Data Server has a RUNNING state.

In order to build the XML configurations in the easy way the authors of
the server provide for this purpose a specialized GUI tool, Component
Designer. The attached to the server XML examples was created by
``XMLFile`` class defined in ``XMLCreator/simpleXML.py``.


Client code of NeXus Configuration Server
=========================================

The **Configuration Server** is a Tango process dedicated to store
NXDL-like configuration needed for running the Data Writer Tango server.
The server uses a MYSQL database as storage system. The ``ndts.sql``
script from the repository can be used to create the required DB tables.

In the Configuration Server the configuration is memorized in two types
of elements: datasources and components.

**DataSources** describe access to input data, i.e. to specific TANGO
devices, other databases as well as to client data.

**Components** specify Nexus tree with positions of datasets for
particular pieces of hardware and writing strategy for their
corresponding data. \* They can include datasources directly as well as
links to datasources defined in the server. In this last case the syntax
``$datasources.<ds_name>`` is used. \* They can also hold links to other
components which describe their dependencies. The
``$components.<comp_name>`` syntax is used for this purpouse. \*
Finally, the components can contain variables. Variables are defined in
XML code by ``$var.<var_name>#'default_value'`` and can be provided to
the Configuration Server via a JSON string. Not defined default value
has a value of an empty string.

All elements of the configuration can be created by a dedicated GUI tool
- **ComponentDesigner**. This tool connects to a Configuration Server,
fetchs the elements from the database, shows them, and allows
modification or creation of new ones. The changes can be also stored in
the database.

In order to create the configuration for an specific nexus file the
Configuration Server merges all required and dependent components,
together with their datasources and provided values of the variables. As
a result it returns a single XML string. This XML string can be passed
directly into a dedicated **Tango Data Writer** attribute.


In this section we present an example how to communicate with the
Configuration Server using PyTango.

::

    import PyTango

    cnfServer = PyTango.DeviceProxy("p00/xmlconfigserver/exp.01")

    cnfServer.JSONSettings = \
        '{"host":"localhost","db":"ndts_p02","read_default_file":"/etc/my.cnf","use_unicode":true}'

    # opens DB connection
    cnfServer.Open()

After creating the server proxy we can set the configuration of the
MYSQL DB connection using the attribute ``JSONSettings``, this attribute
is memorized so ,after setting it the first time, it has to be set again
only when the configuration of the DB connection is changed. The command
Open opens the connection to the DB specified by ``JSONSettings``.

::

    # stores default component
    cpxml = open("default.xml", 'r').read()
    cnfServer.XMLString = cpxml
    cnfServer.StoreComponent('default')

    # stores slit1 component in DB
    cpxml = open("slit1.xml", 'r').read()
    cnfServer.XMLString = cpxml
    cnfServer.StoreComponent('slit1')

    # stores slit2 component in DB
    cpxml = open("slit2.xml", 'r').read()
    cnfServer.XMLString = cpxml
    cnfServer.StoreComponent('slit2')

    # stores slit3 component in DB
    cpxml = open("slit3.xml", 'r').read()
    cnfServer.XMLString = cpxml
    cnfServer.StoreComponent('slit3')

    # stores pilatus300k component in DB
    cpxml = open("pilatus.xml", 'r').read()
    cnfServer.XMLString = cpxml
    cnfServer.StoreComponent('pilatus300k')


    # stores motor01 datasource in DB
    dsxml = open("motor.ds.xml", 'r').read()
    cnfServer.XMLString = dsxml
    cnfServer.StoreDataSource('motor01')

    # stores motor02 datasource in DB
    dsxml = open("motor.ds.xml", 'r').read()
    cnfServer.XMLString = dsxml
    cnfServer.StoreDataSource('motor02')


    # removes slit3 component from DB
    cnfServer.DeleteComponent('slit3')

    # removes motor02 datasource from DB
    cnfServer.DeleteDataSource('motor02')

The examples above show an alternative to the ComponentDesigner for
creating or deleting components and datasources using python scripts.

::

    # provides names of available components
    cmpNameList = cnfServer.AvailableComponents()
    # provides names of available datasources
    dsNameList = cnfServer.AvailableDataSources()

The commands above get information about names of available components
and datasources in the Configuration Server.

::

    # provides a list of required components
    cmpList = cnfServer.Components(cmpNameList)
    # provides a list of required Datasources
    dsList = cnfServer.DataSources(dsNameList)

The commands above provide the XML string corresponding to the stored
elements given as argument.

::

    # provides a list of DataSources from given components
    dsList = cnfServer.ComponentDataSources('pilatus300k')
    dsList = cnfServer.ComponentsDataSources(['pilatus300k', 'slit1'])

Using the above commands one get the datasources related to the
requested components.

::

    # provides a dependent components
    cpList = cnfServer.DependentComponents(['pilatus300k', 'slit3'])

The list of dependent components can be requested using the command
above.

::

    # provides a list of Variables from a given components
    varList = cnfServer.ComponentVariables('pilatus300k')
    varList = cnfServer.ComponentsVariables(['pilatus300k', 'slit3'])

Commands for getting the list of variables related to particular
components are also provided.

::

    # sets values of variables
    cnfServer.Variables = '{"entry_id":"123","beamtime_id":"123453535453"}'

The values of the variables can be passed to the Configuration Server
via a JSON string.

::

    # sets given component as mandatory for the final configuration
    cnfServer.SetMandatoryComponents(['default','slit1'])
    # un-sets given component as mandatory for the final configuration
    cnfServer.UnsetMandatoryComponents(['slit1'])

    # provides names of mandatory components
    man = cnfServer.MandatoryComponents()

Some of the components can be set as mandatory in the final
configuration, the Configuration Server provides the above commands for
set/unset or display them.

::

    # provides the current configuration version
    version = cnfServer.Version

The revision number associated to a given configuration is provided
together with Configuration Server version in the attribute ``Version``.

::

    # defines datasources set to fetched in the strategy STEP mode
    cnfServer.STEPDataSources = ["exp_mot01", "exp_mot05"]
    # creates the final configuration from slit2 and pilatus300k 
    # as well as all mandatory components
    cnfServer.CreateConfiguration(['slit2', 'pilatus300k'])
    # XML string ready to use by Tango Data Server
    finalXML = cnfServer.XMLString 

In order to create our final configuration we execute
``CreateConfiguration`` command with a list of names of required
components. The command merges these components with mandatory ones and
provides the resulting NXDL-like configuration in the ``XMLString``
attribute. Before calling ``CreateConfiguration`` the user may define
which INIT/FINAL component datasources should be fetched and written in
the strategy STEP mode.

::

    # merges given components
    mergedComp = cnfServer.Merge(['slit2', 'pilatus300k'])

Similarly,the ``Merge`` command provides configuration by unresolved
links to datasoures and with non-assigned variable values.

::

    # closes connection to DB
    cnfServer.close()

Command ``close`` terminates our connection to the DB server.
