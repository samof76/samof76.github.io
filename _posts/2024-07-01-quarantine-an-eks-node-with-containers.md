---
layout: post
---

**While performing these step please practice caution, and have the right dashboards that will point to any issues.**

**Step 0.** Annotate the node, `ip-10-102-11-188.ec2.internal`, in question, to not participate in the auto-scaling.

```
$ kubectl annotate node ip-10-102-11-188.ec2.internal cluster-autoscaler.kubernetes.io/scale-down-disabled=true
```

**Step 1.** Cordon the affected the node.

```
kubectl cordon ip-10-102-11-188.ec2.internal
```

**Step 2.** Get the list of app pods running on that instance

```
$ kubectl get pods -n podinfo --field-selector spec.nodeName=ip-10-102-11-188.ec2.internal --show-labels
NAME                      READY   STATUS    RESTARTS   AGE   LABELS
podinfo-ffb8d6b8d-bbs5r   1/1     Running   0          48m   app=podinfo,pod-template-hash=ffb8d6b8d
podinfo-ffb8d6b8d-jfs9b   1/1     Running   0          48m   app=podinfo,pod-template-hash=ffb8d6b8d
podinfo-ffb8d6b8d-mpskj   1/1     Running   0          48m   app=podinfo,pod-template-hash=ffb8d6b8d
```

**Step 3.** Label the pods for quarantine

```
$ kubectl label pod podinfo-ffb8d6b8d-bbs5r -n podinfo app=quarantine --overwrite
pod/podinfo-ffb8d6b8d-bbs5r labeled
```

After you’ve changed the label, you will notice that `ReplicaSet` creates a new pod, but the pod with name `podinfo-ffb8d6b8d-bbs5r` stays around.

NOTE: Do this for every pod you found in the step 2

**Step 4.** Seek out any `Deployment` resource pods, that relevant from to evicted, from that node.

```
$ kubectl get pods --all-namespaces --field-selector spec.nodeName=ip-10-102-11-188.ec2.internal
NAMESPACE           NAME                                              READY   STATUS             RESTARTS   AGE
kube-system         aws-node-b5qb8                                    1/1     Running            0          82d
kube-system         coredns-59dfd6b59f-hn5sk                          1/1     Running            0          56m
kube-system         debug-agent-z2r2x                                 1/1     Running            0          82d
kube-system         kube-proxy-g4sr7                                  1/1     Running            0          82d
podinfo             podinfo-ffb8d6b8d-bbs5r                           1/1     Running            0          50m
podinfo             podinfo-ffb8d6b8d-jfs9b                           1/1     Running            0          50m
podinfo             podinfo-ffb8d6b8d-mpskj                           1/1     Running            0          50m
```

For example in the above list `coredns-59dfd6b59f-hn5sk` pod from the `coredns` deployment, evict this.

```
$ kubectl delete pod -n kube-system coredns-59dfd6b59f-hn5sk
```

**Step 5.** SSH into the node `ip-10-102-11-188.ec2.internal` to stop and disable kubelet

```
$ ssh ec2-user@ip-10-102-11-188.ec2.internal
Last login: Tue Sep 21 09:27:28 2021 from ip-10-102-11-188.ec2.internal

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
14 package(s) needed for security, out of 44 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-10-102-11-188 ~]$ sudo systemctl stop kubelet
[ec2-user@ip-10-102-11-188 ~]$ sudo systemctl disable kubelet
Removed symlink /etc/systemd/system/multi-user.target.wants/kubelet.service.
```

This above operation should send node into `NotReady`  state.

**Step 6.** Once the Node `ip-10-102-11-188.ec2.internal` goes into a `NotReady` state `delete` the node

```
$ kubectl get nodes
NAME                               STATUS                        ROLES    AGE    VERSION
ip-10-102-11-188.ec2.internal      NotReady,SchedulingDisabled   <none>   154m   v1.30.2-eks-ac51f2
$ kubectl delete ip-10-102-11-188.ec2.internal
node "ip-10-102-11-188.ec2.internal" deleted
```

This will ensure the the node is removed and no longer part of the K8s cluster. The above steps ensure the the instance is running with all the docker containers intact, but no longer attached to the K8s cluster. This ends our work on the terminal, the following steps need to be performed on AWS Management Console.

**Step 7.** At this point when you will need to logged in to the node, you should `pause` all the docker containers. But before that you might want to check for any spurious connection from the running containers.

```
# docker ps -q | xargs -n1 -I{} -P0 bash -c "docker inspect -f '{{.State.Pid}}' {}; exit 0;" | xargs -n1 -I{} -P0 bash -c 'nsenter -t {} -n ss -p -a -t4 state established; echo; exit 0'
```

And the pause the running containers.

```
$ docker ps -q | xargs -n1 -I{} -P0 bash -c 'docker pause {}; exit 0'
```


**Step 8.** Head over to the EC2 > ASG dashboard on the AWS Management Console, and detach the instance from the ASG.

**Step 9.** Go to instance details dashboard, and remove the existing security group and add the QUARANTINE security group.

## The Script

Well steps are ok but the it would nice to have script, right?
