---
title: "Caddy + Docker: Why Labels Beat Config Files"
date: 2025-12-09T03:10:00+03:00
draft: false
description: "In the battle between Ansible purity and Docker convenience, I chose the latter. Here is how to build a secure atomic infrastructure with Caddy and Socket Proxy."
tags: ["docker", "caddy", "ansible", "security", "architecture"]
categories: ["DevOps", "HomeLab"]
---

When you start writing your own Ansible collection for infrastructure (in my case, a stack with AmneziaWG, monitoring, and Caddy), you inevitably hit an architectural fork in the road: **how do you link services together?**

Specifically: how do you tell the reverse proxy (Caddy) that a new container has spun up and needs HTTPS?

In Ansible textbooks and guides on "proper DevOps," the answer is unequivocal: **Config Files**.
In the reality of the Docker community, the answer is also unequivocal, but different: **Labels**.

I faced this choice while rewriting the role for Amnezia, and I decided to document why I consciously chose the "non-canonical" path.

### Path 1: Files (The Ansible Way)

The idea is simple: Ansible is a configuration management system. Therefore, it should generate configs.
The `amnezia` role should generate an `amnezia.caddy` file and place it in the `/etc/caddy/sites/` folder.

**Why I said "No":**
1.  **Breaking Separation of Concerns.** The VPN service role suddenly needs to know the internal directory structure of the web server role.
2.  **State Drift.** A container might crash or be removed, but the config file remains. Caddy will keep trying to hammer a dead upstream.
3.  **Loss of Context.** I want to open `docker-compose.yml` and see the entire service description in one place: ports, volumes, and how it is exposed to the internet. Splitting this between a YAML manifest and a Caddyfile means complicating debugging for myself six months down the line.

### Path 2: Labels (The Docker Way)

The "Cloud Native" idea: infrastructure should be descriptive. I attach a label `caddy: vpn.example.com` to the container, and the infrastructure adapts automatically.

**The Main Fear:**
For this to work, you need to give Caddy access to `/var/run/docker.sock`. In security chats, you’d get a slap on the wrist for this. If Caddy is compromised, the attacker gains root access to the entire host via the Docker API.

### My Solution: Labels + Wollomatic Proxy

I decided that operational convenience (Self-Contained services) is more important to me than Ansible purism. However, leaving a security hole is not an option.

So, my architecture looks like this:

1.  **Caddy Docker Proxy** is used as the controller. It reads labels and builds routes on the fly.
2.  **No direct socket access.** In the Caddy `docker-compose`, I do not mount `/var/run/docker.sock`.
3.  **Wollomatic Socket Proxy.** A tiny sidecar container (written in Go, scratch image) runs alongside it. It exposes the socket to the internal network, but with a strict `allowlist`.

#### Security Config

I allow only `GET` requests to `containers` and `events`.

```
services:
socket-proxy:
image: wollomatic/socket-proxy:1
command:
- '-loglevel=info'
- '-allowGET=/v1\..{1,2}/(version|containers/.*|events.*)' \# ✅ Allow read-only
- '-watchdoginterval=3600'
volumes:
- /var/run/docker.sock:/var/run/docker.sock:ro
networks:
- caddy-network
```

Even if Caddy is hacked, the attacker will hit this filter. They won't be able to create a new container, kill an existing one, or read secrets.

### Summary

Yes, writing JSON configs inside YAML labels is a distinct kind of "pleasure." It can look pretty spooky at times:

```
labels:
caddy.route.0_forward_auth.copy_headers: "Remote-User Remote-Name..."
```

But this is the price for **atomicity**. I can remove the Amnezia role from the playbook, and my Caddy will automatically stop serving that domain, without leaving "garbage" files in the configs.

For my collection (and for my peace of mind while maintaining this stack), this turned out to be the deciding factor.

Which side are you on: "Clean Files" or "Smart Containers"?
