############
Introduction
############

Project goals
=============

The aim of this project is to implement a Tango server that manages data
IO for synchrotron (and maybe neutron) beamlines. The server should
satisfy the following requirements

* remove responsibility for data IO
  from the beamline control client
* provide a simple configuration mechanism via NXDL
* read data from the following sources without client interaction
* SQL databases (MySQL, Postgres, Oracle, DB2, ...)
* other TANGO servers
* JSON records (important for the interaction
  with the client and SARDANA)
* the first implementation of the server will be written in Python
* the communication model of the first implementation will be strictly synchronous
  (future version most probably will support other communication models too)
* the control client software has full control over the behavior of the server via
  TANGO commands
* only low data-rate sources will be handled directly by the server.
  High data-rate sources will write their data independently
  and additional software will add this data to the Nexus file produced by
  the server once the experiment is finished.

The server should make it easy to implement control clients which write
Nexus files as the entire Nexus logic is kept in the server. Clients
only produce NXDL configurations or use third party tools for this job.
The first Python implementation of this server will serve as a proof of
concept.

The importance of the client
============================

.. image:: files/system_layout.png

All operations carried out on a beamline are orchestrated by the control
client (CC), a software operated by the beamline-scientist and/or a
user. Although the term client suggests that it is only a minor
component aside from all the hardware control servers, databases, and
whatever software is running on a beamline it is responsible for all the
other components and tells them what to do at which point in time. In
terms of an orchestra the CC is the director which tells each group of
instruments or individual artist what to do at a certain point in time.

It is important to understand the role of the CC in the entire software
system on a beamline as it determines who is responsible for certain
operations. The CC might be a simple single script running on the
control PC which can is configured by the user before start or it might
be a whole application of its own like SPEC or ONLINE. Historically it
is the job of the CC to write the data recorded during the experiment
(this is true at least for low rate data-sources). However, with the
appearance of complex data formats like Nexus the IO code becomes more
complex.

