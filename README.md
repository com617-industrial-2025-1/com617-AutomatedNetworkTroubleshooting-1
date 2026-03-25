# COM617 - Automated Network Troubleshooting System
### Monitor -> Analyse -> React -> Report (MARR Framework)

**Module:** COM617 Industrial Consulting Project  
**Sponsor:** James Whale, Cisco SRE  
**Tutor:** Craig Gallen, Southampton Solent University  
**Author:** Jose Vasconcelos  
**Repo:** github.com/KariocaMarron/com617-AutomatedNetworkTroubleshooting-1

---

## Architecture Overview

    SIMULATION       MONITORING       ANALYSIS        AUTOMATION      REPORTING
    Containerlab --> OpenNMS      --> Python       --> Ansible     --> Mattermost
    FRR routers      (alerts)        classifier      playbooks       #network-alerts
    172.20.0.11-13   Monitor         Analyse:5000    docker exec     Report

| Component      | Image                        | IP            | Port          |
|----------------|------------------------------|---------------|---------------|
| router1        | frrouting/frr:v8.4.1         | 172.20.0.11   | —             |
| router2        | frrouting/frr:v8.4.1         | 172.20.0.12   | —             |
| router3        | frrouting/frr:v8.4.1         | 172.20.0.13   | —             |
| snmp-notifier  | ubuntu:22.04                 | 172.20.0.14   | 162/udp       |
| Mattermost     | mattermost-team-edition:9.5  | localhost      | 8065          |
| PostgreSQL     | postgres:15-alpine           | 172.21.0.10   | 5432 internal |
| Python recv    | Flask 3.0                    | localhost      | 5000          |

---

## Prerequisites

| Tool              | Min Version | Check command                                          |
|-------------------|-------------|--------------------------------------------------------|
| Ubuntu            | 22.04+      | lsb_release -a                                         |
| Docker            | 24.0+       | docker --version                                       |
| Docker Compose    | v2.0+       | docker compose version                                 |
| Containerlab      | 0.74+       | containerlab version                                   |
| Python            | 3.10+       | python3 --version                                      |
| Ansible           | core 2.14+  | ansible --version                                      |
| community.docker  | 3.0+        | ansible-galaxy collection list community.docker        |

User group check:

    groups $USER
    # Required: docker  ubridge  clab_admins

---

## First-Time Setup

    git clone git@github.com:KariocaMarron/com617-AutomatedNetworkTroubleshooting-1.git
    cd com617-AutomatedNetworkTroubleshooting-1

    pip3 install -r python/requirements.txt --break-system-packages
    ansible-galaxy collection install community.docker frr.frr

    cp .env.example .env
    nano .env

.env contents:

    MATTERMOST_WEBHOOK_URL=http://localhost:8065/hooks/your-webhook-id
    RECEIVER_PORT=5000

---

## Starting the Lab

### Automated (recommended)

    ansible-playbook scripts/lab-start.yml

### Manual (step by step)

    # Step 1 - Mattermost + PostgreSQL
    docker compose up -d
    docker compose ps

    # Step 2 - Containerlab network
    sudo containerlab deploy --topo containerlab/topology.yml

    # Step 3 - Python alert receiver
    cd python && python3 alert_receiver.py > /tmp/marr_receiver.log 2>&1 &
    cd ..

    # Step 4 - Verify
    curl -s http://localhost:5000/health | python3 -m json.tool

---

## Verifying the Lab

    # Receiver health check
    curl -s http://localhost:5000/health

    # OSPF neighbours on router1 (expect 2 x Full/DR)
    sudo docker exec clab-com617-marr-lab-router1 vtysh -c "show ip ospf neighbor"

    # BGP summary on router1 (expect 2 peers)
    sudo docker exec clab-com617-marr-lab-router1 vtysh -c "show ip bgp summary"

    # Watch receiver log
    tail -f /tmp/marr_receiver.log

    # Mattermost UI - open in browser
    # http://localhost:8065

---

## Sending a Test Alert

    curl -s -X POST http://localhost:5000/alert \
      -H 'Content-Type: application/json' \
      -d '{"id":"TEST-001","nodeLabel":"router1","severity":"CRITICAL",
           "uei":"uei.opennms.org/generic/traps/SNMP_Link_Down",
           "description":"Interface eth1 down on router1","ifDescr":"eth1"}' \
      | python3 -m json.tool

Expected response:

    {
        "alert_id": "TEST-001",
        "diag_rc": 0,
        "fault_type": "link_down",
        "playbook": "diagnose_link_down",
        "status": "processed"
    }

### Simulate a real link failure

    # Bring down router1 eth1
    sudo docker exec clab-com617-marr-lab-router1 ip link set eth1 down

    # Watch OSPF reconverge on router2
    sudo docker exec clab-com617-marr-lab-router2 vtysh -c "show ip ospf neighbour."

    # Restore
    sudo docker exec clab-com617-marr-lab-router1 ip link set eth1 up

---

## Stopping the Lab

### Automated (recommended)

    # Standard stop - preserves Mattermost data
    ansible-playbook scripts/lab-stop.yml

    # Full wipe - removes all Docker volumes
    ansible-playbook scripts/lab-stop.yml --extra-vars "wipe_data=true"

### Manual (step by step)

    # Step 1 - Stop Python receiver
    pkill -f alert_receiver.py

    # Step 2 - Destroy Containerlab
    sudo containerlab destroy --topo containerlab/topology.yml

    # Step 3 - Stop Docker Compose
    docker compose down

    # Step 4 - Verify
    docker ps | grep -E "marr|clab"

---

## Troubleshooting

| Problem                          | Fix                                                        |
|----------------------------------|------------------------------------------------------------|
| containerlab not found           | bash -c "$(curl -sL https://get.containerlab.dev)"         |
| Docker permission denied         | sudo usermod -aG docker $USER && newgrp docker             |
| Failed to verify bind path       | Run containerlab from repo root                            |
| Port 5000 already in use         | pkill -f alert_receiver.py                                 |
| MATTERMOST_WEBHOOK_URL not set   | Check .env exists in repo root                             |
| Mattermost not reachable         | docker compose up -d                                       |
| BGP shows (Policy)               | Normal for FRR lab - peers are up                          |
| OSPF not converged               | Wait 30s after deploy                                      |

---

## Git Workflow

    # Push to personal repo
    git push origin main

    # Share with team
    git checkout feature/jose-marr-implementation
    git merge main
    git push team feature/jose-marr-implementation
    git checkout main

| Remote  | URL                                                        | Purpose       |
|---------|------------------------------------------------------------|---------------|
| origin  | git@github.com:KariocaMarron/com617-...                    | Personal repo |
| team    | https://github.com/com617-industrial-2025-1/com617-...     | Team repo     |
| opennms | git@github.com:KariocaMarron/opennms-pyu-monitoring        | Subtree       |

---

*COM617 Industrial Consulting Project - Southampton Solent University - 2025/26*
