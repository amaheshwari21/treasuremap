..
      Copyright 2017 AT&T Intellectual Property.
      All Rights Reserved.

      Licensed under the Apache License, Version 2.0 (the "License"); you may
      not use this file except in compliance with the License. You may obtain
      a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

      Unless required by applicable law or agreed to in writing, software
      distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
      WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
      License for the specific language governing permissions and limitations
      under the License.

Airship Deployment
==============

Terminology
-----------

**Cloud**: A platform that provides a standard set of interfaces for `IaaS <https://en.wikipedia.org/wiki/Infrastructure_as_a_service>`_ consumers.

**UCP**: (`UnderCloud Platform <https://github.com/att-comdev>`_) is a broad integration of several components enabling
an automated, resilient Kubernetes-based infrastructure for hosting Helm-deployed
containerized workloads.

**OSH**: (`OpenStack Helm <https://docs.openstack.org/openstack-helm/latest/>`_) is a collection of Helm charts used to deploy OpenStack
on kubernetes.

**Undercloud/Overcloud**: Terms used to distinguish which cloud is deployed on
top of the other. In this implementation, OSH (overcloud) is deployed on top of
UCP (underlcoud).

**Airship**: (OpenStack on Kubernetes) refers to the specific implementation of
OpenStack Helm charts onto UCP kubernetes that is documented in this project.

**Control Plane**: From the point of view of the cloud service provider, the
control plane refers to the set of resources (hardware, network, storage, etc).
sourced to run cloud services.
The reference deployment of Airship does not distinguish between UCP and OSH
control planes, as it assumes the hardware is the same for both.

**Data Plane**: From the point of view of the cloud service provider, the data
plane is the set of resources (hardware, network, storage, etc.) sourced to
to run consumer workloads.
When used in this document, "data plane" refers to the data plane of the
overcloud (OSH).

**Host Profile**: A host profile is a standard way of configuring a bare metal
host. Encompasses items such as the number of bonds, bond slaves, physical
storage mapping and partitioning, and kernel parameters.

Component Overview
^^^^^^^^^^^^^^^^^^

.. image:: diagrams/component_list.png

Node overview
^^^^^^^^^^^^^

This document refers to several types of nodes, which vary in their purpose, and
to some degree in their orchestration / setup:

- **Build node**: This refers to the environment where configuration documents are
  built for your environment.
- **Genesis node**: The "genesis" or "seed node" refers to a short-lived node used
  to get a new deployment off the ground, and is the first node built in a new
  deployment environment.
- **Control / Controller nodes**: The nodes that make up the control plane
- **Compute nodes / Worker Nodes**: The nodes that make up the data plane

Support
-------

Bugs may be followed against components as follows:

- General: If you're unsure of the root cause of an issue, you may file it in
  the `Github issues for Treasuremap <https://github.com/att-comdev/treasuremap/issues>`_.
  However, if you know the root issue, then the more expedient path to issue
  resolution is to file bugs against OSH and specific UCP projects as follows:

  - UCP: Bugs may be filed using OpenStack Storyboard for specific projects in `Airship group <https://storyboard.openstack.org/#!/project_group/85>`_:

    - `Airship Armada <https://storyboard.openstack.org/#!/project/1002>`_
    - `Airship Berth <https://storyboard.openstack.org/#!/project/1003>`_
    - `Airship Deckhand <https://storyboard.openstack.org/#!/project/1004>`_
    - `Airship Divingbell <https://storyboard.openstack.org/#!/project/1001>`_
    - `Airship Drydock <https://storyboard.openstack.org/#!/project/1005>`_
    - `Airship MaaS <https://storyboard.openstack.org/#!/project/1007>`_
    - `Airship Pegleg <https://storyboard.openstack.org/#!/project/1008>`_
    - `Airship Promenade <https://storyboard.openstack.org/#!/project/1009>`_
    - `Airship Shipyard <https://storyboard.openstack.org/#!/project/1010>`_
    - `Airship in a Bottle <https://storyboard.openstack.org/#!/project/1006>`_

  - OSH: Bugs may be filed against OpenStack Helm in `launchpad <https://bugs.launchpad.net/openstack-helm/>`_.

Pre-req
-------

1. Ensure your environment has access to the artifactory instance where
   UCP and OSH artifacts are published.
2. Establish a location you will use to archive/track your configuration
   documents (e.g., github)

Hardware Prep
-------------

Disk
^^^^

1. Control plane server disks:

   - Two-disk RAID-1 mirror for operating system
   - Remaining disks as JBOD for Ceph. (Ceph journals should preferentially be
     deployed to SSDs where available.)

2. Data plane server disks:

   - Two-disk RAID-1 mirror for operating system
   - Remaining disks need to be configured according to the host profile target
     for each given server (e.g., RAID-6; no Ceph).

BIOS and IPMI
^^^^^^^^^^^^^

1. Virtualization enabled in BIOS
2. IPMI enabled in server BIOS (e.g., IPMI over LAN option enabled)
3. IPMI IPs assigned, and routed to the environment you will deploy into
   Note: Firmware bugs related to IPMI are common. Ensure you are running the
   latest firmware version for your hardware. Otherwise, it is recommended to
   perform an iLo/iDrac reset, as IPMI bugs with long-running firmware are not
   uncommon.
