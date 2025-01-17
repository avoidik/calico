---
title: Encrypt in-cluster pod traffic
description: Enable WireGuard for state-of-the-art cryptographic security between pods for Calico clusters.
---

### Big picture

Enable WireGuard to secure on-the-wire, in-cluster pod traffic in a {{site.prodname}} cluster.

### Value

{{ site.prodname }} automatically creates and manages WireGuard tunnels between nodes providing transport-level security for on- the-wire, in-cluster pod traffic. WireGuard provides {% include open-new-window.html text='formally verified' url='https://www.wireguard.com/formal-verification/' %} secure and {% include open-new-window.html text='performant tunnels' url='https://www.wireguard.com/performance/' %} without any specialized hardware. For a deep dive in to WireGuard implementation, see this {% include open-new-window.html text='whitepaper' url='https://www.wireguard.com/papers/wireguard.pdf' %}.

### Features

This how-to guide uses the following {{site.prodname}} features:

- **Felix configuration resource** with WireGuard configuration parameters

### Before you begin...

- [Install and configure calicoctl]({{site.baseurl}}/getting-started/clis/calicoctl/install)
- Verify the operating system(s) running on the nodes in the cluster {% include open-new-window.html text='support WireGuard' url='https://www.wireguard.com/install/' %}.
- WireGuard in {{site.prodname}} requires node IP addresses to establish secure tunnels between nodes. {{site.prodname}} can automatically detect IP address of a node using [IP Setting]({{site.baseurl}}/reference/node/configuration#ip-setting) and [IP autodetection method]({{site.baseurl}}/reference/node/configuration#ip-autodetection-methods) in [calico/node]({{site.baseurl}}/reference/node/configuration) resource.
    - Set `IP` (or `IP6`) environment variable to `autodetect`.
    - Set `IP_AUTODETECTION_METHOD` (or `IP6_AUTODETECTION_METHOD`) to an appropriate value. If there are multiple interfaces on a node, set the value to detect the IP address of the primary interface.

> **Note**: WireGuard in {{site.prodname}} does not support IPv6 at this time. Also, encryption using WireGuard is not supported if not using Calico CNI (e.g. managed Kubernetes platforms EKS, AKS and GKE).
{: .alert .alert-info}

### How to

- [Install WireGuard](#install-wireguard)
- [Enable WireGuard for a cluster](#enable-wireguard-for-a-cluster)
- [Disable WireGuard for an individual node](#disable-wireguard-for-an-individual-node)
- [Verify configuration](#verify-configuration)
- [Disable WireGuard for a cluster](#disable-wireguard-for-a-cluster)

#### Install WireGuard

Install WireGuard on cluster nodes using {% include open-new-window.html text='instructions for your operating system' url='https://www.wireguard.com/install/' %}. Note that you may need to reboot your nodes after installing WireGuard to make the kernel modules available on your system.

   Use the following instructions for these operating systems that are not listed on the WireGuard installation page.
{% tabs %}
<label:EKS,active:true>
<%
To install WireGuard on the default Amazon Machine Image (AMI):

   ```bash
   sudo yum install kernel-devel-`uname -r` -y
   sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y
   sudo curl -o /etc/yum.repos.d/jdoss-wireguard-epel-7.repo https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
   sudo yum install wireguard-dkms wireguard-tools -y
   ```
%>
<label:OpenShift>
<%
To install WireGuard for OpenShift v4.6:

  This approach uses kernel modules via container installation as outlined here {% include open-new-window.html text='atmoic wireguard' url='https://github.com/projectcalico/atomic-wireguard' %} 

   1. Create MachineConfig for WireGuard on your local machine.
   ```bash
   cat <<EOF > mc-wg-worker.yaml
   apiVersion: machineconfiguration.openshift.io/v1
   kind: MachineConfig
   metadata:
     labels:
       machineconfiguration.openshift.io/role: worker
     name: 10-kvc-wireguard-kmod
   spec:
     config:
   EOF
   ```

   2. Create base {% include open-new-window.html text='Ignition' url='https://github.com/coreos/ignition' %} config.
   ```bash
   cat <<EOF > ./wg-config.ign
   {
     "ignition": { "version": "2.2.0" },
     "systemd": {
       "units": [{
         "name": "require-kvc-wireguard-kmod.service",
         "enabled": true,
         "contents": "[Unit]\nRequires=kmods-via-containers@wireguard-kmod.service\n[Service]\nType=oneshot\nExecStart=/usr/bin/true\n\n[Install]\nWantedBy=multi-user.target"
       }]
     }
   }
   EOF
   ```

   3. Download and configure the tools needed for kmods.
   ```bash
   FAKEROOT=$(mktemp -d)
   git clone https://github.com/kmods-via-containers/kmods-via-containers
   cd kmods-via-containers
   make install DESTDIR=${FAKEROOT}/usr/local CONFDIR=${FAKEROOT}/etc/
   cd ..
   git clone https://github.com/tigera/kvc-wireguard-kmod
   cd kvc-wireguard-kmod
   make install DESTDIR=${FAKEROOT}/usr/local CONFDIR=${FAKEROOT}/etc/
   cd ..
   ```

   4. You must then set the URLs for the `KERNEL_CORE_RPM`, `KERNEL_DEVEL_RPM` and `KERNEL_MODULES_RPM` packages in the conf file `$FAKEROOT/etc/kvc/wireguard-kmod.conf`. You can determine the host kernel version to use in the URL by running `uname -r` on a host in your cluster. You can find links to  official packages in your {% include open-new-window.html text='Red Hat subscription' url='https://access.redhat.com/downloads/content/package-browser' %} 


   5. Get RHEL Entitlement data from your own RHEL8 system from a host in your cluster.
   ```bash
   tar -czf subs.tar.gz /etc/pki/entitlement/ /etc/rhsm/ /etc/yum.repos.d/redhat.repo
   ```

   6. Copy the `subs.tar.gz` file to your workspace and then extract the contents using the following command.
   ```bash
   tar -x -C ${FAKEROOT} -f subs.tar.gz
   ```

   7. Get filetranspiler to generate the usable machine-config.
   ```bash
   git clone https://github.com/ashcrow/filetranspiler
   ./filetranspiler/filetranspile -i ./wg-config.ign -f ${FAKEROOT} --format=yaml --dereference-symlinks | sed 's/^/     /' | (cat mc-wg-worker.yaml -) > mc-wg.yaml
   ```

   8. With the KUBECONFIG set for your cluster, run the following command to apply the MachineConfig which will install WireGuard across your cluster.
   ```bash
   oc create -f mc-wg.yaml
   ```
%>
{% endtabs %}

#### Enable WireGuard for a cluster

   > **Note**: Nodes that do not support WireGuard will not be secured by WireGuard tunnels, even if traffic running on the node to and from the pods goes to nodes that do support WireGuard.
   {: .alert .alert-info}

Enable WireGuard encryption across all the nodes using the following command.

```bash
calicoctl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'
```

For OpenShift, add the Felix configuration with WireGuard enabled [under custom resources]({{site.baseurl}}/getting-started/openshift/installation#optionally-provide-additional-configuration).

   > **Note**: The above command can be used to change other WireGuard attributes. For a list of other WireGuard parameters and configuration evaluation, see the [Felix configuration]({{site.baseurl}}/reference/resources/felixconfig#felix-configuration-definition).
   {: .alert .alert-info}

We recommend that you review and modify the MTU used by Calico networking when WireGuard is enabled to increase network performance. Follow the instructions in the [Configure MTU to maximize network performance]({{site.baseurl}}/networking/mtu) guide to set the MTU to a value appropriate for your network.

#### Disable WireGuard for an individual node

To disable WireGuard on a specific node with WireGuard installed, modify the node-specific Felix configuration. For example:

  ```bash
calicoctl patch felixconfiguration node.<Node-Name> --type='merge' -p '{"spec":{"wireguardEnabled":false}}'
  ```

To disable encryption for pod traffic on node `my-node`, use the following command:

  ```bash
calicoctl patch felixconfiguration node.my-node --type='merge' -p '{"spec":{"wireguardEnabled":false}}'
  ```

With the above command, Calico will not encrypt any of the pod traffic to or from node `my-node`.

#### Verify configuration

To verify that the nodes are configured for WireGuard encryption, check the node status set by Felix using `calicoctl`. For example:

   ```
   $ calicoctl get node <NODE-NAME> -o yaml
   ...
   status:
     ...
     wireguardPublicKey: jlkVyQYooZYzI2wFfNhSZez5eWh44yfq1wKVjLvSXgY=
     ...
   ```

#### Disable WireGuard for a cluster

To disable WireGuard on all nodes modify the default Felix configuration. For example:

  ```bash
  calicoctl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":false}}'
  ```

### Above and beyond

- [Secure Calico component communications]({{site.baseurl}}/security/comms)
- [Configure MTU to maximize network performance]({{site.baseurl}}/networking/mtu)
