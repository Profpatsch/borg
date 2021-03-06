.. include:: global.rst.inc
.. _deployment:

Deployment
==========

This chapter will give an example how to setup a borg repository server for multiple
clients.

Machines
--------

There are multiple machines used in this chapter and will further be named by their
respective fully qualified domain name (fqdn).

* The backup server: `backup01.srv.local`
* The clients:

  - John Doe's desktop: `johndoe.clnt.local`
  - Webserver 01: `web01.srv.local`
  - Application server 01: `app01.srv.local`

User and group
--------------

The repository server needs to have only one UNIX user for all the clients.
Recommended user and group with additional settings:

* User: `backup`
* Group: `backup`
* Shell: `/bin/bash` (or other capable to run the `borg serve` command)
* Home: `/home/backup`

Most clients shall initiate a backup from the root user to catch all
users, groups and permissions (e.g. when backing up `/home`).

Folders
-------

The following folder tree layout is suggested on the repository server:

* User home directory, /home/backup
* Repositories path (storage pool): /home/backup/repos
* Clients restricted paths (`/home/backup/repos/<client fqdn>`):

  - johndoe.clnt.local: `/home/backup/repos/johndoe.clnt.local`
  - web01.srv.local: `/home/backup/repos/web01.srv.local`
  - app01.srv.local: `/home/backup/repos/app01.srv.local`

Restrictions
------------

Borg is instructed to restrict clients into their own paths:
``borg serve --restrict-path /home/backup/repos/<client fqdn>``

There is only one ssh key per client allowed. Keys are added for ``johndoe.clnt.local``, ``web01.srv.local`` and
``app01.srv.local``. But they will access the backup under only one UNIX user account as:
``backup@backup01.srv.local``. Every key in ``$HOME/.ssh/authorized_keys`` has a
forced command and restrictions applied as shown below:

::

  command="cd /home/backup/repos/<client fqdn>;
           borg serve --restrict-path /home/backup/repos/<client fqdn>",
           no-port-forwarding,no-X11-forwarding,no-pty <keytype> <key> <host>

.. note:: The text shown above needs to be written on a single line!

The options which are added to the key will perform the following:

1. Force command on the ssh key and dont allow any other command to run
2. Change working directory
3. Run ``borg serve`` restricted at the client base path
4. Restrict ssh and do not allow stuff which imposes a security risk

Due to the ``cd`` command we use, the server automatically changes the current
working directory. Then client doesn't need to have knowledge of the absolute
or relative remote repository path and can directly access the repositories at
``<user>@<host>:<repo>``.

.. note:: The setup above ignores all client given commandline parameters
          which are normally appended to the `borg serve` command.

Client
------

The client needs to initialize the `pictures` repository like this:

 borg init backup@backup01.srv.local:pictures

Or with the full path (should actually never be used, as only for demonstrational purposes).
The server should automatically change the current working directory to the `<client fqdn>` folder.

  borg init backup@backup01.srv.local:/home/backup/repos/johndoe.clnt.local/pictures

When `johndoe.clnt.local` tries to access a not restricted path the following error is raised.
John Doe tries to backup into the Web 01 path:

  borg init backup@backup01.srv.local:/home/backup/repos/web01.srv.local/pictures

::

  ~~~ SNIP ~~~
  Remote: borg.remote.PathNotAllowed: /home/backup/repos/web01.srv.local/pictures
  ~~~ SNIP ~~~
  Repository path not allowed

Ansible
-------

Ansible takes care of all the system-specific commands to add the user, create the
folder. Even when the configuration is changed the repository server configuration is
satisfied and reproducable.

Automate setting up an repository server with the user, group, folders and
permissions a Ansible playbook could be used. Keep in mind the playbook
uses the Arch Linux `pacman <https://www.archlinux.org/pacman/pacman.8.html>`_
package manager to install and keep borg up-to-date.

::

  - hosts: backup01.srv.local
    vars:
      user: backup
      group: backup
      home: /home/backup
      pool: "{{ home }}/repos"
      auth_users:
        - host: johndoe.clnt.local
          key: "{{ lookup('file', '/path/to/keys/johndoe.clnt.local.pub') }}"
        - host: web01.clnt.local
          key: "{{ lookup('file', '/path/to/keys/web01.clnt.local.pub') }}"
        - host: app01.clnt.local
          key: "{{ lookup('file', '/path/to/keys/app01.clnt.local.pub') }}"
    tasks:
    - pacman: name=borg state=latest update_cache=yes
    - group: name="{{ group }}" state=present
    - user: name="{{ user }}" shell=/bin/bash home="{{ home }}" createhome=yes group="{{ group }}" groups= state=present
    - file: path="{{ home }}" owner="{{ user }}" group="{{ group }}" mode=0700 state=directory
    - file: path="{{ home }}/.ssh" owner="{{ user }}" group="{{ group }}" mode=0700 state=directory
    - file: path="{{ pool }}" owner="{{ user }}" group="{{ group }}" mode=0700 state=directory
    - authorized_key: user="{{ user }}"
                      key="{{ item.key }}"
                      key_options='command="cd {{ pool }}/{{ item.host }};borg serve --restrict-to-path {{ pool }}/{{ item.host }}",no-port-forwarding,no-X11-forwarding,no-pty'
      with_items: auth_users
    - file: path="{{ home }}/.ssh/authorized_keys" owner="{{ user }}" group="{{ group }}" mode=0600 state=file
    - file: path="{{ pool }}/{{ item.host }}" owner="{{ user }}" group="{{ group }}" mode=0700 state=directory
      with_items: auth_users

Enhancements
------------

As this chapter only describes a simple and effective setup it could be further
enhanced when supporting (a limited set) of client supplied commands. A wrapper
for starting `borg serve` could be written. Or borg itself could be enhanced to
autodetect it runs under SSH by checking the `SSH_ORIGINAL_COMMAND` environment
variable. This is left open for future improvements.

When extending ssh autodetection in borg no external wrapper script is necessary
and no other interpreter or apllication has to be deployed.

See also
--------

* `SSH Daemon manpage <http://www.openbsd.org/cgi-bin/man.cgi/OpenBSD-current/man8/sshd.8>`_
* `Ansible <https://docs.ansible.com>`_
