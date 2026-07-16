# Vprofile Multi-Tier Deployment — Manual → Automated

A 5-tier Java web application stack (Nginx, Tomcat, MySQL, Memcache, RabbitMQ) deployed with Vagrant + VirtualBox — first provisioned manually to understand each service, then automated with shell provisioners.

## Architecture

```
User → Nginx (reverse proxy, port 80)
         → Tomcat (app server, port 8080)
             → MySQL (database, port 3306)
             → Memcache (DB caching, port 11211)
             → RabbitMQ (message broker, port 5672)
```

5 VMs, each running one service:

| VM    | Role                  | IP              |
|-------|-----------------------|-----------------|
| web01 | Nginx (reverse proxy) | 192.168.56.11   |
| app01 | Tomcat (app server)   | 192.168.56.12   |
| mc01  | Memcache              | 192.168.56.14   |
| db01  | MySQL/MariaDB         | 192.168.56.15   |
| rmq01 | RabbitMQ              | 192.168.56.16   |

## Attribution

Provisioning scripts and application source are based on the vprofile-project course reference architecture ([hkhcoder/vprofile-project](https://github.com/hkhcoder/vprofile-project)), followed as part of my DevOps learning path. This repo documents my own manual provisioning attempt, the debugging process I went through when things broke, and my review of the automated scripts.

## Phase 1: Manual Provisioning

Configured each of the 5 VMs by hand via SSH — installing and configuring MySQL, Memcache, RabbitMQ, Tomcat, and Nginx in that dependency order.

This is where the real learning happened. The stack didn't work first try — I hit a 502 Bad Gateway, then a database connection failure, then a RabbitMQ error, then a Memcache service crash. Debugging these one by one across a live multi-VM network is documented in full in [`manual-provisioning/DEBUGGING_NOTES.md`](manual-provisioning/DEBUGGING_NOTES.md).

**Summary of root causes found:**
- `firewalld` on 3 separate VMs (app01, db01, rmq01) had never had their required ports opened, despite the setup steps including the commands to do so — likely lost after VM reloads
- Memcache's own crash was a different root cause entirely: its config was set to bind to both IPv4 and an unavailable IPv6 address, causing the service itself to fail to start

## Phase 2: Automated Provisioning

Converted the manual steps into Vagrant shell provisioners (`automated-provisioning/*.sh`), explicitly baking in the fixes identified during manual debugging:
- `mysql.sh` opens `3306/tcp` explicitly
- `memcache.sh` opens `11211/tcp` and `11111/udp` explicitly
- `rabbitmq.sh` opens `5672/tcp` explicitly

**Known inconsistency (intentionally left in, noted here for transparency):** `tomcat.sh` disables `firewalld` entirely rather than opening only port `8080/tcp` like the other services. This works, but is a different — and weaker — security posture than the explicit-port approach used elsewhere. Flagged as a fix for a future pass rather than silently changed, since the goal of this repo is to document real work, including its rough edges.

## Setup

```bash
cd automated-provisioning/
vagrant up
```

Then visit `http://192.168.56.11` in your browser.

## Configuration

`application.properties` is gitignored since it contains service credentials. Copy the example file and fill in real values before running:

```bash
cp application.properties.example application.properties
```

## What I Learned

- How to trace a failure through a multi-tier network stack — distinguishing app-layer errors from network-layer errors from config errors
- Reading `journalctl` and `catalina.out` logs to find root cause instead of guessing
- `firewalld` port state can be lost across VM reloads, and why explicit provisioning scripts are more reliable than manual one-off commands
- Reviewing automated scripts with the same scrutiny as manual work — catching the `tomcat.sh` firewall inconsistency during my own review, not because something broke

## Next Steps

- Standardize the firewall approach across all services (explicit ports everywhere, not disabling firewalld)
- Replace shell provisioning with Ansible
- Containerize the stack with Docker
- Migrate to AWS with Terraform
