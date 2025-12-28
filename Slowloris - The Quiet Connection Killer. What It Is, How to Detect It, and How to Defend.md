# Meta description

Slowloris is a stealthy application-layer DoS technique that holds connections open to exhaust server resources. Learn how it works conceptually, what symptoms to monitor, and practical, safe mitigations and detection strategies.

---

## Introduction

Not every denial-of-service attack looks like a flood of traffic. Slowloris is a classic example of an attack that quietly ties up server resources by opening and holding many slow or incomplete HTTP connections. Because it consumes worker slots and connection state rather than raw bandwidth, Slowloris can be effective against poorly configured web servers and application stacks.

This post explains the technique at a conceptual level, shows how defenders can spot it in logs and metrics, and covers proven mitigations you can apply today — all without sharing exploit code or instructions to carry out attacks.

> **Safety note:** the discussion below explains concepts only. Do not attempt to test attacks against systems you do not own or have explicit written permission to test.

---

## Concept: how Slowloris works (high level)

At its core, Slowloris exploits how many web servers allocate and manage per-connection resources:

1. A client opens an HTTP connection to the server.
    
2. Instead of sending a full request immediately, the client sends a partial request (for example, some headers) and then sends subsequent bytes extremely slowly or intermittently.
    
3. The server keeps the connection open while waiting for the full request; depending on the server and configuration, a worker thread, process, or connection slot remains allocated for that client.
    
4. Repeat the above from many clients (or many sockets from a single host) and the server’s connection/thread pool becomes exhausted — legitimate clients cannot obtain free slots, making the site slow or unreachable.
    

Crucially, Slowloris does **not** need large amounts of bandwidth. Its power comes from holding connection state and server resources open.

---

## Why it succeeds

Slowloris can be effective when:

- The web server has a limited number of maximum simultaneous connections or worker threads.
    
- Timeouts are permissive (long `client_body_timeout`, `request_timeout`, etc.), so half-open or very slow requests tie up resources for a long time.
    
- There’s no edge protection (CDN, WAF) or the edge is configured to pass slow clients through to the origin.
    

Modern deployments with CDNs, reverse proxies, and tuned timeouts are far less vulnerable — but misconfigurations still exist in the wild.

---

## What defenders see — symptoms & artifacts

If you suspect a Slowloris-style event, look for these observable indicators (use examples from your own tooling and log formats):

- **High number of long-duration TCP connections**: check connection duration histograms; a heavy tail of long-lived connections is suspicious.
    
- **Many `ESTABLISHED` connections with low or near-zero throughput**: low bytes/sec but lots of concurrent established sessions.
    
- **Worker/thread exhaustion or `502`/`503` errors**: web server logs or metrics showing worker pool saturation.
    
- **Access logs with many partial requests**: repeated connections with headers sent but no or tiny bodies; timestamps show long idle periods.
    
- **Spike in server queue length or request backlog**: reverse proxy or server metrics showing queue growth.
    
- **Multiple connections from few IPs (or from many IPs but with similar slow patterns)**: look for patterns in per-IP connection counts and durations.
    

Collect context: timestamps, connection source IPs, request durations, server status pages (e.g., `nginx` stub status), and any available packet captures or tcpstate snapshots — but only in controlled, authorized investigations.

---

## Detection rules and monitoring ideas

Set up monitoring and alerts that focus on the unique resource-exhaustion profile of Slowloris:

- Alert when median/95th percentile connection duration increases significantly.
    
- Alert on the ratio: `established_connections / available_worker_slots` exceeding a threshold.
    
- Log and monitor the number of connections per IP and per subnet; alert on sudden increases or many long-lived connections from a single entity.
    
- Track `request_rate` vs `active_connections` — a growing gap (many connections but low completed requests) is a red flag.
    
- Correlate server logs, reverse proxy metrics, and network telemetry (flow logs) to identify distributed patterns.
    

Avoid relying on single signals; combine indicators to reduce false positives.

---

## Practical mitigations (defensive, actionable)

