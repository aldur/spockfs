The SpockFS uWSGI plugin
========================

This is a plugin (with official modifier 179, try to guess why ;) ) for the uWSGI server allowing it to expose directories via the SpockFS protocol.

A single instance of uWSGI can manage multiple namespaces (named "mountpoints"), so HTTP requests for /foo (and below) could map to /var/www while requests for /bar (and below) could map to /var/site/test and so on.

To build the plugin you can simply run:

```sh
uwsgi --build-plugin https://github.com/unbit/spockfs
```

or if you want a single binary with the plugin embedded (ensure to have a c compiler, python and libpcre):

```sh
curl https://uwsgi.it/install.sh | UWSGI_EMBED_PLUGINS="spockfs=https://github.com/unbit/spockfs" bash -s nolang /tmp/spockfs-server
```

(change /tmp/spockfs-server to whatever you want, this will be the name of the server)

Plugin options
==============

The following options are exposed by the plugin

* --spockfs-mount <mountpoint>=<path> (mount <path> under <mountpoint>)
* --spockfs-ro-mount <mountpoint>=<path> (mount <path> under <mountpoint> in readonly mode)
* --spockfs-xattr-limit <size> (set the maximum size of xattr values, default 64k)


Serving directories
===================

We use the .ini configuration format (you are free to use whatever uWSGI supports), save it as spockfs.ini

```ini
[uwsgi]
; load the spockfs plugin in slot 0, useless if you have built it into the binary
plugin = 0:spockfs

; bind to tcp port 9090
http-socket = :9090

; spawn 8 threads
threads = 8

; mount /var/www as /
spockfs-mount = /=var/www

; mount /var/spool as /spool
spockfs-mount = /spool=/var/spool

; mount /opt as /opt in readonly mode
spockfs-ro-mount = /opt=/opt

; ensure the mountpoints are correctly parsed as SCRIPT_NAME (this is required only if you have multiple spockfs mountpoints)
manage-script-name = true
```

run the server:

```sh
uwsgi spockfs.ini
```

or if you have used the installer:

```sh
/tmp/spockfs-server spockfs.ini
```

Ensure to not run the server as root, eventually you can drop privileges with the uid and gid options:

```ini
[uwsgi]
; load the spockfs plugin in slot 0, useless if you have built it into the binary
plugin = 0:spockfs

; bind to tcp port 9090
http-socket = :9090

; spawn 8 threads
threads = 8

; mount /var/www as /
spockfs-mount = /=var/www

; mount /var/spool as /spool
spockfs-mount = /spool=/var/spool

; mount /opt as /opt in readonly mode
spockfs-ro-mount = /opt=/opt

; ensure the mountpoints are correctly parsed as SCRIPT_NAME (this is required only if you have multiple spockfs mountpoints)
manage-script-name = true

;drop privileges
uid = www-data
gid = www-daya
```

Concurrency
===========

Multiprocess and multithreaded modes are highly suggested (instead of async).

In this example we spawn 2 processes with 2 threads each, all governed by a master (consider it as an embedded monitor that will automatically respawn dead processes and will monitor the server status)

```ini
[uwsgi]
; load the spockfs plugin in slot 0, useless if you have built it into the binary
plugin = 0:spockfs

; bind to tcp port 9090
http-socket = :9090

; run the master process
master = true
; spawn 2 processes
processes = 2
; spawn 2 threads for each process
threads = 8

; mount /var/www as /
spockfs-mount = /=var/www

; mount /var/spool as /spool
spockfs-mount = /spool=/var/spool

; mount /opt as /opt in readonly mode

spockfs-ro-mount = /opt=/opt

;drop privileges
uid = www-data
gid = www-data
```

uWSGI gives you tons of metrics and monitoring tools. Consider enabling the stats server (to use tools like uwsgitop)


