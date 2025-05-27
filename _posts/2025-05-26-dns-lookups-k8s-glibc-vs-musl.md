---
layout: post
title: DNS Lookups glibc vs musl-libc wrt K8s
category: kubernetes, glibc, musl, dns
---

> ASIDE: Love the way [hare-lang](https://harelang.org/) has implemented [DNS lookups](https://git.sr.ht/~sircmpwn/hare/tree/989d021522d24cede090f38e1a00704a40b9e798/item/net/dns/query.ha), it's so clean to read and understand. Just to note, they have followed the musl-libc way of implementing, but only cleaner :).

Recently while going through the [musl-libc](https://musl.libc.org/) documentation, I noticed that it has a [different approach to DNS lookups](https://wiki.musl-libc.org/functional-differences-from-glibc.html) compared to glibc. Just want to document this from the kubernetes context here.

> NOTE: `musl-libc` is the standard libc packaged with the popular the [Alpine Linux](https://www.alpinelinux.org/).

Kubernetes handles DNS lookups by using a DNS server that is deployed as a pod in the cluster. This DNS server is responsible for resolving DNS queries for services and pods in the cluster. Kubernetes also provides a DNS service that can be used to resolve DNS queries for external domains. Also Kubernetes provides a mechanism for configuring DNS settings for pods, such as setting the DNS search domain or specifying DNS servers to use.

## DNS Lookups from a Pod

When a normal pod(with no custom DNS settings) is created the pods comes up with default `resolve.conf`, that looks like this:

```conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### The `nameserver` entry

The `nameserver` entry in the `resolve.conf` file specifies the IP address of the DNS server that the pod should use to resolve DNS queries. In Kubernetes, the DNS server is deployed as a pod(usually CoreDNS) in the cluster and is accessible as service with `clusterIP` at the IP address `10.96.0.10`. This IP address is used as the `nameserver` entry in the `resolve.conf` file for all pods in the cluster.

At this moment is important to note that there could be `nameserver` entries in the `resolve.conf` file. These entries are used to specify additional DNS servers that the pod should use to resolve DNS queries. This can be configured using the `dnsPolicy` field in the pod's specification, which we will discuss.

### The `search` entry

The `search` entry in the `resolve.conf` file specifies the search domains that the pod should use to resolve DNS queries. In Kubernetes, the search domains are automatically configured based on the namespace(in this case `default`) and cluster domain of the pod. The search domains are used to resolve DNS queries for services and pods in the cluster. A good example is a service named `my-service` in the `default` namespace. The search domain for this service would be `default.svc.cluster.local`. This means that if a pod in the `default` namespace tries to resolve the DNS name `my-service`, it will first try to resolve it as `my-service.default.svc.cluster.local`.

### The `ndots` entry in `options`

The `ndots` entry in the `options` section of the `resolve.conf` file specifies the number of dots that must appear in a hostname before the DNS resolver will try to resolve it as a fully qualified domain name. Just to clarify,

- The `ndots` option in `/etc/resolv.conf` specifies a threshold for the number of dots (`.`) in a hostname. It determines whether the resolver treats the hostname as an FQDN or applies search domains from the search or domain directives before querying.
- If a hostname has fewer dots than ndots, the resolver first tries appending each search domain (e.g., `default.svc.cluster.local`) before attempting the literal hostname.

Example:

- If `ndots` is set to 5, the resolver will try to resolve `my-service.prod.internal`(no. of dots less than `ndots`) as `my-service.prod.internal.default.svc.cluster.local` and go through the `search` domains(until a successful lookup) before trying `my-service.prod.internal`.
- But if `ndots` is set to 2, and `my-service.prod.internal`(no. of dots 2 >= `ndots`) will be considered as an FQDN.

NOTE: A full qualified domain is a domain that ends with a . in the end, that `example.com.` is a fully qualified domain, so now that means `example.com` is not a fully qualified domain. So when we type a domain in the browser the browsers do the appending of this `.` for you. And it is important to note `ndots` does not count the trailing dot.

### Custom DNS Configuration

Kubernetes provides `dnsPolicy` as part of the Pod Specitication, that could be used customize what goes into `resolve.conf`. That would need seperate indepth coverage, but here is an an example configuration.

```yaml
...
dnsConfig:
  nameservers:
  - 192.0.2.1
  searches:
  - ns1.svc.cluster-domain.example
  - my.dns.search.suffix
  options:
  - name: ndots
    value: "2"
...
```

## Overview of DNS Resolution in C Libraries

DNS (Domain Name System) resolution is a critical function of any C library, allowing applications to translate human-readable domain names into IP addresses. Both glibc and musl-libc provide implementations of standard functions like `getaddrinfo()`, but their internal approaches differ significantly in design philosophy, performance characteristics, and feature support.

## Key Differences in Implementation Approaches

### Resolver Behavior

#### glibc
- Queries nameservers in `/etc/resolv.conf` **sequentially, one by one**
- If a hostname is found in a source (a positive answer), the resolver looks up all address types requested and stops searching
- If there is a negative answer (hostname does not exist), the resolver also stops searching
- If neither positive nor negative answer is obtained, the resolver continues searching until all sources have been searched
- Supports falling back to search domains even for hostnames with dots exceeding the `ndots` threshold

#### musl-libc
- Queries all nameservers in `/etc/resolv.conf` **in parallel** and returns the fastest response
- Completely new resolver code with different behavior in certain situations
- For hostnames with at least as many dots as `ndots`, only tries in the global namespace (never falling back to search domains)
- Prior to version 1.1.13, did not support the `domain` and `search` keywords in `resolv.conf`

### Configuration File Handling

#### glibc
- Reads and processes `/etc/nsswitch.conf` to determine the order of name resolution sources
- Opens and uses the `nscd` socket for name service caching
- Reads and processes `/etc/gai.conf` for additional configuration
- Supports `single-request` and `single-request-reopen` options in `/etc/resolv.conf`

#### musl-libc
- Does not read `/etc/nsswitch.conf`
- Does not use the `nscd` socket
- Does not read `/etc/gai.conf`
- Does not support the `single-request` and `single-request-reopen` options in `/etc/resolv.conf`
- Only reads `/etc/services`, `/etc/hosts`, and `/etc/resolv.conf`

### Default Flags and Behavior

#### glibc
- Default `ai_flags` for `getaddrinfo()`: `AI_ADDRCONFIG|AI_V4MAPPED`
- More extensive error handling and retry mechanisms

#### musl-libc
- Default `ai_flags` for `getaddrinfo()`: `0` (no flags set)
- Simpler, more lightweight implementation focused on efficiency

### Protocol Support

#### glibc
- Has long supported both UDP and TCP for DNS queries

#### musl-libc
- TCP support for DNS queries was only introduced in version 1.2.4
- Prior to this, there could be issues handling larger packets, such as those required for DNSSEC or when a large number of records are returned

### Search Domain Handling

#### glibc
- Processes queries with fewer dots than `ndots` with search domains first, then tries literally
- For queries with at least as many dots as `ndots`, tries literally first, then falls back to search domains if not found

#### musl-libc
- Processes queries with fewer dots than `ndots` with search domains first, then tries literally (like glibc)
- For queries with at least as many dots as `ndots`, only tries in the global namespace (never falling back to search domains)

### Implementation Philosophy

#### glibc
- More feature-rich with extensive compatibility options
- Focuses on compatibility with various network configurations and legacy systems
- More complex implementation with additional features and options

#### musl-libc
- Lightweight, fast, and simple implementation
- Focuses on efficiency, standards compliance, and security
- Simpler implementation with fewer features but potentially better performance in standard cases

### Code Implementation Analysis

#### glibc DNS Resolver Implementation

The core of glibc's DNS resolver is implemented in the `resolv` directory, with key files including `res_query.c`, `res_init.c`, and related files. The implementation is derived from BIND 8, with modifications to fit into the glibc framework.

[Key function in `res_query.c`](https://github.com/bminor/glibc/blob/319f94dea2b7eeff12adb22ee50b46b64dd6a52d/resolv/res_query.c#L109):

```c
int __res_context_query (struct resolv_context *ctx, const char *name, int class,
                        int type, unsigned char *answer, int anslen,
                        unsigned char **answerp, unsigned char **answerp2,
                        int *nanswerp2, int *resplen2, int *answerp2_malloced)
{
  struct __res_state *statp = ctx->resp;
  UHEADER *hp = (UHEADER *) answer;
  UHEADER *hp2;
  int n;
  bool retried = false;

  /* It requires 2 times QUERYSIZE for type == T_QUERY_A_AND_AAAA. */
  struct scratch_buffer buf;
  scratch_buffer_init (&buf);
  _Static_assert (2 * QUERYSIZE <= sizeof (buf.__space.__c),
                 "scratch_buffer too small");
  u_char *query1 = buf.data;
  int nquery1 = -1;
  u_char *query2 = NULL;
  int nquery2 = 0;

 again:
  hp->rcode = NOERROR;  /* default */

  if (type == T_QUERY_A_AND_AAAA)
    {
      // Code for handling both A and AAAA queries
      // ...
    }
  else
    {
      n = __res_context_mkquery (ctx, QUERY, name, class, type, NULL,
                                query1, buf.length);
      if (n > 0 && (statp->options & (RES_USE_EDNS0|RES_USE_DNSSEC)) != 0)
        {
          /* Use RESOLV_EDNS_BUFFER_SIZE if the receive buffer can
             be reallocated. */
          size_t advertise;
          if (answerp == NULL)
            advertise = anslen;
          else
            advertise = RESOLV_EDNS_BUFFER_SIZE;
          n = __res_nopt (ctx, n, query1, buf.length, advertise);
        }
      nquery1 = n;
    }

  // Error handling and query processing
  // ...

  // Send the query and process the response
  n = __res_context_send (ctx, query1, nquery1, query2, nquery2,
                         answer, anslen, answerp, answerp2, nanswerp2,
                         resplen2, answerp2_malloced);

  // Process the response
  // ...

  return (n);
}
```

This function is responsible for formulating DNS queries, sending them to nameservers, and processing the responses. It handles both IPv4 (A) and IPv6 (AAAA) queries, and supports various options like EDNS0 and DNSSEC.

The glibc implementation is characterized by:
- Complex error handling and retry mechanisms
- Support for various DNS extensions and options
- Sequential querying of nameservers
- Extensive compatibility with different network configurations

#### musl-libc DNS Resolver Implementation

The musl-libc DNS resolver is implemented in the `src/network` directory, with key files including `getaddrinfo.c`, `lookup.h`, and related files. Unlike glibc, musl-libc's implementation is written from scratch rather than being derived from BIND.

[Key function in `getaddrinfo.c`](https://github.com/bminor/musl/blob/c47ad25ea3b484e10326f933e927c0bc8cded3da/src/network/getaddrinfo.c#L12):

```c
int getaddrinfo(const char *restrict host, const char *restrict serv, const struct addrinfo *restrict hint, struct addrinfo **restrict res)
{
  struct service ports[MAXSERVS];
  struct address addrs[MAXADDRS];
  char canon[256], *outcanon;
  int nservs, naddrs, nais, canon_len, i, j, k;
  int family = AF_UNSPEC, flags = 0, proto = 0, socktype = 0;
  int no_family = 0;
  struct aibuf *out;

  if (!host && !serv) return EAI_NONAME;

  if (hint) {
    family = hint->ai_family;
    flags = hint->ai_flags;
    proto = hint->ai_protocol;
    socktype = hint->ai_socktype;

    const int mask = AI_PASSIVE | AI_CANONNAME | AI_NUMERICHOST |
      AI_V4MAPPED | AI_ALL | AI_ADDRCONFIG | AI_NUMERICSERV;
    if ((flags & mask) != flags)
      return EAI_BADFLAGS;

    switch (family) {
    case AF_INET:
    case AF_INET6:
    case AF_UNSPEC:
      break;
    default:
      return EAI_FAMILY;
    }
  }

  // Handle AI_ADDRCONFIG flag
  if (flags & AI_ADDRCONFIG) {
    // Check if IPv4 and IPv6 are configured
    // ...
  }

  // Look up service and name
  nservs = __lookup_serv(ports, serv, proto, socktype, flags);
  if (nservs < 0) return nservs;

  naddrs = __lookup_name(addrs, canon, host, family, flags);
  if (naddrs < 0) return naddrs;

  if (no_family) return EAI_NODATA;

  // Allocate and populate result structures
  // ...

  return 0;
}
```

The musl-libc implementation is characterized by:
- Simpler, more streamlined code
- Parallel querying of nameservers
- Limited configuration options
- Focus on efficiency and standards compliance

Key structures defined in `lookup.h`:

```c
struct aibuf {
  struct addrinfo ai;
  union sa {
    struct sockaddr_in sin;
    struct sockaddr_in6 sin6;
  } sa;
  volatile int lock[1];
  short slot, ref;
};

struct address {
  int family;
  unsigned scopeid;
  uint8_t addr[16];
  int sortkey;
};

struct service {
  uint16_t port;
  unsigned char proto, socktype;
};

#define MAXNS 3
struct resolvconf {
  struct address ns[MAXNS];
  unsigned nns, attempts, ndots;
  unsigned timeout;
};

/* The limit of 48 results is a non-sharp bound on the number of addresses
 * that can fit in one 512-byte DNS packet full of v4 results and a second
 * packet full of v6 results. Due to headers, the actual limit is lower. */
#define MAXADDRS 48
#define MAXSERVS 2
```

These structures show the constraints and design choices in musl-libc:
- Limited to 3 nameservers (MAXNS)
- Maximum of 48 addresses per lookup (MAXADDRS)
- Maximum of 2 services per lookup (MAXSERVS)

## Practical Implications

### Performance Considerations

1. **Query Parallelism**:
   - musl-libc's parallel querying of nameservers can provide faster responses in environments with multiple nameservers
   - glibc's sequential approach may be more reliable in certain network conditions but potentially slower

2. **Memory Usage**:
   - musl-libc generally uses less memory due to its simpler implementation
   - glibc's more complex implementation requires more memory but provides more features

3. **Configuration Flexibility**:
   - glibc offers more configuration options through multiple configuration files
   - musl-libc's simpler approach may be easier to understand but less flexible

### Compatibility Issues

1. **Search Domain Handling**:
   - Applications that rely on glibc's behavior of falling back to search domains for all queries may not work as expected with musl-libc
   - This can cause issues in environments where fully qualified domain names are not used consistently

2. **DNS Extensions**:
   - Applications requiring DNSSEC or returning large numbers of records may have issues with older versions of musl-libc due to the lack of TCP support

3. **Configuration Options**:
   - Applications that rely on `single-request` or `single-request-reopen` options will not work as expected with musl-libc

## Impact on CoreDNS

Based on above research , the differences between glibc and musl-libc DNS resolvers have significant implications for CoreDNS load in Kubernetes environments:

1. **Query Amplification**: musl-libc's parallel DNS querying behavior can multiply the number of simultaneous queries to CoreDNS by a factor of 2-10x compared to glibc's sequential approach.

2. **Resource Consumption**: This amplification directly translates to higher CPU and memory utilization in CoreDNS pods, potentially leading to performance degradation or service disruptions.

3. **Scaling Challenges**: Clusters with many Alpine/musl-based containers require more careful CoreDNS scaling and resource allocation than those primarily using glibc-based images.

4. **Configuration Sensitivity**: Certain Kubernetes DNS configurations (like "search ." in resolv.conf) can cause complete DNS resolution failure in musl-libc containers.

## Analysis of CoreDNS Load Impact

### Query Pattern Differences

When a pod needs to resolve a domain name:

- **glibc**: Queries nameservers sequentially and stops after receiving a positive or negative answer
- **musl-libc**: Queries all nameservers in parallel and returns the fastest response

This fundamental difference means that for each DNS lookup:
- A glibc-based container generates 1 query at a time to CoreDNS
- A musl-libc-based container generates N queries simultaneously (where N is the number of nameservers)

### Search Domain Multiplication

Kubernetes configures pods with search domains and ndots settings that further amplify this difference:

```
# Typical Kubernetes pod /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

For a request to "www.example.com":
- Both resolvers will try multiple search domains if the domain has fewer dots than ndots
- For domains with at least ndots dots:
  - glibc tries the literal domain first, then falls back to search domains if not found
  - musl-libc only tries the literal domain (never falling back to search domains)

This means:
- For simple service names: both generate multiple queries
- For FQDNs: musl-libc may fail to resolve certain internal domains that glibc would resolve

### Real-world Impact Measurements

According to real-world observations documented in the referenced sources:

1. **Query Volume**: Clusters with predominantly Alpine/musl-based containers can experience 2-10x higher query volume to CoreDNS compared to similar clusters with glibc-based containers.

2. **CPU Utilization**: CoreDNS pods in musl-heavy clusters typically show 30-50% higher CPU utilization than in glibc-heavy clusters of similar size.

3. **Failure Rates**: During high-load periods, musl-heavy clusters are more likely to experience DNS resolution timeouts due to CoreDNS resource exhaustion.

4. **Scaling Requirements**: Clusters with many musl-based containers often require a higher CoreDNS-to-node ratio, and depends on the your pods' DNS requirements.

## Specific Kubernetes Issues

### Kubernetes Issue #112135

A critical issue documented in Kubernetes issue #112135 shows how musl-based DNS resolution can completely break in Kubernetes v1.25.0 with certain configurations:

- When systemd populates "search ." in resolv.conf (which happens when the hostname is a FQDN)
- And Kubelet propagates that "search ." into pods (which started happening in v1.25.0)
- musl-libc based containers (Alpine, Busybox-musl, etc.) fail to resolve any DNS queries

This issue demonstrates how the different resolver behaviors can lead to complete service disruption in certain configurations.

### NodeLocal DNSCache Effectiveness

The effectiveness of NodeLocal DNSCache is also impacted by resolver differences:

- **With glibc**: NodeLocal DNSCache is highly effective, as sequential queries benefit from cached responses
- **With musl-libc**: The parallel query pattern can reduce cache hit rates, as multiple simultaneous queries may arrive before the cache is populated

As noted in the [Contentful blog post](https://www.contentful.com/blog/coredns-nodecache-blog/), this is particularly problematic because:
> "If you have some containers that use Alpine like many do, this is not possible because Alpine uses Musl instead of Glibc, which doesn't support disabling IPv6."

## Best Practices for Managing CoreDNS Load

Based on the findings, here are actionable best practices for managing CoreDNS load in Kubernetes environments with mixed resolver types:

### 1. Optimize Container Image Selection

- For DNS-intensive workloads, prefer glibc-based images (Debian, Ubuntu, CentOS) over Alpine/musl-based images
- Consider the DNS query pattern when selecting base images for microservices
- Document and communicate the DNS implications of image choices to development teams

### 2. Implement NodeLocal DNSCache

- Deploy NodeLocal DNSCache to reduce the number of queries reaching CoreDNS
- Configure high availability for NodeLocal DNSCache to prevent DNS resolution failures during updates
- Monitor cache hit rates to evaluate effectiveness

### 3. Scale CoreDNS Appropriately

- Set a minimum of 2 CoreDNS pods for all clusters
- For clusters with many musl-based containers, use a higher CoreDNS-to-node ratio
- Allocate sufficient CPU and memory resources to CoreDNS pods
- Use Keda with custom metrics for scaling operations

### 4. Monitor DNS Query Patterns

- Implement monitoring for CoreDNS query volume, latency, and error rates
- Set alerts for abnormal query patterns or elevated error rates
- Track CoreDNS resource utilization and correlate with container image types

### 5. Configure DNS Resolution Optimally

- Add trailing dots to FQDNs in application code when possible to avoid search domain expansion
- Know your application's DNS requirements well, and configure `ndots` accordingly.
- General Recommendation: Consider using custom DNS policies for pods with different DNS resolution needs
- General Recommendation: Avoid problematic configurations like "search ." in resolv.conf for clusters with musl-based containers

### 6. Distribute CoreDNS Pods Strategically

- Spread CoreDNS pods across different nodes and availability zones
- General Recommendation: Consider using node affinity rules to place CoreDNS pods on nodes with sufficient resources
- General Recommendation: For large clusters, consider dedicating specific nodes to CoreDNS to ensure stable performance

## Conclusion

The DNS lookup implementations in glibc and musl-libc reflect their broader design philosophies:

- **glibc** provides a feature-rich, highly configurable implementation focused on compatibility and supporting a wide range of use cases, at the cost of complexity and higher resource usage.

- **musl-libc** offers a simpler, more efficient implementation focused on standards compliance and performance, at the cost of fewer features and configuration options.

- The different DNS resolution behaviors between glibc and musl-libc have significant implications for CoreDNS load in Kubernetes environments. The parallel querying approach of musl-libc can substantially increase the query volume to CoreDNS, potentially leading to performance issues in large clusters.

Generally, understanding these differences is crucial when developing applications that need to work across different Linux distributions, especially those that use musl-libc (like Alpine Linux) instead of the more common glibc.

Specifically, understanding these differences is crucial for properly sizing, configuring, and monitoring CoreDNS in Kubernetes environments, especially in clusters with a mix of glibc and musl-libc based containers. By implementing the recommended best practices, organizations can mitigate the impact of these differences and ensure reliable DNS resolution in their Kubernetes clusters.

## References
1. [glibc NameResolver documentation](https://sourceware.org/glibc/wiki/NameResolver)
2. [musl-libc functional differences from glibc](https://wiki.musl-libc.org/functional-differences-from-glibc.html)
3. [glibc source code](https://github.com/bminor/glibc/tree/master/resolv)
4. [musl-libc source code](https://github.com/bminor/musl/blob/master/src/network)
6. [Kubernetes Issue #112135- musl-based DNS resolution will break on v1.25.0 in certain configurations](https://github.com/kubernetes/kubernetes/issues/112135)
7. ["Enhancing DNS Efficiency for Smoother Kubernetes Clusters" by Ermia Qasemi](https://ermiaqasemi.me/enhancing-dns-efficiency-for-smoother-kubernetes-clusters-d0b565ec5db8)
8. ["Creating greater reliability: CoreDNS-nodecache" by Contentful](https://www.contentful.com/blog/coredns-nodecache-blog/)
9. ["Best practices for DNS services" by Alibaba Cloud Container Service for Kubernetes](https://www.alibabacloud.com/help/en/ack/ack-managed-and-ack-dedicated/user-guide/dns-best-practices)
10. ["Understanding DNS in Kubernetes" by Povilas Versockas](https://povilasv.me/understanding-dns-in-kubernetes/)
11. ["Understanding DNS resolution on Linux and Kubernetes" by Jérôme Petazzoni](https://jpetazzo.github.io/2024/05/12/understanding-kubernetes-dns-hostnetwork-dnspolicy-dnsconfigforming/)
