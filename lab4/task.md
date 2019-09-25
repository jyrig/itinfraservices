# Lab 4

## Task

Write an Ansible playbook `wordpress.yml` that has multiple plays.

One play should install and configure
 - Apache HTTPd with PHP module
 - PHP module to connect to MySQL
 - WordPress and MySQL connection parameters

Another play should install and configure
 - MySQL server (MariaDB is also acceptable if you like it better)
 - MySQL user for WordPress

These plays should use different `hosts` pattern, so that it should be possible
to install WordPress and MySQL on different servers.

You may add more plays if needed.

Requirements:
 - Apache should serve WordPress from the default HTTP port 80
 - WordPress should use dedicated username/password to connect to MySQL
 - No secrets should be committed to Git repository

Expected files in your Git repository:
 - `lab4/hosts`
 - `lab4/wordpress.yml`

May add more files as needed.

Note: WordPress setup may require some configuration in the web UI for the first
start. It is okay to do this step manually.

## Acceptance test

This command run once in your lab directory should install and configure all the
required services:

    ansible-playbook wordpress.yml

It is okay to run it with one additional parameter:

    ansible-playbook wordpress.yml --ask-vault-pass

WordPress web interface should be served from the default web server HTTP port.

No actual secrets are found in the Git history.

## Hints

Use the previous lab task as a starting point. You already have all the needed
steps to install Apache HTTPd and make it work with PHP. You need to:
 - Install PHP module to connect to MySQL
 - Install WordPress
 - Configure WordPress
 - Add another play that installs the MySQL server and manages the users

One possible way to install WordPress is their official guide:
https://wordpress.org/support/article/how-to-install-wordpress

Another possible way is to use the package from Ubuntu repository:
https://packages.ubuntu.com/bionic/wordpress.
This may be simpler to install but more tricky to configure.

Ansible variables tutorial can be found here:
https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html

Ansible Vault tutorial can be found here:
https://docs.ansible.com/ansible/latest/user_guide/vault.html

You inventory file could look like this:

    [database_servers]
    db1  ansible_host=127.0.0.1 ansible_port=2222 (modify parameters as needed)

    [frontend_servers]
    web1  ansible_host=127.0.0.1 ansible_port=2222 (modify parameters as needed)

Note that host identifiers are different but the parameters are the same. With
that you can have separate plays for database servers and frontend servers;
should you later decide to separate the servers you will just need to change the
inventory file, not the play code.

If you have committed the secret, change it in the configuration files, and make
sure not to commit the updated value accidentally!
