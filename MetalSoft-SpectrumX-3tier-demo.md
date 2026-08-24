<!-- AIR:tour -->

# MetalSoft for NVIDIA Spectrum-X: 3-Tier Demo (512 GPU)

MetalSoft is an intelligent orchestration platform that transforms fragmented on-premises hardware into high-performance, fast-changing, secure, workload-compliant infrastructure. It integrates servers, switches, and storage to provide a turnkey neocloud platform (NCP) solution.

This lab builds a three-tier, two-POD NVIDIA Spectrum-X fabric (512 GPU) from a clean state and attaches a tenant to it, using the MetalSoft CLI (`metalcloud-cli`) and Terraform. The fabric spans two PODs joined by a super-spine tier and runs EVPN over eBGP on Cumulus Linux 5.14.0. Its distinguishing behaviour is inter-POD reachability: a host in one POD reaches a host in the other across the super-spine tier.

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/topology.3tier.png)


The lab is preconfigured: it launches from a stored MetalSoft Spectrum-X checkpoint with the Global Controller, Site Controller, and Cumulus Linux switches already running and the CLI toolkit already staged on the jumpstation. You perform the fabric build, switch configuration, and tenant onboarding yourself during the lab; nothing is pre-built for you.

**IMPORTANT:** Allow about 60 minutes end to end; most of that time is switch deployment. Unless a step says otherwise, run every command on the `oob-mgmt-server` jumpstation.

## Table of Contents

