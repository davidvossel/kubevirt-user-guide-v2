Installation
============

KubeVirt is a virtualization add-on to Kubernetes and this guide assumes
that a Kubernetes cluster is already installed.

If installed on OKD, the web console is extended for management of
virtual machines.

Requirements
------------

A few requirements need to be met before you can begin:

-   [Kubernetes](https://kubernetes.io) cluster
    ([OpenShift](https://github.com/openshift/origin), Tectonic)

-   `kubectl` client utility

-   `git`

### Minimum Cluster Requirements

Kubernetes 1.10 or later is required to run KubeVirt.

In addition it can be that feature gates need to be opened.

### Cluster Configuration

The Kubernetes apiserver needs to be configured to allow privileged pods
since we run a privileged Daemon Set on every node. The required flag is
`--allow-privileged=true`.

#### Runtime

KubeVirt is currently supported on the following container runtimes:

-   docker

-   crio (with runv)

Other container runtimes, which do not use virtualization features,
should work too. However, they are not tested.

### Virtualization support

There are several distributions of Kubernetes, you need to decide on one
and deploy it.

Hardware with virtualization support is recommended. You can use
virt-host-validate to ensure that your hosts are capable of running
virtualization workloads:

    $ virt-host-validate qemu
      QEMU: Checking for hardware virtualization                                 : PASS
      QEMU: Checking if device /dev/kvm exists                                   : PASS
      QEMU: Checking if device /dev/kvm is accessible                            : PASS
      QEMU: Checking if device /dev/vhost-net exists                             : PASS
      QEMU: Checking if device /dev/net/tun exists                               : PASS
    ...

#### Software emulation

If hardware virtualization is not available, then a [software emulation
fallback](https://github.com/kubevirt/kubevirt/blob/master/docs/software-emulation.md)
can be enabled using:

    $ kubectl create namespace kubevirt
    $ kubectl create configmap -n kubevirt kubevirt-config \
        --from-literal debug.useEmulation=true

This ConfigMap needs to be created before deployment or the
virt-controller deployment has to be restarted.

### Hugepages support

For hugepages support you need at least Kubernetes version `1.9`.

#### Enable feature-gate

To enable hugepages on Kubernetes, check the [official
documentation](https://kubernetes.io/docs/tasks/manage-hugepages/scheduling-hugepages/).

To enable hugepages on OKD, check the [official
documentation](https://docs.openshift.org/3.9/scaling_performance/managing_hugepages.html#huge-pages-prerequisites).

#### Pre-allocate hugepages on a node

To pre-allocate hugepages on boot time, you will need to specify
hugepages under kernel boot parameters `hugepagesz=2M hugepages=64` and
restart your machine.

You can find more about hugepages under [official
documentation](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt).

Cluster side add-on deployment
------------------------------

### Core components

Once Kubernetes is deployed, you will need to deploy the KubeVirt
add-on.

The installation uses the KubeVirt operator, which manages lifecycle of
all KubeVirt core components:

    # Deploy the KubeVirt operator
    $ kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/v0.24.0/kubevirt-operator.yaml
    # Create the KubeVirt CR (instance deployment request)
    $ kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/v0.24.0/kubevirt-cr.yaml
    # wait until all KubeVirt components is up
    $ kubectl -n kubevirt wait kv kubevirt --for condition=Available

> Note: Prior to release v0.20.0 the condition for the `kubectl wait`
> command was named "Ready" instead of "Available"

All new components will be deployed under the `kubevirt` namespace:

    kubectl get pods -n kubevirt
    NAME                                           READY     STATUS        RESTARTS   AGE
    virt-api-6d4fc3cf8a-b2ere                      1/1       Running       0          1m
    virt-controller-5d9fc8cf8b-n5trt               1/1       Running       0          1m
    virt-handler-vwdjx                             1/1       Running       0          1m
    ...

### Restricting virt-handler DaemonSet

You can patch the `virt-handler` DaemonSet post-deployment to restrict
it to a specific subset of nodes with a nodeSelector. For example, to
restrict the DaemonSet to only nodes with the "region=primary" label:

    kubectl patch ds/virt-handler -n kubevirt -p '{"spec": {"template": {"spec": {"nodeSelector": {"region": "primary"}}}}}'

Deploying on OKD

    The following
    https://docs.openshift.com/container-platform/3.11/admin_guide/manage_scc.html[SCC]
    needs to be added prior KubeVirt deployment:

    [source,bash]
    ----
    $ oc adm policy add-scc-to-user privileged -n kubevirt -z kubevirt-operator
    ----

    Once privileges are granted, the KubeVirt can be deployed as described above.

    Web user interface on OKD
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

    No additional steps are required to extend OKD's web console for KubeVirt.

    The virtualization extension is automatically enabled when KubeVirt deployment is detected.

    Client side `virtctl` deployment

Basic VirtualMachineInstance operations can be performed with the stock
`kubectl` utility. However, the `virtctl` binary utility is required to
use advanced features such as:

-   Serial and graphical console access

It also provides convenience commands for:

-   Starting and stopping VirtualMachineInstances

-   Live migrating VirtualMachineInstances

There are two ways to get it:

-   the most recent version of the tool can be retrieved from the
    [official release
    page](https://github.com/kubevirt/kubevirt/releases)

-   it can be installed as a `kubectl` plugin using
    [krew](https://krew.dev/)

#### Install `virtctl` with `krew`

It is required to [install `krew` plugin
manager](https://github.com/kubernetes-sigs/krew/#installation)
beforehand. If `krew` is installed, `virtctl` can be installed via
`krew`:

    $ kubectl krew install virt

Then `virtctl` can be used as a kubectl plugin. For a list of available
commands run:

    $ kubectl virt help

Every occurrence throughout this guide of

    $ ./virtctl <command>...

should then be read as

    $ kubectl virt <command>...

### From Service Catalog as an APB

You can find KubeVirt in the OKD Service Catalog and install it from
there. In order to do that please follow the documentation in the
[KubeVirt APB
repository](https://github.com/ansibleplaybookbundle/kubevirt-apb).

Deploying from Source
---------------------

See the [Developer Getting Started
Guide](https://github.com/kubevirt/kubevirt/blob/master/docs/getting-started.md)
to understand how to build and deploy KubeVirt from source.

Installing network plugins (optional)
-------------------------------------

KubeVirt alone does not bring any additional network plugins, it just
allows user to utilize them. If you want to attach your VMs to multiple
networks (Multus CNI) or have full control over L2 (OVS CNI), you need
to deploy respective network plugins. For more information, refer to
[OVS CNI installation
guide](https://github.com/kubevirt/ovs-cni/blob/master/docs/deployment-on-arbitrary-cluster.md).

> Note: KubeVirt Ansible [network
> playbook](https://github.com/kubevirt/kubevirt-ansible/tree/master/playbooks#network)
> installs these plugins by default.

Update
------

Zero downtime rolling updates are supported starting with release
`v0.17.0` onward. Updating from any release prior to the KubeVirt
`v0.17.0` release is not supported.

> Note: Updating is only supported from N-1 to N release.

Updates are triggered one of two ways.

1.  By changing the imageTag value in the KubeVirt CR’s spec.

For example, updating from `v0.17.0-alpha.1` to `v0.17.0` is as simple
as patching the KubeVirt CR with the `imageTag: v0.17.0` value. From
there the KubeVirt operator will begin the process of rolling out the
new version of KubeVirt. Existing VM/VMIs will remain uninterrupted both
during and after the update succeeds.

    $ kubectl patch kv kubevirt -n kubevirt --type=json -p '[{ "op": "add", "path": "/spec/imageTag", "value": "v0.17.0" }]'

1.  Or, by updating the kubevirt operator if no imageTag value is set.

When no imageTag value is set in the kubevirt CR, the system assumes
that the version of KubeVirt is locked to the version of the operator.
This means that updating the operator will result in the underlying
KubeVirt installation being updated as well.

    $ export RELEASE=v0.17.0
    $ kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml

The first way provides a fine granular approach where you have full
control over what version of KubeVirt is installed independently of what
version of the KubeVirt operator you might be running. The second
approach allows you to lock both the operator and operand to the same
version.

Newer KubeVirt may require additional or extended RBAC rules. In this
case, the \#1 update method may fail, because the virt-operator present
in the cluster doesn’t have these RBAC rules itself. In this case, you
need to update the `virt-operator` first, and then proceed to update
kubevirt. See [this issue for more
details](https://github.com/kubevirt/kubevirt/issues/2533).

Delete
------

To delete the KubeVirt you should first to delete `KubeVirt` custom
resource and then delete the KubeVirt operator.

    $ export RELEASE=v0.17.0
    $ kubectl delete -n kubevirt kubevirt kubevirt --wait=true # --wait=true should anyway be default
    $ kubectl delete apiservices v1alpha3.subresources.kubevirt.io # this needs to be deleted to avoid stuck terminating namespaces
    $ kubectl delete mutatingwebhookconfigurations virt-api-mutator # not blocking but would be left over
    $ kubectl delete validatingwebhookconfigurations virt-api-validator # not blocking but would be left over
    $ kubectl delete -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml --wait=false

> Note: If by mistake you deleted the operator first, the KV custom
> resource will get stuck in the `Terminating` state, to fix it, delete
> manually finalizer from the resource.
>
> Note: The `apiservice` and the `webhookconfigurations` need to be
> deleted manually due to a bug.
>
>     $ kubectl -n kubevirt patch kv kubevirt --type=json -p '[{ "op": "remove", "path": "/metadata/finalizers" }]'