Here are layered defenses that reduce exposure and keep services available:

1. **Edge protections (best first):**
    
    - Put services behind a CDN or reverse proxy that can terminate and manage client connections at the edge (Cloudflare, Fastly, Akamai, etc.). These platforms handle vast numbers of connections and can filter slow-client patterns before they reach your origin.
        
2. **Harden your reverse proxy / load balancer:**
    
    - Configure aggressive but safe `client_header_timeout`, `client_body_timeout`, and `keepalive_timeout` settings. The exact values depend on application behavior; test for legitimate slow clients like mobile networks.
        
    - Enable per-IP connection limits at the proxy layer (with sensible per-IP rate limiting and connection caps).
        
3. **Server settings:**
    
    - Tune server worker counts and keepalive settings to balance resource usage and legitimate traffic.
        
    - Use evented servers (e.g., `nginx` or evented modes) over thread-per-connection models when possible; evented architectures are more resilient to many idle connections.
        
4. **Rate limiting & per-connection policies:**
    
    - Apply rate limits for requests per second, concurrent connections per client, request timeouts, and “initial read” timeouts that drop super-slow clients early.
        
5. **Network-level protections:**
    
    - Use SYN cookies to mitigate SYN floods (different but complementary).
        
    - Ensure firewalls and load balancers are tuned to drop suspiciously long idle connections if they match known abusive patterns.
        
6. **Instrumentation & automation:**
    
    - Instrument connection durations, worker usage, and queue lengths. Automate scaling or mitigations when worker exhaustion is detected (e.g., auto-scale front ends or enable tighter timeouts automatically under attack).
        
7. **WAF / behavioral rules:**
    
    - Web Application Firewalls can detect slow-read patterns and block or rate-limit them. Use behavioral rules that focus on per-connection duration anomalies rather than crude signature matches.
        

---

## How to test safely (lab guidance)

If you need to reproduce or test Slowloris-like behavior for hardening and detection, follow strict safety discipline:

- **Use isolated test environments** only — air-gapped VM or cloud tenant you control.
    
- **Use legal and documented test harnesses** or traffic generators designed for load testing (and ensure you understand what they do). Never use tools against third-party infrastructure without written authorization.
    
- **Sanitize test inputs and document expected effects.** Focus on telemetry you can collect (logs, metrics) and tune protections incrementally.
    
- **Rotate/restore test systems** after tests (revert snapshots) to avoid residual misconfigurations.
    

I won’t provide or assist in deploying exploit code or attack scripts. If you want help building a safe test harness that simulates slow connections at the network level without performing attacks on third parties, I can help design one conceptually (no attack code).

---

## Incident response checklist (short)

If you suspect an active Slowloris-style event:

1. Capture affected server metrics and connection tables (timestamps).
    
2. Export proxy and server logs for the relevant time window.
    
3. If possible, switch traffic to an upstream CDN/proxy or enable stricter edge rules to protect origin.
    
4. Apply temporary per-IP connection caps and shorter timeouts at the edge.
    
5. Coordinate with hosting/ISP if the attack persists or is distributed.
    
6. After containment, analyze logs and patch any server configuration weaknesses.
    

---

## Conclusion

Slowloris is a reminder that many successful attacks are _quiet_ and exploit resource management decisions rather than raw bandwidth. Defending against it is mostly about layered, sensible defaults: terminate and manage client connections at the edge, enforce sensible timeouts, monitor connection durations, and apply rate limits and WAF rules as needed.

If you’d like, I can help you next with any of these follow-ups (all defensive and safe):

- A publishable **how-to** for tuning `nginx`/`HAProxy` timeouts and connection limits (configuration examples are safe to share).
    
- A **detection playbook** with concrete metric thresholds and sample alert rules for Prometheus/Grafana, Datadog, or CloudWatch.
    
- A **Manim** storyboard/animation script that visualizes Slowloris (safe, conceptual; ideal for videos or training).
    

Which of those would you like me to produce next?