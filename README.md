# My Docker Swarm
Documenting the current design 
# High Level Architecture
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/docker_swarm.png" />


<h2>Management Stack</h2>
<h3>Ingress Layer</h3>
<h4>Keepalived</h4> 
Running Keepalived as single container on each node provides the benefit of one single virtual IP for the cluster. 
While Docker Swarm already provides load balancing and connects to the right service, the IP is independent from the node. This serves as ingress point to the cluster.

<h3>Docker Swarm - Management Layer</h3>
<h4>Traefik</h4> 
I used NGINX for this in the past. Switching to Traefik as it is build for cloud applications. It allows me to add SSL to container HTTP traffic without the need to create a sperate certificate for each container. 

<h4>Portainer</h4>

<h4>Swarmpit</h4>

<h4>InfluxDB</h4>

<h4>Grafana</h4>

<h3>Docker Swarm - Services Layer</h3>
<h4>NodeRed</h4>

<h4>Mosquitto</h4>

<h4>Zigbee2MQTT</h4>
This provides the bridge of my Ikea and other Zigbee devices to the message bus for further processing and automation. It runs oof a CC2531 controller attached to Node4. I need to order a backup CC2531 but had not much luck lately as my orders got cancelled. 
The idea is that in case of Node4 failing Node2 will take over with a stick attached to the USB port. I need to investigate how I ensure that the controller is not active when powered by the USB port and only activates when the service is running on the node. If this cant be done by software, an possible solution is to create a small adapter that controls power via a GPIO pin on the node. Quite simple to design. 

<h4>Home Assistant</h4>

<h4>Unifi Controller</h4>

<h4>PiHole</h4>

<h2>System Monitoring</h2>
<h4>Telegraf</h4>
Running Telegraf in each node to collect system data and report via MQTT and store in InfluxDB. While I could create a container and run it on each node, it won't be able to run [exec] plugins to collect CPU temperature.
Telegraf will also make data available to MQTT in JSON format for futher processing in NodeRed if required. 

<img scr="https://github.com/antil697/docker-swarm/blob/master/Images/grafana.png">


# Dashboard

<img src="https://github.com/antil697/docker-swarm/blob/master/Images/Dashboard.png" />
