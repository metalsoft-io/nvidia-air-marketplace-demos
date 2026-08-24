<!-- AIR:tour -->

# MetalSoft for NVIDIA Spectrum-X: 1-SU Demo (256 GPU)

MetalSoft is an intelligent orchestration platform that transforms fragmented on-premises hardware into high-performance, fast-changing, secure, workload-compliant infrastructure. It integrates servers, switches, and storage to provide a turnkey neocloud platform (NCP) solution.

This lab builds a single scalability-unit (256 GPU) NVIDIA Spectrum-X fabric from a clean state and attaches a tenant to it, using the MetalSoft CLI (`metalcloud-cli`) and Terraform. The fabric is a two-tier leaf-spine design running EVPN over eBGP on Cumulus Linux 5.14.0.

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/topology.1su.png)

The lab is preconfigured: it launches from a stored MetalSoft Spectrum-X checkpoint with the Global Controller, Site Controller, and Cumulus Linux switches already running and the CLI toolkit already staged on the jumpstation. You perform the fabric build, switch configuration, and tenant onboarding yourself during the lab; nothing is pre-built for you.

**IMPORTANT:** Allow about 40 minutes end to end; most of that time is switch deployment. Unless a step says otherwise, run every command on the `oob-mgmt-server` jumpstation.

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

A network or infrastructure engineer is evaluating MetalSoft as the orchestration layer for an NVIDIA Spectrum-X GPU fabric. Today, standing up a Spectrum-X fabric means hand-configuring every leaf and spine switch, a slow and error-prone process that does not scale past a handful of racks. The engineer wants to know whether a single control plane can describe the fabric declaratively, push a consistent configuration to every switch, and onboard a GPU tenant without ever opening a switch CLI.

In this lab, the engineer drives the entire fabric lifecycle from the jumpstation with `metalcloud-cli` and Terraform: create the fabric, import the switches, let MetalSoft discover the physical links, configure and deploy the Cumulus Linux underlay and EVPN overlay, register the HGX hosts, and onboard a tenant network. The lab closes with a full rail-mesh connectivity test across all four HGX hosts, which proves the fabric, the tenant network, and the host rail configuration all work together end to end.

## Features and Services

This demo includes the following features and services:

- MetalSoft Global Controller and Site Controller orchestrating a Cumulus Linux Spectrum-X fabric
- CLI-driven fabric creation, switch import, link discovery, and configuration templating (`metalcloud-cli`)
- Terraform-based tenant onboarding
- MetalSoft Fabric Manager and Infrastructure Designer web UI (optional; the lab can be run entirely from the CLI)
- Two-tier leaf-spine underlay with EVPN over eBGP
- HGX host rail networking (eight rail NICs per host) validated with a full mesh connectivity test

## What You Will Do in This Lab

- Stand up a 1-SU (256 GPU) Spectrum-X fabric from scratch with `metalcloud-cli`
- Configure and deploy the Cumulus Linux underlay and EVPN overlay
- Register the four HGX hosts as endpoints and onboard a tenant with Terraform
- Configure host rail networking and verify full-mesh RoCE connectivity across all rails

<!-- AIR:page -->

## Demo Topology Overview

The lab is a single scalability unit: six Cumulus Linux switches (four leaves and two spines) in a two-tier leaf-spine design, and four HGX hosts, each connected to the leaf layer over eight rail NICs. MetalSoft's Global Controller and Site Controller run alongside the fabric and are reachable from the jumpstation; nothing on the fabric is preconfigured; you build it in the Lab Flow section below.


### Device Naming

- Leaf: `leaf-su00-r0`, `leaf-su00-r1`, `leaf-su00-r2`, `leaf-su00-r3`
- Spine: `spine-s00`, `spine-s01`
- HGX host: `hgx-su00-h00`, `hgx-su00-h08`, `hgx-su00-h16`, `hgx-su00-h24`

### Devices

| __Role__ | __Device Names__ |
| -------- | ----------------- |
| Leaf | `leaf-su00-r0`, `leaf-su00-r1`, `leaf-su00-r2`, `leaf-su00-r3` |
| Spine | `spine-s00`, `spine-s01` |
| HGX host | `hgx-su00-h00`, `hgx-su00-h08`, `hgx-su00-h16`, `hgx-su00-h24` |
| Other | Jumpstation (`oob-mgmt-server`), Global Controller, Site Controller |

### Resources used for controllers

| __Hostname__ | __vCPU__ | __RAM__ |
| ------------ | -------- | ------- |
| Global Controller | 12 | 8 GB |
| Site Controller | 4 | 4 GB |

<!-- AIR:page -->

## Demo Topology Information

### IPAM