```ini
[uwsgi]
; load the spockfs plugin in slot 0, useless if you have built it into the binary
plugin = 0:spockfs

; bind to tcp port 9090
http-socket = :9090

; run the master process
master = true
; spawn 2 processes
processes = 2
; spawn 2 threads for each process
threads = 8

; mount /var/www as /
spockfs-mount = /=var/www

; mount /var/spool as /spool
spockfs-mount = /spool=/var/spool

; mount /opt as /opt in readonly mode

spockfs-ro-mount = /opt=/opt

;drop privileges
uid = www-data
gid = www-data

; bind the stats server on 127.0.0.1:9091
stats = 127.0.0.1:9091
```

telnet/nc to 127.0.0.1:9091 will give you a lot of infos (in json format). The https://github.com/unbit/uwsgitop tool will give you a top-like interface for the stats.

HTTPS
=====

If you want to place your instance behind HTTPS, use https-socket instead of http-socket passing it the certificate and the key:

```ini
[uwsgi]
; load the spockfs plugin in slot 0, useless if you have built it into the binary
plugin = 0:spockfs

; bind to tcp port 9090 with ssl
https-socket = :9090,path_to_cert,path_to_key

; run the master process
master = true
; spawn 2 processes
processes = 2
; spawn 2 threads for each process
threads = 8

; mount /var/www as /
spockfs-mount = /=var/www

; mount /var/spool as /spool
spockfs-mount = /spool=/var/spool

; mount /opt as /opt in readonly mode

spockfs-ro-mount = /opt=/opt

;drop privileges
uid = www-data
gid = www-data

; bind the stats server on 127.0.0.1:9091
stats = 127.0.0.1:9091
```

Placing behind nginx
====================

If you plan to use the SpockFS server in a LAN, using --http-socket uWSGI option will be more than enough (and you will get really good performance). Instead, when wanting to expose it over internet you should proxy it behind a full-featured webserver. uWSGI has its HTTP proxy embedded (the so called 'http-router'), but (as its name implies) it is only a 'router' without any kind of logic (expect for mapping domain names to specific backends and for doing load balancing).

Nginx instead, has tons of advanced features that makes it a perfect candidate for the job. In addition to this it supports out of the box the 'uwsgi protocol' (that is a faster way to pass informations between the proxy and the backend).

To configure a location in nginx to be forwarded to uWSGI'spockfs plugin:

```
location /spool {
    include uwsgi_params;
    uwsgi_modifier1 179;
    uwsgi_pass 127.0.0.1:3031;
}
```

this will instruct nginx to pass all of the requests for /spool to the 127.0.0.1:3031 address using the uwsgi protocol and asking (via the modifier 179 directive) for the SpockFS plugin of the uWSGI instance.

The uWSGI config will be a bit different this time

```ini
[uwsgi]
; load the spockfs plugin in its default slot (179) as nginx will directly ask for it
plugin = spockfs

; bind to uwsgi tcp port 3031
socket = 127.0.0.1:3031

; run the master process
master = true
; spawn 2 processes
processes = 2
; spawn 2 threads for each process
threads = 8

; mount /var/www as /
spockfs-mount = /=var/www

; mount /var/spool as /spool
spockfs-mount = /spool=/var/spool

; mount /opt as /opt in readonly mode

spockfs-ro-mount = /opt=/opt

;drop privileges
uid = www-data
gid = www-data

; bind the stats server on 127.0.0.1:9091
stats = 127.0.0.1:9091
```

the only differences are the use of 'socket' instead of 'http-socket' and the load of the spockfs plugin (if needed) in its default slot (instead of forcing it to 0).

Having nginx on front as a proxy will have minimal impact (between 1% and 3%). Using unix sockets instead of TCP will reduce this impact a bit (at the cost of a bit more attention when setting permissions, as unix sockets require write permissions to be accessed)

```
location /spool {
    include uwsgi_params;
    uwsgi_modifier1 179;
    uwsgi_pass unix:/var/run/spockfs.socket;
}
```

