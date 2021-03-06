# Home Setup
Documenting the current design. Work in progress!<br/>
Detailed design blueprints for each component to be completed in due time.

# High Level Architecture
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/docker_swarm.png" />

# Network Diagram
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/network_diagram.png" /><br/>
Docker Macvlan provides PiHole with a host based network where I can guarantee the IP address. I create a Macvlan exlusivly for PiHole and on any node it will retain the same IP. That ensures my network devices can always reach out to the same IP and speak to a DNS server. While Keepalived provides also a single IP, I still have to work out how to run PiHole in a normal container. It needs to use port 80 and 443 which also is required by Traefik. So, Macvlan is the best option to allow these two to co-exist.

# Design Considerations
Why 3 manger nodes?<br/>
In order to achieve fault taulerance, I need 3 managers.
Most of my services distributed across the nodes do not put any strain on the system. While I have a lot of capacity left (for now), I achieved High Availability and Fault Tolerance.<br/>
At this point in time I am running 3 Nodes on Raspberry Pi 4B+ 4GB.</br>
14 Stacks defining 27 services runing on 41 containers with each node using on average 20% CPU and less then 30% memory. 


<h2>Cloudflare</h2>
I am using Cloudflare as DNS solution. It allows me to protect my services, e.g. limiting access to my services only to the country I reside. My firewall only accepts incoming connections for my services from Cloudflare.<br/>
<h2>Management Stack</h2>
<h3>Ingress Layer / Docker</h3>
<h4>Keepalived</h4> 
Running Keepalived as single container on each node provides the benefit of one single virtual IP for the cluster. 
While Docker Swarm already provides load balancing and connects to the right service, the IP is independent from the node. This serves as ingress point to the cluster.

<h4>Zigbee2MQTT</h4>
This provides the bridge of my Ikea and other Zigbee devices to the message bus for further processing and automation. It runs oof a CC2531 controller attached to Node1. I need to order a backup CC2531 but had not much luck lately as my orders got cancelled. For now, since I Docker Swarm does not support device mounts, I am stuck with this beeing at the physical node. I am running the CC2531 coordinator on a Pi Zero W and use ser2net to connect it to the container.<br/>
However, here are some ideas to investigate:<br/>
In order to have redundant CC2531 coordinators (when only one is allowed on the network), it seems that using <a href="https://github.com/mvp/uhubctl">uhubctl</a> would allow the control of power to USB ports. By using NodeRed to monitor the message bus where the zigbee2mqtt container is running it can turn on/off the respective USB port and activate the device. I need to investigate the use of smart routing with Traefik to connect it to a specific device and build redudancy. Work in progress. 



<h3>Docker Swarm - Management Layer</h3>
<h4>Traefik</h4> 
I used NGINX for this in the past. Switching to Traefik as it is build for cloud applications but it is not for the faint hearted. It allows me to add SSL to container HTTP traffic without the need to create a seperate certificate for each container. In addition to securing the transmission, I can also control access for services that do not have access control from a central system. Since I am exposing some ports to the internet through my firewall, I want to secure these services.<br/>
Given that this took me quite some time to get to work (compared to NGINX) I need to document this more detailed as I build this out. Anyone struggleing with this I can say only one thing, get your DNS setup right and working. 

<img src="https://github.com/antil697/docker-swarm/blob/master/Images/traefik.png" /><br/>

<h4>Authelia</h4>
Enforcing the entry path to my services via Cloudflare -> Unifi -> KeepAliveD -> Traefik, Authelia provides 2FA authentication through Traefik using the DUO app and service. This allows me to easily login and approve the login on my Apple Watch from the DUO push notificaiton.
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/IMG_1155.PNG" width="100" /><img src="https://github.com/antil697/docker-swarm/blob/master/Images/IMG_1156.PNG" width="100" /><br/>
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/1FA.png" /><br/>
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/archi.png" /><br/>
Authelia stack provides MariaDB as storage backend for user preferences and Redis for session peristence.
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
<h4>Registry</h4>
To ensure consitency of images and speed up deployment across nodes when new instances are forked on a a node, a local registry is provided as proxy service.<br/>
All pulls from docker will be maintained locally and are available to each node. This also means that my own builds need to be pushed first.<br/>

<h3>Docker Swarm - Services Layer</h3>
<h4>NodeRed</h4>
The core of my system. After starting with OpenHab and HomeAssistant, I found that the combination of NodeRed with MQTT and Zigbee2MQTT is a clear winner. HomeAssistant is only used for its floorplan addon that delivers the SVG dashboard to display on the touchscreen of Node4.
Every logic operates within NodeRed. While my core lights are Phillips Hue based and I could in theory move Hue to Zigbee2MQTT, I have not done this so far. The logic was to keep the system independant so the basic light functions work should my Raspberry Pi fail. NodeRed would only enhance the Hue hub. But with this new HA design, I can move the devices over to my Zigbee coordinator.<br/>
The flow shown below connects to Kodi and checks if a movie is playing. In case that the playtime is more than 10 minutes and lights are on, the flow send instructions to the message bus to dimm lights and enable the scene defined to watch movies. 
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/nodered.png" />

<h4>Mosquitto</h4>
Part of the key system. Mosquitto provides the message bus for all automation actions and status updates. While NodeRed is the brain, Mosquitto is the nervous systems that reports sensor states and provides commands for actions to actors.<br/>
Running an MQTT Broker on every node provides high availablility as docker swarm manages the connection. I use the KeepAliveD IP as the broker address for my IOT devices to report to. However, this creates a problem. A devices connected to the broker on Node1 may not see the messages by a devices that Docker Swarm connects to Node2. In order to overcome this, 2 MQTT Bridges are deployed. Bridges connect the brokers with each other and ensure that all see the messages. Mosquitto can be configured to use a primary and secondary bridges for this purpose. 
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/mqtt.png" />

<h4>Home Assistant</h4>
The only function that HomeAssiant provides is the dashboard to control and monitor basic functions from a central system. 
It uses the <a href="https://github.com/pkozul/ha-floorplan">floorplan</a> addon. Node3 is attached to a the offical Raspberry Pi touchscreen and runs in kiosk mode. It also provides services for less intensive workloads as backup.

<h4>Unifi Controller</h4>
The controller for my USG firewall.

<h4>PiHole</h4>
Using PiHole to get rid of pesky trackers and ads. I generally avoid going overboard with blocklists. I worked out the right balance for me. Bye bye ads and trackers.<br/>
Assigning one instance to Node1 and Node2 with a decicated IP addresses within the range reserved on each node using mcvlan network allows for a primary and secondary instance. The Unifi UGS that manages DHCP provides the addresses for these as DNS servers on the network.<br/>
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/pihole.png" />

<h3>GlusterFS</h3>
Moved away from mounting my Qnap as NFS share and installed GlusterFS. This gives more flexibility. Currently still monitoring the performance. <br/>

<h2>System Monitoring</h2>
<h4>Telegraf</h4>
Running Telegraf in each node to collect system data and report via MQTT and store in InfluxDB. While I could create a container and run it on each node, it won't be able to run [exec] plugins to collect CPU temperature.
Telegraf will also make data available to MQTT in JSON format for futher processing in NodeRed if required. 




# Dashboard
A dashboard using HomeAssitant floorplan add-on. It is displayed on a Raspberry Pi 3B with a touchscreen in kiosk mode providing basic controll functionality.
<img src="https://github.com/antil697/docker-swarm/blob/master/Images/Dashboard.png" />

# Arduinos/Tasmota

A number of devices are running on ESP8622, ESP32 or variants.<br/>
Tasmota is the prefered choice. Only a few run customised code. 