| __Hostname__ | __Role__ | __IP Address__ |
| ------------- | -------- | --------------- |
| `oob-mgmt-server` | Jumpstation | reached over the external SSH service, see [SSH Access](#ssh-access) |
| Global Controller | MetalSoft control plane and web UI | `192.168.200.3` |
| Site Controller | Site agent | `192.168.200.2` |
| Leaf and spine switches | Cumulus Linux 5.14.0 | `192.168.200.11` and up, one address per switch, in the order listed in `switches.1su.yaml` |
| HGX hosts | Ubuntu compute nodes | `192.168.200.17` to `.20` |

Once the fabric is deployed (Lab Flow, Step 6), every switch also carries a `10.253.128.x/32` loopback assigned in the same order as the management addresses above (`leaf-su00-r0` is `10.253.128.1`, `r1` is `.2`, and so on).

### Physical Connectivity

MetalSoft discovers the leaf-spine links automatically over LLDP; there is no cabling table to prepare by hand. Lab Flow Step 7 (Discover links and redeploy) shows how to confirm the discovered links from a switch, and the Fabric Manager's Fabric View shows the same links visually once they are imported.

<!-- AIR:page -->

## Demo Environment Access

### Load Time and Readiness

Allow the Global Controller and Site Controller a few minutes to finish booting after the lab starts. Lab Flow Step 1 shows how to confirm both are up and connected before you run anything else. The full lab takes about 40 minutes end to end, most of it switch deployment time in Step 9.

### Console Access

After the lab is loaded, double-click any node in the NVIDIA Air topology view to open its console in the browser.

### Device Credentials

| __Device__ | __Username__ | __Password__ | __IP Address / Access Method__ |
| ---------- | ------------- | ------------- | -------------------------------- |
| `oob-mgmt-server` (Jumpstation) | `ubuntu` | `nvidia` | NVIDIA Air console, then SSH (see [SSH Access](#ssh-access)) |
| Global Controller | `root` | `MetalsoftR0cks@$@$` | `ssh -l root 192.168.200.3` |
| Site Controller | `root` | `MetalsoftR0cks@$@$` | `ssh -l root 192.168.200.2` |
| MetalSoft web UI | `demo@metalsoft.io` | `MetalsoftR0cks@$@$` | `https://<external-host-name>:<https-service-port>` |
| MetalSoft Fabric Manager UI | `demo@metalsoft.io` | `MetalsoftR0cks@$@$` | `https://<external-host-name>:<https-service-port>/designer/dashboard` |
| Cumulus switches | `cumulus` | set in `switches.1su.yaml` | `ssh cumulus@<switch-ip>` |
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
cumulus-5.14-templates  ethernet-fabric.1su.yaml       l3-profile-tenant1.1su.yaml  netplan              route-domain-tenant1.1su.yaml  switches.1su.yaml
endpoints.1su.yaml      fabric-config.1su.l3evpn.yaml  l3-profile-tenant2.1su.yaml  oob-subnet.1su.yaml  route-domain-tenant2.1su.yaml  terraform
ubuntu@oob-mgmt-server:~/nvidia$
```

The switch password has been set in `switches.1su.yaml`.

For more background on NVIDIA Air services, SSH access, SSH keys, nodes, and consoles, see the [NVIDIA DSX Air Quick Start](https://docs.nvidia.com/networking-ethernet-software/nvidia-air/Quick-Start/).

### Web UI Access

This demo runs entirely from the command line, but the UI is useful to better understand the software's state.

The UI runs on the Global Controller; the steps below expose it and open it from your workstation.

In NVIDIA Air, there should already be a MetalSoft UI service enabled.

NVIDIA Air returns an external host name and port for the service (for example `worker-0375f999.dsx-air.nvidia.com` and `25990`). Note both; NVIDIA Air assigns a new external port on each restart.

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/metalsoft-https-service.webp)

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

Open `https://<external-host-name>:<https-service-port>/` in the browser, using the port from above, and log in:

- Username: `demo@metalsoft.io`
- Password: `MetalsoftR0cks@$@$`

For our example it should be `https://worker-0375f999.dsx-air.nvidia.com:25990/`

In case an error with `503 Service Unavailable` is displayed, that means that not all the services from the Global Controller are UP and running. Use the troubleshooting steps from Step 1 in case the error persists after 1-2 minutes.

Once logged in, you can reach every MetalSoft component from the "burger" menu at the top left, next to the logo:

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/admin-ui.webp)

| __Component__ | __Comments__ | __URL__ |
| -------------- | -------------- | -------- |
| Sites, Infrastructures & Global Configurations | Home of the application; create sites, see tenant infrastructures and ongoing deploys | `https://<external-host-name>:<https-service-port>/` |
| Fabric Manager | Manage network equipment | `https://<external-host-name>:<https-service-port>/designer/fabrics/1/topology` |
| Compute & Storage Manager | Not used in this demo | |
| Infrastructure Designer | Tenant's view | `https://<external-host-name>:<https-service-port>/designer/infrastructures/1` |

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
metalcloud-cli fabric create 1 spectrumx-1su-514 ethernet "Spectrum-X 1-SU (5.14)" \
        --config-source ethernet-fabric.1su.yaml
```

```bash
metalcloud-cli fabric activate 1
```

```bash
metalcloud-cli subnet create --config-source oob-subnet.1su.yaml
```

Expected result:

- The fabric `spectrumx-1su-514` exists, is active, and has an out-of-band management subnet. Keep the fabric label `spectrumx-1su-514`; the Terraform manifest in Step 12 resolves the fabric by that label.

Validation:

- (Optional) In the web UI, open the Fabric Manager to see the fabric that was created.

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/fabric.1su.webp)

### Step 3. Import the Switches

**Goal:** Register the six lab switches with the fabric.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; the switch management password is read from `switches.1su.yaml`.

**Expected wait time:** A few seconds.

Set `fabricId` in `switches.1su.yaml` to `1`, then import the six switches:

```bash
metalcloud-cli fabric import-devices 1 --config-source switches.1su.yaml
```

Expected result:

- All six switches (four leaves, two spines) appear as devices on the fabric.

Validation:

- (Optional) In the web UI, navigate to **Fabric Manager > Network devices** where equipment list shows the imported switches:

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/network-devices.1su.webp)

### Step 4. Discover Switch Interfaces

**Goal:** Trigger interface discovery on every switch and capture a clean baseline before any configuration is applied.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.1su.yaml`) on the switches.

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

- `get-ports` returns a non-empty list of `swpNsN` ports. If the list is empty, the switch is not reachable; check the address and password in `switches.1su.yaml`.

The switches are imported and reachable, but MetalSoft has not configured them yet. The switch hostnames for this topology are:

- Leaves: `leaf-su00-r0`, `leaf-su00-r1`, `leaf-su00-r2`, `leaf-su00-r3`
- Spines: `spine-s00`, `spine-s01`

Validation:

Log in as `cumulus` (the password is read from `switches.1su.yaml`) and capture the baseline, so later steps have something to compare against.

For ease of use, there is a validation script named `spcx-run` located in `~/spcx-air/` that can check the current configuration of the switches. An environment variable needs to be exported for the desired topology:

```bash
export SPCX_INVENTORY=~/spcx-air/inventory.1su.yml
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
leaf-su00-r0 | cumulus
################################################################################
leaf-su00-r1 | cumulus
################################################################################
leaf-su00-r2 | cumulus
################################################################################
leaf-su00-r3 | cumulus
################################################################################
spine-s00 | cumulus
################################################################################
spine-s01 | cumulus
################################################################################

ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br addr show lo"
========================================
Running: ip -br addr show lo
========================================
################################################################################
leaf-su00-r0 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-su00-r1 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-su00-r2 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-su00-r3 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-s01 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################

ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp summary\""
========================================
Running: sudo vtysh -c "show bgp summary"
========================================
################################################################################
leaf-su00-r0 | bgpd is not running
################################################################################
leaf-su00-r1 | bgpd is not running
################################################################################
leaf-su00-r2 | bgpd is not running
################################################################################
leaf-su00-r3 | bgpd is not running
################################################################################
spine-s00 | bgpd is not running
################################################################################
spine-s01 | bgpd is not running
################################################################################
```

Expected result:

- At this stage the loopback carries only `127.0.0.1/8`, no `10.253.x.x/32` address is assigned and BGP is not running.

### Step 5. Configure the Switches

**Goal:** Assign hostnames, ASNs, loopbacks, fabric point-to-point addresses, and host downlinks to every switch.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few minutes.

Set `fabricId` in `fabric-config.1su.l3evpn.yaml` to `1`:

```bash
metalcloud-cli fabric configure-switches 1 --config-source fabric-config.1su.l3evpn.yaml
```

Expected result:

- The command completes without error; the switch configuration is staged but not yet deployed.

Validation:

- Proceed to Step 6, which deploys and verifies this configuration on the switches.

### Step 6. Deploy the Underlay Addressing

**Goal:** Push hostnames, loopbacks, point-to-point addresses, and port settings to every switch.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.1su.yaml`) on the switches.

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
leaf-su00-r0 | leaf-su00-r0
################################################################################
leaf-su00-r1 | leaf-su00-r1
################################################################################
leaf-su00-r2 | leaf-su00-r2
################################################################################
leaf-su00-r3 | leaf-su00-r3
################################################################################
spine-s00 | spine-s00
################################################################################
spine-s01 | spine-s01
################################################################################

ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br addr show lo"
========================================
Running: ip -br addr show lo
========================================
################################################################################
leaf-su00-r0 | lo               UNKNOWN        127.0.0.1/8 10.253.128.1/32 ::1/128
################################################################################
leaf-su00-r1 | lo               UNKNOWN        127.0.0.1/8 10.253.128.2/32 ::1/128
################################################################################
leaf-su00-r2 | lo               UNKNOWN        127.0.0.1/8 10.253.128.3/32 ::1/128
################################################################################
leaf-su00-r3 | lo               UNKNOWN        127.0.0.1/8 10.253.128.4/32 ::1/128
################################################################################
spine-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.5/32 ::1/128
################################################################################
spine-s01 | lo               UNKNOWN        127.0.0.1/8 10.253.128.6/32 ::1/128
################################################################################

ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp summary\""
========================================
Running: sudo vtysh -c "show bgp summary"
========================================
################################################################################
leaf-su00-r0 | bgpd is not running
################################################################################
leaf-su00-r1 | bgpd is not running
################################################################################
leaf-su00-r2 | bgpd is not running
################################################################################
leaf-su00-r3 | bgpd is not running
################################################################################
spine-s00 | bgpd is not running
################################################################################
spine-s01 | bgpd is not running
################################################################################
```

The switch now reports its assigned hostname and a `10.253.128.x/32` loopback (`leaf-su00-r0` is `10.253.128.1`, `r1` is `.2`, and so on) and BGP is not running.

### Step 7. Discover Links and Redeploy

**Goal:** Re-scan the fabric so the discovered leaf-spine links become managed, then redeploy.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.1su.yaml`) on the switches.

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
leaf-su00-r0 | Interface  Speed  Type  Remote Host             Remote Port
leaf-su00-r0 | ---------  -----  ----  ----------------------  -----------
leaf-su00-r0 | eth0       1G     eth   oob-mgmt-switch-leaf-1  swp6
leaf-su00-r0 | swp33s0    1G     swp   spine-s00               swp1s0
leaf-su00-r0 | swp33s1    1G     swp   spine-s00               swp1s1
leaf-su00-r0 | swp34s0    1G     swp   spine-s00               swp2s0
leaf-su00-r0 | swp34s1    1G     swp   spine-s00               swp2s1
leaf-su00-r0 | swp35s0    1G     swp   spine-s00               swp3s0
leaf-su00-r0 | swp35s1    1G     swp   spine-s00               swp3s1
leaf-su00-r0 | swp36s0    1G     swp   spine-s00               swp4s0
leaf-su00-r0 | swp36s1    1G     swp   spine-s00               swp4s1
leaf-su00-r0 | swp37s0    1G     swp   spine-s00               swp5s0
leaf-su00-r0 | swp37s1    1G     swp   spine-s00               swp5s1
leaf-su00-r0 | swp38s0    1G     swp   spine-s00               swp6s0
leaf-su00-r0 | swp38s1    1G     swp   spine-s00               swp6s1
leaf-su00-r0 | swp39s0    1G     swp   spine-s00               swp7s0
leaf-su00-r0 | swp39s1    1G     swp   spine-s00               swp7s1
leaf-su00-r0 | swp40s0    1G     swp   spine-s00               swp8s0
leaf-su00-r0 | swp40s1    1G     swp   spine-s00               swp8s1
leaf-su00-r0 | swp41s0    1G     swp   spine-s00               swp9s0
leaf-su00-r0 | swp41s1    1G     swp   spine-s00               swp9s1
leaf-su00-r0 | swp42s0    1G     swp   spine-s00               swp10s0
leaf-su00-r0 | swp42s1    1G     swp   spine-s00               swp10s1
leaf-su00-r0 | swp43s0    1G     swp   spine-s00               swp11s0
leaf-su00-r0 | swp43s1    1G     swp   spine-s00               swp11s1
leaf-su00-r0 | swp44s0    1G     swp   spine-s00               swp12s0
leaf-su00-r0 | swp44s1    1G     swp   spine-s00               swp12s1
leaf-su00-r0 | swp45s0    1G     swp   spine-s00               swp13s0
leaf-su00-r0 | swp45s1    1G     swp   spine-s00               swp13s1
leaf-su00-r0 | swp46s0    1G     swp   spine-s00               swp14s0
leaf-su00-r0 | swp46s1    1G     swp   spine-s00               swp14s1
leaf-su00-r0 | swp47s0    1G     swp   spine-s00               swp15s0
leaf-su00-r0 | swp47s1    1G     swp   spine-s00               swp15s1
leaf-su00-r0 | swp48s0    1G     swp   spine-s00               swp16s0
leaf-su00-r0 | swp48s1    1G     swp   spine-s00               swp16s1
leaf-su00-r0 | swp49s0    1G     swp   spine-s01               swp1s0
leaf-su00-r0 | swp49s1    1G     swp   spine-s01               swp1s1
leaf-su00-r0 | swp50s0    1G     swp   spine-s01               swp2s0
leaf-su00-r0 | swp50s1    1G     swp   spine-s01               swp2s1
leaf-su00-r0 | swp51s0    1G     swp   spine-s01               swp3s0
leaf-su00-r0 | swp51s1    1G     swp   spine-s01               swp3s1
leaf-su00-r0 | swp52s0    1G     swp   spine-s01               swp4s0
leaf-su00-r0 | swp52s1    1G     swp   spine-s01               swp4s1
leaf-su00-r0 | swp53s0    1G     swp   spine-s01               swp5s0
leaf-su00-r0 | swp53s1    1G     swp   spine-s01               swp5s1
leaf-su00-r0 | swp54s0    1G     swp   spine-s01               swp6s0
leaf-su00-r0 | swp54s1    1G     swp   spine-s01               swp6s1
leaf-su00-r0 | swp55s0    1G     swp   spine-s01               swp7s0
leaf-su00-r0 | swp55s1    1G     swp   spine-s01               swp7s1
leaf-su00-r0 | swp56s0    1G     swp   spine-s01               swp8s0
leaf-su00-r0 | swp56s1    1G     swp   spine-s01               swp8s1
leaf-su00-r0 | swp57s0    1G     swp   spine-s01               swp9s0
leaf-su00-r0 | swp57s1    1G     swp   spine-s01               swp9s1
leaf-su00-r0 | swp58s0    1G     swp   spine-s01               swp10s0
leaf-su00-r0 | swp58s1    1G     swp   spine-s01               swp10s1
leaf-su00-r0 | swp59s0    1G     swp   spine-s01               swp11s0
leaf-su00-r0 | swp59s1    1G     swp   spine-s01               swp11s1
leaf-su00-r0 | swp60s0    1G     swp   spine-s01               swp12s0
leaf-su00-r0 | swp60s1    1G     swp   spine-s01               swp12s1
leaf-su00-r0 | swp61s0    1G     swp   spine-s01               swp13s0
leaf-su00-r0 | swp61s1    1G     swp   spine-s01               swp13s1
leaf-su00-r0 | swp62s0    1G     swp   spine-s01               swp14s0
leaf-su00-r0 | swp62s1    1G     swp   spine-s01               swp14s1
leaf-su00-r0 | swp63s0    1G     swp   spine-s01               swp15s0
leaf-su00-r0 | swp63s1    1G     swp   spine-s01               swp15s1
leaf-su00-r0 | swp64s0    1G     swp   spine-s01               swp16s0
leaf-su00-r0 | swp64s1    1G     swp   spine-s01               swp16s1
################################################################################
```

Each fabric-facing port lists the neighbour it discovered: a leaf sees its spines, and a spine sees the leaves below it.

- (Optional) In the web UI, navigate to **Fabric Manager > Fabrics > Select the desired fabric (spectrumx-1su-514) > Topology** where the Fabric View should look like this:

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/fabric-topology.1su.webp)

### Step 8. Register the Fabric Templates

**Goal:** Register the Cumulus 5.14 configuration templates that Step 9 deploys.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** About a minute; `--verify-render` renders every switch's configuration before writing.

`fabric-config.1su.l3evpn.yaml` names the Cumulus 5.14 templates under `cumulus-5.14-templates/`. Register them in two commands. The order matters: the base profile installs the QoS, adaptive-routing, and VTEP configuration that the BGP and EVPN profiles depend on.

```bash
metalcloud-cli fabric configure-freeform 1 \
        --config-source fabric-config.1su.l3evpn.yaml --verify-render
```

```bash
metalcloud-cli fabric configure-bgp 1 \
        --config-source fabric-config.1su.l3evpn.yaml --verify-render
```

Expected result:

- Both commands complete without error. `--verify-render` stops before writing if any switch fails to render.

Validation:

- (Optional) In the web UI, the registered templates appear under **Fabric Manager > Configuration Libraries > Network Device Configuration Templates**:

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/configuration_templates.webp)

### Step 9. Deploy the Fabric

