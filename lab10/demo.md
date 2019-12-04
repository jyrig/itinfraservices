# Lab 10 demo: Proxies and load balancers

Intro
-----

Expected setup:
 1. Virtual machine with Docker daemon installed, up and running
 2. `www.demo` hostname resolving to this machine

Note that you may use another hostname for your setup -- just make sure to
update the related configuration accordingly.

> Note: Ansible code examples in this task are not complete solution; they just
> illustrate some of the setup steps.


First Service
-------------

Let's deploy a simple web service in Docker; one HTML file served by Nginx will
be enough. This can be achieved with these Ansible tasks:

    - name: Add service files
      copy:
        src: index.html
        dest: /srv/index.html

    - name: Start service instance 1
      docker_container:
        name: www-1
        image: nginx
        published_ports: 8180:80
        volumes: /srv:/usr/share/nginx/html:ro

 - `index.html` file needs to be created first
 - Docker image used: https://hub.docker.com/_/nginx

Once deployed, the Nginx should listen on port 80 of the container named `www-1`
and serve the `index.html` file mentioned above. The service page should be
publicly available at http://www.demo:8180.


HTTP Proxy
----------

Although the service is up an running now the URL http://www.demo:8180 is not
really convenient to use. Let's add another service that would listen on
`www.demo` port 80 for client requests and forward them to `www-1` container.
Its response will be sent to client on behalf of this service -- this approach
is called 'reverse proxying'.

There are lots of tools that can do reverse proxying, but quite popular choices
for simpler setups are [Nginx](https://www.nginx.com) and
[HAProxy](https://www.haproxy.org). We will use HAPRoxy in this demo.

HAProxy can be installed on the same machine where Docker daemon is running. Not
necessarily this is the best setup for production use, but for the demo it is
completely fine.

HAProxy can be installed with these Ansible tasks:

    - name: Install HAProxy
      apt:
        package: haproxy

    - name: Enable and start HAProxy
      service:
        name: haproxy
        state: started
        enabled: yes

And configured with this task:

    - name: Configure HAProxy
      template:
        src: haproxy.cfg
        dest: /etc/haproxy/haproxy.cfg
        validate: haproxy -c -f %s
      notify:
        haproxy_restart

`haproxy.cfg` file that comes with the APT package can be reused; we will add
some additional configuration there later.

> Note the `validate` attribute of the `template` module -- this will validate
> the configuration file before writing it.

Next, let's configure HAProxy as a reverse proxy for our web service -- add this
to HAProxy configuration file:

    listen www.demo
        bind :80
        server www-1 localhost:8180

Reload or restart HAProxy. That's it! The service should now be available at
http://www.demo (default port 80). HTTP requests are now received by HAProxy
and forwarded to the service running on local port 8180.

> Note that starting from this point we do not need service port
> (www.demo:8180) to be publicly available; clients do not connect there
> directly anymore, only HAProxy does.

You can find more details on basics of HAProxy configuration here:
https://www.haproxy.com/blog/the-four-essential-sections-of-an-haproxy-configuration/


HAProxy stats
-------------

HAProxy comes with default statistics page included, but it is not being served
by default. It should be activated explicitly in HAProxy configuration:

    listen www.demo
        bind :80
        server www-1 localhost:8180

        stats enable
        stats uri /proxy-stats
        #stats refresh 5s  # enable this to refresh the stats page automatically

Once configuration is reloaded the stats page will appear at
http://www.demo/proxy-stats. Yes, this page looks a bit 90's but has a lot of
useful information. You can find more details about HAProxy status page here:
https://www.haproxy.com/blog/exploring-the-haproxy-stats-page/

> Good news! There is
> [HAProxy-to-Prometheus exporter](https://github.com/prometheus/haproxy_exporter)
> so you can visualize HAProxy stats in Grafana with a few additional steps.
> Also recent versions of HAProxy come with Prometheus endpoint built in, so you
> don't even need a separate exporter:
> https://www.haproxy.com/blog/haproxy-exposes-a-prometheus-metrics-endpoint/


More service instances
----------------------

No once we have a reverse proxy set up we can add another service instance and
turn this simple proxy into actual load balancer.

Second service instance can be added with this Ansible task:

    - name: Start service instance 2
      docker_container:
        name: www-2
        image: nginx
        published_ports: 8280:80
        volumes: /srv:/usr/share/nginx/html:ro

Here the same HTML file is used that is already served by service instance 1;
this is the same service after all.

> Note that in the demo a slightly different files were used to deploy another
> service instance. You may want to do the same to better understand which
> service instance is serving each request.

HAProxy configuration should be modified accordingly to support another service
instance as a backend:

    listen www.demo
        bind :80
        server www-1 localhost:8180
        server www-1 localhost:8280

Once the configuration is reloaded you will notice that another backend was
added to HAProxy stats page; it should be called `www-2`.

Try requesting the service page multiple times now: http://www.demo. It should
be served from different backends.

> Note that if you are are refreshing the page in your web browser you may need
> to clean the cache for every refresh, otherwise the browser the show you the
> cached page without actually contacting the backend web server. In Firefox the
> page cache can be cleaned `Ctrl+F5`.


Health checks
-------------

So far our setup is not much better than round-robin DNS. If one of the backend
instances is stopped HAProxy will fail to detect it. Try this:

    docker stop www-1

and refresh the service page multiple times: http://www.demo. You will notice
that some of the requests fail, but the others are still served.

This is happening because HAProxy has no idea if this exact backend instance is
alive or not. It just forwards the next request to the next backend instance
from the list. Luckily we can tell HAProxy to perform these health checks before
actually forwarding the request to the instance, and it is easier than you
think -- just add a `check` keyword after the server address:

    listen www.demo
        bind :80
        server www-1 localhost:8180 check
        server www-1 localhost:8280 check

Check the proxy stats page now: http://www.demo/proxy-stats -- note that the
backend rows are painted green and red now; this means that HAProxy is able to
identify if the backend is healthy or not.

Service page is also served without errors now: http://www.demo. It is served
by only one backend instance as we've stopped another one, but the clients
should not get eny errors anymore.