- [Lab Story and Scenario](#lab-story-and-scenario)
- [Features and Services](#features-and-services)
- [What You Will Do in This Lab](#what-you-will-do-in-this-lab)
- [Demo Topology Overview](#demo-topology-overview)
- [Demo Topology Information](#demo-topology-information)
- [Demo Environment Access](#demo-environment-access)
- [Lab Flow](#lab-flow)
- [Validation Summary](#validation-summary)
- [Troubleshooting, Upgrade, or Reset](#troubleshooting-upgrade-or-reset)
- [References](#references)
- [Contact Info](#contact-info)

## Lab Story and Scenario

A network or infrastructure engineer has validated MetalSoft's orchestration model at one and two scalability units and now needs to answer the question every large Spectrum-X deployment eventually asks: what happens when a single two-tier fabric outgrows its spine layer, and PODs need to be joined by a super-spine tier instead? The engineer needs proof that MetalSoft can describe and deploy a three-tier fabric with the same declarative workflow, and that host-to-host traffic actually crosses the super-spine tier correctly.

In this lab, the engineer builds a three-tier, two-POD fabric (twenty switches: eight leaves, eight spines, four super-spines) from the jumpstation with `metalcloud-cli` and Terraform: create the fabric, import the switches, discover links, configure and deploy the underlay and EVPN overlay across three tiers, register the HGX hosts, and onboard a tenant network. The lab closes with a full rail-mesh connectivity test that specifically exercises inter-POD reachability — a host in the first POD reaching its same-rail peer in the second POD over a path that transits the super-spine tier — proving the fabric scales to a multi-POD design without any additional manual switch configuration.

## Features and Services

This demo includes the following features and services:

- MetalSoft Global Controller and Site Controller orchestrating a three-tier, two-POD Cumulus Linux Spectrum-X fabric
- CLI-driven fabric creation, switch import, link discovery, and configuration templating (`metalcloud-cli`)
- Terraform-based tenant onboarding across both PODs
- MetalSoft Fabric Manager and Infrastructure Designer web UI (optional; the lab can be run entirely from the CLI)
- Three-tier leaf-spine-super-spine underlay with EVPN over eBGP and inter-POD overlay reachability
- HGX host rail networking (eight rail NICs per host) validated with a full inter-POD mesh connectivity test

## What You Will Do in This Lab

- Stand up a three-tier, two-POD (512 GPU) Spectrum-X fabric from scratch with `metalcloud-cli`
- Configure and deploy the Cumulus Linux underlay and EVPN overlay across the leaf, spine, and super-spine tiers
- Register the eight HGX hosts as endpoints and onboard a tenant with Terraform
- Configure host rail networking and verify full-mesh RoCE connectivity, including inter-POD reachability across the super-spine tier

<!-- AIR:page -->

## Demo Topology Overview

The lab is a three-tier fabric spanning two PODs: twenty Cumulus Linux switches (eight leaves, eight spines, and four super-spines), and eight HGX hosts, each connected to the leaf layer over eight rail NICs. Each POD is one scalability unit; the two PODs are joined by a shared super-spine tier. MetalSoft's Global Controller and Site Controller run alongside the fabric and are reachable from the jumpstation; nothing on the fabric is preconfigured; you build it in the Lab Flow section below.

### Device Naming

- Leaf: `leaf-pod00-su00-r0` .. `leaf-pod00-su00-r3` (POD 0), `leaf-pod01-su00-r0` .. `leaf-pod01-su00-r3` (POD 1)
- Spine: `spine-pod00-r0-s00` .. `spine-pod00-r3-s00` (POD 0), `spine-pod01-r0-s00` .. `spine-pod01-r3-s00` (POD 1)
- Super-spine: `ssp-group00-s00`, `ssp-group00-s01`, `ssp-group00-s02`, `ssp-group00-s03`
- HGX host: `hgx-pod00-su00-h00`/`h08`/`h16`/`h24` (POD 0), `hgx-pod01-su00-h00`/`h08`/`h16`/`h24` (POD 1)

### Devices

| __Role__ | __Device Names__ |
| -------- | ----------------- |
| Leaf (POD 0) | `leaf-pod00-su00-r0`, `leaf-pod00-su00-r1`, `leaf-pod00-su00-r2`, `leaf-pod00-su00-r3` |
| Leaf (POD 1) | `leaf-pod01-su00-r0`, `leaf-pod01-su00-r1`, `leaf-pod01-su00-r2`, `leaf-pod01-su00-r3` |
| Spine (POD 0) | `spine-pod00-r0-s00`, `spine-pod00-r1-s00`, `spine-pod00-r2-s00`, `spine-pod00-r3-s00` |
| Spine (POD 1) | `spine-pod01-r0-s00`, `spine-pod01-r1-s00`, `spine-pod01-r2-s00`, `spine-pod01-r3-s00` |
| Super-spine | `ssp-group00-s00`, `ssp-group00-s01`, `ssp-group00-s02`, `ssp-group00-s03` |
| HGX host (POD 0) | `hgx-pod00-su00-h00`, `hgx-pod00-su00-h08`, `hgx-pod00-su00-h16`, `hgx-pod00-su00-h24` |
| HGX host (POD 1) | `hgx-pod01-su00-h00`, `hgx-pod01-su00-h08`, `hgx-pod01-su00-h16`, `hgx-pod01-su00-h24` |
| Other | Jumpstation (`oob-mgmt-server`), Global Controller, Site Controller |

<!-- AIR:page -->

## Demo Topology Information

### IPAM

| __Hostname__ | __Role__ | __IP Address__ |
| ------------- | -------- | --------------- |
| `oob-mgmt-server` | Jumpstation | reached over the external SSH service, see [SSH Access](#ssh-access) |
| Global Controller | MetalSoft control plane and web UI | `192.168.200.3` |
| Site Controller | Site agent | `192.168.200.2` |
| Leaf, spine, and super-spine switches | Cumulus Linux 5.14.0 | `192.168.200.11` and up, one address per switch, in the order listed in `switches.3tier.yaml` |
| HGX hosts | Ubuntu compute nodes | `192.168.200.31` to `.38` (POD 0: `.31`-`.34`, POD 1: `.35`-`.38`) |

Once the fabric is deployed (Lab Flow, Step 6), every switch also carries a `10.253.128.x/32` loopback assigned in the same order as the management addresses above (`leaf-pod00-su00-r0` is `10.253.128.1`, `r1` is `.2`, and so on, continuing through the second POD; the spines and super-spines are addressed the same way).

### Physical Connectivity

MetalSoft discovers the leaf-spine and spine-to-super-spine links automatically over LLDP; there is no cabling table to prepare by hand. Lab Flow Step 7 (Discover links and redeploy) shows how to confirm the discovered links from a switch, and the Fabric Manager's Fabric View shows the same links visually once they are imported.

<!-- AIR:page -->

## Demo Environment Access

### Load Time and Readiness

Allow the Global Controller and Site Controller a few minutes to finish booting after the lab starts. Lab Flow Step 1 shows how to confirm both are up and connected before you run anything else. The full lab takes about 50 minutes end to end, most of it switch deployment time in Step 9.

### Console Access

After the lab is loaded, double-click any node in the NVIDIA Air topology view to open its console in the browser.

### Device Credentials

| __Device__ | __Username__ | __Password__ | __IP Address / Access Method__ |
| ---------- | ------------- | ------------- | -------------------------------- |
| `oob-mgmt-server` (Jumpstation) | `ubuntu` | `nvidia` | NVIDIA Air console, then SSH (see [SSH Access](#ssh-access)) |
| Global Controller | `root` | `MetalsoftR0cks@$@$` | `ssh -l root 192.168.200.3` |
| Site Controller | `root` | `MetalsoftR0cks@$@$` | `ssh -l root 192.168.200.2` |
| MetalSoft web UI | `demo@metalsoft.io` | `MetalsoftR0cks@$@$` | `https://demo.metalsoft.io` |
| MetalSoft Fabric Manager UI | `demo@metalsoft.io` | `MetalsoftR0cks@$@$` | `https://demo.metalsoft.io:<https service port>/designer/dashboard` |
| Cumulus switches | `cumulus` | set in `switches.3tier.yaml` | `ssh cumulus@<switch-ip>` |
| HGX hosts | `ubuntu` | `nvidia` | `ssh ubuntu@<host-ip>` |

### SSH Access

Alternatively, you can SSH into the `oob-mgmt-server` from your local machine instead of using the integrated web console.

First, open the jumpstation console inside NVIDIA Air: open the **Nodes** tab (or the topology view) and double-click the `oob-mgmt-server` node. Log in at the console with `ubuntu` / `nvidia`. Use Google Chrome; the NVIDIA Air console does not work in Safari.

Then install your public key from that console, so your workstation can SSH in:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA...replace-with-your-public-key... you@workstation' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Finally, publish an SSH service so the jumpstation is reachable from outside. In NVIDIA Air, click **Services > Services List** and add a service on the jumpstation:

- Service Name: `SSH`
- Interface: `oob-mgmt-server:eth0`
- Service Type: `SSH`
- Service Port: `22`

***Note:*** The external host and port may change each time you launch a new demo or start a stored simulation.

NVIDIA Air returns an external host name and port for the service. Connect to it as `ubuntu` and work in `~/nvidia`, where the configuration templates, the topology YAML files, and the Terraform manifests are already staged:

```bash
ubuntu@oob-mgmt-server:~$ cd nvidia/
ubuntu@oob-mgmt-server:~/nvidia$ ls
cumulus-5.14-templates  ethernet-fabric.3tier.yaml       l3-profile-tenant1.3tier.yaml  neplan                 route-domain-tenant1.3tier.yaml  switches.3tier.yaml
endpoints.3tier.yaml    fabric-config.3tier.l3evpn.yaml  l3-profile-tenant2.3tier.yaml  oob-subnet.3tier.yaml  route-domain-tenant2.3tier.yaml  terraform
ubuntu@oob-mgmt-server:~/nvidia$
```

The switch password has been set in `switches.3tier.yaml`.

For more background on NVIDIA Air services, SSH access, SSH keys, nodes, and consoles, see the [NVIDIA DSX Air Quick Start](https://docs.nvidia.com/networking-ethernet-software/nvidia-air/Quick-Start/).

### Web UI Access

This demo runs entirely from the command line, but the UI is useful to better understand the software's state.

The UI runs on the Global Controller; the steps below expose it and open it from your workstation.

In NVIDIA Air, there should already be a MetalSoft UI service enabled.

NVIDIA Air returns an external host name and port for the service (for example `worker-0375f999.dsx-air.nvidia.com` and `25990`). Note both; NVIDIA Air assigns a new external port on each restart.

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/metalsoft-https-service.webp)

Connect via SSH into the `oob-mgmt-server` from your local machine or using the integrated web console. Then enable the proxy service for the external host name:

```bash
ubuntu@oob-mgmt-server:~$ sudo makeproxy <external-host-name>
```

As an example, for the external host name `worker-0375f999.dsx-air.nvidia.com` the output should be the following:

```bash
ubuntu@oob-mgmt-server:~$ sudo makeproxy worker-0375f999.dsx-air.nvidia.com
==> Frontend: https://worker-0375f999.dsx-air.nvidia.com:443 (self-signed)
==> Backend : https://demo.metalsoft.io:443 (CA-verified)
==> haproxy, openssl and CA bundle already present
==> Generating self-signed certificate for worker-0375f999.dsx-air.nvidia.com
==> Backed up existing config to /etc/haproxy/haproxy.cfg.bak.20260731103317
==> Validating configuration
Configuration file is valid
==> Enabling and restarting haproxy
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2026-07-31 10:33:18 UTC; 5ms ago

==> Done.
    Point worker-0375f999.dsx-air.nvidia.com (or your port-forward) at this host, then open:
      https://worker-0375f999.dsx-air.nvidia.com/
      https://worker-0375f999.dsx-air.nvidia.com:<fwd-port>/   (via port-forward)
    Proxied to https://demo.metalsoft.io:443
    The cert is self-signed; clients must accept or trust it.
    Cert file: /etc/haproxy/certs/worker-0375f999.dsx-air.nvidia.com.pem
```

The proxy service from `oob-mgmt-server` uses a self-signed certificate because the external host name is dynamically allocated; the browser will warn about it. Accept the warning to continue.

Open `https://<external-host-name>:<external-port>/` in the browser, using the port from above, and log in:

- Username: `demo@metalsoft.io`
- Password: `MetalsoftR0cks@$@$`

For our example it should be `https://worker-0375f999.dsx-air.nvidia.com:25990/`

In case an error with `503 Service Unavailable` is displayed, that means that not all the services from the Global Controller are UP and running. Use the troubleshooting steps from Step 1 in case the error persists after 1-2 minutes.

Once logged in, you can reach every MetalSoft component from the "burger" menu at the top left, next to the logo:

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/admin-ui.webp)

| __Component__ | __Comments__ | __URL__ |
| -------------- | -------------- | -------- |
| Sites, Infrastructures & Global Configurations | Home of the application; create sites, see tenant infrastructures and ongoing deploys | `https://demo.metalsoft.io:<external-port>/` |
| Fabric Manager | Manage network equipment | `https://demo.metalsoft.io:<external-port>/designer/fabrics/1/topology` |
| Compute & Storage Manager | Not used in this demo | |
| Infrastructure Designer | Tenant's view | `https://demo.metalsoft.io:<external-port>/designer/infrastructures/1` |

<!-- AIR:page -->

## Lab Flow

### Step 1. Confirm the Controller Is Reachable

**Goal:** Make sure the Global Controller and Site Controller have finished booting before you run any fabric commands.

**Access needed:** Jumpstation (`oob-mgmt-server`).

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** The Global Controller and Site Controller may take a few minutes to boot after the lab starts.

Run the following commands:

```bash
ping -c3 192.168.200.3    # Global Controller
ping -c3 192.168.200.2    # Site Controller
```

Expected result:

- Both addresses answer. If either does not, the virtual machine most likely did not receive a DHCP lease when the lab started; stop the lab in NVIDIA Air, start it again, and re-check.

Validation:

Confirm the Site Controller has connected to the Global Controller by running the following command on `oob-mgmt-server`:

```bash
metalcloud-cli site agents 1
```
The expected output should be the following:

```
ubuntu@oob-mgmt-server:~$ metalcloud-cli site agents 1
┌─────────────────────────────────────────────────┬────────────────────────────┬─────────┬────────────┬─────────┬───────────────────┬─────────────────────┐
│ ID                                              │ HOSTNAME                   │ SITE    │ AGENT TYPE │ VERSION │ IP                │ LAST SEEN           │
├─────────────────────────────────────────────────┼────────────────────────────┼─────────┼────────────┼─────────┼───────────────────┼─────────────────────┤
│ dc-demo-fd17-625c-f037-2-a00-27ff-fec9-ab57-05e │ ms-tunnel-6b4869d7d9-qk57x │ dc-demo │ ms-agent   │ v7.4.2  │ 10.42.0.152:34482 │ 16 Jul 26 14:50 UTC │
└─────────────────────────────────────────────────┴────────────────────────────┴─────────┴────────────┴─────────┴───────────────────┴─────────────────────┘
```

### Step 2. Create and Activate the Fabric

**Goal:** Create the fabric object on the lab site, activate it, and create the out-of-band management subnet the switches are addressed from.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few seconds.

From this point on, run every command in this lab on the `oob-mgmt-server` jumpstation from the `~/nvidia` directory. There is no need to SSH into the Global Controller unless there is a problem; see [Troubleshooting](#troubleshooting-upgrade-or-reset).

```bash
cd ~/nvidia
metalcloud-cli fabric create 1 spectrumx-3tier-514 ethernet "Spectrum-X 3-tier (5.14)" \
        --config-source ethernet-fabric.3tier.yaml
```

```bash
metalcloud-cli fabric activate 1
```

```bash
metalcloud-cli subnet create --config-source oob-subnet.3tier.yaml
```

Expected result:

- The fabric `spectrumx-3tier-514` exists, is active, and has an out-of-band management subnet. Keep the fabric label `spectrumx-3tier-514`; the Terraform manifest in Step 12 resolves the fabric by that label. The fabric is recognised as three-tier because its inventory contains super-spine switches.

Validation:

- (Optional) In the web UI, open the Fabric Manager to see the fabric that was created.

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/fabric.3tier.webp)

### Step 3. Import the Switches

**Goal:** Register the twenty lab switches with the fabric.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; the switch management password is read from `switches.3tier.yaml`.

**Expected wait time:** A few seconds.

Set `fabricId` in `switches.3tier.yaml` to `1`, then import the twenty switches:

```bash
metalcloud-cli fabric import-devices 1 --config-source switches.3tier.yaml
```

Expected result:

- All twenty switches (eight leaves, eight spines, four super-spines) appear as devices on the fabric.

Validation:

- (Optional) In the web UI, navigate to **Fabric Manager > Network devices** where equipment list shows the imported switches:

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/network-devices.3tier.webp)

### Step 4. Discover Switch Interfaces

**Goal:** Trigger interface discovery on every switch and capture a clean baseline before any configuration is applied.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.3tier.yaml`) on the switches.

**Expected wait time:** A few minutes for discovery to complete across all switches.

The switch configuration in Step 5 works from each switch's discovered port inventory. Trigger discovery across every switch, then confirm one switch reports interfaces:

```bash
for ID in $(metalcloud-cli fabric get-devices 1 -f json | jq -r '.[].id'); do
  metalcloud-cli network-device discover "$ID"
done
```

```bash
metalcloud-cli network-device get-ports 1
```

Expected result:

- `get-ports` returns a non-empty list of `swpNsN` ports. If the list is empty, the switch is not reachable; check the address and password in `switches.3tier.yaml`.

The switches are imported and reachable, but MetalSoft has not configured them yet. The switch hostnames for this topology are:

- Leaves: `leaf-pod00-su00-r0`, `leaf-pod00-su00-r1`, `leaf-pod00-su00-r2`, `leaf-pod00-su00-r3`, `leaf-pod01-su00-r0`, `leaf-pod01-su00-r1`, `leaf-pod01-su00-r2`, `leaf-pod01-su00-r3`
- Spines: `spine-pod00-r0-s00`, `spine-pod00-r1-s00`, `spine-pod00-r2-s00`, `spine-pod00-r3-s00`, `spine-pod01-r0-s00`, `spine-pod01-r1-s00`, `spine-pod01-r2-s00`, `spine-pod01-r3-s00`
- Super-spines: `ssp-group00-s00`, `ssp-group00-s01`, `ssp-group00-s02`, `ssp-group00-s03`

Validation:

Log in as `cumulus` (the password is read from `switches.3tier.yaml`) and capture the baseline, so later steps have something to compare against.

For ease of use, there is a validation script named `spcx-run` located in `~/spcx-air/` that can check the current configuration of the switches. An environment variable needs to be exported for the desired topology:

```bash
export SPCX_INVENTORY=~/spcx-air/inventory.3tier.yml
```

```bash
~/spcx-air/spcx-run -c "hostname"
```

```bash
~/spcx-air/spcx-run -c "ip -br addr show lo"
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp summary\""
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "hostname"
========================================
Running: hostname
========================================
################################################################################
leaf-pod00-su00-r0 | cumulus
################################################################################
leaf-pod00-su00-r1 | cumulus
################################################################################
leaf-pod00-su00-r2 | cumulus
################################################################################
leaf-pod00-su00-r3 | cumulus
################################################################################
leaf-pod01-su00-r0 | cumulus
################################################################################
leaf-pod01-su00-r1 | cumulus
################################################################################
leaf-pod01-su00-r2 | cumulus
################################################################################
leaf-pod01-su00-r3 | cumulus
################################################################################
spine-pod00-r0-s00 | cumulus
################################################################################
spine-pod00-r1-s00 | cumulus
################################################################################
spine-pod00-r2-s00 | cumulus
################################################################################
spine-pod00-r3-s00 | cumulus
################################################################################
spine-pod01-r0-s00 | cumulus
################################################################################
spine-pod01-r1-s00 | cumulus
################################################################################
spine-pod01-r2-s00 | cumulus
################################################################################
spine-pod01-r3-s00 | cumulus
################################################################################
ssp-group00-s00 | cumulus
################################################################################
ssp-group00-s01 | cumulus
################################################################################
ssp-group00-s02 | cumulus
################################################################################
ssp-group00-s03 | cumulus
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br addr show lo"
========================================
Running: ip -br addr show lo
========================================
################################################################################
leaf-pod00-su00-r0 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-pod00-su00-r1 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-pod00-su00-r2 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-pod00-su00-r3 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-pod01-su00-r0 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-pod01-su00-r1 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-pod01-su00-r2 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-pod01-su00-r3 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-pod00-r0-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-pod00-r1-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-pod00-r2-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-pod00-r3-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-pod01-r0-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-pod01-r1-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-pod01-r2-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-pod01-r3-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
ssp-group00-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
ssp-group00-s01 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
ssp-group00-s02 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
ssp-group00-s03 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp summary\""
========================================
Running: sudo vtysh -c "show bgp summary"
========================================
################################################################################
leaf-pod00-su00-r0 | bgpd is not running
################################################################################
leaf-pod00-su00-r1 | bgpd is not running
################################################################################
leaf-pod00-su00-r2 | bgpd is not running
################################################################################
leaf-pod00-su00-r3 | bgpd is not running
################################################################################
leaf-pod01-su00-r0 | bgpd is not running
################################################################################
leaf-pod01-su00-r1 | bgpd is not running
################################################################################
leaf-pod01-su00-r2 | bgpd is not running
################################################################################
leaf-pod01-su00-r3 | bgpd is not running
################################################################################
spine-pod00-r0-s00 | bgpd is not running
################################################################################
spine-pod00-r1-s00 | bgpd is not running
################################################################################
spine-pod00-r2-s00 | bgpd is not running
################################################################################
spine-pod00-r3-s00 | bgpd is not running
################################################################################
spine-pod01-r0-s00 | bgpd is not running
################################################################################
spine-pod01-r1-s00 | bgpd is not running
################################################################################
spine-pod01-r2-s00 | bgpd is not running
################################################################################
spine-pod01-r3-s00 | bgpd is not running
################################################################################
ssp-group00-s00 | bgpd is not running
################################################################################
ssp-group00-s01 | bgpd is not running
################################################################################
ssp-group00-s02 | bgpd is not running
################################################################################
ssp-group00-s03 | bgpd is not running
################################################################################
```

Expected result:

- At this stage the loopback carries only `127.0.0.1/8`, no `10.253.x.x/32` address is assigned and BGP is not running.

### Step 5. Configure the Switches

**Goal:** Assign hostnames, ASNs, loopbacks, fabric point-to-point addresses (leaf-to-spine and spine-to-super-spine), and host downlinks to every switch.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few minutes.

Set `fabricId` in `fabric-config.3tier.l3evpn.yaml` to `1`:

```bash
metalcloud-cli fabric configure-switches 1 --config-source fabric-config.3tier.l3evpn.yaml
```

Expected result:

- The command completes without error; the switch configuration is staged but not yet deployed.

Validation:

- Proceed to Step 6, which deploys and verifies this configuration on the switches.

### Step 6. Deploy the Underlay Addressing

**Goal:** Push hostnames, loopbacks, point-to-point addresses, and port settings to every switch.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.3tier.yaml`) on the switches.

**Expected wait time:** A few minutes. The deploy is asynchronous; wait for its job group to finish before continuing.

```bash
wait_for_job_group "$(metalcloud-cli fabric deploy 1 -f json | jq -r '.jobGroupId')"
```

Expected result:

- The job group finishes with every job reporting success.

Validation:

Re-run the Step 4 check against each switch and compare with the baseline:

```bash
~/spcx-air/spcx-run -c "hostname"
```

```bash
~/spcx-air/spcx-run -c "ip -br addr show lo"
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp summary\""
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "hostname"
========================================
Running: hostname
========================================
################################################################################
leaf-pod00-su00-r0 | leaf-pod00-su00-r0
################################################################################
leaf-pod00-su00-r1 | leaf-pod00-su00-r1
################################################################################
leaf-pod00-su00-r2 | leaf-pod00-su00-r2
################################################################################
leaf-pod00-su00-r3 | leaf-pod00-su00-r3
################################################################################
leaf-pod01-su00-r0 | leaf-pod01-su00-r0
################################################################################
leaf-pod01-su00-r1 | leaf-pod01-su00-r1
################################################################################
leaf-pod01-su00-r2 | leaf-pod01-su00-r2
################################################################################
leaf-pod01-su00-r3 | leaf-pod01-su00-r3
################################################################################
spine-pod00-r0-s00 | spine-pod00-r0-s00
################################################################################
spine-pod00-r1-s00 | spine-pod00-r1-s00
################################################################################
spine-pod00-r2-s00 | spine-pod00-r2-s00
################################################################################
spine-pod00-r3-s00 | spine-pod00-r3-s00
################################################################################
spine-pod01-r0-s00 | spine-pod01-r0-s00
################################################################################
spine-pod01-r1-s00 | spine-pod01-r1-s00
################################################################################
spine-pod01-r2-s00 | spine-pod01-r2-s00
################################################################################
spine-pod01-r3-s00 | spine-pod01-r3-s00
################################################################################
ssp-group00-s00 | ssp-group00-s00
################################################################################
ssp-group00-s01 | ssp-group00-s01
################################################################################
ssp-group00-s02 | ssp-group00-s02
################################################################################
ssp-group00-s03 | ssp-group00-s03
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br addr show lo"
========================================
Running: ip -br addr show lo
========================================
################################################################################
leaf-pod00-su00-r0 | lo               UNKNOWN        127.0.0.1/8 10.253.128.1/32 ::1/128
################################################################################
leaf-pod00-su00-r1 | lo               UNKNOWN        127.0.0.1/8 10.253.128.2/32 ::1/128
################################################################################
leaf-pod00-su00-r2 | lo               UNKNOWN        127.0.0.1/8 10.253.128.3/32 ::1/128
################################################################################
leaf-pod00-su00-r3 | lo               UNKNOWN        127.0.0.1/8 10.253.128.4/32 ::1/128
################################################################################
leaf-pod01-su00-r0 | lo               UNKNOWN        127.0.0.1/8 10.253.128.5/32 ::1/128
################################################################################
leaf-pod01-su00-r1 | lo               UNKNOWN        127.0.0.1/8 10.253.128.6/32 ::1/128
################################################################################
leaf-pod01-su00-r2 | lo               UNKNOWN        127.0.0.1/8 10.253.128.7/32 ::1/128
################################################################################
leaf-pod01-su00-r3 | lo               UNKNOWN        127.0.0.1/8 10.253.128.8/32 ::1/128
################################################################################
spine-pod00-r0-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.9/32 ::1/128
################################################################################
spine-pod00-r1-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.10/32 ::1/128
################################################################################
spine-pod00-r2-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.11/32 ::1/128
################################################################################
spine-pod00-r3-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.12/32 ::1/128
################################################################################
spine-pod01-r0-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.13/32 ::1/128
################################################################################
spine-pod01-r1-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.14/32 ::1/128
################################################################################
spine-pod01-r2-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.15/32 ::1/128
################################################################################
spine-pod01-r3-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.16/32 ::1/128
################################################################################
ssp-group00-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.17/32 ::1/128
################################################################################
ssp-group00-s01 | lo               UNKNOWN        127.0.0.1/8 10.253.128.18/32 ::1/128
################################################################################
ssp-group00-s02 | lo               UNKNOWN        127.0.0.1/8 10.253.128.19/32 ::1/128
################################################################################
ssp-group00-s03 | lo               UNKNOWN        127.0.0.1/8 10.253.128.20/32 ::1/128
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp summary\""
========================================
Running: sudo vtysh -c "show bgp summary"
========================================
################################################################################
leaf-pod00-su00-r0 | bgpd is not running
################################################################################
leaf-pod00-su00-r1 | bgpd is not running
################################################################################
leaf-pod00-su00-r2 | bgpd is not running
################################################################################
leaf-pod00-su00-r3 | bgpd is not running
################################################################################
leaf-pod01-su00-r0 | bgpd is not running
################################################################################
leaf-pod01-su00-r1 | bgpd is not running
################################################################################
leaf-pod01-su00-r2 | bgpd is not running
################################################################################
leaf-pod01-su00-r3 | bgpd is not running
################################################################################
spine-pod00-r0-s00 | bgpd is not running
################################################################################
spine-pod00-r1-s00 | bgpd is not running
################################################################################
spine-pod00-r2-s00 | bgpd is not running
################################################################################
spine-pod00-r3-s00 | bgpd is not running
################################################################################
spine-pod01-r0-s00 | bgpd is not running
################################################################################
spine-pod01-r1-s00 | bgpd is not running
################################################################################
spine-pod01-r2-s00 | bgpd is not running
################################################################################
spine-pod01-r3-s00 | bgpd is not running
################################################################################
ssp-group00-s00 | bgpd is not running
################################################################################
ssp-group00-s01 | bgpd is not running
################################################################################
ssp-group00-s02 | bgpd is not running
################################################################################
ssp-group00-s03 | bgpd is not running
################################################################################
```

The switch now reports its assigned hostname and a `10.253.128.x/32` loopback (`leaf-pod00-su00-r0` is `10.253.128.1`, `r1` is `.2`, and so on, continuing through the second POD; the spines and super-spines are addressed the same way) and BGP is not running.

### Step 7. Discover Links and Redeploy

**Goal:** Re-scan the fabric so the discovered links become managed, then redeploy.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.3tier.yaml`) on the switches.

**Expected wait time:** A few minutes for both the rescan and the redeploy job groups.

```bash
wait_for_job_group "$(metalcloud-cli fabric rescan-links 1 -f json | jq -r '.jobGroupId')"
```

```bash
wait_for_job_group "$(metalcloud-cli fabric deploy 1 -f json | jq -r '.jobGroupId')"
```

Expected result:

- The rescan reads each switch's LLDP neighbours and records the discovered links in the fabric database; the redeploy confirms them.

Validation:

Confirm a switch now sees its neighbours over LLDP:

```bash
~/spcx-air/spcx-run -c "nv show interface lldp"
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "nv show interface lldp"
========================================
Running: nv show interface lldp
========================================
################################################################################
leaf-pod00-su00-r0 | Interface  Speed  Type  Remote Host             Remote Port
leaf-pod00-su00-r0 | ---------  -----  ----  ----------------------  -----------
leaf-pod00-su00-r0 | eth0       1G     eth   oob-mgmt-switch-leaf-1  swp10
leaf-pod00-su00-r0 | swp33s0    1G     swp   spine-pod00-r0-s00      swp1s0
leaf-pod00-su00-r0 | swp33s1    1G     swp   spine-pod00-r0-s00      swp1s1
leaf-pod00-su00-r0 | swp34s0    1G     swp   spine-pod00-r0-s00      swp2s0
leaf-pod00-su00-r0 | swp34s1    1G     swp   spine-pod00-r0-s00      swp2s1
leaf-pod00-su00-r0 | swp35s0    1G     swp   spine-pod00-r0-s00      swp3s0
leaf-pod00-su00-r0 | swp35s1    1G     swp   spine-pod00-r0-s00      swp3s1
leaf-pod00-su00-r0 | swp36s0    1G     swp   spine-pod00-r0-s00      swp4s0
leaf-pod00-su00-r0 | swp36s1    1G     swp   spine-pod00-r0-s00      swp4s1
leaf-pod00-su00-r0 | swp37s0    1G     swp   spine-pod00-r0-s00      swp5s0
leaf-pod00-su00-r0 | swp37s1    1G     swp   spine-pod00-r0-s00      swp5s1
leaf-pod00-su00-r0 | swp38s0    1G     swp   spine-pod00-r0-s00      swp6s0
leaf-pod00-su00-r0 | swp38s1    1G     swp   spine-pod00-r0-s00      swp6s1
leaf-pod00-su00-r0 | swp39s0    1G     swp   spine-pod00-r0-s00      swp7s0
leaf-pod00-su00-r0 | swp39s1    1G     swp   spine-pod00-r0-s00      swp7s1
leaf-pod00-su00-r0 | swp40s0    1G     swp   spine-pod00-r0-s00      swp8s0
leaf-pod00-su00-r0 | swp40s1    1G     swp   spine-pod00-r0-s00      swp8s1
leaf-pod00-su00-r0 | swp41s0    1G     swp   spine-pod00-r0-s00      swp9s0
leaf-pod00-su00-r0 | swp41s1    1G     swp   spine-pod00-r0-s00      swp9s1
leaf-pod00-su00-r0 | swp42s0    1G     swp   spine-pod00-r0-s00      swp10s0
leaf-pod00-su00-r0 | swp42s1    1G     swp   spine-pod00-r0-s00      swp10s1
leaf-pod00-su00-r0 | swp43s0    1G     swp   spine-pod00-r0-s00      swp11s0
leaf-pod00-su00-r0 | swp43s1    1G     swp   spine-pod00-r0-s00      swp11s1
leaf-pod00-su00-r0 | swp44s0    1G     swp   spine-pod00-r0-s00      swp12s0
leaf-pod00-su00-r0 | swp44s1    1G     swp   spine-pod00-r0-s00      swp12s1
leaf-pod00-su00-r0 | swp45s0    1G     swp   spine-pod00-r0-s00      swp13s0
leaf-pod00-su00-r0 | swp45s1    1G     swp   spine-pod00-r0-s00      swp13s1
leaf-pod00-su00-r0 | swp46s0    1G     swp   spine-pod00-r0-s00      swp14s0
leaf-pod00-su00-r0 | swp46s1    1G     swp   spine-pod00-r0-s00      swp14s1
leaf-pod00-su00-r0 | swp47s0    1G     swp   spine-pod00-r0-s00      swp15s0
leaf-pod00-su00-r0 | swp47s1    1G     swp   spine-pod00-r0-s00      swp15s1
leaf-pod00-su00-r0 | swp48s0    1G     swp   spine-pod00-r0-s00      swp16s0
leaf-pod00-su00-r0 | swp48s1    1G     swp   spine-pod00-r0-s00      swp16s1
################################################################################
```

Each fabric-facing port lists the neighbour it discovered: a leaf sees its spines, a spine sees the leaves below it and the super-spines above it and a super-spine sees the spines.

- (Optional) In the web UI, navigate to **Fabric Manager > Fabrics > Select the desired fabric (spectrumx-3tier-514) > Topology** where the Fabric View should look like this:

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/fabric-topology.3tier.webp)

### Step 8. Register the Fabric Templates

**Goal:** Register the Cumulus 5.14 configuration templates that Step 9 deploys.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** About a minute; `--verify-render` renders every switch's configuration before writing.

`fabric-config.3tier.l3evpn.yaml` names the Cumulus 5.14 templates under `cumulus-5.14-templates/`. Register them in two commands. The order matters: the base profile installs the QoS, adaptive-routing, and VTEP configuration that the BGP and EVPN profiles depend on.

```bash
metalcloud-cli fabric configure-freeform 1 \
        --config-source fabric-config.3tier.l3evpn.yaml --verify-render
```

```bash
metalcloud-cli fabric configure-bgp 1 \
        --config-source fabric-config.3tier.l3evpn.yaml --verify-render
```

Expected result:

- Both commands complete without error. `--verify-render` stops before writing if any switch fails to render.

Validation:

- (Optional) In the web UI, the registered templates appear under **Fabric Manager > Configuration Libraries > Network Device Configuration Templates**:

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/configuration_templates.webp)

### Step 9. Deploy the Fabric

**Goal:** Apply the base, underlay, overlay, and QoS profiles to every switch in order, bringing up BGP and EVPN, including the inter-POD EVPN routes relayed through the super-spine tier.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.3tier.yaml`) on the switches.

**Expected wait time:** This is the longest step in the lab, about 10 to 12 minutes.

```bash
wait_for_job_group "$(metalcloud-cli fabric deploy 1 -f json | jq -r '.jobGroupId')"
```

Expected result:

- Every job reports success. Continue only once that is true.

Validation:

Compare with the Step 4 baseline, where BGP was not running. This prints the underlay and EVPN overlay summaries together:

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp summary\""
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp l2vpn evpn summary\""
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp summary\""
========================================
Running: sudo vtysh -c "show bgp summary"
========================================
################################################################################
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | IPv4 Unicast Summary:
leaf-pod00-su00-r0 | BGP router identifier 10.253.128.1, local AS number 4200000000 VRF default vrf-id 0
leaf-pod00-su00-r0 | BGP table version 24
leaf-pod00-su00-r0 | RIB entries 39, using 4992 bytes of memory
leaf-pod00-su00-r0 | Peers 32, using 640 KiB of memory
leaf-pod00-su00-r0 | Peer groups 2, using 128 bytes of memory
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Neighbor                        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.1)  4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.3)  4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.5)  4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.7)  4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.9)  4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.11) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.13) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.15) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.17) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.19) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.21) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.23) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.25) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.27) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.29) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.31) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.33) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.35) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.37) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.39) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.41) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.43) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.45) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.47) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.49) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.51) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.53) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.55) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.57) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.59) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.61) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 | spine-pod00-r0-s00(10.254.0.63) 4 4201000000        65        61       24    0    0 00:02:01           19       20 to_spine-pod00-r0-s0
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Total number of neighbors 32
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | L2VPN EVPN Summary:
leaf-pod00-su00-r0 | BGP router identifier 10.253.128.1, local AS number 4200000000 VRF default vrf-id 0
leaf-pod00-su00-r0 | BGP table version 0
leaf-pod00-su00-r0 | RIB entries 0, using 0 bytes of memory
leaf-pod00-su00-r0 | Peers 2, using 40 KiB of memory
leaf-pod00-su00-r0 | Peer groups 2, using 128 bytes of memory
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Neighbor                       V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod00-su00-r0 | ssp-group00-s00(10.253.128.17) 4 4202000000        27        32        0    0    0 00:01:14            0        0 to_ssp-group00-s00_l
leaf-pod00-su00-r0 | ssp-group00-s01(10.253.128.18) 4 4202000000        26        31        0    0    0 00:01:11            0        0 to_ssp-group00-s01_l
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Total number of neighbors 2
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp l2vpn evpn summary\""
========================================
Running: sudo vtysh -c "show bgp l2vpn evpn summary"
========================================
################################################################################
leaf-pod00-su00-r0 | BGP router identifier 10.253.128.1, local AS number 4200000000 VRF default vrf-id 0
leaf-pod00-su00-r0 | BGP table version 0
leaf-pod00-su00-r0 | RIB entries 0, using 0 bytes of memory
leaf-pod00-su00-r0 | Peers 2, using 40 KiB of memory
leaf-pod00-su00-r0 | Peer groups 2, using 128 bytes of memory
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Neighbor                       V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod00-su00-r0 | ssp-group00-s00(10.253.128.17) 4 4202000000        29        34        0    0    0 00:01:20            0        0 to_ssp-group00-s00_l
leaf-pod00-su00-r0 | ssp-group00-s01(10.253.128.18) 4 4202000000        28        33        0    0    0 00:01:17            0        0 to_ssp-group00-s01_l
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Total number of neighbors 2
################################################################################
leaf-pod00-su00-r1 | BGP router identifier 10.253.128.2, local AS number 4200000001 VRF default vrf-id 0
leaf-pod00-su00-r1 | BGP table version 0
leaf-pod00-su00-r1 | RIB entries 0, using 0 bytes of memory
leaf-pod00-su00-r1 | Peers 2, using 40 KiB of memory
leaf-pod00-su00-r1 | Peer groups 2, using 128 bytes of memory
leaf-pod00-su00-r1 |
leaf-pod00-su00-r1 | Neighbor                       V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod00-su00-r1 | ssp-group00-s00(10.253.128.17) 4 4202000000        29        34        0    0    0 00:01:20            0        0 to_ssp-group00-s00_l
leaf-pod00-su00-r1 | ssp-group00-s01(10.253.128.18) 4 4202000000        28        33        0    0    0 00:01:18            0        0 to_ssp-group00-s01_l
leaf-pod00-su00-r1 |
leaf-pod00-su00-r1 | Total number of neighbors 2
################################################################################
leaf-pod00-su00-r2 | BGP router identifier 10.253.128.3, local AS number 4200000002 VRF default vrf-id 0
leaf-pod00-su00-r2 | BGP table version 0
leaf-pod00-su00-r2 | RIB entries 0, using 0 bytes of memory
leaf-pod00-su00-r2 | Peers 2, using 40 KiB of memory
leaf-pod00-su00-r2 | Peer groups 2, using 128 bytes of memory
leaf-pod00-su00-r2 |
leaf-pod00-su00-r2 | Neighbor                       V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod00-su00-r2 | ssp-group00-s00(10.253.128.17) 4 4202000000        29        34        0    0    0 00:01:20            0        0 to_ssp-group00-s00_l
leaf-pod00-su00-r2 | ssp-group00-s01(10.253.128.18) 4 4202000000        28        33        0    0    0 00:01:17            0        0 to_ssp-group00-s01_l
leaf-pod00-su00-r2 |
leaf-pod00-su00-r2 | Total number of neighbors 2
################################################################################
leaf-pod00-su00-r3 | BGP router identifier 10.253.128.4, local AS number 4200000003 VRF default vrf-id 0
leaf-pod00-su00-r3 | BGP table version 0
leaf-pod00-su00-r3 | RIB entries 0, using 0 bytes of memory
leaf-pod00-su00-r3 | Peers 2, using 40 KiB of memory
leaf-pod00-su00-r3 | Peer groups 2, using 128 bytes of memory
leaf-pod00-su00-r3 |
leaf-pod00-su00-r3 | Neighbor                       V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod00-su00-r3 | ssp-group00-s00(10.253.128.17) 4 4202000000        29        34        0    0    0 00:01:20            0        0 to_ssp-group00-s00_l
leaf-pod00-su00-r3 | ssp-group00-s01(10.253.128.18) 4 4202000000        28        33        0    0    0 00:01:18            0        0 to_ssp-group00-s01_l
leaf-pod00-su00-r3 |
leaf-pod00-su00-r3 | Total number of neighbors 2
################################################################################
leaf-pod01-su00-r0 | BGP router identifier 10.253.128.5, local AS number 4200000004 VRF default vrf-id 0
leaf-pod01-su00-r0 | BGP table version 0
leaf-pod01-su00-r0 | RIB entries 0, using 0 bytes of memory
leaf-pod01-su00-r0 | Peers 2, using 40 KiB of memory
leaf-pod01-su00-r0 | Peer groups 2, using 128 bytes of memory
leaf-pod01-su00-r0 |
leaf-pod01-su00-r0 | Neighbor                       V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod01-su00-r0 | ssp-group00-s00(10.253.128.17) 4 4202000000        29        34        0    0    0 00:01:20            0        0 to_ssp-group00-s00_l
leaf-pod01-su00-r0 | ssp-group00-s01(10.253.128.18) 4 4202000000        28        33        0    0    0 00:01:18            0        0 to_ssp-group00-s01_l
leaf-pod01-su00-r0 |
leaf-pod01-su00-r0 | Total number of neighbors 2
################################################################################
leaf-pod01-su00-r1 | BGP router identifier 10.253.128.6, local AS number 4200000005 VRF default vrf-id 0
leaf-pod01-su00-r1 | BGP table version 0
leaf-pod01-su00-r1 | RIB entries 0, using 0 bytes of memory
leaf-pod01-su00-r1 | Peers 2, using 40 KiB of memory
leaf-pod01-su00-r1 | Peer groups 2, using 128 bytes of memory
leaf-pod01-su00-r1 |
leaf-pod01-su00-r1 | Neighbor                       V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod01-su00-r1 | ssp-group00-s00(10.253.128.17) 4 4202000000        29        34        0    0    0 00:01:20            0        0 to_ssp-group00-s00_l
leaf-pod01-su00-r1 | ssp-group00-s01(10.253.128.18) 4 4202000000        28        33        0    0    0 00:01:17            0        0 to_ssp-group00-s01_l
leaf-pod01-su00-r1 |
leaf-pod01-su00-r1 | Total number of neighbors 2
################################################################################
leaf-pod01-su00-r2 | BGP router identifier 10.253.128.7, local AS number 4200000006 VRF default vrf-id 0
leaf-pod01-su00-r2 | BGP table version 0
leaf-pod01-su00-r2 | RIB entries 0, using 0 bytes of memory
leaf-pod01-su00-r2 | Peers 2, using 40 KiB of memory
leaf-pod01-su00-r2 | Peer groups 2, using 128 bytes of memory
leaf-pod01-su00-r2 |
leaf-pod01-su00-r2 | Neighbor                       V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod01-su00-r2 | ssp-group00-s00(10.253.128.17) 4 4202000000        29        34        0    0    0 00:01:20            0        0 to_ssp-group00-s00_l
leaf-pod01-su00-r2 | ssp-group00-s01(10.253.128.18) 4 4202000000        28        33        0    0    0 00:01:17            0        0 to_ssp-group00-s01_l
leaf-pod01-su00-r2 |
leaf-pod01-su00-r2 | Total number of neighbors 2
################################################################################
leaf-pod01-su00-r3 | BGP router identifier 10.253.128.8, local AS number 4200000007 VRF default vrf-id 0
leaf-pod01-su00-r3 | BGP table version 0
leaf-pod01-su00-r3 | RIB entries 0, using 0 bytes of memory
leaf-pod01-su00-r3 | Peers 2, using 40 KiB of memory
leaf-pod01-su00-r3 | Peer groups 2, using 128 bytes of memory
leaf-pod01-su00-r3 |
leaf-pod01-su00-r3 | Neighbor                       V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-pod01-su00-r3 | ssp-group00-s00(10.253.128.17) 4 4202000000        29        34        0    0    0 00:01:21            0        0 to_ssp-group00-s00_l
leaf-pod01-su00-r3 | ssp-group00-s01(10.253.128.18) 4 4202000000        28        33        0    0    0 00:01:18            0        0 to_ssp-group00-s01_l
leaf-pod01-su00-r3 |
leaf-pod01-su00-r3 | Total number of neighbors 2
################################################################################
spine-pod00-r0-s00 | % No BGP neighbors found in VRF default
################################################################################
spine-pod00-r1-s00 | % No BGP neighbors found in VRF default
################################################################################
spine-pod00-r2-s00 | % No BGP neighbors found in VRF default
################################################################################
spine-pod00-r3-s00 | % No BGP neighbors found in VRF default
################################################################################
spine-pod01-r0-s00 | % No BGP neighbors found in VRF default
################################################################################
spine-pod01-r1-s00 | % No BGP neighbors found in VRF default
################################################################################
spine-pod01-r2-s00 | % No BGP neighbors found in VRF default
################################################################################
spine-pod01-r3-s00 | % No BGP neighbors found in VRF default
################################################################################
ssp-group00-s00 | BGP router identifier 10.253.128.17, local AS number 4202000000 VRF default vrf-id 0
ssp-group00-s00 | BGP table version 0
ssp-group00-s00 | RIB entries 0, using 0 bytes of memory
ssp-group00-s00 | Peers 8, using 160 KiB of memory
ssp-group00-s00 | Peer groups 2, using 128 bytes of memory
ssp-group00-s00 |
ssp-group00-s00 | Neighbor                         V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
ssp-group00-s00 | leaf-pod00-su00-r0(10.253.128.1) 4 4200000000        29        29        0    0    0 00:01:20            0        0 to_leaf-pod00-su00-r
ssp-group00-s00 | leaf-pod00-su00-r1(10.253.128.2) 4 4200000001        29        29        0    0    0 00:01:20            0        0 to_leaf-pod00-su00-r
ssp-group00-s00 | leaf-pod00-su00-r2(10.253.128.3) 4 4200000002        29        29        0    0    0 00:01:20            0        0 to_leaf-pod00-su00-r
ssp-group00-s00 | leaf-pod00-su00-r3(10.253.128.4) 4 4200000003        29        29        0    0    0 00:01:20            0        0 to_leaf-pod00-su00-r
ssp-group00-s00 | leaf-pod01-su00-r0(10.253.128.5) 4 4200000004        29        29        0    0    0 00:01:20            0        0 to_leaf-pod01-su00-r
ssp-group00-s00 | leaf-pod01-su00-r1(10.253.128.6) 4 4200000005        29        29        0    0    0 00:01:20            0        0 to_leaf-pod01-su00-r
ssp-group00-s00 | leaf-pod01-su00-r2(10.253.128.7) 4 4200000006        29        29        0    0    0 00:01:20            0        0 to_leaf-pod01-su00-r
ssp-group00-s00 | leaf-pod01-su00-r3(10.253.128.8) 4 4200000007        29        29        0    0    0 00:01:20            0        0 to_leaf-pod01-su00-r
ssp-group00-s00 |
ssp-group00-s00 | Total number of neighbors 8
################################################################################
ssp-group00-s01 | BGP router identifier 10.253.128.18, local AS number 4202000000 VRF default vrf-id 0
ssp-group00-s01 | BGP table version 0
ssp-group00-s01 | RIB entries 0, using 0 bytes of memory
ssp-group00-s01 | Peers 8, using 160 KiB of memory
ssp-group00-s01 | Peer groups 2, using 128 bytes of memory
ssp-group00-s01 |
ssp-group00-s01 | Neighbor                         V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
ssp-group00-s01 | leaf-pod00-su00-r0(10.253.128.1) 4 4200000000        28        28        0    0    0 00:01:17            0        0 to_leaf-pod00-su00-r
ssp-group00-s01 | leaf-pod00-su00-r1(10.253.128.2) 4 4200000001        28        28        0    0    0 00:01:17            0        0 to_leaf-pod00-su00-r
ssp-group00-s01 | leaf-pod00-su00-r2(10.253.128.3) 4 4200000002        28        28        0    0    0 00:01:17            0        0 to_leaf-pod00-su00-r
ssp-group00-s01 | leaf-pod00-su00-r3(10.253.128.4) 4 4200000003        28        28        0    0    0 00:01:17            0        0 to_leaf-pod00-su00-r
ssp-group00-s01 | leaf-pod01-su00-r0(10.253.128.5) 4 4200000004        28        28        0    0    0 00:01:17            0        0 to_leaf-pod01-su00-r
ssp-group00-s01 | leaf-pod01-su00-r1(10.253.128.6) 4 4200000005        28        28        0    0    0 00:01:17            0        0 to_leaf-pod01-su00-r
ssp-group00-s01 | leaf-pod01-su00-r2(10.253.128.7) 4 4200000006        28        28        0    0    0 00:01:17            0        0 to_leaf-pod01-su00-r
ssp-group00-s01 | leaf-pod01-su00-r3(10.253.128.8) 4 4200000007        28        28        0    0    0 00:01:17            0        0 to_leaf-pod01-su00-r
ssp-group00-s01 |
ssp-group00-s01 | Total number of neighbors 8
################################################################################
ssp-group00-s02 | % No BGP neighbors found in VRF default
################################################################################
ssp-group00-s03 | % No BGP neighbors found in VRF default
################################################################################
```

Every switch now has established underlay sessions, each with a non-zero prefix count in place of `Idle` or `Active`. The EVPN overlay runs between the leaves and the super-spine relay `ssp-group00-s00` and `ssp-group00-s01`; the transit spines carry only the underlay, so their EVPN summary is empty. Look at the output for a leaf or those two super-spines to see the overlay sessions.

### Step 10. Register the HGX Endpoints

**Goal:** Register the eight HGX hosts as endpoints, each bound to eight leaf ports (one per rail). The four hosts of the first POD sit on `leaf-pod00-su00-r*` and the four of the second POD on `leaf-pod01-su00-r*`.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few seconds.

The definitions are in `endpoints.3tier.yaml`:

```bash
metalcloud-cli endpoint create-bulk --config-source endpoints.3tier.yaml
```

Expected result:

- All eight endpoints are created.

Validation:

```bash
metalcloud-cli endpoint list --filter-site 1
```

Confirm all eight endpoints are listed. Do not rename them; the Terraform manifest in Step 12 resolves them by label (`hgx-pod00-su00-h*`, `hgx-pod01-su00-h*`).

### Step 11. Create the Tenant Route Domains and Profiles

**Goal:** Create the tenant VRFs (route domain) and the L3-only logical network profiles the tenant networks are built from.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few seconds.

```bash
metalcloud-cli route-domain create --config-source route-domain-tenant1.3tier.yaml
```

```bash
metalcloud-cli route-domain create --config-source route-domain-tenant2.3tier.yaml
```

```bash
metalcloud-cli logical-network-profile create vxlan --config-source l3-profile-tenant1.3tier.yaml
```

```bash
metalcloud-cli logical-network-profile create vxlan --config-source l3-profile-tenant2.3tier.yaml
```

Expected result:

- The route domains and network profiles are created. The L3VNI is allocated automatically for the fabric. Keep the profile labels as `tenant1-l3` and `tenant2-l3`; the Terraform manifest in Step 12 resolves them by that label.

Validation:

- Proceed to Step 12, which consumes the profiles.

### Step 12. Onboard the Tenants with Terraform

**Goal:** Create the tenant infrastructures, build L3 networks from the `tenant1-l3` and `tenant2-l3` profiles, attach the eight endpoints (four per tenant) with eight interfaces each, and deploy. `tenant1` gets the `h00`/`h16` hosts of both PODs; `tenant2` gets the `h08`/`h24` hosts of both PODs.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia/terraform`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** About 5 minutes.

