---
layout: post
title: CPUManagerPolicy Under the Hood
category: kubernetes, kubelet, cpu-manager, cpu-manager-policy
---

> WARNING: There are known issues with using this method, that might cause diruptions in some edge cases and are yet to be fixed. Here are the [bugs](https://github.com/orgs/kubernetes/projects/185/views/1?filterQuery=cpumanager) that are being tracked.

According to the documentation, when `static` CPUManager is used, the [`cpuset` cgroup controller](https://docs.kernel.org/admin-guide/cgroup-v1/cpusets.html) is used to assign specific cores to the container with guaranteed CPU resources. This ensures that the container has exclusive access to the specified cores, preventing interference from other processes, reducing noisy neighbor issues and also ensuring the container itself is not noisy neighor. First a look at the section where this configuration is defined in `kubelet-config.json`.

```bash
[root@ip-w-x-y-z ~]# cat /etc/kubernetes/kubelet/kubelet-config.json
...
  "cpuManagerPolicy": "static",
  "cpuManagerPolicyOptions": {
    "full-pcpus-only": "true"
  },
...
```

And this could be analyzed by inspecting the processes in question. In this case the process we are concerned about is `clickhouse`, we can use the `crictl ps` command to get the container ID of the process.

First lets take a look at get an node where we have not enabled CPUManager, let's the container ID of clickhouse.

```bash
[root@ip-a-b-c-d ~]# crictl ps --name clickhouse
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
29ebb158a0a16       2ac35038b7301       3 hours ago         Running             clickhouse-backup         0                   21cea07c4139a       chi-clickhouse-vcputest-0-0-0
0921f5a6a8ab2       639814aaef121       3 hours ago         Running             clickhouse                0                   21cea07c4139a       chi-clickhouse-vcputest-0-0-0
```

Notice, `0921f5a6a8ab2` is `ID` of the `clickhouse` container.

```bash
[root@ip-a-b-c-d ~]# crictl inspect 0921f5a6a8ab2 | jq .status.resources.linux
{
  "cpuPeriod": "100000",
  "cpuQuota": "500000",
  "cpuShares": "5120",
  "cpusetCpus": "",
  "cpusetMems": "",
  "hugepageLimits": [],
  "memoryLimitInBytes": "8589934592",
  "memorySwapLimitInBytes": "8589934592",
  "oomScoreAdj": "-997",
  "unified": {}
}
```

Notice in this case, where CPUManager is not used/enabled, the `cpusetCpus` is empty. This means that the container is not restricted to specific cores and can use any available CPU resources. Also, note cpu_manager_state.

```bash
[root@ip-a-b-c-d ~]# cat /var/lib/kubelet/cpu_manager_state | jq .
{
  "policyName": "none",
  "defaultCpuSet": "",
  "checksum": 1353318690
}
```

Now, lets take a look at a node where we have enabled CPUManager. Let's take a look at the `clickhouse` container on this node.

```bash
[root@ip-w-x-y-z ~]# crictl ps --name clickhouse
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
e166cc07430cd       2ac35038b7301       5 hours ago         Running             clickhouse-backup   0                   4f28ab9656be0       chi-clickhouse-pcputest-0-0-0
27f87a744a759       639814aaef121       5 hours ago         Running             clickhouse          0                   4f28ab9656be0       chi-clickhouse-pcputest-0-0-0
```

The `27f87a744a759` is our container of interest.

```bash
[root@ip-w-x-y-z ~]# crictl inspect 27f87a744a759 | jq .status.resources.linux
{
  "cpuPeriod": "100000",
  "cpuQuota": "500000",
  "cpuShares": "5120",
  "cpusetCpus": "1-5",
  "cpusetMems": "",
  "hugepageLimits": [],
  "memoryLimitInBytes": "8589934592",
  "memorySwapLimitInBytes": "8589934592",
  "oomScoreAdj": "-997",
  "unified": {}
}
```
Notice in this case, where CPUManager is used/enabled, the `cpusetCpus` is set to `1-5`. This means that the container is restricted to specific cores and can only use the specified cores, and those CPU cores are dedicated to the container. And the `cpu_manager_state` is also set to `1-5`.

```bash
[root@ip-w-x-y-z ~]# cat /var/lib/kubelet/cpu_manager_state | jq .
{
  "policyName": "static",
  "defaultCpuSet": "0,6-7",
  "entries": {
    "d8d68b09-ce62-455c-88ba-57050e4169f0": {
      "clickhouse": "1-5"
    }
  },
  "checksum": 2344224312
}
```

## What is `/var/lib/kubelet/cpu_manager_state`?

The `/var/lib/kubelet/cpu_manager_state` file is a key component of Kubernetes' CPU management system. By inspecting this file, you can verify that CPU allocations are being handled correctly and consistently. When combined with containerd's implementation of CPU pinning via cgroups, this ensures that your Pods receive the exclusive CPU resources they require, especially when using the `full-pcpus-only` policy. Also this file maintains the state of CPU allocations across kubelet restarts, ensuring that CPU assignments are consistent and persistent.

## Verifying pcpu is not shared

To verify the CPUs provided to the guaranteed pod's clickhouse container, we did the following experiment:

1. Created the ClickHouse pod with guaranteed resources
2. Created a burstable pod running `stress-ng` container utilizing cores beyond the limit set

In this case the expected behavior is that burstable pod's `stress-ng` container will not be able to utilize the CPU cores allocated to the guaranteed pod's clickhouse container.

Here is the `/var/lib/kubelet/cpu_manager_state` on that node.

```bash
[root@ip-w-x-y-z ~]# cat /var/lib/kubelet/cpu_manager_state | jq .
{
  "policyName": "static",
  "defaultCpuSet": "0,4-7",
  "entries": {
    "54b3c938-b248-4373-992d-adec7ea2692a": {
      "clickhouse": "1-3"
    }
  },
  "checksum": 1600324503
}
```

And there is the pod yaml for the `stress-ng`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-ng
  namespace: stress-test
spec:
  nodeSelector:
      node.kubernetes.io/instance-type: r7g.2xlarge
      karpenter.sh/nodepool: arm-cpumanager-nodepool
      kubernetes.io/hostname: ip-w-x-y-z.ec2.internal
  containers:
  - name: stress-ng
    image: alpine:latest
    command:
    - sh
    - -c
    - apk add --no-cache stress-ng && stress-ng --cpu 4 --timeout 300s
    resources:
      limits:
        cpu: 3
        memory: 8Gi
      requests:
        cpu: 2
        memory: 8Gi
  restartPolicy: Never
```

So noticed the following using `htop`, the `stress-ng` container was not able to utilize the CPU cores allocated to the guaranteed pod's clickhouse container.

![Stress-NG Outside PCPU](assets/images/cpumanager_full_pcpu_htop.png)

**ALSO**, we flip the switch to check if a container inside of guaranteed pod only uses the CPU that it is provide using cpuset. So we run `stress-ng` as a guaranteed pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-ng
  namespace: stress-test
spec:
  nodeSelector:
      node.kubernetes.io/instance-type: r7g.2xlarge
      karpenter.sh/nodepool: arm-cpumanager-nodepool
      kubernetes.io/hostname: ip-w-x-y-z.ec2.internal
  containers:
  - name: stress-ng
    image: alpine:latest
    command:
    - sh
    - -c
    - apk add --no-cache stress-ng && stress-ng --cpu 4 --timeout 300s
    resources:
      limits:
        cpu: 3
        memory: 4Gi
      requests:
        cpu: 3
        memory: 4Gi
  restartPolicy: Never
```

Check the cpu assignment for this container.

```bash
[root@ip-w-x-y-z ~]# cat /var/lib/kubelet/cpu_manager_state | jq .
{
  "policyName": "static",
  "defaultCpuSet": "0,4-7",
  "entries": {
    "c2748791-0c91-4c75-9adb-cbc9d7cd1855": {
      "stress-ng": "1-3"
    }
  },
  "checksum": 1477588470
}
```

Here is the htop output, notice only assigned CPUs are used.

![Stress-NG Inside PCPU](assets/images/cpumanager_full_pcp_htop2.png)
