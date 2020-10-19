# Understanding vCPUs, Requests and Limits in OpenShift Virtualisation

When running VMs on OpenShift Virtualisation, you may be tempted to dive into the `VirtualMachine` definition and crank up the `cpus.cores`, `cpus.sockets` and `cpus.threads` configuration options, figuring this will help drive more performance in your VM.

## CPU topology

`cpus.cores`, `cpus.sockets` and `cpus.threads` should be read as the following:

* `cpus.cores` = "cores per socket"
* `cpus.sockets` = "total sockets"
* `cpus.threads` = "threads per core"

So, a configuration like so:

```
spec:
    cpus:
        cores: 2
        sockets: 2
        threads: 2
```

Has two sockets, with two cores per socket, and two threads per core.

From the VMs perspective, it will see 2 sockets * 2 cores per socket * 2 threads per core = 8 logical processors.

You would modify the CPU topology for a couple of reasons:

* You have software that is highly specific on the number of sockets it expects to see. So you adjust the VM topology to suit (e.g. databases licensed by socket).
* You need the VM to be NUMA-node aware, e.g. the VM will be spread across two NUMA nodes on the hypervisor and you need the VM to be aware of that, so the VM's OS can schedule threads to suit.

For a standard VM that doesn't require a custom CPU topology, I would leave this alone.

The default is a topology of 1 core per socket, 1 thread per core, and the number of sockets equal to the desired number of vCPUS.

## Pod Requests

The `virt-launcher` pod only asks for a default of 100 millicores as a request, and has no limit specified. The `qemu-kvm` process that is your VM will run within the requests/limits constraints of the `virt-launcher` pod.

That means your VM will be given whatever spare capacity is available on the host above 100 millicores worth of CPU time, **regardless of how many cores/threads/sockets you specify in the `VirtualMachine` definition**.

If you increase the vCPU count by adjusting the sockets, cores and threads parameters above, all you will do is increase the number of QEMU threads running in your `qemu-kvm` process. All of these threads will still reside within the cgroup constraints of your pod - in short, they'll start competing with one another for increasingly scarce CPU time.

This means that if the node is heavily committed, you will end up with your VM being squeezed into just 100 millicores worth of time - this is not going to work, and the VM will run miserably.

## Setting a guaranteed quality-of-service for the VM

To ensure that your VM pod gets the expected core allocation, change `spec.domain.resources.requests.cpu` and `spec.domain.resources.limits.cpu` in the `VirtualMachine` definition.

The CPU request and limit must be identical.

Start the VM again, and the `virt-launcher` pod will have an increased `request` and `limit`.

You cannot set both `spec.domain.cpu` and `spec.domain.requests.cpu`/`spec.domain.limits.cpu` at the same time. One or the other.

## Dedicated Resources

For VMs that need no-kidding dedicated resources, i.e. their own CPU cores that nothing else will ever be scheduled on, dedicated resources are required. This is CPU pinning; for VMs that require predicatable latency this is a must-do.

[Dedicated resources documentation](https://docs.openshift.com/container-platform/4.5/virt/virtual_machines/advanced_vm_management/virt-dedicated-resources-vm.html).

[CPU Manager documentation](https://docs.openshift.com/container-platform/4.5/virt/virtual_machines/advanced_vm_management/virt-dedicated-resources-vm.html).

[Kubevirt doco](https://kubevirt.io/user-guide/#/creation/dedicated-cpu).

## Gotchas

There is nothing to stop you from having more vCPUs for your VM than there are physical cores on the system. This is a Bad Thing, however, and it should be avoided.

When you have more vCPUs than you do cores on the hypervisor, all you do is create contention between your vCPU QEMU threads for physical core time. They'll be competing for time on the silicon, and this will impact your VM performance.


