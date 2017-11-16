====================
Date and time format
====================

Block Storage uses an ISO-8601 compliant date format for the display and
consumption of date time values.

**Example 2.3. DB service date and time format**

.. code::

    yyyy-MM-dd'T'HH:mm:ss.SSSZ

May 19th, 2011 at 8:07:08 AM, GMT-5 has the following format:

.. code::

    2011-05-19T08:07:08-05:00

|

The following table describes the date time format codes:

**Table 2.4. Date time format codes**

====  ===========
Code  Description
====  ===========

yyyy  Four digit year

MM    Two digit month

dd    Two digit day of month

T     Separator for date time

HH    Two digit hour of day (00-23)

mm    Two digit minutes of hour

ss    Two digit seconds of the minute

SSS   Three digit milliseconds of the second

Z     RFC-822 timezone
