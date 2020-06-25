# Overview

An all-in-one MAAS setup.

This project is for installing a MAAS cluster on a single machine. 

Description of the entire environment:

* 1 large host (the "KVM host") running Ubuntu 18.04 LTS or Ubuntu 20.04 LTS

* 6 KVM guests residing on the KVM host:
     - 1 for the MAAS host itself
     - 1 for the Juju controller
     - 4 for the MAAS nodes (available for deployments)

* 2 libvirt networks:
     - 'external' for the external side of the MAAS host (DHCP enabled)
     - 'internal' for the internal side of the MAAS host (DHCP disabled)

* The KVM host, beyond hosting the guests, will act as the Juju client

MAAS is installed from a snap.

The four MAAS nodes are powerful machines with multiple network interfaces
and disks. The original intent was the deployment of Charmed OpenStack.
Adjust per your needs and desires.

Network diagram:

[ INSERT NETWORK DIAGRAM HERE ]

The subnet DNS:

    10.0.0.1

The subnet gateway:

    10.0.0.1

The reserved IP ranges:

    10.0.0.1   - 10.0.0.9     Infra
    10.0.0.10  - 10.0.0.99    VIP       <-- HA workloads
    10.0.0.100 - 10.0.0.119   Dynamic  	<-- DHCP (enlistment, commissioning)

So any deployed nodes will use:
   
    10.0.0.120 - 10.0.0.254

## Before you begin

Before you begin look over all the files. They're pretty simple.

## Install the software

SSH to host with agent forwarding enabled. Forwarding can help with connectivity
as uvtool can auto-install the agent's keys on its created instances. 

    ssh -A <kvm-host>
    
    cd
    sudo apt update
    sudo apt full-upgrade -y
    sudo apt install -y uvtool virtinst
    sudo uvt-simplestreams-libvirt sync release=focal arch=amd64
    sudo snap install juju --classic
    sudo snap install charm --classic
    sudo snap install openstackclients --classic
    charm pull openstack-base
    git clone https://github.com/pmatulis/maas-one

The `uvt-simplestreams-libvirt` command provides the release for the MAAS
host itself.

## Set up the environment

Log out and back in again and ensure the 'default' libvirt network exists:

    virsh net-list --all

OPTIONAL: Use ZFS pools with extra disks
(or some other way to optimise the disk sub-system) 

If choosing ZFS like this, perform the steps in zfs-pools.txt now.

Create the libvirt networks:

    cd ~/maas-one
    ./create-networks.sh

Create a test instance to discover the names of the two MAAS host network
interfaces (created via template-maas.xml). Reference these in
user-data-maas.yaml, the cloud-init file for the MAAS host.

    uvt-kvm create --template ./template-maas.xml test release=focal
    uvt-kvm ssh test ip a  # e.g. enp1s0 and enp2s0
    uvt-kvm destroy test

Edit user-data-maas.yaml:

Your personal SSH key(s) are imported three times (INSERT YOURS instead
of 'petermatulis'):

1. to the MAAS host 'ubuntu' user
   - to allow basic connections to the MAAS host

1. to the MAAS host 'root' user
   - to allow transferring the 'root' user public SSH key to the KVM host
     (for MAAS to be able to manage power of KVM guests)

1. to the MAAS server 'admin' user
   - key will be installed on every MAAS-deployed node

## Create the MAAS host and server

Create the MAAS host and server from the KVM host:

    cd ~/maas-one
    uvt-kvm create \
       --template ./template-maas.xml \
       --user-data ./user-data-maas.yaml \
       --cpu 4 --memory 4096 --disk 30 maas \
       release=focal

Wait 5 minutes before attempting to contact the MAAS host:

    ssh ubuntu@10.0.0.2 tail -f /var/log/cloud-init-output.log

## Post install MAAS tasks

Transfer over the MAAS 'admin' user's API key:

    scp ubuntu@10.0.0.2:admin-api-key ~

Install the MAAS host 'root' user public SSH key into the 'ubuntu'
user account on the KVM host:

    ssh root@10.0.0.2 cat /var/snap/maas/current/root/.ssh/id_rsa.pub >> /home/ubuntu/.ssh/authorized_keys

Confirm that the 'root' user can query the KVM host's guests:

    ssh ubuntu@10.0.0.2
    sudo snap run --shell maas
    virsh -c qemu+ssh://ubuntu@10.0.0.1/system list --all
    exit
    exit

Transfer some scripts to the MAAS host:

    scp config-maas.sh config-nodes.sh maas-login.sh ubuntu@10.0.0.2:

## Configure MAAS

Connect to the MAAS host and run a script:

    ssh ubuntu@10.0.0.2
    ./config-maas.sh
    exit

## Create the nodes

Run a script on the KVM host:

    cd ~/maas-one
    ./create-nodes.sh

## Verify the web UI

Set up local port forwarding from your workstation:

    ssh -N -L 8002:10.0.0.2:5240 ubuntu@<kvm-host>

Access the web UI:

    http://localhost:8002/MAAS
    credentials: admin/ubuntu

Confirm everything (node names, power, networking, images).

Verify controller status ('regiond' to 'dhcpd' should be green)
If not green:

    ssh ubuntu@10.0.0.2 sudo systemctl restart maas-rackd.service
    ssh ubuntu@10.0.0.2 sudo systemctl restart maas-regiond.service

Continue when the nodes are all in the 'New' state. 

## Configure the nodes

Rename, configure power, and commission the nodes.

Connect to the MAAS host:

    ssh ubuntu@10.0.0.2
    ./config-nodes.sh
    exit

## Configure Juju

Define a MAAS cloud, add it to Juju, and add a cloud credential:

Run a script on the KVM host:

    cd ~/maas-one
    ./add-cloud-and-creds.sh

## Create the Juju controller

Create the controller from the KVM host:

    juju bootstrap --bootstrap-constraints tags=juju mymaas maas-controller