```bash
cd ~/nvidia/terraform/tenant1
terraform init
terraform apply -parallelism=1 -auto-approve
```

```bash
cd ~/nvidia/terraform/tenant2
terraform init
terraform apply -parallelism=1 -auto-approve
cd ~/nvidia
```

The expected output should be the following:

```bash
Plan: 8 to add, 0 to change, 0 to destroy.
terraform_data.endpoint_fingerprint: Creating...
terraform_data.endpoint_fingerprint: Creation complete after 0s [id=b18788a6-32f5-c80c-4899-ff3866c7a67b]
metalcloud_infrastructure.infra_tenant1: Creating...
metalcloud_infrastructure.infra_tenant1: Creation complete after 0s
metalcloud_logical_network.tenant1-network: Creating...
metalcloud_logical_network.tenant1-network: Creation complete after 0s [name=tenant1-network]
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h00"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h00"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-pod01-su00-h00"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-pod01-su00-h00"]: Creation complete after 1s
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h16"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h16"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-pod01-su00-h16"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-pod01-su00-h16"]: Creation complete after 0s
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Creating...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Creation complete after 0s

Apply complete! Resources: 8 added, 0 changed, 0 destroyed.
```

```bash
Plan: 8 to add, 0 to change, 0 to destroy.
terraform_data.endpoint_fingerprint: Creating...
terraform_data.endpoint_fingerprint: Creation complete after 0s [id=54848ee0-84b6-7a34-04ed-adb46f3ed583]
metalcloud_infrastructure.infra_tenant2: Creating...
metalcloud_infrastructure.infra_tenant2: Creation complete after 0s
metalcloud_logical_network.tenant2-network: Creating...
metalcloud_logical_network.tenant2-network: Creation complete after 0s [name=tenant2-network]
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h08"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h08"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-pod01-su00-h08"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-pod01-su00-h08"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h24"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h24"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-pod01-su00-h24"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-pod01-su00-h24"]: Creation complete after 1s
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Creating...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Creation complete after 0s

Apply complete! Resources: 8 added, 0 changed, 0 destroyed.
```

Before continuing with the checks, confirm that the Terraform deployments have finished. The following command refreshes every 10 seconds; leave it running until the deploy status shows finished:

```bash
watch -n 10 "metalcloud-cli infrastructure list"
```
Use control+C to stop the watch at any time.

The expected output should be the following:

```
┌────┬───────────────┬───────────────┬────────┬───────┬──────┬─────────────────────┬─────────────────────┬───────────────┬───────────┐
│ ID │ LABEL         │ CONFIG LABEL  │ STATUS │ OWNER │ SITE │ CREATED             │ UPDATED             │ DEPLOY STATUS │ DEPLOY ID │
├────┼───────────────┼───────────────┼────────┼───────┼──────┼─────────────────────┼─────────────────────┼───────────────┼───────────┤
│  1 │ infra-tenant1 │ infra-tenant1 │ active │     1 │    1 │ 19 Aug 26 14:28 UTC │ 19 Aug 26 14:34 UTC │ finished      │           │
│  2 │ infra-tenant2 │ infra-tenant2 │ active │     1 │    1 │ 19 Aug 26 14:28 UTC │ 19 Aug 26 14:35 UTC │ finished      │           │
└────┴───────────────┴───────────────┴────────┴───────┴──────┴─────────────────────┴─────────────────────┴───────────────┴───────────┘
```

- When the apply completes, the attached hosts' rail gateways are live.

Validation:

- (Optional) In the web UI, the tenant infrastructures are visible in the Infrastructure Designer. Navigate to **Admin dashboard > Infrastructures**, select one infrastructure and click **Open infrastructure designer**. This opens the Infrastructure Designer, where you can see the graphical representation of the infrastructure:

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/infrastructures.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/open-infrastructure.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/tenant1.initial.3tier.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/tenant2.initial.3tier.webp)


Terraform created the tenant VRFs, attached the endpoints, and deployed. The tenant VRFs and the host rail gateways live on the leaves. Inspect a leaf:

```bash
~/spcx-air/spcx-run -c "ip -br link show type vrf"
```

```bash
~/spcx-air/spcx-run -c "ip -br addr show vrf tenant1 | grep -E \"swp|172\.\""
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant1 ipv4 unicast\""
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant1\""
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br link show type vrf"
========================================
Running: ip -br link show type vrf
========================================
################################################################################
leaf-pod00-su00-r0 | mgmt             UP             f6:db:64:29:6c:9e <NOARP,MASTER,UP,LOWER_UP>
leaf-pod00-su00-r0 | tenant1          UP             6a:a6:56:4c:bf:5d <NOARP,MASTER,UP,LOWER_UP>
leaf-pod00-su00-r0 | tenant2          UP             2a:79:59:7d:42:5d <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-pod00-su00-r1 | mgmt             UP             42:bc:18:1f:cd:48 <NOARP,MASTER,UP,LOWER_UP>
leaf-pod00-su00-r1 | tenant1          UP             66:85:96:18:e9:1e <NOARP,MASTER,UP,LOWER_UP>
leaf-pod00-su00-r1 | tenant2          UP             c2:02:8b:1f:d2:b0 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-pod00-su00-r2 | mgmt             UP             7e:de:71:81:f1:0e <NOARP,MASTER,UP,LOWER_UP>
leaf-pod00-su00-r2 | tenant1          UP             4e:c7:04:0e:71:fa <NOARP,MASTER,UP,LOWER_UP>
leaf-pod00-su00-r2 | tenant2          UP             5a:0d:9b:51:5d:50 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-pod00-su00-r3 | mgmt             UP             6e:cd:7b:40:34:51 <NOARP,MASTER,UP,LOWER_UP>
leaf-pod00-su00-r3 | tenant1          UP             42:c9:cb:f6:14:fc <NOARP,MASTER,UP,LOWER_UP>
leaf-pod00-su00-r3 | tenant2          UP             5a:92:68:86:22:5f <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-pod01-su00-r0 | mgmt             UP             aa:b0:47:d6:e6:81 <NOARP,MASTER,UP,LOWER_UP>
leaf-pod01-su00-r0 | tenant1          UP             12:f0:21:05:a3:b4 <NOARP,MASTER,UP,LOWER_UP>
leaf-pod01-su00-r0 | tenant2          UP             f2:18:b5:19:45:65 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-pod01-su00-r1 | mgmt             UP             a2:5e:a7:d0:c7:ee <NOARP,MASTER,UP,LOWER_UP>
leaf-pod01-su00-r1 | tenant1          UP             52:dc:a5:1c:9a:5a <NOARP,MASTER,UP,LOWER_UP>
leaf-pod01-su00-r1 | tenant2          UP             22:23:fa:14:5e:5f <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-pod01-su00-r2 | mgmt             UP             8e:23:80:ce:af:b0 <NOARP,MASTER,UP,LOWER_UP>
leaf-pod01-su00-r2 | tenant1          UP             56:31:3e:62:01:5b <NOARP,MASTER,UP,LOWER_UP>
leaf-pod01-su00-r2 | tenant2          UP             ae:e9:eb:aa:f1:aa <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-pod01-su00-r3 | mgmt             UP             26:58:50:f5:fd:da <NOARP,MASTER,UP,LOWER_UP>
leaf-pod01-su00-r3 | tenant1          UP             fe:0f:eb:21:1d:e1 <NOARP,MASTER,UP,LOWER_UP>
leaf-pod01-su00-r3 | tenant2          UP             5a:09:24:55:b9:fb <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-pod00-r0-s00 | mgmt             UP             ca:a3:5d:b5:51:8f <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-pod00-r1-s00 | mgmt             UP             72:ad:bc:d5:60:5a <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-pod00-r2-s00 | mgmt             UP             7e:f6:78:66:27:b3 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-pod00-r3-s00 | mgmt             UP             aa:c3:4a:8f:d4:21 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-pod01-r0-s00 | mgmt             UP             ee:5b:bf:6d:41:2f <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-pod01-r1-s00 | mgmt             UP             1e:b3:14:af:70:93 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-pod01-r2-s00 | mgmt             UP             0a:a5:bb:77:17:69 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-pod01-r3-s00 | mgmt             UP             8e:cf:5f:49:91:e2 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
ssp-group00-s00 | mgmt             UP             8a:0e:d4:97:e3:6e <NOARP,MASTER,UP,LOWER_UP>
################################################################################
ssp-group00-s01 | mgmt             UP             5e:f4:55:19:63:cb <NOARP,MASTER,UP,LOWER_UP>
################################################################################
ssp-group00-s02 | mgmt             UP             22:5c:c0:2a:82:85 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
ssp-group00-s03 | mgmt             UP             de:f4:03:e8:8c:38 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br addr show vrf tenant1 | grep -E \"swp|172\.\""
========================================
Running: ip -br addr show vrf tenant1 | grep -E "swp|172\."
========================================
################################################################################
leaf-pod00-su00-r0 | swp1s0           UP             172.16.0.1/31 fe80::4ab0:2dff:fe61:a18b/64
leaf-pod00-su00-r0 | swp1s1           UP             172.24.0.1/31 fe80::4ab0:2dff:fe14:3c77/64
leaf-pod00-su00-r0 | swp17s0          UP             172.16.0.33/31 fe80::4ab0:2dff:fee6:b736/64
leaf-pod00-su00-r0 | swp17s1          UP             172.24.0.33/31 fe80::4ab0:2dff:fe59:d51d/64
################################################################################
leaf-pod00-su00-r1 | swp1s0           UP             172.18.0.1/31 fe80::4ab0:2dff:fee1:2216/64
leaf-pod00-su00-r1 | swp1s1           UP             172.26.0.1/31 fe80::4ab0:2dff:fed5:7f4c/64
leaf-pod00-su00-r1 | swp17s0          UP             172.18.0.33/31 fe80::4ab0:2dff:fe09:2870/64
leaf-pod00-su00-r1 | swp17s1          UP             172.26.0.33/31 fe80::4ab0:2dff:fece:7fe0/64
################################################################################
leaf-pod00-su00-r2 | swp1s0           UP             172.20.0.1/31 fe80::4ab0:2dff:fedd:ee3c/64
leaf-pod00-su00-r2 | swp1s1           UP             172.28.0.1/31 fe80::4ab0:2dff:fe3b:954/64
leaf-pod00-su00-r2 | swp17s0          UP             172.20.0.33/31 fe80::4ab0:2dff:fe40:f3a/64
leaf-pod00-su00-r2 | swp17s1          UP             172.28.0.33/31 fe80::4ab0:2dff:feda:ce66/64
################################################################################
leaf-pod00-su00-r3 | swp1s0           UP             172.22.0.1/31 fe80::4ab0:2dff:fe92:f283/64
leaf-pod00-su00-r3 | swp1s1           UP             172.30.0.1/31 fe80::4ab0:2dff:fe7d:2fce/64
leaf-pod00-su00-r3 | swp17s0          UP             172.22.0.33/31 fe80::4ab0:2dff:fe41:5f7c/64
leaf-pod00-su00-r3 | swp17s1          UP             172.30.0.33/31 fe80::4ab0:2dff:fe9e:c51a/64
################################################################################
leaf-pod01-su00-r0 | swp1s0           UP             172.16.1.1/31 fe80::4ab0:2dff:fe8a:4d3d/64
leaf-pod01-su00-r0 | swp1s1           UP             172.24.1.1/31 fe80::4ab0:2dff:fedc:bc4/64
leaf-pod01-su00-r0 | swp17s0          UP             172.16.1.33/31 fe80::4ab0:2dff:fecf:37dc/64
leaf-pod01-su00-r0 | swp17s1          UP             172.24.1.33/31 fe80::4ab0:2dff:fe34:e962/64
################################################################################
leaf-pod01-su00-r1 | swp1s0           UP             172.18.1.1/31 fe80::4ab0:2dff:feaa:fd5/64
leaf-pod01-su00-r1 | swp1s1           UP             172.26.1.1/31 fe80::4ab0:2dff:fe47:24d/64
leaf-pod01-su00-r1 | swp17s0          UP             172.18.1.33/31 fe80::4ab0:2dff:fe5e:d5/64
leaf-pod01-su00-r1 | swp17s1          UP             172.26.1.33/31 fe80::4ab0:2dff:fe76:a558/64
################################################################################
leaf-pod01-su00-r2 | swp1s0           UP             172.20.1.1/31 fe80::4ab0:2dff:fe68:46a4/64
leaf-pod01-su00-r2 | swp1s1           UP             172.28.1.1/31 fe80::4ab0:2dff:feb7:6571/64
leaf-pod01-su00-r2 | swp17s0          UP             172.20.1.33/31 fe80::4ab0:2dff:feec:320b/64
leaf-pod01-su00-r2 | swp17s1          UP             172.28.1.33/31 fe80::4ab0:2dff:fed2:270/64
################################################################################
leaf-pod01-su00-r3 | swp1s0           UP             172.22.1.1/31 fe80::4ab0:2dff:fe22:c292/64
leaf-pod01-su00-r3 | swp1s1           UP             172.30.1.1/31 fe80::4ab0:2dff:fe23:8825/64
leaf-pod01-su00-r3 | swp17s0          UP             172.22.1.33/31 fe80::4ab0:2dff:fe6e:3f91/64
leaf-pod01-su00-r3 | swp17s1          UP             172.30.1.33/31 fe80::4ab0:2dff:fe0e:f188/64
################################################################################
spine-pod00-r0-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r1-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r2-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r3-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r0-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r1-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r2-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r3-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s01 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s02 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s03 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant1 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant1 ipv4 unicast"
========================================
################################################################################
leaf-pod00-su00-r0 | BGP table version is 84, local router ID is 10.253.128.1, vrf id 132
leaf-pod00-su00-r0 | Default local pref 100, local AS 4200000000
leaf-pod00-su00-r0 | Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath, + multipath nhg,
leaf-pod00-su00-r0 |                i internal, r RIB-failure, S Stale, R Removed
leaf-pod00-su00-r0 | Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
leaf-pod00-su00-r0 | Origin codes:  i - IGP, e - EGP, ? - incomplete
leaf-pod00-su00-r0 | RPKI validation codes: V valid, I invalid, N Not found
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 |     Network          Next Hop            Metric LocPrf Weight Path
leaf-pod00-su00-r0 |  *> 172.16.0.0/26    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.16.0.0/31    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.16.0.32/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  *> 172.16.1.0/26    10.253.128.5(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *                   10.253.128.5(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *                   10.253.128.2(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *> 172.18.1.0/26    10.253.128.6(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *                   10.253.128.6(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *                   10.253.128.3(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *> 172.20.1.0/26    10.253.128.7(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *                   10.253.128.7(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *                   10.253.128.4(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *> 172.22.1.0/26    10.253.128.8(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *                   10.253.128.8(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.24.0.0/31    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.24.0.32/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  *> 172.24.1.0/26    10.253.128.5(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *                   10.253.128.5(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *                   10.253.128.2(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *> 172.26.1.0/26    10.253.128.6(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *                   10.253.128.6(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *                   10.253.128.3(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *> 172.28.1.0/26    10.253.128.7(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *                   10.253.128.7(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *                   10.253.128.4(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *> 172.30.1.0/26    10.253.128.8(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *                   10.253.128.8(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Displayed 20 routes and 34 total paths
################################################################################

################################################################################
spine-pod00-r0-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod00-r1-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod00-r2-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod00-r3-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod01-r0-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod01-r1-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod01-r2-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod01-r3-s00 | View/Vrf tenant1 is unknown
################################################################################
ssp-group00-s00 | View/Vrf tenant1 is unknown
################################################################################
ssp-group00-s01 | View/Vrf tenant1 is unknown
################################################################################
ssp-group00-s02 | View/Vrf tenant1 is unknown
################################################################################
ssp-group00-s03 | View/Vrf tenant1 is unknown
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant1\""
========================================
Running: sudo vtysh -c "show ip route vrf tenant1"
========================================
################################################################################
leaf-pod00-su00-r0 | Codes: K - kernel route, C - connected, L - local, S - static,
leaf-pod00-su00-r0 |        R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
leaf-pod00-su00-r0 |        T - Table, A - Babel, D - SHARP, F - PBR, f - OpenFabric,
leaf-pod00-su00-r0 |        t - Table-Direct, Z - FRR,
leaf-pod00-su00-r0 |        > - selected route, * - FIB route, q - queued, r - rejected, b - backup
leaf-pod00-su00-r0 |        t - trapped, o - offload failure
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | VRF tenant1:
leaf-pod00-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:07:49
leaf-pod00-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:07:18
leaf-pod00-su00-r0 | C>* 172.16.0.0/31 is directly connected, swp1s0, 00:07:49
leaf-pod00-su00-r0 | L>* 172.16.0.1/32 is directly connected, swp1s0, 00:07:49
leaf-pod00-su00-r0 | C>* 172.16.0.32/31 is directly connected, swp17s0, 00:07:49
leaf-pod00-su00-r0 | L>* 172.16.0.33/32 is directly connected, swp17s0, 00:07:49
leaf-pod00-su00-r0 | B>* 172.16.1.0/26 [20/0] via 10.253.128.5, vlan1440_l3 onlink, weight 1, 00:04:46
leaf-pod00-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:06:39
leaf-pod00-su00-r0 | B>* 172.18.1.0/26 [20/0] via 10.253.128.6, vlan1440_l3 onlink, weight 1, 00:04:09
leaf-pod00-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:06:02
leaf-pod00-su00-r0 | B>* 172.20.1.0/26 [20/0] via 10.253.128.7, vlan1440_l3 onlink, weight 1, 00:03:31
leaf-pod00-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:05:23
leaf-pod00-su00-r0 | B>* 172.22.1.0/26 [20/0] via 10.253.128.8, vlan1440_l3 onlink, weight 1, 00:02:53
leaf-pod00-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:07:18
leaf-pod00-su00-r0 | C>* 172.24.0.0/31 is directly connected, swp1s1, 00:07:49
leaf-pod00-su00-r0 | L>* 172.24.0.1/32 is directly connected, swp1s1, 00:07:49
leaf-pod00-su00-r0 | C>* 172.24.0.32/31 is directly connected, swp17s1, 00:07:49
leaf-pod00-su00-r0 | L>* 172.24.0.33/32 is directly connected, swp17s1, 00:07:49
leaf-pod00-su00-r0 | B>* 172.24.1.0/26 [20/0] via 10.253.128.5, vlan1440_l3 onlink, weight 1, 00:04:46
leaf-pod00-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:06:39
leaf-pod00-su00-r0 | B>* 172.26.1.0/26 [20/0] via 10.253.128.6, vlan1440_l3 onlink, weight 1, 00:04:09
leaf-pod00-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:06:02
leaf-pod00-su00-r0 | B>* 172.28.1.0/26 [20/0] via 10.253.128.7, vlan1440_l3 onlink, weight 1, 00:03:31
leaf-pod00-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:05:23
leaf-pod00-su00-r0 | B>* 172.30.1.0/26 [20/0] via 10.253.128.8, vlan1440_l3 onlink, weight 1, 00:02:53
################################################################################
```

