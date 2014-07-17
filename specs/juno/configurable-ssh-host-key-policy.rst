..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Configurable SSH Host Key Policies for Cinder
=============================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/configurable-ssh-host-key-policy

To address concerns of weak ssh security in Cinder, the way that
ssh host keys are handled by Cinder should be configurable, allowing
system administrators to choose how secure they wish their SSH connections
to be.

Problem description
===================

During Icehouse testing and development it was discovered that
SSHPool in cinder/utils.py was using the paramiko.AutoAddPolicy()
option.  This meant that changes to the SSH key on the storage back-end
would just be accepted with only a warning being printed.  This leaves
a security weakness as it is possible for a Man-In-the-Middle (MITM)
attack to happen between the Cinder Volume host and the back-end storage.
If a MITM attack were to happen, the users data could be compromised.
In a worst case scenario, users could be tricked into attaching or even
booting spoofed volumes containing malicious code.


Proposed change
===============

The spec and associated blueprint propose making the way that SSH
host keys are handled configurable, allowing system administrators
to make a conscious decision about the level of security they need
on their system.

The solution would require two new configuration items as well as
a change to the current default behavior.  First, there would need
to a 'strict_ssh_host_key_policy' configuration option with possible
settings of 'false' (default) or 'true'.  When this option is set to
'false' it will automatically accept the host key on the first connection
and then will throw an exception if the host key changes in the future.
This is where the default behavior changes from the current functionality.

In the case that 'strict_ssh_host_key_policy' is set to 'true' then a
second option 'ssh_host_keys_file' must be configured.  When the strict
configuration is used it is assumed that the administrator is going to
have pre-configured ssh host keys and any deviation from those expected
keys will be handled with an exception.

Alternatives
------------

Obviously, we could keep the current functionality but this leaves
a security weakness.  We could also just require that an ssh host key
file be specified, but I feel this is undesirable as it is yet another
configuration option that users must deal with.  The last option would
be to make the functionality fully configurable, making it possible to
keep the current functionality, where changes in the host key only cause
a warning to be reported, in addition to the new configuration options
listed above, but I don't feel leaving the security weakness in place
is the right approach if we are making this change.  We are actually
bringing Cinder's handling of ssh keys in line with what users are
used to from the command line, which seems like the most sensible
approach.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

This change improves security by handling ssh keys in a manner that
will help to avoid the possibility of Man-in-the-Middle attacks.

Notifications impact
--------------------

None

Other end user impact
---------------------

As mentioned above, this change will make it so that, by default, the
user will see a failure connecting to their back-end storage system
in the case the ssh keys on that system change.  The user will have
the ability to set the level of security they wish to enforce on their
volume server using the 'strict_ssh_host_key_policy' and also be able
to specify the host keys they wish to use with the 'ssh_host_keys_file'
option.

Performance Impact
------------------

None

Other deployer impact
---------------------

Deployers that wish to have a more secure ssh implementation will need to
set the 'strict_ssh_host_key_policy' and 'ssh_host_keys_file'
configuration options.

Developer impact
----------------

Drivers that currently use ssh to connect to their back-end storage
systems will need to ensure that their drivers are using this approach
for securing their ssh keys.  Future driver developers will also need
to be consistent with this improved security model.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jsbryant (jsbrant@us.ibm.com or jungleboyj on IRC)

Work Items
----------

Initial changes to cinder/utils.py to enable this new functionality.
Update existing drivers to properly utilize the functionality.


Dependencies
============

None


Testing
=======

Beyond unit tests, I don't know that there is much more testing that can be
done unless there is a way, in Tempest, to simulate bad ssh keys being
returned.


Documentation Impact
====================

Documentation will need to be updated for the configuration options and
explanation of how the functionality is designed to work.


References
==========
Original bug which started this discussion:  https://bugs.launchpad.net/cinder/+bug/1320056
Initial fix for utils.py in the community: https://review.openstack.org/#/c/94165/
Weekly Cinder Meeting discussion on this topic:  http://eavesdrop.openstack.org/meetings/cinder/2014/cinder.2014-05-28-16.00.log.html#l-104

