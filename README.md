# Centralized Logging with Grafana Loki & Alloy using Ansible

This repository contains the configuration and Ansible playbook to deploy a centralized logging system. The setup consists of multiple nodes running Grafana Alloy agents, which collect systemd journal logs and ship them to a central Grafana Loki instance for storage and visualization in Grafana.

This project demonstrates a real-world scenario of setting up, troubleshooting, and securing a modern observability stack.

## Architecture Overview

* **Log Collection:** [Grafana Alloy](https://grafana.com/docs/alloy/) is deployed as a Docker container on multiple nodes with different architectures (`amd64` and `arm64`). It is configured to collect logs from the `systemd` journal.
* **Log Aggregation:** A central [Grafana Loki](https://grafana.com/oss/loki/) instance, running in Docker on one of the nodes, receives logs from all Alloy agents.
* **Deployment:** The Alloy agents are deployed and configured automatically using Ansible. One agent is deployed locally on the Ansible control node itself (a Raspberry Pi), while the others are deployed to remote servers. The Loki instance was set up manually.
* **Network Architecture:** The central Loki instance is hosted on a public VPS. This "push" model is necessary because one of the nodes (the Raspberry Pi) is located on a home network behind Carrier-Grade NAT (CGNAT), making it unreachable from the public internet. All Alloy agents push their logs outwards to the publicly accessible Loki server.
* **Visualization:** A separate Grafana instance (not included in this repo) is used to query Loki and display the logs.

## Repository Structure
├── README.md               # This file  
├── deploy_alloy.yml        # The main Ansible playbook for deploying Alloy agents  
├── inventory.ini           # Ansible inventory file defining the target nodes  
├── loki-server/  
│   └── docker-compose.yml  # Docker Compose file for the central Loki server  
└── templates/  
    └── config.alloy.j2     # Jinja2 template for the Alloy configuration  

## Setup and Installation

### 1. Loki Server (Manual Setup)

The Loki server was set up manually on one of the nodes.

1.  SSH into the designated Loki server (which must have a public IP address).
2.  Create a directory, e.g., `~/loki-server`.
3.  Place the `loki-server/docker-compose.yml` file in this directory.
4.  Run `docker-compose up -d` to start the Loki service.
5.  Ensure port `3100` is open in the firewall for the other nodes to be able to send logs.

### 2. Alloy Agents (Ansible Deployment)

The Alloy agents are deployed via Ansible.

1.  **Prerequisites:**
    * Ansible installed on your control machine.
    * Docker installed on all target nodes (`alloy_nodes`).
    * SSH access configured from the control machine to the remote target nodes.
2.  **Configuration:**
    * Update the `inventory.ini` file with the IP addresses or hostnames of your nodes. Note the special configuration for `pi.local` which uses a local connection for deployment on the control node itself.
    * In `deploy_alloy.yml`, verify the `loki_url` variable points to your Loki server's public IP address.
3.  **Run the Playbook:**
    ```bash
    ansible-playbook -i inventory.ini deploy_alloy.yml
    ```
    This will deploy and start the Grafana Alloy container on all nodes defined in the `alloy_nodes` group.

## Troubleshooting Journey & Lessons Learned

Setting up this system involved a comprehensive troubleshooting process that revealed several layers of issues. This journey was crucial for creating a robust and stable logging pipeline.

### Initial Problem: No Data in Grafana

After the initial deployment, no logs were visible in Grafana's Explore view, even though the Alloy containers were running without errors and the Grafana-Loki connection test was successful.

### Investigation & Solutions:

1.  **Network Connectivity:**
    * **Problem:** Could the Alloy agents reach the Loki server? Could Grafana reach Loki for queries?
    * **Solution:** I used `curl` and `wget` from within the respective containers to test the connections. This confirmed that basic network connectivity was fine, but it was an important first step to rule out simple network issues.

2.  **Firewall Inconsistencies (`iptables` vs. `ufw`):**
    * **Problem:** One of the four nodes could not be reached on its Alloy UI port (`12345`), despite `ufw allow 12345` being set.
    * **Investigation:** Using `sudo ss -tlpn` and `curl http://127.0.0.1:12345` on the problematic node, I confirmed the Alloy process was listening locally. This isolated the issue to an external network block.
    * **Solution:** A deep dive into `sudo iptables -L -n -v` revealed that the server's default `INPUT` policy was `DROP`, and the `ufw` rule had not been correctly applied to the `iptables` chain. **Manually inserting an `iptables` rule (`iptables -I INPUT ...`) immediately solved the connectivity issue.** This was a key lesson in how firewall layers can interact.

3.  **Missing Volume Mount (`/etc/machine-id`):**
    * **Problem:** Even with network connectivity fixed, no logs were being sent to Loki. The Loki logs showed it was receiving queries but had `total_entries=0`.
    * **Investigation:** I deduced the problem had to be with Alloy's ability to *read* the journal logs.
    * **Solution:** The `systemd` journal relies on `/etc/machine-id` to identify and access persistent log files located in `/var/log/journal/`. The Docker container was missing the volume mount for this file. **Adding `- /etc/machine-id:/etc/machine-id:ro` to the Ansible playbook was the final, critical fix** that allowed Alloy to read the logs and start streaming them to Loki.

### Architectural Decisions & Findings:

1.  **Designing for Network Constraints (CGNAT):**
    * **Challenge:** A key architectural challenge was that the Ansible control node (`pi.local`) is located on a home network behind Carrier-Grade NAT (CGNAT).
    * **Impact:** CGNAT means the Raspberry Pi does not have a public, routable IP address. It cannot host services that need to be reached by external servers. Therefore, it was not possible to host the central Loki instance on the Pi.
    * **Solution:** The architecture was designed as a "push" model. The Loki server was intentionally placed on a public VPS with a static IP address. This allows all Alloy agents, including the one on the CGNAT'd Pi, to reliably push their logs outwards to a stable, reachable endpoint.

2.  **Cross-Architecture Deployment (`amd64` vs `arm64`):**
    * **Challenge:** The deployment targets included standard `amd64` servers as well as a Raspberry Pi (`arm64`) which served as the Ansible control node.
    * **Solution:** The official `grafana/alloy` Docker image is a multi-architecture image. This means Docker automatically pulls the correct image version for the host's architecture (`amd64` for the servers, `arm64` for the Pi). The Ansible playbook did not require any architecture-specific changes, which simplifies the deployment process significantly.

### Operational Improvements Using Log Data
Once the data started flowing, the centralized logs immediately became a powerful tool for root cause analysis and system hardening. The following are two key examples of how the logs were used to improve the system's stability and security:

1.  **Root Cause Analysis (System Updates):**
    * **Finding:** I identified that a Pterodactyl Wings "crash" was actually a symptom of the Docker service being gracefully restarted by a nightly `apt upgrade` script.
    * **Solution:** The nightly update cron job was moved to align with a planned game server restart, synchronizing the two brief downtimes into one predictable event.

2.  **Security Hardening (Firewall Noise):**
    * **Finding:** The logs clearly showed constant scanning attempts from bots across the internet (`[UFW BLOCK]`), validating the firewall's effectiveness.
    * **Solution:** This prompted a port change for the SFTP service to significantly reduce log noise and improve the signal-to-noise ratio.

This project serves as a practical example of building a resilient system and using observability tools to understand, debug, and improve its operation.