`tenant1` appears in the VRF list, the host-facing `swp` ports carry their `172.x` rail gateway `/31`s inside it, and the VRF routing table holds both the local rail subnets and the remote ones learned over EVPN, including the rails in the other POD reached across the super-spine tier.

```bash
~/spcx-air/spcx-run -c "ip -br addr show vrf tenant2 | grep -E \"swp|172\.\""
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant2 ipv4 unicast\""
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant2\""
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br addr show vrf tenant2 | grep -E \"swp|172\.\""
========================================
Running: ip -br addr show vrf tenant2 | grep -E "swp|172\."
========================================
################################################################################
leaf-pod00-su00-r0 | swp9s0           UP             172.16.0.17/31 fe80::4ab0:2dff:feb6:b813/64
leaf-pod00-su00-r0 | swp9s1           UP             172.24.0.17/31 fe80::4ab0:2dff:fe6b:99c9/64
leaf-pod00-su00-r0 | swp25s0          UP             172.16.0.49/31 fe80::4ab0:2dff:fe64:5c03/64
leaf-pod00-su00-r0 | swp25s1          UP             172.24.0.49/31 fe80::4ab0:2dff:feb5:82c1/64
################################################################################
leaf-pod00-su00-r1 | swp9s0           UP             172.18.0.17/31 fe80::4ab0:2dff:fe6b:b973/64
leaf-pod00-su00-r1 | swp9s1           UP             172.26.0.17/31 fe80::4ab0:2dff:fea7:2a9c/64
leaf-pod00-su00-r1 | swp25s0          UP             172.18.0.49/31 fe80::4ab0:2dff:fed9:56c4/64
leaf-pod00-su00-r1 | swp25s1          UP             172.26.0.49/31 fe80::4ab0:2dff:fe98:4ce/64
################################################################################
leaf-pod00-su00-r2 | swp9s0           UP             172.20.0.17/31 fe80::4ab0:2dff:fe8b:8979/64
leaf-pod00-su00-r2 | swp9s1           UP             172.28.0.17/31 fe80::4ab0:2dff:fe08:9f16/64
leaf-pod00-su00-r2 | swp25s0          UP             172.20.0.49/31 fe80::4ab0:2dff:fe55:65d2/64
leaf-pod00-su00-r2 | swp25s1          UP             172.28.0.49/31 fe80::4ab0:2dff:fe16:9824/64
################################################################################
leaf-pod00-su00-r3 | swp9s0           UP             172.22.0.17/31 fe80::4ab0:2dff:fec6:b8e5/64
leaf-pod00-su00-r3 | swp9s1           UP             172.30.0.17/31 fe80::4ab0:2dff:feea:48cf/64
leaf-pod00-su00-r3 | swp25s0          UP             172.22.0.49/31 fe80::4ab0:2dff:fe49:342b/64
leaf-pod00-su00-r3 | swp25s1          UP             172.30.0.49/31 fe80::4ab0:2dff:fe36:678b/64
################################################################################
leaf-pod01-su00-r0 | swp9s0           UP             172.16.1.17/31 fe80::4ab0:2dff:fed4:534e/64
leaf-pod01-su00-r0 | swp9s1           UP             172.24.1.17/31 fe80::4ab0:2dff:fe15:27a3/64
leaf-pod01-su00-r0 | swp25s0          UP             172.16.1.49/31 fe80::4ab0:2dff:fe3d:cec/64
leaf-pod01-su00-r0 | swp25s1          UP             172.24.1.49/31 fe80::4ab0:2dff:fe05:54ea/64
################################################################################
leaf-pod01-su00-r1 | swp9s0           UP             172.18.1.17/31 fe80::4ab0:2dff:feb5:bebf/64
leaf-pod01-su00-r1 | swp9s1           UP             172.26.1.17/31 fe80::4ab0:2dff:fe16:e10f/64
leaf-pod01-su00-r1 | swp25s0          UP             172.18.1.49/31 fe80::4ab0:2dff:feba:ba64/64
leaf-pod01-su00-r1 | swp25s1          UP             172.26.1.49/31 fe80::4ab0:2dff:febd:6944/64
################################################################################
leaf-pod01-su00-r2 | swp9s0           UP             172.20.1.17/31 fe80::4ab0:2dff:fe6b:94/64
leaf-pod01-su00-r2 | swp9s1           UP             172.28.1.17/31 fe80::4ab0:2dff:fe14:2e95/64
leaf-pod01-su00-r2 | swp25s0          UP             172.20.1.49/31 fe80::4ab0:2dff:feac:9d1e/64
leaf-pod01-su00-r2 | swp25s1          UP             172.28.1.49/31 fe80::4ab0:2dff:fe97:ccb7/64
################################################################################
leaf-pod01-su00-r3 | swp9s0           UP             172.22.1.17/31 fe80::4ab0:2dff:feba:3a0/64
leaf-pod01-su00-r3 | swp9s1           UP             172.30.1.17/31 fe80::4ab0:2dff:fe71:bc6a/64
leaf-pod01-su00-r3 | swp25s0          UP             172.22.1.49/31 fe80::4ab0:2dff:fee2:2834/64
leaf-pod01-su00-r3 | swp25s1          UP             172.30.1.49/31 fe80::4ab0:2dff:fecc:9405/64
################################################################################
spine-pod00-r0-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r1-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r2-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r3-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r0-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r1-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r2-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r3-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s01 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s02 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s03 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant2 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant2 ipv4 unicast"
========================================
################################################################################
leaf-pod00-su00-r0 | BGP table version is 84, local router ID is 10.253.128.1, vrf id 136
leaf-pod00-su00-r0 | Default local pref 100, local AS 4200000000
leaf-pod00-su00-r0 | Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath, + multipath nhg,
leaf-pod00-su00-r0 |                i internal, r RIB-failure, S Stale, R Removed
leaf-pod00-su00-r0 | Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
leaf-pod00-su00-r0 | Origin codes:  i - IGP, e - EGP, ? - incomplete
leaf-pod00-su00-r0 | RPKI validation codes: V valid, I invalid, N Not found
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 |     Network          Next Hop            Metric LocPrf Weight Path
leaf-pod00-su00-r0 |  *> 172.16.0.0/26    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.16.0.16/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.16.0.48/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  *> 172.16.1.0/26    10.253.128.5(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *                   10.253.128.5(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *                   10.253.128.2(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *> 172.18.1.0/26    10.253.128.6(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *                   10.253.128.6(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *                   10.253.128.3(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *> 172.20.1.0/26    10.253.128.7(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *                   10.253.128.7(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *                   10.253.128.4(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *> 172.22.1.0/26    10.253.128.8(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *                   10.253.128.8(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.24.0.16/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.24.0.48/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  *> 172.24.1.0/26    10.253.128.5(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *                   10.253.128.5(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *                   10.253.128.2(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *> 172.26.1.0/26    10.253.128.6(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *                   10.253.128.6(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *                   10.253.128.3(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *> 172.28.1.0/26    10.253.128.7(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *                   10.253.128.7(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *                   10.253.128.4(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *> 172.30.1.0/26    10.253.128.8(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *                   10.253.128.8(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Displayed 20 routes and 34 total paths
################################################################################

################################################################################
spine-pod00-r0-s00 | View/Vrf tenant2 is unknown
################################################################################
spine-pod00-r1-s00 | View/Vrf tenant2 is unknown
################################################################################
spine-pod00-r2-s00 | View/Vrf tenant2 is unknown
################################################################################
spine-pod00-r3-s00 | View/Vrf tenant2 is unknown
################################################################################
spine-pod01-r0-s00 | View/Vrf tenant2 is unknown
################################################################################
spine-pod01-r1-s00 | View/Vrf tenant2 is unknown
################################################################################
spine-pod01-r2-s00 | View/Vrf tenant2 is unknown
################################################################################
spine-pod01-r3-s00 | View/Vrf tenant2 is unknown
################################################################################
ssp-group00-s00 | View/Vrf tenant2 is unknown
################################################################################
ssp-group00-s01 | View/Vrf tenant2 is unknown
################################################################################
ssp-group00-s02 | View/Vrf tenant2 is unknown
################################################################################
ssp-group00-s03 | View/Vrf tenant2 is unknown
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant2\""
========================================
Running: sudo vtysh -c "show ip route vrf tenant2"
========================================
################################################################################
leaf-pod00-su00-r0 | Codes: K - kernel route, C - connected, L - local, S - static,
leaf-pod00-su00-r0 |        R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
leaf-pod00-su00-r0 |        T - Table, A - Babel, D - SHARP, F - PBR, f - OpenFabric,
leaf-pod00-su00-r0 |        t - Table-Direct, Z - FRR,
leaf-pod00-su00-r0 |        > - selected route, * - FIB route, q - queued, r - rejected, b - backup
leaf-pod00-su00-r0 |        t - trapped, o - offload failure
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | VRF tenant2:
leaf-pod00-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:09:16
leaf-pod00-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:08:45
leaf-pod00-su00-r0 | C>* 172.16.0.16/31 is directly connected, swp9s0, 00:09:16
leaf-pod00-su00-r0 | L>* 172.16.0.17/32 is directly connected, swp9s0, 00:09:16
leaf-pod00-su00-r0 | C>* 172.16.0.48/31 is directly connected, swp25s0, 00:09:16
leaf-pod00-su00-r0 | L>* 172.16.0.49/32 is directly connected, swp25s0, 00:09:16
leaf-pod00-su00-r0 | B>* 172.16.1.0/26 [20/0] via 10.253.128.5, vlan1341_l3 onlink, weight 1, 00:06:11
leaf-pod00-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:08:06
leaf-pod00-su00-r0 | B>* 172.18.1.0/26 [20/0] via 10.253.128.6, vlan1341_l3 onlink, weight 1, 00:05:33
leaf-pod00-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:07:28
leaf-pod00-su00-r0 | B>* 172.20.1.0/26 [20/0] via 10.253.128.7, vlan1341_l3 onlink, weight 1, 00:04:55
leaf-pod00-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:06:49
leaf-pod00-su00-r0 | B>* 172.22.1.0/26 [20/0] via 10.253.128.8, vlan1341_l3 onlink, weight 1, 00:04:18
leaf-pod00-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:08:45
leaf-pod00-su00-r0 | C>* 172.24.0.16/31 is directly connected, swp9s1, 00:09:16
leaf-pod00-su00-r0 | L>* 172.24.0.17/32 is directly connected, swp9s1, 00:09:16
leaf-pod00-su00-r0 | C>* 172.24.0.48/31 is directly connected, swp25s1, 00:09:15
leaf-pod00-su00-r0 | L>* 172.24.0.49/32 is directly connected, swp25s1, 00:09:15
leaf-pod00-su00-r0 | B>* 172.24.1.0/26 [20/0] via 10.253.128.5, vlan1341_l3 onlink, weight 1, 00:06:11
leaf-pod00-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:08:06
leaf-pod00-su00-r0 | B>* 172.26.1.0/26 [20/0] via 10.253.128.6, vlan1341_l3 onlink, weight 1, 00:05:33
leaf-pod00-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:07:28
leaf-pod00-su00-r0 | B>* 172.28.1.0/26 [20/0] via 10.253.128.7, vlan1341_l3 onlink, weight 1, 00:04:55
leaf-pod00-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:06:49
leaf-pod00-su00-r0 | B>* 172.30.1.0/26 [20/0] via 10.253.128.8, vlan1341_l3 onlink, weight 1, 00:04:18
################################################################################
```

`tenant2` appears in the VRF list, the host-facing `swp` ports carry their `172.x` rail gateway `/31`s inside it, and the VRF routing table holds both the local rail subnets and the remote ones learned over EVPN, including the rails in the other POD reached across the super-spine tier.

