Exam task (Monday group)
------------------------

 1. Write Ansible playbook `infra.yaml` to deploy the following infrastructure.
 2. Create a dashboard in Grafana with at least one graph for every service
    (min. 6 graphs).
 3. Write a backup SLA document for this infrastructure (free text).

Application server:
 - Wordpress instance (in Docker container)

Backend server:
 - MySQL server with a database for Wordpress

Monitoring server:
 - Prometheus
 - Grafana (in Docker container)

DNS servers:
 - Bind master
 - Bind slave

It is allowed to co-locate some of the services.
Services that **SHOULD NOT** run on the same machine are:
 - Wordpress and MySQL
 - Wordpress and Prometheus
 - Bind master and Bind slave

Requirements to infrastructure setup:
 1. Wordpress landing page should be available at `http://www.<your-domain>`
 2. Every virtual machine should have Prometheus node exporter set up and
    running
 3. Prometheus MySQL exporter should be set up and running on MySQL server
 4. Prometheus Bind exporter should be set up and running on every Bind server
 5. Prometheus configuration should not use any IP addresses -- only hostnames
 6. All services should 'survive' the machine restart -- i. e. started
    automatically once machine is rebooted
 7. All passwords and secrets in your GitHub repository must be encrypted!

Requirements to Ansible code:
 1. Playbook `infra.yaml` should not contain any tasks -- only included roles
 2. Playbook `infra.yaml` should not ask for any passwords -- only read the
    values from variable files
 3. Playbook `infra.yaml` should not configure SSH, sudo or networking -- if
    needed, do this in a separate playbook called `init.yaml`

This command (run as unprivileged user, with no additional parameters) must be
enough to provision the entire required infrastructure:

    ansible-playbook infra.yaml

Once ready, make a screenshot from your Grafana dashboard and upload it to your
Git repository.

Files expected in your repository (you can add more files as needed):
 - `exam/ansible.cfg`
 - `exam/hosts`
 - `exam/infra.yaml`

It is allowed to use any materials you feel sensible, except
 - videos
 - sound
 - other students' help