4. Set PXE as first boot device and ensure the correct NIC is selected for PXE

Network
^^^^^^^

1. You have a network you can successfully PXE boot with your network topology
   and bonding settings (dedicated PXE interace on untagged/native VLAN in this
   example)
2. You have (VLAN) segmented, routed networks accessible by all nodes for:

   1. Management network(s) (k8s control channel)
   2. Calico network(s)
   3. Storage network(s)
   4. Overlay network(s)
   5. Public network(s)

HW Sizing and minimum requirements
----------------------------------

+----------+----------+----------+----------+
|  Node    |   disk   |  memory  |   cpu    |
+==========+==========+==========+==========+
|  Build   |   10 GB  |  4 GB    |   1      |
+----------+----------+----------+----------+
| Genesis  |   100 GB |  16 GB   |   8      |
+----------+----------+----------+----------+
| Control  |   10 TB  |  128 GB  |   24     |
+----------+----------+----------+----------+
| Compute  |   N/A*   |  N/A*    |   N/A*   |
+----------+----------+----------+----------+

* Workload driven (determined by host profile)

Establishing build node environment
-----------------------------------

1. On the machine you wish to use to generate deployment files, install required
   tooling::

    sudo apt -y install docker.io git

2. Clone and link the required git repos as follows::

    cd ~
    git clone https://github.com/openstack/airship-pegleg pegleg
    git clone https://github.com/att-comdev/treasuremap

Building Site documents
-----------------------

This section goes over how to put together site documents according to your
specific environment, and generate the initial Promenade bundle needed to start
the site deployment.

Preparing deployment documents
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In its current form, pegleg provides an organized structure for YAML elements,
in order to separate common site elements (i.e., ``global`` folder) from unique
site elements (i.e., ``site`` folder).

To gain a full understanding of the pegleg strutcure, it is highly recommended
to read pegleg documentation on this `here <https://pegleg.readthedocs.io/en/latest/artifacts.html>`_.

Change directory to the ``treasuremap/deployment_files`` folder and copy an existing
site to use as a reference for $NEW_SITE::

    NEW_SITE=mySite
    cd treasuremap/deployment_files
    cp -r site/atl-lab1 site/$NEW_SITE

The follow sections will highligh changes that should be made to each YAML to
correctly configure your environment's deployment.

Generate secrets
^^^^^^^^^^^^^^^^

Generate the passphrases used in your environment as follows::

    (cd secrets_tools && ./gen.sh)

Move the secrets to your $NEW_SITE's location for passphrase secrets::

    mkdir -p site/$NEW_SITE/secrets/passphrases
    mv secrets_tools/*.yaml site/$NEW_SITE/secrets/passphrases

Public SSH keys for environment access are stored under
``global/common/secrets/publickey/``. Make copies of ``sb464f_ssh_public_key.yaml``
and name the copies according to each ssh key you wish to specify that will have
bare metal SSH acess. Delete any unneeded keys leftover from ``atl-lab1``.
Modify the contents of each remaining file as follows:

- metadata/name: Specify the name of public SSH key
- data: Specify the public SSH key (``ssh-rsa ...``)

site/$NEW_SITE/profiles/region.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File containing the Drydock region definition for this site. Setting highlights:

- metadata/name: Set to the desired region name (e.g., ``$NEW_SITE``). For current
  deployment purposes, the region name should be set the same as the site name
  in the next section.
- metadata/substitutions: Substitutions for SSH public key passed to Drydock.
  These keys will be deployed to bare metal when it is PXE booted. Define
  substitutions for each SSH key defined in the previous section, e.g.::

    substitutions:
      - dest:
          path: .authorized_keys[0]
        src:
          schema: deckhand/PublicKey/v1
          name: sb464f_ssh_public_key
          path: .
      - dest:
          path: .authorized_keys[1]
        src:
          schema: deckhand/PublicKey/v1
          name: am240k_ssh_public_key
          path: .

  where the number enclosed in square brackets is a zero-indexed iterable, and
  the ``name`` for each matches the names of the SSH keyes defined in the
  publickey secrets from the previous section.

site/$NEW_SITE/site-definition.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The root level site definition file. Setting highlights:

- data/revision: Set to the desired revision of shared ``global`` and
  ``type`` elements in the site heirarhcy. For example, you would specify ``v1.0``
  to overlay your site data onto elements from ``./pegleg/global/v1.0`` and
  ``./pegleg/type/*/v1.0``.
- data/site_type: Set to the desired site type (e.g., ``cicd``, ``large``, etc)
- metadata/name: Set to the desired site name (e.g., ``$NEW_SITE``)

site/$NEW_SITE/networks/physical/rack06-network.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File containing Drydock definitions of NetworkLink and Network elements.

Begin by reviewing each ``drydock/Network/v1`` element. In this example, the
networks we reference are:

- Rack06 PXE: rack06-pxe
- Rack06 Management: rack06-mgmt
- Rack06 Storage: rack06-storage
- Rack06 Calico: rack06-calico
- Rack06 OpenStack SDN: rack06-ossdn
- Rack06 Contrail: rack06-contrail
- Rack06 Publically routed network: rack06-public

