.. _pki_validation:

Checking Your SIMP PKI Communication
====================================

SIMP comes with a fully functional :term:`Public Key Infrastructure` in the
guise of an aptly named Fake CA.

The Fake CA can be very useful for getting your environment running prior to
obtaining proper certificates from an official CA.

.. warning::

  The Fake CA is **not** hardware backed by default and should not be used for
  sensitive cryptographic operations unless there is no other alternative

Each Puppet environment contains its own Fake CA and, therefore, you must know
which environment is serving the systems that are having issues prior to
proceeding.

For this section, we will assume that it is the 'simp' environment located at
the active environment path.

.. note::

  Just as with Puppet certificates, the time on your system must be correct and
  your DNS must be fully functional. Check that these are correct before
  proceeding.

For the remainder of this section, we will assume that the :term:`FQDN` of the
system with issues is 'system.my.domain' and the LDAP server to which it is
attempting to connect is 'ldap.my.domain'.

Navigate to the environment *keydist* directory and validate the system
certificates.

When validating certificates, you want to make sure that there are no errors
regarding your certificate or :term:`CA`. Ideally, the command will simply
return the string 'OK'.

.. code-block:: bash

  $ cd `puppet config print environmentpath`/simp/keydist

  # Validate the client system
  $ openssl verify -CApath cacerts system.my.domain

  # Validate the LDAP system
  $ openssl verify -CApath cacerts ldap.my.domain

If there are any issues, you may need to follow the Fake CA README to generate
new certificates for one or more of your hosts.
