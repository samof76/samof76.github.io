---
layout: post
categories: kubernetes, eks, issue
---

# TIL: Sandbox Image Service

Well totally by accident I ran into this service called, `sandbox-image` on EKS clusters nodes. While rolling out some pods on 1.29 EKS on AmazonLinux 2 instance, we ran into this very strange issue, the pods where stuck were not coming up and while descrining the events I found this.

```
Failed to create pod sandbox: rpc error: code = Unknown desc = failed to get sandbox image "900889452093.dkr.ecr.ap-south-2.amazonaws.com/eks/pause:3.5": failed to pull image "900889452093.dkr.ecr.ap-south-2.amazonaws.com/eks/pause:3.5": failed to pull and unpack image "900889452093.dkr.ecr.ap-south-2.amazonaws.com/eks/pause:3.5": failed to resolve reference "900889452093.dkr.ecr.ap-south-2.amazonaws.com/eks/pause:3.5": pulling from host 900889452093.dkr.ecr.ap-south-2.amazonaws.com failed with status code [manifests 3.5]: 401 Unauthorized
```

> **What is a Pause container?** A "Pause container" in Kubernetes is a small, lightweight container that acts as the foundation for all other containers within a pod, essentially setting up the shared network namespace and other critical resources that all containers in the pod need to communicate with each other, ensuring the pod's network configuration remains consistent even if individual containers restart; it's essentially the "parent" container for all other containers in a pod.

The expectation is that a Pause container image must have been baked into AMI, and will never be deleted, but turns that was not the case, as [this issue](https://github.com/awslabs/amazon-eks-ami/issues/1597) outlines. The issue was the ImageGC configured as part of the kubelet configuration, would also clean up Pause container image. And in those cases, we would run into the issue where pods get stuck with above error.

Though this is a bug, and it has been [fixed](https://github.com/awslabs/amazon-eks-ami/pull/1605), we might still run into scenarios, where some accidental delete of the Pause container image. This is where `sandbox-image` service comes in handy.

## The `sandbox-image` Service

The `sandbox-image` service is `oneshot` systemd service, that is [enabled and started](https://github.com/awslabs/amazon-eks-ami/blob/2e8fa251cb55dfb8dcc7a8e45776d08c68f499e8/templates/al2/runtime/bootstrap.sh#L581-L582) as part of the EKS Kubernetes node bootup, and which does the job of pulling the Pause container image onto, the running node. So on nodes where we run into the issue where pods get stuck failing to pull the Pause container image, all you have to do is.

```bash
sudo systemctl restart sandbox-image
```

This `sandbox-image` service, is only available on the Amazon Linux 2, and not on AmazonLinux 2023. And here further details.
- `sandbox-image` service is available [here](https://github.com/awslabs/amazon-eks-ami/blob/2e8fa251cb55dfb8dcc7a8e45776d08c68f499e8/templates/al2/runtime/sandbox-image.service)
- `sandbox-image` on startup runs [this](https://github.com/awslabs/amazon-eks-ami/blob/2e8fa251cb55dfb8dcc7a8e45776d08c68f499e8/templates/al2/runtime/pull-sandbox-image.sh) script

## How now Pause is not GC'ed?

It is Kubelet that is responsible to cleanup the unused container images from the node node to free up space, which could clog other images from being pulled due to space constraints. [These](https://github.com/kubernetes/kubernetes/blob/6cb457bc6684520793b770ffdf155ae9bdd84025/pkg/kubelet/apis/config/types.go#L200C1-L215C1) are the Kubelet configurations that govern this.

```go
	// ImageMinimumGCAge is the minimum age for an unused image before it is
	// garbage collected.
	ImageMinimumGCAge metav1.Duration
	// ImageMaximumGCAge is the maximum age an image can be unused before it is garbage collected.
	// The default of this field is "0s", which disables this field--meaning images won't be garbage
	// collected based on being unused for too long.
	ImageMaximumGCAge metav1.Duration
	// imageGCHighThresholdPercent is the percent of disk usage after which
	// image garbage collection is always run. The percent is calculated as
	// this field value out of 100.
	ImageGCHighThresholdPercent int32
	// imageGCLowThresholdPercent is the percent of disk usage before which
	// image garbage collection is never run. Lowest disk usage to garbage
	// collect to. The percent is calculated as this field value out of 100.
	ImageGCLowThresholdPercent int32
```

To better understand these settings, first lets get `ImageMaximumGCAge` out of the way. If not set to `0` the `ImageMaximumGCAge` is an anycase settings, which will be GC all the unused images beyond

## And in AmazonLinux 2023

AmazonLinux 2023 decided to move away from the downloading at startup to baking the Pause container image as part of the baking of AMI itself.
