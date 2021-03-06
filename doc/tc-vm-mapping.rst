.. |br| raw:: html

   <br />

Assigning VMs to thinclients: Database layout
=============================================


Introduction
------------

Thinclients use a postgres database to determine the virtual machine
that shall be displayed.

The database determines:

-  The VM shall be determined on a TC
-  When shall it be displayed (to display different VMs for different
   courses, ...)
-  If the VM shall `shut down <start-and-stop-management.html>`__ when the
   TC is shutting down

Accessing the database is documented
`here <virtesk-infrastructure-server.html#accessing-database>`__,
database setup is documented
`there <virtesk-infrastructure-server.html#setting-up-postgres-database>`__,
and switching virtual rooms is documented
`here <switching-virtual-rooms.html>`__

1:1-Mapping between TCs and VMs
-------------------------------

Any any given point in time, there shall be a 1:1-mapping between TCs
and VMs: A thinclient needs to uniquely determine the virtual machine
assigned to it, and on the other hand, one VM can only be displayed on
one TC at a time.

However, it makes also sense to talk about a **1:2-mapping** (or
1:3-mapping, ...). One VM is mapped to the thinclient. The other VMs can
be maintained (fresh rollout of virtual rooms, ...) without impacting
users. This is also interesting for having dedicated VMs for special
courses (exam VMs, VMs with special software, VMs with different
operating systems, ...).

This mapping is mandatory. Other VDI concepts, like pools of VMs,
on-the-fly creation of VMs, per-User-VMs, or user-choosen VMs are not
supported.

Thinclient perspective: SQL Query
---------------------------------

The following SQL Query is executed on a thinclient whenever virtesk-tc-connectspice wants to determine the virtual machine assigned to the thinclient, e.g. on thinclient startup, when re-connecting, and on thinclient shutdown, for shutting down the assined virtual machine if configured to do so.

::

    cur.execute("SELECT vm, thinclient, prio, id, shutdown_vm FROM thinclient_everything_view WHERE dhcp_hostname = ANY (%s) OR systemuuid = ANY (%s);", (dhcp_hostnames, sys_uuids))

There is only one database view that is accessed by thinclients:
``thinclient_everything_view``. This allows to adapt the database layout
to individual needs, the only mandatory aspect is the view
``thinclient_everything_view`` which must be compatible with the query
above.

Thinclient identification criteria
----------------------------------

It is necessary to define a mapping from thinclients to virtual
machines. For this mapping, some kind of thinclient identification is
necessary.

Virtesk-tc-connectspice supports two identifiers to uniquely identify
the thinclient it is running on:

**dhcp\_hostname**:

-  Determined by parsing the dhcp-leasefile on the thinclient. The
   leasefile is found by looking at the ``-lf`` Parameter of
   ``dhclient``.
-  Usefull to identify a workplace, independent of the thinclient device
   at that workplace. If the device needs replacement, the system
   administrator can simply adjust the dhcp configuration (new MAC
   adress). The system administrator does not need to adjust the
   thinclient-to-vm mapping, because the dhcp\_hostname will stay the
   same.

**systemuuid**:

-  Determined by parsing the output of ``dmidecode -t1`` on the
   thinclient.
-  Usefull for uniquely identifying the thinclient device. It doesn't
   matter where or in what network the thinclient is located. The
   thinclient is able to connect to its assigned virtual machine, as
   long as it is able to connect to the postgres database and the
   virtualization manager/hosts.

Sample database layout
----------------------

A sample database layout is provided in
``sample_config/database-layout.sql``. Instructions for loading the file
are provided
`here <virtesk-infrastructure-server.html#setting-up-postgres-database>`__.

Thinclient DNS Domain
---------------------

Depending on the DHCP server software used, **dhcp\_hostname** can be a
simple hostname or a fully qualified domain name.

Often it is handy to define the mapping from the
*thinclient*-database-key to the dhcp\_hostname automatically. This is
done in the view ``dhcphostname_to_thinclient_auto_mapping``. If FQDNs
are used, then you need to adapt the dns domain in the definition of the
view:

::

    (((timed_thinclient_to_vm_mapping.thinclient)::text || '.thinclients.yourdomain.site'::text))::character

You can also change the view later, for example using pgadmin3.

Important tables and views
--------------------------

Table timed\_thinclient\_to\_vm\_mapping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Defines the mapping between thinclients and VMs.

+---------------+-------------+------------+---------------------------------------------+
| start\_date   | end\_date   | priority   | comment                                     |
+===============+=============+============+=============================================+
| NULL          | NULL        | low        | useful for permanent mapping                |
+---------------+-------------+------------+---------------------------------------------+
| defined       | NULL        | medium     | switch to new VMs on start\_darte           |
+---------------+-------------+------------+---------------------------------------------+
| defined       | defined     | high       | override assignment for a period of time.   |
+---------------+-------------+------------+---------------------------------------------+

The column ``shutdown_vm`` determines if the VM shall be shut down upon
TC shutdown.

The column ``thinclient`` is any arbitrary name to identify the thinclient. It does not need to be equal to the local host name or the dhcp host name. However, it is usefull if this database key and the dhcp hostname are equal, see `Thinclient DNS Domain <#thinclient-dns-domain>`__ and `dhcphostname\_to\_thinclient\_auto\_mapping <#view-dhcphostname-to-thinclient-auto-mapping>`__

Table dhcphostname\_to\_thinclient\_mapping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Manual mapping from dhcp hostname to thinclient. Use this table to deal
with apart thinclients that do not adhere to any naming convention.

See also:
`dhcphostname\_to\_thinclient\_auto\_mapping <#view-dhcphostname-to-thinclient-auto-mapping>`__

Table systemuuid\_to\_thinclient\_mapping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Mapping from System UUID to thinclient database identifier:

::

    vdi=> select * from systemuuid_to_thinclient_mapping;
                  systemuuid              | thinclient  
    --------------------------------------+-------------
     C7E99E73-5ADB-48B3-8B03-30FDF9E4B238 | test01-tc04
    (1 row)

For every thinclient that you wanna identify by System UUID, one row
needs to be added.

View current\_thinclient\_to\_vm\_mapping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Helping View. Used to filter and prioritize the entries in
``timed_thinclient_to_vm_mapping`` based on ``start_date`` and
``end_date``.

View dhcphostname\_to\_thinclient\_auto\_mapping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Helping View. Automatically creates a mapping
``myTC.thinclients.yourdomain.site  ---> myTC`` for every myTC listed in
``timed_thinclient_to_vm_mapping``.

See also: `Thinclient DNS Domain <#thinclient-dns-domain>`__.

View sysinfo\_to\_thinclient\_mapping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Helping View. Union of dhcphostname\_to\_thinclient\_auto\_mapping,
dhcphostname\_to\_thinclient\_mapping, and
systemuuid\_to\_thinclient\_mapping, with defined priorities.

View thinclient\_everything\_view
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*"One view to rule them all, one view to find them,
one view to connect them all and using Spice to bind them."*

All information in the other tables and views is condensed in this one
big view, ready for use by `virtesk-tc-connectspice <virtesk-tc-connectspice.html>`__.

See also: `Thinclient perspective: SQL
Query <#thinclient-perspective-sql-query>`__