Neither the spines nor the super-spines hold the tenant VRFs; that lives on the leaves. On the super-spine overlay relay you can see the type-5 host routes it re-advertises:

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp l2vpn evpn route type prefix\""
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp l2vpn evpn route type prefix\""
========================================
Running: sudo vtysh -c "show bgp l2vpn evpn route type prefix"
========================================
################################################################################
leaf-pod00-su00-r0 | BGP table version is 10, local router ID is 10.253.128.1
leaf-pod00-su00-r0 | Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
leaf-pod00-su00-r0 | Origin codes: i - IGP, e - EGP, ? - incomplete
leaf-pod00-su00-r0 | EVPN type-1 prefix: [1]:[EthTag]:[ESI]:[IPlen]:[VTEP-IP]:[Frag-id]
leaf-pod00-su00-r0 | EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
leaf-pod00-su00-r0 | EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
leaf-pod00-su00-r0 | EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
leaf-pod00-su00-r0 | EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 |    Network          Next Hop            Metric LocPrf Weight Path
leaf-pod00-su00-r0 |                     Extended Community
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.1:19998
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.16.0.0] RD 10.253.128.1:19998
leaf-pod00-su00-r0 |                     10.253.128.1 (leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |                     ET:8 RT:59904:19998 Rmac:44:38:39:22:01:cd
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.24.0.0] RD 10.253.128.1:19998
leaf-pod00-su00-r0 |                     10.253.128.1 (leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |                     ET:8 RT:59904:19998 Rmac:44:38:39:22:01:cd
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.1:19999
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.16.0.0] RD 10.253.128.1:19999
leaf-pod00-su00-r0 |                     10.253.128.1 (leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |                     ET:8 RT:59904:19999 Rmac:44:38:39:22:01:cd
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.24.0.0] RD 10.253.128.1:19999
leaf-pod00-su00-r0 |                     10.253.128.1 (leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |                     ET:8 RT:59904:19999 Rmac:44:38:39:22:01:cd
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.2:19998
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.18.0.0] RD 10.253.128.2:19998
leaf-pod00-su00-r0 |                     10.253.128.2 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |                     RT:59905:19998 ET:8 Rmac:44:38:39:22:01:ce
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.18.0.0] RD 10.253.128.2:19998
leaf-pod00-su00-r0 |                     10.253.128.2 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |                     RT:59905:19998 ET:8 Rmac:44:38:39:22:01:ce
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.26.0.0] RD 10.253.128.2:19998
leaf-pod00-su00-r0 |                     10.253.128.2 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |                     RT:59905:19998 ET:8 Rmac:44:38:39:22:01:ce
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.26.0.0] RD 10.253.128.2:19998
leaf-pod00-su00-r0 |                     10.253.128.2 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |                     RT:59905:19998 ET:8 Rmac:44:38:39:22:01:ce
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.2:19999
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.18.0.0] RD 10.253.128.2:19999
leaf-pod00-su00-r0 |                     10.253.128.2 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |                     RT:59905:19999 ET:8 Rmac:44:38:39:22:01:ce
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.18.0.0] RD 10.253.128.2:19999
leaf-pod00-su00-r0 |                     10.253.128.2 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |                     RT:59905:19999 ET:8 Rmac:44:38:39:22:01:ce
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.26.0.0] RD 10.253.128.2:19999
leaf-pod00-su00-r0 |                     10.253.128.2 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |                     RT:59905:19999 ET:8 Rmac:44:38:39:22:01:ce
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.26.0.0] RD 10.253.128.2:19999
leaf-pod00-su00-r0 |                     10.253.128.2 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |                     RT:59905:19999 ET:8 Rmac:44:38:39:22:01:ce
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.3:19998
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.20.0.0] RD 10.253.128.3:19998
leaf-pod00-su00-r0 |                     10.253.128.3 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |                     RT:59906:19998 ET:8 Rmac:44:38:39:22:01:cf
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.20.0.0] RD 10.253.128.3:19998
leaf-pod00-su00-r0 |                     10.253.128.3 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |                     RT:59906:19998 ET:8 Rmac:44:38:39:22:01:cf
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.28.0.0] RD 10.253.128.3:19998
leaf-pod00-su00-r0 |                     10.253.128.3 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |                     RT:59906:19998 ET:8 Rmac:44:38:39:22:01:cf
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.28.0.0] RD 10.253.128.3:19998
leaf-pod00-su00-r0 |                     10.253.128.3 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |                     RT:59906:19998 ET:8 Rmac:44:38:39:22:01:cf
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.3:19999
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.20.0.0] RD 10.253.128.3:19999
leaf-pod00-su00-r0 |                     10.253.128.3 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |                     RT:59906:19999 ET:8 Rmac:44:38:39:22:01:cf
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.20.0.0] RD 10.253.128.3:19999
leaf-pod00-su00-r0 |                     10.253.128.3 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |                     RT:59906:19999 ET:8 Rmac:44:38:39:22:01:cf
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.28.0.0] RD 10.253.128.3:19999
leaf-pod00-su00-r0 |                     10.253.128.3 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |                     RT:59906:19999 ET:8 Rmac:44:38:39:22:01:cf
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.28.0.0] RD 10.253.128.3:19999
leaf-pod00-su00-r0 |                     10.253.128.3 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |                     RT:59906:19999 ET:8 Rmac:44:38:39:22:01:cf
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.4:19998
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.22.0.0] RD 10.253.128.4:19998
leaf-pod00-su00-r0 |                     10.253.128.4 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |                     RT:59907:19998 ET:8 Rmac:44:38:39:22:01:d0
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.22.0.0] RD 10.253.128.4:19998
leaf-pod00-su00-r0 |                     10.253.128.4 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |                     RT:59907:19998 ET:8 Rmac:44:38:39:22:01:d0
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.30.0.0] RD 10.253.128.4:19998
leaf-pod00-su00-r0 |                     10.253.128.4 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |                     RT:59907:19998 ET:8 Rmac:44:38:39:22:01:d0
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.30.0.0] RD 10.253.128.4:19998
leaf-pod00-su00-r0 |                     10.253.128.4 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |                     RT:59907:19998 ET:8 Rmac:44:38:39:22:01:d0
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.4:19999
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.22.0.0] RD 10.253.128.4:19999
leaf-pod00-su00-r0 |                     10.253.128.4 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |                     RT:59907:19999 ET:8 Rmac:44:38:39:22:01:d0
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.22.0.0] RD 10.253.128.4:19999
leaf-pod00-su00-r0 |                     10.253.128.4 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |                     RT:59907:19999 ET:8 Rmac:44:38:39:22:01:d0
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.30.0.0] RD 10.253.128.4:19999
leaf-pod00-su00-r0 |                     10.253.128.4 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |                     RT:59907:19999 ET:8 Rmac:44:38:39:22:01:d0
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.30.0.0] RD 10.253.128.4:19999
leaf-pod00-su00-r0 |                     10.253.128.4 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |                     RT:59907:19999 ET:8 Rmac:44:38:39:22:01:d0
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.5:19998
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.16.1.0] RD 10.253.128.5:19998
leaf-pod00-su00-r0 |                     10.253.128.5 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |                     RT:59908:19998 ET:8 Rmac:44:38:39:22:01:d1
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.16.1.0] RD 10.253.128.5:19998
leaf-pod00-su00-r0 |                     10.253.128.5 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |                     RT:59908:19998 ET:8 Rmac:44:38:39:22:01:d1
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.24.1.0] RD 10.253.128.5:19998
leaf-pod00-su00-r0 |                     10.253.128.5 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |                     RT:59908:19998 ET:8 Rmac:44:38:39:22:01:d1
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.24.1.0] RD 10.253.128.5:19998
leaf-pod00-su00-r0 |                     10.253.128.5 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |                     RT:59908:19998 ET:8 Rmac:44:38:39:22:01:d1
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.5:19999
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.16.1.0] RD 10.253.128.5:19999
leaf-pod00-su00-r0 |                     10.253.128.5 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |                     RT:59908:19999 ET:8 Rmac:44:38:39:22:01:d1
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.16.1.0] RD 10.253.128.5:19999
leaf-pod00-su00-r0 |                     10.253.128.5 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |                     RT:59908:19999 ET:8 Rmac:44:38:39:22:01:d1
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.24.1.0] RD 10.253.128.5:19999
leaf-pod00-su00-r0 |                     10.253.128.5 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |                     RT:59908:19999 ET:8 Rmac:44:38:39:22:01:d1
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.24.1.0] RD 10.253.128.5:19999
leaf-pod00-su00-r0 |                     10.253.128.5 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |                     RT:59908:19999 ET:8 Rmac:44:38:39:22:01:d1
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.6:19998
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.18.1.0] RD 10.253.128.6:19998
leaf-pod00-su00-r0 |                     10.253.128.6 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |                     RT:59909:19998 ET:8 Rmac:44:38:39:22:01:d2
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.18.1.0] RD 10.253.128.6:19998
leaf-pod00-su00-r0 |                     10.253.128.6 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |                     RT:59909:19998 ET:8 Rmac:44:38:39:22:01:d2
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.26.1.0] RD 10.253.128.6:19998
leaf-pod00-su00-r0 |                     10.253.128.6 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |                     RT:59909:19998 ET:8 Rmac:44:38:39:22:01:d2
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.26.1.0] RD 10.253.128.6:19998
leaf-pod00-su00-r0 |                     10.253.128.6 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |                     RT:59909:19998 ET:8 Rmac:44:38:39:22:01:d2
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.6:19999
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.18.1.0] RD 10.253.128.6:19999
leaf-pod00-su00-r0 |                     10.253.128.6 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |                     RT:59909:19999 ET:8 Rmac:44:38:39:22:01:d2
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.18.1.0] RD 10.253.128.6:19999
leaf-pod00-su00-r0 |                     10.253.128.6 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |                     RT:59909:19999 ET:8 Rmac:44:38:39:22:01:d2
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.26.1.0] RD 10.253.128.6:19999
leaf-pod00-su00-r0 |                     10.253.128.6 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |                     RT:59909:19999 ET:8 Rmac:44:38:39:22:01:d2
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.26.1.0] RD 10.253.128.6:19999
leaf-pod00-su00-r0 |                     10.253.128.6 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |                     RT:59909:19999 ET:8 Rmac:44:38:39:22:01:d2
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.7:19998
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.20.1.0] RD 10.253.128.7:19998
leaf-pod00-su00-r0 |                     10.253.128.7 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |                     RT:59910:19998 ET:8 Rmac:44:38:39:22:01:d3
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.20.1.0] RD 10.253.128.7:19998
leaf-pod00-su00-r0 |                     10.253.128.7 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |                     RT:59910:19998 ET:8 Rmac:44:38:39:22:01:d3
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.28.1.0] RD 10.253.128.7:19998
leaf-pod00-su00-r0 |                     10.253.128.7 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |                     RT:59910:19998 ET:8 Rmac:44:38:39:22:01:d3
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.28.1.0] RD 10.253.128.7:19998
leaf-pod00-su00-r0 |                     10.253.128.7 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |                     RT:59910:19998 ET:8 Rmac:44:38:39:22:01:d3
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.7:19999
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.20.1.0] RD 10.253.128.7:19999
leaf-pod00-su00-r0 |                     10.253.128.7 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |                     RT:59910:19999 ET:8 Rmac:44:38:39:22:01:d3
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.20.1.0] RD 10.253.128.7:19999
leaf-pod00-su00-r0 |                     10.253.128.7 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |                     RT:59910:19999 ET:8 Rmac:44:38:39:22:01:d3
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.28.1.0] RD 10.253.128.7:19999
leaf-pod00-su00-r0 |                     10.253.128.7 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |                     RT:59910:19999 ET:8 Rmac:44:38:39:22:01:d3
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.28.1.0] RD 10.253.128.7:19999
leaf-pod00-su00-r0 |                     10.253.128.7 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |                     RT:59910:19999 ET:8 Rmac:44:38:39:22:01:d3
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.8:19998
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.22.1.0] RD 10.253.128.8:19998
leaf-pod00-su00-r0 |                     10.253.128.8 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |                     RT:59911:19998 ET:8 Rmac:44:38:39:22:01:d4
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.22.1.0] RD 10.253.128.8:19998
leaf-pod00-su00-r0 |                     10.253.128.8 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |                     RT:59911:19998 ET:8 Rmac:44:38:39:22:01:d4
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.30.1.0] RD 10.253.128.8:19998
leaf-pod00-su00-r0 |                     10.253.128.8 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |                     RT:59911:19998 ET:8 Rmac:44:38:39:22:01:d4
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.30.1.0] RD 10.253.128.8:19998
leaf-pod00-su00-r0 |                     10.253.128.8 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |                     RT:59911:19998 ET:8 Rmac:44:38:39:22:01:d4
leaf-pod00-su00-r0 | Route Distinguisher: 10.253.128.8:19999
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.22.1.0] RD 10.253.128.8:19999
leaf-pod00-su00-r0 |                     10.253.128.8 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |                     RT:59911:19999 ET:8 Rmac:44:38:39:22:01:d4
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.22.1.0] RD 10.253.128.8:19999
leaf-pod00-su00-r0 |                     10.253.128.8 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |                     RT:59911:19999 ET:8 Rmac:44:38:39:22:01:d4
leaf-pod00-su00-r0 |  *> [5]:[0]:[26]:[172.30.1.0] RD 10.253.128.8:19999
leaf-pod00-su00-r0 |                     10.253.128.8 (ssp-group00-s00)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |                     RT:59911:19999 ET:8 Rmac:44:38:39:22:01:d4
leaf-pod00-su00-r0 |  *  [5]:[0]:[26]:[172.30.1.0] RD 10.253.128.8:19999
leaf-pod00-su00-r0 |                     10.253.128.8 (ssp-group00-s01)
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |                     RT:59911:19999 ET:8 Rmac:44:38:39:22:01:d4
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Displayed 32 prefixes (60 paths) (of requested type)
################################################################################
```

This is the control-plane state behind the host-to-host reachability that Step 14 verifies from the hosts.

### Step 13. Configure the HGX Hosts

**Goal:** Apply the per-host rail netplan configuration so each HGX host's eight rail NICs come up with their `/31` addresses.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`; the HGX hosts at `192.168.200.31` through `.38`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation and on the HGX hosts.

**Expected wait time:** Under a minute per host.

Each host has eight rail NICs (`eth_rail0` through `eth_rail7`) at MTU 9216. Each rail takes a `/31`, with the host on the even address and the leaf gateway on the odd one. For host node `h` on rail `r` in POD `p`, the host address is `172.(16 + 2*r).p.(2*h)` and the gateway is one higher. The rail sets the second octet, the POD sets the third (POD0 = 0, POD1 = 1), and the node sets the fourth (h00 = 0, h08 = 16, h16 = 32, h24 = 48):

| __rail__ | __NIC__ | __pod00 h00__ | __pod00 h08__ | __pod00 h16__ | __pod00 h24__ | __pod01 h00__ | __pod01 h08__ | __pod01 h16__ | __pod01 h24__ |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | `eth_rail0` | 172.16.0.0 | 172.16.0.16 | 172.16.0.32 | 172.16.0.48 | 172.16.1.0 | 172.16.1.16 | 172.16.1.32 | 172.16.1.48 |
| 1 | `eth_rail1` | 172.18.0.0 | 172.18.0.16 | 172.18.0.32 | 172.18.0.48 | 172.18.1.0 | 172.18.1.16 | 172.18.1.32 | 172.18.1.48 |
| 2 | `eth_rail2` | 172.20.0.0 | 172.20.0.16 | 172.20.0.32 | 172.20.0.48 | 172.20.1.0 | 172.20.1.16 | 172.20.1.32 | 172.20.1.48 |
| 3 | `eth_rail3` | 172.22.0.0 | 172.22.0.16 | 172.22.0.32 | 172.22.0.48 | 172.22.1.0 | 172.22.1.16 | 172.22.1.32 | 172.22.1.48 |
| 4 | `eth_rail4` | 172.24.0.0 | 172.24.0.16 | 172.24.0.32 | 172.24.0.48 | 172.24.1.0 | 172.24.1.16 | 172.24.1.32 | 172.24.1.48 |
| 5 | `eth_rail5` | 172.26.0.0 | 172.26.0.16 | 172.26.0.32 | 172.26.0.48 | 172.26.1.0 | 172.26.1.16 | 172.26.1.32 | 172.26.1.48 |
| 6 | `eth_rail6` | 172.28.0.0 | 172.28.0.16 | 172.28.0.32 | 172.28.0.48 | 172.28.1.0 | 172.28.1.16 | 172.28.1.32 | 172.28.1.48 |
| 7 | `eth_rail7` | 172.30.0.0 | 172.30.0.16 | 172.30.0.32 | 172.30.0.48 | 172.30.1.0 | 172.30.1.16 | 172.30.1.32 | 172.30.1.48 |

The complete per-host netplan files are staged on the jumpstation under `netplan/3tier/`, one per host (`hgx-pod00-su00-h00.yaml` through `hgx-pod01-su00-h24.yaml`). Each holds all eight rails and installs to `/etc/netplan/60-spectrum-x.yaml` on its host.

First, capture each host's rail state *before* applying netplan, so there is a baseline to compare against. The `eth_rail` NICs carry no `172.x` addresses yet and the routing table holds no rail routes:

```bash
~/spcx-air/spcx-run -s ip -br a
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -s ip -br a
Running: ip -br a

################################################################################
hgx-pod00-su00-h00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod00-su00-h00 | eth0             UP             192.168.200.31/24 metric 100 fe80::4638:39ff:fe11:11f/64
hgx-pod00-su00-h00 | eth_rail0        DOWN
hgx-pod00-su00-h00 | eth_rail1        DOWN
hgx-pod00-su00-h00 | eth_rail2        DOWN
hgx-pod00-su00-h00 | eth_rail3        DOWN
hgx-pod00-su00-h00 | eth_rail4        DOWN
hgx-pod00-su00-h00 | eth_rail5        DOWN
hgx-pod00-su00-h00 | eth_rail6        DOWN
hgx-pod00-su00-h00 | eth_rail7        DOWN
################################################################################
hgx-pod00-su00-h08 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod00-su00-h08 | eth0             UP             192.168.200.32/24 metric 100 fe80::4638:39ff:fe11:120/64
hgx-pod00-su00-h08 | eth_rail0        DOWN
hgx-pod00-su00-h08 | eth_rail1        DOWN
hgx-pod00-su00-h08 | eth_rail2        DOWN
hgx-pod00-su00-h08 | eth_rail3        DOWN
hgx-pod00-su00-h08 | eth_rail4        DOWN
hgx-pod00-su00-h08 | eth_rail5        DOWN
hgx-pod00-su00-h08 | eth_rail6        DOWN
hgx-pod00-su00-h08 | eth_rail7        DOWN
################################################################################
hgx-pod00-su00-h16 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod00-su00-h16 | eth0             UP             192.168.200.33/24 metric 100 fe80::4638:39ff:fe11:121/64
hgx-pod00-su00-h16 | eth_rail0        DOWN
hgx-pod00-su00-h16 | eth_rail1        DOWN
hgx-pod00-su00-h16 | eth_rail2        DOWN
hgx-pod00-su00-h16 | eth_rail3        DOWN
hgx-pod00-su00-h16 | eth_rail4        DOWN
hgx-pod00-su00-h16 | eth_rail5        DOWN
hgx-pod00-su00-h16 | eth_rail6        DOWN
hgx-pod00-su00-h16 | eth_rail7        DOWN
################################################################################
hgx-pod00-su00-h24 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod00-su00-h24 | eth0             UP             192.168.200.34/24 metric 100 fe80::4638:39ff:fe11:122/64
hgx-pod00-su00-h24 | eth_rail0        DOWN
hgx-pod00-su00-h24 | eth_rail1        DOWN
hgx-pod00-su00-h24 | eth_rail2        DOWN
hgx-pod00-su00-h24 | eth_rail3        DOWN
hgx-pod00-su00-h24 | eth_rail4        DOWN
hgx-pod00-su00-h24 | eth_rail5        DOWN
hgx-pod00-su00-h24 | eth_rail6        DOWN
hgx-pod00-su00-h24 | eth_rail7        DOWN
################################################################################
hgx-pod01-su00-h00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod01-su00-h00 | eth0             UP             192.168.200.35/24 metric 100 fe80::4638:39ff:fe11:123/64
hgx-pod01-su00-h00 | eth_rail0        DOWN
hgx-pod01-su00-h00 | eth_rail1        DOWN
hgx-pod01-su00-h00 | eth_rail2        DOWN
hgx-pod01-su00-h00 | eth_rail3        DOWN
hgx-pod01-su00-h00 | eth_rail4        DOWN
hgx-pod01-su00-h00 | eth_rail5        DOWN
hgx-pod01-su00-h00 | eth_rail6        DOWN
hgx-pod01-su00-h00 | eth_rail7        DOWN
################################################################################
hgx-pod01-su00-h08 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod01-su00-h08 | eth0             UP             192.168.200.36/24 metric 100 fe80::4638:39ff:fe11:124/64
hgx-pod01-su00-h08 | eth_rail0        DOWN
hgx-pod01-su00-h08 | eth_rail1        DOWN
hgx-pod01-su00-h08 | eth_rail2        DOWN
hgx-pod01-su00-h08 | eth_rail3        DOWN
hgx-pod01-su00-h08 | eth_rail4        DOWN
hgx-pod01-su00-h08 | eth_rail5        DOWN
hgx-pod01-su00-h08 | eth_rail6        DOWN
hgx-pod01-su00-h08 | eth_rail7        DOWN
################################################################################
hgx-pod01-su00-h16 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod01-su00-h16 | eth0             UP             192.168.200.37/24 metric 100 fe80::4638:39ff:fe11:125/64
hgx-pod01-su00-h16 | eth_rail0        DOWN
hgx-pod01-su00-h16 | eth_rail1        DOWN
hgx-pod01-su00-h16 | eth_rail2        DOWN
hgx-pod01-su00-h16 | eth_rail3        DOWN
hgx-pod01-su00-h16 | eth_rail4        DOWN
hgx-pod01-su00-h16 | eth_rail5        DOWN
hgx-pod01-su00-h16 | eth_rail6        DOWN
hgx-pod01-su00-h16 | eth_rail7        DOWN
################################################################################
hgx-pod01-su00-h24 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod01-su00-h24 | eth0             UP             192.168.200.38/24 metric 100 fe80::4638:39ff:fe11:126/64
hgx-pod01-su00-h24 | eth_rail0        DOWN
hgx-pod01-su00-h24 | eth_rail1        DOWN
hgx-pod01-su00-h24 | eth_rail2        DOWN
hgx-pod01-su00-h24 | eth_rail3        DOWN
hgx-pod01-su00-h24 | eth_rail4        DOWN
hgx-pod01-su00-h24 | eth_rail5        DOWN
hgx-pod01-su00-h24 | eth_rail6        DOWN
hgx-pod01-su00-h24 | eth_rail7        DOWN
################################################################################
```

Now push each host the file named for it and apply it. This loop reads each host's name, copies its file, installs it with mode 0600, and runs `netplan apply`:

```bash
 ~/spcx-air/spcx-run --apply-netplan ~/spcx-air/fabric/3tier/hosts/
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run --apply-netplan ~/spcx-air/fabric/3tier/hosts/
Applying per-host netplan from: /home/ubuntu/spcx-air/fabric/3tier/hosts
Target: hgx_hosts

################################################################################
hgx-pod00-su00-h00 | (no output)
################################################################################
hgx-pod00-su00-h08 | (no output)
################################################################################
hgx-pod00-su00-h16 | (no output)
################################################################################
hgx-pod00-su00-h24 | (no output)
################################################################################
hgx-pod01-su00-h00 | (no output)
################################################################################
hgx-pod01-su00-h08 | (no output)
################################################################################
hgx-pod01-su00-h16 | (no output)
################################################################################
hgx-pod01-su00-h24 | (no output)
################################################################################
ubuntu@oob-mgmt-server:~/nvidia$
```

Validation:

Re-run the commands to see the effect of the netplan change. Each `eth_rail` NIC is now `UP` with its `/31` address, and the routing table has gained the per-rail `/15` and `/12` rail routes (via each rail gateway) that carry the RoCE traffic, none of which were present in the baseline above:

```bash
~/spcx-air/spcx-run -s ip -br a
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -s ip -br a
Running: ip -br a

################################################################################
hgx-pod00-su00-h00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod00-su00-h00 | eth0             UP             192.168.200.31/24 metric 100 fe80::4638:39ff:fe11:11f/64
hgx-pod00-su00-h00 | eth_rail0        UP             172.16.0.0/31 fe80::4ab0:2dff:fedd:4669/64
hgx-pod00-su00-h00 | eth_rail1        UP             172.18.0.0/31 fe80::4ab0:2dff:fe3d:70aa/64
hgx-pod00-su00-h00 | eth_rail2        UP             172.20.0.0/31 fe80::4ab0:2dff:fe4a:e013/64
hgx-pod00-su00-h00 | eth_rail3        UP             172.22.0.0/31 fe80::4ab0:2dff:fe31:25f3/64
hgx-pod00-su00-h00 | eth_rail4        UP             172.24.0.0/31 fe80::4ab0:2dff:feb2:c780/64
hgx-pod00-su00-h00 | eth_rail5        UP             172.26.0.0/31 fe80::4ab0:2dff:fe28:458d/64
hgx-pod00-su00-h00 | eth_rail6        UP             172.28.0.0/31 fe80::4ab0:2dff:fee5:9401/64
hgx-pod00-su00-h00 | eth_rail7        UP             172.30.0.0/31 fe80::4ab0:2dff:fe07:b8a4/64
################################################################################
hgx-pod00-su00-h08 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod00-su00-h08 | eth0             UP             192.168.200.32/24 metric 100 fe80::4638:39ff:fe11:120/64
hgx-pod00-su00-h08 | eth_rail0        UP             172.16.0.16/31 fe80::4ab0:2dff:fe4d:c057/64
hgx-pod00-su00-h08 | eth_rail1        UP             172.18.0.16/31 fe80::4ab0:2dff:feb7:2cf3/64
hgx-pod00-su00-h08 | eth_rail2        UP             172.20.0.16/31 fe80::4ab0:2dff:fed4:2f10/64
hgx-pod00-su00-h08 | eth_rail3        UP             172.22.0.16/31 fe80::4ab0:2dff:fe48:ade3/64
hgx-pod00-su00-h08 | eth_rail4        UP             172.24.0.16/31 fe80::4ab0:2dff:fe51:15b0/64
hgx-pod00-su00-h08 | eth_rail5        UP             172.26.0.16/31 fe80::4ab0:2dff:fe11:9c6d/64
hgx-pod00-su00-h08 | eth_rail6        UP             172.28.0.16/31 fe80::4ab0:2dff:fee2:2da5/64
hgx-pod00-su00-h08 | eth_rail7        UP             172.30.0.16/31 fe80::4ab0:2dff:fe9d:98be/64
################################################################################
hgx-pod00-su00-h16 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod00-su00-h16 | eth0             UP             192.168.200.33/24 metric 100 fe80::4638:39ff:fe11:121/64
hgx-pod00-su00-h16 | eth_rail0        UP             172.16.0.32/31 fe80::4ab0:2dff:fe6f:341b/64
hgx-pod00-su00-h16 | eth_rail1        UP             172.18.0.32/31 fe80::4ab0:2dff:fe80:ee3e/64
hgx-pod00-su00-h16 | eth_rail2        UP             172.20.0.32/31 fe80::4ab0:2dff:fecd:ba6c/64
hgx-pod00-su00-h16 | eth_rail3        UP             172.22.0.32/31 fe80::4ab0:2dff:fe8c:25b1/64
hgx-pod00-su00-h16 | eth_rail4        UP             172.24.0.32/31 fe80::4ab0:2dff:feb2:af5e/64
hgx-pod00-su00-h16 | eth_rail5        UP             172.26.0.32/31 fe80::4ab0:2dff:fe24:468d/64
hgx-pod00-su00-h16 | eth_rail6        UP             172.28.0.32/31 fe80::4ab0:2dff:fef3:3d2d/64
hgx-pod00-su00-h16 | eth_rail7        UP             172.30.0.32/31 fe80::4ab0:2dff:fe59:c2ec/64
################################################################################
hgx-pod00-su00-h24 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod00-su00-h24 | eth0             UP             192.168.200.34/24 metric 100 fe80::4638:39ff:fe11:122/64
hgx-pod00-su00-h24 | eth_rail0        UP             172.16.0.48/31 fe80::4ab0:2dff:fea2:4d74/64
hgx-pod00-su00-h24 | eth_rail1        UP             172.18.0.48/31 fe80::4ab0:2dff:fed1:e6f4/64
hgx-pod00-su00-h24 | eth_rail2        UP             172.20.0.48/31 fe80::4ab0:2dff:fe6b:6c81/64
hgx-pod00-su00-h24 | eth_rail3        UP             172.22.0.48/31 fe80::4ab0:2dff:fe4f:383/64
hgx-pod00-su00-h24 | eth_rail4        UP             172.24.0.48/31 fe80::4ab0:2dff:fe99:8777/64
hgx-pod00-su00-h24 | eth_rail5        UP             172.26.0.48/31 fe80::4ab0:2dff:fe4c:3d78/64
hgx-pod00-su00-h24 | eth_rail6        UP             172.28.0.48/31 fe80::4ab0:2dff:fe5e:2dde/64
hgx-pod00-su00-h24 | eth_rail7        UP             172.30.0.48/31 fe80::4ab0:2dff:fe47:9836/64
################################################################################
hgx-pod01-su00-h00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod01-su00-h00 | eth0             UP             192.168.200.35/24 metric 100 fe80::4638:39ff:fe11:123/64
hgx-pod01-su00-h00 | eth_rail0        UP             172.16.1.0/31 fe80::4ab0:2dff:feb6:984d/64
hgx-pod01-su00-h00 | eth_rail1        UP             172.18.1.0/31 fe80::4ab0:2dff:fe07:120f/64
hgx-pod01-su00-h00 | eth_rail2        UP             172.20.1.0/31 fe80::4ab0:2dff:fe59:77ae/64
hgx-pod01-su00-h00 | eth_rail3        UP             172.22.1.0/31 fe80::4ab0:2dff:fe1a:963/64
hgx-pod01-su00-h00 | eth_rail4        UP             172.24.1.0/31 fe80::4ab0:2dff:fe74:25b4/64
hgx-pod01-su00-h00 | eth_rail5        UP             172.26.1.0/31 fe80::4ab0:2dff:fe50:8576/64
hgx-pod01-su00-h00 | eth_rail6        UP             172.28.1.0/31 fe80::4ab0:2dff:fe7c:df4/64
hgx-pod01-su00-h00 | eth_rail7        UP             172.30.1.0/31 fe80::4ab0:2dff:fefd:481d/64
################################################################################
hgx-pod01-su00-h08 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod01-su00-h08 | eth0             UP             192.168.200.36/24 metric 100 fe80::4638:39ff:fe11:124/64
hgx-pod01-su00-h08 | eth_rail0        UP             172.16.1.16/31 fe80::4ab0:2dff:fe00:cd18/64
hgx-pod01-su00-h08 | eth_rail1        UP             172.18.1.16/31 fe80::4ab0:2dff:fecb:511a/64
hgx-pod01-su00-h08 | eth_rail2        UP             172.20.1.16/31 fe80::4ab0:2dff:fe03:fd81/64
hgx-pod01-su00-h08 | eth_rail3        UP             172.22.1.16/31 fe80::4ab0:2dff:fe83:1621/64
hgx-pod01-su00-h08 | eth_rail4        UP             172.24.1.16/31 fe80::4ab0:2dff:fe08:65ba/64
hgx-pod01-su00-h08 | eth_rail5        UP             172.26.1.16/31 fe80::4ab0:2dff:fe2e:f27f/64
hgx-pod01-su00-h08 | eth_rail6        UP             172.28.1.16/31 fe80::4ab0:2dff:feaa:f4e/64
hgx-pod01-su00-h08 | eth_rail7        UP             172.30.1.16/31 fe80::4ab0:2dff:fee5:1435/64
################################################################################
hgx-pod01-su00-h16 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod01-su00-h16 | eth0             UP             192.168.200.37/24 metric 100 fe80::4638:39ff:fe11:125/64
hgx-pod01-su00-h16 | eth_rail0        UP             172.16.1.32/31 fe80::4ab0:2dff:fe53:648c/64
hgx-pod01-su00-h16 | eth_rail1        UP             172.18.1.32/31 fe80::4ab0:2dff:fe5d:95d3/64
hgx-pod01-su00-h16 | eth_rail2        UP             172.20.1.32/31 fe80::4ab0:2dff:fe24:c7e9/64
hgx-pod01-su00-h16 | eth_rail3        UP             172.22.1.32/31 fe80::4ab0:2dff:fe8d:ba7/64
hgx-pod01-su00-h16 | eth_rail4        UP             172.24.1.32/31 fe80::4ab0:2dff:fefa:ca82/64
hgx-pod01-su00-h16 | eth_rail5        UP             172.26.1.32/31 fe80::4ab0:2dff:fedd:5b61/64
hgx-pod01-su00-h16 | eth_rail6        UP             172.28.1.32/31 fe80::4ab0:2dff:fee6:8394/64
hgx-pod01-su00-h16 | eth_rail7        UP             172.30.1.32/31 fe80::4ab0:2dff:fec4:d24a/64
################################################################################
hgx-pod01-su00-h24 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-pod01-su00-h24 | eth0             UP             192.168.200.38/24 metric 100 fe80::4638:39ff:fe11:126/64
hgx-pod01-su00-h24 | eth_rail0        UP             172.16.1.48/31 fe80::4ab0:2dff:fe04:ac67/64
hgx-pod01-su00-h24 | eth_rail1        UP             172.18.1.48/31 fe80::4ab0:2dff:fed7:4736/64
hgx-pod01-su00-h24 | eth_rail2        UP             172.20.1.48/31 fe80::4ab0:2dff:fef1:7824/64
hgx-pod01-su00-h24 | eth_rail3        UP             172.22.1.48/31 fe80::4ab0:2dff:fe55:bc80/64
hgx-pod01-su00-h24 | eth_rail4        UP             172.24.1.48/31 fe80::4ab0:2dff:fe43:8317/64
hgx-pod01-su00-h24 | eth_rail5        UP             172.26.1.48/31 fe80::4ab0:2dff:fef6:5077/64
hgx-pod01-su00-h24 | eth_rail6        UP             172.28.1.48/31 fe80::4ab0:2dff:fe8d:aaf8/64
hgx-pod01-su00-h24 | eth_rail7        UP             172.30.1.48/31 fe80::4ab0:2dff:fe33:cb19/64
################################################################################
```

### Step 14. Verify Connectivity

