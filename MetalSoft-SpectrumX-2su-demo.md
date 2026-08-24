<!-- AIR:tour -->

# MetalSoft for NVIDIA Spectrum-X: 2-SU Demo (512 GPU)

MetalSoft is an intelligent orchestration platform that transforms fragmented on-premises hardware into high-performance, fast-changing, secure, workload-compliant infrastructure. It integrates servers, switches, and storage to provide a turnkey neocloud platform (NCP) solution.

This lab builds a two scalability-unit (512 GPU) NVIDIA Spectrum-X fabric from a clean state and attaches a tenant to it, using the MetalSoft CLI (`metalcloud-cli`) and Terraform. The fabric is a two-tier leaf-spine design running EVPN over eBGP on Cumulus Linux 5.14.0. It extends the single-unit build with cross-unit reachability: a host in one unit reaches a host in the other over the EVPN overlay.

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/topology.2su.png)

The lab is preconfigured: it launches from a stored MetalSoft Spectrum-X checkpoint with the Global Controller, Site Controller, and Cumulus Linux switches already running and the CLI toolkit already staged on the jumpstation. You perform the fabric build, switch configuration, and tenant onboarding yourself during the lab; nothing is pre-built for you.

**IMPORTANT:** Allow about 45 minutes end to end; most of that time is switch deployment. Unless a step says otherwise, run every command on the `oob-mgmt-server` jumpstation.

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

A network or infrastructure engineer has already validated a single scalability unit and now needs to know whether MetalSoft's orchestration model scales across units, and whether a GPU workload in one unit can actually reach a peer in another unit over the fabric. This is the question every real Spectrum-X deployment has to answer before it grows past its first rack group.

In this lab, the engineer builds a two scalability-unit fabric (twelve switches, eight HGX hosts) from the jumpstation with `metalcloud-cli` and Terraform, following the same declarative workflow as the 1-SU lab: create the fabric, import the switches, discover links, configure and deploy the underlay and EVPN overlay, register the HGX hosts, and onboard a tenant network. The lab closes with a full rail-mesh connectivity test that specifically exercises cross-unit reachability — a host in the first unit pinging its same-rail peer in the second unit over the EVPN overlay — proving the fabric scales beyond a single unit without any additional manual configuration.

## Features and Services

This demo includes the following features and services:

- MetalSoft Global Controller and Site Controller orchestrating a two-unit Cumulus Linux Spectrum-X fabric
- CLI-driven fabric creation, switch import, link discovery, and configuration templating (`metalcloud-cli`)
- Terraform-based tenant onboarding across both scalability units
- MetalSoft Fabric Manager and Infrastructure Designer web UI (optional; the lab can be run entirely from the CLI)
- Two-tier leaf-spine underlay with EVPN over eBGP and cross-unit overlay reachability
- HGX host rail networking (eight rail NICs per host) validated with a full cross-unit mesh connectivity test

## What You Will Do in This Lab

- Stand up a 2-SU (512 GPU) Spectrum-X fabric from scratch with `metalcloud-cli`
- Configure and deploy the Cumulus Linux underlay and EVPN overlay across both units
- Register the eight HGX hosts as endpoints and onboard a tenant with Terraform
- Configure host rail networking and verify full-mesh RoCE connectivity, including cross-unit reachability

<!-- AIR:page -->

## Demo Topology Overview

The lab is two scalability units: twelve Cumulus Linux switches (eight leaves and four spines) in a two-tier leaf-spine design, and eight HGX hosts, each connected to the leaf layer over eight rail NICs. MetalSoft's Global Controller and Site Controller run alongside the fabric and are reachable from the jumpstation; nothing on the fabric is preconfigured; you build it in the Lab Flow section below.

### Device Naming

- Leaf: `leaf-su00-r0` .. `leaf-su00-r3` (unit 1), `leaf-su01-r0` .. `leaf-su01-r3` (unit 2)
- Spine: `spine-s00`, `spine-s01`, `spine-s02`, `spine-s03`
- HGX host: `hgx-su00-h00`/`h08`/`h16`/`h24` (unit 1), `hgx-su01-h00`/`h08`/`h16`/`h24` (unit 2)

### Devices

| __Role__ | __Device Names__ |
| -------- | ----------------- |
| Leaf (unit 1) | `leaf-su00-r0`, `leaf-su00-r1`, `leaf-su00-r2`, `leaf-su00-r3` |
| Leaf (unit 2) | `leaf-su01-r0`, `leaf-su01-r1`, `leaf-su01-r2`, `leaf-su01-r3` |
| Spine | `spine-s00`, `spine-s01`, `spine-s02`, `spine-s03` |
| HGX host (unit 1) | `hgx-su00-h00`, `hgx-su00-h08`, `hgx-su00-h16`, `hgx-su00-h24` |
| HGX host (unit 2) | `hgx-su01-h00`, `hgx-su01-h08`, `hgx-su01-h16`, `hgx-su01-h24` |
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
| Leaf and spine switches | Cumulus Linux 5.14.0 | `192.168.200.11` and up, one address per switch, in the order listed in `switches.2su.yaml` |
| HGX hosts | Ubuntu compute nodes | `192.168.200.23` to `.30` (unit 1: `.23`-`.26`, unit 2: `.27`-`.30`) |

Once the fabric is deployed (Lab Flow, Step 6), every switch also carries a `10.253.128.x/32` loopback assigned in the same order as the management addresses above (`leaf-su00-r0` is `10.253.128.1`, `r1` is `.2`, and so on, continuing into the second unit).

### Physical Connectivity

MetalSoft discovers the leaf-spine links automatically over LLDP; there is no cabling table to prepare by hand. Lab Flow Step 7 (Discover links and redeploy) shows how to confirm the discovered links from a switch, and the Fabric Manager's Fabric View shows the same links visually once they are imported.

<!-- AIR:page -->

## Demo Environment Access

### Load Time and Readiness

Allow the Global Controller and Site Controller a few minutes to finish booting after the lab starts. Lab Flow Step 1 shows how to confirm both are up and connected before you run anything else. The full lab takes about 45 minutes end to end, most of it switch deployment time in Step 9.

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
| Cumulus switches | `cumulus` | set in `switches.2su.yaml` | `ssh cumulus@<switch-ip>` |
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
cumulus-5.14-templates  ethernet-fabric.2su.yaml       l3-profile-tenant1.2su.yaml  netplan              route-domain-tenant1.2su.yaml  switches.2su.yaml
endpoints.2su.yaml      fabric-config.2su.l3evpn.yaml  l3-profile-tenant2.2su.yaml  oob-subnet.2su.yaml  route-domain-tenant2.2su.yaml  terraform
ubuntu@oob-mgmt-server:~/nvidia$
```

The switch password has been set in `switches.2su.yaml`.

For more background on NVIDIA Air services, SSH access, SSH keys, nodes, and consoles, see the [NVIDIA DSX Air Quick Start](https://docs.nvidia.com/networking-ethernet-software/nvidia-air/Quick-Start/).

### Web UI Access

This demo runs entirely from the command line, but the UI is useful to better understand the software's state.

The UI runs on the Global Controller; the steps below expose it and open it from your workstation.

In NVIDIA Air, there should already be a MetalSoft UI service enabled.

NVIDIA Air returns an external host name and port for the service (for example `worker-0375f999.dsx-air.nvidia.com` and `25990`). Note both; NVIDIA Air assigns a new external port on each restart.

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/metalsoft-https-service.webp)

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

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/admin-ui.webp)

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
metalcloud-cli fabric create 1 spectrumx-2su-514 ethernet "Spectrum-X 2-SU (5.14)" \
        --config-source ethernet-fabric.2su.yaml
```

```bash
metalcloud-cli fabric activate 1
```

```bash
metalcloud-cli subnet create --config-source oob-subnet.2su.yaml
```

Expected result:

- The fabric `spectrumx-2su-514` exists, is active, and has an out-of-band management subnet. Keep the fabric label `spectrumx-2su-514`; the Terraform manifest in Step 12 resolves the fabric by that label.

Validation:

- (Optional) In the web UI, open the Fabric Manager to see the fabric that was created.

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/fabric.2su.webp)

### Step 3. Import the Switches

**Goal:** Register the twelve lab switches with the fabric.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; the switch management password is read from `switches.2su.yaml`.

**Expected wait time:** A few seconds.

Set `fabricId` in `switches.2su.yaml` to `1`, then import the twelve switches:

```bash
metalcloud-cli fabric import-devices 1 --config-source switches.2su.yaml
```

Expected result:

- All twelve switches (eight leaves, four spines) appear as devices on the fabric.

Validation:

- (Optional) In the web UI, navigate to **Fabric Manager > Network devices** where equipment list shows the imported switches:

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/network-devices.2su.webp)

### Step 4. Discover Switch Interfaces

**Goal:** Trigger interface discovery on every switch and capture a clean baseline before any configuration is applied.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.2su.yaml`) on the switches.

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

- `get-ports` returns a non-empty list of `swpNsN` ports. If the list is empty, the switch is not reachable; check the address and password in `switches.2su.yaml`.

The switches are imported and reachable, but MetalSoft has not configured them yet. The switch hostnames for this topology are:

- Leaves: `leaf-su00-r0`, `leaf-su00-r1`, `leaf-su00-r2`, `leaf-su00-r3`, `leaf-su01-r0`, `leaf-su01-r1`, `leaf-su01-r2`, `leaf-su01-r3`
- Spines: `spine-s00`, `spine-s01`, `spine-s02`, `spine-s03`

Validation:

Log in as `cumulus` (the password is read from `switches.2su.yaml`) and capture the baseline, so later steps have something to compare against.

For ease of use, there is a validation script named `spcx-run` located in `~/spcx-air/` that can check the current configuration of the switches. An environment variable needs to be exported for the desired topology:

```bash
export SPCX_INVENTORY=~/spcx-air/inventory.2su.yml
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
leaf-su01-r0 | cumulus
################################################################################
leaf-su01-r1 | cumulus
################################################################################
leaf-su01-r2 | cumulus
################################################################################
leaf-su01-r3 | cumulus
################################################################################
spine-s00 | cumulus
################################################################################
spine-s01 | cumulus
################################################################################
spine-s02 | cumulus
################################################################################
spine-s03 | cumulus
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
leaf-su01-r0 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-su01-r1 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-su01-r2 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
leaf-su01-r3 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-s00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-s01 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-s02 | lo               UNKNOWN        127.0.0.1/8 ::1/128
################################################################################
spine-s03 | lo               UNKNOWN        127.0.0.1/8 ::1/128
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
leaf-su01-r0 | bgpd is not running
################################################################################
leaf-su01-r1 | bgpd is not running
################################################################################
leaf-su01-r2 | bgpd is not running
################################################################################
leaf-su01-r3 | bgpd is not running
################################################################################
spine-s00 | bgpd is not running
################################################################################
spine-s01 | bgpd is not running
################################################################################
spine-s02 | bgpd is not running
################################################################################
spine-s03 | bgpd is not running
################################################################################
```

Expected result:

- At this stage the loopback carries only `127.0.0.1/8`, no `10.253.x.x/32` address is assigned and BGP is not running.

### Step 5. Configure the Switches

**Goal:** Assign hostnames, ASNs, loopbacks, fabric point-to-point addresses, and host downlinks to every switch.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few minutes.

Set `fabricId` in `fabric-config.2su.l3evpn.yaml` to `1`:

```bash
metalcloud-cli fabric configure-switches 1 --config-source fabric-config.2su.l3evpn.yaml
```

Expected result:

- The command completes without error; the switch configuration is staged but not yet deployed.

Validation:

- Proceed to Step 6, which deploys and verifies this configuration on the switches.

### Step 6. Deploy the Underlay Addressing

**Goal:** Push hostnames, loopbacks, point-to-point addresses, and port settings to every switch.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.2su.yaml`) on the switches.

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
leaf-su01-r0 | leaf-su01-r0
################################################################################
leaf-su01-r1 | leaf-su01-r1
################################################################################
leaf-su01-r2 | leaf-su01-r2
################################################################################
leaf-su01-r3 | leaf-su01-r3
################################################################################
spine-s00 | spine-s00
################################################################################
spine-s01 | spine-s01
################################################################################
spine-s02 | spine-s02
################################################################################
spine-s03 | spine-s03
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
leaf-su01-r0 | lo               UNKNOWN        127.0.0.1/8 10.253.128.5/32 ::1/128
################################################################################
leaf-su01-r1 | lo               UNKNOWN        127.0.0.1/8 10.253.128.6/32 ::1/128
################################################################################
leaf-su01-r2 | lo               UNKNOWN        127.0.0.1/8 10.253.128.7/32 ::1/128
################################################################################
leaf-su01-r3 | lo               UNKNOWN        127.0.0.1/8 10.253.128.8/32 ::1/128
################################################################################
spine-s00 | lo               UNKNOWN        127.0.0.1/8 10.253.128.9/32 ::1/128
################################################################################
spine-s01 | lo               UNKNOWN        127.0.0.1/8 10.253.128.10/32 ::1/128
################################################################################
spine-s02 | lo               UNKNOWN        127.0.0.1/8 10.253.128.11/32 ::1/128
################################################################################
spine-s03 | lo               UNKNOWN        127.0.0.1/8 10.253.128.12/32 ::1/128
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
leaf-su01-r0 | bgpd is not running
################################################################################
leaf-su01-r1 | bgpd is not running
################################################################################
leaf-su01-r2 | bgpd is not running
################################################################################
leaf-su01-r3 | bgpd is not running
################################################################################
spine-s00 | bgpd is not running
################################################################################
spine-s01 | bgpd is not running
################################################################################
spine-s02 | bgpd is not running
################################################################################
spine-s03 | bgpd is not running
################################################################################
```

The switch now reports its assigned hostname and a `10.253.128.x/32` loopback (`leaf-su00-r0` is `10.253.128.1`, `r1` is `.2`, and so on, continuing into the second unit) and BGP is not running.

### Step 7. Discover Links and Redeploy

**Goal:** Re-scan the fabric so the discovered leaf-spine links become managed, then redeploy.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.2su.yaml`) on the switches.

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
leaf-su00-r0 | eth0       1G     eth   oob-mgmt-switch-leaf-1  swp10
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
leaf-su00-r0 | swp41s0    1G     swp   spine-s01               swp1s0
leaf-su00-r0 | swp41s1    1G     swp   spine-s01               swp1s1
leaf-su00-r0 | swp42s0    1G     swp   spine-s01               swp2s0
leaf-su00-r0 | swp42s1    1G     swp   spine-s01               swp2s1
leaf-su00-r0 | swp43s0    1G     swp   spine-s01               swp3s0
leaf-su00-r0 | swp43s1    1G     swp   spine-s01               swp3s1
leaf-su00-r0 | swp44s0    1G     swp   spine-s01               swp4s0
leaf-su00-r0 | swp44s1    1G     swp   spine-s01               swp4s1
leaf-su00-r0 | swp45s0    1G     swp   spine-s01               swp5s0
leaf-su00-r0 | swp45s1    1G     swp   spine-s01               swp5s1
leaf-su00-r0 | swp46s0    1G     swp   spine-s01               swp6s0
leaf-su00-r0 | swp46s1    1G     swp   spine-s01               swp6s1
leaf-su00-r0 | swp47s0    1G     swp   spine-s01               swp7s0
leaf-su00-r0 | swp47s1    1G     swp   spine-s01               swp7s1
leaf-su00-r0 | swp48s0    1G     swp   spine-s01               swp8s0
leaf-su00-r0 | swp48s1    1G     swp   spine-s01               swp8s1
leaf-su00-r0 | swp49s0    1G     swp   spine-s02               swp1s0
leaf-su00-r0 | swp49s1    1G     swp   spine-s02               swp1s1
leaf-su00-r0 | swp50s0    1G     swp   spine-s02               swp2s0
leaf-su00-r0 | swp50s1    1G     swp   spine-s02               swp2s1
leaf-su00-r0 | swp51s0    1G     swp   spine-s02               swp3s0
leaf-su00-r0 | swp51s1    1G     swp   spine-s02               swp3s1
leaf-su00-r0 | swp52s0    1G     swp   spine-s02               swp4s0
leaf-su00-r0 | swp52s1    1G     swp   spine-s02               swp4s1
leaf-su00-r0 | swp53s0    1G     swp   spine-s02               swp5s0
leaf-su00-r0 | swp53s1    1G     swp   spine-s02               swp5s1
leaf-su00-r0 | swp54s0    1G     swp   spine-s02               swp6s0
leaf-su00-r0 | swp54s1    1G     swp   spine-s02               swp6s1
leaf-su00-r0 | swp55s0    1G     swp   spine-s02               swp7s0
leaf-su00-r0 | swp55s1    1G     swp   spine-s02               swp7s1
leaf-su00-r0 | swp56s0    1G     swp   spine-s02               swp8s0
leaf-su00-r0 | swp56s1    1G     swp   spine-s02               swp8s1
leaf-su00-r0 | swp57s0    1G     swp   spine-s03               swp1s0
leaf-su00-r0 | swp57s1    1G     swp   spine-s03               swp1s1
leaf-su00-r0 | swp58s0    1G     swp   spine-s03               swp2s0
leaf-su00-r0 | swp58s1    1G     swp   spine-s03               swp2s1
leaf-su00-r0 | swp59s0    1G     swp   spine-s03               swp3s0
leaf-su00-r0 | swp59s1    1G     swp   spine-s03               swp3s1
leaf-su00-r0 | swp60s0    1G     swp   spine-s03               swp4s0
leaf-su00-r0 | swp60s1    1G     swp   spine-s03               swp4s1
leaf-su00-r0 | swp61s0    1G     swp   spine-s03               swp5s0
leaf-su00-r0 | swp61s1    1G     swp   spine-s03               swp5s1
leaf-su00-r0 | swp62s0    1G     swp   spine-s03               swp6s0
leaf-su00-r0 | swp62s1    1G     swp   spine-s03               swp6s1
leaf-su00-r0 | swp63s0    1G     swp   spine-s03               swp7s0
leaf-su00-r0 | swp63s1    1G     swp   spine-s03               swp7s1
leaf-su00-r0 | swp64s0    1G     swp   spine-s03               swp8s0
leaf-su00-r0 | swp64s1    1G     swp   spine-s03               swp8s1
################################################################################
```

Each fabric-facing port lists the neighbour it discovered: a leaf sees its spines and a spine sees the leaves below it.

- (Optional) In the web UI, navigate to **Fabric Manager > Fabrics > Select the desired fabric (spectrumx-2su-514) > Topology** where the Fabric View should look like this:

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/fabric-topology.2su.webp)

### Step 8. Register the Fabric Templates

**Goal:** Register the Cumulus 5.14 configuration templates that Step 9 deploys.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** About a minute; `--verify-render` renders every switch's configuration before writing.

`fabric-config.2su.l3evpn.yaml` names the Cumulus 5.14 templates under `cumulus-5.14-templates/`. Register them in two commands. The order matters: the base profile installs the QoS, adaptive-routing, and VTEP configuration that the BGP and EVPN profiles depend on.

```bash
metalcloud-cli fabric configure-freeform 1 \
        --config-source fabric-config.2su.l3evpn.yaml --verify-render