**Goal:** Apply the base, underlay, overlay and QoS profiles to every switch in order, bringing up BGP and EVPN.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.1su.yaml`) on the switches.

**Expected wait time:** This is the longest step in the lab, about 10 to 12 minutes.

```bash
wait_for_job_group "$(metalcloud-cli fabric deploy 1 -f json | jq -r '.jobGroupId')"
```

Expected result:

- Every job reports success. Continue only once that is true; BGP and EVPN come up during this deploy.

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
leaf-su00-r0 |
leaf-su00-r0 | IPv4 Unicast Summary:
leaf-su00-r0 | BGP router identifier 10.253.128.1, local AS number 4200000000 VRF default vrf-id 0
leaf-su00-r0 | BGP table version 9
leaf-su00-r0 | RIB entries 11, using 1408 bytes of memory
leaf-su00-r0 | Peers 64, using 1280 KiB of memory
leaf-su00-r0 | Peer groups 2, using 128 bytes of memory
leaf-su00-r0 |
leaf-su00-r0 | Neighbor               V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r0 | spine-s00(10.254.0.1)  4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp1s0
leaf-su00-r0 | spine-s00(10.254.0.3)  4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp1s1
leaf-su00-r0 | spine-s00(10.254.0.5)  4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp2s0
leaf-su00-r0 | spine-s00(10.254.0.7)  4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp2s1
leaf-su00-r0 | spine-s00(10.254.0.9)  4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp3s0
leaf-su00-r0 | spine-s00(10.254.0.11) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp3s1
leaf-su00-r0 | spine-s00(10.254.0.13) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp4s0
leaf-su00-r0 | spine-s00(10.254.0.15) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp4s1
leaf-su00-r0 | spine-s00(10.254.0.17) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp5s0
leaf-su00-r0 | spine-s00(10.254.0.19) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp5s1
leaf-su00-r0 | spine-s00(10.254.0.21) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp6s0
leaf-su00-r0 | spine-s00(10.254.0.23) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp6s1
leaf-su00-r0 | spine-s00(10.254.0.25) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp7s0
leaf-su00-r0 | spine-s00(10.254.0.27) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp7s1
leaf-su00-r0 | spine-s00(10.254.0.29) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp8s0
leaf-su00-r0 | spine-s00(10.254.0.31) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp8s1
leaf-su00-r0 | spine-s00(10.254.0.33) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp9s0
leaf-su00-r0 | spine-s00(10.254.0.35) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp9s1
leaf-su00-r0 | spine-s00(10.254.0.37) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp10s0
leaf-su00-r0 | spine-s00(10.254.0.39) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp10s1
leaf-su00-r0 | spine-s00(10.254.0.41) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp11s0
leaf-su00-r0 | spine-s00(10.254.0.43) 4 4201000000        63        67        9    0    0 00:02:47            4        6 to_spine-s00_swp11s1
leaf-su00-r0 | spine-s00(10.254.0.45) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp12s0
leaf-su00-r0 | spine-s00(10.254.0.47) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp12s1
leaf-su00-r0 | spine-s00(10.254.0.49) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp13s0
leaf-su00-r0 | spine-s00(10.254.0.51) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp13s1
leaf-su00-r0 | spine-s00(10.254.0.53) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp14s0
leaf-su00-r0 | spine-s00(10.254.0.55) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp14s1
leaf-su00-r0 | spine-s00(10.254.0.57) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp15s0
leaf-su00-r0 | spine-s00(10.254.0.59) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp15s1
leaf-su00-r0 | spine-s00(10.254.0.61) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp16s0
leaf-su00-r0 | spine-s00(10.254.0.63) 4 4201000000        63        68        9    0    0 00:02:47            4        6 to_spine-s00_swp16s1
leaf-su00-r0 | spine-s01(10.254.1.1)  4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp1s0
leaf-su00-r0 | spine-s01(10.254.1.3)  4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp1s1
leaf-su00-r0 | spine-s01(10.254.1.5)  4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp2s0
leaf-su00-r0 | spine-s01(10.254.1.7)  4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp2s1
leaf-su00-r0 | spine-s01(10.254.1.9)  4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp3s0
leaf-su00-r0 | spine-s01(10.254.1.11) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp3s1
leaf-su00-r0 | spine-s01(10.254.1.13) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp4s0
leaf-su00-r0 | spine-s01(10.254.1.15) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp4s1
leaf-su00-r0 | spine-s01(10.254.1.17) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp5s0
leaf-su00-r0 | spine-s01(10.254.1.19) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp5s1
leaf-su00-r0 | spine-s01(10.254.1.21) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp6s0
leaf-su00-r0 | spine-s01(10.254.1.23) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp6s1
leaf-su00-r0 | spine-s01(10.254.1.25) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp7s0
leaf-su00-r0 | spine-s01(10.254.1.27) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp7s1
leaf-su00-r0 | spine-s01(10.254.1.29) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp8s0
leaf-su00-r0 | spine-s01(10.254.1.31) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp8s1
leaf-su00-r0 | spine-s01(10.254.1.33) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp9s0
leaf-su00-r0 | spine-s01(10.254.1.35) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp9s1
leaf-su00-r0 | spine-s01(10.254.1.37) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp10s0
leaf-su00-r0 | spine-s01(10.254.1.39) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp10s1
leaf-su00-r0 | spine-s01(10.254.1.41) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp11s0
leaf-su00-r0 | spine-s01(10.254.1.43) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp11s1
leaf-su00-r0 | spine-s01(10.254.1.45) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp12s0
leaf-su00-r0 | spine-s01(10.254.1.47) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp12s1
leaf-su00-r0 | spine-s01(10.254.1.49) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp13s0
leaf-su00-r0 | spine-s01(10.254.1.51) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp13s1
leaf-su00-r0 | spine-s01(10.254.1.53) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp14s0
leaf-su00-r0 | spine-s01(10.254.1.55) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp14s1
leaf-su00-r0 | spine-s01(10.254.1.57) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp15s0
leaf-su00-r0 | spine-s01(10.254.1.59) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp15s1
leaf-su00-r0 | spine-s01(10.254.1.61) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp16s0
leaf-su00-r0 | spine-s01(10.254.1.63) 4 4201000000        63        68        9    0    0 00:02:46            4        6 to_spine-s01_swp16s1
leaf-su00-r0 |
leaf-su00-r0 | Total number of neighbors 64
leaf-su00-r0 |
leaf-su00-r0 | L2VPN EVPN Summary:
leaf-su00-r0 | BGP router identifier 10.253.128.1, local AS number 4200000000 VRF default vrf-id 0
leaf-su00-r0 | BGP table version 0
leaf-su00-r0 | RIB entries 0, using 0 bytes of memory
leaf-su00-r0 | Peers 2, using 40 KiB of memory
leaf-su00-r0 | Peer groups 2, using 128 bytes of memory
leaf-su00-r0 |
leaf-su00-r0 | Neighbor                V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r0 | spine-s00(10.253.128.5) 4 4201000000        42        47        0    0    0 00:01:58            0        0 to_spine-s00_loopbac
leaf-su00-r0 | spine-s01(10.253.128.6) 4 4201000000        42        47        0    0    0 00:01:59            0        0 to_spine-s01_loopbac
leaf-su00-r0 |
leaf-su00-r0 | Total number of neighbors 2
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp l2vpn evpn summary\""
========================================
Running: sudo vtysh -c "show bgp l2vpn evpn summary"
========================================
################################################################################
leaf-su00-r0 | BGP router identifier 10.253.128.1, local AS number 4200000000 VRF default vrf-id 0
leaf-su00-r0 | BGP table version 0
leaf-su00-r0 | RIB entries 0, using 0 bytes of memory
leaf-su00-r0 | Peers 2, using 40 KiB of memory
leaf-su00-r0 | Peer groups 2, using 128 bytes of memory
leaf-su00-r0 |
leaf-su00-r0 | Neighbor                V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r0 | spine-s00(10.253.128.5) 4 4201000000        45        50        0    0    0 00:02:06            0        0 to_spine-s00_loopbac
leaf-su00-r0 | spine-s01(10.253.128.6) 4 4201000000        45        50        0    0    0 00:02:07            0        0 to_spine-s01_loopbac
leaf-su00-r0 |
leaf-su00-r0 | Total number of neighbors 2
################################################################################
leaf-su00-r1 | BGP router identifier 10.253.128.2, local AS number 4200000001 VRF default vrf-id 0
leaf-su00-r1 | BGP table version 0
leaf-su00-r1 | RIB entries 0, using 0 bytes of memory
leaf-su00-r1 | Peers 2, using 40 KiB of memory
leaf-su00-r1 | Peer groups 2, using 128 bytes of memory
leaf-su00-r1 |
leaf-su00-r1 | Neighbor                V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r1 | spine-s00(10.253.128.5) 4 4201000000        45        50        0    0    0 00:02:06            0        0 to_spine-s00_loopbac
leaf-su00-r1 | spine-s01(10.253.128.6) 4 4201000000        45        50        0    0    0 00:02:07            0        0 to_spine-s01_loopbac
leaf-su00-r1 |
leaf-su00-r1 | Total number of neighbors 2
################################################################################
leaf-su00-r2 | BGP router identifier 10.253.128.3, local AS number 4200000002 VRF default vrf-id 0
leaf-su00-r2 | BGP table version 0
leaf-su00-r2 | RIB entries 0, using 0 bytes of memory
leaf-su00-r2 | Peers 2, using 40 KiB of memory
leaf-su00-r2 | Peer groups 2, using 128 bytes of memory
leaf-su00-r2 |
leaf-su00-r2 | Neighbor                V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r2 | spine-s00(10.253.128.5) 4 4201000000        45        50        0    0    0 00:02:07            0        0 to_spine-s00_loopbac
leaf-su00-r2 | spine-s01(10.253.128.6) 4 4201000000        45        50        0    0    0 00:02:07            0        0 to_spine-s01_loopbac
leaf-su00-r2 |
leaf-su00-r2 | Total number of neighbors 2
################################################################################
leaf-su00-r3 | BGP router identifier 10.253.128.4, local AS number 4200000003 VRF default vrf-id 0
leaf-su00-r3 | BGP table version 0
leaf-su00-r3 | RIB entries 0, using 0 bytes of memory
leaf-su00-r3 | Peers 2, using 40 KiB of memory
leaf-su00-r3 | Peer groups 2, using 128 bytes of memory
leaf-su00-r3 |
leaf-su00-r3 | Neighbor                V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r3 | spine-s00(10.253.128.5) 4 4201000000        45        50        0    0    0 00:02:06            0        0 to_spine-s00_loopbac
leaf-su00-r3 | spine-s01(10.253.128.6) 4 4201000000        45        50        0    0    0 00:02:07            0        0 to_spine-s01_loopbac
leaf-su00-r3 |
leaf-su00-r3 | Total number of neighbors 2
################################################################################
spine-s00 | BGP router identifier 10.253.128.5, local AS number 4201000000 VRF default vrf-id 0
spine-s00 | BGP table version 0
spine-s00 | RIB entries 0, using 0 bytes of memory
spine-s00 | Peers 4, using 80 KiB of memory
spine-s00 | Peer groups 2, using 128 bytes of memory
spine-s00 |
spine-s00 | Neighbor                   V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
spine-s00 | leaf-su00-r0(10.253.128.1) 4 4200000000        45        45        0    0    0 00:02:07            0        0 to_leaf-su00-r0_loop
spine-s00 | leaf-su00-r1(10.253.128.2) 4 4200000001        45        45        0    0    0 00:02:07            0        0 to_leaf-su00-r1_loop
spine-s00 | leaf-su00-r2(10.253.128.3) 4 4200000002        45        45        0    0    0 00:02:07            0        0 to_leaf-su00-r2_loop
spine-s00 | leaf-su00-r3(10.253.128.4) 4 4200000003        45        45        0    0    0 00:02:07            0        0 to_leaf-su00-r3_loop
spine-s00 |
spine-s00 | Total number of neighbors 4
################################################################################
spine-s01 | BGP router identifier 10.253.128.6, local AS number 4201000000 VRF default vrf-id 0
spine-s01 | BGP table version 0
spine-s01 | RIB entries 0, using 0 bytes of memory
spine-s01 | Peers 4, using 80 KiB of memory
spine-s01 | Peer groups 2, using 128 bytes of memory
spine-s01 |
spine-s01 | Neighbor                   V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
spine-s01 | leaf-su00-r0(10.253.128.1) 4 4200000000        45        45        0    0    0 00:02:06            0        0 to_leaf-su00-r0_loop
spine-s01 | leaf-su00-r1(10.253.128.2) 4 4200000001        45        45        0    0    0 00:02:06            0        0 to_leaf-su00-r1_loop
spine-s01 | leaf-su00-r2(10.253.128.3) 4 4200000002        45        45        0    0    0 00:02:06            0        0 to_leaf-su00-r2_loop
spine-s01 | leaf-su00-r3(10.253.128.4) 4 4200000003        45        45        0    0    0 00:02:06            0        0 to_leaf-su00-r3_loop
spine-s01 |
spine-s01 | Total number of neighbors 4
################################################################################
```

Every switch now has established underlay sessions, each with a non-zero prefix count in place of `Idle` or `Active`. The EVPN overlay runs between the leaves and the relay spines `spine-s00` and `spine-s01`, so the EVPN summary is populated on the leaves and on those two spines.

### Step 10. Register the HGX Endpoints

**Goal:** Register the four HGX hosts as endpoints, each bound to eight leaf ports (one per rail).

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few seconds.

The definitions are in `endpoints.1su.yaml`:

```bash
metalcloud-cli endpoint create-bulk --config-source endpoints.1su.yaml
```

Expected result:

- All four endpoints are created.

Validation:

```bash
metalcloud-cli endpoint list --filter-site 1
```

Confirm all four endpoints are listed. Do not rename them; the Terraform manifest in Step 12 resolves them by label (`hgx-su00-h00`, `hgx-su00-h08`, `hgx-su00-h16`, `hgx-su00-h24`).

### Step 11. Create the Tenant Route Domains and Profiles

**Goal:** Create the tenant VRFs (route domain) and the L3-only logical network profiles that the tenant networks are built from.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few seconds.

```bash
metalcloud-cli route-domain create --config-source route-domain-tenant1.1su.yaml
```

```bash
metalcloud-cli route-domain create --config-source route-domain-tenant2.1su.yaml
```

```bash
metalcloud-cli logical-network-profile create vxlan --config-source l3-profile-tenant1.1su.yaml
```

```bash
metalcloud-cli logical-network-profile create vxlan --config-source l3-profile-tenant2.1su.yaml
```

Expected result:

