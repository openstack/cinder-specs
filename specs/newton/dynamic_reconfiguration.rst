..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Dynamic Reconfiguration after Modfications of cinder.conf
============================================================

https://blueprints.launchpad.net/cinder/+spec/dynamic-reconfiguration/

Problem description
===================
As it stands, whenever changes are made to the Cinder configuration file all
cinder services need to be restarted in order to pick up the new
configuration setup. In a real world environment, a user does not want to
be dealing with services shutting down and spinning back up when some aspect
of configuration gets changed. In order to stay consistent with this,
constantly running services that can be dynamically updated as changes are
made provides a much more realistic view of how users interact with services.

Use Cases
=========
* Allows developers testing a new configuration option to change what the
option is set to and test without having to restart all the Cinder services.
* Allows developers to test new functionality implemented in a driver by
enabling and disabling configuration options in cinder.conf without
restarting Cinder services each time the option value is changed from
'disabled' to 'enabled'.
* Allows admins to manage ip addresses for various backends in cinder.conf
and have the connections dynamically update.

Proposed change
===============
SIGHUP Approach: The administrator will send the SIGHUP signal to be caught by
the Cinder processes.  The processes would then be instrumented to catch the
signal, drain in progress actions and reload the configuration file. This will
also take care of re-initializing the drivers. It still requires the
administrator to take an additional manual step to send the SIGHUP after
changing the configuration file. This approach requires deployers to update
their sysinit scripts to support reloading the processes. Overall, this is a
cleaner approach because everything is shut down, the memory is cleaned up,
and then the sysinit scripts will take care of reloading the processes.

Note: This is all done on a per host basis; in a multinode setup, the signal
will need to be sent to each node. Currently in Active/Active environments
this is a sizable issue that needs to be investigated.

Alternatives
------------
* Manual Approach: Continue to manually restart Cinder services when changes
are made to settings in the cinder.conf file. This is the current approach and
is what we are trying to get around.
* Reload & Restart Approach: First each process would need to finish the
actions that were ongoing. Next the db and RPC connections would need to
be dropped and then all caches and all driver caches would need to be
flushed before reloading. This approach adds a lot of complexity and lots of
possibilities for failure since each cache has to be flushed- things could get
missed or not flushed properly.
* File Watcher: Code will be added to the cinder processes to watch for
changes to the cinder.conf file. Once the processes see that changes have been
made, the process will drain and take the necessary actions to reconfigure
itself and then auto-restart. This capability could also be controlled by a
configuration option so that if the user didn't want dynamic reconfiguration
enabled, they could just disable it in the cinder.conf file. This approach is
dangerous because it wouldn't account for configuration options that are saved
into variables. To fix these cases, there would be a sizeable impact for
developers finding and replacing all instances of configuration variables and
in doing so a number of assumptions of deployment, configuration, tools, and
packaging would be broken.

Data model impact
-----------------
None

REST API impact
---------------
None

Security impact
---------------
In order to edit the cinder.conf file, the user must have root authority.
So, there is not an additional security impact.

Notifications impact
--------------------
There would need to be notifications of when the reload starts and ends.

Other end user impact
---------------------
None

Performance Impact
------------------
None

Other deployer impact
---------------------
Update sysinit scripts to support the new 'cinder-<service> reload'
command.

Developer impact
----------------
Test functionality with vendor's drivers to ensure correct behavior.


Implementation
==============

Assignee(s)
-----------
Primary assignee:
  kendall-nelson

Secondary assignee:
  jacob-gregor

Work Items
----------
*Implement handling of SIGHUP signal
**Ensure caches are cleared
**Dependent vars are updated
**Connections are cleanly dropped
*Write Unit tests
*Testing on a variety of drivers
*Update devref to describe SIGHUP/reload process

Dependencies
============

Testing
=======

Documentation Impact
====================
We will need to update documentation to describe the new capability.

References
==========
Weekly Meeting Logs[1]

[1]http://eavesdrop.openstack.org/meetings/cinder/2016/cinder.2016-03-16-16.01.log.txt