Although we have only one rack of servers in our example, we assume a naming
convention that implies a per-rack broadcast domain to support the possibility
of future rack expansion in this environment.

Create and configure the ``drydock/Network/v1`` elements according to your
environment's network. Setting highlights:

- data/cidr: Populate with the expected CIDR for each logical network.
- data/dhcp_relay/upstream_target: If your environment contains more than one
  broadcast domain for PXE traffic, you should use this parameter to specify the
  IP address of a DHCP relay which will forward DHCP broadcasts between PXE L2
  networks.
- data/routes: Populate with the list of routes for each network. The default
  route should be defined on the management network. Define static routes to
  reach local subnets (routing from rack06 storage to rack07 storage, etc).
- data/ranges: Populate with the allocation ranges for each network.

  - Use ``type: 'static'`` for the IP range you want to allocate from.
  - Define one or more ``type: 'reserved'`` elements to reserve IP ranges to prevent
    address conflicts with other infrastructure. By convention, the first and/or
    last several IP addresses in a subnet are often used for the gateway IP,
    HSRP, VPN, or other network infrastructure.
  - Use ``type: 'dhcp'`` for PXE networks, in addition to the 'static' range.
    Currently Drydock uses default MaaS behavior, which is to PXE boot nodes
    using this dhcp range (for disocvery and commissioning), and then to deploy
    nodes using IPs from the static pool defined. This requires twice the IP
    address space, but facilitates Promenade-driven kubernetes cluster formation
    which currently requires knowing node IP addresses in advance.

- data/dns/domain: The domain which will be configured for PXE booted nodes.
- data/dns/servers: The DNS servers which will be configured for PXE booted
  nodes. You may specify corporate DNS servers here, as long as those servers
  can resolve upstream (internet) FQDNs.

This file should also be populated with a ``drydock/NetworkLink/v1`` definition
for each type of logical interface you plan to use. In this example, there are
three:

- One NetworkLink for the out of band logical interface (IPMI)
- One NetworkLink for PXE logical interface
- One NetworkLink for a single link aggregated bond

(Other environments that leverage LACP fallback would have only two NetworkLink
elements, as PXE would be combined with the bond interface.)

NetworkLinks should be configured according to your environment. Pay special
attention to the aggregation protocol (if using bonding), the interface MTU, and
the allowed_networks. Configure the allowed_networks for each NetworkLink with
the names of the L3 Network elements you want to go over these interfaces.

Also, note that the NetworkLink for the out of band interface has an extra data
label, ``noconfig: 'enabled'`` to indicate that the network will not be created by
Drydock/MaaS, as this network is assumed to already be in place and managed by
existing infrastructure as a prerequisite to site deployment.

site/$NEW_SITE/networks/common-address.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File containing a number of high-level UCP network related parameters. Setting
highlights:

- data/calico/ip_autodetection_method: The genesis node interface that calico
  will use. In practice, this should be the interface that is assigned a routed
  IP address (i.e. on the management network). Specify as ``interface=ens5`` or
  multiple matches with ``interface=bond0.22|ens5``, adjusting according to your
  genesis node interface name(s).
- data/dns/upstream_servers: Upstream DNS servers. You may specify corporate DNS
  servers here, as long as those servers can resolve upstream (internet) FQDNs.
- data/genesis/hostname: Set to the hostname used to provision the genesis node.
- data/genesis/ip: Set to the static IP address which was manually configured
  for the genesis node.
- data/masters: Designate nodes that will run kubernetes master services. You
  should specify the same list of nodes which will run UCP services (control
  plane nodes).
- data/workers: Designate nodes that will not run kubernetes master services and
  will be used for hosting user workloads (e.g., compute nodes)
- data/ntp/servers_joined: Upstream NTP servers. Use local NTP sources if
  available, or corporate or other reachable external sources where local NTP is
  not available.
- data/storage/ceph/cluster_cidr: CIDR(s) for Ceph internal traffic. Set this to
  the list of all management networks used in the environment that will host
  Ceph services. In practice, this means the list of the management networks
  assigned to nodes designated to run UCP services (control plane nodes).
- data/storage/ceph/public_cidr: Set the same as above.

site/$NEW_SITE/profiles/hardware/hw_generic.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File containg the generic HardwareProfile for this site.

In the future, this file will track hardware detail such as the hardware
manufacturer, firmware versions, and PCI IDs for NICs. Currently these values
are not used, but some dummy values need to be present. Use this file as-is.

site/$NEW_SITE/profiles/host/
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This directory contains a list of files that define ``drydock/HostProfile/v1``
elements. This example demonstrates layering of host profiles, as it defines a
``base_control_plane`` profile, which is inherited by another profile,
``rack6_control_plane``. Another host profile, ``base_data_plane`` is inherited by
``rack6_data_plane``.

This example demonstrates a typical use-case where data-plane nodes may have a
different bond configuration than control-plane nodes. If we added another rack
with its own CIDRs, we could inherit the same base host profiles to avoid
unnecessary duplication of information.

site/$NEW_SITE/profiles/host/base_control_plane.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An example host profile that defines a desired bonding configuration for control
plane nodes.