- The route domains and network profiles are created. The L3VNI is allocated automatically for the fabric. Keep the profile labels as `tenant1-l3` and `tenant2-l3`; the Terraform manifest in Step 12 resolves them by that label.

Validation:

- Proceed to Step 12, which consumes the profiles.

### Step 12. Onboard the Tenant with Terraform

**Goal:** Create the tenant infrastructures, build L3 networks from the `tenant1-l3` and `tenant2-l3` profiles, attach the four endpoints with eight interfaces each, and deploy.

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
Plan: 6 to add, 0 to change, 0 to destroy.
terraform_data.endpoint_fingerprint: Creating...
terraform_data.endpoint_fingerprint: Creation complete after 0s [id=d78f8911-b46e-019c-a6ad-ef9497433c5c]
metalcloud_infrastructure.infra_tenant1: Creating...
metalcloud_infrastructure.infra_tenant1: Creation complete after 0s
metalcloud_logical_network.tenant1-network: Creating...
metalcloud_logical_network.tenant1-network: Creation complete after 0s [name=tenant1-network]
metalcloud_endpoint_instance_group.groups["hgx-su00-h00"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h00"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Creation complete after 0s
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Creating...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Creation complete after 0s

Apply complete! Resources: 6 added, 0 changed, 0 destroyed.
```

```bash
Plan: 6 to add, 0 to change, 0 to destroy.
metalcloud_infrastructure.infra_tenant2: Creating...
metalcloud_infrastructure.infra_tenant2: Creation complete after 0s
terraform_data.endpoint_fingerprint: Creating...
terraform_data.endpoint_fingerprint: Creation complete after 0s [id=a9dddcde-7bb7-d688-6cbc-9ed61393d339]
metalcloud_logical_network.tenant2-network: Creating...
metalcloud_logical_network.tenant2-network: Creation complete after 0s [name=tenant2-network]
metalcloud_endpoint_instance_group.groups["hgx-su00-h08"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h08"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-su00-h24"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h24"]: Creation complete after 0s
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Creating...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Creation complete after 0s

Apply complete! Resources: 6 added, 0 changed, 0 destroyed.
```

Before continuing with the checks, confirm that the Terraform deployments have finished. The following command refreshes every 10 seconds; leave it running until the deploy status shows finished:

```bash
watch -n 10 "metalcloud-cli infrastructure list"
```
Use control+C to stop the watch at any time.

The expected output should be the following:

```
┌────┬──────────────────┬──────────────────┬────────┬───────┬──────┬─────────────────────┬─────────────────────┬───────────────┬───────────┐
│ ID │ LABEL            │ CONFIG LABEL     │ STATUS │ OWNER │ SITE │ CREATED             │ UPDATED             │ DEPLOY STATUS │ DEPLOY ID │
├────┼──────────────────┼──────────────────┼────────┼───────┼──────┼─────────────────────┼─────────────────────┼───────────────┼───────────┤
│  3 │ infra-tenant1    │ infra-tenant1    │ active │     1 │    1 │ 12 Aug 26 20:43 UTC │ 12 Aug 26 20:48 UTC │ finished      │           │
│  4 │ infra-tenant2    │ infra-tenant2    │ active │     1 │    1 │ 12 Aug 26 20:44 UTC │ 12 Aug 26 20:48 UTC │ finished      │           │
└────┴──────────────────┴──────────────────┴────────┴───────┴──────┴─────────────────────┴─────────────────────┴───────────────┴───────────┘
```

- When the apply completes, the attached hosts' rail gateways are live.

Validation:

- (Optional) In the web UI, the tenant infrastructures are visible in the Infrastructure Designer. Navigate to **Admin dashboard > Infrastructures**, select one infrastructure and click **Open infrastructure designer**. This opens the Infrastructure Designer, where you can see the graphical representation of the infrastructure:

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/infrastructures.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/open-infrastructure.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/tenant1.1su.webp)

Terraform created the tenant VRFs, attached the endpoints and deployed. The tenant VRFs and the host rail gateways live on the leaves. Inspect a leaf:

```bash
~/spcx-air/spcx-run -c "ip -br link show type vrf"
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br link show type vrf"
========================================
Running: ip -br link show type vrf
========================================
################################################################################
leaf-su00-r0 | mgmt             UP             32:ea:99:4b:0e:cd <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r0 | tenant2          UP             6e:f2:15:1e:c7:2f <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r0 | tenant1          UP             b6:9f:d1:19:52:70 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su00-r1 | mgmt             UP             72:15:d0:6f:26:11 <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r1 | tenant2          UP             ce:8b:25:51:47:c5 <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r1 | tenant1          UP             4e:4f:67:50:b6:2f <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su00-r2 | mgmt             UP             9e:f7:c3:8e:98:31 <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r2 | tenant2          UP             4e:25:d9:96:e7:81 <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r2 | tenant1          UP             6a:c8:9f:d5:01:29 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su00-r3 | mgmt             UP             0a:66:c9:ed:70:50 <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r3 | tenant2          UP             7a:8f:98:ae:20:16 <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r3 | tenant1          UP             ca:49:14:9b:15:c0 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-s00 | mgmt             UP             be:38:c5:93:bd:eb <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-s01 | mgmt             UP             2a:ed:48:64:45:28 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
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
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br addr show vrf tenant1 | grep -E \"swp|172\.\""
========================================
Running: ip -br addr show vrf tenant1 | grep -E "swp|172\."
========================================
################################################################################
leaf-su00-r0 | swp1s0           UP             172.16.0.1/31 fe80::4ab0:2dff:fea2:de1c/64
leaf-su00-r0 | swp1s1           UP             172.24.0.1/31 fe80::4ab0:2dff:fe08:12c6/64
leaf-su00-r0 | swp17s0          UP             172.16.0.33/31 fe80::4ab0:2dff:fef5:49c4/64
leaf-su00-r0 | swp17s1          UP             172.24.0.33/31 fe80::4ab0:2dff:fe9e:e970/64
################################################################################
leaf-su00-r1 | swp1s0           UP             172.18.0.1/31 fe80::4ab0:2dff:fe30:371c/64
leaf-su00-r1 | swp1s1           UP             172.26.0.1/31 fe80::4ab0:2dff:fe9a:7591/64
leaf-su00-r1 | swp17s0          UP             172.18.0.33/31 fe80::4ab0:2dff:fecc:6255/64
leaf-su00-r1 | swp17s1          UP             172.26.0.33/31 fe80::4ab0:2dff:fe61:4efb/64
################################################################################
leaf-su00-r2 | swp1s0           UP             172.20.0.1/31 fe80::4ab0:2dff:fe93:1a35/64
leaf-su00-r2 | swp1s1           UP             172.28.0.1/31 fe80::4ab0:2dff:fe08:e8ff/64
leaf-su00-r2 | swp17s0          UP             172.20.0.33/31 fe80::4ab0:2dff:fe54:17de/64
leaf-su00-r2 | swp17s1          UP             172.28.0.33/31 fe80::4ab0:2dff:fe09:a1b/64
################################################################################
leaf-su00-r3 | swp1s0           UP             172.22.0.1/31 fe80::4ab0:2dff:fe0e:409e/64
leaf-su00-r3 | swp1s1           UP             172.30.0.1/31 fe80::4ab0:2dff:fe83:3fd5/64
leaf-su00-r3 | swp17s0          UP             172.22.0.33/31 fe80::4ab0:2dff:fe8f:aad2/64
leaf-su00-r3 | swp17s1          UP             172.30.0.33/31 fe80::4ab0:2dff:fe0c:f578/64
################################################################################
spine-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-s01 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant1 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant1 ipv4 unicast"
========================================
################################################################################
leaf-su00-r0 | BGP table version is 44, local router ID is 10.253.128.1, vrf id 136
leaf-su00-r0 | Default local pref 100, local AS 4200000000
leaf-su00-r0 | Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath, + multipath nhg,
leaf-su00-r0 |                i internal, r RIB-failure, S Stale, R Removed
leaf-su00-r0 | Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
leaf-su00-r0 | Origin codes:  i - IGP, e - EGP, ? - incomplete
leaf-su00-r0 | RPKI validation codes: V valid, I invalid, N Not found
leaf-su00-r0 |
leaf-su00-r0 |     Network          Next Hop            Metric LocPrf Weight Path
leaf-su00-r0 |  *> 172.16.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.16.0.0/31    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.16.0.32/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.0/31    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.32/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |
leaf-su00-r0 | Displayed 12 routes and 18 total paths
################################################################################

################################################################################
spine-s00 | View/Vrf tenant1 is unknown
################################################################################
spine-s01 | View/Vrf tenant1 is unknown
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant1\""
========================================
Running: sudo vtysh -c "show ip route vrf tenant1"
========================================
################################################################################
leaf-su00-r0 | Codes: K - kernel route, C - connected, L - local, S - static,
leaf-su00-r0 |        R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
leaf-su00-r0 |        T - Table, A - Babel, D - SHARP, F - PBR, f - OpenFabric,
leaf-su00-r0 |        t - Table-Direct, Z - FRR,
leaf-su00-r0 |        > - selected route, * - FIB route, q - queued, r - rejected, b - backup
leaf-su00-r0 |        t - trapped, o - offload failure
leaf-su00-r0 |
leaf-su00-r0 | VRF tenant1:
leaf-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:05:47
leaf-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:05:17
leaf-su00-r0 | C>* 172.16.0.0/31 is directly connected, swp1s0, 00:05:47
leaf-su00-r0 | L>* 172.16.0.1/32 is directly connected, swp1s0, 00:05:47
leaf-su00-r0 | C>* 172.16.0.32/31 is directly connected, swp17s0, 00:05:47
leaf-su00-r0 | L>* 172.16.0.33/32 is directly connected, swp17s0, 00:05:47
leaf-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:04:38
leaf-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:04:00
leaf-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:03:20
leaf-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:05:17
leaf-su00-r0 | C>* 172.24.0.0/31 is directly connected, swp1s1, 00:05:47
leaf-su00-r0 | L>* 172.24.0.1/32 is directly connected, swp1s1, 00:05:47
leaf-su00-r0 | C>* 172.24.0.32/31 is directly connected, swp17s1, 00:05:47
leaf-su00-r0 | L>* 172.24.0.33/32 is directly connected, swp17s1, 00:05:47
leaf-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:04:38
leaf-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:04:00
leaf-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:03:20
```

`tenant1` appears in the VRF list, the host-facing `swp` ports carry their `172.x` rail gateway `/31`s inside it and the VRF routing table holds both the local rail subnets and the remote ones learned over EVPN.

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
leaf-su00-r0 | swp9s0           UP             172.16.0.17/31 fe80::4ab0:2dff:feef:9af8/64
leaf-su00-r0 | swp9s1           UP             172.24.0.17/31 fe80::4ab0:2dff:fe00:de8f/64
leaf-su00-r0 | swp25s0          UP             172.16.0.49/31 fe80::4ab0:2dff:fe95:6e03/64
leaf-su00-r0 | swp25s1          UP             172.24.0.49/31 fe80::4ab0:2dff:fe9b:ee7/64
################################################################################
leaf-su00-r1 | swp9s0           UP             172.18.0.17/31 fe80::4ab0:2dff:fe21:cec3/64
leaf-su00-r1 | swp9s1           UP             172.26.0.17/31 fe80::4ab0:2dff:fe32:5c09/64
leaf-su00-r1 | swp25s0          UP             172.18.0.49/31 fe80::4ab0:2dff:fe11:c578/64
leaf-su00-r1 | swp25s1          UP             172.26.0.49/31 fe80::4ab0:2dff:fe3f:97ea/64
################################################################################
leaf-su00-r2 | swp9s0           UP             172.20.0.17/31 fe80::4ab0:2dff:fe34:eb83/64
leaf-su00-r2 | swp9s1           UP             172.28.0.17/31 fe80::4ab0:2dff:fe04:592d/64
leaf-su00-r2 | swp25s0          UP             172.20.0.49/31 fe80::4ab0:2dff:feb2:b0c5/64
leaf-su00-r2 | swp25s1          UP             172.28.0.49/31 fe80::4ab0:2dff:fe99:16e3/64
################################################################################
leaf-su00-r3 | swp9s0           UP             172.22.0.17/31 fe80::4ab0:2dff:fe81:43e2/64
leaf-su00-r3 | swp9s1           UP             172.30.0.17/31 fe80::4ab0:2dff:fe4a:bb39/64
leaf-su00-r3 | swp25s0          UP             172.22.0.49/31 fe80::4ab0:2dff:fea4:1e4c/64
leaf-su00-r3 | swp25s1          UP             172.30.0.49/31 fe80::4ab0:2dff:fe98:acee/64
################################################################################
spine-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-s01 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant2 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant2 ipv4 unicast"
========================================
################################################################################
leaf-su00-r0 | BGP table version is 44, local router ID is 10.253.128.1, vrf id 132
leaf-su00-r0 | Default local pref 100, local AS 4200000000
leaf-su00-r0 | Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath, + multipath nhg,
leaf-su00-r0 |                i internal, r RIB-failure, S Stale, R Removed
leaf-su00-r0 | Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
leaf-su00-r0 | Origin codes:  i - IGP, e - EGP, ? - incomplete
leaf-su00-r0 | RPKI validation codes: V valid, I invalid, N Not found
leaf-su00-r0 |
leaf-su00-r0 |     Network          Next Hop            Metric LocPrf Weight Path
leaf-su00-r0 |  *> 172.16.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.16.0.16/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.16.0.48/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.16/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.48/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |
leaf-su00-r0 | Displayed 12 routes and 18 total paths
################################################################################

################################################################################
spine-s00 | View/Vrf tenant2 is unknown
################################################################################
spine-s01 | View/Vrf tenant2 is unknown
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant2\""
========================================
Running: sudo vtysh -c "show ip route vrf tenant2"
========================================
################################################################################
leaf-su00-r0 | Codes: K - kernel route, C - connected, L - local, S - static,
leaf-su00-r0 |        R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
leaf-su00-r0 |        T - Table, A - Babel, D - SHARP, F - PBR, f - OpenFabric,
leaf-su00-r0 |        t - Table-Direct, Z - FRR,
leaf-su00-r0 |        > - selected route, * - FIB route, q - queued, r - rejected, b - backup
leaf-su00-r0 |        t - trapped, o - offload failure
leaf-su00-r0 |
leaf-su00-r0 | VRF tenant2:
leaf-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:06:47
leaf-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:06:17
leaf-su00-r0 | C>* 172.16.0.16/31 is directly connected, swp9s0, 00:06:47
leaf-su00-r0 | L>* 172.16.0.17/32 is directly connected, swp9s0, 00:06:47
leaf-su00-r0 | C>* 172.16.0.48/31 is directly connected, swp25s0, 00:06:47
leaf-su00-r0 | L>* 172.16.0.49/32 is directly connected, swp25s0, 00:06:47
leaf-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:05:37
leaf-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:04:59
leaf-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:04:20
leaf-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:06:17
leaf-su00-r0 | C>* 172.24.0.16/31 is directly connected, swp9s1, 00:06:47
leaf-su00-r0 | L>* 172.24.0.17/32 is directly connected, swp9s1, 00:06:47
leaf-su00-r0 | C>* 172.24.0.48/31 is directly connected, swp25s1, 00:06:47
leaf-su00-r0 | L>* 172.24.0.49/32 is directly connected, swp25s1, 00:06:47
leaf-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:05:37
leaf-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:04:59
leaf-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:04:20
################################################################################
```

`tenant2` appears in the VRF list, the host-facing `swp` ports carry their `172.x` rail gateway `/31`s inside it, and the VRF routing table holds both the local rail subnets and the remote ones learned over EVPN.


The spines do not hold the tenant VRF; that lives on the leaves. On the EVPN overlay relay you can see the type-5 host routes it re-advertises:

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
leaf-su00-r0 | BGP table version is 10, local router ID is 10.253.128.1
leaf-su00-r0 | Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
leaf-su00-r0 | Origin codes: i - IGP, e - EGP, ? - incomplete
leaf-su00-r0 | EVPN type-1 prefix: [1]:[EthTag]:[ESI]:[IPlen]:[VTEP-IP]:[Frag-id]
leaf-su00-r0 | EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
leaf-su00-r0 | EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
leaf-su00-r0 | EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
leaf-su00-r0 | EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]
leaf-su00-r0 |
leaf-su00-r0 |    Network          Next Hop            Metric LocPrf Weight Path
leaf-su00-r0 |                     Extended Community
leaf-su00-r0 | Route Distinguisher: 10.253.128.1:19998
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.16.0.0] RD 10.253.128.1:19998
leaf-su00-r0 |                     10.253.128.1 (leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |                     ET:8 RT:59904:19998 Rmac:44:38:39:22:01:cd
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.24.0.0] RD 10.253.128.1:19998
leaf-su00-r0 |                     10.253.128.1 (leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |                     ET:8 RT:59904:19998 Rmac:44:38:39:22:01:cd
leaf-su00-r0 | Route Distinguisher: 10.253.128.1:19999
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.16.0.0] RD 10.253.128.1:19999
leaf-su00-r0 |                     10.253.128.1 (leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |                     ET:8 RT:59904:19999 Rmac:44:38:39:22:01:cd
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.24.0.0] RD 10.253.128.1:19999
leaf-su00-r0 |                     10.253.128.1 (leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |                     ET:8 RT:59904:19999 Rmac:44:38:39:22:01:cd
leaf-su00-r0 | Route Distinguisher: 10.253.128.2:19998
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.18.0.0] RD 10.253.128.2:19998
leaf-su00-r0 |                     10.253.128.2 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |                     RT:59905:19998 ET:8 Rmac:44:38:39:22:01:ce
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.18.0.0] RD 10.253.128.2:19998
leaf-su00-r0 |                     10.253.128.2 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |                     RT:59905:19998 ET:8 Rmac:44:38:39:22:01:ce
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.26.0.0] RD 10.253.128.2:19998
leaf-su00-r0 |                     10.253.128.2 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |                     RT:59905:19998 ET:8 Rmac:44:38:39:22:01:ce
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.26.0.0] RD 10.253.128.2:19998
leaf-su00-r0 |                     10.253.128.2 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |                     RT:59905:19998 ET:8 Rmac:44:38:39:22:01:ce
leaf-su00-r0 | Route Distinguisher: 10.253.128.2:19999
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.18.0.0] RD 10.253.128.2:19999
leaf-su00-r0 |                     10.253.128.2 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |                     RT:59905:19999 ET:8 Rmac:44:38:39:22:01:ce
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.18.0.0] RD 10.253.128.2:19999
leaf-su00-r0 |                     10.253.128.2 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |                     RT:59905:19999 ET:8 Rmac:44:38:39:22:01:ce
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.26.0.0] RD 10.253.128.2:19999
leaf-su00-r0 |                     10.253.128.2 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |                     RT:59905:19999 ET:8 Rmac:44:38:39:22:01:ce
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.26.0.0] RD 10.253.128.2:19999
leaf-su00-r0 |                     10.253.128.2 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |                     RT:59905:19999 ET:8 Rmac:44:38:39:22:01:ce
leaf-su00-r0 | Route Distinguisher: 10.253.128.3:19998
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.20.0.0] RD 10.253.128.3:19998
leaf-su00-r0 |                     10.253.128.3 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |                     RT:59906:19998 ET:8 Rmac:44:38:39:22:01:cf
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.20.0.0] RD 10.253.128.3:19998
leaf-su00-r0 |                     10.253.128.3 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |                     RT:59906:19998 ET:8 Rmac:44:38:39:22:01:cf
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.28.0.0] RD 10.253.128.3:19998
leaf-su00-r0 |                     10.253.128.3 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |                     RT:59906:19998 ET:8 Rmac:44:38:39:22:01:cf
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.28.0.0] RD 10.253.128.3:19998
leaf-su00-r0 |                     10.253.128.3 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |                     RT:59906:19998 ET:8 Rmac:44:38:39:22:01:cf
leaf-su00-r0 | Route Distinguisher: 10.253.128.3:19999
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.20.0.0] RD 10.253.128.3:19999
leaf-su00-r0 |                     10.253.128.3 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |                     RT:59906:19999 ET:8 Rmac:44:38:39:22:01:cf
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.20.0.0] RD 10.253.128.3:19999
leaf-su00-r0 |                     10.253.128.3 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |                     RT:59906:19999 ET:8 Rmac:44:38:39:22:01:cf
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.28.0.0] RD 10.253.128.3:19999
leaf-su00-r0 |                     10.253.128.3 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |                     RT:59906:19999 ET:8 Rmac:44:38:39:22:01:cf
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.28.0.0] RD 10.253.128.3:19999
leaf-su00-r0 |                     10.253.128.3 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |                     RT:59906:19999 ET:8 Rmac:44:38:39:22:01:cf
leaf-su00-r0 | Route Distinguisher: 10.253.128.4:19998
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.22.0.0] RD 10.253.128.4:19998
leaf-su00-r0 |                     10.253.128.4 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |                     RT:59907:19998 ET:8 Rmac:44:38:39:22:01:d0
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.22.0.0] RD 10.253.128.4:19998
leaf-su00-r0 |                     10.253.128.4 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |                     RT:59907:19998 ET:8 Rmac:44:38:39:22:01:d0
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.30.0.0] RD 10.253.128.4:19998
leaf-su00-r0 |                     10.253.128.4 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |                     RT:59907:19998 ET:8 Rmac:44:38:39:22:01:d0
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.30.0.0] RD 10.253.128.4:19998
leaf-su00-r0 |                     10.253.128.4 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |                     RT:59907:19998 ET:8 Rmac:44:38:39:22:01:d0
leaf-su00-r0 | Route Distinguisher: 10.253.128.4:19999
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.22.0.0] RD 10.253.128.4:19999
leaf-su00-r0 |                     10.253.128.4 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |                     RT:59907:19999 ET:8 Rmac:44:38:39:22:01:d0
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.22.0.0] RD 10.253.128.4:19999
leaf-su00-r0 |                     10.253.128.4 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |                     RT:59907:19999 ET:8 Rmac:44:38:39:22:01:d0
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.30.0.0] RD 10.253.128.4:19999
leaf-su00-r0 |                     10.253.128.4 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |                     RT:59907:19999 ET:8 Rmac:44:38:39:22:01:d0
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.30.0.0] RD 10.253.128.4:19999
leaf-su00-r0 |                     10.253.128.4 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |                     RT:59907:19999 ET:8 Rmac:44:38:39:22:01:d0
leaf-su00-r0 |
leaf-su00-r0 | Displayed 16 prefixes (28 paths) (of requested type)
################################################################################
```

This is the control-plane state behind the host-to-host reachability that Step 14 verifies from the hosts.

### Step 13. Configure the HGX Hosts

**Goal:** Apply the per-host rail netplan configuration so each HGX host's eight rail NICs come up with their `/31` addresses.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`; the HGX hosts at `192.168.200.17` through `.20`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation and on the HGX hosts.

