HOWTO: Third Party Testing
==========================

Overview
--------

Gerrit has an event stream which can be subscribed to, using this it is possible
to test commits against testing systems beyond those supplied by OpenStack's
Jenkins setup.  It is also possible for these systems to feed information back
into Gerrit and soon they will also be able to leave votes on Gerrit review
requests.

An example of one such system is `Smokestack <http://smokestack.openstack.org/>`_.
Smokestack reads the Gerrit event stream and runs it's own tests on the commits.
If one of the tests fails it will publish information and links to the failure
on the review in Gerrit.

Reading the Event Stream
------------------------

It is possible to use ssh to connect to ``review.openstack.org`` on port 29418
with your ssh key if you are signed up as an OpenStack developer on Launchpad.

Documentation on how to use the events stream can be found in `Gerrit's stream event documentation page <http://gerrit-documentation.googlecode.com/svn/Documentation/2.3/cmd-stream-events.html>`_.

Posting Result To Gerrit
------------------------

In the near future we will make is possible for external testing systems to give
non-gating votes to Gerrit.  In the mean time they can post their results as
comments.  We do, however, ask that the comments contain public links to the
failure so that the developer can see what caused the failure.

An example of how to post this is as follows:

.. code-block:: bash

   $ ssh -p 29418 review.example.com gerrit review -m '"Test failed on MegaTestSystem <http://megatestsystem.org/tests/1234>"' c0ff33

In this example ``c0ff33`` is the commit ID for the review.

Further documentation on the `review` command in Gerrit can be found in the `Gerrit review documentation page <http://gerrit-documentation.googlecode.com/svn/Documentation/2.3/cmd-review.html>`_.

We do suggest cautious testing of these systems and have a development Gerrit
setup to test on if required.  In SmokeStack's case all failures are manually
reviewed before getting pushed to OpenStack, whilst this may no scale it is
advisable during initial testing of the setup.

Feel free to contact the CI team to arrange setting up a dedicated user so your
system can post reviews up using a system name rather than your user name. 