```ini
[uwsgi]
; load the spockfs plugin in its default slot (179) as nginx will directly ask for it
plugin = spockfs

; bind to uwsgi unix socket /var/run/spockfs.socket
socket = /var/run/spockfs.socket
; ensure the socket is writable by nginx
chmod-socket = 666

; run the master process
master = true
; spawn 2 processes
processes = 2
; spawn 2 threads for each process
threads = 8

; mount /var/www as /
spockfs-mount = /=var/www

; mount /var/spool as /spool
spockfs-mount = /spool=/var/spool

; mount /opt as /opt in readonly mode

spockfs-ro-mount = /opt=/opt

;drop privileges
uid = www-data
gid = www-data

; bind the stats server on 127.0.0.1:9091
stats = 127.0.0.1:9091
```

Playing with internal routing
=============================

Internal routing (http://uwsgi-docs.readthedocs.org/en/latest/InternalRouting.html) is one of the most advanced uWSGI features, consider it as a meta-language for changing the request/response cycle at runtime. For apache users, see it as a more powerful mod_rewrite.

You can use it for enforcing special rules for every SpockFS request. As an example you may want to reject any request from hosts starting with address 10.*:

```ini
[uwsgi]
plugin = spockfs
http-socket = :9090
spockfs-mount = /=/var/www

route-remote-addr = ^/10\. break:403 Forbidden
```

or you can change the resource requested to implement aliasing

```ini
[uwsgi]
plugin = spockfs
http-socket = :9090
spockfs-mount = /=/var/www

route = ^/foobar$ setpathinfo:/thetrueone
```

or enforce specific behaviours to authenticated users

```ini
[uwsgi]
plugin = spockfs
http-socket = :9090
spockfs-mount = /=/var/www

; always returns 404 to the kratos user
route-remote-user = ^kratos$ break:404 Not Found
```

Internal routing allows basic authentication too, the following one is a simple hardcoded session

```ini
[uwsgi]
; load the spockfs plugin (if needed by your build)
plugin = spockfs
; again, useless in monolithic builds, or those installed via the network installer
plugin = router_basicauth

http-socket = :9090
spockfs-mount = /=/var/www

; route-run will force a route to always run without conditions
route-run = basicauth:SpockFS realm,kratos:deimos
```

another example using radius (requires uWSGI radius plugin)

```ini
[uwsgi]
; load the spockfs plugin (if needed by your build)
plugin = spockfs
; the radius plugin is generally not built in official uWSGI installers, so you need to load it
plugin = router_radius

http-socket = :9090
spockfs-mount = /=/var/www

; route-run will force a route to always run without conditions
route-run = radius:realm=SpockFS,server=192.168.173.17:1703,secret=unknown
```

or ldap (requires uWSGI ldap plugin)

```ini
[uwsgi]
; load the spockfs plugin (if needed by your build)
plugin = spockfs
; the radius plugin is generally not built in official uWSGI installers, so you need to load it
plugin = router_radius

http-socket = :9090
spockfs-mount = /=/var/www

; route-run will force a route to always run without conditions
route-run = ldapauth:url=ldaps://192.168.173.22:3030/,binddn=cn=foobar,bindpw=unknown
```

Remember that you can use gotos too:

```ini
[uwsgi]
plugin = spockfs
http-socket = :9090
spockfs-mount = /=/var/www

; when the user is kratos, jump to 'kratosrules'
route-remote-user = ^kratos$ goto:kratosrules
; ..otherwise interrupt the chain and continue normally
route-run = last:

; the kratos rules
route-label = kratosrules
; if kratos asks for /medusa, reject it
route = ^/medusa$ break:403 Forbidden
; if kratos asks for /null, returns not found
route = ^/null$ break:404 Not Found
; if kratos asks for /fake, rewrite it to /true
route = ^/fake$ setpathinfo:/true
; if kratos is WebDAV addicted, and uses MKCOL instead of MKDIR, rewrite it
route-if = equal:${REQUEST_METHOD};MKCOL setmethod:MKDIR
```

The possibilities are unlimited, have fun
