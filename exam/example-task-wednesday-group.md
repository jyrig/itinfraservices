Example tasks for the exam (Wednesday group)
--------------------------------------------

 1. Write Ansible playbook `infra.yaml` to deploy the following infrastructure.
 2. Create a dashboard in Grafana with at least one graph for every service and
    one for every virtual machine (min. 10 graphs).
 3. Write a backup SLA document for this infrastructure (free text).

Application server:
 - 2 Wordpress instances (in Docker containers)
 - HAProxy

Backend server:
 - MySQL server with a database for Wordpress

Monitoring server:
 - InfluxDB
 - Prometheus
 - Grafana (in Docker container)

DNS servers:
 - Bind master
 - Bind slave

It is allowed to co-locate some of the services.
Services that **SHOULD NOT** run on the same machine are:
 - Wordpress and MySQL
 - Wordpress and InfluxDB
 - Wordpress and Prometheus
 - Bind master and Bind slave

Requirements to infrastructure setup:
 1.  Wordpress landing page should be available at `http://www.<your-domain>`
 2.  HAProxy should serve all Wordpress containers, i. e. Wordpress landing page
     should still be accessible if one of Wordpress containers is stopped
 3.  Every virtual machine should forward syslog to InfluxDB
 4.  Every virtual machine should have Prometheus node exporter set up and
     running
 5.  Prometheus MySQL exporter should be set up and running on MySQL server
 6.  Prometheus Bind exporter should be set up and running on every Bind server
 7.  Prometheus configuration should not use any IP addresses -- only hostnames
 8.  Grafana should have at least 2 data sources configured: InfluxDB and
     Prometheus
 9.  All services should 'survive' the machine restart -- i. e. started
     automatically once machine is rebooted
 10. All passwords and secrets in your GitHub repository must be encrypted!

Requirements to Ansible code:
 1. Playbook `infra.yaml` should not contain any tasks -- only included roles
 2. Playbook `infra.yaml` should not ask for any passwords -- only read the
    values from variable files
 3. Playbook `infra.yaml` should not configure SSH, sudo or networking -- if
    needed, do this in a separate playbook called `init.yaml`

This command (run as unprivileged user, with no additional parameters) must be
enough to provision the entire required infrastructure:

    ansible-playbook infra.yaml


Hints
-----

Example setup:
 - Virtual machine 1: Wordpress, Bind slave
 - Virtual machine 2: MySQL, InfluxDB, Prometheus, Grafana, Bind master

You may need to add more memory to virtual machine running MySQL, InfluxDB and
Prometheus if you decide to co-locate these.

APT packages for Prometheus exporters can be found here:
https://packages.ubuntu.com/search?suite=bionic&keywords=prometheus

The only password you need is the MySQL password for Wordpress. No other
passwords are needed for this setup, but you may introduce some if you feel the
need -- it is not forbidden. Still, **all** the secrets must be encrypted.

You may want to make a screenshot of your Grafana dashboard and upload it to the
your GitHub repository.