**Expected wait time:** Under a minute per host.

Each host has eight rail NICs (`eth_rail0` through `eth_rail7`) at MTU 9216. Each rail takes a `/31`, with the host on the even address and the leaf gateway on the odd one. For host node `h` on rail `r`, the host address is `172.(16 + 2*r).0.(2*h)` and the gateway is one higher:

| __rail__ | __NIC__ | __h00__ | __h08__ | __h16__ | __h24__ |
| --- | --- | --- | --- | --- | --- |
| 0 | `eth_rail0` | 172.16.0.0 | 172.16.0.16 | 172.16.0.32 | 172.16.0.48 |
| 1 | `eth_rail1` | 172.18.0.0 | 172.18.0.16 | 172.18.0.32 | 172.18.0.48 |
| 2 | `eth_rail2` | 172.20.0.0 | 172.20.0.16 | 172.20.0.32 | 172.20.0.48 |
| 3 | `eth_rail3` | 172.22.0.0 | 172.22.0.16 | 172.22.0.32 | 172.22.0.48 |
| 4 | `eth_rail4` | 172.24.0.0 | 172.24.0.16 | 172.24.0.32 | 172.24.0.48 |
| 5 | `eth_rail5` | 172.26.0.0 | 172.26.0.16 | 172.26.0.32 | 172.26.0.48 |
| 6 | `eth_rail6` | 172.28.0.0 | 172.28.0.16 | 172.28.0.32 | 172.28.0.48 |
| 7 | `eth_rail7` | 172.30.0.0 | 172.30.0.16 | 172.30.0.32 | 172.30.0.48 |

The complete per-host netplan files are staged on the jumpstation under `netplan/1su/`, one per host (`hgx-su00-h00.yaml` through `hgx-su00-h24.yaml`). Each holds all eight rails and installs to `/etc/netplan/60-spectrum-x.yaml` on its host.

First, capture each host's rail state *before* applying netplan, so there is a baseline to compare against. The `eth_rail` NICs carry no `172.x` addresses yet and the routing table holds no rail routes:

