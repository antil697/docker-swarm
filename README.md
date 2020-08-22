# My Docker Swarm
Documenting the current design. Work in progress!
Detailed design blueprints for each component to be completed in due time. 

# High Level Architecture
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/docker_swarm.png" />

# Network Diagram
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/network_diagram.png" /><br/>

# Design Considerations
Why 3 manger nodes and 1 worker?<br/>
In order to achieve fault taulerance, I need 3 managers. The Pi3B+ is not as powerful as my 4s and is used with a touchscreen as control panel. I allow it to run some ligth loads.<br/>
Most of my services dirtibuted across the nodes do not put any strain on the system. While I have a lot of capacity left (for now), I achieved High Availability and Fault Tolerance.<br/>

<h2>Management Stack</h2>
<h3>Ingress Layer / Docker</h3>
<h4>Keepalived</h4> 
Running Keepalived as single container on each node provides the benefit of one single virtual IP for the cluster. 
While Docker Swarm already provides load balancing and connects to the right service, the IP is independent from the node. This serves as ingress point to the cluster.

<h4>Zigbee2MQTT</h4>
This provides the bridge of my Ikea and other Zigbee devices to the message bus for further processing and automation. It runs oof a CC2531 controller attached to Node1. I need to order a backup CC2531 but had not much luck lately as my orders got cancelled. For now, since I Docker Swarm does not support device mounts, I am stuck with this beeing at the physical node. There is a workaround but I have had no time to implement it yet. For now it has to stay on the node where teh CC2531 usb stick resides.<br/>
However, here are some ideas to investigate:<br/>
In order to have redundant CC2531 coordinators (when only one is allowed on the network), I need to investigate how to ensure that the controller is not active when powered by the USB port and only activates when the service is running on the node. If this can't be done by software, an possible solution is to create a small adapter that controls power via a GPIO pin on the node and use NodeRed to activate the right coordinator. Quite simple to design. 


<h3>Docker Swarm - Management Layer</h3>
<h4>Traefik</h4> 
I used NGINX for this in the past. Switching to Traefik as it is build for cloud applications. It allows me to add SSL to container HTTP traffic without the need to create a seperate certificate for each container. In addition to securing the transmission, I can also control access for services that do not have access control from a central system. Since I am exposing some ports to the internet through my firewall, I want to secure these services.

<img src="https://github.com/antil697/docker-swarm/blob/master/Images/traefik.png" />

<h4>Portainer</h4>
Portainer is similar to Swarmpit. Some things works better with Portainer than SwarmPit so I run both.

<img src="https://github.com/antil697/docker-swarm/blob/master/Images/portainer.png" />
<h4>Swarmpit</h4>
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/Swarmpit.png" />
<h4>InfluxDB</h4>
Part of the swarmpit stack and also used by NodeRed and Telegraf to store metrics. 
<h4>Grafana</h4>
The visualisation component. Makes data come to live. When combined with Kapacitor, it allows for alerting. Since NodeRed can access the data as I am sending most of it via the mqtt message bus, I can use NodeRed to create alerts.<br/>
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/grafana.png" />

<h3>Docker Swarm - Services Layer</h3>
<h4>NodeRed</h4>
The core of my system. After starting with OpenHab and HomeAssistant, I found that the combination of NodeRed with MQTT and Zigbee2MQTT is a clear winner. HomeAssistant is only used for its floorplan addon that delivers the SVG dashboard to display on the touchscreen of Node4.
Every logic operates within NodeRed. While my core lights are Phillips Hue based and I could in theory move Hue to Zigbee2MQTT, I have not done this so far. The logic was to keep the system independant so the basic light functions work should my Raspberry Pi fail. NodeRed would only enhance the Hue hub. But with this new HA design, I can move the devices over to my Zigbee coordinator. 
The flow shown below conects to Kodi and checks if a movie is playing. In case that the playtime is more than 10 minutes and lights are on, the flow send instructions to the message bus to dimm lights and enable the scene defined to watch movies. 
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/nodered.png" />

<h4>Mosquitto</h4>
Part of the key system. Mosquitto provides the message bus for all automation actions and status updates. While NodeRed is the brain, Mosquitto is the nervous systems that reports sensor states and provides commands for actions to actors. 
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/mqtt.png" />

<h4>Home Assistant</h4>
The only function that HomeAssiant provides is the dashboard to control and monitor basic functions from a central system. 
It uses the <a href="https://github.com/pkozul/ha-floorplan">floorplan</a> addon. Node3 is attached to a the offical Raspberry Pi touchscreen and runs in kiosk mode. It also provides services for less intensive workloads as backup.

<h4>Unifi Controller</h4>
The controller for my USG firewall.

<h4>PiHole</h4>
PiHole provides DHCP to my network. I could use the Unifi UGS but when it comes to troubleshoot issues with block lists, it is prefered that I have the query listed per client. I generally avoid going overboard with blocklists. I worked out the right balance for me. Bye bye ads and trackers. 
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/pihole.png" />

<h3>NFS File Storage</h3>
Using my Qnap to provide NFS file storage for containers. Any container moving to an other node will mount its assigned NFS volume. This allows a container such as PiHole to change a node without loosing track of who it assigned an IP address to. While there are smarter cloud storage options around, this will do for now. 
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/nfs.png" width=500/>

<h2>System Monitoring</h2>
<h4>Telegraf</h4>
Running Telegraf in each node to collect system data and report via MQTT and store in InfluxDB. While I could create a container and run it on each node, it won't be able to run [exec] plugins to collect CPU temperature.
Telegraf will also make data available to MQTT in JSON format for futher processing in NodeRed if required. 




# Dashboard

<img src="https://github.com/antil697/docker-swarm/blob/master/Images/Dashboard.png" />
