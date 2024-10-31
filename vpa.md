# Resource Optimization Discussion

To illustrate, let’s take Pinto as an example to explore how we can optimize resource usage.

## Resource Allocation Challenges for Services

One of the biggest challenges is determining the right resource allocations for each service. There are several factors to consider:

- **Bootstrap vs. Runtime Resource Needs**: Some services need more CPU or memory during startup but significantly less once the initial processes complete.
- **Variable Load Scenarios**: For applications like Kafka or Couchbase, resource usage can spike during regression or integration testing due to the increased API calls.
- **Scheduling and Resource Requests**: When we request resources via the API server, the scheduler only considers the requested CPU and memory, not the actual usage. This can lead to scenarios where resources are underutilized—for instance, a 60-core request might only result in 40 cores of actual usage. This highlights an opportunity to optimize further.

## Potential Optimizations

The most immediate solution that comes to mind is **Vertical Pod Autoscaling (VPA)**.

### Known Limitations

- VPA requires pod restarts for resource adjustments, which means the pod reboots, incurring high resource needs again during bootstrapping.
- An improvement here is Kubernetes’ new alpha feature (from v1.27), which supports **in-place resource patching**.
- To make this work, we need to synchronize three components: our DevStack controller, VPA, and in-place Kubernetes resource allocation.

### Implementation Steps

1. **Testing with Formula**:
    - We can start by testing a formula to observe its impact on our cluster. For instance, let’s say a user requests 1000m of CPU.
    - For VPA, we’ll set boundaries around this request, allowing a minimum of -20% and a maximum of +20%.
2. **In-Place Resource Patching**:
    - Using in-place resource patching, we can avoid pod restarts when adjusting resource allocations.
3. **Adapting to the Reconciler**:
    - The challenge lies in making these adjustments without conflicting with our reconciler. The reconciler will still see the desired state as defined in our DevStack YAML, so we need a watcher that tracks the VPA actions. When the VPA controller triggers a resize, it should be reflected at the DevStack component level.

---

This refined approach should help clarify the resource management plan and integration with VPA and in-place resource adjustments.

