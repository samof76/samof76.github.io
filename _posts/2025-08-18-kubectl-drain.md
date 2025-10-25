---
layout: post
title: Understanding the Kubectl's `drain` command
category: kubernetes, kubectl, cli
---

Command's help

```bash
kubectl drain --help
Drain node in preparation for maintenance.

 The given node will be marked unschedulable to prevent new pods from arriving. 'drain' evicts the pods if the API
server supports https://kubernetes.io/docs/concepts/workloads/pods/disruptions/ . Otherwise, it will use normal DELETE
to delete the pods. The 'drain' evicts or deletes all pods except mirror pods (which cannot be deleted through the API
server).  If there are daemon set-managed pods, drain will not proceed without --ignore-daemonsets, and regardless it
will not delete any daemon set-managed pods, because those pods would be immediately replaced by the daemon set
controller, which ignores unschedulable markings.  If there are any pods that are neither mirror pods nor managed by a
replication controller, replica set, daemon set, stateful set, or job, then drain will not delete any pods unless you
use --force.  --force will also allow deletion to proceed if the managing resource of one or more pods is missing.

 'drain' waits for graceful termination. You should not operate on the machine until the command completes.

 When you are ready to put the node back into service, use kubectl uncordon, which will make the node schedulable again.

 https://kubernetes.io/images/docs/kubectl_drain.svg

Examples:
  # Drain node "foo", even if there are pods not managed by a replication controller, replica set, job, daemon set or
stateful set on it
  kubectl drain foo --force

  # As above, but abort if there are pods not managed by a replication controller, replica set, job, daemon set or
stateful set, and use a grace period of 15 minutes
  kubectl drain foo --grace-period=900

Options:
      --chunk-size=500: Return large lists in chunks rather than all at once. Pass 0 to disable. This flag is beta and
may change in the future.
      --delete-emptydir-data=false: Continue even if there are pods using emptyDir (local data that will be deleted when
the node is drained).
      --disable-eviction=false: Force drain to use delete, even if eviction is supported. This will bypass checking
PodDisruptionBudgets, use with caution.
      --dry-run='none': Must be "none", "server", or "client". If client strategy, only print the object that would be
sent, without sending it. If server strategy, submit server-side request without persisting the resource.
      --force=false: Continue even if there are pods not managed by a ReplicationController, ReplicaSet, Job, DaemonSet
or StatefulSet.
      --grace-period=-1: Period of time in seconds given to each pod to terminate gracefully. If negative, the default
value specified in the pod will be used.
      --ignore-daemonsets=false: Ignore DaemonSet-managed pods.
      --ignore-errors=false: Ignore errors occurred between drain nodes in group.
      --pod-selector='': Label selector to filter pods on the node
  -l, --selector='': Selector (label query) to filter on
      --skip-wait-for-delete-timeout=0: If pod DeletionTimestamp older than N seconds, skip waiting for the pod.
Seconds must be greater than 0 to skip.
      --timeout=0s: The length of time to wait before giving up, zero means infinite

Usage:
  kubectl drain NODE [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).
```
## Sequence Diagram

