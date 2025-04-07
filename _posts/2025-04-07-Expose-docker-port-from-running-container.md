---
layout: post
title:  "Expose docker port from running container"
---


## The use case

We have an **internal** [mongodb](https://www.mongodb.com/) container running on a server. It is used by a backend service, and in normal conditions it is not necessary to access it from outside the server. However, in some cases, we need to access it from our local machine for debugging purposes.

So I did a quick search and found a solution that works for me. The inspiration came from [docker forums](https://forums.docker.com/t/how-to-expose-port-on-running-container/3252/7), there were several solutions, for example some were pointing to add rules to iptables, but this changes the host configuration, specifically the `DOCKER` chain in the nat table, which I wanted to avoid.

So, scrolling down the thread, I found a solution that uses `socat` to forward the port from the container to the host. This is a good solution because it does not require any changes to the host configuration, and it is easy to remove when we no longer need it.

## The solution

After testing the proposed solution locally, I decided to implement it in my server. The steps are as follows:

```bash
# ssh into the server
ssh user@server

# get IP of the container
CONTAINER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container_name_or_id>)

# use [netshoot](https://github.com/nicolaka/netshoot) image to forward the port
docker run --rm -it --network host nicolaka/netshoot socat tcp-listen:27017,fork tcp:${CONTAINER_IP}:27017
```

voila! now you can access the mongodb instance from your local machine

**This server does not have a public IP. Access is allowed only inside the VPN/Vnet**