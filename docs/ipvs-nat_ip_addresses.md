# How to Seamlessly Migrate a Service to a New Address

When you need to change a server's IP address but clients must continue connecting to the old one (temporary or permanent migration), engineers choose from several methods. All of them solve the same problem: deliver traffic coming to the old IP to the new server, and return responses to the client as if nothing has changed.

The choice of method depends on:

- The OSI model layer at which the service operates (TCP/UDP or HTTP/HTTPS)
- The need to preserve the client's real IP address
- Tolerable configuration and maintenance complexity
- Performance and high availability requirements

Below is a brief overview of popular approaches with their pros and cons.

### Simple Routing (Static Route / Policy Routing)

**Essence:** On the router through which traffic to the old IP passes, a static route to the new server (or a tunnel) is configured. Works at L3, knows nothing about ports and protocols.

**Pros:**

- Minimal latency, no additional overhead
- Transparent to any protocols (TCP, UDP, ICMP, SMB, FTP)
- No software installation required on the load balancer

**Cons:**

- Return traffic must go through the same path (symmetric routing required), otherwise the client will see a response from an unexpected IP
- Does not alter the source address — the server sees the client's real IP, but for successful bidirectional communication, SNAT often needs to be configured (i.e., an address translator is still required)
- Requires administrative access to routers

**Typical scenarios:** Temporary migration when both old and new IPs are in the same network or easily reachable through common routing.

### NAT on a Firewall or Router (iptables / nftables / UFW)

**Essence:** On the device receiving traffic destined for the old IP, DNAT (destination address translation) and SNAT (source address translation) are configured so packets are rewritten in both directions.

**Pros:**

- Simple implementation using standard Linux/FreeBSD kernel tools
- Works at L3/L4, no application-layer protocol knowledge required
- High performance

**Cons:**

- Must configure rules for each service (IP + port)
- Client's original IP is lost (replaced by the NAT device's address), which may be critical for auditing or geolocation
- Configuration becomes messy with many rules
- Manual VIP management on network interfaces required

**Typical scenarios:** TCP/UDP-level migration without the need to preserve the client's real IP.

### IPVS (IP Virtual Server) — L4 Load Balancing

**Essence:** A kernel-level Linux mechanism that directs incoming packets to real servers based on configured rules. Supports Direct Routing (DR), NAT, and tunneling modes. In this case, NAT mode with masquerading was used.

**Pros:**

- Very high performance (operates within the kernel)
- Can load balance both TCP and UDP
- Flexible routing based on fwmark (firewall marks) — can direct all traffic to a specific VIP without specifying ports
- Supports preserving the client's real IP (in DR mode or with transparent SNAT)

**Cons:**

- Requires kernel modules and manual configuration via the ipvsadm utility
- Debugging complexity (it's not always obvious why it doesn't work)
- In NAT mode requires conntrack and additional SNAT rules for return traffic
- Cannot terminate TLS/SSL or analyze the application layer

**Typical scenarios:** High-load TCP/UDP services, protocol-agnostic proxying, server migration while preserving old IP addresses.

### HAProxy / Nginx (L7 Reverse Proxy)

**Essence:** A user-space process that accepts client connections, analyzes HTTP/HTTPS headers, can modify URLs, add X-Forwarded-For headers, terminate SSL, and then establish a new connection to the backend server.

**Pros:**

- Full control over HTTP/HTTPS: routing by domain, URI, cookies
- Ability to display custom error pages, collect metrics, configure complex policies
- Easy to add SSL/TLS at the proxy without touching the backend
- Actively developed, good documentation, huge community

**Cons:**

- Only works with HTTP/HTTPS and partially TCP (in stream mode). Not suitable for arbitrary TCP/UDP without modifications
- Each connection is terminated and a new one is created — adds latency
- Client IP is lost (by default the backend sees the proxy's IP), requires X-Forwarded-For headers and application configuration
- Requires more CPU and memory resources compared to IPVS

**Typical scenarios:** Web services, HTTP/HTTPS load balancing, complex request routing, A/B testing.

### DNS Redirection (Changing A Records)

**Essence:** Simply update DNS so the domain name points to the new IP. Clients will start connecting to the new address directly.

**Pros:**

- Simplest solution, no network manipulation required
- Zero overhead

**Cons:**

- Only works if clients use domain names rather than IP addresses
- Long DNS caching (TTL), old IPs may remain cached for hours/days
- Not suitable if static IPs are used in configurations, whitelists, certificates

**Typical scenarios:** Planned web service migration with tolerance for gradual switchover.

### Tunnels (GRE, IPIP, VXLAN) and SDN Solutions

**Essence:** A tunnel is established between the old and new locations, and traffic destined for the old IP is encapsulated and delivered to the new server. Applicable when servers are geographically distributed.

**Pros:**

- Completely transparent to protocols, preserves the client's original IP
- No address translation required (tunnel-based routing)

**Cons:**

- Additional overhead due to encapsulation
- More complex to diagnose
- Requires MTU configuration and may cause packet fragmentation

**Typical scenarios:** Service migration between data centers, cloud migrations.

### Which Method to Choose?

- For a **universal TCP/UDP proxy with minimal latency** — IPVS (NAT or DR)
- For **HTTP/HTTPS with smart routing, SSL, and metrics** — HAProxy/Nginx
- For **quick and simple migration where losing the client IP is acceptable** — iptables/nftables SNAT+DNAT
- When **servers are in the same network and routing is manageable** — policy routing
- For **web-only services where switchover time is not critical** — DNS changes

Real-world projects often combine approaches: HAProxy for web, IPVS for auxiliary protocols, and DNS switching to complete the migration.

Below, this article demonstrates IPVS with nftables configuration in detail — a flexible solution that allowed seamless migration of SSH, PostgreSQL, Chrony, and SMB services to new addresses while preserving old IPs for clients. Step-by-step: installation, rules, masquerading, and unexpected pitfalls.