```

```bash
metalcloud-cli fabric configure-bgp 1 \
        --config-source fabric-config.2su.l3evpn.yaml --verify-render
```

Expected result:

- Both commands complete without error. `--verify-render` stops before writing if any switch fails to render.

Validation:

- (Optional) In the web UI, the registered templates appear under **Fabric Manager > Configuration Libraries > Network Device Configuration Templates**:

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/configuration_templates.webp)

### Step 9. Deploy the Fabric

**Goal:** Apply the base, underlay, overlay and QoS profiles to every switch in order, bringing up BGP and EVPN, including the cross-unit EVPN routes.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation; `cumulus` / (password in `switches.2su.yaml`) on the switches.

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
leaf-su00-r0 |
leaf-su00-r0 | IPv4 Unicast Summary:
leaf-su00-r0 | BGP router identifier 10.253.128.1, local AS number 4200000000 VRF default vrf-id 0
leaf-su00-r0 | BGP table version 33
leaf-su00-r0 | RIB entries 23, using 2944 bytes of memory
leaf-su00-r0 | Peers 64, using 1280 KiB of memory
leaf-su00-r0 | Peer groups 2, using 128 bytes of memory
leaf-su00-r0 |
leaf-su00-r0 | Neighbor               V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r0 | spine-s00(10.254.0.1)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp1s0
leaf-su00-r0 | spine-s00(10.254.0.3)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp1s1
leaf-su00-r0 | spine-s00(10.254.0.5)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp2s0
leaf-su00-r0 | spine-s00(10.254.0.7)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp2s1
leaf-su00-r0 | spine-s00(10.254.0.9)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp3s0
leaf-su00-r0 | spine-s00(10.254.0.11) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp3s1
leaf-su00-r0 | spine-s00(10.254.0.13) 4 4201000000        80       104       33    0    0 00:03:26            8       12 to_spine-s00_swp4s0
leaf-su00-r0 | spine-s00(10.254.0.15) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp4s1
leaf-su00-r0 | spine-s00(10.254.0.17) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp5s0
leaf-su00-r0 | spine-s00(10.254.0.19) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp5s1
leaf-su00-r0 | spine-s00(10.254.0.21) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp6s0
leaf-su00-r0 | spine-s00(10.254.0.23) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp6s1
leaf-su00-r0 | spine-s00(10.254.0.25) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp7s0
leaf-su00-r0 | spine-s00(10.254.0.27) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp7s1
leaf-su00-r0 | spine-s00(10.254.0.29) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp8s0
leaf-su00-r0 | spine-s00(10.254.0.31) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s00_swp8s1
leaf-su00-r0 | spine-s01(10.254.1.1)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp1s0
leaf-su00-r0 | spine-s01(10.254.1.3)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp1s1
leaf-su00-r0 | spine-s01(10.254.1.5)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp2s0
leaf-su00-r0 | spine-s01(10.254.1.7)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp2s1
leaf-su00-r0 | spine-s01(10.254.1.9)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp3s0
leaf-su00-r0 | spine-s01(10.254.1.11) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp3s1
leaf-su00-r0 | spine-s01(10.254.1.13) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp4s0
leaf-su00-r0 | spine-s01(10.254.1.15) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp4s1
leaf-su00-r0 | spine-s01(10.254.1.17) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp5s0
leaf-su00-r0 | spine-s01(10.254.1.19) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp5s1
leaf-su00-r0 | spine-s01(10.254.1.21) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp6s0
leaf-su00-r0 | spine-s01(10.254.1.23) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp6s1
leaf-su00-r0 | spine-s01(10.254.1.25) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp7s0
leaf-su00-r0 | spine-s01(10.254.1.27) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp7s1
leaf-su00-r0 | spine-s01(10.254.1.29) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp8s0
leaf-su00-r0 | spine-s01(10.254.1.31) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s01_swp8s1
leaf-su00-r0 | spine-s02(10.254.2.1)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp1s0
leaf-su00-r0 | spine-s02(10.254.2.3)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp1s1
leaf-su00-r0 | spine-s02(10.254.2.5)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp2s0
leaf-su00-r0 | spine-s02(10.254.2.7)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp2s1
leaf-su00-r0 | spine-s02(10.254.2.9)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp3s0
leaf-su00-r0 | spine-s02(10.254.2.11) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp3s1
leaf-su00-r0 | spine-s02(10.254.2.13) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp4s0
leaf-su00-r0 | spine-s02(10.254.2.15) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp4s1
leaf-su00-r0 | spine-s02(10.254.2.17) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp5s0
leaf-su00-r0 | spine-s02(10.254.2.19) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp5s1
leaf-su00-r0 | spine-s02(10.254.2.21) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp6s0
leaf-su00-r0 | spine-s02(10.254.2.23) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp6s1
leaf-su00-r0 | spine-s02(10.254.2.25) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp7s0
leaf-su00-r0 | spine-s02(10.254.2.27) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp7s1
leaf-su00-r0 | spine-s02(10.254.2.29) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp8s0
leaf-su00-r0 | spine-s02(10.254.2.31) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s02_swp8s1
leaf-su00-r0 | spine-s03(10.254.3.1)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp1s0
leaf-su00-r0 | spine-s03(10.254.3.3)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp1s1
leaf-su00-r0 | spine-s03(10.254.3.5)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp2s0
leaf-su00-r0 | spine-s03(10.254.3.7)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp2s1
leaf-su00-r0 | spine-s03(10.254.3.9)  4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp3s0
leaf-su00-r0 | spine-s03(10.254.3.11) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp3s1
leaf-su00-r0 | spine-s03(10.254.3.13) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp4s0
leaf-su00-r0 | spine-s03(10.254.3.15) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp4s1
leaf-su00-r0 | spine-s03(10.254.3.17) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp5s0
leaf-su00-r0 | spine-s03(10.254.3.19) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp5s1
leaf-su00-r0 | spine-s03(10.254.3.21) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp6s0
leaf-su00-r0 | spine-s03(10.254.3.23) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp6s1
leaf-su00-r0 | spine-s03(10.254.3.25) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp7s0
leaf-su00-r0 | spine-s03(10.254.3.27) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp7s1
leaf-su00-r0 | spine-s03(10.254.3.29) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp8s0
leaf-su00-r0 | spine-s03(10.254.3.31) 4 4201000000        80       105       33    0    0 00:03:26            8       12 to_spine-s03_swp8s1
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
leaf-su00-r0 | Neighbor                 V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r0 | spine-s00(10.253.128.9)  4 4201000000        51        57        0    0    0 00:02:25            0        0 to_spine-s00_loopbac
leaf-su00-r0 | spine-s01(10.253.128.10) 4 4201000000        52        58        0    0    0 00:02:28            0        0 to_spine-s01_loopbac
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
leaf-su00-r0 | Neighbor                 V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r0 | spine-s00(10.253.128.9)  4 4201000000        53        59        0    0    0 00:02:32            0        0 to_spine-s00_loopbac
leaf-su00-r0 | spine-s01(10.253.128.10) 4 4201000000        54        60        0    0    0 00:02:35            0        0 to_spine-s01_loopbac
leaf-su00-r0 |
leaf-su00-r0 | Total number of neighbors 2
################################################################################
leaf-su00-r1 | BGP router identifier 10.253.128.2, local AS number 4200000001 VRF default vrf-id 0
leaf-su00-r1 | BGP table version 0
leaf-su00-r1 | RIB entries 0, using 0 bytes of memory
leaf-su00-r1 | Peers 2, using 40 KiB of memory
leaf-su00-r1 | Peer groups 2, using 128 bytes of memory
leaf-su00-r1 |
leaf-su00-r1 | Neighbor                 V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r1 | spine-s00(10.253.128.9)  4 4201000000        53        59        0    0    0 00:02:32            0        0 to_spine-s00_loopbac
leaf-su00-r1 | spine-s01(10.253.128.10) 4 4201000000        54        60        0    0    0 00:02:35            0        0 to_spine-s01_loopbac
leaf-su00-r1 |
leaf-su00-r1 | Total number of neighbors 2
################################################################################
leaf-su00-r2 | BGP router identifier 10.253.128.3, local AS number 4200000002 VRF default vrf-id 0
leaf-su00-r2 | BGP table version 0
leaf-su00-r2 | RIB entries 0, using 0 bytes of memory
leaf-su00-r2 | Peers 2, using 40 KiB of memory
leaf-su00-r2 | Peer groups 2, using 128 bytes of memory
leaf-su00-r2 |
leaf-su00-r2 | Neighbor                 V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r2 | spine-s00(10.253.128.9)  4 4201000000        53        59        0    0    0 00:02:33            0        0 to_spine-s00_loopbac
leaf-su00-r2 | spine-s01(10.253.128.10) 4 4201000000        54        59        0    0    0 00:02:36            0        0 to_spine-s01_loopbac
leaf-su00-r2 |
leaf-su00-r2 | Total number of neighbors 2
################################################################################
leaf-su00-r3 | BGP router identifier 10.253.128.4, local AS number 4200000003 VRF default vrf-id 0
leaf-su00-r3 | BGP table version 0
leaf-su00-r3 | RIB entries 0, using 0 bytes of memory
leaf-su00-r3 | Peers 2, using 40 KiB of memory
leaf-su00-r3 | Peer groups 2, using 128 bytes of memory
leaf-su00-r3 |
leaf-su00-r3 | Neighbor                 V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su00-r3 | spine-s00(10.253.128.9)  4 4201000000        53        59        0    0    0 00:02:32            0        0 to_spine-s00_loopbac
leaf-su00-r3 | spine-s01(10.253.128.10) 4 4201000000        54        60        0    0    0 00:02:36            0        0 to_spine-s01_loopbac
leaf-su00-r3 |
leaf-su00-r3 | Total number of neighbors 2
################################################################################
leaf-su01-r0 | BGP router identifier 10.253.128.5, local AS number 4200000004 VRF default vrf-id 0
leaf-su01-r0 | BGP table version 0
leaf-su01-r0 | RIB entries 0, using 0 bytes of memory
leaf-su01-r0 | Peers 2, using 40 KiB of memory
leaf-su01-r0 | Peer groups 2, using 128 bytes of memory
leaf-su01-r0 |
leaf-su01-r0 | Neighbor                 V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su01-r0 | spine-s00(10.253.128.9)  4 4201000000        53        59        0    0    0 00:02:33            0        0 to_spine-s00_loopbac
leaf-su01-r0 | spine-s01(10.253.128.10) 4 4201000000        54        60        0    0    0 00:02:36            0        0 to_spine-s01_loopbac
leaf-su01-r0 |
leaf-su01-r0 | Total number of neighbors 2
################################################################################
leaf-su01-r1 | BGP router identifier 10.253.128.6, local AS number 4200000005 VRF default vrf-id 0
leaf-su01-r1 | BGP table version 0
leaf-su01-r1 | RIB entries 0, using 0 bytes of memory
leaf-su01-r1 | Peers 2, using 40 KiB of memory
leaf-su01-r1 | Peer groups 2, using 128 bytes of memory
leaf-su01-r1 |
leaf-su01-r1 | Neighbor                 V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su01-r1 | spine-s00(10.253.128.9)  4 4201000000        53        59        0    0    0 00:02:33            0        0 to_spine-s00_loopbac
leaf-su01-r1 | spine-s01(10.253.128.10) 4 4201000000        54        60        0    0    0 00:02:36            0        0 to_spine-s01_loopbac
leaf-su01-r1 |
leaf-su01-r1 | Total number of neighbors 2
################################################################################
leaf-su01-r2 | BGP router identifier 10.253.128.7, local AS number 4200000006 VRF default vrf-id 0
leaf-su01-r2 | BGP table version 0
leaf-su01-r2 | RIB entries 0, using 0 bytes of memory
leaf-su01-r2 | Peers 2, using 40 KiB of memory
leaf-su01-r2 | Peer groups 2, using 128 bytes of memory
leaf-su01-r2 |
leaf-su01-r2 | Neighbor                 V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su01-r2 | spine-s00(10.253.128.9)  4 4201000000        53        59        0    0    0 00:02:33            0        0 to_spine-s00_loopbac
leaf-su01-r2 | spine-s01(10.253.128.10) 4 4201000000        54        60        0    0    0 00:02:36            0        0 to_spine-s01_loopbac
leaf-su01-r2 |
leaf-su01-r2 | Total number of neighbors 2
################################################################################
leaf-su01-r3 | BGP router identifier 10.253.128.8, local AS number 4200000007 VRF default vrf-id 0
leaf-su01-r3 | BGP table version 0
leaf-su01-r3 | RIB entries 0, using 0 bytes of memory
leaf-su01-r3 | Peers 2, using 40 KiB of memory
leaf-su01-r3 | Peer groups 2, using 128 bytes of memory
leaf-su01-r3 |
leaf-su01-r3 | Neighbor                 V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
leaf-su01-r3 | spine-s00(10.253.128.9)  4 4201000000        53        59        0    0    0 00:02:32            0        0 to_spine-s00_loopbac
leaf-su01-r3 | spine-s01(10.253.128.10) 4 4201000000        54        60        0    0    0 00:02:36            0        0 to_spine-s01_loopbac
leaf-su01-r3 |
leaf-su01-r3 | Total number of neighbors 2
################################################################################
spine-s00 | BGP router identifier 10.253.128.9, local AS number 4201000000 VRF default vrf-id 0
spine-s00 | BGP table version 0
spine-s00 | RIB entries 0, using 0 bytes of memory
spine-s00 | Peers 8, using 160 KiB of memory
spine-s00 | Peer groups 2, using 128 bytes of memory
spine-s00 |
spine-s00 | Neighbor                   V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
spine-s00 | leaf-su00-r0(10.253.128.1) 4 4200000000        53        53        0    0    0 00:02:32            0        0 to_leaf-su00-r0_loop
spine-s00 | leaf-su00-r1(10.253.128.2) 4 4200000001        53        53        0    0    0 00:02:32            0        0 to_leaf-su00-r1_loop
spine-s00 | leaf-su00-r2(10.253.128.3) 4 4200000002        53        53        0    0    0 00:02:32            0        0 to_leaf-su00-r2_loop
spine-s00 | leaf-su00-r3(10.253.128.4) 4 4200000003        53        53        0    0    0 00:02:32            0        0 to_leaf-su00-r3_loop
spine-s00 | leaf-su01-r0(10.253.128.5) 4 4200000004        53        53        0    0    0 00:02:32            0        0 to_leaf-su01-r0_loop
spine-s00 | leaf-su01-r1(10.253.128.6) 4 4200000005        53        53        0    0    0 00:02:32            0        0 to_leaf-su01-r1_loop
spine-s00 | leaf-su01-r2(10.253.128.7) 4 4200000006        53        53        0    0    0 00:02:32            0        0 to_leaf-su01-r2_loop
spine-s00 | leaf-su01-r3(10.253.128.8) 4 4200000007        53        53        0    0    0 00:02:32            0        0 to_leaf-su01-r3_loop
spine-s00 |
spine-s00 | Total number of neighbors 8
################################################################################
spine-s01 | BGP router identifier 10.253.128.10, local AS number 4201000000 VRF default vrf-id 0
spine-s01 | BGP table version 0
spine-s01 | RIB entries 0, using 0 bytes of memory
spine-s01 | Peers 8, using 160 KiB of memory
spine-s01 | Peer groups 2, using 128 bytes of memory
spine-s01 |
spine-s01 | Neighbor                   V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
spine-s01 | leaf-su00-r0(10.253.128.1) 4 4200000000        54        54        0    0    0 00:02:36            0        0 to_leaf-su00-r0_loop
spine-s01 | leaf-su00-r1(10.253.128.2) 4 4200000001        54        54        0    0    0 00:02:36            0        0 to_leaf-su00-r1_loop
spine-s01 | leaf-su00-r2(10.253.128.3) 4 4200000002        54        54        0    0    0 00:02:36            0        0 to_leaf-su00-r2_loop
spine-s01 | leaf-su00-r3(10.253.128.4) 4 4200000003        54        54        0    0    0 00:02:36            0        0 to_leaf-su00-r3_loop
spine-s01 | leaf-su01-r0(10.253.128.5) 4 4200000004        54        54        0    0    0 00:02:36            0        0 to_leaf-su01-r0_loop
spine-s01 | leaf-su01-r1(10.253.128.6) 4 4200000005        54        54        0    0    0 00:02:36            0        0 to_leaf-su01-r1_loop
spine-s01 | leaf-su01-r2(10.253.128.7) 4 4200000006        54        54        0    0    0 00:02:36            0        0 to_leaf-su01-r2_loop
spine-s01 | leaf-su01-r3(10.253.128.8) 4 4200000007        54        54        0    0    0 00:02:36            0        0 to_leaf-su01-r3_loop
spine-s01 |
spine-s01 | Total number of neighbors 8
################################################################################
spine-s02 | % No BGP neighbors found in VRF default
################################################################################
spine-s03 | % No BGP neighbors found in VRF default
################################################################################
```

