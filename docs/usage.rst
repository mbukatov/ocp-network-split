.. _usage:

======================
 Single Cluster Usage
======================

Assumptions about cluster zones
===============================

A `k8s zone`_ is a set of cluster nodes with the same value of `k8s label`_ key
``topology.kubernetes.io/zone``, see an example of zone ``data-1``:

.. code-block:: console

   $ oc get nodes -l topology.kubernetes.io/zone=data-1
   NAME              STATUS   ROLES    AGE     VERSION
   compute-0         Ready    worker   7d14h   v1.20.0+bafe72f
   compute-1         Ready    worker   7d14h   v1.20.0+bafe72f
   compute-2         Ready    worker   7d14h   v1.20.0+bafe72f
   control-plane-0   Ready    master   7d14h   v1.20.0+bafe72f

We assume that there are 3 zones in the cluster, and that every node belongs to
some zone, eg:

.. code-block:: console

   $ oc get nodes -L topology.kubernetes.io/zone
   NAME              STATUS   ROLES    AGE   VERSION           ZONE
   compute-0         Ready    worker   8d    v1.20.0+bafe72f   data-1
   compute-1         Ready    worker   8d    v1.20.0+bafe72f   data-1
   compute-2         Ready    worker   8d    v1.20.0+bafe72f   data-1
   compute-3         Ready    worker   8d    v1.20.0+bafe72f   data-2
   compute-4         Ready    worker   8d    v1.20.0+bafe72f   data-2
   compute-5         Ready    worker   8d    v1.20.0+bafe72f   data-2
   control-plane-0   Ready    master   8d    v1.20.0+bafe72f   data-1
   control-plane-1   Ready    master   8d    v1.20.0+bafe72f   data-2
   control-plane-2   Ready    master   8d    v1.20.0+bafe72f   arbiter

There is no limitation on the design of cluster zones or their names
(values of ``topology.kubernetes.io/zone`` label key). The ocp-network-split
references zones under single letter names (such as ``a``, ``b`` ... see
:py:const:`ocpnetsplit.zone.ZONES`), so that you will just need to
create mapping between ocp-network-split names and actual zone names as shown
in the following sections.

.. _`k8s zone`: https://kubernetes.io/docs/reference/labels-annotations-taints/#topologykubernetesiozone
.. _`k8s label`: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/

External zone 
=============

Besides normal cluster zones, there is a special zone ``x`` which represents
external services running outside of a cluster. Specifying list of IP addresses
for ``x`` zone allows you to block traffic to these IP addresses in both
directions later.

Command line tools
==================

There are also 2 command line tools:

- ``ocp-network-split-setup``: based on given zone name assignment, it fetches
  IP addresses of all nodes for every zone (to create env file with zone
  configuration), and creates ``MachineConfig`` yaml file to deploy the zone
  configuration along with firewall script and systemd unit files to every node
  of the cluster. This is done only once.

- ``ocp-network-split-sched``: schedules given network split configuration
  which will start at given time and stop after given number of minutes.

Setting up network split
------------------------

Let's have a look how the zone configuration generated by the setup script
looks like (the example also shows how to define zone name mapping):

.. code-block:: console

   $ ocp-network-split-setup -a arbiter -b data-1 -c data-2 --print-env-only
   ZONE_A="198.51.100.36"
   ZONE_B="198.51.100.127 198.51.100.158 198.51.100.160 198.51.100.163"
   ZONE_C="198.51.100.103 198.51.100.162 198.51.100.65 198.51.100.98"

If this looks good, we can go on and create ``MachineConfig`` yaml file, which
you can inspect as well.

.. code-block:: console

    $ ocp-network-split-setup -a arbiter -b data-1 -c data-2 -o network-split.yaml
    $ head network-split.yaml
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      labels:
        machineconfiguration.openshift.io/role: master
      name: 99-master-network-split
    spec:
      config:
        ignition:
          version: 3.1.0

Then you can use ``oc create`` to deploy the configuration:

.. code-block:: console

    $ oc create -f network-split.yaml
    machineconfig.machineconfiguration.openshift.io/95-master-network-zone-config created
    machineconfig.machineconfiguration.openshift.io/99-master-network-split created
    machineconfig.machineconfiguration.openshift.io/95-worker-network-zone-config created
    machineconfig.machineconfiguration.openshift.io/99-worker-network-split created

Note that there are 2 ``MachineConfig`` resources for each node type:
network-zone-config provides zone configuration and can be shared with latency
machine config (see bellow) while network-split provides firewall split
scripts.

Introducing additional network latency
--------------------------------------

If you need to configure additional artificial network latency between nodes
from different cluster zones, you can specify the desired one way latency in
milliseconds via ``--latency`` option.

In the following example, we are using extremely large number of 106 ms for
demonstration purposes, which will give us full round trip latency of 212 ms:

.. code-block:: console

    $ ocp-network-split-setup -a arbiter -b data-1 -c data-2 --latency 106 -o split-latency.yaml
    $ oc create -f split-latency.yaml
    machineconfig.machineconfiguration.openshift.io/95-master-network-zone-config created
    machineconfig.machineconfiguration.openshift.io/99-master-network-latency created
    machineconfig.machineconfiguration.openshift.io/99-master-network-split created
    machineconfig.machineconfiguration.openshift.io/95-worker-network-zone-config created
    machineconfig.machineconfiguration.openshift.io/99-worker-network-latency created
    machineconfig.machineconfiguration.openshift.io/99-worker-network-split created

The additional latency is configured via systemd service which is enabled to
start during boot, so that the latency is effective almost immediately and will
remain applied even after node reboot.

The only way to remove it is to delete it's machineconfig resources.

Scheduling network split
------------------------

When the machine config is applied (check ``oc get mcp`` if both pools are
updated), we can schedule 5 minute long network split of particular
configuration ``ab`` (cutting connection between zones ``a`` and ``b``) at
given time:

.. code-block:: console

    $ ocp-network-split-sched ab -t 2021-04-09T16:30 --split-len 5

When the time details are omitted, the sched script will just list net split
timers for given split configuration on all nodes. In the following example,
we can see one split was schedule 26 minutes ago, while another is going to
happen in about 4 minutes:

.. code-block:: console

    $ ocp-network-split-sched ab
    node/compute-0
    NEXT                         LEFT          LAST                         PASSED    UNIT                                    ACTIVATES
    Fri 2021-04-09 14:30:00 UTC  3min 50s left n/a                          n/a       network-split-ab-setup@1617978600.timer network-split@ab.service
    n/a                          n/a           Fri 2021-04-09 14:00:00 UTC  26min ago network-split-ab-setup@1617976800.timer network-split@ab.service
    
    node/compute-1
    NEXT                         LEFT          LAST                         PASSED    UNIT                                    ACTIVATES
    Fri 2021-04-09 14:30:00 UTC  3min 48s left n/a                          n/a       network-split-ab-setup@1617978600.timer network-split@ab.service
    n/a                          n/a           Fri 2021-04-09 14:00:00 UTC  26min ago network-split-ab-setup@1617976800.timer network-split@ab.service
    
    ... rest of the output is ommited ...

You can schedule multiple splits in advance, or wait for one network split to
end before going on with another one.
