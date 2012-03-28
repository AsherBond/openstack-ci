Puppet Modules
==============

Overview
--------

Much of the OpenStack project infrastructure is deployed and managed using
puppet.
The OpenStack CI team manage a number of custom puppet modules outlined in this
document.

Doc Server
----------

The doc_server module configures nginx [4]_ to serve the documentation for
several specified OpenStack projects.  At the moment to add a site to this
you need to edit ``modules/doc_server/manifests/init.pp`` and add a line as
follows:

.. code-block:: ruby
   :linenos:

   doc_server::site { "swift": }

In this example nginx will be configured to serve ``swift.openstack.org``
from ``/srv/docs/swift`` and ``swift.openstack.org/tarballs/`` from 
``/srv/tarballs/swift``

Lodgeit
-------

The lodgeit module installs and configures lodgeit [1]_ on required servers to
be used as paste installations.  For OpenStack we use a fork of this maintained
by dcolish [2]_ which contains bug fixes necessary for us to use it.

Puppet will configure lodgeit to use drizzle [3]_ as a database backend,
nginx [4]_ as a front-end proxy and upstart scripts to run the lodgeit
instances.  It will store and maintain local branch of the the mercurial
repository for lodgeit in ``/tmp/lodgeit-main``.

To use this module you need to add something similar to the following in the
main ``site.pp`` manifest:

.. code-block:: ruby
   :linenos:

   node "paste.openstack.org" {
     include openstack_server
     include lodgeit
     lodgeit::site { "openstack":
       port => "5000",
       image => "header-bg2.png"
     }

     lodgeit::site { "drizzle":
       port => "5001"
     }
   }

In this example we include the lodgeit module which will install all the
pre-requisites for Lodgeit as well as creating a checkout ready.
The ``lodgeit::site`` calls create the individual paste sites.

The name in the ``lodgeit::site`` call will be used to determine the URL, path
and name of the site.  So "openstack" will create ``paste.openstack.org``,
place it in ``/srv/lodgeit/openstack`` and give it an upstart script called
``openstack-paste``.  It will also change the h1 tag to say "Openstack".

The port number given needs to be a unique port which the lodgeit service will
run on.  The puppet script will then configure nginx to proxy to that port.

Finally if an image is given that will be used instead of text inside the h1
tag of the site.  The images need to be stored in the ``modules/lodgeit/files``
directory.

Lodgeit Backups
^^^^^^^^^^^^^^^

The lodgeit module will automatically create a git repository in ``/var/backups/lodgeit_db``.  Inside this every site will have its own SQL file, for example "openstack" will have a file called ``openstack.sql``.  Every day a cron job will update the SQL file (one job per file) and commit it to the git repository.

.. note::
   Ideally the SQL files would have a row on every line to keep the diffs stored
   in git small, but ``drizzledump`` does not yet support this.

Planet
------

The planet module installs Planet Venus [5]_ along with required dependancies
on a server.  It also configures specified planets based on options given.

Planet Venus works by having a cron job which creates static files.  In this
module the static files are served using nginx [4]_.

To use this module you need to add something similar to the following into the
main ``site.pp`` manifest:

.. code-block:: ruby
   :linenos:

   node "planet.openstack.org" {
     include planet

     planet::site { "openstack":
       git_url => "https://github.com/openstack/openstack-planet.git"
     }
   }

In this example the name "openstack" is used to create the site
``paste.openstack.org``.  The site will be served from
``/srv/planet/openstack/`` and the checkout of the ``git_url`` supplied will
be maintained in ``/var/lib/planet/openstack/``.

This module will also create a cron job to pull new feed data 3 minutes past each hour.

The ``git_url`` parameter needs to point to a git repository which stores the
planet.ini configuration for the planet (which stores a list of feeds) and any required theme data.  This will be pulled every time puppet is run.

Gerrit
------

The Gerrit puppet module configures the basic needs of a Gerrit server.  It does
not (yet) install Gerrit itself and mostly deals with the configuration files
and skinning of Gerrit.

Using Gerrit
^^^^^^^^^^^^

Gerrit is set up when the following class call is added to a node in the site
manifest:

.. code-block:: ruby

  class { 'gerrit':
    canonicalweburl => "https://review.stackforge.org/",
    email => "review@stackforge.org",
    github_projects => [ {
                         name => 'stackforge/MRaaS',
                         close_pull => 'true'
                         } ],
    logo => 'stackforge.png'
  }

Most of these options are self-explanitory.  The github_projects is a list of
all projects in GitHub which are managed by the gerrit server.

Skinning
^^^^^^^^

Gerrit is skinned using files supplied by the puppet module.  The skin is
automatically applied as soon as the module is executed.  In the site manifest
setting the logo is important:

.. code-block:: ruby

   class { 'gerrit':
     ...
     logo => 'openstack.png'
   }

This specifies a PNG file which must be stored in the ``modules/gerrit/files/``
directory.

Jenkins Master
--------------

The Jenkins Master puppet module installs and supplies a basic Jenkins
configuration.  It also supplies a skin to Jenkins to make it look more like an
OpenStack site.  It does not (yet) install the additional Jenkins plugins used
by the OpenStack project.

Using Jenkins Master
^^^^^^^^^^^^^^^^^^^^

In the site manifest a node can be configured to be a Jenkins master simply by
adding the class call below:

.. code-block:: ruby

   class { 'jenkins_master':
     site => 'jenkins.openstack.org',
     serveradmin => 'webmaster@openstack.org',
     logo => 'openstack.png'
   }

The ``site`` and ``serveradmin`` parameters are used to configure Apache.  You
will also need in this instance the following files for Apache to start::

   /etc/ssl/certs/jenkins.openstack.org.pem
   /etc/ssl/private/jenkins.openstack.org.key
   /etc/ssl/certs/intermediate.pem

The ``jenkins.openstack.org`` is replace by the setting in the ``site``
parameter.

Skinning
^^^^^^^^

The Jenkins skin uses the `Simple Theme Plugin
<http://wiki.jenkins-ci.org/display/JENKINS/Simple+Theme+Plugin>`_ for Jenkins.
The puppet module will install and configure most aspects of the skin
automatically, with a few adjustments needed.

In the site.pp file the ``logo`` parameter is important:

.. code-block:: ruby

   class { 'jenkins_master':
     ...
     logo => 'openstack.png'
   }

This relates to a PNG file that must be in the ``modules/jenkins_master/files/``
directory.

Once puppet installs this and the plugin is installed you need to go into
``Manage Jenkins -> Configure System`` and look for the ``Theme`` heading.
Assuming we are skinning the main OpenStack Jenkins site, in the ``CSS`` box
enter
``https://jenkins.openstack.org/plugin/simple-theme-plugin/openstack.css`` and
in the ``JS`` box enter
``https://jenkins.openstack.org/plugin/simple-theme-plugin/openstack.js``.

Jenkins Jobs
------------

The Jenkins Jobs puppet module uses configures a standard set of Jenkins jobs
for a given project using a batch of building blocks.  The standard jobs
created by this are:

* coverage
* docs
* merge
* pep8
* ppa
* python26
* python27
* tarball
* venv

These will be created prepended with the project name and a dash.

Using Jenkins Jobs
^^^^^^^^^^^^^^^^^^

To use the Jenkins Jobs module simply add a section to the puppet manifest for
the Jenkins node:

.. code-block:: ruby

   class { "jenkins_jobs":
     site => "openstack",
     projects => ["project1", "project2"]
   }

The ``site`` parameter is mainly used for git URLs and the ``projects``
parameter should be a list of projects the module maintains the jobs for.

Editing a Job
^^^^^^^^^^^^^

The list of Jenkins jobs is stored in ``modules/jenkins_jobs/add_jobs.pp``.
This file determines which templates should be pulled together to make the
job.  It will then automaticaly combine the building blocks to create an XML
file for Jenkins.

The XML building blocks can be found in ``modules/jenkins_jobs/templates/``.
The ``body.xml.erb`` file is the main template XML file which is populated using
a combination of other blocks.

An example job can be seen below:

.. code-block:: ruby

  jenkins_jobs::job { "${name}-coverage":
    site => "${site}",
    project => "${name}",
    job => "coverage",
    logrotate => template("jenkins_jobs/logrotate.xml.erb"),
    builders => [template("jenkins_jobs/builder_copy_bundle.xml.erb"), template("jenkins_jobs/builder_coverage.xml.erb")],
    publishers => template("jenkins_jobs/publisher_coverage.xml.erb"),
    triggers => template("jenkins_jobs/trigger_timed_15mins.xml.erb"),
    scm => template("jenkins_jobs/scm_git.xml.erb")
  }

The templates can be singles or in arrays to chain multiple things together,
but the XML will be built in the order of the items array.  So in this case
the builder ``builder_copy_bundle`` will be added, then ``builder_coverage``.


.. rubric:: Footnotes
.. [1] `Lodgeit homepage <http://www.pocoo.org/projects/lodgeit/>`_
.. [2] `dcolish's Lodgeit fork <https://bitbucket.org/dcolish/lodgeit-main>`_
.. [3] `Drizzle homepage <http://www.dirzzle.org/>`_
.. [4] `nginx homepage <http://nginx.org/en/>`_
.. [5] `Planet Venus <http://intertwingly.net/code/venus/docs/index.html>`_
