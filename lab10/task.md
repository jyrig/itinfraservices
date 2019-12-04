# Lab 10

## Task

Modify your `wordpress` playbook from [lab 9](../lab9/task.md) to add a reverse
proxy and load balancer in front of Wordpress instances.

Requirements:
 1. Every Wordpress instance should be running in a Docker container
 2. It should be possible to run 1, 2 or 3 Wordpress instaces
 3. Wordpress database connection parameters should be provided with Ansible
    (not manually via web UI)
 4. HAProxy should listen on public interface port 80
 5. HAProxy should check the health of Wordpress instances; if the container is
    stopped no requests should be forwarded to that backend anymore;
    **this should work without HAProxy restart!**
 5. Wordpress user should only have privileges in `wordpress` database, not any
    other database
 6. No SSH, sudo or networking configuration should be done in `wordpress`
    playbook -- only in `init` playbook

It is okay if your code supports more than 3 Wordpress instances, but it is not
required.


## Expected files in your Git repository

 - `lab10/ansible.cfg`
 - `lab10/hosts`
 - `lab10/init.yaml`
 - `lab10/wordpress.yaml`


## Acceptance test

This command run once in your lab directory should install and configure all the
required services:

    ansible-playbook init.yaml --ask-pass --ask-become-pass
    ansible-playbook wordpress.yml

Verify that HAProxy instance does backend health checks:
 1. Stop one of the Wordpress instances with `docker stop <container-id>`
 2. **Do not restart or reload HAProxy**
 3. Wait for 1..2 seconds and refresh the service page multiple times
 4. You should receive no errors
 5. Run `ansible-playbook wordpress.yml` to restore the container


## Hints

This Ansible module may be helpful to manage Docker containers:
https://docs.ansible.com/ansible/latest/modules/docker_container_module.html.
Check out the `env` parameter there and also
[Wordpress Docker image usage instruction](https://hub.docker.com/_/wordpress/)
for ideas how to configure Wordpress in Docker.

Add a new role called `haproxy` and use it to set up HAProxy. HAProxy
configuration file installed by APT package is good enough, you will just need
to add some proxying directives there.

Here is one nice HAProxy configuration guide:
https://www.haproxy.com/blog/the-four-essential-sections-of-an-haproxy-configuration/.
It is a bit different approach from what was shown in the demo; you can use
configure HAProxy either with `listen` or `frontend/backend` directives -- as
you like more.
