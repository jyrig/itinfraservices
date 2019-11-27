# Lab 9

## Task 1

Modify your `wordpress` playbook from [lab 4](../lab4/task.md) to deploy
Wordpress in a Docker container, and MySQL server to a virtual machine.

Requirements:
 1. Docker container with Wordpress and MySQL server should be running on
    **different** machines
 2. Wordpress should listen on port 8080 of the Docker host
 3. Wordpress process should be able to connect to MySQL server
 4. Wordpress user should only have privileges in `wordpress` database, not any
    other database
 5. Ansible plays should not include any tasks directly -- only roles
 6. No SSH, sudo or networking configuration should be done in `wordpress`
   playbook -- only in `init` playbook

It is okay to configure MySQL connection parameters for Wordpress via web
interface (can be done with Ansible, but don't have to, for this task).


## Task 2

Modify your `grafana` playbook from [lab 7](../lab7/tasks) to deploy Grafana in
a Docker container, and Prometheus server to a virtual machine.

Requirements:
 1. Docker container with Grafana and Prometheus server should be running on
    **different** machines
 2. Grafana should listen on port 3000 of the Docker host
 3. Grafana process should be able to connect to Prometheus server
 4. Grafana should have Prometheus datasource configured
 5. Grafana should have at least one dashboard configured, with at least one
    graph based on Prometheus datasource
 6. Ansible plays should not include any tasks directly -- only roles
 7. No SSH, sudo or networking configuration should be done in `grafana`
    playbook -- only in `init` playbook

It is okay to reuse the same Docker host from the task 1 for Grafana container.

It is okay to co-locate MySQL server (from the task 1) and Prometheus server on
one virtual machine.


## Expected files in your Git repository

 - `lab9/ansible.cfg`
 - `lab9/hosts`
 - `lab9/init.yaml`
 - `lab9/wordpress.yaml`
 - `lab9/grafana.yaml`


## Acceptance test

This command run once in your lab directory should install and configure all the
required services:

    ansible-playbook init.yaml --ask-pass --ask-become-pass
    ansible-playbook wordpress.yml
    ansible-playbook grafana.yml


## Hints

This Ansible module may be helpful to manage Docker containers:
https://docs.ansible.com/ansible/latest/modules/docker_container_module.html

Note that it requires additional Python module installed on managed host,
similarly to `python3-mysqldb`. Exact package name can be found here:
https://packages.ubuntu.com/search

In order to allow connections from Wordpress container to MySQL you may need to:
 - reconfigure MySQL server to listen on public interface
 - set the host for MySQL user explicitly; default is localhost