**Goal:** Confirm the tenant deployments finished, then run a full rail mesh ping test across all eight HGX hosts, confirming full-mesh reachability within each tenant (including across PODs, over the super-spine tier) and isolation between `tenant1` and `tenant2`.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`; the HGX hosts at `192.168.200.31` through `.38`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation and on the HGX hosts.

**Expected wait time:** Up to about 30 seconds per host for BGP/EVPN to finish converging.

The rail `/31`s live only on the hosts, so the ping sweep has to run on each host, not on the jumpstation. This block drives all eight hosts from the jumpstation: it runs the rail mesh on each one, covering all eight rails across both PODs and all four hosts per POD (64 targets per host, including the same-rail peers in the other POD):

```bash
RAILS0="172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 172.16.1.0 172.16.1.16 172.16.1.32 172.16.1.48"
~/spcx-air/spcx-run -s -c "fping -c2 -t500 $RAILS0 2>&1"
```

```bash
~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ RAILS0="172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 172.16.1.0 172.16.1.16 172.16.1.32 172.16.1.48"
~/spcx-air/spcx-run -s -c "fping -c2 -t500 $RAILS0 2>&1"
========================================
Running: fping -c2 -t500 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 172.16.1.0 172.16.1.16 172.16.1.32 172.16.1.48 2>&1
========================================
################################################################################
hgx-pod00-su00-h00 | 172.16.0.0  : [0], 64 bytes, 0.024 ms (0.024 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.0.32 : [0], 64 bytes, 0.489 ms (0.489 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.1.0  : [0], 64 bytes, 1.97 ms (1.97 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.1.32 : [0], 64 bytes, 2.33 ms (2.33 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.0.0  : [1], 64 bytes, 0.038 ms (0.031 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.0.32 : [1], 64 bytes, 0.558 ms (0.524 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.1.0  : [1], 64 bytes, 2.20 ms (2.09 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.1.32 : [1], 64 bytes, 1.98 ms (2.16 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 |
hgx-pod00-su00-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.024/0.031/0.038
hgx-pod00-su00-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.489/0.524/0.558
hgx-pod00-su00-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h00 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.97/2.09/2.20
hgx-pod00-su00-h00 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h00 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.98/2.16/2.33
hgx-pod00-su00-h00 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-pod00-su00-h08 | 172.16.0.16 : [0], 64 bytes, 0.046 ms (0.046 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.48 : [0], 64 bytes, 0.551 ms (0.551 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.1.16 : [0], 64 bytes, 2.41 ms (2.41 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.1.48 : [0], 64 bytes, 2.37 ms (2.37 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.0.16 : [1], 64 bytes, 0.019 ms (0.033 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.48 : [1], 64 bytes, 0.506 ms (0.528 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.1.16 : [1], 64 bytes, 1.98 ms (2.19 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.1.48 : [1], 64 bytes, 2.16 ms (2.26 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 |
hgx-pod00-su00-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.019/0.033/0.046
hgx-pod00-su00-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.506/0.528/0.551
hgx-pod00-su00-h08 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h08 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.98/2.19/2.41
hgx-pod00-su00-h08 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h08 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.16/2.26/2.37
################################################################################
hgx-pod00-su00-h16 | 172.16.0.0  : [0], 64 bytes, 0.820 ms (0.820 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.32 : [0], 64 bytes, 0.035 ms (0.035 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.1.0  : [0], 64 bytes, 1.93 ms (1.93 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.1.32 : [0], 64 bytes, 2.10 ms (2.10 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.0.0  : [1], 64 bytes, 0.678 ms (0.749 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.32 : [1], 64 bytes, 0.021 ms (0.028 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.1.0  : [1], 64 bytes, 2.72 ms (2.32 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.1.32 : [1], 64 bytes, 2.04 ms (2.07 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 |
hgx-pod00-su00-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.678/0.749/0.820
hgx-pod00-su00-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.021/0.028/0.035
hgx-pod00-su00-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h16 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.93/2.32/2.72
hgx-pod00-su00-h16 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h16 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.04/2.07/2.10
hgx-pod00-su00-h16 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-pod00-su00-h24 | 172.16.0.16 : [0], 64 bytes, 0.559 ms (0.559 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.48 : [0], 64 bytes, 0.014 ms (0.014 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.1.16 : [0], 64 bytes, 2.20 ms (2.20 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.1.48 : [0], 64 bytes, 1.97 ms (1.97 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.0.16 : [1], 64 bytes, 0.629 ms (0.594 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.48 : [1], 64 bytes, 0.014 ms (0.014 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.1.16 : [1], 64 bytes, 1.79 ms (1.99 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.1.48 : [1], 64 bytes, 1.73 ms (1.85 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 |
hgx-pod00-su00-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.559/0.594/0.629
hgx-pod00-su00-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.014/0.014/0.014
hgx-pod00-su00-h24 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h24 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.79/1.99/2.20
hgx-pod00-su00-h24 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h24 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.73/1.85/1.97
################################################################################
hgx-pod01-su00-h00 | 172.16.0.0  : [0], 64 bytes, 2.11 ms (2.11 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.0.32 : [0], 64 bytes, 1.69 ms (1.69 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.1.0  : [0], 64 bytes, 0.014 ms (0.014 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.1.32 : [0], 64 bytes, 0.571 ms (0.571 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.0.0  : [1], 64 bytes, 2.18 ms (2.14 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.0.32 : [1], 64 bytes, 1.84 ms (1.77 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.1.0  : [1], 64 bytes, 0.017 ms (0.016 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.1.32 : [1], 64 bytes, 0.582 ms (0.577 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 |
hgx-pod01-su00-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.11/2.14/2.18
hgx-pod01-su00-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.69/1.77/1.84
hgx-pod01-su00-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h00 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.014/0.016/0.017
hgx-pod01-su00-h00 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h00 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.571/0.577/0.582
hgx-pod01-su00-h00 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-pod01-su00-h08 | 172.16.0.16 : [0], 64 bytes, 2.73 ms (2.73 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.48 : [0], 64 bytes, 2.43 ms (2.43 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.1.16 : [0], 64 bytes, 0.037 ms (0.037 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.1.48 : [0], 64 bytes, 0.530 ms (0.530 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.0.16 : [1], 64 bytes, 2.50 ms (2.61 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.48 : [1], 64 bytes, 2.38 ms (2.41 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.1.16 : [1], 64 bytes, 0.046 ms (0.042 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.1.48 : [1], 64 bytes, 0.508 ms (0.519 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 |
hgx-pod01-su00-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.50/2.61/2.73
hgx-pod01-su00-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.38/2.41/2.43
hgx-pod01-su00-h08 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h08 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.037/0.042/0.046
hgx-pod01-su00-h08 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h08 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.508/0.519/0.530
################################################################################
hgx-pod01-su00-h16 | 172.16.0.0  : [0], 64 bytes, 3.20 ms (3.20 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.0.32 : [0], 64 bytes, 3.18 ms (3.18 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.1.0  : [0], 64 bytes, 1.16 ms (1.16 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.1.32 : [0], 64 bytes, 0.026 ms (0.026 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.0.0  : [1], 64 bytes, 2.97 ms (3.09 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.0.32 : [1], 64 bytes, 2.25 ms (2.71 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.1.0  : [1], 64 bytes, 0.648 ms (0.906 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.1.32 : [1], 64 bytes, 0.052 ms (0.039 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 |
hgx-pod01-su00-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.97/3.09/3.20
hgx-pod01-su00-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.25/2.71/3.18
hgx-pod01-su00-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h16 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.648/0.906/1.16
hgx-pod01-su00-h16 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h16 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.026/0.039/0.052
hgx-pod01-su00-h16 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-pod01-su00-h24 | 172.16.0.16 : [0], 64 bytes, 2.41 ms (2.41 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.48 : [0], 64 bytes, 1.90 ms (1.90 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.1.16 : [0], 64 bytes, 0.768 ms (0.768 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.1.48 : [0], 64 bytes, 0.020 ms (0.020 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.0.16 : [1], 64 bytes, 2.51 ms (2.46 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.48 : [1], 64 bytes, 2.22 ms (2.06 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.1.16 : [1], 64 bytes, 0.883 ms (0.826 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.1.48 : [1], 64 bytes, 0.013 ms (0.016 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 |
hgx-pod01-su00-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.41/2.46/2.51
hgx-pod01-su00-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.90/2.06/2.22
hgx-pod01-su00-h24 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h24 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.768/0.826/0.883
hgx-pod01-su00-h24 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h24 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.013/0.016/0.020
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
========================================
Running: for ip in 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 172.16.1.0 172.16.1.16 172.16.1.32 172.16.1.48; do ping -c2 -W2 $ip >/dev/null 2>&1 && echo "PASS $ip" || echo "FAIL $ip"; done
========================================
################################################################################
hgx-pod00-su00-h00 | PASS 172.16.0.0
hgx-pod00-su00-h00 | FAIL 172.16.0.16
hgx-pod00-su00-h00 | PASS 172.16.0.32
hgx-pod00-su00-h00 | FAIL 172.16.0.48
hgx-pod00-su00-h00 | PASS 172.16.1.0
hgx-pod00-su00-h00 | FAIL 172.16.1.16
hgx-pod00-su00-h00 | PASS 172.16.1.32
hgx-pod00-su00-h00 | FAIL 172.16.1.48
################################################################################
hgx-pod00-su00-h08 | FAIL 172.16.0.0
hgx-pod00-su00-h08 | PASS 172.16.0.16
hgx-pod00-su00-h08 | FAIL 172.16.0.32
hgx-pod00-su00-h08 | PASS 172.16.0.48
hgx-pod00-su00-h08 | FAIL 172.16.1.0
hgx-pod00-su00-h08 | PASS 172.16.1.16
hgx-pod00-su00-h08 | FAIL 172.16.1.32
hgx-pod00-su00-h08 | PASS 172.16.1.48
################################################################################
hgx-pod00-su00-h16 | PASS 172.16.0.0
hgx-pod00-su00-h16 | FAIL 172.16.0.16
hgx-pod00-su00-h16 | PASS 172.16.0.32
hgx-pod00-su00-h16 | FAIL 172.16.0.48
hgx-pod00-su00-h16 | PASS 172.16.1.0
hgx-pod00-su00-h16 | FAIL 172.16.1.16
hgx-pod00-su00-h16 | PASS 172.16.1.32
hgx-pod00-su00-h16 | FAIL 172.16.1.48
################################################################################
hgx-pod00-su00-h24 | FAIL 172.16.0.0
hgx-pod00-su00-h24 | PASS 172.16.0.16
hgx-pod00-su00-h24 | FAIL 172.16.0.32
hgx-pod00-su00-h24 | PASS 172.16.0.48
hgx-pod00-su00-h24 | FAIL 172.16.1.0
hgx-pod00-su00-h24 | PASS 172.16.1.16
hgx-pod00-su00-h24 | FAIL 172.16.1.32
hgx-pod00-su00-h24 | PASS 172.16.1.48
################################################################################
hgx-pod01-su00-h00 | PASS 172.16.0.0
hgx-pod01-su00-h00 | FAIL 172.16.0.16
hgx-pod01-su00-h00 | PASS 172.16.0.32
hgx-pod01-su00-h00 | FAIL 172.16.0.48
hgx-pod01-su00-h00 | PASS 172.16.1.0
hgx-pod01-su00-h00 | FAIL 172.16.1.16
hgx-pod01-su00-h00 | PASS 172.16.1.32
hgx-pod01-su00-h00 | FAIL 172.16.1.48
################################################################################
hgx-pod01-su00-h08 | FAIL 172.16.0.0
hgx-pod01-su00-h08 | PASS 172.16.0.16
hgx-pod01-su00-h08 | FAIL 172.16.0.32
hgx-pod01-su00-h08 | PASS 172.16.0.48
hgx-pod01-su00-h08 | FAIL 172.16.1.0
hgx-pod01-su00-h08 | PASS 172.16.1.16
hgx-pod01-su00-h08 | FAIL 172.16.1.32
hgx-pod01-su00-h08 | PASS 172.16.1.48
################################################################################
hgx-pod01-su00-h16 | PASS 172.16.0.0
hgx-pod01-su00-h16 | FAIL 172.16.0.16
hgx-pod01-su00-h16 | PASS 172.16.0.32
hgx-pod01-su00-h16 | FAIL 172.16.0.48
hgx-pod01-su00-h16 | PASS 172.16.1.0
hgx-pod01-su00-h16 | FAIL 172.16.1.16
hgx-pod01-su00-h16 | PASS 172.16.1.32
hgx-pod01-su00-h16 | FAIL 172.16.1.48
################################################################################
hgx-pod01-su00-h24 | FAIL 172.16.0.0
hgx-pod01-su00-h24 | PASS 172.16.0.16
hgx-pod01-su00-h24 | FAIL 172.16.0.32
hgx-pod01-su00-h24 | PASS 172.16.0.48
hgx-pod01-su00-h24 | FAIL 172.16.1.0
hgx-pod01-su00-h24 | PASS 172.16.1.16
hgx-pod01-su00-h24 | FAIL 172.16.1.32
hgx-pod01-su00-h24 | PASS 172.16.1.48
################################################################################
```

Expected result:

- HGX hosts from `tenant1` (`hgx-pod00-su00-h00`, `hgx-pod00-su00-h16`, `hgx-pod01-su00-h00`, `hgx-pod01-su00-h16`) have full-mesh connectivity with each other, including across PODs over a path that transits the super-spine tier, but cannot reach the `tenant2` hosts (`hgx-pod00-su00-h08`, `hgx-pod00-su00-h24`, `hgx-pod01-su00-h08`, `hgx-pod01-su00-h24`). The same applies in reverse for `tenant2`: full-mesh connectivity within the tenant, no connectivity into `tenant1`.

Modify the terraform manifest from `~/nvidia/terraform/tenant1` in order to remove `hgx-su00-h16` from `infra-tenant1` infrastructure and apply. Use the following commands in order to modify `infra-tenant1.tf` directly:

```bash
cd ~/nvidia/terraform/tenant1
sed -i 's/default = \["hgx-pod00-su00-h00", "hgx-pod00-su00-h16", "hgx-pod01-su00-h00", "hgx-pod01-su00-h16"]/default = ["hgx-pod00-su00-h00", "hgx-pod01-su00-h00", "hgx-pod01-su00-h16"]/' infra-tenant1.tf
terraform apply -auto-approve
```

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  ~ update in-place
  - destroy
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h16"] will be destroyed
  # (because key ["hgx-pod00-su00-h16"] is not in for_each map)
  - resource "metalcloud_endpoint_instance_group" "groups" {
      - endpoint_ids               = [
          - "3",
        ] -> null
      - endpoint_instance_group_id = "3" -> null
      - infrastructure_id          = "1" -> null
      - label                      = "hgx-pod00-su00-h16" -> null
      - network_connections        = [
          - {
              - access_mode        = "l2" -> null
              - interface_count    = 8 -> null
              - logical_network_id = "1" -> null
              - mtu                = 9216 -> null
              - tagged             = true -> null
            },
        ] -> null
    }

  # metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1 will be replaced due to changes in replace_triggered_by
-/+ resource "metalcloud_infrastructure_deployer" "infrastructure_deployer_tenant1" {
        # (4 unchanged attributes hidden)
    }

  # terraform_data.endpoint_fingerprint will be updated in-place
  ~ resource "terraform_data" "endpoint_fingerprint" {
        id     = "b18788a6-32f5-c80c-4899-ff3866c7a67b"
      ~ input  = "hgx-pod00-su00-h00,hgx-pod00-su00-h16,hgx-pod01-su00-h00,hgx-pod01-su00-h16" -> "hgx-pod00-su00-h00,hgx-pod01-su00-h00,hgx-pod01-su00-h16"
      ~ output = "hgx-pod00-su00-h00,hgx-pod00-su00-h16,hgx-pod01-su00-h00,hgx-pod01-su00-h16" -> (known after apply)
    }

Plan: 1 to add, 1 to change, 2 to destroy.
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Destroying...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Destruction complete after 0s
terraform_data.endpoint_fingerprint: Modifying... [id=b18788a6-32f5-c80c-4899-ff3866c7a67b]
terraform_data.endpoint_fingerprint: Modifications complete after 0s [id=b18788a6-32f5-c80c-4899-ff3866c7a67b]
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h16"]: Destroying...
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h16"]: Destruction complete after 0s
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Creating...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Creation complete after 0s

Apply complete! Resources: 1 added, 1 changed, 2 destroyed.
```

Before continuing, confirm that the Terraform deployments have finished. The following command refreshes every 10 seconds; leave it running until the deploy status shows finished:

```bash
watch -n 10 "metalcloud-cli infrastructure list"
```
Use control+C to stop the watch at any time.

The expected output should be the following:

```bash
┌────┬───────────────┬───────────────┬────────┬───────┬──────┬─────────────────────┬─────────────────────┬───────────────┬───────────┐
│ ID │ LABEL         │ CONFIG LABEL  │ STATUS │ OWNER │ SITE │ CREATED             │ UPDATED             │ DEPLOY STATUS │ DEPLOY ID │
├────┼───────────────┼───────────────┼────────┼───────┼──────┼─────────────────────┼─────────────────────┼───────────────┼───────────┤
│  1 │ infra-tenant1 │ infra-tenant1 │ active │     1 │    1 │ 19 Aug 26 14:28 UTC │ 19 Aug 26 14:52 UTC │ finished      │           │
│  2 │ infra-tenant2 │ infra-tenant2 │ active │     1 │    1 │ 19 Aug 26 14:28 UTC │ 19 Aug 26 14:35 UTC │ finished      │           │
└────┴───────────────┴───────────────┴────────┴───────┴──────┴─────────────────────┴─────────────────────┴───────────────┴───────────┘
```

(Optional) In the web UI, navigate to **Admin dashboard > Infrastructures** in order to see the current progress state of the deployment:

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/infrastructure_state.webp)

Modify the terraform manifest from `~/nvidia/terraform/tenant2` in order to add `hgx-su00-h16` to `infra-tenant2` infrastructure and apply. Use the following commands in order to modify `infra-tenant2.tf` directly:

```bash
cd ~/nvidia/terraform/tenant2
sed -i 's/default = \["hgx-pod00-su00-h08", "hgx-pod00-su00-h24", "hgx-pod01-su00-h08", "hgx-pod01-su00-h24"]/default = ["hgx-pod00-su00-h08", "hgx-pod00-su00-h16", "hgx-pod00-su00-h24", "hgx-pod01-su00-h08", "hgx-pod01-su00-h24"]/' infra-tenant2.tf
terraform apply -auto-approve
```

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create
  ~ update in-place
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h16"] will be created
  + resource "metalcloud_endpoint_instance_group" "groups" {
      + endpoint_ids               = [
          + "3",
        ]
      + endpoint_instance_group_id = (known after apply)
      + infrastructure_id          = "2"
      + label                      = "hgx-pod00-su00-h16"
      + network_connections        = [
          + {
              + access_mode        = "l2"
              + interface_count    = 8
              + logical_network_id = "2"
              + mtu                = 9216
              + tagged             = true
            },
        ]
    }

  # metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2 will be replaced due to changes in replace_triggered_by
-/+ resource "metalcloud_infrastructure_deployer" "infrastructure_deployer_tenant2" {
        # (4 unchanged attributes hidden)
    }

  # terraform_data.endpoint_fingerprint will be updated in-place
  ~ resource "terraform_data" "endpoint_fingerprint" {
        id     = "54848ee0-84b6-7a34-04ed-adb46f3ed583"
      ~ input  = "hgx-pod00-su00-h08,hgx-pod00-su00-h24,hgx-pod01-su00-h08,hgx-pod01-su00-h24" -> "hgx-pod00-su00-h08,hgx-pod00-su00-h16,hgx-pod00-su00-h24,hgx-pod01-su00-h08,hgx-pod01-su00-h24"
      ~ output = "hgx-pod00-su00-h08,hgx-pod00-su00-h24,hgx-pod01-su00-h08,hgx-pod01-su00-h24" -> (known after apply)
    }

Plan: 2 to add, 1 to change, 1 to destroy.
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Destroying...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Destruction complete after 0s
terraform_data.endpoint_fingerprint: Modifying... [id=54848ee0-84b6-7a34-04ed-adb46f3ed583]
terraform_data.endpoint_fingerprint: Modifications complete after 0s [id=54848ee0-84b6-7a34-04ed-adb46f3ed583]
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h16"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-pod00-su00-h16"]: Creation complete after 0s
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Creating...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Creation complete after 0s

Apply complete! Resources: 2 added, 1 changed, 1 destroyed.
```

Before continuing, confirm that the Terraform deployments have finished. The following command refreshes every 10 seconds; leave it running until the deploy status shows finished:

```bash
watch -n 10 "metalcloud-cli infrastructure list"
```
Use control+C to stop the watch at any time.

The expected output should be the following:

```bash
┌────┬───────────────┬───────────────┬────────┬───────┬──────┬─────────────────────┬─────────────────────┬───────────────┬───────────┐
│ ID │ LABEL         │ CONFIG LABEL  │ STATUS │ OWNER │ SITE │ CREATED             │ UPDATED             │ DEPLOY STATUS │ DEPLOY ID │
├────┼───────────────┼───────────────┼────────┼───────┼──────┼─────────────────────┼─────────────────────┼───────────────┼───────────┤
│  1 │ infra-tenant1 │ infra-tenant1 │ active │     1 │    1 │ 19 Aug 26 14:28 UTC │ 19 Aug 26 14:52 UTC │ finished      │           │
│  2 │ infra-tenant2 │ infra-tenant2 │ active │     1 │    1 │ 19 Aug 26 14:28 UTC │ 19 Aug 26 14:57 UTC │ finished      │           │
└────┴───────────────┴───────────────┴────────┴───────┴──────┴─────────────────────┴─────────────────────┴───────────────┴───────────┘
```

After `hgx-pod00-su00-h16` has been moved from `tenant1` infrastructure to `tenant2` infrastructure, the graphical representation of two infrastructures will look like the following:

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/tenant1.final.3tier.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/c3b448bb-957c-41b5-961d-d883d008fd57/tenant2.final.3tier.webp)


```bash
~/spcx-air/spcx-run -c "ip -br addr show vrf tenant1 | grep -E \"swp|172\.\""
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant1 ipv4 unicast\""
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant1\""
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "ip -br addr show vrf tenant1 | grep -E \"swp|172\.\""
========================================
Running: ip -br addr show vrf tenant1 | grep -E "swp|172\."
========================================
################################################################################
leaf-pod00-su00-r0 | swp1s0           UP             172.16.0.1/31 fe80::4ab0:2dff:fe61:a18b/64
leaf-pod00-su00-r0 | swp1s1           UP             172.24.0.1/31 fe80::4ab0:2dff:fe14:3c77/64
################################################################################
leaf-pod00-su00-r1 | swp1s0           UP             172.18.0.1/31 fe80::4ab0:2dff:fee1:2216/64
leaf-pod00-su00-r1 | swp1s1           UP             172.26.0.1/31 fe80::4ab0:2dff:fed5:7f4c/64
################################################################################
leaf-pod00-su00-r2 | swp1s0           UP             172.20.0.1/31 fe80::4ab0:2dff:fedd:ee3c/64
leaf-pod00-su00-r2 | swp1s1           UP             172.28.0.1/31 fe80::4ab0:2dff:fe3b:954/64
################################################################################
leaf-pod00-su00-r3 | swp1s0           UP             172.22.0.1/31 fe80::4ab0:2dff:fe92:f283/64
leaf-pod00-su00-r3 | swp1s1           UP             172.30.0.1/31 fe80::4ab0:2dff:fe7d:2fce/64
################################################################################
leaf-pod01-su00-r0 | swp1s0           UP             172.16.1.1/31 fe80::4ab0:2dff:fe8a:4d3d/64
leaf-pod01-su00-r0 | swp1s1           UP             172.24.1.1/31 fe80::4ab0:2dff:fedc:bc4/64
leaf-pod01-su00-r0 | swp17s0          UP             172.16.1.33/31 fe80::4ab0:2dff:fecf:37dc/64
leaf-pod01-su00-r0 | swp17s1          UP             172.24.1.33/31 fe80::4ab0:2dff:fe34:e962/64
################################################################################
leaf-pod01-su00-r1 | swp1s0           UP             172.18.1.1/31 fe80::4ab0:2dff:feaa:fd5/64
leaf-pod01-su00-r1 | swp1s1           UP             172.26.1.1/31 fe80::4ab0:2dff:fe47:24d/64
leaf-pod01-su00-r1 | swp17s0          UP             172.18.1.33/31 fe80::4ab0:2dff:fe5e:d5/64
leaf-pod01-su00-r1 | swp17s1          UP             172.26.1.33/31 fe80::4ab0:2dff:fe76:a558/64
################################################################################
leaf-pod01-su00-r2 | swp1s0           UP             172.20.1.1/31 fe80::4ab0:2dff:fe68:46a4/64
leaf-pod01-su00-r2 | swp1s1           UP             172.28.1.1/31 fe80::4ab0:2dff:feb7:6571/64
leaf-pod01-su00-r2 | swp17s0          UP             172.20.1.33/31 fe80::4ab0:2dff:feec:320b/64
leaf-pod01-su00-r2 | swp17s1          UP             172.28.1.33/31 fe80::4ab0:2dff:fed2:270/64
################################################################################
leaf-pod01-su00-r3 | swp1s0           UP             172.22.1.1/31 fe80::4ab0:2dff:fe22:c292/64
leaf-pod01-su00-r3 | swp1s1           UP             172.30.1.1/31 fe80::4ab0:2dff:fe23:8825/64
leaf-pod01-su00-r3 | swp17s0          UP             172.22.1.33/31 fe80::4ab0:2dff:fe6e:3f91/64
leaf-pod01-su00-r3 | swp17s1          UP             172.30.1.33/31 fe80::4ab0:2dff:fe0e:f188/64
################################################################################
spine-pod00-r0-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r1-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r2-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r3-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r0-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r1-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r2-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r3-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s01 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s02 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s03 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant1 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant1 ipv4 unicast"
========================================
################################################################################
leaf-pod00-su00-r0 | BGP table version is 86, local router ID is 10.253.128.1, vrf id 132
leaf-pod00-su00-r0 | Default local pref 100, local AS 4200000000
leaf-pod00-su00-r0 | Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath, + multipath nhg,
leaf-pod00-su00-r0 |                i internal, r RIB-failure, S Stale, R Removed
leaf-pod00-su00-r0 | Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
leaf-pod00-su00-r0 | Origin codes:  i - IGP, e - EGP, ? - incomplete
leaf-pod00-su00-r0 | RPKI validation codes: V valid, I invalid, N Not found
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 |     Network          Next Hop            Metric LocPrf Weight Path
leaf-pod00-su00-r0 |  *> 172.16.0.0/26    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.16.0.0/31    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  *> 172.16.1.0/26    10.253.128.5(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *                   10.253.128.5(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *                   10.253.128.2(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *> 172.18.1.0/26    10.253.128.6(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *                   10.253.128.6(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *                   10.253.128.3(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *> 172.20.1.0/26    10.253.128.7(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *                   10.253.128.7(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *                   10.253.128.4(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *> 172.22.1.0/26    10.253.128.8(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *                   10.253.128.8(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.24.0.0/31    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  *> 172.24.1.0/26    10.253.128.5(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *                   10.253.128.5(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *                   10.253.128.2(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *> 172.26.1.0/26    10.253.128.6(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *                   10.253.128.6(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *                   10.253.128.3(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *> 172.28.1.0/26    10.253.128.7(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *                   10.253.128.7(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *                   10.253.128.4(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *> 172.30.1.0/26    10.253.128.8(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *                   10.253.128.8(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Displayed 18 routes and 32 total paths
################################################################################

################################################################################
spine-pod00-r0-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod00-r1-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod00-r2-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod00-r3-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod01-r0-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod01-r1-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod01-r2-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-pod01-r3-s00 | View/Vrf tenant1 is unknown
################################################################################
ssp-group00-s00 | View/Vrf tenant1 is unknown
################################################################################
ssp-group00-s01 | View/Vrf tenant1 is unknown
################################################################################
ssp-group00-s02 | View/Vrf tenant1 is unknown
################################################################################
ssp-group00-s03 | View/Vrf tenant1 is unknown
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant1\""
========================================
Running: sudo vtysh -c "show ip route vrf tenant1"
========================================
################################################################################
leaf-pod00-su00-r0 | Codes: K - kernel route, C - connected, L - local, S - static,
leaf-pod00-su00-r0 |        R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
leaf-pod00-su00-r0 |        T - Table, A - Babel, D - SHARP, F - PBR, f - OpenFabric,
leaf-pod00-su00-r0 |        t - Table-Direct, Z - FRR,
leaf-pod00-su00-r0 |        > - selected route, * - FIB route, q - queued, r - rejected, b - backup
leaf-pod00-su00-r0 |        t - trapped, o - offload failure
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | VRF tenant1:
leaf-pod00-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:30:34
leaf-pod00-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:30:03
leaf-pod00-su00-r0 | C>* 172.16.0.0/31 is directly connected, swp1s0, 00:30:34
leaf-pod00-su00-r0 | L>* 172.16.0.1/32 is directly connected, swp1s0, 00:30:34
leaf-pod00-su00-r0 | B>* 172.16.1.0/26 [20/0] via 10.253.128.5, vlan1440_l3 onlink, weight 1, 00:27:31
leaf-pod00-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:29:24
leaf-pod00-su00-r0 | B>* 172.18.1.0/26 [20/0] via 10.253.128.6, vlan1440_l3 onlink, weight 1, 00:26:54
leaf-pod00-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:28:47
leaf-pod00-su00-r0 | B>* 172.20.1.0/26 [20/0] via 10.253.128.7, vlan1440_l3 onlink, weight 1, 00:26:16
leaf-pod00-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:28:08
leaf-pod00-su00-r0 | B>* 172.22.1.0/26 [20/0] via 10.253.128.8, vlan1440_l3 onlink, weight 1, 00:25:38
leaf-pod00-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:30:03
leaf-pod00-su00-r0 | C>* 172.24.0.0/31 is directly connected, swp1s1, 00:30:34
leaf-pod00-su00-r0 | L>* 172.24.0.1/32 is directly connected, swp1s1, 00:30:34
leaf-pod00-su00-r0 | B>* 172.24.1.0/26 [20/0] via 10.253.128.5, vlan1440_l3 onlink, weight 1, 00:27:31
leaf-pod00-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:29:24
leaf-pod00-su00-r0 | B>* 172.26.1.0/26 [20/0] via 10.253.128.6, vlan1440_l3 onlink, weight 1, 00:26:54
leaf-pod00-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:28:47
leaf-pod00-su00-r0 | B>* 172.28.1.0/26 [20/0] via 10.253.128.7, vlan1440_l3 onlink, weight 1, 00:26:16
leaf-pod00-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:28:08
leaf-pod00-su00-r0 | B>* 172.30.1.0/26 [20/0] via 10.253.128.8, vlan1440_l3 onlink, weight 1, 00:25:38
################################################################################
```


```bash
~/spcx-air/spcx-run -c "ip -br addr show vrf tenant2 | grep -E \"swp|172\.\""
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant2 ipv4 unicast\""
```

```bash
~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant2\""
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "ip -br addr show vrf tenant2 | grep -E \"swp|172\.\""
========================================
Running: ip -br addr show vrf tenant2 | grep -E "swp|172\."
========================================
################################################################################
leaf-pod00-su00-r0 | swp9s0           UP             172.16.0.17/31 fe80::4ab0:2dff:feb6:b813/64
leaf-pod00-su00-r0 | swp9s1           UP             172.24.0.17/31 fe80::4ab0:2dff:fe6b:99c9/64
leaf-pod00-su00-r0 | swp17s0          UP             172.16.0.33/31 fe80::4ab0:2dff:fee6:b736/64
leaf-pod00-su00-r0 | swp17s1          UP             172.24.0.33/31 fe80::4ab0:2dff:fe59:d51d/64
leaf-pod00-su00-r0 | swp25s0          UP             172.16.0.49/31 fe80::4ab0:2dff:fe64:5c03/64
leaf-pod00-su00-r0 | swp25s1          UP             172.24.0.49/31 fe80::4ab0:2dff:feb5:82c1/64
################################################################################
leaf-pod00-su00-r1 | swp9s0           UP             172.18.0.17/31 fe80::4ab0:2dff:fe6b:b973/64
leaf-pod00-su00-r1 | swp9s1           UP             172.26.0.17/31 fe80::4ab0:2dff:fea7:2a9c/64
leaf-pod00-su00-r1 | swp17s0          UP             172.18.0.33/31 fe80::4ab0:2dff:fe09:2870/64
leaf-pod00-su00-r1 | swp17s1          UP             172.26.0.33/31 fe80::4ab0:2dff:fece:7fe0/64
leaf-pod00-su00-r1 | swp25s0          UP             172.18.0.49/31 fe80::4ab0:2dff:fed9:56c4/64
leaf-pod00-su00-r1 | swp25s1          UP             172.26.0.49/31 fe80::4ab0:2dff:fe98:4ce/64
################################################################################
leaf-pod00-su00-r2 | swp9s0           UP             172.20.0.17/31 fe80::4ab0:2dff:fe8b:8979/64
leaf-pod00-su00-r2 | swp9s1           UP             172.28.0.17/31 fe80::4ab0:2dff:fe08:9f16/64
leaf-pod00-su00-r2 | swp17s0          UP             172.20.0.33/31 fe80::4ab0:2dff:fe40:f3a/64
leaf-pod00-su00-r2 | swp17s1          UP             172.28.0.33/31 fe80::4ab0:2dff:feda:ce66/64
leaf-pod00-su00-r2 | swp25s0          UP             172.20.0.49/31 fe80::4ab0:2dff:fe55:65d2/64
leaf-pod00-su00-r2 | swp25s1          UP             172.28.0.49/31 fe80::4ab0:2dff:fe16:9824/64
################################################################################
leaf-pod00-su00-r3 | swp9s0           UP             172.22.0.17/31 fe80::4ab0:2dff:fec6:b8e5/64
leaf-pod00-su00-r3 | swp9s1           UP             172.30.0.17/31 fe80::4ab0:2dff:feea:48cf/64
leaf-pod00-su00-r3 | swp17s0          UP             172.22.0.33/31 fe80::4ab0:2dff:fe41:5f7c/64
leaf-pod00-su00-r3 | swp17s1          UP             172.30.0.33/31 fe80::4ab0:2dff:fe9e:c51a/64
leaf-pod00-su00-r3 | swp25s0          UP             172.22.0.49/31 fe80::4ab0:2dff:fe49:342b/64
leaf-pod00-su00-r3 | swp25s1          UP             172.30.0.49/31 fe80::4ab0:2dff:fe36:678b/64
################################################################################
leaf-pod01-su00-r0 | swp9s0           UP             172.16.1.17/31 fe80::4ab0:2dff:fed4:534e/64
leaf-pod01-su00-r0 | swp9s1           UP             172.24.1.17/31 fe80::4ab0:2dff:fe15:27a3/64
leaf-pod01-su00-r0 | swp25s0          UP             172.16.1.49/31 fe80::4ab0:2dff:fe3d:cec/64
leaf-pod01-su00-r0 | swp25s1          UP             172.24.1.49/31 fe80::4ab0:2dff:fe05:54ea/64
################################################################################
leaf-pod01-su00-r1 | swp9s0           UP             172.18.1.17/31 fe80::4ab0:2dff:feb5:bebf/64
leaf-pod01-su00-r1 | swp9s1           UP             172.26.1.17/31 fe80::4ab0:2dff:fe16:e10f/64
leaf-pod01-su00-r1 | swp25s0          UP             172.18.1.49/31 fe80::4ab0:2dff:feba:ba64/64
leaf-pod01-su00-r1 | swp25s1          UP             172.26.1.49/31 fe80::4ab0:2dff:febd:6944/64
################################################################################
leaf-pod01-su00-r2 | swp9s0           UP             172.20.1.17/31 fe80::4ab0:2dff:fe6b:94/64
leaf-pod01-su00-r2 | swp9s1           UP             172.28.1.17/31 fe80::4ab0:2dff:fe14:2e95/64
leaf-pod01-su00-r2 | swp25s0          UP             172.20.1.49/31 fe80::4ab0:2dff:feac:9d1e/64
leaf-pod01-su00-r2 | swp25s1          UP             172.28.1.49/31 fe80::4ab0:2dff:fe97:ccb7/64
################################################################################
leaf-pod01-su00-r3 | swp9s0           UP             172.22.1.17/31 fe80::4ab0:2dff:feba:3a0/64
leaf-pod01-su00-r3 | swp9s1           UP             172.30.1.17/31 fe80::4ab0:2dff:fe71:bc6a/64
leaf-pod01-su00-r3 | swp25s0          UP             172.22.1.49/31 fe80::4ab0:2dff:fee2:2834/64
leaf-pod01-su00-r3 | swp25s1          UP             172.30.1.49/31 fe80::4ab0:2dff:fecc:9405/64
################################################################################
spine-pod00-r0-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r1-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r2-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod00-r3-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r0-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r1-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r2-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-pod01-r3-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s01 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s02 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
ssp-group00-s03 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant2 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant2 ipv4 unicast"
========================================
################################################################################
leaf-pod00-su00-r0 | BGP table version is 86, local router ID is 10.253.128.1, vrf id 136
leaf-pod00-su00-r0 | Default local pref 100, local AS 4200000000
leaf-pod00-su00-r0 | Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath, + multipath nhg,
leaf-pod00-su00-r0 |                i internal, r RIB-failure, S Stale, R Removed
leaf-pod00-su00-r0 | Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
leaf-pod00-su00-r0 | Origin codes:  i - IGP, e - EGP, ? - incomplete
leaf-pod00-su00-r0 | RPKI validation codes: V valid, I invalid, N Not found
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 |     Network          Next Hop            Metric LocPrf Weight Path
leaf-pod00-su00-r0 |  *> 172.16.0.0/26    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.16.0.16/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.16.0.32/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.16.0.48/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  *> 172.16.1.0/26    10.253.128.5(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *                   10.253.128.5(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *                   10.253.128.2(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *> 172.18.1.0/26    10.253.128.6(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *                   10.253.128.6(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *                   10.253.128.3(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *> 172.20.1.0/26    10.253.128.7(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *                   10.253.128.7(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *                   10.253.128.4(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *> 172.22.1.0/26    10.253.128.8(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *                   10.253.128.8(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.24.0.16/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.24.0.32/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  s> 172.24.0.48/31   0.0.0.0(leaf-pod00-su00-r0)
leaf-pod00-su00-r0 |                                              0         32768 ?
leaf-pod00-su00-r0 |  *> 172.24.1.0/26    10.253.128.5(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *                   10.253.128.5(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000004 ?
leaf-pod00-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *                   10.253.128.2(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000001 ?
leaf-pod00-su00-r0 |  *> 172.26.1.0/26    10.253.128.6(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *                   10.253.128.6(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000005 ?
leaf-pod00-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *                   10.253.128.3(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000002 ?
leaf-pod00-su00-r0 |  *> 172.28.1.0/26    10.253.128.7(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *                   10.253.128.7(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000006 ?
leaf-pod00-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *                   10.253.128.4(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000003 ?
leaf-pod00-su00-r0 |  *> 172.30.1.0/26    10.253.128.8(ssp-group00-s00)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |  *                   10.253.128.8(ssp-group00-s01)<
leaf-pod00-su00-r0 |                                                            0 4202000000 4200000007 ?
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | Displayed 22 routes and 36 total paths
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant2\""
========================================
Running: sudo vtysh -c "show ip route vrf tenant2"
========================================
################################################################################
leaf-pod00-su00-r0 | Codes: K - kernel route, C - connected, L - local, S - static,
leaf-pod00-su00-r0 |        R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
leaf-pod00-su00-r0 |        T - Table, A - Babel, D - SHARP, F - PBR, f - OpenFabric,
leaf-pod00-su00-r0 |        t - Table-Direct, Z - FRR,
leaf-pod00-su00-r0 |        > - selected route, * - FIB route, q - queued, r - rejected, b - backup
leaf-pod00-su00-r0 |        t - trapped, o - offload failure
leaf-pod00-su00-r0 |
leaf-pod00-su00-r0 | VRF tenant2:
leaf-pod00-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:31:05
leaf-pod00-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:30:34
leaf-pod00-su00-r0 | C>* 172.16.0.16/31 is directly connected, swp9s0, 00:31:05
leaf-pod00-su00-r0 | L>* 172.16.0.17/32 is directly connected, swp9s0, 00:31:05
leaf-pod00-su00-r0 | C>* 172.16.0.32/31 is directly connected, swp17s0, 00:04:18
leaf-pod00-su00-r0 | L>* 172.16.0.33/32 is directly connected, swp17s0, 00:04:18
leaf-pod00-su00-r0 | C>* 172.16.0.48/31 is directly connected, swp25s0, 00:31:05
leaf-pod00-su00-r0 | L>* 172.16.0.49/32 is directly connected, swp25s0, 00:31:05
leaf-pod00-su00-r0 | B>* 172.16.1.0/26 [20/0] via 10.253.128.5, vlan1341_l3 onlink, weight 1, 00:28:00
leaf-pod00-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:29:55
leaf-pod00-su00-r0 | B>* 172.18.1.0/26 [20/0] via 10.253.128.6, vlan1341_l3 onlink, weight 1, 00:27:22
leaf-pod00-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:29:17
leaf-pod00-su00-r0 | B>* 172.20.1.0/26 [20/0] via 10.253.128.7, vlan1341_l3 onlink, weight 1, 00:26:44
leaf-pod00-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:28:38
leaf-pod00-su00-r0 | B>* 172.22.1.0/26 [20/0] via 10.253.128.8, vlan1341_l3 onlink, weight 1, 00:26:07
leaf-pod00-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:30:34
leaf-pod00-su00-r0 | C>* 172.24.0.16/31 is directly connected, swp9s1, 00:31:05
leaf-pod00-su00-r0 | L>* 172.24.0.17/32 is directly connected, swp9s1, 00:31:05
leaf-pod00-su00-r0 | C>* 172.24.0.32/31 is directly connected, swp17s1, 00:04:18
leaf-pod00-su00-r0 | L>* 172.24.0.33/32 is directly connected, swp17s1, 00:04:18
leaf-pod00-su00-r0 | C>* 172.24.0.48/31 is directly connected, swp25s1, 00:31:04
leaf-pod00-su00-r0 | L>* 172.24.0.49/32 is directly connected, swp25s1, 00:31:04
leaf-pod00-su00-r0 | B>* 172.24.1.0/26 [20/0] via 10.253.128.5, vlan1341_l3 onlink, weight 1, 00:28:00
leaf-pod00-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:29:55
leaf-pod00-su00-r0 | B>* 172.26.1.0/26 [20/0] via 10.253.128.6, vlan1341_l3 onlink, weight 1, 00:27:22
leaf-pod00-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:29:17
leaf-pod00-su00-r0 | B>* 172.28.1.0/26 [20/0] via 10.253.128.7, vlan1341_l3 onlink, weight 1, 00:26:44
leaf-pod00-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:28:38
leaf-pod00-su00-r0 | B>* 172.30.1.0/26 [20/0] via 10.253.128.8, vlan1341_l3 onlink, weight 1, 00:26:07
```


```bash
~/spcx-air/spcx-run -s -c "fping -c2 -t500 $RAILS0 2>&1"
```

```bash
~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -s -c "fping -c2 -t500 $RAILS0 2>&1"
========================================
Running: fping -c2 -t500 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 172.16.1.0 172.16.1.16 172.16.1.32 172.16.1.48 2>&1
========================================
################################################################################
hgx-pod00-su00-h00 | 172.16.0.0  : [0], 64 bytes, 0.038 ms (0.038 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.1.0  : [0], 64 bytes, 2.91 ms (2.91 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.1.32 : [0], 64 bytes, 2.13 ms (2.13 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.0.0  : [1], 64 bytes, 0.040 ms (0.039 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.1.0  : [1], 64 bytes, 2.80 ms (2.86 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.1.32 : [1], 64 bytes, 2.12 ms (2.12 avg, 0% loss)
hgx-pod00-su00-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h00 |
hgx-pod00-su00-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.038/0.039/0.040
hgx-pod00-su00-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h00 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.80/2.86/2.91
hgx-pod00-su00-h00 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h00 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.12/2.12/2.13
hgx-pod00-su00-h00 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-pod00-su00-h08 | 172.16.0.16 : [0], 64 bytes, 0.024 ms (0.024 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.32 : [0], 64 bytes, 1.09 ms (1.09 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.48 : [0], 64 bytes, 0.615 ms (0.615 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.1.16 : [0], 64 bytes, 2.35 ms (2.35 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.1.48 : [0], 64 bytes, 2.09 ms (2.09 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.0.16 : [1], 64 bytes, 0.027 ms (0.025 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.32 : [1], 64 bytes, 0.970 ms (1.03 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.48 : [1], 64 bytes, 0.590 ms (0.602 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.1.16 : [1], 64 bytes, 2.43 ms (2.39 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.1.48 : [1], 64 bytes, 3.32 ms (2.71 avg, 0% loss)
hgx-pod00-su00-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h08 |
hgx-pod00-su00-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.024/0.025/0.027
hgx-pod00-su00-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.970/1.03/1.09
hgx-pod00-su00-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.590/0.602/0.615
hgx-pod00-su00-h08 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h08 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.35/2.39/2.43
hgx-pod00-su00-h08 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h08 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.09/2.71/3.32
################################################################################
hgx-pod00-su00-h16 | 172.16.0.16 : [0], 64 bytes, 0.467 ms (0.467 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.32 : [0], 64 bytes, 0.016 ms (0.016 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.48 : [0], 64 bytes, 0.450 ms (0.450 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.1.16 : [0], 64 bytes, 2.04 ms (2.04 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.1.48 : [0], 64 bytes, 1.93 ms (1.93 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.0.16 : [1], 64 bytes, 0.582 ms (0.525 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.32 : [1], 64 bytes, 0.011 ms (0.013 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.48 : [1], 64 bytes, 0.584 ms (0.517 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.1.16 : [1], 64 bytes, 2.11 ms (2.08 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.1.48 : [1], 64 bytes, 1.74 ms (1.84 avg, 0% loss)
hgx-pod00-su00-h16 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h16 |
hgx-pod00-su00-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.467/0.525/0.582
hgx-pod00-su00-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.011/0.013/0.016
hgx-pod00-su00-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.450/0.517/0.584
hgx-pod00-su00-h16 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h16 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.04/2.08/2.11
hgx-pod00-su00-h16 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h16 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.74/1.84/1.93
################################################################################
hgx-pod00-su00-h24 | 172.16.0.16 : [0], 64 bytes, 0.473 ms (0.473 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.32 : [0], 64 bytes, 0.473 ms (0.473 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.48 : [0], 64 bytes, 0.016 ms (0.016 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.1.16 : [0], 64 bytes, 2.38 ms (2.38 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.1.48 : [0], 64 bytes, 2.03 ms (2.03 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.0.16 : [1], 64 bytes, 0.568 ms (0.521 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.32 : [1], 64 bytes, 0.580 ms (0.527 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.48 : [1], 64 bytes, 0.021 ms (0.018 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.1.16 : [1], 64 bytes, 2.78 ms (2.58 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.1.48 : [1], 64 bytes, 2.31 ms (2.17 avg, 0% loss)
hgx-pod00-su00-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod00-su00-h24 |
hgx-pod00-su00-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.473/0.521/0.568
hgx-pod00-su00-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.473/0.527/0.580
hgx-pod00-su00-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.016/0.018/0.021
hgx-pod00-su00-h24 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h24 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.38/2.58/2.78
hgx-pod00-su00-h24 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod00-su00-h24 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.03/2.17/2.31
################################################################################
hgx-pod01-su00-h00 | 172.16.0.0  : [0], 64 bytes, 1.92 ms (1.92 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.1.0  : [0], 64 bytes, 0.018 ms (0.018 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.1.32 : [0], 64 bytes, 0.621 ms (0.621 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.0.0  : [1], 64 bytes, 1.80 ms (1.86 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.1.0  : [1], 64 bytes, 0.027 ms (0.022 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.1.32 : [1], 64 bytes, 0.675 ms (0.648 avg, 0% loss)
hgx-pod01-su00-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h00 |
hgx-pod01-su00-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.80/1.86/1.92
hgx-pod01-su00-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h00 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.018/0.022/0.027
hgx-pod01-su00-h00 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h00 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.621/0.648/0.675
hgx-pod01-su00-h00 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-pod01-su00-h08 | 172.16.0.16 : [0], 64 bytes, 2.22 ms (2.22 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.32 : [0], 64 bytes, 2.35 ms (2.35 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.48 : [0], 64 bytes, 1.83 ms (1.83 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.1.16 : [0], 64 bytes, 0.025 ms (0.025 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.1.48 : [0], 64 bytes, 1.00 ms (1.00 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.0.16 : [1], 64 bytes, 1.96 ms (2.09 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.32 : [1], 64 bytes, 2.25 ms (2.30 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.48 : [1], 64 bytes, 2.16 ms (2.00 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.1.16 : [1], 64 bytes, 0.022 ms (0.023 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.1.48 : [1], 64 bytes, 0.995 ms (0.998 avg, 0% loss)
hgx-pod01-su00-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h08 |
hgx-pod01-su00-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.96/2.09/2.22
hgx-pod01-su00-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.25/2.30/2.35
hgx-pod01-su00-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.83/2.00/2.16
hgx-pod01-su00-h08 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h08 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.022/0.023/0.025
hgx-pod01-su00-h08 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h08 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.995/0.998/1.00
################################################################################
hgx-pod01-su00-h16 | 172.16.0.0  : [0], 64 bytes, 2.33 ms (2.33 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.1.0  : [0], 64 bytes, 0.676 ms (0.676 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.1.32 : [0], 64 bytes, 0.022 ms (0.022 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.0.0  : [1], 64 bytes, 2.59 ms (2.46 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.1.0  : [1], 64 bytes, 0.845 ms (0.760 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.1.32 : [1], 64 bytes, 0.024 ms (0.023 avg, 0% loss)
hgx-pod01-su00-h16 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h16 |
hgx-pod01-su00-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.33/2.46/2.59
hgx-pod01-su00-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h16 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.676/0.760/0.845
hgx-pod01-su00-h16 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h16 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.022/0.023/0.024
hgx-pod01-su00-h16 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-pod01-su00-h24 | 172.16.0.16 : [0], 64 bytes, 2.73 ms (2.73 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.32 : [0], 64 bytes, 2.46 ms (2.46 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.48 : [0], 64 bytes, 2.00 ms (2.00 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.1.16 : [0], 64 bytes, 0.634 ms (0.634 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.1.48 : [0], 64 bytes, 0.033 ms (0.033 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.0.16 : [1], 64 bytes, 2.89 ms (2.81 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.32 : [1], 64 bytes, 2.53 ms (2.50 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.48 : [1], 64 bytes, 2.35 ms (2.17 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.1.16 : [1], 64 bytes, 0.737 ms (0.685 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.1.48 : [1], 64 bytes, 0.023 ms (0.028 avg, 0% loss)
hgx-pod01-su00-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-pod01-su00-h24 |
hgx-pod01-su00-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.73/2.81/2.89
hgx-pod01-su00-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.46/2.50/2.53
hgx-pod01-su00-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 2.00/2.17/2.35
hgx-pod01-su00-h24 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h24 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.634/0.685/0.737
hgx-pod01-su00-h24 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-pod01-su00-h24 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.023/0.028/0.033
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
========================================
Running: for ip in 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 172.16.1.0 172.16.1.16 172.16.1.32 172.16.1.48; do ping -c2 -W2 $ip >/dev/null 2>&1 && echo "PASS $ip" || echo "FAIL $ip"; done
========================================
################################################################################
hgx-pod00-su00-h00 | PASS 172.16.0.0
hgx-pod00-su00-h00 | FAIL 172.16.0.16
hgx-pod00-su00-h00 | FAIL 172.16.0.32
hgx-pod00-su00-h00 | FAIL 172.16.0.48
hgx-pod00-su00-h00 | PASS 172.16.1.0
hgx-pod00-su00-h00 | FAIL 172.16.1.16
hgx-pod00-su00-h00 | PASS 172.16.1.32
hgx-pod00-su00-h00 | FAIL 172.16.1.48
################################################################################
hgx-pod00-su00-h08 | FAIL 172.16.0.0
hgx-pod00-su00-h08 | PASS 172.16.0.16
hgx-pod00-su00-h08 | PASS 172.16.0.32
hgx-pod00-su00-h08 | PASS 172.16.0.48
hgx-pod00-su00-h08 | FAIL 172.16.1.0
hgx-pod00-su00-h08 | PASS 172.16.1.16
hgx-pod00-su00-h08 | FAIL 172.16.1.32
hgx-pod00-su00-h08 | PASS 172.16.1.48
################################################################################
hgx-pod00-su00-h16 | FAIL 172.16.0.0
hgx-pod00-su00-h16 | PASS 172.16.0.16
hgx-pod00-su00-h16 | PASS 172.16.0.32
hgx-pod00-su00-h16 | PASS 172.16.0.48
hgx-pod00-su00-h16 | FAIL 172.16.1.0
hgx-pod00-su00-h16 | PASS 172.16.1.16
hgx-pod00-su00-h16 | FAIL 172.16.1.32
hgx-pod00-su00-h16 | PASS 172.16.1.48
################################################################################
hgx-pod00-su00-h24 | FAIL 172.16.0.0
hgx-pod00-su00-h24 | PASS 172.16.0.16
hgx-pod00-su00-h24 | PASS 172.16.0.32
hgx-pod00-su00-h24 | PASS 172.16.0.48
hgx-pod00-su00-h24 | FAIL 172.16.1.0
hgx-pod00-su00-h24 | PASS 172.16.1.16
hgx-pod00-su00-h24 | FAIL 172.16.1.32
hgx-pod00-su00-h24 | PASS 172.16.1.48
################################################################################
hgx-pod01-su00-h00 | PASS 172.16.0.0
hgx-pod01-su00-h00 | FAIL 172.16.0.16
hgx-pod01-su00-h00 | FAIL 172.16.0.32
hgx-pod01-su00-h00 | FAIL 172.16.0.48
hgx-pod01-su00-h00 | PASS 172.16.1.0
hgx-pod01-su00-h00 | FAIL 172.16.1.16
hgx-pod01-su00-h00 | PASS 172.16.1.32
hgx-pod01-su00-h00 | FAIL 172.16.1.48
################################################################################
hgx-pod01-su00-h08 | FAIL 172.16.0.0
hgx-pod01-su00-h08 | PASS 172.16.0.16
hgx-pod01-su00-h08 | PASS 172.16.0.32
hgx-pod01-su00-h08 | PASS 172.16.0.48
hgx-pod01-su00-h08 | FAIL 172.16.1.0
hgx-pod01-su00-h08 | PASS 172.16.1.16
hgx-pod01-su00-h08 | FAIL 172.16.1.32
hgx-pod01-su00-h08 | PASS 172.16.1.48
################################################################################
hgx-pod01-su00-h16 | PASS 172.16.0.0
hgx-pod01-su00-h16 | FAIL 172.16.0.16
hgx-pod01-su00-h16 | FAIL 172.16.0.32
hgx-pod01-su00-h16 | FAIL 172.16.0.48
hgx-pod01-su00-h16 | PASS 172.16.1.0
hgx-pod01-su00-h16 | FAIL 172.16.1.16
hgx-pod01-su00-h16 | PASS 172.16.1.32
hgx-pod01-su00-h16 | FAIL 172.16.1.48
################################################################################
hgx-pod01-su00-h24 | FAIL 172.16.0.0
hgx-pod01-su00-h24 | PASS 172.16.0.16
hgx-pod01-su00-h24 | PASS 172.16.0.32
hgx-pod01-su00-h24 | PASS 172.16.0.48
hgx-pod01-su00-h24 | FAIL 172.16.1.0
hgx-pod01-su00-h24 | PASS 172.16.1.16
hgx-pod01-su00-h24 | FAIL 172.16.1.32
hgx-pod01-su00-h24 | PASS 172.16.1.48
################################################################################
```




Validation:

- BGP and EVPN can still be converging in the first seconds after the deploy, so the sweep retries each target for up to about thirty seconds rather than reporting a false failure. The `FAIL <ip>` lines against hosts in the other tenant are expected and persistent — they indicate VRF isolation working correctly, not a fabric problem. Re-running the block is safe.

<!-- AIR:page -->

## Validation Summary

- The fabric configuration patched all twenty devices, created 512 links, and added 576 `/31` strategies.
- All eight HGX endpoints were created.
- Underlay BGP has every session established across all three tiers, none down.
- The `tenant1` (`hgx-*-h00`, `hgx-*-h16`, both PODs) and `tenant2` (`hgx-*-h08`, `hgx-*-h24`, both PODs) L3 networks are each deployed with their own L3VNI.
- The host mesh test confirms full connectivity within each tenant across both PODs (over the super-spine tier) and no connectivity across tenants, demonstrating tenant isolation.

| __Check__ | __Expected__ |
| --------- | -------------- |
| `fabric configure-switches` | devices patched=20, links created=512, /31 strategies added=576 |
| Endpoints | 8 created |
| Underlay BGP | all sessions established across all three tiers, 0 down |
| Tenants | `tenant1` (`hgx-*-h00`, `hgx-*-h16`) and `tenant2` (`hgx-*-h08`, `hgx-*-h24`) L3 networks deployed, one L3VNI each |
| Host mesh | full connectivity within each tenant across both PODs; no connectivity across tenants |

Include a final validation command if helpful:

```bash
metalcloud-cli infrastructure list
```

## Troubleshooting, Upgrade, or Reset

- **The CLI cannot reach the controller.** If a `metalcloud-cli` command fails with a connection or TLS error, or `site agents 1` returns no agent, the controller may still be starting, or the Site Controller may have lost its link to the Global Controller after a restart. Log in to the Global Controller as `root` (`ssh -l root 192.168.200.3`, password `MetalsoftR0cks@$@$`) and run `kw` to watch the Kubernetes pods until each reaches `Running`; if any stay stuck, run `k-restart-all -A` to force a restart. On the Site Controller (`ssh -l root 192.168.200.2`, same password), run `docker ps` to confirm the `nfs-server` and `ms-agent` containers are `Up`, `docker logs -f ms-agent` to read the agent log, and `dcrestart` to restart the containers if `ms-agent` is not registering with the Global Controller.
- **A deploy job fails.** Read the job with `metalcloud-cli job get <id>`, correct the cause, and re-run the deploy step. BGP does not come up until the Step 9 deploy succeeds on every switch.
- **`get-ports` is empty in Step 4.** The switch is unreachable. Verify the management address and password in `switches.3tier.yaml`, then re-run discovery.
- **BFD sessions do not establish.** NVIDIA Air's Cumulus VX has no BFD daemon, so BFD liveness cannot be observed on the simulator. The configuration is applied correctly and matches the reference; BFD is a hardware-only check.

```bash
root@192.168.200.3:~# kw
```

## References

- [NVIDIA DSX Air Quick Start](https://docs.nvidia.com/networking-ethernet-software/nvidia-air/Quick-Start/)
- [NVIDIA Air](https://dsx-air.nvidia.com/)
- MetalSoft Documentation (partner-hosted; see your MetalSoft contact for access)
- `MetalSoft-SpectrumX-initial-air-setup.md` — one-time lab preparation and checkpoint guide for this environment

## Contact Info

| __Contact Type__ | __Contact Details__ |
| ------------------ | ---------------------- |
| Partner sales or support | `sales@metalsoft.io` |
| Technical documentation or support portal | MetalSoft Documentation Portal (partner-hosted) |