```bash
~/spcx-air/spcx-run -s ip -br a
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -s ip -br a
Running: ip -br a

################################################################################
hgx-su00-h00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h00 | eth0             UP             192.168.200.17/24 metric 100 fe80::4638:39ff:fe11:111/64
hgx-su00-h00 | eth_rail0        DOWN
hgx-su00-h00 | eth_rail1        DOWN
hgx-su00-h00 | eth_rail2        DOWN
hgx-su00-h00 | eth_rail3        DOWN
hgx-su00-h00 | eth_rail4        DOWN
hgx-su00-h00 | eth_rail5        DOWN
hgx-su00-h00 | eth_rail6        DOWN
hgx-su00-h00 | eth_rail7        DOWN
################################################################################
hgx-su00-h08 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h08 | eth0             UP             192.168.200.18/24 metric 100 fe80::4638:39ff:fe11:112/64
hgx-su00-h08 | eth_rail0        DOWN
hgx-su00-h08 | eth_rail1        DOWN
hgx-su00-h08 | eth_rail2        DOWN
hgx-su00-h08 | eth_rail3        DOWN
hgx-su00-h08 | eth_rail4        DOWN
hgx-su00-h08 | eth_rail5        DOWN
hgx-su00-h08 | eth_rail6        DOWN
hgx-su00-h08 | eth_rail7        DOWN
################################################################################
hgx-su00-h16 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h16 | eth0             UP             192.168.200.19/24 metric 100 fe80::4638:39ff:fe11:113/64
hgx-su00-h16 | eth_rail0        DOWN
hgx-su00-h16 | eth_rail1        DOWN
hgx-su00-h16 | eth_rail2        DOWN
hgx-su00-h16 | eth_rail3        DOWN
hgx-su00-h16 | eth_rail4        DOWN
hgx-su00-h16 | eth_rail5        DOWN
hgx-su00-h16 | eth_rail6        DOWN
hgx-su00-h16 | eth_rail7        DOWN
################################################################################
hgx-su00-h24 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h24 | eth0             UP             192.168.200.20/24 metric 100 fe80::4638:39ff:fe11:114/64
hgx-su00-h24 | eth_rail0        DOWN
hgx-su00-h24 | eth_rail1        DOWN
hgx-su00-h24 | eth_rail2        DOWN
hgx-su00-h24 | eth_rail3        DOWN
hgx-su00-h24 | eth_rail4        DOWN
hgx-su00-h24 | eth_rail5        DOWN
hgx-su00-h24 | eth_rail6        DOWN
hgx-su00-h24 | eth_rail7        DOWN
################################################################################
```

Now push each host the file named for it and apply it. This loop reads each host's name, copies its file, installs it with mode 0600, and runs `netplan apply`:

```bash
 ~/spcx-air/spcx-run --apply-netplan ~/spcx-air/fabric/1su/hosts/
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$  ~/spcx-air/spcx-run --apply-netplan ~/spcx-air/fabric/1su/hosts/
Applying per-host netplan from: /home/ubuntu/spcx-air/fabric/1su/hosts
Target: hgx_hosts

################################################################################
hgx-su00-h00 | (no output)
################################################################################
hgx-su00-h08 | (no output)
################################################################################
hgx-su00-h16 | (no output)
################################################################################
hgx-su00-h24 | (no output)
################################################################################
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
hgx-su00-h00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h00 | eth0             UP             192.168.200.17/24 metric 100 fe80::4638:39ff:fe11:111/64
hgx-su00-h00 | eth_rail0        UP             172.16.0.0/31 fe80::4ab0:2dff:fe58:140a/64
hgx-su00-h00 | eth_rail1        UP             172.18.0.0/31 fe80::4ab0:2dff:fe21:355e/64
hgx-su00-h00 | eth_rail2        UP             172.20.0.0/31 fe80::4ab0:2dff:fe72:b4b8/64
hgx-su00-h00 | eth_rail3        UP             172.22.0.0/31 fe80::4ab0:2dff:fe86:769e/64
hgx-su00-h00 | eth_rail4        UP             172.24.0.0/31 fe80::4ab0:2dff:fedf:dfe7/64
hgx-su00-h00 | eth_rail5        UP             172.26.0.0/31 fe80::4ab0:2dff:fef5:1bad/64
hgx-su00-h00 | eth_rail6        UP             172.28.0.0/31 fe80::4ab0:2dff:fedd:4980/64
hgx-su00-h00 | eth_rail7        UP             172.30.0.0/31 fe80::4ab0:2dff:fea2:cff3/64
################################################################################
hgx-su00-h08 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h08 | eth0             UP             192.168.200.18/24 metric 100 fe80::4638:39ff:fe11:112/64
hgx-su00-h08 | eth_rail0        UP             172.16.0.16/31 fe80::4ab0:2dff:fe02:2041/64
hgx-su00-h08 | eth_rail1        UP             172.18.0.16/31 fe80::4ab0:2dff:fec9:e6c5/64
hgx-su00-h08 | eth_rail2        UP             172.20.0.16/31 fe80::4ab0:2dff:feea:4e4f/64
hgx-su00-h08 | eth_rail3        UP             172.22.0.16/31 fe80::4ab0:2dff:fe33:877a/64
hgx-su00-h08 | eth_rail4        UP             172.24.0.16/31 fe80::4ab0:2dff:fe01:bff/64
hgx-su00-h08 | eth_rail5        UP             172.26.0.16/31 fe80::4ab0:2dff:feab:80e7/64
hgx-su00-h08 | eth_rail6        UP             172.28.0.16/31 fe80::4ab0:2dff:fe10:a63a/64
hgx-su00-h08 | eth_rail7        UP             172.30.0.16/31 fe80::4ab0:2dff:fec0:62cb/64
################################################################################
hgx-su00-h16 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h16 | eth0             UP             192.168.200.19/24 metric 100 fe80::4638:39ff:fe11:113/64
hgx-su00-h16 | eth_rail0        UP             172.16.0.32/31 fe80::4ab0:2dff:fe8c:37a3/64
hgx-su00-h16 | eth_rail1        UP             172.18.0.32/31 fe80::4ab0:2dff:fe1f:1ceb/64
hgx-su00-h16 | eth_rail2        UP             172.20.0.32/31 fe80::4ab0:2dff:fe97:904c/64
hgx-su00-h16 | eth_rail3        UP             172.22.0.32/31 fe80::4ab0:2dff:fe1e:3c2/64
hgx-su00-h16 | eth_rail4        UP             172.24.0.32/31 fe80::4ab0:2dff:fedb:5e58/64
hgx-su00-h16 | eth_rail5        UP             172.26.0.32/31 fe80::4ab0:2dff:fe5a:2595/64
hgx-su00-h16 | eth_rail6        UP             172.28.0.32/31 fe80::4ab0:2dff:fed9:9a07/64
hgx-su00-h16 | eth_rail7        UP             172.30.0.32/31 fe80::4ab0:2dff:fe62:c79a/64
################################################################################
hgx-su00-h24 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h24 | eth0             UP             192.168.200.20/24 metric 100 fe80::4638:39ff:fe11:114/64
hgx-su00-h24 | eth_rail0        UP             172.16.0.48/31 fe80::4ab0:2dff:fe53:b590/64
hgx-su00-h24 | eth_rail1        UP             172.18.0.48/31 fe80::4ab0:2dff:fe31:da81/64
hgx-su00-h24 | eth_rail2        UP             172.20.0.48/31 fe80::4ab0:2dff:fe4c:555d/64
hgx-su00-h24 | eth_rail3        UP             172.22.0.48/31 fe80::4ab0:2dff:fea9:688f/64
hgx-su00-h24 | eth_rail4        UP             172.24.0.48/31 fe80::4ab0:2dff:fe0c:5c1b/64
hgx-su00-h24 | eth_rail5        UP             172.26.0.48/31 fe80::4ab0:2dff:fea4:b192/64
hgx-su00-h24 | eth_rail6        UP             172.28.0.48/31 fe80::4ab0:2dff:feb3:a50a/64
hgx-su00-h24 | eth_rail7        UP             172.30.0.48/31 fe80::4ab0:2dff:fe0a:53e9/64
################################################################################
```

### Step 14. Verify Connectivity

