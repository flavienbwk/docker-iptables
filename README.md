# docker-iptables

Secure your Docker install with additional and understandable iptables.

## Use case

A lot of solutions on the web are suggesting ways to restrict external access to containers running on a server. Most of them are about disabling Docker's iptables (which causes containers to not be able to access the external network) or [using Docker's DOCKER-USER and `ufw-user-forward` iptables chains](https://github.com/chaifeng/ufw-docker) which doesn't allow managing **published** port.

Let's say you want to secure a production server exposed on your network where you deployed your Docker applications. You want to be able to **block all ports but the ones you want to expose**.

## Problem

Docker iptables and UFW iptables are independent. Meaning that you can try enabling UFW blocking _all_ ports and find your containers still accessible.

This is due to Docker using [its own iptables chains](https://docs.docker.com/network/iptables/#add-iptables-policies-before-dockers-rules) : `DOCKER` and `DOCKER-USER`, which are evaluated before the classic `FORWARD` chain (managed by UFW).

## Combining UFW with iptables

I recommend combining UFW with iptables managing our Docker network. BTW : UFW uses iptables under the hood.

- UFW : managing non-Docker / system ports
- custom Docker iptables : managing Docker apps ports
