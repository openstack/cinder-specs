..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Generalized filtered listing on resources
=========================================

https://blueprints.launchpad.net/cinder/+spec/generalized-filtering-for-cinder-list-resource

Problem description
===================
We continue adding various filter options to the Cinder List API's.  Things
like "filter results by x=y".  This results in a good deal of code being
added for one-off preferences or user convenience.  It also means that we
continue to bump micro-versions for such additions and create more versions
of the API than we probably care to document, test and maintain.

What's annoying about this is that it's not even that we're adding things
to the server side, we're instead just popping things off of the params
in the request body based on what micro-version is specified.

The biggest area of churn for this sort of thing is in the client side of
things.

Use Cases
=========
Filters are convenient for operators doing things with large clouds.  It's
certainly a nice-to-have feature to be able to do things like just ask
Cinder for a list of all volumes with an `error` status, or that are
`bootable` or the latest, suggestion all those that are attached to host
xyz.

Proposed change
===============
In order to add these today, we add a specific arg to the clients, and then
we add specific arg parsing on the Cinder API server to interpret and parse
these args.  These parsed arguments are then just used to build a filter that
we pass in to the sql call to get_volumes (or get_<resource>).

Rather than continuing to add these for the various resources in an ad-hoc
manner, this spec proposes a generalized filtered-list mechanism.  Rather
than add a specific filter by name over and over, instead provide a single
`filter` argument that supplies Key Value pairs to be used to build the
db query filter.  There's likely some concern that people won't *know* what
valid filter keys are, for that we can also provide a `list-filters` command
that could be used in an introspective sort of way, querying the object model
for the valid filter items and returning it to the caller.

The `list-filters` component is derived from an Admin provided list of valid
and enabled filters for end users to access.  This obviously serves two
purposes:

   1. Advertise what filters are enabled/available to the end user
   2. Gives the Cloud Operator the ability to pick what filters are enabled

It's important to note that not all clouds will want to expose filters like
backend-host and other internals that shouldn't typically be leaked to the
end user.

What's interesting about this is from the API Servers perspective we don't
even really care about any of this.  We can already pass in filters via
the params in the request body and this just works.  If you're using curl
this just works already as is.  If you are using the client you'd just need
a way to expose it properly:
(https://gist.github.com/j-griffith/f455e51d3e597af96bfe7022974b5bd9)


Alternatives
------------
* We could certainly continue doing what we're doing now and just adding
  specific filter arguments to the clients and the API, but I don't think
  that's sustainable.

* We could provide a generalized "filters" (pick a better name) controller
  that could be called directly and then call the specified resource item
  with filters.  This would mean you would do things like:
  `cinder list-filters --resource volume`  to get a list of valid filter
  items for a volume (volumes should be the default).

  `cinder list-filters --resource snapshot` for snapshots, etc.

  But then have something like a `cinder [<resource>-]list --filters xyz=abc`
  which would go to a resource list controller and then call the resource
  controller appropriately.

  This might be a good way to reduce some duplicate code and churn even further
  but it may require just as much work and code as just having each resource
  have it's own "--filters" options to the list commands.

Data model impact
-----------------
None

REST API impact
---------------
Other than adding the ability to view the valid filter keys we don't
even need to change anything on the Cinder server side currently.

Security impact
---------------
None that I'm aware of

Notifications impact
--------------------


Other end user impact
---------------------
Life would be better

Performance Impact
------------------
None

Other deployer impact
---------------------
None

Developer impact
----------------
Life would be better


Implementation
==============

Assignee(s)
-----------
Primary assignee:
  john-griffith


Work Items
----------
* Add the filters arg to the client
* Add the ability to query the API server
  for valid filters for a resource

Dependencies
============

Testing
=======

Documentation Impact
====================
We will need to update documentation to describe the new capability,
although I expect anybody not using the client may already be exposing
this on their own.

References
==========
