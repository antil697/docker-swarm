# docker-swarm
Documenting the current design 

# High Level Architecture
<h4>Keepalived</h4> 
Running Keepalived as single container on each node provides the benefit of one single virtual IP for the cluster. 
While Docker Swarm already provides load balancing and connects to the right service, the IP is independent from the node. This serves as ingress point to the cluster.

<img src="https://github.com/antil697/docker-swarm/blob/master/Images/docker_swarm.png" />


# Dashboard

<img src="https://github.com/antil697/docker-swarm/blob/master/Images/Dashboard.png" />
