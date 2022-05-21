# docker-iptables

Secure your Docker install with additional and understandable iptables.

## Use case

A lot of solutions on the web are suggesting ways to restrict external access to containers running on a server. Most of them are about disabling Docker's iptables (which causes containers to not be able to access the external network) or [using Docker's DOCKER-USER and `ufw-user-forward` iptables chains](https://github.com/chaifeng/ufw-docker) which doesn't allow managing **published** port.

Let's say you want to secure a production server exposed on your network where you deployed your Docker applications. You want to be able to **block all ports but the ones you want to expose**.

## Problem

Docker iptables and UFW iptables are independent. Meaning that you can try enabling UFW blocking _all_ ports and find your containers still accessible.

This is due to Docker using [its own iptables chains](https://docs.docker.com/network/iptables/#add-iptables-policies-before-dockers-rules) : `DOCKER` and `DOCKER-USER`, which are evaluated before the classic `FORWARD` chain (managed by UFW).

## Combining UFW with iptables

I recommend combining UFW with iptables managing our Docker network. BTW : UFW uses iptables under the hood. The technics described here does not require disabling iptables in the Docker daemon. [Which is good](#problem).

- UFW : managing non-Docker / system ports
- custom Docker iptables : managing Docker apps ports

Install first UFW (example with `apt`) :

> :information_source: Installing UFW won't directly enable it

```bash
sudo apt install -y ufw
```

You will probably want to allow port 22 for SSH. Allow the system ports you want to expose with the following command :

```bash
sudo ufw allow 22
```

When you're ready to go, enable UFW with :

```bash
sudo ufw enable
```

> :information_source: If you currently have running containers, you can try accessing any port exposed by a container and see that it is accessible although we've blocked EVERTHING but port 22, [as explained before](#problem).

Now let's configure Docker's iptables.

Here, we want to expose a container publishing port 8080. E.g: `docker run -i nginx -p 80:8080`.

> :information_source: To find your network interface (`eth0` for the example), type the `ip a` command. This should be `eth0` or a name prefixed by `eth`, `enp` or `esp`. Eventually `wlan` or `wls` if you are working over Wi-Fi.

```bash
iptables -I DOCKER-USER -i eth0 -p tcp -m conntrack --ctorigdstport 8080 -j ACCEPT
```

If you have multiple ports published (e.g: 8080, 4444...), this snippet might be handy :

```bash
NETWORK_INTERFACE=eth0
for DOCKER_PORT in 8080 4444
do
    iptables -I DOCKER-USER -i "$NETWORK_INTERFACE" -p tcp -m conntrack --ctorigdstport $DOCKER_PORT -j ACCEPT
done
```

Finally, we need to make sure internal networks can communicate :

```bash
sudo iptables -I DOCKER-USER -j RETURN -s 10.0.0.0/8
sudo iptables -I DOCKER-USER -j RETURN -s 172.16.0.0/12
sudo iptables -I DOCKER-USER -j RETURN -s 192.168.0.0/16
sudo ufw allow from 172.16.0.0/12
sudo ufw allow from 192.168.0.0/16
```