site/$NEW_SITE/profiles/host/rack6_control_plane.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An exapmle host profile that defines a desired bonding configuration for data-
plane nodes.

site/$NEW_SITE/baremetal/rack6.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File containing the ``drydock/BareMetalNode/v1`` resources for this site.

Populate with a BareMetalNode element for each bare metal node in the
environment. Setting highlights:

- metadata/name: Set to the desired hostname of the node
- data/host_profile: Set the host profile that will be applied to the node
- data/metadata/rack: Set the node's rack number / ID here
- data/metadata/tags: Tag with ``'masters'`` to designate nodes which will run the
  kubernetes master services, and with ``'workers'`` to designate nodes which will
  be kubernetes workers.
- data/addressing: Manually set unqiue IP network address for each node, using
  IPs within the static ranges specified for the same networks in
  ``rack06-network.yaml``.

site/$NEW_SITE/pki/pki-catalog.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File containing management IPs and hostnames of nodes. Each node in the
environment will require its own certificate definition for each of the defined
certificate authorities (kubernetes, kubernetes-etcd, kubernetes-etcd-peer,
calico-etcd, calico-etcd-peer, etc. Setting highlights:

- data/certificate_authorities/\*/certificates/common_name: Hostname of the node
  that is used to generate certificates. Ensure this matches what has been
  specified in ``rack06-baremetal.yaml`` for each node. In addition, there needs
  to be an entry for the ``genesis`` node.
- data/certificate_authorities/\*/certificates/document_name: Repeat the
  hostname of the node here.
- data/certificate_authorities/\*/certificates/hosts: A YAML list containing the
  node's hostname and IP address(es). Update hostname and IP information
  according to your environment.

site/$NEW_SITE/baremetal/bootactions.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File containing defined tasks to run after PXE boot (boot actions), so that
newly provisioned bare metal can retrieve their ``join-<NODE>.sh`` scripts and
run them, without a manual execution. (This script will join the node to the UCP
kubernetes cluster.) Setting highlights for ``promjoin`` bootaction:

- data/assets/location: URL where ``join-<NODE>.sh`` script will be found.
  Replace ``rack06_mgmt`` with the name of your management network, if different.

site/$NEW_SITE/software/charts/kubernetes/container-networking/etcd.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File containing calico-etcd certificates and certificate keys. Setting highlights:

- metadata/substitutions: Substitutions for Node names should be done as follows::

    -
      src:
        schema: pegleg/CommonAddresses/v1
        name: common-addresses
        path: .masters[0].hostname
      dest:
        path: .values.nodes[0].name
    -
      src:
        schema: pegleg/CommonAddresses/v1
        name: common-addresses
        path: .masters[1].hostname
      dest:
        path: .values.nodes[1].name
    -
      src:
        schema: pegleg/CommonAddresses/v1
        name: common-addresses
        path: .genesis.hostname
      dest:
        path: .values.nodes[2].name

The list does not need to include all nodes in your environment. Only nodes with
``calico-etcd`` set to ``enabled`` (as defined in host profile metadata) need to
be listed. Usually this is just the control plane nodes plus the genesis node.

Adjust the list of node names according to your environment. Cross-reference the
``site/$NEW_SITE/networks/common-address.yaml`` file to ensure the correct node
count.

Then for the same list of nodes, perform the tls cert and key substitutions for
both tls peer and tls client, e.g.::

    # Master node 1 certs
    -
      src:
        schema: deckhand/Certificate/v1
        name: calico-etcd-${MASTER_1_HOSTNAME}
        path: .
      dest:
        path: .values.nodes[0].tls.client.cert
    -
      src:
        schema: deckhand/CertificateKey/v1
        name: calico-etcd-${MASTER_1_HOSTNAME}
        path: .
      dest:
        path: .values.nodes[0].tls.client.key
    -
      src:
        schema: deckhand/Certificate/v1
        name: calico-etcd-${MASTER_1_HOSTNAME}-peer
        path: .
      dest:
        path: .values.nodes[0].tls.peer.cert
    -
      src:
        schema: deckhand/CertificateKey/v1
        name: calico-etcd-${MASTER_1_HOSTNAME}-peer
        path: .
      dest:
        path: .values.nodes[0].tls.peer.key

    # Master node 2 certs
    -
      src:
        schema: deckhand/Certificate/v1
        name: calico-etcd-${MASTER_2_HOSTNAME}
        path: .
      dest:
        path: .values.nodes[1].tls.client.cert
    -
      src:
        schema: deckhand/CertificateKey/v1
        name: calico-etcd-${MASTER_2_HOSTNAME}
        path: .
      dest:
        path: .values.nodes[1].tls.client.key
    -
      src:
        schema: deckhand/Certificate/v1
        name: calico-etcd-${MASTER_2_HOSTNAME}-peer
        path: .
      dest:
        path: .values.nodes[1].tls.peer.cert
    -
      src:
        schema: deckhand/CertificateKey/v1
        name: calico-etcd-${MASTER_2_HOSTNAME}-peer
        path: .
      dest:
        path: .values.nodes[1].tls.peer.key

    # Genesis certs
    -
      src:
        schema: deckhand/Certificate/v1
        name: calico-etcd-${GENESIS_HOSTNAME}
        path: .
      dest:
        path: .values.nodes[2].tls.client.cert
    -
      src:
        schema: deckhand/CertificateKey/v1
        name: calico-etcd-${GENESIS_HOSTNAME}
        path: .
      dest:
        path: .values.nodes[2].tls.client.key
    -
      src:
        schema: deckhand/Certificate/v1
        name: calico-etcd-${GENESIS_HOSTNAME}-peer
        path: .
      dest:
        path: .values.nodes[2].tls.peer.cert
    -
      src:
        schema: deckhand/CertificateKey/v1
        name: calico-etcd-${GENESIS_HOSTNAME}-peer
        path: .
      dest:
        path: .values.nodes[2].tls.peer.key

and substituting node hostnames where prompted by environment variable syntax.

site/$NEW_SITE/software/charts/ucp/ceph/ceph.yaml
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File containing site-level settings for Ceph. Ceph is deployed on control plane
nodes. Setting highlights:

- data/values/conf/storage/osd[*]/data/location: The block device that will be
  formatted by the Ceph chart and used as a Ceph OSD disk
- data/values/conf/storage/osd[*]/journal/location: The directory backing the
  ceph journal used by this OSD. Refer to the journal paradigm below.
- data/values/conf/pool/target/osd: Set this to match the number of OSD disks

Assumptions:

1. Ceph disks are not configured for any type of RAID (i.e., they are configured
   as JBOD if connected through a RAID controller). (If RAID controller does not
   support JBOD, put each disk in its own RAID-0 and enable RAID cache and
   write-back cache if the RAID controller supports it.)
2. Ceph disk mapping, disk layout, journal and OSD setup is the same across Ceph
   nodes (i.e. only one control plane host profile and hardware profile)
3. If doing a fresh install, disk are unlabeled or not labeled from a previous
   Ceph install, so that Ceph chart will not fail disk initialization

This document covers two Ceph journal deployment paradigms:

1. Servers with SSD/HDD mix (disregarding operating system disks).
2. Servers with no SSDs (disregarding operating system disks). In other words,
   exclusively spinning disk HDDs available for Ceph.

If you have an operating system available on the target hardware, you can
determine HDD and SSD layout with::

    lsblk -d -o name,rota

where a ``rota`` (rotational) value of ``1`` indicates a spinning HDD, and where
a value of ``0`` indicates non-spinning disk (i.e. SSD). (Note - Some SSDs still
report a value of ``1``, so it is best to go by your server specifications).

In case #1, the SSDs will be used for journals and the HDDs for OSDs.

For OSDs, pass in the whole block device (e.g., ``/dev/sdd``), and the Ceph
chart will take care of disk partitioning, formatting, mounting, etc.

For journals, divide the number of journal disks as evenly as possible between
the OSD disks. We will also use the whole block device, however we cannot pass
that block device to the Ceph chart like we can for the OSD disks.

Instead, the journal devices must be already partitioned, formatted, and mounted
prior to Ceph chart execution. This should be done by MaaS as part of the
Drydock host-profile being used for control plane nodes.

Consider the follow example where:

- /dev/sda is the operating system RAID-1 device
- /dev/sdb and /dev/sdc are SSDs
- /dev/sd[defg] are HDDs

Then, the data section of this file would look like::

    data:
      values:
        conf:
          storage:
            osd:
              - data:
                  type: block-logical
                  location: /dev/sdd
                journal:
                  type: directory
                  location: /var/lib/openstack-helm/ceph/journal0/journal-sdd
              - data:
                  type: block-logical
                  location: /dev/sde
                journal:
                  type: directory
                  location: /var/lib/openstack-helm/ceph/journal0/journal-sde
              - data:
                  type: block-logical
                  location: /dev/sdf
                journal:
                  type: directory
                  location: /var/lib/openstack-helm/ceph/journal1/journal-sdf
              - data:
                  type: block-logical
                  location: /dev/sdg
                journal:
                  type: directory
                  location: /var/lib/openstack-helm/ceph/journal1/journal-sdg
          pool:
            target:
              osd: 4

where the following mounts are setup by MaaS via Drydock host profile for the
control-plane nodes::

    /dev/sdb is mounted to /var/lib/openstack-helm/ceph/journal0
    /dev/sdc is mounted to /var/lib/openstack-helm/ceph/journal1

In case #2, Ceph best practice is to allocate journal space on all OSD disks.
The Ceph chart assumes this partitioning has been done beforehand. Ensure that
your control plane host profile is partitioning each disk between the Ceph OSD
and Ceph journal, and that it is mounting the journal partitions. (Drydock will
drive these disk layouts via MaaS provisioning). Note the mountpoints for the
journals and the partition mappings. Consider the following example where:

- /dev/sda is the operating system RAID-1 device
- /dev/sd[bcde] are HDDs

Then, the data section of this file will look similar to the following::

    data:
      values:
        conf:
          storage:
            osd:
              - data:
                  type: block-logical
                  location: /dev/sdb2
                journal:
                  type: directory
                  location: /var/lib/openstack-helm/ceph/journal0/journal-sdb
              - data:
                  type: block-logical
                  location: /dev/sdc2
                journal:
                  type: directory
                  location: /var/lib/openstack-helm/ceph/journal1/journal-sdc
              - data:
                  type: block-logical
                  location: /dev/sdd2
                journal:
                  type: directory
                  location: /var/lib/openstack-helm/ceph/journal2/journal-sdd
              - data:
                  type: block-logical
                  location: /dev/sde2
                journal:
                  type: directory
                  location: /var/lib/openstack-helm/ceph/journal3/journal-sde
          pool:
            target:
              osd: 4

where the following mounts are setup by MaaS via Drydock host profile for the
control-plane nodes::

    /dev/sdb1 is mounted to /var/lib/openstack-helm/ceph/journal0
    /dev/sdc1 is mounted to /var/lib/openstack-helm/ceph/journal1
    /dev/sdd1 is mounted to /var/lib/openstack-helm/ceph/journal2
    /dev/sde1 is mounted to /var/lib/openstack-helm/ceph/journal3

Generating site YAML files
^^^^^^^^^^^^^^^^^^^^^^^^^^

After constituent YAML configurations are finalized, use Pegleg to lint your
manifests, and resolve any issues that result from linting before proceeding::

    sudo sh -c "WORKSPACE=~/treasuremap/deployment_files ~/pegleg/tools/pegleg.sh \
      lint -p /workspace"

Note: ``P001`` linting errors are expected for missing certificates, as they are
not generated until the next section. You may suppress this warning by appending
``-x P001`` to the lint command.

Next, use pegleg to perform the merge that will yield the combined global +
site type + site YAML::

    mkdir -p ~/${NEW_SITE}_yaml
    sudo sh -c "WORKSPACE=~/treasuremap/deployment_files ~/pegleg/tools/pegleg.sh \
      site -p /workspace collect $NEW_SITE -s /workspace"
    mv ~/treasuremap/deployment_files/workspace.yaml ~/${NEW_SITE}_yaml/$NEW_SITE.yaml

Perform a visual inspection of the output. If any errors are discovered, you may
fix your manifests and re-run the ``lint`` and ``collect`` commands. It is this
output which will be used in subsequent steps.

Lastly, you should also perform a ``render`` on the documents. The resulting
render from Pegleg will not be used as input in subsequent steps, but is useful
for understanding what the document will look like once Deckhand has performed
all substitutions, replacements, etc. This is also useful for troubleshooting,
and addressing any Deckhand errors prior to submitting via Shipyard::

    sudo sh -c "WORKSPACE=~/treasuremap/deployment_files \
      ~/pegleg/tools/pegleg.sh site -p /workspace render $NEW_SITE"

Inspect the rendered document for any errors. If there are errors, address them
in your manifests and re-run this section of the document.

Building the Promenade bundle
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Clone the Promenade repo::

    cd ~
    git clone https://github.com/openstack/airship-promenade.git promenade

Refer to the ``data/charts/ucp/promenade/reference`` field in
``treasuremap/deployment_files/global/v1.0/software/config/versions.yaml``. If
this is a pinned reference (i.e., any reference that's not ``master``), then you
should checkout the same version of the Promenade repository. For example, if
the Promenade reference was ``86c3c11...`` in the versions file, checkout the
same version of the Promenade repo which was cloned previously::

    (cd promenade && git checkout 86c3c11)

Likewise, before running the ``simple-deployment.sh`` script, you should refer
to the ``data/images/ucp/promenade/promenade`` field in
``treasuremap/deployment_files/global/v1.0/software/config/versions.yaml``. If
there is a pinned reference (i.e., any image reference that's not ``latest``),
then this reference should be used to set the ``IMAGE_PROMENADE`` environment
variable. For example, if the Promenade image was pinned to
``artifacts-aic.atlantafoundry.com/att-comdev/promenade@sha256:d30397f...`` in
the versions file, then export the previously mentioned environment variable::

    export IMAGE_PROMENADE=artifacts-aic.atlantafoundry.com/att-comdev/promenade@sha256:d30397f...

Now, create an output directory for Promenade bundles and run the
``simple-deployment.sh`` script::

    mkdir ~/${NEW_SITE}_bundle
    sudo promenade/tools/simple-deployment.sh ~/${NEW_SITE}_yaml ~/${NEW_SITE}_bundle

Estimated runtime: About **1 minute**

After the bundle has been successfully created, copy the generated certificates
into your site definition. Ex::

    mkdir -p ~/treasuremap/deployment_files/site/$NEW_SITE/secrets/certificates/
    sudo cp ~/${NEW_SITE}_bundle/certificates.yaml \
    ~/treasuremap/deployment_files/site/$NEW_SITE/secrets/certificates/certificates.yaml

Commit the entire site configuration to the source control system identified in
the `Pre-req`_ section to track configuration documents.

Genesis node
------------

Initial setup
^^^^^^^^^^^^^

Start with a manual install of Ubuntu 16.04 on the node you wish to use to seed
the rest of your environment. Ensure the host has outbound internet access and
can resolve public DNS entries.

Ensure that the hostname matches the hostname specified in the Genesis.yaml file
used in the previously generated configuration. If it does not, then either
change the hostname of the node to match the configuration documents, or re-
generate the configuration with the correct hostname.

Installing matching kernel version
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Install the same kernel version on the Genesis host that MaaS will use to deploy
new baremetal nodes.

In order to do this, first you must determine the kernel version that will be
deployed to those nodes. Start by looking at the host profile definition used to
deploy other control plane nodes by searching for ``control-plane: enabled``.
Most likely this will be a file under ``global/v1.0/profiles/host``. In this
file, find the kernel info - e.g.::

    platform:
        image: 'xenial'
        kernel: 'hwe-16.04'

In this case, it is the hardware enablement kernel for 16.04. To find the exact
kernel version that will be deployed, we must look into the simple-stream image
cache that will be used by MaaS to deploy nodes with. Locate the ``data/images/ucp/maas/maas_cache``
key in within ``global/v1.0/software/config/versions.yaml``. This is the image
that you will need to fetch, using a node with docker installed that has access
and can reach the site/location hosting the image. For example, from the **build
node**, the command would take the form::

    sudo docker pull YOUR_SSTREAM_IMAGE

Then, create a container from that image::

    sudo sh -c "$(docker images | grep sstream | head -1 | awk '{print $1}' > image_name)"
    sudo docker create --name sstream $(cat image_name)

Then use the container ID returned from the last command as follows::

    sudo docker start sstream
    sudo docker exec -it sstream /bin/bash

In the container, install the ``file`` package. Define any proxy environment
variables needed for your environment to reach public ubuntu package repos::

    sudo apt-get update
    sudo apt-get -y install file

In the container, ``cd`` to the following location (substituting for the platform
image and platform kernel identified in the host profile previously, and choosing
the folder corresponding to the most current date if more than one are available)
and run the ``file`` command on the ``boot-kernel`` file::

    cd /var/www/html/maas/images/ephemeral-v3/daily/PLATFORM_IMAGE/amd64/LATEST_DATE/PLATFORM_KERNEL/generic
    file boot-kernel

This will produce the complete kernel version. E.g.::

    Linux kernel x86 boot executable bzImage, version 4.13.0-43-generic (buildd@lcy01-amd64-029) #48~16.04.1-Ubuntu S, RO-rootFS, swap_dev 0x7, Normal VGA

In this example, the kernel version is ``4.13.0-43-generic``. Now install the
matching kernel on the Genesis host (make sure to run on Genesis host, not the
build host)::

    sudo apt-get install 4.13.0-43-generic

Check the installed packages on the genesis host with ``dpkg --list``. If there
are any later kernel versions installed, remove them with ``sudo apt-get remove``,
so that the newly install kernel is the latest available.

Lastly if you wish to cleanup your build node, you may run the following::

    exit # (to quit the container)
    sudo docker stop sstream
    sudo docker rm sstream
    sudo docker image rm $(cat image_name)
    sudo rm image_name

Install ntpdate/ntp
^^^^^^^^^^^^^^^^^^^

Install and run ntpdate, to ensure a reasonably sane time on genesis host before
proceeding::

    sudo apt -y install ntpdate
    sudo ntpdate ntp.ubuntu.com

If your network policy does not allow time sync with external time sources,
specify a local NTP server instead of using ``ntp.ubuntu.com``.

Then, install the NTP client::

    sudo apt -y install ntp

Add the list of NTP servers specified in ``data/ntp/servers_joined`` in file
``site/$NEW_SITE/networks/common-address.yaml`` to ``/etc/ntp.conf`` as follows::

    pool NTP_SERVER1 iburst
    pool NTP_SERVER2 iburst
    (repeat for each NTP server with correct NTP IP or FQDN)

Then, restart the NTP service::

    sudo service ntp restart

Refer to `troubleshooting <operations.html#Troubleshooting time sync issues>`__ to ensure that the NTP stats are healthy on genesis
node before proceeding. If you cannot get good time to your selected time
servers, consider using alternate time sources for your deployment.

Disable the apparmor profile for ntpd::

    sudo ln -s /etc/apparmor.d/usr.sbin.ntpd /etc/apparmor.d/disable/
    sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.ntpd

This prevents an issue with the MaaS containers, which otherwise get permission
denied errors from apparmor when the MaaS container tries to leverage libc6 for
/bin/sh when MaaS container ntpd is forcefully disabled.

Setup Ceph Journals
^^^^^^^^^^^^^^^^^^^

Until genesis node reprovisioning is implemented, it is necessary to manually
perform host-level disk partitioning and mounting on the genesis node, for
activites that would otherwise have been addressed by a bare metal node
provision via Drydock host profile data by MaaS.

Assuming your genesis HW matches the HW used in your control plane host profile,
you should manually apply to the genesis node the same Ceph partitioning (OSDs &
journals) and formatting + mounting (journals only) as defined in the control
plane host profile. See
``treasuremap/deployment_files/global/v1.0/profiles/host/base_control_plane.yaml``.

For example, if we have a journal SSDs ``/dev/sdb`` on the genesis node, then
use the ``cfdisk`` tool to format it::

    sudo cfdisk /dev/sdb

Then:

1. Select ``gpt`` label for the disk
2. Select ``New`` to create a new partition
3. If scenario #1 applies in `site/$NEW_SITE/software/charts/ucp/ceph/ceph.yaml`_,
   then accept default partition size (entire disk). If scenario #2 applies,
   then only allocate as much space as defined in the journal disk partitions
   mounted in the control plane host profile.
4. Select ``Write`` option to commit changes, then ``Quit``
5. If scenario #2 applies, create a second partition that takes up all of the
   remaining disk space. This will be used as the OSD partition (``/dev/sdb2``).

Then, construct an XFS filesystem on the journal partition::

    sudo mkfs.xfs /dev/sdb1

Create a directory as mount point for ``/dev/sdb1`` to match those defined in the same host profile ceph journals::

    sudo mkdir -p /var/lib/openstack-helm/ceph/journal0/journal-sdb1

Use the ``blkid`` command to get the UUID for ``/dev/sdb1``, then populate
``/etc/fstab`` accordingly. Ex::

    sudo sh -c 'echo "UUID=01234567-ffff-aaaa-bbbb-abcdef012345 /var/lib/openstack-helm/ceph/journal0/journal-sdb1 xfs defaults 0 0" >> /etc/fstab'

Repeat all preceeding steps in this section for each journal device in the Ceph
cluster. After this is completed for all journals, mount the partitions::

    sudo mount -a

Promenade bootstrap
^^^^^^^^^^^^^^^^^^^

Copy the ``genesis.sh`` script generated in the ``promenade/build`` directory
on the build node to the genesis node. Then, run the script as sudo on the
genesis node::

    sudo ./genesis.sh

Estimated runtime: **40m**

In the event of failures, refer to `genesis troubleshooting <https://promenade.readthedocs.io/en/latest/troubleshooting/genesis.html>`_.

Following completion, run the ``validate-genesis.sh`` script to ensure correct
provisioning of the genesis node::

    sudo ./validate-genesis.sh

Estimated runtime: **2m**

Deploy Site with Shipyard
^^^^^^^^^^^^^^^^^^^^^^^^^

Start by cloning the shipyard repository to the Genesis node::

    git clone https://github.com/openstack/airship-shipyard shipyard

Refer to the ``data/charts/ucp/shipyard/reference`` field in
``treasuremap/deployment_files/global/v1.0/software/config/versions.yaml``. If
this is a pinned reference (i.e., any reference that's not ``master``), then you
should checkout the same version of the Shipyard repository. For example, if
the Shipyard reference was ``7046ad3...`` in the versions file, checkout the
same version of the Shipyard repo which was cloned previously::

    (cd shipyard && git checkout 7046ad3)

Likewise, before running the ``deckhand_load_yaml.sh`` script, you should refer
to the ``data/images/ucp/shipyard/shipyard`` field in
``treasuremap/deployment_files/global/v1.0/software/config/versions.yaml``. If
there is a pinned reference (i.e., any image reference that's not ``latest``),
then this reference should be used to set the ``SHIPYARD_IMAGE`` environment
variable. For example, if the Shipyard image was pinned to
``artifacts-aic.atlantafoundry.com/att-comdev/shipyard@sha256:dfc25e1...`` in
the versions file, then export the previously mentioned environment variable::

    export SHIPYARD_IMAGE=artifacts-aic.atlantafoundry.com/att-comdev/shipyard@sha256:dfc25e1...

Export valid login credentials for one of the UCP Keystone users defined for the
site. Currently there is no authorization checks in place, so the credentials
for any of the site-defined users will work. For example, we can use the
``shipyard`` user, with the password that was defined in
``site/$NEW_SITE/secrets/passphrases/ucp_shipyard_keystone_password.yaml``. Ex::

    export OS_USERNAME=shipyard
    export OS_PASSWORD=46a75e4...

(Note: Default auth variables are defined `here <https://github.com/openstack/airship-shipyard/blob/master/tools/shipyard_docker_base_command.sh>`_, and should otherwise be
correct, barring any customizations of these site parameters).

Next, run the deckhand_load_yaml.sh script as follows::

    sudo ./shipyard/tools/deckhand_load_yaml.sh $REGION $PATH_TO_ALL_YAMLS

where REGION is the region name (as defined in drydock.yaml), and PATH_TO_ALL_YAMLS
is the path to a directory containing all YAML files generated in previous
sections.

Estimated runtime: **3m**

Troubleshooting placeholder

Now deploy the site with shipyard::

    sudo ./shipyard/tools/deploy_site.sh

Estimated runtime: **1h30m**

Troubleshooting placeholder

The message ``Site Successfully Deployed`` is the expected output at the end of a
successful deployment. In this example, this means that UCP and OSH should be
fully deployed.

Disable password-based login on Genesis
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Before proceeding, verify that your SSH access to the Genesis node is working
with your SSH key (i.e., not using password-based authentication).

Then, disable password-based SSH authentication on Genesis in
``/etc/ssh/sshd_config`` by uncommenting the ``PasswordAuthentication`` and
setting its value to ``no``. Ex::

    PasswordAuthentication no

Then, restart the ssh service::

    sudo systemctl restart ssh
