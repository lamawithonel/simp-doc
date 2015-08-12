Upgrading SIMP
==============

This chapter provides information on how to upgrade a running instance
to the latest codebase.

Pre-Upgrade Recommendations
---------------------------

The following process should be followed before upgrade.

.. list-table::
   :widths: 8 276
   :header-rows: 1

   * - Step
     - Process/Action
   * - 1.
     - Run ``puppet agent --disable`` to disable puppet.
   * - 
     - **Note:** If you think you will need more than 4 hours to complete this task, also disable puppet in root's crontab.
   * - 2.
     - You may wish to block all communications with agents while updating the server. This is not required but could spare you some headaches if something doesn't work properly.
   * - 
     - 
   * - 
     - The simplest way to do this is to set the catalog retrieval capability to 127.0.0.1 in /etc/puppet/auth.conf as shown below.
   * - 
     - 
   * - 
     - .. code-block:: bash
   * - 
     - 
   * - 
     - path ~ ^/catalog/([^/]+)$
   * - 
     - method find
   * - 
     - # Uncomment this when complete and delete the other entries
   * - 
     - #allow $1
   * - 
     - allow 127.0.0.1
   * - 
     - 
   * - 
     - 
   * - 
     - Using the syntax above, you can add fully qualified domain names, one at a time, to the 'allow' list and only those hosts will be able to retrieve their catalog from the running server. 127.0.0.1 serves as a placeholder so that no host can actually retrieve their catalog.

Table: SIMP Pre Upgrade

Migrating To Environments
-------------------------

SIMP 4.1 and 5.0 used the traditional, Rack-based, Puppet Master.
Starting with 4.2 and 5.1, SIMP now uses the Clojure-based Puppet
Server. Unfortunately, there are some conflicts with directly upgrading
from the Puppet Master to the Puppet Server since some of the RPM
package prerequisites conflict. This new Puppet Server can properly
utilize Puppet Environments. To provide our users with this capability,
and to facilitate more dynamic workflows in the future, the SIMP team
has migrated **all** existing material to a native *simp* environment.
To help facilitate your migration, the SIMP team has created two
migration scripts that both upgrade your Puppet Server and migrate your
existing data into the new *simp* environment.

    **Warning**

    You must have at least **2.2G** of **free memory** to run the new
    Puppet Server.

Migration Script Features
-------------------------

The migration script will perform the following actions on your system:

-  Remove the **puppet-server** package from your system

-  Install the **puppetserver** package onto your system

-  Update all packages from your repositories

-  Create a backup folder at
   */etc/puppet/environments/pre\_migration.simp*

-  Create a Git repository in the backup folder under a timestamped
   directory

-  Commit all current materials from /etc/puppet into the backup Git
   repository

-  Checkout the backup Git repository under the timestamped directory as
   *backup\_data* for ease of use

-  Migrate all existing data into the new **simp** environment under
   */etc/puppet/environments/simp*

    **Note**

    All future upgrades will only affect the new **simp** environment.
    You may create new environments and/or modify the contents of
    */etc/puppet/modules* without fear of the SIMP packages overwriting
    your work.

Migration Script Execution
--------------------------


Table: Executing the Migration Script

Your new Puppet Server should now be running and a run of *puppet agent
-t* should complete as usual.

Converting from Extdata to Hiera
--------------------------------

SIMP now uses Hiera natively instead of Extdata. Tools have been put
into place by Puppet Labs and SIMP to make the conversion as easy as
possible. Two scripts have been provided to automatically convert
generic csv files and simp\_def.csv to yaml. The first example shows how
to convert an Extdata csv file called foo.csv into a Hiera yaml file
called bar.yaml:

.. code-block:: ruby

                extdata2hiera -i foo.csv -o bar.yaml


The second example shows how to convert an Extdata csv simp\_def file
called simp\_def.csv into a Hiera yaml file called simp\_def.yaml.

.. code-block:: ruby

                simpdef2hiera --in simp_def.csv --out simp_def.yaml


Puppet will automatically retrieve class parameters from Hiera, using
lookup keys like myclass::parameter\_one. Puppet classes can optionally
include parameters in their definition. This lets the class ask for data
to be passed in at the time that it’s declared, and it can use that data
as normal variables throughout its definition.

There are two main ways to reference Hiera data in puppet manifests. The
first, and preferred way, is to use the automatic class variable lookup
capability. For each class that you create, the variables will be
automatically discovered in hiera should they exist. This is quite
powerful in that you no longer need to provide class parameters in your
manifests and can finally properly separate your data from your code.

    **Note**

    For more information on the lookup functions, see
    http://docs.puppetlabs.com/hiera/1/puppet.html#hiera-lookup-functions.

.. code-block:: ruby

            # Some class file in scope...
            class foo (
              $param1 = 'default1'
              $param2 = 'default2'
            ) { .... }

            # /etc/puppet/hieradata/default.yaml
            ---
            foo::param1: 'custom1'


The second is similar to the old Extdata way, and looks like the
following:

.. code-block:: ruby

            $var = hiera("some_hiera_variable", "default_value")


The following is from the Puppet Labs documentation, and explains the
reason for switching to Hiera.

Automatic parameter lookup is good for writing reusable code because it
is regular and predictable. Anyone downloading your module can look at
the first line of each manifest and easily see which keys they need to
set in their own Hiera data. If you use the Hiera functions in the body
of a class instead, you will need to clearly document which keys the
user needs to set.

    **Note**

    For more information on hiera and puppet in general, see
    http://docs.puppetlabs.com/hiera/1/complete_example.html.

Scope Functions
---------------

All scope functions must take arguments in array form. For example in
/etc/puppet/modules/apache/templates/ssl.conf.erb, <%=
scope.function\_bracketize(l) %> becomes <%=
scope.function\_bracketize([l]) %>.

Commands
--------

Deprecated commands mentioned in Puppet 2.7 upgrade are now completely
removed.

Lock File
---------

Puppet agent now uses the two lock files instead of one. These are the
run-in-progress lockfile (agent\_catalog\_run\_lockfile) and the
disabled lockfile (agent\_disabled\_lockfile). The puppetagent\_cron
file (made by the pupmod module) must be edited to suit this change.