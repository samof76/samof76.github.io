---
layout: post
title: Understanding VPC Flow Logs
date: 2025-02-24
---

_VPC Flow Logs are a feature of Amazon Virtual Private Cloud (VPC) that provide detailed information about the network traffic passing through your VPC. They can help you monitor and troubleshoot network issues, identify potential security threats, and optimize your network performance._ But that's not that easy its take a while to wrap your head around it, if you are not sure what is the central piece that the flow log is actually recording or from whose perspective it is recording the traffic.

From my understanding I hope people reading this post and familiar with VPC Flow Logs will agree that the central piece to a flow log is the AWS `eni`(Elastic Network Interface) that the flows are recorded from. The flow log records the traffic that passes through the `eni`, including the source and destination IP addresses, port numbers, protocol, and direction of the traffic.

The next thing that stumps people is the direction of traffic, and number of flows per request on the `eni`. This depends on to what AWS service the `eni` is attached to. So let's understand this.

## EC2 Instance

So lets slighly complicate the use case to where the EC2 instance is a worker node of an EKS cluster, and a pod with IP address `172.16.2.2/16` is on the attached `eni-1234`(with primary IP address `172.16.2.1/16`). And assume the application on this pod, `p1` is making an HTTP get request to [ggl.co](https://www.google.com) at IP `204.79.197.219/32`.

```

                                                 xxxxxx
 ┌──────────────────────┐                        x    xxx
 │                      │                        xx    xxxx
 │                    ┌─┴─┐204.79.197.219/32  xxxxx     x xxx
 │                    │eni┼─────────────►     x             x
 │                     ▲┬─┘                   xx x   xxxxxxxx
 │   ┌───┐GET ggl.co/  ││eni-1234              xxxxxxx
 │   │p1 ┼─────────────┘│172.16.2.1/16
 │   └───┘172.16.2.2/16 │
 └──────────────────────┘
  i-abcd
```

Before we get to technical terminology, that request is outgoing through the `eni-1234` interface. So the request originated from the pod `p1` with IP address `172.16.2.2/16` to destination IP address `204.79.197.219/32` through the `eni-1234` interface. At the `eni-1234` interface, this flow is recorded as an `EGRESS` flow from the `eni-1234` interface's perspective. And here is what the log looks like.

```

```

But that's not all, since the request will have a response let's look at the response flow.

```

                                                 xxxxxx
 ┌──────────────────────┐                        x    xxx
 │                      │                        xx    xxxx
 │                    ┌─┴─┐204.79.197.219/32  xxxxx     x xxx
 │                    │eni◄─────────────────  x             x
 │                    └┬┬─┘                   xx x   xxxxxxxx
 │   ┌───┐RESP ggl.co/ ││eni-1234              xxxxxxx
 │   │p1 ◄─────────────┘│172.16.2.1/16
 │   └───┘172.16.2.2/16 │
 └──────────────────────┘
  i-abcd
```

In this case the response is incoming through the `eni-1234` interface. So the response originated from [ggl.co](https://google.com) with `204.79.197.219/32` to `p1` at `172.16.2.2/16` through the `eni-1234` interface. At the `eni-1234` interface, this flow is recorded as an `INGRESS` flow from the `eni-1234` interface's perspective. And here is what the log looks like.

```

```
