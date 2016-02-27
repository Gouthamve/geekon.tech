+++
date = "2016-02-27T21:13:14+05:30"
draft = false
title = "Monitoring GeekOn using InfluxDB, Telegraf and Grafana on CoreOS"

banner = "/img/posts/five/cover.png"

slug = "monitoring-geekon-influxdb-telegraf-grafana-coreos"
socialsharing = true

author = "Goutham"
authortwitter = "https://twitter.com/putadent"

categories = ["devops", "influxdb", "docker"]
+++

Who doesn't like graphs?

# Prologue

I was deploying two static sites behind [Caddy](https://caddyserver.com), a [ShoutIRC server](https://irc.geekon.tech) with docker and couple of other services on a 512MB droplet. And I wanted to see if I could add a couple more services in without loading the server. Also I wanted see the load on the server over time.

So, I decided to setup a monitoring solution for GeekOn. I chose to use [influxdb](https://influxdata.com/time-series-platform/influxdb/) for data storage and [telegraf](https://influxdata.com/time-series-platform/telegraf/) for collecting the metrics and [Grafana](http://grafana.org) for visualisation. There are really amazing set of tutorials on using Influx, Grafana and Telegraf [on](http://docs.grafana.org/datasources/influxdb/) [the](https://influxdata.com/get-started/sending-data-to-influxdb-with-telegraf/) [internet](http://milinda.svbtle.com/cluster-and-service-monitoring-using-grafana-influxdb-and-collecd).

Instead, I am presenting how I deployed the stack on coreOS behind DigitalOceans private network!

# Deployment

## How it works

GeekOn is currently running on a 512MB droplet (Looking for sponsors!) and I didn't want to run influxDB on the same server. Which meant if I had to put influx on another server, then there would be lots of traffic between those servers. But DigitalOcean has an amazing feature called Private Networking which allows you to have droplet-to-droplet links within the same datacenter. I am going to describe the architecture and the droplet which hosts this blog will be called `geekon` and the droplet with influxdb will be called `influx-geekon`.

`Influx-geekon` is another 512MB droplet running a beta version of coreOS. `telegraf` will be running on `geekon` and will be forwarding metrics to `influx-geekon` via the private interface. On `influx-geekon`, influxdb and grafana will be running inside docker. Influxdb will be exposed on the private interface while `grafana` container will be linked to a `caddy` container which is exposed to the world. Caddy will be forwarding requests from me to grafana, while grafana will be accessing influx using the private interface IP. The architecture is illustrated below:

<img src="/img/posts/five/influx-arch.jpg" height="400px"></img>

## Grafana & Caddy

I deployed grafana using the [official docker image](https://github.com/grafana/grafana-docker) with the changed admin user/password. However, I couldn't get the username password to change, but I changed it from the admin page.
```
docker run -d --name grafana \
    -v /var/lib/grafana:/var/lib/grafana \
    -e "GF_USERS_ALLOW_SIGN_UP=false" \
    -e "GF_USERS_ALLOW_ORG_CREATE=false" \
    -e "GF_SECURITY_ADMIN_USER=user" \
    -e "GF_SECURITY_ADMIN_PASSWORD=password" \
    grafana/grafana:develop
```

  If you observed, I haven't forwarded any ports! I have Caddy running in another container but with ports `80` and `443` both forwarded to host. Now I have used a [docker link](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/) to connect grafana to Caddy. And the best part about Caddy is that the Caddyfile can be configured using environment variables (Nope, thats not the best part)! Heres the Caddyfile I used (tls email for [let's encrypt](https://letsencrypt.org)):

```
monitor.geekon.tech {
	proxy / {$GRAFANA_PORT_3000_TCP_ADDR}:{$GRAFANA_PORT_3000_TCP_PORT}
	tls <email>
}
```

Now, I have `monitor.geekon.tech` forwarding to grafana.

## InfluxDB

I then deployed influx with ports forwarded to the private network interface. I used this [config.toml](https://gist.github.com/Gouthamve/c4101c36b142ae9544ff) which was better for single node deployments.
```
docker run -d -p <privateIP>:8086:8086 \
--name=influxdb \
-v /home/core/config.toml:/config/config.toml \
--volume=/var/influxdb:/data \
-e ADMIN_USER="user"  \
-e INFLUXDB_INIT_PWD="pass" \
-e PRE_CREATE_DB="telegraf" \
tutum/influxdb:0.10
```

While having authentication doesn't seem to be necessary as the interface is private, its a shared private network, meaning any droplet in the same DataCenter can access it. Now, due to the way the tutum/influxdb works, we can specify the initial admin user/pass using an environment variable. But creating the initial admin can only be done while `auth-enabled = false`. 

So to add authentication, we need to start the container with the given env-vars and then change the config.toml to reflect `auth-enabled = true` and do `docker restart influxdb`. After this is done, access grafana and add influx as a datasource where you choose

```
influxDB 0.9.x
URL: http://PrivateIP:8086
Access: Proxy

Database: telegraf
User: user
Password: password
```

Now that the datasource was added successfully, I moved on to `telegraf`.

## Telegraf

While Caddy, Grafana and influx are deployed on `influx-geekon`, telegraf is the metric collection daemon and it needs to be deployed on `geekon`.

Installation is very straight forward. Just download [this tarball](http://get.influxdb.org/telegraf/telegraf-0.10.4.1-1_linux_amd64.tar.gz) and run ```sudo tar -C / -zxvf ./telegraf-0.10.4.1-1_linux_amd64.tar.gz``` Now I am collecting a lot of metrics so that I better understand `influx`. 
I was collecting the following and heres my [config file](https://gist.github.com/Gouthamve/88968ab63ca04a3008db).

  1. cpu
  2. mem
  3. netstat
  4. disk
  5. docker

These export a ton of fields and I have no clue what to do with them, but I plan to add some more and setup some downsampling and cleanup of data. That will be explained in a forthcoming post.

# Epilogue 

Now I got back to grafana, followed [this tutorial](https://www.youtube.com/watch?v=sKNZMtoSHN4) to create a dashboard. Successfully made a dashboard to have CPU metrics, % memory free, memory used by docker, Open TCP Connections and Memory usage of caddy. Ran a stress test and it worked pretty well!

TODO:

  1. Add more metrics
  2. Understand and blog about how to make sense of metrics
  3. Understand and blog about advanced InfluxQL queries
  4. Setup Chronograf and Kapacitor for alerts!

[Complete dashboard](https://snapshot.raintank.io/dashboard/snapshot/E4LEt8sxwdhQkzrN8qk1rx7N9vQIYotj)

<iframe src="https://snapshot.raintank.io/dashboard/snapshot/E4LEt8sxwdhQkzrN8qk1rx7N9vQIYotj" width="100%" height="800px"></iframe>

<script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>

<script type='text/javascript'>$(document).ready(function(){$("a[href^='http://']").each(function(){-1==this.href.indexOf(location.hostname)&&$(this).attr("target","_blank")}),$("a[href^='https://']").each(function(){-1==this.href.indexOf(location.hostname)&&$(this).attr("target","_blank")})});</script>