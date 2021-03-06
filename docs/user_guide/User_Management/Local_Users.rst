.. _local_user_management:

Managing Local/Service Users
============================

Though the SIMP team **highly recommends** using :ref:`LDAP <ldap_user_management>`
to centrally manage your users, you may occasionally need to set up a
:term:`service account` or specific local users on your systems.

This section walks you through doing this in a way that is compatible with SIMP.

The following examples assume that you are using the *site* module to set up
your users. The examples may easily be extrapolated into defined types if you
wish but are presented as classes for simplicity.

If you are not familiar with setting up :term:`SSH` keys, you may want to follow the
relevant `GitHub documentation <https://help.github.com/articles/generating-ssh-keys/>`__.

Service Account
---------------

.. code-block:: ruby

  class site::service_account {
    include 'ssh'

    $_svc_account_user           = 'svcuser'
    $_svc_account_group          = 'svcgroup'
    $_svc_account_id             = '1777',
    $_svc_account_homedir        = "/var/local/${_svc_account_user}"

    # Since this is a service account, automatically generate an SSH key for
    # the user and store it on the Puppet master for distribution.
    $_svc_account_ssh_private_key = ssh_keygen($_svc_account_user, '2048', true)
    $_svc_account_ssh_public_key = ssh_keygen($_svc_account_user, '2048')

    group { $_svc_account_group:
      gid       => $_svc_account_id,
      allowdupe => false,
    }

    user { $_svc_account_user:
      uid        => $_svc_account_id,
      allowdupe  => false,
      gid        => $_svc_account_group,
      home       => $_svc_account_homedir,
      managehome => true,
      shell      => '/bin/bash'
    }

    file { "${_svc_account_homedir}/.ssh":
      ensure => directory,
      owner  => $_svc_account_user,
      group  => $_svc_account_group,
      mode   => '0700'
    }

    ssh_authorized_key { $_svc_account_user:
      type    => 'ssh-rsa',
      key     => $_svc_account_ssh_public_key,
      target  => "${_svc_account_homedir}/.ssh/authorized_keys",
      require => [
        File["${_svc_account_homedir}/.ssh"],
        User[$_svc_account_user]
      ]
    }

    file { "${_svc_account_homedir}/.ssh/id_rsa":
      mode    => '0600',
      owner   => $_svc_account_user,
      group   => $_svc_account_group,
      content => $_svc_account_ssh_private_key
    }

    file { "/etc/ssh/local_keys/${_svc_account_user}":
      owner  => 'root',
      group  => $_svc_account_group,
      mode   => '0644',
      source => "puppet:///site/ssh_autokeys/${_svc_account_user}.pub"
    }

    sudo::user_specification { $_svc_account_user:
      user_list => ["(${_svc_account_group})"],
      host_list => [$::fqdn],
      runas     => 'root',
      cmnd      => ['/bin/cat /var/log/app.log'],
      passwd    => false
    }

    # Allow this service account from everywhere
    pam::access::manage { "Allow ${_svc_account_user}":
      users   => ${_svc_account_user},
      origins => ['ALL']
    }
  }

Local User Account
------------------

.. code-block:: ruby

  class site::service_account {
    include 'ssh'

    $_local_account_user           = 'localuser'
    $_local_account_group          = 'localgroup'
    $_local_account_id             = '1778',

    # You'll probably want this in /home unless you're using NFS
    $_local_account_homedir        = "/home/${_local_account_user}"

    # You'll need to get this from the user as it is their public key.
    $_local_account_ssh_public_key = 'AAA...=='

    group { $_local_account_group:
      gid       => $_local_account_id,
      allowdupe => false,
    }

    user { $_local_account_user:
      uid        => $_local_account_id,
      allowdupe  => false,
      gid        => $_local_account_group,
      home       => $_local_account_homedir,
      managehome => true,
      shell      => '/bin/bash'
    }

    file { "/etc/ssh/local_keys/${_local_account_user}":
      owner  => 'root',
      group  => $_local_account_group,
      mode   => '0644',
      source => "puppet:///site/ssh_autokeys/${_local_account_user}.pub"
    }

    sudo::user_specification { $_local_account_user:
      user_list => ["(${_local_account_group})"],
      host_list => [$::fqdn],
      runas     => 'root',
      cmnd      => ['/bin/cat /var/log/app.log'],
      passwd    => false
    }

    # Allow this account from everywhere
    pam::access::manage { "Allow ${_local_account_user}":
      users   => ${_local_account_user},
      origins => ['ALL']
    }
  }

Testing
-------

The table below lists the steps to test that the configuration was
applied correctly.

1. Log on to a server that has the template code configuration applied.
2. Type ``su - ***<USERNAME>***``
3. Type ``exec /usr/bin/ssh-agent /bin/bash`` to ensure that ssh-agent has a
   shell running.
4. Type ``/usr/bin/ssh-add`` to attach the user's certificates.
5. **Optional**: Type ``/usr/bin/ssh-add -l`` to double check that the user's
   certificates were added successfully.
6. Type ``ssh ***<HOST>***`` to SSH to a target machine that has the template
   code configuration applied.

If successful, the user should be authenticated and gain access to the
target machine without entering a password.

If the user is prompted for a password, check to see if the permissions are set
up properly and that the certificate keys are in the correct locations. In
addition, check the ``/etc/security/access.conf`` file to ensure that it
contains the user or user's group in an allow statement. See ``access.conf(5)``
for details.