**Goal:** Confirm the tenant deployment finished, then run a full rail mesh ping test across all four HGX hosts.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`; the HGX hosts at `192.168.200.17` through `.20`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation and on the HGX hosts.

**Expected wait time:** Up to about 30 seconds per host for BGP/EVPN to finish converging.

The rail `/31`s live only on the hosts, so the ping sweep has to run on each host, not on the jumpstation. This block drives all four hosts from the jumpstation: it SSHes in with the lab credentials and runs the full rail mesh on each one, covering all eight rails across all four hosts (32 targets per host):

```bash
RAILS0="172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48"
~/spcx-air/spcx-run -s -c "fping -c2 -t500 $RAILS0 2>&1"
```

```bash
~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ RAILS0="172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48"
~/spcx-air/spcx-run -s -c "fping -c2 -t500 $RAILS0 2>&1"
========================================
Running: fping -c2 -t500 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 2>&1
========================================
################################################################################
hgx-su00-h00 | 172.16.0.0  : [0], 64 bytes, 0.039 ms (0.039 avg, 0% loss)
hgx-su00-h00 | 172.16.0.32 : [0], 64 bytes, 0.445 ms (0.445 avg, 0% loss)
hgx-su00-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.0  : [1], 64 bytes, 0.045 ms (0.042 avg, 0% loss)
hgx-su00-h00 | 172.16.0.32 : [1], 64 bytes, 0.577 ms (0.511 avg, 0% loss)
hgx-su00-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 |
hgx-su00-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.039/0.042/0.045
hgx-su00-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.445/0.511/0.577
hgx-su00-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su00-h08 | 172.16.0.16 : [0], 64 bytes, 0.025 ms (0.025 avg, 0% loss)
hgx-su00-h08 | 172.16.0.48 : [0], 64 bytes, 0.673 ms (0.673 avg, 0% loss)
hgx-su00-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.0.16 : [1], 64 bytes, 0.016 ms (0.020 avg, 0% loss)
hgx-su00-h08 | 172.16.0.48 : [1], 64 bytes, 0.567 ms (0.620 avg, 0% loss)
hgx-su00-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 |
hgx-su00-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.016/0.020/0.025
hgx-su00-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.567/0.620/0.673
################################################################################
hgx-su00-h16 | 172.16.0.0  : [0], 64 bytes, 0.865 ms (0.865 avg, 0% loss)
hgx-su00-h16 | 172.16.0.32 : [0], 64 bytes, 0.048 ms (0.048 avg, 0% loss)
hgx-su00-h16 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.0.0  : [1], 64 bytes, 0.601 ms (0.733 avg, 0% loss)
hgx-su00-h16 | 172.16.0.32 : [1], 64 bytes, 0.014 ms (0.031 avg, 0% loss)
hgx-su00-h16 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 |
hgx-su00-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.601/0.733/0.865
hgx-su00-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.014/0.031/0.048
hgx-su00-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su00-h24 | 172.16.0.16 : [0], 64 bytes, 0.760 ms (0.760 avg, 0% loss)
hgx-su00-h24 | 172.16.0.48 : [0], 64 bytes, 0.062 ms (0.062 avg, 0% loss)
hgx-su00-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.0.16 : [1], 64 bytes, 0.755 ms (0.757 avg, 0% loss)
hgx-su00-h24 | 172.16.0.48 : [1], 64 bytes, 0.016 ms (0.039 avg, 0% loss)
hgx-su00-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 |
hgx-su00-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.755/0.757/0.760
hgx-su00-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.016/0.039/0.062
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
========================================
Running: for ip in 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48; do ping -c2 -W2 $ip >/dev/null 2>&1 && echo "PASS $ip" || echo "FAIL $ip"; done
========================================
################################################################################
hgx-su00-h00 | PASS 172.16.0.0
hgx-su00-h00 | FAIL 172.16.0.16
hgx-su00-h00 | PASS 172.16.0.32
hgx-su00-h00 | FAIL 172.16.0.48
################################################################################
hgx-su00-h08 | FAIL 172.16.0.0
hgx-su00-h08 | PASS 172.16.0.16
hgx-su00-h08 | FAIL 172.16.0.32
hgx-su00-h08 | PASS 172.16.0.48
################################################################################
hgx-su00-h16 | PASS 172.16.0.0
hgx-su00-h16 | FAIL 172.16.0.16
hgx-su00-h16 | PASS 172.16.0.32
hgx-su00-h16 | FAIL 172.16.0.48
################################################################################
hgx-su00-h24 | FAIL 172.16.0.0
hgx-su00-h24 | PASS 172.16.0.16
hgx-su00-h24 | FAIL 172.16.0.32
hgx-su00-h24 | PASS 172.16.0.48
################################################################################
```

Expected result:

- HGX nodes from `tenant1` infrastructure (`hgx-su00-h00` and `hgx-su00-h16`) have connectivity with each other, as they share the same `tenant1` infrastructure, but cannot communicate with `hgx-su00-h08` and `hgx-su00-h24`, which are in a different infrastructure (`tenant2`). The same applies for `hgx-su00-h08` and `hgx-su00-h24`, which have connectivity with each other, but cannot communicate with `hgx-su00-h00` and `hgx-su00-h16`.


Modify the terraform manifest from `~/nvidia/terraform/tenant1` in order to remove `hgx-su00-h16` from `infra-tenant1` infrastructure and apply. Use the following commands in order to modify `infra-tenant1.tf` directly:

```bash
cd ~/nvidia/terraform/tenant1
sed -i 's/default = \["hgx-su00-h00", "hgx-su00-h16"\]/default = ["hgx-su00-h00"]/' infra-tenant1.tf
terraform apply -auto-approve
```


```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  ~ update in-place
  - destroy
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # metalcloud_endpoint_instance_group.groups["hgx-su00-h16"] will be destroyed
  # (because key ["hgx-su00-h16"] is not in for_each map)
  - resource "metalcloud_endpoint_instance_group" "groups" {
      - endpoint_ids               = [
          - "3",
        ] -> null
      - endpoint_instance_group_id = "8" -> null
      - infrastructure_id          = "3" -> null
      - label                      = "hgx-su00-h16" -> null
      - network_connections        = [
          - {
              - access_mode        = "l2" -> null
              - interface_count    = 8 -> null
              - logical_network_id = "3" -> null
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
        id     = "d78f8911-b46e-019c-a6ad-ef9497433c5c"
      ~ input  = "hgx-su00-h00,hgx-su00-h16" -> "hgx-su00-h00"
      ~ output = "hgx-su00-h00,hgx-su00-h16" -> (known after apply)
    }

Plan: 1 to add, 1 to change, 2 to destroy.
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Destroying...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Destruction complete after 0s
terraform_data.endpoint_fingerprint: Modifying... [id=d78f8911-b46e-019c-a6ad-ef9497433c5c]
terraform_data.endpoint_fingerprint: Modifications complete after 0s [id=d78f8911-b46e-019c-a6ad-ef9497433c5c]
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Destroying...
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Destruction complete after 0s
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
┌────┬──────────────────┬──────────────────┬────────┬───────┬──────┬─────────────────────┬─────────────────────┬───────────────┬───────────┐
│ ID │ LABEL            │ CONFIG LABEL     │ STATUS │ OWNER │ SITE │ CREATED             │ UPDATED             │ DEPLOY STATUS │ DEPLOY ID │
├────┼──────────────────┼──────────────────┼────────┼───────┼──────┼─────────────────────┼─────────────────────┼───────────────┼───────────┤
│  3 │ infra-tenant1    │ infra-tenant1    │ active │     1 │    1 │ 12 Aug 26 20:43 UTC │ 12 Aug 26 21:02 UTC │ finished      │           │
│  4 │ infra-tenant2    │ infra-tenant2    │ active │     1 │    1 │ 12 Aug 26 20:44 UTC │ 12 Aug 26 20:48 UTC │ finished      │           │
└────┴──────────────────┴──────────────────┴────────┴───────┴──────┴─────────────────────┴─────────────────────┴───────────────┴───────────┘
```

(Optional) In the web UI, navigate to **Admin dashboard > Infrastructures** in order to see the current progress state of the deployment:

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/infrastructure_state.webp)

Modify the terraform manifest from `~/nvidia/terraform/tenant2` in order to add `hgx-su00-h16` to `infra-tenant2` infrastructure and apply. Use the following commands in order to modify `infra-tenant2.tf` directly:

```bash
cd ~/nvidia/terraform/tenant2
sed -i 's/default = \["hgx-su00-h08", "hgx-su00-h24"\]/default = ["hgx-su00-h08", "hgx-su00-h24", "hgx-su00-h16"]/' infra-tenant2.tf
terraform apply -auto-approve
```
The expected output should be the following:

```bash
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create
  ~ update in-place
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # metalcloud_endpoint_instance_group.groups["hgx-su00-h16"] will be created
  + resource "metalcloud_endpoint_instance_group" "groups" {
      + endpoint_ids               = [
          + "3",
        ]
      + endpoint_instance_group_id = (known after apply)
      + infrastructure_id          = "4"
      + label                      = "hgx-su00-h16"
      + network_connections        = [
          + {
              + access_mode        = "l2"
              + interface_count    = 8
              + logical_network_id = "4"
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
        id     = "a9dddcde-7bb7-d688-6cbc-9ed61393d339"
      ~ input  = "hgx-su00-h08,hgx-su00-h24" -> "hgx-su00-h08,hgx-su00-h16,hgx-su00-h24"
      ~ output = "hgx-su00-h08,hgx-su00-h24" -> (known after apply)
    }

Plan: 2 to add, 1 to change, 1 to destroy.
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Destroying...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Destruction complete after 0s
terraform_data.endpoint_fingerprint: Modifying... [id=a9dddcde-7bb7-d688-6cbc-9ed61393d339]
terraform_data.endpoint_fingerprint: Modifications complete after 0s [id=a9dddcde-7bb7-d688-6cbc-9ed61393d339]
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Creation complete after 0s
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Creating...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Creation complete after 0s

Apply complete! Resources: 2 added, 1 changed, 1 destroyed.
```

Before continuing with the checks, confirm that the Terraform deployments have finished. The following command refreshes every 10 seconds; leave it running until the deploy status shows finished:

```bash
watch -n 10 "metalcloud-cli infrastructure list"
```
Use control+C to stop the watch at any time.

The expected output should be the following:

```bash
┌────┬──────────────────┬──────────────────┬────────┬───────┬──────┬─────────────────────┬─────────────────────┬───────────────┬───────────┐
│ ID │ LABEL            │ CONFIG LABEL     │ STATUS │ OWNER │ SITE │ CREATED             │ UPDATED             │ DEPLOY STATUS │ DEPLOY ID │
├────┼──────────────────┼──────────────────┼────────┼───────┼──────┼─────────────────────┼─────────────────────┼───────────────┼───────────┤
│  3 │ infra-tenant1    │ infra-tenant1    │ active │     1 │    1 │ 12 Aug 26 20:43 UTC │ 12 Aug 26 21:02 UTC │ finished      │           │
│  4 │ infra-tenant2    │ infra-tenant2    │ active │     1 │    1 │ 12 Aug 26 20:44 UTC │ 12 Aug 26 21:06 UTC │ finished      │           │
└────┴──────────────────┴──────────────────┴────────┴───────┴──────┴─────────────────────┴─────────────────────┴───────────────┴───────────┘
```

After `hgx-su00-h16` has been moved from `tenant1` infrastructure to `tenant2` infrastructure, the graphical representation of two infrastructures will look like the following:

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/tenant1-without-h16.1su.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/430981a5-a04b-4355-9a37-e70f3f98844b/tenant2-with-h16.1su.webp)

Verify connectivity after `hgx-su00-h16` was moved to `tenant2` infrastructure:

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
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br addr show vrf tenant1 | grep -E \"swp|172\.\""
========================================
Running: ip -br addr show vrf tenant1 | grep -E "swp|172\."
========================================
################################################################################
leaf-su00-r0 | swp1s0           UP             172.16.0.1/31 fe80::4ab0:2dff:fea2:de1c/64
leaf-su00-r0 | swp1s1           UP             172.24.0.1/31 fe80::4ab0:2dff:fe08:12c6/64
################################################################################
leaf-su00-r1 | swp1s0           UP             172.18.0.1/31 fe80::4ab0:2dff:fe30:371c/64
leaf-su00-r1 | swp1s1           UP             172.26.0.1/31 fe80::4ab0:2dff:fe9a:7591/64
################################################################################
leaf-su00-r2 | swp1s0           UP             172.20.0.1/31 fe80::4ab0:2dff:fe93:1a35/64
leaf-su00-r2 | swp1s1           UP             172.28.0.1/31 fe80::4ab0:2dff:fe08:e8ff/64
################################################################################
leaf-su00-r3 | swp1s0           UP             172.22.0.1/31 fe80::4ab0:2dff:fe0e:409e/64
leaf-su00-r3 | swp1s1           UP             172.30.0.1/31 fe80::4ab0:2dff:fe83:3fd5/64
################################################################################
spine-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-s01 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant1 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant1 ipv4 unicast"
========================================
################################################################################
leaf-su00-r0 | BGP table version is 46, local router ID is 10.253.128.1, vrf id 136
leaf-su00-r0 | Default local pref 100, local AS 4200000000
leaf-su00-r0 | Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath, + multipath nhg,
leaf-su00-r0 |                i internal, r RIB-failure, S Stale, R Removed
leaf-su00-r0 | Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
leaf-su00-r0 | Origin codes:  i - IGP, e - EGP, ? - incomplete
leaf-su00-r0 | RPKI validation codes: V valid, I invalid, N Not found
leaf-su00-r0 |
leaf-su00-r0 |     Network          Next Hop            Metric LocPrf Weight Path
leaf-su00-r0 |  *> 172.16.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.16.0.0/31    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.0/31    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |
leaf-su00-r0 | Displayed 10 routes and 16 total paths
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant1\""
========================================
Running: sudo vtysh -c "show ip route vrf tenant1"
========================================
################################################################################
leaf-su00-r0 | Codes: K - kernel route, C - connected, L - local, S - static,
leaf-su00-r0 |        R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
leaf-su00-r0 |        T - Table, A - Babel, D - SHARP, F - PBR, f - OpenFabric,
leaf-su00-r0 |        t - Table-Direct, Z - FRR,
leaf-su00-r0 |        > - selected route, * - FIB route, q - queued, r - rejected, b - backup
leaf-su00-r0 |        t - trapped, o - offload failure
leaf-su00-r0 |
leaf-su00-r0 | VRF tenant1:
leaf-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:23:06
leaf-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:22:36
leaf-su00-r0 | C>* 172.16.0.0/31 is directly connected, swp1s0, 00:23:06
leaf-su00-r0 | L>* 172.16.0.1/32 is directly connected, swp1s0, 00:23:06
leaf-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:21:57
leaf-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:21:19
leaf-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:20:39
leaf-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:22:36
leaf-su00-r0 | C>* 172.24.0.0/31 is directly connected, swp1s1, 00:23:06
leaf-su00-r0 | L>* 172.24.0.1/32 is directly connected, swp1s1, 00:23:06
leaf-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:21:57
leaf-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:21:19
leaf-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:20:39
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
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "ip -br addr show vrf tenant2 | grep -E \"swp|172\.\""
========================================
Running: ip -br addr show vrf tenant2 | grep -E "swp|172\."
========================================
################################################################################
leaf-su00-r0 | swp9s0           UP             172.16.0.17/31 fe80::4ab0:2dff:feef:9af8/64
leaf-su00-r0 | swp9s1           UP             172.24.0.17/31 fe80::4ab0:2dff:fe00:de8f/64
leaf-su00-r0 | swp17s0          UP             172.16.0.33/31 fe80::4ab0:2dff:fef5:49c4/64
leaf-su00-r0 | swp17s1          UP             172.24.0.33/31 fe80::4ab0:2dff:fe9e:e970/64
leaf-su00-r0 | swp25s0          UP             172.16.0.49/31 fe80::4ab0:2dff:fe95:6e03/64
leaf-su00-r0 | swp25s1          UP             172.24.0.49/31 fe80::4ab0:2dff:fe9b:ee7/64
################################################################################
leaf-su00-r1 | swp9s0           UP             172.18.0.17/31 fe80::4ab0:2dff:fe21:cec3/64
leaf-su00-r1 | swp9s1           UP             172.26.0.17/31 fe80::4ab0:2dff:fe32:5c09/64
leaf-su00-r1 | swp17s0          UP             172.18.0.33/31 fe80::4ab0:2dff:fecc:6255/64
leaf-su00-r1 | swp17s1          UP             172.26.0.33/31 fe80::4ab0:2dff:fe61:4efb/64
leaf-su00-r1 | swp25s0          UP             172.18.0.49/31 fe80::4ab0:2dff:fe11:c578/64
leaf-su00-r1 | swp25s1          UP             172.26.0.49/31 fe80::4ab0:2dff:fe3f:97ea/64
################################################################################
leaf-su00-r2 | swp9s0           UP             172.20.0.17/31 fe80::4ab0:2dff:fe34:eb83/64
leaf-su00-r2 | swp9s1           UP             172.28.0.17/31 fe80::4ab0:2dff:fe04:592d/64
leaf-su00-r2 | swp17s0          UP             172.20.0.33/31 fe80::4ab0:2dff:fe54:17de/64
leaf-su00-r2 | swp17s1          UP             172.28.0.33/31 fe80::4ab0:2dff:fe09:a1b/64
leaf-su00-r2 | swp25s0          UP             172.20.0.49/31 fe80::4ab0:2dff:feb2:b0c5/64
leaf-su00-r2 | swp25s1          UP             172.28.0.49/31 fe80::4ab0:2dff:fe99:16e3/64
################################################################################
leaf-su00-r3 | swp9s0           UP             172.22.0.17/31 fe80::4ab0:2dff:fe81:43e2/64
leaf-su00-r3 | swp9s1           UP             172.30.0.17/31 fe80::4ab0:2dff:fe4a:bb39/64
leaf-su00-r3 | swp17s0          UP             172.22.0.33/31 fe80::4ab0:2dff:fe8f:aad2/64
leaf-su00-r3 | swp17s1          UP             172.30.0.33/31 fe80::4ab0:2dff:fe0c:f578/64
leaf-su00-r3 | swp25s0          UP             172.22.0.49/31 fe80::4ab0:2dff:fea4:1e4c/64
leaf-su00-r3 | swp25s1          UP             172.30.0.49/31 fe80::4ab0:2dff:fe98:acee/64
################################################################################
spine-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-s01 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant2 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant2 ipv4 unicast"
========================================
################################################################################
leaf-su00-r0 | BGP table version is 46, local router ID is 10.253.128.1, vrf id 132
leaf-su00-r0 | Default local pref 100, local AS 4200000000
leaf-su00-r0 | Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath, + multipath nhg,
leaf-su00-r0 |                i internal, r RIB-failure, S Stale, R Removed
leaf-su00-r0 | Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
leaf-su00-r0 | Origin codes:  i - IGP, e - EGP, ? - incomplete
leaf-su00-r0 | RPKI validation codes: V valid, I invalid, N Not found
leaf-su00-r0 |
leaf-su00-r0 |     Network          Next Hop            Metric LocPrf Weight Path
leaf-su00-r0 |  *> 172.16.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.16.0.16/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.16.0.32/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.16.0.48/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.16/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.32/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.48/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |
leaf-su00-r0 | Displayed 14 routes and 20 total paths
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant2\""
========================================
Running: sudo vtysh -c "show ip route vrf tenant2"
========================================
################################################################################
leaf-su00-r0 | Codes: K - kernel route, C - connected, L - local, S - static,
leaf-su00-r0 |        R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
leaf-su00-r0 |        T - Table, A - Babel, D - SHARP, F - PBR, f - OpenFabric,
leaf-su00-r0 |        t - Table-Direct, Z - FRR,
leaf-su00-r0 |        > - selected route, * - FIB route, q - queued, r - rejected, b - backup
leaf-su00-r0 |        t - trapped, o - offload failure
leaf-su00-r0 |
leaf-su00-r0 | VRF tenant2:
leaf-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:24:08
leaf-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:23:38
leaf-su00-r0 | C>* 172.16.0.16/31 is directly connected, swp9s0, 00:24:08
leaf-su00-r0 | L>* 172.16.0.17/32 is directly connected, swp9s0, 00:24:08
leaf-su00-r0 | C>* 172.16.0.32/31 is directly connected, swp17s0, 00:03:42
leaf-su00-r0 | L>* 172.16.0.33/32 is directly connected, swp17s0, 00:03:42
leaf-su00-r0 | C>* 172.16.0.48/31 is directly connected, swp25s0, 00:24:08
leaf-su00-r0 | L>* 172.16.0.49/32 is directly connected, swp25s0, 00:24:08
leaf-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:22:58
leaf-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:22:20
leaf-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:21:41
leaf-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:23:38
leaf-su00-r0 | C>* 172.24.0.16/31 is directly connected, swp9s1, 00:24:08
leaf-su00-r0 | L>* 172.24.0.17/32 is directly connected, swp9s1, 00:24:08
leaf-su00-r0 | C>* 172.24.0.32/31 is directly connected, swp17s1, 00:03:42
leaf-su00-r0 | L>* 172.24.0.33/32 is directly connected, swp17s1, 00:03:42
leaf-su00-r0 | C>* 172.24.0.48/31 is directly connected, swp25s1, 00:24:08
leaf-su00-r0 | L>* 172.24.0.49/32 is directly connected, swp25s1, 00:24:08
leaf-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:22:58
leaf-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:22:20
leaf-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:21:41
################################################################################
```


```bash
~/spcx-air/spcx-run -s -c "fping -c2 -t500 $RAILS0 2>&1"
```

```bash
~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -s -c "fping -c2 -t500 $RAILS0 2>&1"
========================================
Running: fping -c2 -t500 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 2>&1
========================================
################################################################################
hgx-su00-h00 | 172.16.0.0  : [0], 64 bytes, 0.040 ms (0.040 avg, 0% loss)
hgx-su00-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.0  : [1], 64 bytes, 0.032 ms (0.036 avg, 0% loss)
hgx-su00-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 |
hgx-su00-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.032/0.036/0.040
hgx-su00-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su00-h08 | 172.16.0.16 : [0], 64 bytes, 0.022 ms (0.022 avg, 0% loss)
hgx-su00-h08 | 172.16.0.32 : [0], 64 bytes, 0.544 ms (0.544 avg, 0% loss)
hgx-su00-h08 | 172.16.0.48 : [0], 64 bytes, 0.497 ms (0.497 avg, 0% loss)
hgx-su00-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.0.16 : [1], 64 bytes, 0.021 ms (0.021 avg, 0% loss)
hgx-su00-h08 | 172.16.0.32 : [1], 64 bytes, 0.590 ms (0.567 avg, 0% loss)
hgx-su00-h08 | 172.16.0.48 : [1], 64 bytes, 0.490 ms (0.493 avg, 0% loss)
hgx-su00-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 |
hgx-su00-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.021/0.021/0.022
hgx-su00-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.544/0.567/0.590
hgx-su00-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.490/0.493/0.497
################################################################################
hgx-su00-h16 | 172.16.0.16 : [0], 64 bytes, 0.896 ms (0.896 avg, 0% loss)
hgx-su00-h16 | 172.16.0.32 : [0], 64 bytes, 0.024 ms (0.024 avg, 0% loss)
hgx-su00-h16 | 172.16.0.48 : [0], 64 bytes, 0.562 ms (0.562 avg, 0% loss)
hgx-su00-h16 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.0.16 : [1], 64 bytes, 0.647 ms (0.772 avg, 0% loss)
hgx-su00-h16 | 172.16.0.32 : [1], 64 bytes, 0.010 ms (0.017 avg, 0% loss)
hgx-su00-h16 | 172.16.0.48 : [1], 64 bytes, 0.648 ms (0.605 avg, 0% loss)
hgx-su00-h16 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 |
hgx-su00-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.647/0.772/0.896
hgx-su00-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.010/0.017/0.024
hgx-su00-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.562/0.605/0.648
################################################################################
hgx-su00-h24 | 172.16.0.16 : [0], 64 bytes, 0.749 ms (0.749 avg, 0% loss)
hgx-su00-h24 | 172.16.0.32 : [0], 64 bytes, 0.627 ms (0.627 avg, 0% loss)
hgx-su00-h24 | 172.16.0.48 : [0], 64 bytes, 0.036 ms (0.036 avg, 0% loss)
hgx-su00-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.0.16 : [1], 64 bytes, 0.671 ms (0.710 avg, 0% loss)
hgx-su00-h24 | 172.16.0.32 : [1], 64 bytes, 0.620 ms (0.623 avg, 0% loss)
hgx-su00-h24 | 172.16.0.48 : [1], 64 bytes, 0.015 ms (0.025 avg, 0% loss)
hgx-su00-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 |
hgx-su00-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.671/0.710/0.749
hgx-su00-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.620/0.623/0.627
hgx-su00-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.015/0.025/0.036
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
========================================
Running: for ip in 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48; do ping -c2 -W2 $ip >/dev/null 2>&1 && echo "PASS $ip" || echo "FAIL $ip"; done
========================================
################################################################################
hgx-su00-h00 | PASS 172.16.0.0
hgx-su00-h00 | FAIL 172.16.0.16
hgx-su00-h00 | FAIL 172.16.0.32
hgx-su00-h00 | FAIL 172.16.0.48
################################################################################
hgx-su00-h08 | FAIL 172.16.0.0
hgx-su00-h08 | PASS 172.16.0.16
hgx-su00-h08 | PASS 172.16.0.32
hgx-su00-h08 | PASS 172.16.0.48
################################################################################
hgx-su00-h16 | FAIL 172.16.0.0
hgx-su00-h16 | PASS 172.16.0.16
hgx-su00-h16 | PASS 172.16.0.32
hgx-su00-h16 | PASS 172.16.0.48
################################################################################
hgx-su00-h24 | FAIL 172.16.0.0
hgx-su00-h24 | PASS 172.16.0.16
hgx-su00-h24 | PASS 172.16.0.32
hgx-su00-h24 | PASS 172.16.0.48
################################################################################
```

Expected result:

- All HGX hosts from `tenant2` infrastructure (`hgx-su00-h08`, `hgx-su00-h16`, and `hgx-su00-h24`) have connectivity with each other, but cannot communicate with `hgx-su00-h00`, as that one is in `tenant1` infrastructure.

Validation:

- BGP and EVPN can still be converging in the first seconds after the deploy, so the sweep retries each target for up to about thirty seconds rather than reporting a false failure; anything still unreachable after that is listed as a red `FAIL <ip>` line under the host that could not reach it. Re-running the block is safe.

<!-- AIR:page -->

## Validation Summary

- The fabric configuration patched all six devices, created 256 links, and added 288 `/31` point-to-point addresses.
- All four HGX endpoints were created.
- Underlay BGP has every session established, none down.
- The `tenant1` (`hgx-su00-h00`) and `tenant2` (`hgx-su00-h08`, `hgx-su00-h16`, `hgx-su00-h24`) L3 networks are each deployed with their own L3VNI.
- The host mesh test confirms full connectivity within each tenant and no connectivity across tenants, demonstrating tenant isolation.

| __Check__ | __Expected__ |
| --------- | -------------- |
| `fabric configure-switches` | devices patched=6, links created=256, /31 addresses added=288 |
| Endpoints | 4 created |
| Underlay BGP | all sessions established, 0 down |
| Tenants | `tenant1` (`hgx-su00-h00`) and `tenant2` (`hgx-su00-h08`, `hgx-su00-h16`, `hgx-su00-h24`) L3 networks deployed, one L3VNI each |
| Host mesh | full connectivity within each tenant; no connectivity across tenants |

Include a final validation command if helpful:

```bash
metalcloud-cli infrastructure list
```

## Troubleshooting, Upgrade, or Reset

- **The CLI cannot reach the controller.** If a `metalcloud-cli` command fails with a connection or TLS error, or `site agents 1` returns no agent, the controller may still be starting, or the Site Controller may have lost its link to the Global Controller after a restart. Log in to the Global Controller as `root` (`ssh -l root 192.168.200.3`, password `MetalsoftR0cks@$@$`) and run `kw` to watch the Kubernetes pods until each reaches `Running`; if any stay stuck, run `k-restart-all -A` to force a restart. On the Site Controller (`ssh -l root 192.168.200.2`, same password), run `docker ps` to confirm the `nfs-server` and `ms-agent` containers are `Up`, `docker logs -f ms-agent` to read the agent log, and `dcrestart` to restart the containers if `ms-agent` is not registering with the Global Controller.
- **A deploy job fails.** Read the job with `metalcloud-cli job get <id>`, correct the cause, and re-run the deploy step. BGP does not come up until the Step 9 deploy succeeds on every switch.
- **`get-ports` is empty in Step 4.** The switch is unreachable. Verify the management address and password in `switches.1su.yaml`, then re-run discovery.
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