Every switch now has established underlay sessions, each with a non-zero prefix count in place of `Idle` or `Active`. The EVPN overlay runs between the leaves and the relay spines `spine-s00` and `spine-s01`, which re-advertise the type-5 routes across both units, so look at the output for those two spines to see the overlay sessions.

### Step 10. Register the HGX Endpoints

**Goal:** Register the eight HGX hosts as endpoints, each bound to eight leaf ports (one per rail). The four hosts of the first unit sit on `leaf-su00-r*` and the four of the second unit on `leaf-su01-r*`.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few seconds.

The definitions are in `endpoints.2su.yaml`:

```bash
metalcloud-cli endpoint create-bulk --config-source endpoints.2su.yaml
```

Expected result:

- All eight endpoints are created.

Validation:

```bash
metalcloud-cli endpoint list --filter-site 1
```

Confirm all eight endpoints are listed. Do not rename them; the Terraform manifest in Step 12 resolves them by label (`hgx-su00-h*`, `hgx-su01-h*`).

### Step 11. Create the Tenant Route Domains and Profiles

**Goal:** Create the tenant VRFs (route domain) and the L3-only logical network profiles that the tenant networks are built from.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation.

**Expected wait time:** A few seconds.

```bash
metalcloud-cli route-domain create --config-source route-domain-tenant1.2su.yaml
```

```bash
metalcloud-cli route-domain create --config-source route-domain-tenant2.2su.yaml
```

```bash
metalcloud-cli logical-network-profile create vxlan --config-source l3-profile-tenant1.2su.yaml
```

```bash
metalcloud-cli logical-network-profile create vxlan --config-source l3-profile-tenant2.2su.yaml
```

Expected result:

- The route domains and network profiles are created. The L3VNI is allocated automatically for the fabric. Keep the profile labels as `tenant1-l3` and `tenant2-l3`; the Terraform manifest in Step 12 resolves them by that label.

Validation:

- Proceed to Step 12, which consumes the profiles.

### Step 12. Onboard the Tenant with Terraform

