After installing the vagrant setup from coreos, I needed to remotely manage fleetctl from my desktop/laptop/wherever I edit my source code.

Fleetctl can run over an ssh tunnel, so all I needed was to be able to run the fleetctl client, and ssh from an environment that had access to all my source code. I used a vagrant box for this.

The process was fairly straight forward.

First, bring up the coreos vagrant cluster

vagrant up

Then, bring up the vagrant box for the fleetctl remote client and ssh into it.

In the vagrant fleetctl client client box, install the vagrant private key:

cd ~/.ssh
wget https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant
chmod 0600 vagrant

Get ssh-agent running, and add the vagrant key

ssh-agent /bin/bash
ssh-add ~/.ssh/vagrant

Next, download fleetctl. Make sure you are grabbing the same version as is running on the cluster you want to manage

cd ~
wget https://github.com/coreos/fleet/releases/download/v0.9.2/fleet-v0.9.2-linux-amd64.tar.gz
tar -zxvf fleet-v0.9.0-linux-amd64.tar.gz
sudo ln -s ~/fleet-v0.9.0-linux-amd64/fleetctl /usr/local/bin/fleetctl

Finally, fleetctl is ready to use. You will need to retrieve the IP address of a machine in the cluster in order to run the following. Easiest way to do this is to use vagrant ssh against one of the boxes in the coreos cluster

vagrant ssh core-01 -c "ip address show eth1 | grep 'inet' "

Once you have the IP you're good to go. Back on the fleetctl remote client run

fleetctl --tunnel 172.17.8.101 list-machines

If you are going to run a few commands, then set the FLEETCTL_TUNNEL environment variable

export FLEETCTL_TUNNEL=172.17.8.101
fleetctl list-units

From https://coreos.com/docs/launching-containers/launching/launching-containers-fleet/ 

# A Hello World Service

```
fleetctl start hello-world.service 
fleetctl list-units
fleetctl journal hello-world.service
fleetctl stop hello-world.service 
fleetctl destroy hello-world.service 
```

# A high availability service

Start the apache web service

```
fleetctl submit apache@.service
fleetctl start apache@7777
fleetctl start apache@8888
```

Start the apache discovery sidekick

```
fleetctl submit apache-discovery@.service
fleetctl start apache-discovery@7777
fleetctl start apache-discovery@8888
fleetctl list-units
```

On core-01 verify etcd values are set

```
etcdctl ls /services/ --recursive
etcdctl get /services/apache/172.17.8.101
```

Start up the nginx load balancer

```
fleetctl start nginx_lb
```


# Creating the nginx reverse proxy

docker run -i -t ubuntu:14.04 /bin/bash
apt-get update
apt-get install nginx curl -y
cd /usr/local/bin
curl -L https://github.com/kelseyhightower/confd/releases/download/v0.8.0/confd-0.8.0-linux-amd64 -o confd
chmod +x confd
mkdir -p /etc/confd/{conf.d,templates}
vi /etc/confd/conf.d/nginx.toml

----
---- /etc/confd/conf.d/nginx.toml
----
[template]

# The name of the template that will be used to render the application's configuration file
# Confd will look in `/etc/conf.d/templates` for these files by default
src = "nginx.tmpl"

# The location to place the rendered configuration file
dest = "/etc/nginx/sites-enabled/app.conf"

# The etcd keys or directory to watch.  This is where the information to fill in
# the template will come from.
keys = [ "/services/apache" ]

# File ownership and mode information
owner = "root"
mode = "0644"

# These are the commands that will be used to check whether the rendered config is
# valid and to reload the actual service once the new config is in place
check_cmd = "/usr/sbin/nginx -t"
reload_cmd = "/usr/sbin/service nginx reload"
----

vi /etc/confd/templates/nginx.tmpl

----
---- /etc/confd/templates/nginx.tmpl
----
upstream apache_pool {
{{ range getvs "/services/apache/*" }}
    server {{ . }};
{{ end }}
}

server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    access_log /var/log/nginx/access.log upstreamlog;

    location / {
        proxy_pass http://apache_pool;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
----

rm /etc/nginx/sites-enabled/default
vi /etc/nginx/nginx.conf

. . .
http {
    ##
    # Basic Settings
    ##
    log_format upstreamlog '[$time_local] $remote_addr passed to: $upstream_addr: $request Upstream Response Time: $upstream_response_time Request time: $request_time';

    sendfile on;
    . . .

vi /usr/local/bin/confd-watch

----
---- /usr/local/bin/confd-watch
----
#!/bin/bash

set -eo pipefail

export ETCD_PORT=${ETCD_PORT:-4001}
export HOST_IP=${HOST_IP:-172.17.42.1}
export ETCD=$HOST_IP:$ETCD_PORT

echo "[nginx] booting container. ETCD: $ETCD."

# Try to make initial configuration every 5 seconds until successful
until confd -onetime -node $ETCD -config-file /etc/confd/conf.d/nginx.toml; do
    echo "[nginx] waiting for confd to create initial nginx configuration."
    sleep 5
done

# Put a continual polling `confd` process into the background to watch
# for changes every 10 seconds
confd -interval 10 -node $ETCD -config-file /etc/confd/conf.d/nginx.toml &
echo "[nginx] confd is now monitoring etcd for changes..."

# Start the Nginx service using the generated config
echo "[nginx] starting nginx service..."
service nginx start

# Follow the logs to allow the script to continue running
tail -f /var/log/nginx/*.log
----

chmod +x /usr/local/bin/confd-watch

exit

docker commit de4f30617499 user_name/nginx_lb

docker login

docker push user_name/nginx_lb

