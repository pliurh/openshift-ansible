# OpenShift Multi-Network PoC

This is a proof of concept for using [multus-cni](https://github.com/intel/multus-cni) as a "meta-plugin" (a CNI plugin which can delegate to other CNI plugins in order to facilitate the attachment of multiple network interfaces to a pod). In particular, this implementation shown here delgates the first network interface attached to a pod to OpenShift SDN.

In particular, this deploys a reference implementation of the [Kubernetes Network Custom Resource Definition De-facto Standard](https://docs.google.com/document/d/1Ny03h6IDVy_e_vmElOqR7UdTPAG_RNydhVE1Kx54kFQ/edit#heading=h.hylsbqoj5fxd) as put forward by the [Network Plumbing Working Group](https://docs.google.com/document/d/1oE93V3SgOGWJ4O1zeD1UmpeToa0ZiiO6LqRAmZBPFWM/edit).

## Disclaimer

This is currently a Work-In-Progress, and generally should be considered a proof-of-concept, and a stub for further work.

Some elements are work-arounds to account for gaps that otherwise need to be filled or decided on.

## Other code references

* This essentially cuts off the Node go code's [way of writing a configuration file](https://github.com/openshift/origin/blob/master/pkg/network/node/node.go#L401-L408). 
    - This may cause un-anticipated issues with how the node likes to say it's ready by properly waiting for OpenShift SDN.
    - It cuts it off by using an alphabetically first CNI configuration name in `/etc/cni/net.d/`, the original starts named as `80-[...].conf` this configuration is named `70-[...].conf`.
* This builds upon [Dan William's PR #7825](https://github.com/openshift/openshift-ansible/pull/7825)
* The original recipe for the work on this playbook is in this [gist by @dougbtv](https://gist.github.com/dougbtv/c18fc70be9afdba703d8a397c30e57ed)

## Giving it a whirl!

This assumes 3 CentOS atomic Hosts.

Firstly, use an inventory file such as:

```
# Last known working @ openshift-ansible commit
# [root@droctagon3 openshift-ansible]# git describe
# openshift-ansible-3.10.0-0.63.0-98-g7a136e9
# [root@droctagon3 openshift-ansible]# git rev-parse HEAD
# 7a136e99c33927a00f2f3a58b2de5e170e880252

threeten-infra.test.example.com ansible_host=192.168.1.140
threeten-master.test.example.com ansible_host=192.168.1.225
threeten-node1.test.example.com ansible_host=192.168.1.32

[masters]
threeten-master.test.example.com

[etcd]
threeten-master.test.example.com

[nodes]
threeten-master.test.example.com openshift_node_group_name="node-config-master"
threeten-infra.test.example.com openshift_node_group_name="node-config-infra"
threeten-node1.test.example.com openshift_node_group_name="node-config-compute"

[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
openshift_deployment_type=origin
openshift_master_default_subdomain=apps.test.example.com
openshift_master_cluster_hostname=threeten-master.test.example.com
openshift_disable_check=disk_availability,memory_availability
openshift_enable_docker_excluder=False
# get the latest base url with: 
# $ curl https://storage.googleapis.com/origin-ci-test/releases/openshift/origin/release-3.10/.latest-rpms
openshift_additional_repos=[{'id': 'openshift-origin-ci', 'name': 'OpenShift-Origin-CI-repostory--caution-not-release', 'baseurl': 'https://storage.googleapis.com/origin-ci-test/logs/test_branch_origin_extended_conformance_gce_310/23/artifacts/rpms', 'enabled': 1, 'gpgcheck': 0}]
debug_level=2
ansible_ssh_user=centos
ansible_become=yes
debug_level=2
ansible_ssh_private_key_file=/root/.ssh/id_vm_rsa
```

*NOTE*: The additional repos may need to be updated periodically. You can get the base URL by curl'ing https://storage.googleapis.com/origin-ci-test/releases/openshift/origin/release-3.10/.latest-rpms

Then run openshift-ansible like:

```
$ ansible-playbook -i inventory/above.inventory playbooks/prerequisites.yml playbooks/deploy_cluster.yml playbooks/openshift-multinetwork/config.yml
```

Log into your master and check that it generally looks OK. Then, you'll need to create an addition config, in this case we create a macvlan config.

```
$ cat <<EOF | kubectl create -f -
apiVersion: "kubernetes.cni.cncf.io/v1"
kind: Network
metadata:
  name: macvlan-conf
spec: 
  config: '{
            "cniVersion": "0.3.0",
            "type": "macvlan",
            "master": "eth0",
            "mode": "bridge",
            "ipam": {
                "type": "host-local",
                "subnet": "192.168.1.0/24",
                "rangeStart": "192.168.1.200",
                "rangeEnd": "192.168.1.216",
                "routes": [
                    { "dst": "0.0.0.0/0" }
                ],
                "gateway": "192.168.1.1"
            }
        }'
EOF
```

Create an example pod.

```
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: bothpod
  annotations:
    kubernetes.v1.cni.cncf.io/networks: macvlan-conf
spec:
  containers:
  - name: bothpod
    image: dougbtv/nginx-toolbox
    ports:
    - containerPort: 8080
EOF
```

Watch it come up...

```
$ watch -n1 oc get pods -o wide
$ watch -n1 oc describe pod bothpod
```

And then you should be able to verify that there's multiple interfaces in the pod...

```
$ oc exec -it bothpod -- ip a
```