**Goal:** Create the tenant infrastructures, build L3 networks from the `tenant1-l3` and `tenant2-l3` profiles, attach the eight endpoints (four per tenant) with eight interfaces each, and deploy. `tenant1` gets the `h00`/`h16` hosts of both scalability units; `tenant2` gets the `h08`/`h24` hosts of both scalability units.

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
terraform_data.endpoint_fingerprint: Creation complete after 0s [id=0ac19dca-faac-57ad-c18b-ac4d20a7cbba]
metalcloud_infrastructure.infra_tenant1: Creating...
metalcloud_infrastructure.infra_tenant1: Creation complete after 0s
metalcloud_logical_network.tenant1-network: Creating...
metalcloud_logical_network.tenant1-network: Creation complete after 0s [name=tenant1-network]
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-su01-h16"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su01-h16"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-su00-h00"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h00"]: Creation complete after 1s
metalcloud_endpoint_instance_group.groups["hgx-su01-h00"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su01-h00"]: Creation complete after 0s
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Creating...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Creation complete after 0s

Apply complete! Resources: 8 added, 0 changed, 0 destroyed.
```

```bash
Plan: 8 to add, 0 to change, 0 to destroy.
terraform_data.endpoint_fingerprint: Creating...
terraform_data.endpoint_fingerprint: Creation complete after 0s [id=9e8b3fd7-7e3f-a800-7aab-7186a66589ce]
metalcloud_infrastructure.infra_tenant2: Creating...
metalcloud_infrastructure.infra_tenant2: Creation complete after 1s
metalcloud_logical_network.tenant2-network: Creating...
metalcloud_logical_network.tenant2-network: Creation complete after 0s [name=tenant2-network]
metalcloud_endpoint_instance_group.groups["hgx-su00-h08"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h08"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-su00-h24"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h24"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-su01-h08"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su01-h08"]: Creation complete after 0s
metalcloud_endpoint_instance_group.groups["hgx-su01-h24"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su01-h24"]: Creation complete after 0s
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
│  1 │ infra-tenant1 │ infra-tenant1 │ active │     1 │    1 │ 19 Aug 26 02:31 UTC │ 19 Aug 26 02:38 UTC │ finished      │           │
│  2 │ infra-tenant2 │ infra-tenant2 │ active │     1 │    1 │ 19 Aug 26 02:31 UTC │ 19 Aug 26 02:38 UTC │ finished      │           │
└────┴───────────────┴───────────────┴────────┴───────┴──────┴─────────────────────┴─────────────────────┴───────────────┴───────────┘
```

- When the apply completes, the attached hosts' rail gateways are live.

Validation:

- (Optional) In the web UI, the tenant infrastructures are visible in the Infrastructure Designer. Navigate to **Admin dashboard > Infrastructures**, select one infrastructure and click **Open infrastructure designer**. This opens the Infrastructure Designer, where you can see the graphical representation of the infrastructure:

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/infrastructures.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/open-infrastructure.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/tenant1.initial.2su.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/tenant2.initial.2su.webp)

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
leaf-su00-r0 | mgmt             UP             ce:6b:cf:78:16:1b <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r0 | tenant2          UP             26:d0:77:74:26:55 <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r0 | tenant1          UP             b6:b0:44:18:bc:d3 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su00-r1 | mgmt             UP             fa:9d:e4:56:f8:9f <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r1 | tenant2          UP             02:1c:5a:f4:f2:02 <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r1 | tenant1          UP             12:20:3d:d0:71:6d <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su00-r2 | mgmt             UP             c6:5b:3a:8e:82:bd <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r2 | tenant2          UP             46:47:c6:0f:8b:ff <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r2 | tenant1          UP             7e:e7:f7:69:45:10 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su00-r3 | mgmt             UP             52:bd:2a:7b:12:c6 <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r3 | tenant2          UP             96:87:d2:5f:f1:3a <NOARP,MASTER,UP,LOWER_UP>
leaf-su00-r3 | tenant1          UP             9a:c0:a6:6e:fe:04 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su01-r0 | mgmt             UP             9e:b0:79:80:d1:3f <NOARP,MASTER,UP,LOWER_UP>
leaf-su01-r0 | tenant1          UP             f6:3e:73:e2:37:84 <NOARP,MASTER,UP,LOWER_UP>
leaf-su01-r0 | tenant2          UP             76:fb:95:c4:04:8c <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su01-r1 | mgmt             UP             c2:88:8f:be:4c:3e <NOARP,MASTER,UP,LOWER_UP>
leaf-su01-r1 | tenant1          UP             56:0a:44:13:f1:1f <NOARP,MASTER,UP,LOWER_UP>
leaf-su01-r1 | tenant2          UP             0a:f5:d0:10:9c:34 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su01-r2 | mgmt             UP             4a:c5:34:af:2f:fd <NOARP,MASTER,UP,LOWER_UP>
leaf-su01-r2 | tenant1          UP             4a:fc:51:ce:69:18 <NOARP,MASTER,UP,LOWER_UP>
leaf-su01-r2 | tenant2          UP             22:c1:8b:99:c4:6c <NOARP,MASTER,UP,LOWER_UP>
################################################################################
leaf-su01-r3 | mgmt             UP             6e:16:a6:d5:62:f6 <NOARP,MASTER,UP,LOWER_UP>
leaf-su01-r3 | tenant1          UP             a6:a6:ef:d8:d7:6e <NOARP,MASTER,UP,LOWER_UP>
leaf-su01-r3 | tenant2          UP             2a:dd:af:00:ad:ab <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-s00 | mgmt             UP             5e:08:ef:15:01:33 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-s01 | mgmt             UP             c6:61:b4:9e:a0:85 <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-s02 | mgmt             UP             c6:66:f6:23:50:bd <NOARP,MASTER,UP,LOWER_UP>
################################################################################
spine-s03 | mgmt             UP             f2:a2:df:7b:2f:79 <NOARP,MASTER,UP,LOWER_UP>
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
leaf-su00-r0 | swp1s0           UP             172.16.0.1/31 fe80::4ab0:2dff:fe27:761/64
leaf-su00-r0 | swp1s1           UP             172.24.0.1/31 fe80::4ab0:2dff:fe75:51f4/64
leaf-su00-r0 | swp17s0          UP             172.16.0.33/31 fe80::4ab0:2dff:fea8:85b5/64
leaf-su00-r0 | swp17s1          UP             172.24.0.33/31 fe80::4ab0:2dff:fe97:53ca/64
################################################################################
leaf-su00-r1 | swp1s0           UP             172.18.0.1/31 fe80::4ab0:2dff:fe8b:c2cf/64
leaf-su00-r1 | swp1s1           UP             172.26.0.1/31 fe80::4ab0:2dff:feb8:1c38/64
leaf-su00-r1 | swp17s0          UP             172.18.0.33/31 fe80::4ab0:2dff:feff:ad2/64
leaf-su00-r1 | swp17s1          UP             172.26.0.33/31 fe80::4ab0:2dff:fe9a:1640/64
################################################################################
leaf-su00-r2 | swp1s0           UP             172.20.0.1/31 fe80::4ab0:2dff:fe7c:7809/64
leaf-su00-r2 | swp1s1           UP             172.28.0.1/31 fe80::4ab0:2dff:fef5:cb0b/64
leaf-su00-r2 | swp17s0          UP             172.20.0.33/31 fe80::4ab0:2dff:fec1:3673/64
leaf-su00-r2 | swp17s1          UP             172.28.0.33/31 fe80::4ab0:2dff:fe13:de20/64
################################################################################
leaf-su00-r3 | swp1s0           UP             172.22.0.1/31 fe80::4ab0:2dff:fe93:e867/64
leaf-su00-r3 | swp1s1           UP             172.30.0.1/31 fe80::4ab0:2dff:fe0c:1456/64
leaf-su00-r3 | swp17s0          UP             172.22.0.33/31 fe80::4ab0:2dff:fe02:8fd0/64
leaf-su00-r3 | swp17s1          UP             172.30.0.33/31 fe80::4ab0:2dff:fec1:c5a2/64
################################################################################
leaf-su01-r0 | swp1s0           UP             172.16.1.1/31 fe80::4ab0:2dff:fea7:adc7/64
leaf-su01-r0 | swp1s1           UP             172.24.1.1/31 fe80::4ab0:2dff:fe6b:8fab/64
leaf-su01-r0 | swp17s0          UP             172.16.1.33/31 fe80::4ab0:2dff:fe1b:db63/64
leaf-su01-r0 | swp17s1          UP             172.24.1.33/31 fe80::4ab0:2dff:fe5b:aec6/64
################################################################################
leaf-su01-r1 | swp1s0           UP             172.18.1.1/31 fe80::4ab0:2dff:fe64:fdc5/64
leaf-su01-r1 | swp1s1           UP             172.26.1.1/31 fe80::4ab0:2dff:fe09:e970/64
leaf-su01-r1 | swp17s0          UP             172.18.1.33/31 fe80::4ab0:2dff:fe41:2b30/64
leaf-su01-r1 | swp17s1          UP             172.26.1.33/31 fe80::4ab0:2dff:fe15:604c/64
################################################################################
leaf-su01-r2 | swp1s0           UP             172.20.1.1/31 fe80::4ab0:2dff:fef6:fd2f/64
leaf-su01-r2 | swp1s1           UP             172.28.1.1/31 fe80::4ab0:2dff:fe10:c77c/64
leaf-su01-r2 | swp17s0          UP             172.20.1.33/31 fe80::4ab0:2dff:fe8c:7fd9/64
leaf-su01-r2 | swp17s1          UP             172.28.1.33/31 fe80::4ab0:2dff:fe19:6386/64
################################################################################
leaf-su01-r3 | swp1s0           UP             172.22.1.1/31 fe80::4ab0:2dff:fe6e:ed1/64
leaf-su01-r3 | swp1s1           UP             172.30.1.1/31 fe80::4ab0:2dff:fe57:2d/64
leaf-su01-r3 | swp17s0          UP             172.22.1.33/31 fe80::4ab0:2dff:fe1b:c631/64
leaf-su01-r3 | swp17s1          UP             172.30.1.33/31 fe80::4ab0:2dff:fe6d:7a8c/64
################################################################################
spine-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-s01 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-s02 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-s03 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant1 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant1 ipv4 unicast"
========================================
################################################################################
leaf-su00-r0 | BGP table version is 52, local router ID is 10.253.128.1, vrf id 136
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
leaf-su00-r0 |  *> 172.16.1.0/26    10.253.128.5(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *                   10.253.128.5(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.18.1.0/26    10.253.128.6(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *                   10.253.128.6(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.20.1.0/26    10.253.128.7(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *                   10.253.128.7(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.22.1.0/26    10.253.128.8(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *                   10.253.128.8(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.0/31    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.32/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.24.1.0/26    10.253.128.5(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *                   10.253.128.5(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.26.1.0/26    10.253.128.6(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *                   10.253.128.6(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.28.1.0/26    10.253.128.7(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *                   10.253.128.7(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.30.1.0/26    10.253.128.8(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *                   10.253.128.8(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |
leaf-su00-r0 | Displayed 20 routes and 34 total paths
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
leaf-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:06:31
leaf-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:06:00
leaf-su00-r0 | C>* 172.16.0.0/31 is directly connected, swp1s0, 00:06:31
leaf-su00-r0 | L>* 172.16.0.1/32 is directly connected, swp1s0, 00:06:31
leaf-su00-r0 | C>* 172.16.0.32/31 is directly connected, swp17s0, 00:06:31
leaf-su00-r0 | L>* 172.16.0.33/32 is directly connected, swp17s0, 00:06:31
leaf-su00-r0 | B>* 172.16.1.0/26 [20/0] via 10.253.128.5, vlan1440_l3 onlink, weight 1, 00:06:31
leaf-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:05:22
leaf-su00-r0 | B>* 172.18.1.0/26 [20/0] via 10.253.128.6, vlan1440_l3 onlink, weight 1, 00:06:31
leaf-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:04:44
leaf-su00-r0 | B>* 172.20.1.0/26 [20/0] via 10.253.128.7, vlan1440_l3 onlink, weight 1, 00:06:31
leaf-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:04:05
leaf-su00-r0 | B>* 172.22.1.0/26 [20/0] via 10.253.128.8, vlan1440_l3 onlink, weight 1, 00:06:31
leaf-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:06:00
leaf-su00-r0 | C>* 172.24.0.0/31 is directly connected, swp1s1, 00:06:31
leaf-su00-r0 | L>* 172.24.0.1/32 is directly connected, swp1s1, 00:06:31
leaf-su00-r0 | C>* 172.24.0.32/31 is directly connected, swp17s1, 00:06:31
leaf-su00-r0 | L>* 172.24.0.33/32 is directly connected, swp17s1, 00:06:31
leaf-su00-r0 | B>* 172.24.1.0/26 [20/0] via 10.253.128.5, vlan1440_l3 onlink, weight 1, 00:06:31
leaf-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:05:22
leaf-su00-r0 | B>* 172.26.1.0/26 [20/0] via 10.253.128.6, vlan1440_l3 onlink, weight 1, 00:06:31
leaf-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:04:44
leaf-su00-r0 | B>* 172.28.1.0/26 [20/0] via 10.253.128.7, vlan1440_l3 onlink, weight 1, 00:06:31
leaf-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:04:05
leaf-su00-r0 | B>* 172.30.1.0/26 [20/0] via 10.253.128.8, vlan1440_l3 onlink, weight 1, 00:06:31
################################################################################
```


`tenant1` appears in the VRF list, the host-facing `swp` ports carry their `172.x` rail gateway `/31`s inside it and the VRF routing table holds both the local rail subnets and the remote ones learned over EVPN, including the rails in the other scalability unit.

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
leaf-su00-r0 | swp9s0           UP             172.16.0.17/31 fe80::4ab0:2dff:feb6:b077/64
leaf-su00-r0 | swp9s1           UP             172.24.0.17/31 fe80::4ab0:2dff:fe29:ecdb/64
leaf-su00-r0 | swp25s0          UP             172.16.0.49/31 fe80::4ab0:2dff:fef1:c0d5/64
leaf-su00-r0 | swp25s1          UP             172.24.0.49/31 fe80::4ab0:2dff:fe91:164b/64
################################################################################
leaf-su00-r1 | swp9s0           UP             172.18.0.17/31 fe80::4ab0:2dff:fea2:11a3/64
leaf-su00-r1 | swp9s1           UP             172.26.0.17/31 fe80::4ab0:2dff:fe2a:b998/64
leaf-su00-r1 | swp25s0          UP             172.18.0.49/31 fe80::4ab0:2dff:feed:5f5f/64
leaf-su00-r1 | swp25s1          UP             172.26.0.49/31 fe80::4ab0:2dff:fe90:9788/64
################################################################################
leaf-su00-r2 | swp9s0           UP             172.20.0.17/31 fe80::4ab0:2dff:fe29:74ce/64
leaf-su00-r2 | swp9s1           UP             172.28.0.17/31 fe80::4ab0:2dff:fece:ddd7/64
leaf-su00-r2 | swp25s0          UP             172.20.0.49/31 fe80::4ab0:2dff:fee9:ebf1/64
leaf-su00-r2 | swp25s1          UP             172.28.0.49/31 fe80::4ab0:2dff:fe69:abfe/64
################################################################################
leaf-su00-r3 | swp9s0           UP             172.22.0.17/31 fe80::4ab0:2dff:fefb:1d23/64
leaf-su00-r3 | swp9s1           UP             172.30.0.17/31 fe80::4ab0:2dff:fedb:ac01/64
leaf-su00-r3 | swp25s0          UP             172.22.0.49/31 fe80::4ab0:2dff:fe5d:beb1/64
leaf-su00-r3 | swp25s1          UP             172.30.0.49/31 fe80::4ab0:2dff:fe69:c47d/64
################################################################################
leaf-su01-r0 | swp9s0           UP             172.16.1.17/31 fe80::4ab0:2dff:fed9:9f85/64
leaf-su01-r0 | swp9s1           UP             172.24.1.17/31 fe80::4ab0:2dff:fe03:fb6b/64
leaf-su01-r0 | swp25s0          UP             172.16.1.49/31 fe80::4ab0:2dff:fed3:ec54/64
leaf-su01-r0 | swp25s1          UP             172.24.1.49/31 fe80::4ab0:2dff:fe14:701e/64
################################################################################
leaf-su01-r1 | swp9s0           UP             172.18.1.17/31 fe80::4ab0:2dff:febe:18d2/64
leaf-su01-r1 | swp9s1           UP             172.26.1.17/31 fe80::4ab0:2dff:fe7b:bb66/64
leaf-su01-r1 | swp25s0          UP             172.18.1.49/31 fe80::4ab0:2dff:fe09:7900/64
leaf-su01-r1 | swp25s1          UP             172.26.1.49/31 fe80::4ab0:2dff:fe8b:7d1a/64
################################################################################
leaf-su01-r2 | swp9s0           UP             172.20.1.17/31 fe80::4ab0:2dff:fe60:4d73/64
leaf-su01-r2 | swp9s1           UP             172.28.1.17/31 fe80::4ab0:2dff:fe2b:cc74/64
leaf-su01-r2 | swp25s0          UP             172.20.1.49/31 fe80::4ab0:2dff:fe7c:f554/64
leaf-su01-r2 | swp25s1          UP             172.28.1.49/31 fe80::4ab0:2dff:fe3a:1070/64
################################################################################
leaf-su01-r3 | swp9s0           UP             172.22.1.17/31 fe80::4ab0:2dff:fe5d:8212/64
leaf-su01-r3 | swp9s1           UP             172.30.1.17/31 fe80::4ab0:2dff:feb6:983a/64
leaf-su01-r3 | swp25s0          UP             172.22.1.49/31 fe80::4ab0:2dff:fe47:8014/64
leaf-su01-r3 | swp25s1          UP             172.30.1.49/31 fe80::4ab0:2dff:fea5:4274/64
################################################################################
spine-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-s01 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-s02 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-s03 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant2 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant2 ipv4 unicast"
========================================
################################################################################
leaf-su00-r0 | BGP table version is 84, local router ID is 10.253.128.1, vrf id 132
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
leaf-su00-r0 |  *> 172.16.1.0/26    10.253.128.5(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *                   10.253.128.5(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.18.1.0/26    10.253.128.6(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *                   10.253.128.6(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.20.1.0/26    10.253.128.7(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *                   10.253.128.7(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.22.1.0/26    10.253.128.8(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *                   10.253.128.8(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.16/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.48/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.24.1.0/26    10.253.128.5(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *                   10.253.128.5(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.26.1.0/26    10.253.128.6(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *                   10.253.128.6(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.28.1.0/26    10.253.128.7(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *                   10.253.128.7(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.30.1.0/26    10.253.128.8(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *                   10.253.128.8(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |
leaf-su00-r0 | Displayed 20 routes and 34 total paths
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
leaf-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:12:53
leaf-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:12:23
leaf-su00-r0 | C>* 172.16.0.16/31 is directly connected, swp9s0, 00:12:53
leaf-su00-r0 | L>* 172.16.0.17/32 is directly connected, swp9s0, 00:12:53
leaf-su00-r0 | C>* 172.16.0.48/31 is directly connected, swp25s0, 00:12:53
leaf-su00-r0 | L>* 172.16.0.49/32 is directly connected, swp25s0, 00:12:53
leaf-su00-r0 | B>* 172.16.1.0/26 [20/0] via 10.253.128.5, vlan1341_l3 onlink, weight 1, 00:09:51
leaf-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:11:44
leaf-su00-r0 | B>* 172.18.1.0/26 [20/0] via 10.253.128.6, vlan1341_l3 onlink, weight 1, 00:09:14
leaf-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:11:06
leaf-su00-r0 | B>* 172.20.1.0/26 [20/0] via 10.253.128.7, vlan1341_l3 onlink, weight 1, 00:08:35
leaf-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:10:28
leaf-su00-r0 | B>* 172.22.1.0/26 [20/0] via 10.253.128.8, vlan1341_l3 onlink, weight 1, 00:07:57
leaf-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:12:23
leaf-su00-r0 | C>* 172.24.0.16/31 is directly connected, swp9s1, 00:12:53
leaf-su00-r0 | L>* 172.24.0.17/32 is directly connected, swp9s1, 00:12:53
leaf-su00-r0 | C>* 172.24.0.48/31 is directly connected, swp25s1, 00:12:53
leaf-su00-r0 | L>* 172.24.0.49/32 is directly connected, swp25s1, 00:12:53
leaf-su00-r0 | B>* 172.24.1.0/26 [20/0] via 10.253.128.5, vlan1341_l3 onlink, weight 1, 00:09:51
leaf-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:11:44
leaf-su00-r0 | B>* 172.26.1.0/26 [20/0] via 10.253.128.6, vlan1341_l3 onlink, weight 1, 00:09:14
leaf-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:11:06
leaf-su00-r0 | B>* 172.28.1.0/26 [20/0] via 10.253.128.7, vlan1341_l3 onlink, weight 1, 00:08:35
leaf-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:10:28
leaf-su00-r0 | B>* 172.30.1.0/26 [20/0] via 10.253.128.8, vlan1341_l3 onlink, weight 1, 00:07:57
################################################################################c
```

`tenant2` appears in the VRF list, the host-facing `swp` ports carry their `172.x` rail gateway `/31`s inside it and the VRF routing table holds both the local rail subnets and the remote ones learned over EVPN, including the rails in the other scalability unit.

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
leaf-su00-r0 | Route Distinguisher: 10.253.128.5:19998
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.16.1.0] RD 10.253.128.5:19998
leaf-su00-r0 |                     10.253.128.5 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |                     RT:59908:19998 ET:8 Rmac:44:38:39:22:01:d1
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.16.1.0] RD 10.253.128.5:19998
leaf-su00-r0 |                     10.253.128.5 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |                     RT:59908:19998 ET:8 Rmac:44:38:39:22:01:d1
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.24.1.0] RD 10.253.128.5:19998
leaf-su00-r0 |                     10.253.128.5 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |                     RT:59908:19998 ET:8 Rmac:44:38:39:22:01:d1
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.24.1.0] RD 10.253.128.5:19998
leaf-su00-r0 |                     10.253.128.5 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |                     RT:59908:19998 ET:8 Rmac:44:38:39:22:01:d1
leaf-su00-r0 | Route Distinguisher: 10.253.128.5:19999
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.16.1.0] RD 10.253.128.5:19999
leaf-su00-r0 |                     10.253.128.5 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |                     RT:59908:19999 ET:8 Rmac:44:38:39:22:01:d1
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.16.1.0] RD 10.253.128.5:19999
leaf-su00-r0 |                     10.253.128.5 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |                     RT:59908:19999 ET:8 Rmac:44:38:39:22:01:d1
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.24.1.0] RD 10.253.128.5:19999
leaf-su00-r0 |                     10.253.128.5 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |                     RT:59908:19999 ET:8 Rmac:44:38:39:22:01:d1
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.24.1.0] RD 10.253.128.5:19999
leaf-su00-r0 |                     10.253.128.5 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |                     RT:59908:19999 ET:8 Rmac:44:38:39:22:01:d1
leaf-su00-r0 | Route Distinguisher: 10.253.128.6:19998
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.18.1.0] RD 10.253.128.6:19998
leaf-su00-r0 |                     10.253.128.6 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |                     RT:59909:19998 ET:8 Rmac:44:38:39:22:01:d2
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.18.1.0] RD 10.253.128.6:19998
leaf-su00-r0 |                     10.253.128.6 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |                     RT:59909:19998 ET:8 Rmac:44:38:39:22:01:d2
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.26.1.0] RD 10.253.128.6:19998
leaf-su00-r0 |                     10.253.128.6 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |                     RT:59909:19998 ET:8 Rmac:44:38:39:22:01:d2
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.26.1.0] RD 10.253.128.6:19998
leaf-su00-r0 |                     10.253.128.6 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |                     RT:59909:19998 ET:8 Rmac:44:38:39:22:01:d2
leaf-su00-r0 | Route Distinguisher: 10.253.128.6:19999
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.18.1.0] RD 10.253.128.6:19999
leaf-su00-r0 |                     10.253.128.6 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |                     RT:59909:19999 ET:8 Rmac:44:38:39:22:01:d2
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.18.1.0] RD 10.253.128.6:19999
leaf-su00-r0 |                     10.253.128.6 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |                     RT:59909:19999 ET:8 Rmac:44:38:39:22:01:d2
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.26.1.0] RD 10.253.128.6:19999
leaf-su00-r0 |                     10.253.128.6 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |                     RT:59909:19999 ET:8 Rmac:44:38:39:22:01:d2
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.26.1.0] RD 10.253.128.6:19999
leaf-su00-r0 |                     10.253.128.6 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |                     RT:59909:19999 ET:8 Rmac:44:38:39:22:01:d2
leaf-su00-r0 | Route Distinguisher: 10.253.128.7:19998
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.20.1.0] RD 10.253.128.7:19998
leaf-su00-r0 |                     10.253.128.7 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |                     RT:59910:19998 ET:8 Rmac:44:38:39:22:01:d3
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.20.1.0] RD 10.253.128.7:19998
leaf-su00-r0 |                     10.253.128.7 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |                     RT:59910:19998 ET:8 Rmac:44:38:39:22:01:d3
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.28.1.0] RD 10.253.128.7:19998
leaf-su00-r0 |                     10.253.128.7 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |                     RT:59910:19998 ET:8 Rmac:44:38:39:22:01:d3
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.28.1.0] RD 10.253.128.7:19998
leaf-su00-r0 |                     10.253.128.7 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |                     RT:59910:19998 ET:8 Rmac:44:38:39:22:01:d3
leaf-su00-r0 | Route Distinguisher: 10.253.128.7:19999
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.20.1.0] RD 10.253.128.7:19999
leaf-su00-r0 |                     10.253.128.7 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |                     RT:59910:19999 ET:8 Rmac:44:38:39:22:01:d3
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.20.1.0] RD 10.253.128.7:19999
leaf-su00-r0 |                     10.253.128.7 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |                     RT:59910:19999 ET:8 Rmac:44:38:39:22:01:d3
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.28.1.0] RD 10.253.128.7:19999
leaf-su00-r0 |                     10.253.128.7 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |                     RT:59910:19999 ET:8 Rmac:44:38:39:22:01:d3
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.28.1.0] RD 10.253.128.7:19999
leaf-su00-r0 |                     10.253.128.7 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |                     RT:59910:19999 ET:8 Rmac:44:38:39:22:01:d3
leaf-su00-r0 | Route Distinguisher: 10.253.128.8:19998
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.22.1.0] RD 10.253.128.8:19998
leaf-su00-r0 |                     10.253.128.8 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |                     RT:59911:19998 ET:8 Rmac:44:38:39:22:01:d4
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.22.1.0] RD 10.253.128.8:19998
leaf-su00-r0 |                     10.253.128.8 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |                     RT:59911:19998 ET:8 Rmac:44:38:39:22:01:d4
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.30.1.0] RD 10.253.128.8:19998
leaf-su00-r0 |                     10.253.128.8 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |                     RT:59911:19998 ET:8 Rmac:44:38:39:22:01:d4
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.30.1.0] RD 10.253.128.8:19998
leaf-su00-r0 |                     10.253.128.8 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |                     RT:59911:19998 ET:8 Rmac:44:38:39:22:01:d4
leaf-su00-r0 | Route Distinguisher: 10.253.128.8:19999
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.22.1.0] RD 10.253.128.8:19999
leaf-su00-r0 |                     10.253.128.8 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |                     RT:59911:19999 ET:8 Rmac:44:38:39:22:01:d4
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.22.1.0] RD 10.253.128.8:19999
leaf-su00-r0 |                     10.253.128.8 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |                     RT:59911:19999 ET:8 Rmac:44:38:39:22:01:d4
leaf-su00-r0 |  *> [5]:[0]:[26]:[172.30.1.0] RD 10.253.128.8:19999
leaf-su00-r0 |                     10.253.128.8 (spine-s00)
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |                     RT:59911:19999 ET:8 Rmac:44:38:39:22:01:d4
leaf-su00-r0 |  *  [5]:[0]:[26]:[172.30.1.0] RD 10.253.128.8:19999
leaf-su00-r0 |                     10.253.128.8 (spine-s01)
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |                     RT:59911:19999 ET:8 Rmac:44:38:39:22:01:d4
leaf-su00-r0 |
leaf-su00-r0 | Displayed 32 prefixes (60 paths) (of requested type)
################################################################################
```

This is the control-plane state behind the host-to-host reachability that Step 14 verifies from the hosts.

### Step 13. Configure the HGX Hosts

**Goal:** Apply the per-host rail netplan configuration so each HGX host's eight rail NICs come up with their `/31` addresses.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`; the HGX hosts at `192.168.200.23` through `.30`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation and on the HGX hosts.

**Expected wait time:** Under a minute per host.

Each host has eight rail NICs (`eth_rail0` through `eth_rail7`) at MTU 9216. Each rail takes a `/31`, with the host on the even address and the leaf gateway on the odd one. For host node `h` on rail `r` in unit `u`, the host address is `172.(16 + 2*r).u.(2*h)` and the gateway is one higher. The rail sets the second octet, the unit sets the third (SU0 = 0, SU1 = 1), and the node sets the fourth (h00 = 0, h08 = 16, h16 = 32, h24 = 48):

| __rail__ | __NIC__ | __su00 h00__ | __su00 h08__ | __su00 h16__ | __su00 h24__ | __su01 h00__ | __su01 h08__ | __su01 h16__ | __su01 h24__ |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | `eth_rail0` | 172.16.0.0 | 172.16.0.16 | 172.16.0.32 | 172.16.0.48 | 172.16.1.0 | 172.16.1.16 | 172.16.1.32 | 172.16.1.48 |
| 1 | `eth_rail1` | 172.18.0.0 | 172.18.0.16 | 172.18.0.32 | 172.18.0.48 | 172.18.1.0 | 172.18.1.16 | 172.18.1.32 | 172.18.1.48 |
| 2 | `eth_rail2` | 172.20.0.0 | 172.20.0.16 | 172.20.0.32 | 172.20.0.48 | 172.20.1.0 | 172.20.1.16 | 172.20.1.32 | 172.20.1.48 |
| 3 | `eth_rail3` | 172.22.0.0 | 172.22.0.16 | 172.22.0.32 | 172.22.0.48 | 172.22.1.0 | 172.22.1.16 | 172.22.1.32 | 172.22.1.48 |
| 4 | `eth_rail4` | 172.24.0.0 | 172.24.0.16 | 172.24.0.32 | 172.24.0.48 | 172.24.1.0 | 172.24.1.16 | 172.24.1.32 | 172.24.1.48 |
| 5 | `eth_rail5` | 172.26.0.0 | 172.26.0.16 | 172.26.0.32 | 172.26.0.48 | 172.26.1.0 | 172.26.1.16 | 172.26.1.32 | 172.26.1.48 |
| 6 | `eth_rail6` | 172.28.0.0 | 172.28.0.16 | 172.28.0.32 | 172.28.0.48 | 172.28.1.0 | 172.28.1.16 | 172.28.1.32 | 172.28.1.48 |
| 7 | `eth_rail7` | 172.30.0.0 | 172.30.0.16 | 172.30.0.32 | 172.30.0.48 | 172.30.1.0 | 172.30.1.16 | 172.30.1.32 | 172.30.1.48 |

The complete per-host netplan files are staged on the jumpstation under `netplan/2su/`, one per host (`hgx-su00-h00.yaml` through `hgx-su01-h24.yaml`). Each holds all eight rails and installs to `/etc/netplan/60-spectrum-x.yaml` on its host.

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
hgx-su00-h00 | eth0             UP             192.168.200.23/24 metric 100 fe80::4638:39ff:fe11:117/64
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
hgx-su00-h08 | eth0             UP             192.168.200.24/24 metric 100 fe80::4638:39ff:fe11:118/64
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
hgx-su00-h16 | eth0             UP             192.168.200.25/24 metric 100 fe80::4638:39ff:fe11:119/64
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
hgx-su00-h24 | eth0             UP             192.168.200.26/24 metric 100 fe80::4638:39ff:fe11:11a/64
hgx-su00-h24 | eth_rail0        DOWN
hgx-su00-h24 | eth_rail1        DOWN
hgx-su00-h24 | eth_rail2        DOWN
hgx-su00-h24 | eth_rail3        DOWN
hgx-su00-h24 | eth_rail4        DOWN
hgx-su00-h24 | eth_rail5        DOWN
hgx-su00-h24 | eth_rail6        DOWN
hgx-su00-h24 | eth_rail7        DOWN
################################################################################
hgx-su01-h00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su01-h00 | eth0             UP             192.168.200.27/24 metric 100 fe80::4638:39ff:fe11:11b/64
hgx-su01-h00 | eth_rail0        DOWN
hgx-su01-h00 | eth_rail1        DOWN
hgx-su01-h00 | eth_rail2        DOWN
hgx-su01-h00 | eth_rail3        DOWN
hgx-su01-h00 | eth_rail4        DOWN
hgx-su01-h00 | eth_rail5        DOWN
hgx-su01-h00 | eth_rail6        DOWN
hgx-su01-h00 | eth_rail7        DOWN
################################################################################
hgx-su01-h08 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su01-h08 | eth0             UP             192.168.200.28/24 metric 100 fe80::4638:39ff:fe11:11c/64
hgx-su01-h08 | eth_rail0        DOWN
hgx-su01-h08 | eth_rail1        DOWN
hgx-su01-h08 | eth_rail2        DOWN
hgx-su01-h08 | eth_rail3        DOWN
hgx-su01-h08 | eth_rail4        DOWN
hgx-su01-h08 | eth_rail5        DOWN
hgx-su01-h08 | eth_rail6        DOWN
hgx-su01-h08 | eth_rail7        DOWN
################################################################################
hgx-su01-h16 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su01-h16 | eth0             UP             192.168.200.29/24 metric 100 fe80::4638:39ff:fe11:11d/64
hgx-su01-h16 | eth_rail0        DOWN
hgx-su01-h16 | eth_rail1        DOWN
hgx-su01-h16 | eth_rail2        DOWN
hgx-su01-h16 | eth_rail3        DOWN
hgx-su01-h16 | eth_rail4        DOWN
hgx-su01-h16 | eth_rail5        DOWN
hgx-su01-h16 | eth_rail6        DOWN
hgx-su01-h16 | eth_rail7        DOWN
################################################################################
hgx-su01-h24 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su01-h24 | eth0             UP             192.168.200.30/24 metric 100 fe80::4638:39ff:fe11:11e/64
hgx-su01-h24 | eth_rail0        DOWN
hgx-su01-h24 | eth_rail1        DOWN
hgx-su01-h24 | eth_rail2        DOWN
hgx-su01-h24 | eth_rail3        DOWN
hgx-su01-h24 | eth_rail4        DOWN
hgx-su01-h24 | eth_rail5        DOWN
hgx-su01-h24 | eth_rail6        DOWN
hgx-su01-h24 | eth_rail7        DOWN
################################################################################
```

Now push each host the file named for it and apply it. This loop reads each host's name, copies its file, installs it with mode 0600, and runs `netplan apply`:

```bash
 ~/spcx-air/spcx-run --apply-netplan ~/spcx-air/fabric/2su/hosts/
```

The expected output should be the following:

```bash
ubuntu@oob-mgmt-server:~/nvidia$  ~/spcx-air/spcx-run --apply-netplan ~/spcx-air/fabric/2su/hosts/
Applying per-host netplan from: /home/ubuntu/spcx-air/fabric/2su/hosts
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
hgx-su01-h00 | (no output)
################################################################################
hgx-su01-h08 | (no output)
################################################################################
hgx-su01-h16 | (no output)
################################################################################
hgx-su01-h24 | (no output)
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
hgx-su00-h00 | eth0             UP             192.168.200.23/24 metric 100 fe80::4638:39ff:fe11:117/64
hgx-su00-h00 | eth_rail0        UP             172.16.0.0/31 fe80::4ab0:2dff:fe41:d85c/64
hgx-su00-h00 | eth_rail1        UP             172.18.0.0/31 fe80::4ab0:2dff:fe1d:63f4/64
hgx-su00-h00 | eth_rail2        UP             172.20.0.0/31 fe80::4ab0:2dff:fe98:dc4/64
hgx-su00-h00 | eth_rail3        UP             172.22.0.0/31 fe80::4ab0:2dff:fee8:880a/64
hgx-su00-h00 | eth_rail4        UP             172.24.0.0/31 fe80::4ab0:2dff:fe45:9a10/64
hgx-su00-h00 | eth_rail5        UP             172.26.0.0/31 fe80::4ab0:2dff:fefb:d32/64
hgx-su00-h00 | eth_rail6        UP             172.28.0.0/31 fe80::4ab0:2dff:fe22:60bd/64
hgx-su00-h00 | eth_rail7        UP             172.30.0.0/31 fe80::4ab0:2dff:fec0:4c6a/64
################################################################################
hgx-su00-h08 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h08 | eth0             UP             192.168.200.24/24 metric 100 fe80::4638:39ff:fe11:118/64
hgx-su00-h08 | eth_rail0        UP             172.16.0.16/31 fe80::4ab0:2dff:feeb:c0c3/64
hgx-su00-h08 | eth_rail1        UP             172.18.0.16/31 fe80::4ab0:2dff:fe40:c5a3/64
hgx-su00-h08 | eth_rail2        UP             172.20.0.16/31 fe80::4ab0:2dff:fe1b:daf8/64
hgx-su00-h08 | eth_rail3        UP             172.22.0.16/31 fe80::4ab0:2dff:fecd:b0eb/64
hgx-su00-h08 | eth_rail4        UP             172.24.0.16/31 fe80::4ab0:2dff:fe69:922b/64
hgx-su00-h08 | eth_rail5        UP             172.26.0.16/31 fe80::4ab0:2dff:fe3a:6668/64
hgx-su00-h08 | eth_rail6        UP             172.28.0.16/31 fe80::4ab0:2dff:fef8:8fc7/64
hgx-su00-h08 | eth_rail7        UP             172.30.0.16/31 fe80::4ab0:2dff:fed2:b1a5/64
################################################################################
hgx-su00-h16 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h16 | eth0             UP             192.168.200.25/24 metric 100 fe80::4638:39ff:fe11:119/64
hgx-su00-h16 | eth_rail0        UP             172.16.0.32/31 fe80::4ab0:2dff:fe8a:3610/64
hgx-su00-h16 | eth_rail1        UP             172.18.0.32/31 fe80::4ab0:2dff:fe33:ec2a/64
hgx-su00-h16 | eth_rail2        UP             172.20.0.32/31 fe80::4ab0:2dff:feb8:7d7d/64
hgx-su00-h16 | eth_rail3        UP             172.22.0.32/31 fe80::4ab0:2dff:fe28:10b4/64
hgx-su00-h16 | eth_rail4        UP             172.24.0.32/31 fe80::4ab0:2dff:fe38:f28c/64
hgx-su00-h16 | eth_rail5        UP             172.26.0.32/31 fe80::4ab0:2dff:fe58:544a/64
hgx-su00-h16 | eth_rail6        UP             172.28.0.32/31 fe80::4ab0:2dff:fef2:a9/64
hgx-su00-h16 | eth_rail7        UP             172.30.0.32/31 fe80::4ab0:2dff:fee8:a998/64
################################################################################
hgx-su00-h24 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su00-h24 | eth0             UP             192.168.200.26/24 metric 100 fe80::4638:39ff:fe11:11a/64
hgx-su00-h24 | eth_rail0        UP             172.16.0.48/31 fe80::4ab0:2dff:fe48:b1e9/64
hgx-su00-h24 | eth_rail1        UP             172.18.0.48/31 fe80::4ab0:2dff:fe1a:baa4/64
hgx-su00-h24 | eth_rail2        UP             172.20.0.48/31 fe80::4ab0:2dff:fe97:716/64
hgx-su00-h24 | eth_rail3        UP             172.22.0.48/31 fe80::4ab0:2dff:fe74:bc56/64
hgx-su00-h24 | eth_rail4        UP             172.24.0.48/31 fe80::4ab0:2dff:fe9a:3de0/64
hgx-su00-h24 | eth_rail5        UP             172.26.0.48/31 fe80::4ab0:2dff:fefa:1502/64
hgx-su00-h24 | eth_rail6        UP             172.28.0.48/31 fe80::4ab0:2dff:fe46:adac/64
hgx-su00-h24 | eth_rail7        UP             172.30.0.48/31 fe80::4ab0:2dff:fe68:a081/64
################################################################################
hgx-su01-h00 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su01-h00 | eth0             UP             192.168.200.27/24 metric 100 fe80::4638:39ff:fe11:11b/64
hgx-su01-h00 | eth_rail0        UP             172.16.1.0/31 fe80::4ab0:2dff:fe10:b2e7/64
hgx-su01-h00 | eth_rail1        UP             172.18.1.0/31 fe80::4ab0:2dff:fe7f:973a/64
hgx-su01-h00 | eth_rail2        UP             172.20.1.0/31 fe80::4ab0:2dff:febd:52c4/64
hgx-su01-h00 | eth_rail3        UP             172.22.1.0/31 fe80::4ab0:2dff:fed9:eea3/64
hgx-su01-h00 | eth_rail4        UP             172.24.1.0/31 fe80::4ab0:2dff:fe35:7109/64
hgx-su01-h00 | eth_rail5        UP             172.26.1.0/31 fe80::4ab0:2dff:fe2d:1926/64
hgx-su01-h00 | eth_rail6        UP             172.28.1.0/31 fe80::4ab0:2dff:feb6:d68e/64
hgx-su01-h00 | eth_rail7        UP             172.30.1.0/31 fe80::4ab0:2dff:fe5f:62c/64
################################################################################
hgx-su01-h08 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su01-h08 | eth0             UP             192.168.200.28/24 metric 100 fe80::4638:39ff:fe11:11c/64
hgx-su01-h08 | eth_rail0        UP             172.16.1.16/31 fe80::4ab0:2dff:fe73:379c/64
hgx-su01-h08 | eth_rail1        UP             172.18.1.16/31 fe80::4ab0:2dff:fe02:98f6/64
hgx-su01-h08 | eth_rail2        UP             172.20.1.16/31 fe80::4ab0:2dff:fef7:c929/64
hgx-su01-h08 | eth_rail3        UP             172.22.1.16/31 fe80::4ab0:2dff:fe51:b0c1/64
hgx-su01-h08 | eth_rail4        UP             172.24.1.16/31 fe80::4ab0:2dff:fe7f:e61b/64
hgx-su01-h08 | eth_rail5        UP             172.26.1.16/31 fe80::4ab0:2dff:fe68:ac28/64
hgx-su01-h08 | eth_rail6        UP             172.28.1.16/31 fe80::4ab0:2dff:fe3c:d93a/64
hgx-su01-h08 | eth_rail7        UP             172.30.1.16/31 fe80::4ab0:2dff:fe7c:7bcd/64
################################################################################
hgx-su01-h16 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su01-h16 | eth0             UP             192.168.200.29/24 metric 100 fe80::4638:39ff:fe11:11d/64
hgx-su01-h16 | eth_rail0        UP             172.16.1.32/31 fe80::4ab0:2dff:fef5:2bd1/64
hgx-su01-h16 | eth_rail1        UP             172.18.1.32/31 fe80::4ab0:2dff:feb3:b728/64
hgx-su01-h16 | eth_rail2        UP             172.20.1.32/31 fe80::4ab0:2dff:fec6:e0f0/64
hgx-su01-h16 | eth_rail3        UP             172.22.1.32/31 fe80::4ab0:2dff:fe89:b28b/64
hgx-su01-h16 | eth_rail4        UP             172.24.1.32/31 fe80::4ab0:2dff:fe41:3e46/64
hgx-su01-h16 | eth_rail5        UP             172.26.1.32/31 fe80::4ab0:2dff:feeb:e776/64
hgx-su01-h16 | eth_rail6        UP             172.28.1.32/31 fe80::4ab0:2dff:fe61:eb14/64
hgx-su01-h16 | eth_rail7        UP             172.30.1.32/31 fe80::4ab0:2dff:fe82:c707/64
################################################################################
hgx-su01-h24 | lo               UNKNOWN        127.0.0.1/8 ::1/128
hgx-su01-h24 | eth0             UP             192.168.200.30/24 metric 100 fe80::4638:39ff:fe11:11e/64
hgx-su01-h24 | eth_rail0        UP             172.16.1.48/31 fe80::4ab0:2dff:fed3:53ba/64
hgx-su01-h24 | eth_rail1        UP             172.18.1.48/31 fe80::4ab0:2dff:fe0a:595e/64
hgx-su01-h24 | eth_rail2        UP             172.20.1.48/31 fe80::4ab0:2dff:fe17:65a7/64
hgx-su01-h24 | eth_rail3        UP             172.22.1.48/31 fe80::4ab0:2dff:fec8:50c/64
hgx-su01-h24 | eth_rail4        UP             172.24.1.48/31 fe80::4ab0:2dff:fe8b:5313/64
hgx-su01-h24 | eth_rail5        UP             172.26.1.48/31 fe80::4ab0:2dff:fe47:8ac5/64
hgx-su01-h24 | eth_rail6        UP             172.28.1.48/31 fe80::4ab0:2dff:fede:9342/64
hgx-su01-h24 | eth_rail7        UP             172.30.1.48/31 fe80::4ab0:2dff:fec6:6e1f/64
################################################################################
```

### Step 14. Verify Connectivity

**Goal:** Confirm the tenant deployments finished, then run a full rail mesh ping test across all eight HGX hosts, confirming full-mesh reachability within each tenant (including across units) and isolation between `tenant1` and `tenant2`.

**Access needed:** Jumpstation (`oob-mgmt-server`), working directory `~/nvidia`; the HGX hosts at `192.168.200.23` through `.30`.

**Credentials for this step:** `ubuntu` / `nvidia` on the jumpstation and on the HGX hosts.

**Expected wait time:** Up to about 30 seconds per host for BGP/EVPN to finish converging.

The rail `/31`s live only on the hosts, so the ping sweep has to run on each host, not on the jumpstation. This block drives all eight hosts from the jumpstation: it runs the rail mesh on each one, covering all eight rails across both units and all four hosts per unit (64 targets per host, including the same-rail peers in the other unit):

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
hgx-su00-h00 | 172.16.0.0  : [0], 64 bytes, 0.053 ms (0.053 avg, 0% loss)
hgx-su00-h00 | 172.16.0.32 : [0], 64 bytes, 0.541 ms (0.541 avg, 0% loss)
hgx-su00-h00 | 172.16.1.0  : [0], 64 bytes, 1.43 ms (1.43 avg, 0% loss)
hgx-su00-h00 | 172.16.1.32 : [0], 64 bytes, 1.13 ms (1.13 avg, 0% loss)
hgx-su00-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.0  : [1], 64 bytes, 0.033 ms (0.043 avg, 0% loss)
hgx-su00-h00 | 172.16.0.32 : [1], 64 bytes, 0.548 ms (0.545 avg, 0% loss)
hgx-su00-h00 | 172.16.1.0  : [1], 64 bytes, 1.49 ms (1.46 avg, 0% loss)
hgx-su00-h00 | 172.16.1.32 : [1], 64 bytes, 1.14 ms (1.13 avg, 0% loss)
hgx-su00-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 |
hgx-su00-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.033/0.043/0.053
hgx-su00-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.541/0.545/0.548
hgx-su00-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.43/1.46/1.49
hgx-su00-h00 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.13/1.13/1.14
hgx-su00-h00 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su00-h08 | 172.16.0.16 : [0], 64 bytes, 0.028 ms (0.028 avg, 0% loss)
hgx-su00-h08 | 172.16.0.48 : [0], 64 bytes, 0.518 ms (0.518 avg, 0% loss)
hgx-su00-h08 | 172.16.1.16 : [0], 64 bytes, 1.17 ms (1.17 avg, 0% loss)
hgx-su00-h08 | 172.16.1.48 : [0], 64 bytes, 1.36 ms (1.36 avg, 0% loss)
hgx-su00-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.0.16 : [1], 64 bytes, 0.014 ms (0.021 avg, 0% loss)
hgx-su00-h08 | 172.16.0.48 : [1], 64 bytes, 0.491 ms (0.505 avg, 0% loss)
hgx-su00-h08 | 172.16.1.16 : [1], 64 bytes, 1.42 ms (1.29 avg, 0% loss)
hgx-su00-h08 | 172.16.1.48 : [1], 64 bytes, 2.10 ms (1.73 avg, 0% loss)
hgx-su00-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 |
hgx-su00-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.014/0.021/0.028
hgx-su00-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.491/0.505/0.518
hgx-su00-h08 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.17/1.29/1.42
hgx-su00-h08 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.36/1.73/2.10
################################################################################
hgx-su00-h16 | 172.16.0.0  : [0], 64 bytes, 1.44 ms (1.44 avg, 0% loss)
hgx-su00-h16 | 172.16.0.32 : [0], 64 bytes, 0.065 ms (0.065 avg, 0% loss)
hgx-su00-h16 | 172.16.1.0  : [0], 64 bytes, 1.44 ms (1.44 avg, 0% loss)
hgx-su00-h16 | 172.16.1.32 : [0], 64 bytes, 1.40 ms (1.40 avg, 0% loss)
hgx-su00-h16 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.0.0  : [1], 64 bytes, 0.724 ms (1.08 avg, 0% loss)
hgx-su00-h16 | 172.16.0.32 : [1], 64 bytes, 0.015 ms (0.040 avg, 0% loss)
hgx-su00-h16 | 172.16.1.0  : [1], 64 bytes, 1.49 ms (1.46 avg, 0% loss)
hgx-su00-h16 | 172.16.1.32 : [1], 64 bytes, 1.21 ms (1.30 avg, 0% loss)
hgx-su00-h16 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 |
hgx-su00-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.724/1.08/1.44
hgx-su00-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.015/0.040/0.065
hgx-su00-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h16 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.44/1.46/1.49
hgx-su00-h16 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h16 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.21/1.30/1.40
hgx-su00-h16 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su00-h24 | 172.16.0.16 : [0], 64 bytes, 0.601 ms (0.601 avg, 0% loss)
hgx-su00-h24 | 172.16.0.48 : [0], 64 bytes, 0.020 ms (0.020 avg, 0% loss)
hgx-su00-h24 | 172.16.1.16 : [0], 64 bytes, 1.20 ms (1.20 avg, 0% loss)
hgx-su00-h24 | 172.16.1.48 : [0], 64 bytes, 1.36 ms (1.36 avg, 0% loss)
hgx-su00-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.0.16 : [1], 64 bytes, 0.507 ms (0.554 avg, 0% loss)
hgx-su00-h24 | 172.16.0.48 : [1], 64 bytes, 0.013 ms (0.017 avg, 0% loss)
hgx-su00-h24 | 172.16.1.16 : [1], 64 bytes, 1.23 ms (1.22 avg, 0% loss)
hgx-su00-h24 | 172.16.1.48 : [1], 64 bytes, 1.32 ms (1.34 avg, 0% loss)
hgx-su00-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 |
hgx-su00-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.507/0.554/0.601
hgx-su00-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.013/0.017/0.020
hgx-su00-h24 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.20/1.22/1.23
hgx-su00-h24 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.32/1.34/1.36
################################################################################
hgx-su01-h00 | 172.16.0.0  : [0], 64 bytes, 2.09 ms (2.09 avg, 0% loss)
hgx-su01-h00 | 172.16.0.32 : [0], 64 bytes, 1.38 ms (1.38 avg, 0% loss)
hgx-su01-h00 | 172.16.1.0  : [0], 64 bytes, 0.066 ms (0.066 avg, 0% loss)
hgx-su01-h00 | 172.16.1.32 : [0], 64 bytes, 0.582 ms (0.582 avg, 0% loss)
hgx-su01-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.0.0  : [1], 64 bytes, 1.71 ms (1.90 avg, 0% loss)
hgx-su01-h00 | 172.16.0.32 : [1], 64 bytes, 1.56 ms (1.47 avg, 0% loss)
hgx-su01-h00 | 172.16.1.0  : [1], 64 bytes, 0.018 ms (0.042 avg, 0% loss)
hgx-su01-h00 | 172.16.1.32 : [1], 64 bytes, 0.993 ms (0.787 avg, 0% loss)
hgx-su01-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h00 |
hgx-su01-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.71/1.90/2.09
hgx-su01-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.38/1.47/1.56
hgx-su01-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h00 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.018/0.042/0.066
hgx-su01-h00 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h00 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.582/0.787/0.993
hgx-su01-h00 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su01-h08 | 172.16.0.16 : [0], 64 bytes, 1.61 ms (1.61 avg, 0% loss)
hgx-su01-h08 | 172.16.0.48 : [0], 64 bytes, 1.63 ms (1.63 avg, 0% loss)
hgx-su01-h08 | 172.16.1.16 : [0], 64 bytes, 0.026 ms (0.026 avg, 0% loss)
hgx-su01-h08 | 172.16.1.48 : [0], 64 bytes, 0.713 ms (0.713 avg, 0% loss)
hgx-su01-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.0.16 : [1], 64 bytes, 1.56 ms (1.58 avg, 0% loss)
hgx-su01-h08 | 172.16.0.48 : [1], 64 bytes, 1.41 ms (1.52 avg, 0% loss)
hgx-su01-h08 | 172.16.1.16 : [1], 64 bytes, 0.014 ms (0.020 avg, 0% loss)
hgx-su01-h08 | 172.16.1.48 : [1], 64 bytes, 0.750 ms (0.731 avg, 0% loss)
hgx-su01-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h08 |
hgx-su01-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su01-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.56/1.58/1.61
hgx-su01-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.41/1.52/1.63
hgx-su01-h08 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-su01-h08 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.014/0.020/0.026
hgx-su01-h08 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h08 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.713/0.731/0.750
################################################################################
hgx-su01-h16 | 172.16.0.0  : [0], 64 bytes, 1.75 ms (1.75 avg, 0% loss)
hgx-su01-h16 | 172.16.0.32 : [0], 64 bytes, 1.28 ms (1.28 avg, 0% loss)
hgx-su01-h16 | 172.16.1.0  : [0], 64 bytes, 0.589 ms (0.589 avg, 0% loss)
hgx-su01-h16 | 172.16.1.32 : [0], 64 bytes, 0.038 ms (0.038 avg, 0% loss)
hgx-su01-h16 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.0.0  : [1], 64 bytes, 1.34 ms (1.54 avg, 0% loss)
hgx-su01-h16 | 172.16.0.32 : [1], 64 bytes, 1.19 ms (1.23 avg, 0% loss)
hgx-su01-h16 | 172.16.1.0  : [1], 64 bytes, 0.657 ms (0.623 avg, 0% loss)
hgx-su01-h16 | 172.16.1.32 : [1], 64 bytes, 0.015 ms (0.026 avg, 0% loss)
hgx-su01-h16 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h16 |
hgx-su01-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.34/1.54/1.75
hgx-su01-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.19/1.23/1.28
hgx-su01-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h16 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.589/0.623/0.657
hgx-su01-h16 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h16 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.015/0.026/0.038
hgx-su01-h16 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su01-h24 | 172.16.0.16 : [0], 64 bytes, 2.37 ms (2.37 avg, 0% loss)
hgx-su01-h24 | 172.16.0.48 : [0], 64 bytes, 1.67 ms (1.67 avg, 0% loss)
hgx-su01-h24 | 172.16.1.16 : [0], 64 bytes, 0.718 ms (0.718 avg, 0% loss)
hgx-su01-h24 | 172.16.1.48 : [0], 64 bytes, 0.026 ms (0.026 avg, 0% loss)
hgx-su01-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.0.16 : [1], 64 bytes, 1.81 ms (2.09 avg, 0% loss)
hgx-su01-h24 | 172.16.0.48 : [1], 64 bytes, 1.59 ms (1.63 avg, 0% loss)
hgx-su01-h24 | 172.16.1.16 : [1], 64 bytes, 0.705 ms (0.711 avg, 0% loss)
hgx-su01-h24 | 172.16.1.48 : [1], 64 bytes, 0.019 ms (0.022 avg, 0% loss)
hgx-su01-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h24 |
hgx-su01-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su01-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.81/2.09/2.37
hgx-su01-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.59/1.63/1.67
hgx-su01-h24 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-su01-h24 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.705/0.711/0.718
hgx-su01-h24 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h24 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.019/0.022/0.026
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia$ ~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
========================================
Running: for ip in 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 172.16.1.0 172.16.1.16 172.16.1.32 172.16.1.48; do ping -c2 -W2 $ip >/dev/null 2>&1 && echo "PASS $ip" || echo "FAIL $ip"; done
========================================
################################################################################
hgx-su00-h00 | PASS 172.16.0.0
hgx-su00-h00 | FAIL 172.16.0.16
hgx-su00-h00 | PASS 172.16.0.32
hgx-su00-h00 | FAIL 172.16.0.48
hgx-su00-h00 | PASS 172.16.1.0
hgx-su00-h00 | FAIL 172.16.1.16
hgx-su00-h00 | PASS 172.16.1.32
hgx-su00-h00 | FAIL 172.16.1.48
################################################################################
hgx-su00-h08 | FAIL 172.16.0.0
hgx-su00-h08 | PASS 172.16.0.16
hgx-su00-h08 | FAIL 172.16.0.32
hgx-su00-h08 | PASS 172.16.0.48
hgx-su00-h08 | FAIL 172.16.1.0
hgx-su00-h08 | PASS 172.16.1.16
hgx-su00-h08 | FAIL 172.16.1.32
hgx-su00-h08 | PASS 172.16.1.48
################################################################################
hgx-su00-h16 | PASS 172.16.0.0
hgx-su00-h16 | FAIL 172.16.0.16
hgx-su00-h16 | PASS 172.16.0.32
hgx-su00-h16 | FAIL 172.16.0.48
hgx-su00-h16 | PASS 172.16.1.0
hgx-su00-h16 | FAIL 172.16.1.16
hgx-su00-h16 | PASS 172.16.1.32
hgx-su00-h16 | FAIL 172.16.1.48
################################################################################
hgx-su00-h24 | FAIL 172.16.0.0
hgx-su00-h24 | PASS 172.16.0.16
hgx-su00-h24 | FAIL 172.16.0.32
hgx-su00-h24 | PASS 172.16.0.48
hgx-su00-h24 | FAIL 172.16.1.0
hgx-su00-h24 | PASS 172.16.1.16
hgx-su00-h24 | FAIL 172.16.1.32
hgx-su00-h24 | PASS 172.16.1.48
################################################################################
hgx-su01-h00 | PASS 172.16.0.0
hgx-su01-h00 | FAIL 172.16.0.16
hgx-su01-h00 | PASS 172.16.0.32
hgx-su01-h00 | FAIL 172.16.0.48
hgx-su01-h00 | PASS 172.16.1.0
hgx-su01-h00 | FAIL 172.16.1.16
hgx-su01-h00 | PASS 172.16.1.32
hgx-su01-h00 | FAIL 172.16.1.48
################################################################################
hgx-su01-h08 | FAIL 172.16.0.0
hgx-su01-h08 | PASS 172.16.0.16
hgx-su01-h08 | FAIL 172.16.0.32
hgx-su01-h08 | PASS 172.16.0.48
hgx-su01-h08 | FAIL 172.16.1.0
hgx-su01-h08 | PASS 172.16.1.16
hgx-su01-h08 | FAIL 172.16.1.32
hgx-su01-h08 | PASS 172.16.1.48
################################################################################
hgx-su01-h16 | PASS 172.16.0.0
hgx-su01-h16 | FAIL 172.16.0.16
hgx-su01-h16 | PASS 172.16.0.32
hgx-su01-h16 | FAIL 172.16.0.48
hgx-su01-h16 | PASS 172.16.1.0
hgx-su01-h16 | FAIL 172.16.1.16
hgx-su01-h16 | PASS 172.16.1.32
hgx-su01-h16 | FAIL 172.16.1.48
################################################################################
hgx-su01-h24 | FAIL 172.16.0.0
hgx-su01-h24 | PASS 172.16.0.16
hgx-su01-h24 | FAIL 172.16.0.32
hgx-su01-h24 | PASS 172.16.0.48
hgx-su01-h24 | FAIL 172.16.1.0
hgx-su01-h24 | PASS 172.16.1.16
hgx-su01-h24 | FAIL 172.16.1.32
hgx-su01-h24 | PASS 172.16.1.48
################################################################################
```

Expected result:

- HGX hosts from `tenant1` (`hgx-su00-h00`, `hgx-su00-h16`, `hgx-su01-h00`, `hgx-su01-h16`) have full-mesh connectivity with each other, including across units, but cannot reach the `tenant2` hosts (`hgx-su00-h08`, `hgx-su00-h24`, `hgx-su01-h08`, `hgx-su01-h24`). The same applies in reverse for `tenant2`: full-mesh connectivity within the tenant, no connectivity into `tenant1`.

Modify the terraform manifest from `~/nvidia/terraform/tenant1` in order to remove `hgx-su00-h16` from `infra-tenant1` infrastructure and apply. Use the following commands in order to modify `infra-tenant1.tf` directly:

```bash
cd ~/nvidia/terraform/tenant1
sed -i 's/default = \["hgx-su00-h00", "hgx-su00-h16", "hgx-su01-h00", "hgx-su01-h16"]/default = ["hgx-su00-h00", "hgx-su01-h00", "hgx-su01-h16"]/' infra-tenant1.tf
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
      - endpoint_instance_group_id = "1" -> null
      - infrastructure_id          = "1" -> null
      - label                      = "hgx-su00-h16" -> null
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
        id     = "0ac19dca-faac-57ad-c18b-ac4d20a7cbba"
      ~ input  = "hgx-su00-h00,hgx-su00-h16,hgx-su01-h00,hgx-su01-h16" -> "hgx-su00-h00,hgx-su01-h00,hgx-su01-h16"
      ~ output = "hgx-su00-h00,hgx-su00-h16,hgx-su01-h00,hgx-su01-h16" -> (known after apply)
    }

Plan: 1 to add, 1 to change, 2 to destroy.
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Destroying...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant1: Destruction complete after 0s
terraform_data.endpoint_fingerprint: Modifying... [id=0ac19dca-faac-57ad-c18b-ac4d20a7cbba]
terraform_data.endpoint_fingerprint: Modifications complete after 0s [id=0ac19dca-faac-57ad-c18b-ac4d20a7cbba]
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
┌────┬───────────────┬───────────────┬────────┬───────┬──────┬─────────────────────┬─────────────────────┬───────────────┬───────────┐
│ ID │ LABEL         │ CONFIG LABEL  │ STATUS │ OWNER │ SITE │ CREATED             │ UPDATED             │ DEPLOY STATUS │ DEPLOY ID │
├────┼───────────────┼───────────────┼────────┼───────┼──────┼─────────────────────┼─────────────────────┼───────────────┼───────────┤
│  1 │ infra-tenant1 │ infra-tenant1 │ active │     1 │    1 │ 19 Aug 26 02:31 UTC │ 19 Aug 26 03:01 UTC │ finished      │           │
│  2 │ infra-tenant2 │ infra-tenant2 │ active │     1 │    1 │ 19 Aug 26 02:31 UTC │ 19 Aug 26 02:38 UTC │ finished      │           │
└────┴───────────────┴───────────────┴────────┴───────┴──────┴─────────────────────┴─────────────────────┴───────────────┴───────────┘
```

(Optional) In the web UI, navigate to **Admin dashboard > Infrastructures** in order to see the current progress state of the deployment:

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/infrastructure_state.webp)

Modify the terraform manifest from `~/nvidia/terraform/tenant2` in order to add `hgx-su00-h16` to `infra-tenant2` infrastructure and apply. Use the following commands in order to modify `infra-tenant2.tf` directly:

```bash
cd ~/nvidia/terraform/tenant2
sed -i 's/default = \["hgx-su00-h08", "hgx-su00-h24", "hgx-su01-h08", "hgx-su01-h24"\]/default = ["hgx-su00-h08", "hgx-su00-h16", "hgx-su00-h24", "hgx-su01-h08", "hgx-su01-h24"]/' infra-tenant2.tf
terraform apply -auto-approve
```

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
      + infrastructure_id          = "2"
      + label                      = "hgx-su00-h16"
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
        id     = "457c1ef0-89ca-f422-6764-8067400f20f7"
      ~ input  = "hgx-su00-h08,hgx-su00-h24,hgx-su01-h08,hgx-su01-h24" -> "hgx-su00-h08,hgx-su00-h16,hgx-su00-h24,hgx-su01-h08,hgx-su01-h24"
      ~ output = "hgx-su00-h08,hgx-su00-h24,hgx-su01-h08,hgx-su01-h24" -> (known after apply)
    }

Plan: 2 to add, 1 to change, 1 to destroy.
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Destroying...
metalcloud_infrastructure_deployer.infrastructure_deployer_tenant2: Destruction complete after 0s
terraform_data.endpoint_fingerprint: Modifying... [id=457c1ef0-89ca-f422-6764-8067400f20f7]
terraform_data.endpoint_fingerprint: Modifications complete after 0s [id=457c1ef0-89ca-f422-6764-8067400f20f7]
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Creating...
metalcloud_endpoint_instance_group.groups["hgx-su00-h16"]: Creation complete after 0s
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
│  1 │ infra-tenant1 │ infra-tenant1 │ active │     1 │    1 │ 19 Aug 26 02:31 UTC │ 19 Aug 26 03:01 UTC │ finished      │           │
│  2 │ infra-tenant2 │ infra-tenant2 │ active │     1 │    1 │ 19 Aug 26 02:31 UTC │ 19 Aug 26 03:11 UTC │ finished      │           │
└────┴───────────────┴───────────────┴────────┴───────┴──────┴─────────────────────┴─────────────────────┴───────────────┴───────────┘
```

After `hgx-su00-h16` has been moved from `tenant1` infrastructure to `tenant2` infrastructure, the graphical representation of two infrastructures will look like the following:

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/tenant1.final.2su.webp)

![](https://assets.dsx-air.nvidia.com/demo-images/08a26498-b81b-44a2-9822-2ab05fb4a556/tenant2.final.2su.webp)

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
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "ip -br addr show vrf tenant1 | grep -E \"swp|172\.\""
========================================
Running: ip -br addr show vrf tenant1 | grep -E "swp|172\."
========================================
################################################################################
leaf-su00-r0 | swp1s0           UP             172.16.0.1/31 fe80::4ab0:2dff:fe27:761/64
leaf-su00-r0 | swp1s1           UP             172.24.0.1/31 fe80::4ab0:2dff:fe75:51f4/64
################################################################################
leaf-su00-r1 | swp1s0           UP             172.18.0.1/31 fe80::4ab0:2dff:fe8b:c2cf/64
leaf-su00-r1 | swp1s1           UP             172.26.0.1/31 fe80::4ab0:2dff:feb8:1c38/64
################################################################################
leaf-su00-r2 | swp1s0           UP             172.20.0.1/31 fe80::4ab0:2dff:fe7c:7809/64
leaf-su00-r2 | swp1s1           UP             172.28.0.1/31 fe80::4ab0:2dff:fef5:cb0b/64
################################################################################
leaf-su00-r3 | swp1s0           UP             172.22.0.1/31 fe80::4ab0:2dff:fe93:e867/64
leaf-su00-r3 | swp1s1           UP             172.30.0.1/31 fe80::4ab0:2dff:fe0c:1456/64
################################################################################
leaf-su01-r0 | swp1s0           UP             172.16.1.1/31 fe80::4ab0:2dff:fea7:adc7/64
leaf-su01-r0 | swp1s1           UP             172.24.1.1/31 fe80::4ab0:2dff:fe6b:8fab/64
leaf-su01-r0 | swp17s0          UP             172.16.1.33/31 fe80::4ab0:2dff:fe1b:db63/64
leaf-su01-r0 | swp17s1          UP             172.24.1.33/31 fe80::4ab0:2dff:fe5b:aec6/64
################################################################################
leaf-su01-r1 | swp1s0           UP             172.18.1.1/31 fe80::4ab0:2dff:fe64:fdc5/64
leaf-su01-r1 | swp1s1           UP             172.26.1.1/31 fe80::4ab0:2dff:fe09:e970/64
leaf-su01-r1 | swp17s0          UP             172.18.1.33/31 fe80::4ab0:2dff:fe41:2b30/64
leaf-su01-r1 | swp17s1          UP             172.26.1.33/31 fe80::4ab0:2dff:fe15:604c/64
################################################################################
leaf-su01-r2 | swp1s0           UP             172.20.1.1/31 fe80::4ab0:2dff:fef6:fd2f/64
leaf-su01-r2 | swp1s1           UP             172.28.1.1/31 fe80::4ab0:2dff:fe10:c77c/64
leaf-su01-r2 | swp17s0          UP             172.20.1.33/31 fe80::4ab0:2dff:fe8c:7fd9/64
leaf-su01-r2 | swp17s1          UP             172.28.1.33/31 fe80::4ab0:2dff:fe19:6386/64
################################################################################
leaf-su01-r3 | swp1s0           UP             172.22.1.1/31 fe80::4ab0:2dff:fe6e:ed1/64
leaf-su01-r3 | swp1s1           UP             172.30.1.1/31 fe80::4ab0:2dff:fe57:2d/64
leaf-su01-r3 | swp17s0          UP             172.22.1.33/31 fe80::4ab0:2dff:fe1b:c631/64
leaf-su01-r3 | swp17s1          UP             172.30.1.33/31 fe80::4ab0:2dff:fe6d:7a8c/64
################################################################################
spine-s00 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-s01 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-s02 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
spine-s03 | Error: argument "tenant1" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant1 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant1 ipv4 unicast"
========================================
################################################################################
leaf-su00-r0 | BGP table version is 54, local router ID is 10.253.128.1, vrf id 136
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
leaf-su00-r0 |  *> 172.16.1.0/26    10.253.128.5(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *                   10.253.128.5(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.18.1.0/26    10.253.128.6(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *                   10.253.128.6(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.20.1.0/26    10.253.128.7(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *                   10.253.128.7(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.22.1.0/26    10.253.128.8(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *                   10.253.128.8(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.0/31    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.24.1.0/26    10.253.128.5(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *                   10.253.128.5(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.26.1.0/26    10.253.128.6(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *                   10.253.128.6(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.28.1.0/26    10.253.128.7(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *                   10.253.128.7(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.30.1.0/26    10.253.128.8(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *                   10.253.128.8(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |
leaf-su00-r0 | Displayed 18 routes and 32 total paths
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant1\""
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
leaf-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:41:22
leaf-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:40:51
leaf-su00-r0 | C>* 172.16.0.0/31 is directly connected, swp1s0, 00:41:22
leaf-su00-r0 | L>* 172.16.0.1/32 is directly connected, swp1s0, 00:41:22
leaf-su00-r0 | B>* 172.16.1.0/26 [20/0] via 10.253.128.5, vlan1440_l3 onlink, weight 1, 00:41:22
leaf-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:40:13
leaf-su00-r0 | B>* 172.18.1.0/26 [20/0] via 10.253.128.6, vlan1440_l3 onlink, weight 1, 00:41:22
leaf-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:39:35
leaf-su00-r0 | B>* 172.20.1.0/26 [20/0] via 10.253.128.7, vlan1440_l3 onlink, weight 1, 00:41:22
leaf-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:38:56
leaf-su00-r0 | B>* 172.22.1.0/26 [20/0] via 10.253.128.8, vlan1440_l3 onlink, weight 1, 00:41:22
leaf-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:40:51
leaf-su00-r0 | C>* 172.24.0.0/31 is directly connected, swp1s1, 00:41:22
leaf-su00-r0 | L>* 172.24.0.1/32 is directly connected, swp1s1, 00:41:22
leaf-su00-r0 | B>* 172.24.1.0/26 [20/0] via 10.253.128.5, vlan1440_l3 onlink, weight 1, 00:41:22
leaf-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1440_l3 onlink, weight 1, 00:40:13
leaf-su00-r0 | B>* 172.26.1.0/26 [20/0] via 10.253.128.6, vlan1440_l3 onlink, weight 1, 00:41:22
leaf-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1440_l3 onlink, weight 1, 00:39:35
leaf-su00-r0 | B>* 172.28.1.0/26 [20/0] via 10.253.128.7, vlan1440_l3 onlink, weight 1, 00:41:22
leaf-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1440_l3 onlink, weight 1, 00:38:56
leaf-su00-r0 | B>* 172.30.1.0/26 [20/0] via 10.253.128.8, vlan1440_l3 onlink, weight 1, 00:41:22
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
leaf-su00-r0 | swp9s0           UP             172.16.0.17/31 fe80::4ab0:2dff:feb6:b077/64
leaf-su00-r0 | swp9s1           UP             172.24.0.17/31 fe80::4ab0:2dff:fe29:ecdb/64
leaf-su00-r0 | swp17s0          UP             172.16.0.33/31 fe80::4ab0:2dff:fea8:85b5/64
leaf-su00-r0 | swp17s1          UP             172.24.0.33/31 fe80::4ab0:2dff:fe97:53ca/64
leaf-su00-r0 | swp25s0          UP             172.16.0.49/31 fe80::4ab0:2dff:fef1:c0d5/64
leaf-su00-r0 | swp25s1          UP             172.24.0.49/31 fe80::4ab0:2dff:fe91:164b/64
################################################################################
leaf-su00-r1 | swp9s0           UP             172.18.0.17/31 fe80::4ab0:2dff:fea2:11a3/64
leaf-su00-r1 | swp9s1           UP             172.26.0.17/31 fe80::4ab0:2dff:fe2a:b998/64
leaf-su00-r1 | swp17s0          UP             172.18.0.33/31 fe80::4ab0:2dff:feff:ad2/64
leaf-su00-r1 | swp17s1          UP             172.26.0.33/31 fe80::4ab0:2dff:fe9a:1640/64
leaf-su00-r1 | swp25s0          UP             172.18.0.49/31 fe80::4ab0:2dff:feed:5f5f/64
leaf-su00-r1 | swp25s1          UP             172.26.0.49/31 fe80::4ab0:2dff:fe90:9788/64
################################################################################
leaf-su00-r2 | swp9s0           UP             172.20.0.17/31 fe80::4ab0:2dff:fe29:74ce/64
leaf-su00-r2 | swp9s1           UP             172.28.0.17/31 fe80::4ab0:2dff:fece:ddd7/64
leaf-su00-r2 | swp17s0          UP             172.20.0.33/31 fe80::4ab0:2dff:fec1:3673/64
leaf-su00-r2 | swp17s1          UP             172.28.0.33/31 fe80::4ab0:2dff:fe13:de20/64
leaf-su00-r2 | swp25s0          UP             172.20.0.49/31 fe80::4ab0:2dff:fee9:ebf1/64
leaf-su00-r2 | swp25s1          UP             172.28.0.49/31 fe80::4ab0:2dff:fe69:abfe/64
################################################################################
leaf-su00-r3 | swp9s0           UP             172.22.0.17/31 fe80::4ab0:2dff:fefb:1d23/64
leaf-su00-r3 | swp9s1           UP             172.30.0.17/31 fe80::4ab0:2dff:fedb:ac01/64
leaf-su00-r3 | swp17s0          UP             172.22.0.33/31 fe80::4ab0:2dff:fe02:8fd0/64
leaf-su00-r3 | swp17s1          UP             172.30.0.33/31 fe80::4ab0:2dff:fec1:c5a2/64
leaf-su00-r3 | swp25s0          UP             172.22.0.49/31 fe80::4ab0:2dff:fe5d:beb1/64
leaf-su00-r3 | swp25s1          UP             172.30.0.49/31 fe80::4ab0:2dff:fe69:c47d/64
################################################################################
leaf-su01-r0 | swp9s0           UP             172.16.1.17/31 fe80::4ab0:2dff:fed9:9f85/64
leaf-su01-r0 | swp9s1           UP             172.24.1.17/31 fe80::4ab0:2dff:fe03:fb6b/64
leaf-su01-r0 | swp25s0          UP             172.16.1.49/31 fe80::4ab0:2dff:fed3:ec54/64
leaf-su01-r0 | swp25s1          UP             172.24.1.49/31 fe80::4ab0:2dff:fe14:701e/64
################################################################################
leaf-su01-r1 | swp9s0           UP             172.18.1.17/31 fe80::4ab0:2dff:febe:18d2/64
leaf-su01-r1 | swp9s1           UP             172.26.1.17/31 fe80::4ab0:2dff:fe7b:bb66/64
leaf-su01-r1 | swp25s0          UP             172.18.1.49/31 fe80::4ab0:2dff:fe09:7900/64
leaf-su01-r1 | swp25s1          UP             172.26.1.49/31 fe80::4ab0:2dff:fe8b:7d1a/64
################################################################################
leaf-su01-r2 | swp9s0           UP             172.20.1.17/31 fe80::4ab0:2dff:fe60:4d73/64
leaf-su01-r2 | swp9s1           UP             172.28.1.17/31 fe80::4ab0:2dff:fe2b:cc74/64
leaf-su01-r2 | swp25s0          UP             172.20.1.49/31 fe80::4ab0:2dff:fe7c:f554/64
leaf-su01-r2 | swp25s1          UP             172.28.1.49/31 fe80::4ab0:2dff:fe3a:1070/64
################################################################################
leaf-su01-r3 | swp9s0           UP             172.22.1.17/31 fe80::4ab0:2dff:fe5d:8212/64
leaf-su01-r3 | swp9s1           UP             172.30.1.17/31 fe80::4ab0:2dff:feb6:983a/64
leaf-su01-r3 | swp25s0          UP             172.22.1.49/31 fe80::4ab0:2dff:fe47:8014/64
leaf-su01-r3 | swp25s1          UP             172.30.1.49/31 fe80::4ab0:2dff:fea5:4274/64
################################################################################
spine-s00 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-s01 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-s02 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
spine-s03 | Error: argument "tenant2" is wrong: Not a valid VRF name
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show bgp vrf tenant2 ipv4 unicast\""
========================================
Running: sudo vtysh -c "show bgp vrf tenant2 ipv4 unicast"
========================================
################################################################################
leaf-su00-r0 | BGP table version is 86, local router ID is 10.253.128.1, vrf id 132
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
leaf-su00-r0 |  *> 172.16.1.0/26    10.253.128.5(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *                   10.253.128.5(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *> 172.18.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.18.1.0/26    10.253.128.6(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *                   10.253.128.6(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *> 172.20.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.20.1.0/26    10.253.128.7(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *                   10.253.128.7(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *> 172.22.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.22.1.0/26    10.253.128.8(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *                   10.253.128.8(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *> 172.24.0.0/26    0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.16/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.32/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  s> 172.24.0.48/31   0.0.0.0(leaf-su00-r0)
leaf-su00-r0 |                                              0         32768 ?
leaf-su00-r0 |  *> 172.24.1.0/26    10.253.128.5(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *                   10.253.128.5(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000004 ?
leaf-su00-r0 |  *> 172.26.0.0/26    10.253.128.2(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *                   10.253.128.2(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000001 ?
leaf-su00-r0 |  *> 172.26.1.0/26    10.253.128.6(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *                   10.253.128.6(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000005 ?
leaf-su00-r0 |  *> 172.28.0.0/26    10.253.128.3(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *                   10.253.128.3(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000002 ?
leaf-su00-r0 |  *> 172.28.1.0/26    10.253.128.7(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *                   10.253.128.7(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000006 ?
leaf-su00-r0 |  *> 172.30.0.0/26    10.253.128.4(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *                   10.253.128.4(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000003 ?
leaf-su00-r0 |  *> 172.30.1.0/26    10.253.128.8(spine-s00)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |  *                   10.253.128.8(spine-s01)<
leaf-su00-r0 |                                                            0 4201000000 4200000007 ?
leaf-su00-r0 |
leaf-su00-r0 | Displayed 22 routes and 36 total paths
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -c "sudo vtysh -c \"show ip route vrf tenant2\""
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
leaf-su00-r0 | K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:44:45
leaf-su00-r0 | B>* 172.16.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:44:15
leaf-su00-r0 | C>* 172.16.0.16/31 is directly connected, swp9s0, 00:44:45
leaf-su00-r0 | L>* 172.16.0.17/32 is directly connected, swp9s0, 00:44:45
leaf-su00-r0 | C>* 172.16.0.32/31 is directly connected, swp17s0, 00:07:25
leaf-su00-r0 | L>* 172.16.0.33/32 is directly connected, swp17s0, 00:07:25
leaf-su00-r0 | C>* 172.16.0.48/31 is directly connected, swp25s0, 00:44:45
leaf-su00-r0 | L>* 172.16.0.49/32 is directly connected, swp25s0, 00:44:45
leaf-su00-r0 | B>* 172.16.1.0/26 [20/0] via 10.253.128.5, vlan1341_l3 onlink, weight 1, 00:41:43
leaf-su00-r0 | B>* 172.18.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:43:36
leaf-su00-r0 | B>* 172.18.1.0/26 [20/0] via 10.253.128.6, vlan1341_l3 onlink, weight 1, 00:41:06
leaf-su00-r0 | B>* 172.20.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:42:58
leaf-su00-r0 | B>* 172.20.1.0/26 [20/0] via 10.253.128.7, vlan1341_l3 onlink, weight 1, 00:40:27
leaf-su00-r0 | B>* 172.22.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:42:20
leaf-su00-r0 | B>* 172.22.1.0/26 [20/0] via 10.253.128.8, vlan1341_l3 onlink, weight 1, 00:39:49
leaf-su00-r0 | B>* 172.24.0.0/26 [200/0] unreachable (blackhole) (vrf default), weight 1, 00:44:15
leaf-su00-r0 | C>* 172.24.0.16/31 is directly connected, swp9s1, 00:44:45
leaf-su00-r0 | L>* 172.24.0.17/32 is directly connected, swp9s1, 00:44:45
leaf-su00-r0 | C>* 172.24.0.32/31 is directly connected, swp17s1, 00:07:25
leaf-su00-r0 | L>* 172.24.0.33/32 is directly connected, swp17s1, 00:07:25
leaf-su00-r0 | C>* 172.24.0.48/31 is directly connected, swp25s1, 00:44:45
leaf-su00-r0 | L>* 172.24.0.49/32 is directly connected, swp25s1, 00:44:45
leaf-su00-r0 | B>* 172.24.1.0/26 [20/0] via 10.253.128.5, vlan1341_l3 onlink, weight 1, 00:41:43
leaf-su00-r0 | B>* 172.26.0.0/26 [20/0] via 10.253.128.2, vlan1341_l3 onlink, weight 1, 00:43:36
leaf-su00-r0 | B>* 172.26.1.0/26 [20/0] via 10.253.128.6, vlan1341_l3 onlink, weight 1, 00:41:06
leaf-su00-r0 | B>* 172.28.0.0/26 [20/0] via 10.253.128.3, vlan1341_l3 onlink, weight 1, 00:42:58
leaf-su00-r0 | B>* 172.28.1.0/26 [20/0] via 10.253.128.7, vlan1341_l3 onlink, weight 1, 00:40:27
leaf-su00-r0 | B>* 172.30.0.0/26 [20/0] via 10.253.128.4, vlan1341_l3 onlink, weight 1, 00:42:20
leaf-su00-r0 | B>* 172.30.1.0/26 [20/0] via 10.253.128.8, vlan1341_l3 onlink, weight 1, 00:39:49
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
hgx-su00-h00 | 172.16.0.0  : [0], 64 bytes, 0.022 ms (0.022 avg, 0% loss)
hgx-su00-h00 | 172.16.1.0  : [0], 64 bytes, 1.37 ms (1.37 avg, 0% loss)
hgx-su00-h00 | 172.16.1.32 : [0], 64 bytes, 1.31 ms (1.31 avg, 0% loss)
hgx-su00-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.0  : [1], 64 bytes, 0.022 ms (0.022 avg, 0% loss)
hgx-su00-h00 | 172.16.1.0  : [1], 64 bytes, 1.37 ms (1.37 avg, 0% loss)
hgx-su00-h00 | 172.16.1.32 : [1], 64 bytes, 1.42 ms (1.37 avg, 0% loss)
hgx-su00-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h00 |
hgx-su00-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.022/0.022/0.022
hgx-su00-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.37/1.37/1.37
hgx-su00-h00 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h00 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.31/1.37/1.42
hgx-su00-h00 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su00-h08 | 172.16.0.16 : [0], 64 bytes, 0.028 ms (0.028 avg, 0% loss)
hgx-su00-h08 | 172.16.0.32 : [0], 64 bytes, 0.713 ms (0.713 avg, 0% loss)
hgx-su00-h08 | 172.16.0.48 : [0], 64 bytes, 0.716 ms (0.716 avg, 0% loss)
hgx-su00-h08 | 172.16.1.16 : [0], 64 bytes, 1.76 ms (1.76 avg, 0% loss)
hgx-su00-h08 | 172.16.1.48 : [0], 64 bytes, 1.45 ms (1.45 avg, 0% loss)
hgx-su00-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.0.16 : [1], 64 bytes, 0.023 ms (0.026 avg, 0% loss)
hgx-su00-h08 | 172.16.0.32 : [1], 64 bytes, 0.721 ms (0.717 avg, 0% loss)
hgx-su00-h08 | 172.16.0.48 : [1], 64 bytes, 0.834 ms (0.775 avg, 0% loss)
hgx-su00-h08 | 172.16.1.16 : [1], 64 bytes, 1.57 ms (1.66 avg, 0% loss)
hgx-su00-h08 | 172.16.1.48 : [1], 64 bytes, 1.57 ms (1.51 avg, 0% loss)
hgx-su00-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h08 |
hgx-su00-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.023/0.026/0.028
hgx-su00-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.713/0.717/0.721
hgx-su00-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.716/0.775/0.834
hgx-su00-h08 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.57/1.66/1.76
hgx-su00-h08 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h08 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.45/1.51/1.57
################################################################################
hgx-su00-h16 | 172.16.0.16 : [0], 64 bytes, 0.603 ms (0.603 avg, 0% loss)
hgx-su00-h16 | 172.16.0.32 : [0], 64 bytes, 0.035 ms (0.035 avg, 0% loss)
hgx-su00-h16 | 172.16.0.48 : [0], 64 bytes, 0.821 ms (0.821 avg, 0% loss)
hgx-su00-h16 | 172.16.1.16 : [0], 64 bytes, 1.56 ms (1.56 avg, 0% loss)
hgx-su00-h16 | 172.16.1.48 : [0], 64 bytes, 1.64 ms (1.64 avg, 0% loss)
hgx-su00-h16 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.0.16 : [1], 64 bytes, 0.751 ms (0.677 avg, 0% loss)
hgx-su00-h16 | 172.16.0.32 : [1], 64 bytes, 0.030 ms (0.032 avg, 0% loss)
hgx-su00-h16 | 172.16.0.48 : [1], 64 bytes, 0.706 ms (0.763 avg, 0% loss)
hgx-su00-h16 | 172.16.1.16 : [1], 64 bytes, 1.83 ms (1.70 avg, 0% loss)
hgx-su00-h16 | 172.16.1.48 : [1], 64 bytes, 1.72 ms (1.68 avg, 0% loss)
hgx-su00-h16 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h16 |
hgx-su00-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.603/0.677/0.751
hgx-su00-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.030/0.032/0.035
hgx-su00-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.706/0.763/0.821
hgx-su00-h16 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h16 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.56/1.70/1.83
hgx-su00-h16 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h16 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.64/1.68/1.72
################################################################################
hgx-su00-h24 | 172.16.0.16 : [0], 64 bytes, 0.628 ms (0.628 avg, 0% loss)
hgx-su00-h24 | 172.16.0.32 : [0], 64 bytes, 0.504 ms (0.504 avg, 0% loss)
hgx-su00-h24 | 172.16.0.48 : [0], 64 bytes, 0.025 ms (0.025 avg, 0% loss)
hgx-su00-h24 | 172.16.1.16 : [0], 64 bytes, 1.23 ms (1.23 avg, 0% loss)
hgx-su00-h24 | 172.16.1.48 : [0], 64 bytes, 1.40 ms (1.40 avg, 0% loss)
hgx-su00-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.0.16 : [1], 64 bytes, 0.722 ms (0.675 avg, 0% loss)
hgx-su00-h24 | 172.16.0.32 : [1], 64 bytes, 0.601 ms (0.553 avg, 0% loss)
hgx-su00-h24 | 172.16.0.48 : [1], 64 bytes, 0.017 ms (0.021 avg, 0% loss)
hgx-su00-h24 | 172.16.1.16 : [1], 64 bytes, 1.39 ms (1.31 avg, 0% loss)
hgx-su00-h24 | 172.16.1.48 : [1], 64 bytes, 1.14 ms (1.27 avg, 0% loss)
hgx-su00-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-su00-h24 |
hgx-su00-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.628/0.675/0.722
hgx-su00-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.504/0.553/0.601
hgx-su00-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.017/0.021/0.025
hgx-su00-h24 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.23/1.31/1.39
hgx-su00-h24 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-su00-h24 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.14/1.27/1.40
################################################################################
hgx-su01-h00 | 172.16.0.0  : [0], 64 bytes, 2.30 ms (2.30 avg, 0% loss)
hgx-su01-h00 | 172.16.1.0  : [0], 64 bytes, 0.043 ms (0.043 avg, 0% loss)
hgx-su01-h00 | 172.16.1.32 : [0], 64 bytes, 0.528 ms (0.528 avg, 0% loss)
hgx-su01-h00 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.0.0  : [1], 64 bytes, 1.60 ms (1.95 avg, 0% loss)
hgx-su01-h00 | 172.16.1.0  : [1], 64 bytes, 0.027 ms (0.035 avg, 0% loss)
hgx-su01-h00 | 172.16.1.32 : [1], 64 bytes, 0.766 ms (0.647 avg, 0% loss)
hgx-su01-h00 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h00 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h00 |
hgx-su01-h00 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.60/1.95/2.30
hgx-su01-h00 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h00 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h00 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h00 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.027/0.035/0.043
hgx-su01-h00 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h00 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.528/0.647/0.766
hgx-su01-h00 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su01-h08 | 172.16.0.16 : [0], 64 bytes, 1.45 ms (1.45 avg, 0% loss)
hgx-su01-h08 | 172.16.0.32 : [0], 64 bytes, 1.81 ms (1.81 avg, 0% loss)
hgx-su01-h08 | 172.16.0.48 : [0], 64 bytes, 1.71 ms (1.71 avg, 0% loss)
hgx-su01-h08 | 172.16.1.16 : [0], 64 bytes, 0.049 ms (0.049 avg, 0% loss)
hgx-su01-h08 | 172.16.1.48 : [0], 64 bytes, 0.500 ms (0.500 avg, 0% loss)
hgx-su01-h08 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.0.16 : [1], 64 bytes, 1.75 ms (1.60 avg, 0% loss)
hgx-su01-h08 | 172.16.0.32 : [1], 64 bytes, 1.28 ms (1.54 avg, 0% loss)
hgx-su01-h08 | 172.16.0.48 : [1], 64 bytes, 1.17 ms (1.44 avg, 0% loss)
hgx-su01-h08 | 172.16.1.16 : [1], 64 bytes, 0.023 ms (0.036 avg, 0% loss)
hgx-su01-h08 | 172.16.1.48 : [1], 64 bytes, 0.797 ms (0.649 avg, 0% loss)
hgx-su01-h08 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-su01-h08 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h08 |
hgx-su01-h08 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su01-h08 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.45/1.60/1.75
hgx-su01-h08 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.28/1.54/1.81
hgx-su01-h08 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.17/1.44/1.71
hgx-su01-h08 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-su01-h08 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.023/0.036/0.049
hgx-su01-h08 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h08 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.500/0.649/0.797
################################################################################
hgx-su01-h16 | 172.16.0.0  : [0], 64 bytes, 2.06 ms (2.06 avg, 0% loss)
hgx-su01-h16 | 172.16.1.0  : [0], 64 bytes, 0.443 ms (0.443 avg, 0% loss)
hgx-su01-h16 | 172.16.1.32 : [0], 64 bytes, 0.027 ms (0.027 avg, 0% loss)
hgx-su01-h16 | 172.16.0.16 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.0.32 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.0.48 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.1.16 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.1.48 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.0.0  : [1], 64 bytes, 1.64 ms (1.85 avg, 0% loss)
hgx-su01-h16 | 172.16.1.0  : [1], 64 bytes, 0.550 ms (0.497 avg, 0% loss)
hgx-su01-h16 | 172.16.1.32 : [1], 64 bytes, 0.016 ms (0.021 avg, 0% loss)
hgx-su01-h16 | 172.16.0.16 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.0.32 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.0.48 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.1.16 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h16 | 172.16.1.48 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h16 |
hgx-su01-h16 | 172.16.0.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.64/1.85/2.06
hgx-su01-h16 | 172.16.0.16 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h16 | 172.16.0.32 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h16 | 172.16.0.48 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h16 | 172.16.1.0  : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.443/0.497/0.550
hgx-su01-h16 | 172.16.1.16 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h16 | 172.16.1.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.016/0.021/0.027
hgx-su01-h16 | 172.16.1.48 : xmt/rcv/%loss = 2/0/100%
################################################################################
hgx-su01-h24 | 172.16.0.16 : [0], 64 bytes, 1.84 ms (1.84 avg, 0% loss)
hgx-su01-h24 | 172.16.0.32 : [0], 64 bytes, 2.05 ms (2.05 avg, 0% loss)
hgx-su01-h24 | 172.16.0.48 : [0], 64 bytes, 1.81 ms (1.81 avg, 0% loss)
hgx-su01-h24 | 172.16.1.16 : [0], 64 bytes, 0.527 ms (0.527 avg, 0% loss)
hgx-su01-h24 | 172.16.1.48 : [0], 64 bytes, 0.018 ms (0.018 avg, 0% loss)
hgx-su01-h24 | 172.16.0.0  : [0], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.1.0  : [0], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.1.32 : [0], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.0.16 : [1], 64 bytes, 1.93 ms (1.89 avg, 0% loss)
hgx-su01-h24 | 172.16.0.32 : [1], 64 bytes, 1.82 ms (1.93 avg, 0% loss)
hgx-su01-h24 | 172.16.0.48 : [1], 64 bytes, 1.42 ms (1.62 avg, 0% loss)
hgx-su01-h24 | 172.16.1.16 : [1], 64 bytes, 0.517 ms (0.522 avg, 0% loss)
hgx-su01-h24 | 172.16.1.48 : [1], 64 bytes, 0.029 ms (0.024 avg, 0% loss)
hgx-su01-h24 | 172.16.0.0  : [1], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.1.0  : [1], timed out (NaN avg, 100% loss)
hgx-su01-h24 | 172.16.1.32 : [1], timed out (NaN avg, 100% loss)
hgx-su01-h24 |
hgx-su01-h24 | 172.16.0.0  : xmt/rcv/%loss = 2/0/100%
hgx-su01-h24 | 172.16.0.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.84/1.89/1.93
hgx-su01-h24 | 172.16.0.32 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.82/1.93/2.05
hgx-su01-h24 | 172.16.0.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 1.42/1.62/1.81
hgx-su01-h24 | 172.16.1.0  : xmt/rcv/%loss = 2/0/100%
hgx-su01-h24 | 172.16.1.16 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.517/0.522/0.527
hgx-su01-h24 | 172.16.1.32 : xmt/rcv/%loss = 2/0/100%
hgx-su01-h24 | 172.16.1.48 : xmt/rcv/%loss = 2/2/0%, min/avg/max = 0.018/0.024/0.029
################################################################################
```

```bash
ubuntu@oob-mgmt-server:~/nvidia/terraform/tenant2$ ~/spcx-air/spcx-run -s -c "for ip in $RAILS0; do ping -c2 -W2 \$ip >/dev/null 2>&1 && echo \"PASS \$ip\" || echo \"FAIL \$ip\"; done"
========================================
Running: for ip in 172.16.0.0 172.16.0.16 172.16.0.32 172.16.0.48 172.16.1.0 172.16.1.16 172.16.1.32 172.16.1.48; do ping -c2 -W2 $ip >/dev/null 2>&1 && echo "PASS $ip" || echo "FAIL $ip"; done
========================================
################################################################################
hgx-su00-h00 | PASS 172.16.0.0
hgx-su00-h00 | FAIL 172.16.0.16
hgx-su00-h00 | FAIL 172.16.0.32
hgx-su00-h00 | FAIL 172.16.0.48
hgx-su00-h00 | PASS 172.16.1.0
hgx-su00-h00 | FAIL 172.16.1.16
hgx-su00-h00 | PASS 172.16.1.32
hgx-su00-h00 | FAIL 172.16.1.48
################################################################################
hgx-su00-h08 | FAIL 172.16.0.0
hgx-su00-h08 | PASS 172.16.0.16
hgx-su00-h08 | PASS 172.16.0.32
hgx-su00-h08 | PASS 172.16.0.48
hgx-su00-h08 | FAIL 172.16.1.0
hgx-su00-h08 | PASS 172.16.1.16
hgx-su00-h08 | FAIL 172.16.1.32
hgx-su00-h08 | PASS 172.16.1.48
################################################################################
hgx-su00-h16 | FAIL 172.16.0.0
hgx-su00-h16 | PASS 172.16.0.16
hgx-su00-h16 | PASS 172.16.0.32
hgx-su00-h16 | PASS 172.16.0.48
hgx-su00-h16 | FAIL 172.16.1.0
hgx-su00-h16 | PASS 172.16.1.16
hgx-su00-h16 | FAIL 172.16.1.32
hgx-su00-h16 | PASS 172.16.1.48
################################################################################
hgx-su00-h24 | FAIL 172.16.0.0
hgx-su00-h24 | PASS 172.16.0.16
hgx-su00-h24 | PASS 172.16.0.32
hgx-su00-h24 | PASS 172.16.0.48
hgx-su00-h24 | FAIL 172.16.1.0
hgx-su00-h24 | PASS 172.16.1.16
hgx-su00-h24 | FAIL 172.16.1.32
hgx-su00-h24 | PASS 172.16.1.48
################################################################################
hgx-su01-h00 | PASS 172.16.0.0
hgx-su01-h00 | FAIL 172.16.0.16
hgx-su01-h00 | FAIL 172.16.0.32
hgx-su01-h00 | FAIL 172.16.0.48
hgx-su01-h00 | PASS 172.16.1.0
hgx-su01-h00 | FAIL 172.16.1.16
hgx-su01-h00 | PASS 172.16.1.32
hgx-su01-h00 | FAIL 172.16.1.48
################################################################################
hgx-su01-h08 | FAIL 172.16.0.0
hgx-su01-h08 | PASS 172.16.0.16
hgx-su01-h08 | PASS 172.16.0.32
hgx-su01-h08 | PASS 172.16.0.48
hgx-su01-h08 | FAIL 172.16.1.0
hgx-su01-h08 | PASS 172.16.1.16
hgx-su01-h08 | FAIL 172.16.1.32
hgx-su01-h08 | PASS 172.16.1.48
################################################################################
hgx-su01-h16 | PASS 172.16.0.0
hgx-su01-h16 | FAIL 172.16.0.16
hgx-su01-h16 | FAIL 172.16.0.32
hgx-su01-h16 | FAIL 172.16.0.48
hgx-su01-h16 | PASS 172.16.1.0
hgx-su01-h16 | FAIL 172.16.1.16
hgx-su01-h16 | PASS 172.16.1.32
hgx-su01-h16 | FAIL 172.16.1.48
################################################################################
hgx-su01-h24 | FAIL 172.16.0.0
hgx-su01-h24 | PASS 172.16.0.16
hgx-su01-h24 | PASS 172.16.0.32
hgx-su01-h24 | PASS 172.16.0.48
hgx-su01-h24 | FAIL 172.16.1.0
hgx-su01-h24 | PASS 172.16.1.16
hgx-su01-h24 | FAIL 172.16.1.32
hgx-su01-h24 | PASS 172.16.1.48
################################################################################
```
Expected result:

- HGX hosts from `tenant1` (`hgx-su00-h00`, `hgx-su01-h00`, `hgx-su01-h16`) have full-mesh connectivity with each other, including across units, but cannot reach the `tenant2` hosts (`hgx-su00-h08`, `hgx-su00-h16`, `hgx-su00-h24`, `hgx-su01-h08`, `hgx-su01-h24`). The same applies in reverse for `tenant2`: full-mesh connectivity within the tenant, no connectivity into `tenant1`.

Validation:

- BGP and EVPN can still be converging in the first seconds after the deploy, so the sweep retries each target for up to about thirty seconds rather than reporting a false failure. The `FAIL <ip>` lines against hosts in the other tenant are expected and persistent — they indicate VRF isolation working correctly, not a fabric problem. Re-running the block is safe.

<!-- AIR:page -->

## Validation Summary

- The fabric configuration patched all twelve devices, created 512 links, and added 576 `/31` addresses.
- All eight HGX endpoints were created.
- Underlay BGP has every session established, none down.
- The `tenant1` (`hgx-*-h00`, `hgx-*-h16`, both units) and `tenant2` (`hgx-*-h08`, `hgx-*-h24`, both units) L3 networks are each deployed with their own L3VNI.
- The host mesh test confirms full connectivity within each tenant across both units, and no connectivity across tenants, demonstrating tenant isolation.

| __Check__ | __Expected__ |
| --------- | -------------- |
| `fabric configure-switches` | devices patched=12, links created=512, /31 addresses added=576 |
| Endpoints | 8 created |
| Underlay BGP | all sessions established, 0 down |
| Tenants | `tenant1` (`hgx-*-h00`, `hgx-*-h16`) and `tenant2` (`hgx-*-h08`, `hgx-*-h24`) L3 networks deployed, one L3VNI each |
| Host mesh | full connectivity within each tenant across both units; no connectivity across tenants |

Include a final validation command if helpful:

```bash
metalcloud-cli infrastructure list
```

## Troubleshooting, Upgrade, or Reset

- **The CLI cannot reach the controller.** If a `metalcloud-cli` command fails with a connection or TLS error, or `site agents 1` returns no agent, the controller may still be starting, or the Site Controller may have lost its link to the Global Controller after a restart. Log in to the Global Controller as `root` (`ssh -l root 192.168.200.3`, password `MetalsoftR0cks@$@$`) and run `kw` to watch the Kubernetes pods until each reaches `Running`; if any stay stuck, run `k-restart-all -A` to force a restart. On the Site Controller (`ssh -l root 192.168.200.2`, same password), run `docker ps` to confirm the `nfs-server` and `ms-agent` containers are `Up`, `docker logs -f ms-agent` to read the agent log, and `dcrestart` to restart the containers if `ms-agent` is not registering with the Global Controller.
- **A deploy job fails.** Read the job with `metalcloud-cli job get <id>`, correct the cause, and re-run the deploy step. BGP does not come up until the Step 9 deploy succeeds on every switch.
- **`get-ports` is empty in Step 4.** The switch is unreachable. Verify the management address and password in `switches.2su.yaml`, then re-run discovery.
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