<!--
.. title: building packages with poudriere
.. slug: building-packages-with-poudriere
.. date: 2018-03-27 23:03:28 UTC-07:00
.. tags: freebsd, poudriere
.. category: 
.. link: 
.. description: 
.. type: text
-->

Poudriere is a simple way to create FreeBSD binary packages. https://pkg.freebsd.org does this, and those packages are sufficient for most use cases. Building your own packages has some advantages:
<!-- TEASER_END -->
<ul>
 	<li>Work on ports, and have a quick method for testing them</li>
 	<li>Compiling ports at a different frequency than upstream</li>
 	<li>Compiling custom port options that upstream will not take</li>
 	<li>Compiling restricted ports</li>
 	<li>Compiling custom ports trees</li>
</ul>

Please keep in mind, the version of FreeBSD you want to run poudriere on must match or be greater than the target systems you are building packages for. This is because the jail binaries that are being executed must be compatible with the kernels ABI. I'm building packages for FreeBSD-10.3-RELEASE and FreeBSD-11.0-RELEASE in this example.

<h2>Poudriere installation</h2>
If you want a more bleeding edge version of poudriere, with newer features try <code>ports-mgmt/poudriere-devel</code>. Otherwise <code>ports-mgmt/poudriere</code>.

<h3>Via ports</h3>
```cd /usr/ports/ports-mgmt/poudriere[-devel]
make install```

<h3>Via pkg</h3>
```pkg install poudriere[-devel]```

<h2>Configuration</h2>
<h3>poudriere.conf</h3>
You should go through <code>/usr/local/etc/poudriere.conf</code> and make sure everything in there matches your use case, and suits you. I changed the following options:
```
ZPOOL=zroot
PKG_REPO_SIGNING_KEY=/usr/local/poudriere/keys/pkg.key
```
If you want to do signing, leave the PKG_REPO_SIGNING_KEY parameter in the configuration and hit the next section. Otherwise, skip the next section.

<h3>Signing</h3>
Signing is easy. There isn't any reason you shouldn't do it!
```
mkdir -p /usr/local/poudriere/keys
chmod 600 /usr/local/poudriere/keys
openssl genrsa -out /usr/local/poudriere/keys/pkg.key 8192
openssl rsa -in /usr/local/poudriere/keys/pkg.key -pubout > /usr/local/poudriere/keys/pkg.cert
```

<h3>Create distfile directory</h3>
This is referenced in the <code>poudriere.conf</code> file.
```mkdir -p /usr/ports/distfiles```

<h2>Prepare jails</h2>
Create a jail for each version of FreeBSD you want to build packages for:
```
poudriere jail -c -j FreeBSD:10:amd64 -v 11.0-RELEASE -a amd64
poudriere jail -c -j FreeBSD:11:amd64 -v 10.3-RELEASE -a amd64
```
<h2>Prepare ports</h2>
```
poudriere ports -c -p upstream
```

<h2>Building</h2>
I wrote this all into a script:
```
PORTS_TREES='upstream'
JAILS='103-RELEASE 110-RELEASE'
PYTHON_VERSION='3.6'
for PORTS_TREE in ${PORTS_TREES}; do

        # Update poudriere ports tree
        poudriere ports -u -p ${PORTS_TREE}

        for JAIL in $JAILS; do
                MAKEFILE="/usr/local/etc/poudriere.d/${JAIL}-make.conf"
                # Ensure the jail is up to date
                poudriere jail -u -j ${JAIL}

                # Build the ports
                echo "DEFAULT_VERSIONS+= python=${PYTHON_VERSION}" > ${MAKEFILE}
                poudriere bulk -f /usr/local/poudriere/pkg-list.txt -j ${JAIL} -p ${PORTS_TREE}
                rm -f ${MAKEFILE}
        done

done
```

Just run the script:
```

```

<h2>Serving built packages</h2>
To serve built packages, I'm going to use nginx.
```
pkg install -y nginx
```

Simple nginx configuration. Just points the location blocks at the packages and logs.
<code>/usr/local/etc/nginx/nginx.conf</code>
```
load_module /usr/local/libexec/nginx/ngx_mail_module.so;
load_module /usr/local/libexec/nginx/ngx_stream_module.so;

worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  localhost;
        location /logs {
            alias   /usr/local/poudriere/data/logs/bulk;
            autoindex on;
        }
        location / {
            root   /usr/local/poudriere/data/packages/;
            index  index.html index.htm;
            autoindex on;
        }
    }
}
```

Enable it at boot time:
```
sysrc nginx_enable="YES"
```
Start it on the running system:
```
service nginx start
```

You should be able to see your packages by going to your poudriere server in your web browser. If you want to see the logs, simply apend /logs to the end of your URL.

<h2>Configuration on target system</h2>
I suggest writing the pkg.conf and placing it in the webroot of your nginx server:
<code>/usr/local/poudriere/data/packages/pkg.douglas-enterprises.com.conf</code>
```
# Douglas Enterprises repo configuration 
pkg.douglas-enterprises.com: {
  url: "pkg+http://pkg.douglas-enterprises.com/${ABI}-upstream",
  mirror_type: "srv",
  signature_type: "pubkey",
  pubkey: "/usr/local/etc/pkg/certs/pkg.douglas-enterprises.com.cert",
  enabled: yes
}
```
I also suggest copying the cert to the webroot for easy fetching:
```
cp /usr/local/poudriere/keys/pkg.cert /usr/local/poudriere/data/packages/pkg.douglas-enterprises.com.cert
```

Now installation of this repo is super easy on the target system:
```
mkdir -p /usr/local/etc/pkg/certs /usr/local/etc/pkg/repos
fetch -o /usr/local/etc/pkg/certs http://pkg.douglas-enterprises.com/pkg.douglas-enterprises.com.cert
fetch -o /usr/local/etc/pkg/repos http://pkg.douglas-enterprises.com/pkg.douglas-enterprises.com.conf
pkg update -f
```

<h2>Automation</h2>
Whenever I want to build, I simply run that script. It keeps everything up to date, and is perfect to get added to cron:
```
0 2 * * * /usr/local/poudriere/run_poudriere.sh
```

I also checked a few files into a github repo https://github.com/elstevi/poudriere-config and set up an auto deploy mechanism with cron. This allows me to change what packages poudriere is building from anywhere without having to ssh into the machine.