![Sequence Diagram](https://kubernetes.io/images/docs/kubectl_drain.svg)

## Code

The kubectl commads are implemented [here](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/tree/staging/src/k8s.io/kubectl). And the code for drain begins [here](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/tree/staging/src/k8s.io/kubectl/pkg/cmd/drain) in `drain.go` with the command definition.

```go
func NewCmdDrain(f cmdutil.Factory, ioStreams genericiooptions.IOStreams) *cobra.Command {
	o := NewDrainCmdOptions(f, ioStreams)

	cmd := &cobra.Command{
		Use:                   "drain NODE",
		DisableFlagsInUseLine: true,
		Short:                 i18n.T("Drain node in preparation for maintenance"),
		Long:                  drainLong,
		Example:               drainExample,
		ValidArgsFunction:     completion.ResourceNameCompletionFunc(f, "node"),
		Run: func(cmd *cobra.Command, args []string) {
			cmdutil.CheckErr(o.Complete(f, cmd, args))
			cmdutil.CheckErr(o.RunDrain())
		},
	}
	cmd.Flags().BoolVar(&o.drainer.Force, "force", o.drainer.Force, "Continue even if there are pods that do not declare a controller.")
	cmd.Flags().BoolVar(&o.drainer.IgnoreAllDaemonSets, "ignore-daemonsets", o.drainer.IgnoreAllDaemonSets, "Ignore DaemonSet-managed pods.")
	cmd.Flags().BoolVar(&o.drainer.DeleteEmptyDirData, "delete-emptydir-data", o.drainer.DeleteEmptyDirData, "Continue even if there are pods using emptyDir (local data that will be deleted when the node is drained).")
	cmd.Flags().IntVar(&o.drainer.GracePeriodSeconds, "grace-period", o.drainer.GracePeriodSeconds, "Period of time in seconds given to each pod to terminate gracefully. If negative, the default value specified in the pod will be used.")
	cmd.Flags().DurationVar(&o.drainer.Timeout, "timeout", o.drainer.Timeout, "The length of time to wait before giving up, zero means infinite")
	cmd.Flags().StringVarP(&o.drainer.PodSelector, "pod-selector", "", o.drainer.PodSelector, "Label selector to filter pods on the node")
	cmd.Flags().BoolVar(&o.drainer.DisableEviction, "disable-eviction", o.drainer.DisableEviction, "Force drain to use delete, even if eviction is supported. This will bypass checking PodDisruptionBudgets, use with caution.")
	cmd.Flags().IntVar(&o.drainer.SkipWaitForDeleteTimeoutSeconds, "skip-wait-for-delete-timeout", o.drainer.SkipWaitForDeleteTimeoutSeconds, "If pod DeletionTimestamp older than N seconds, skip waiting for the pod.  Seconds must be greater than 0 to skip.")

	cmdutil.AddChunkSizeFlag(cmd, &o.drainer.ChunkSize)
	cmdutil.AddDryRunFlag(cmd)
	cmdutil.AddLabelSelectorFlagVar(cmd, &o.drainer.Selector)
	return cmd
}
```

The command calls `RunDrain` function as you can see in this snippet.

```go
...
		Run: func(cmd *cobra.Command, args []string) {
			cmdutil.CheckErr(o.Complete(f, cmd, args))
			cmdutil.CheckErr(o.RunDrain())
		},
...
```

> `Complete` populates some fields from the factory, grabs command line arguments and looks up the node using Builder

Let's just focus on the happy path, and [follow `RunDrain`](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/kubectl/pkg/cmd/drain/drain.go?L324:27-324:35), and since its a method of `DrainCmdOptions` all its properties are readily available. Now let go with the flow.

1. First calls `RunCordonOrUncordon(true)` this ensures that [node cordoned before evicting the node](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/kubectl/pkg/cmd/drain/drain.go?L401:27-401:46).
2. Then it calls the `deleteOrEvictPodsSimple` passing [it the node info](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/kubectl/pkg/cmd/drain/drain.go?L365:27-365:50).
3. `<deleteOrEvictPodsSimple>` this calls `GetPodsForDeletion` which get the [list of pods](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/kubectl/pkg/drain/drain.go?L181:18-181:36), filtering based on the option provided to drain.
4. `<deleteOrEvictPodsSimple>` calls the `DeleteOrEvictPods` on the list of pods got for deletion in the previous steps, which is actually is [responsible for evicting pods](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/kubectl/pkg/drain/drain.go?L244:18-244:35).
5. `<deleteOrEvictPodsSimple / DeleteOrEvictPods>` if `disableEnviction` flag is not set(which is the default, other pods are delete and not evicted), it will call the [`evictPods` on the list if pods](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/kubectl/pkg/drain/drain.go?L268:18-268:27). This `evictPods` method also gets, [Eviction API Group Version](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/),  and a callback function to get pod info, apart from pod list.
6. `<deleteOrEvictPodsSimple / DeleteOrEvictPods / evictPods>` this makes the call to `EvictPod`, passing the pod object and Eviction API Group version, [ref](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/kubectl/pkg/drain/drain.go?L149:18-149:26).
7. `<deleteOrEvictPodsSimple / DeleteOrEvictPods / evictPods / EvictPod>` based on the Eviction API Group version scheme, calls [Evict(and not delete)](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/kubectl/pkg/drain/drain.go?L162).

So what's happening here...

```go
    case policyv1.SchemeGroupVersion:
		// send policy/v1 if the server supports it
		eviction := &policyv1.Eviction{
			ObjectMeta: metav1.ObjectMeta{
				Name:      pod.Name,
				Namespace: pod.Namespace,
			},
			DeleteOptions: &delOpts,
		}
		return d.Client.PolicyV1().Evictions(eviction.Namespace).Evict(d.getContext(), eviction)
```

This is creating something like this,

```json
{
  "apiVersion": "policy/v1",
  "kind": "Eviction",
  "metadata": {
    "name": "<podname>",
    "namespace": "<namespace>"
  }
}
```

And making a `POST` call to the API server as you could [see here](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/client-go/kubernetes/typed/policy/v1/eviction_expansion.go?L30:21-30:26).

Now how does this Evict API work is documented [here](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/#how-api-initiated-eviction-works). As you can see that, it honours the PDB of pod in question. You would end up with [this](https://sourcegraph.com/github.com/kubernetes/kubernetes@810e9e212ec5372d16b655f57b9231d8654a2179/-/blob/staging/src/k8s.io/kubectl/pkg/drain/drain.go?L332).
