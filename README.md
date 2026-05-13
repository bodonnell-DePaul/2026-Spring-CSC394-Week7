<!-- _class: lead -->

# Application Scalability

## From Data Center to Global Cloud

CSC 394 - Software Projects - Week 7

---

<!-- header: 'CSC 394 - Week 7' -->
<!-- footer: 'Application Scalability' -->

# Today's Big Idea

Scalability is not only "make the server bigger."

It is the ability to keep the application useful as users, data, traffic, geography, and team complexity grow.

| Dimension | Scaling Question |
| --- | --- |
| Compute | Can the app handle more work? |
| Data | Can state stay correct and fast? |
| Geography | Can users far away get a good experience? |
| Operations | Can the team see, secure, and repeat the system? |

---

# Learning Goals

By the end of today, you should be able to explain:

1. How scaling changes from data center to cloud
2. Local, regional, multi-regional, and global application patterns
3. Vertical scaling versus horizontal scaling
4. Containers, VMs, and PaaS tradeoffs
5. Why observability becomes mandatory as systems scale
6. How security changes when one app becomes many moving parts

---

<!-- _class: lead -->

# Part 1

## From Data Center to Cloud

---

# The Data Center Mental Model

In a traditional data center, capacity is planned ahead of time.

| Constraint | What It Means |
| --- | --- |
| Hardware lead time | Servers must be bought, shipped, racked, and configured |
| Fixed capacity | You pay for peak even during quiet periods |
| Manual operations | Networking, patching, backups, and recovery need staff process |
| Physical location | Users far away experience more latency |

Data centers can scale, but scaling is slower, more capital-intensive, and more operationally heavy.

---

# The Cloud Mental Model

Cloud platforms turn infrastructure into an API.

| Data Center Habit | Cloud Habit |
| --- | --- |
| Buy servers before demand arrives | Add capacity when demand appears |
| Configure machines by hand | Define resources in files and automation |
| Treat environments as special | Recreate environments from repeatable steps |
| Watch servers | Watch services, user experience, and cost |

The cloud does not remove architecture work. It changes which architecture decisions matter most.

---

# The Scalability Ladder

| Stage | Typical Shape | New Problem |
| --- | --- | --- |
| Local | One developer machine or one small server | Consistent setup |
| Regional | One cloud region, managed database, load balancer | Reliability and failover |
| Multi-regional | Active in more than one region | Data consistency and deployment coordination |
| Global | Users routed worldwide through edge networks | Latency, compliance, abuse, and cost control |

Each step adds capability and operational complexity.

---

# Local Scalability

Local scalability means the app can grow on one machine or one small deployment without architectural drama.

Good local-scale habits:

- Keep configuration in environment variables
- Use a real database, not files for important state
- Make setup repeatable with scripts or Docker Compose
- Avoid hidden state in one developer's machine
- Document the commands a teammate needs

For student teams, this is the first scalability milestone.

---

# Regional Scalability

Regional scalability means the app serves users reliably from one cloud region.

Core components:

- Load balancer or PaaS router
- Multiple app instances when needed
- Managed relational database
- Object storage for uploads
- Health checks and uptime monitoring
- CI/CD deployment workflow

This is the practical target for most CSC 394 projects.

---

# Multi-Regional Scalability

Multi-regional systems run in more than one cloud region.

New architectural questions:

- Which region handles each user?
- Is data active-active or primary-replica?
- What happens when one region is down?
- How do deployments roll out safely across regions?
- Which logs, metrics, and alerts represent the whole system?

Multi-region is where "just add another server" stops being enough.

---

# Global Scalability

Global applications optimize for users across continents.

Common ingredients:

- CDN and edge caching for static content
- Geo-aware routing to nearby regions
- Replicated databases or globally distributed data stores
- Regional compliance boundaries
- Abuse detection and rate limiting at the edge
- Centralized observability across all regions

The hardest part is usually not compute. It is data, security, and operations.

---

# What Gets Harder at Each Step?

| Move | Architect's New Difficulty |
| --- | --- |
| Local to regional | Externalize state and automate setup |
| Regional to multi-regional | Handle failover, replication, and partial outages |
| Multi-regional to global | Balance latency, consistency, compliance, and cost |

Architects are paid to manage tradeoffs, not to find one perfect scaling pattern.

---

<!-- _class: lead -->

# Part 2

## Vertical and Horizontal Scaling

---

# Vertical Scaling: Scale Up

Vertical scaling gives one machine more power.

| What Changes | Example |
| --- | --- |
| CPU | More cores for request processing |
| Memory | More room for app runtime and cache |
| Disk I/O | Faster reads and writes |
| Network | Higher throughput between services |

It is simple because the app still thinks it is running in one place.

---

# Vertical Scaling: Strengths and Limits

| Strength | Limit |
| --- | --- |
| Simple mental model | Hardware ceiling arrives eventually |
| Often no code changes | Bigger machines get expensive quickly |
| Good for databases and early-stage apps | Scaling may require restarts or downtime |
| Fewer distributed-system problems | One large server can still be one failure point |

Vertical scaling buys time. It does not remove the need for good architecture.

---

# Horizontal Scaling: Scale Out

Horizontal scaling adds more copies of the same service.

```
User traffic -> Load balancer -> App 1
                              -> App 2
                              -> App 3
                              -> Shared database
```

The goal is interchangeability: any healthy app instance can handle any request.

---

# Horizontal Scaling: Requirements

For horizontal scaling to work, the app must be mostly stateless.

| Concern | Scalable Approach |
| --- | --- |
| Database | Shared managed database |
| File uploads | Object storage such as S3 or Cloudinary |
| Sessions | JWT, Redis, or database-backed sessions |
| Cache | Shared cache such as Redis when correctness matters |
| Configuration | Environment variables and secret stores |

If an instance disappears, another instance should be able to continue the work.

---

# Vertical vs. Horizontal

| Question | Vertical | Horizontal |
| --- | --- | --- |
| How do we add capacity? | Bigger machine | More machines |
| Operational complexity | Lower | Higher |
| Failure isolation | Weaker | Stronger if designed well |
| Cost curve | Can jump sharply | Can grow more gradually |
| Main design pressure | Machine limits | Statelessness and coordination |

Most production systems use both: scale up components that need it, scale out services that can be replicated.

---

# Load Balancers

A load balancer is the traffic director for horizontally scaled services.

It can:

- Send requests only to healthy instances
- Spread load across multiple app copies
- Terminate HTTPS before traffic reaches the app
- Support zero-downtime deploys
- Route traffic by path, host, or geography

PaaS platforms often include this automatically. VM and container platforms usually make it more explicit.

---

# State Is the Scaling Boundary

Compute is easy to duplicate. State is not.

| State Type | Scaling Risk | Safer Pattern |
| --- | --- | --- |
| Uploaded files | Lost on restart or hidden on one instance | Object storage |
| Login sessions | User appears logged out on another instance | JWT or shared session store |
| In-memory cache | Different answers from different instances | Shared cache or cache invalidation |
| Background jobs | Duplicate work or missed work | Queue with worker coordination |

When scaling fails, state is usually where it fails.

---

<!-- _class: lead -->

# Part 3

## Compute Models: VMs, Containers, PaaS

---

# The Compute Spectrum

| Model | Control | Operations Work | Good Fit |
| --- | --- | --- | --- |
| VM | Highest | Highest | Legacy apps, custom servers, unusual runtimes |
| Container | High | Medium | Portable services and reproducible environments |
| PaaS | Medium | Low | Web apps and APIs that fit platform defaults |

More control usually means more responsibility.

---

# Virtual Machines

A VM is a full server-like machine in the cloud.

You manage:

- Operating system updates
- Runtime installation
- Firewall rules and network access
- Process managers and restarts
- Disk space, backups, and patching
- Scaling strategy

VMs are flexible, but the team owns the server discipline.

---

# Containers

A container packages the application and its runtime assumptions into an image.

Containers help with:

- Matching local, CI, staging, and production environments
- Shipping the same artifact repeatedly
- Running several services with Docker Compose
- Moving workloads between platforms more easily

Containers do not automatically solve scaling. They make packaging and deployment more repeatable.

---

# PaaS

Platform-as-a-Service lets teams deploy without managing most server details.

PaaS commonly handles:

- Runtime hosting
- HTTPS and routing
- Basic logs
- Health checks and restarts
- Environment variables
- Simple horizontal scaling

For most student web apps, PaaS plus a managed database is the fastest path to a reliable deployment.

---

# Dockerfile Example

This is the only code we need today. Focus on the ideas, not the syntax.

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
ENV NODE_ENV=production
USER app
CMD ["node", "dist/index.js"]
```

---

# What the Dockerfile Teaches

| Pattern | Why It Matters |
| --- | --- |
| Multi-stage build | Build tools stay out of the final runtime image |
| Copy dependency files first | Docker can cache expensive dependency installs |
| Deterministic install | The image matches the lockfile |
| Non-root user | A compromised app has less power inside the container |
| Environment variable | Behavior changes by environment, not by editing code |

Containerization is also a security and reproducibility practice.

---

<!-- _class: lead -->

# Part 4

## Observability and Operations

---

# Monitoring vs. Observability

Monitoring asks: is the system healthy?

Observability asks: can we understand what happened inside the system from its outputs?

| Signal | What It Reveals |
| --- | --- |
| Logs | Events and context: what happened? |
| Metrics | Trends and thresholds: how often, how slow, how many? |
| Traces | Request path: where did time or failure occur? |

As scale increases, guessing becomes too expensive.

---

# Logging as a Scaling Tool

Logs should help answer operational questions quickly.

Good production logs include:

- Timestamp
- Severity level
- Request or correlation ID
- User or team identifier when appropriate
- Operation name
- Error details without leaking secrets

Structured logs are easier to filter, aggregate, and alert on than plain text debugging messages.

---

# Metrics That Matter

Start with a small set of high-signal metrics.

| Metric | What It Tells You |
| --- | --- |
| Request rate | How much traffic is arriving |
| Error rate | How much traffic fails |
| Latency | How long users wait |
| Saturation | How close resources are to limits |
| Queue depth | Whether background work is falling behind |

These map well to the question students ask during demos: "Is the app holding up?"

---

# Alerts and Health Checks

Not every error deserves a panic.

| Signal | Action |
| --- | --- |
| Health check failing | Alert immediately |
| Error rate above threshold | Investigate soon |
| Latency rising | Watch and diagnose |
| One-off 404 or bot traffic | Log and ignore |

For student projects, a health endpoint plus UptimeRobot is a strong starting point.

---

# Infrastructure as Code and Repeatability

Scaling also means recreating the system without relying on someone's memory.

| Manual Click-Ops | Repeatable Infrastructure |
| --- | --- |
| Settings live in dashboards | Settings live in files or scripts |
| Hard to review | Pull requests show intent |
| Easy for staging and production to drift | Environments can be aligned |
| Disaster recovery depends on memory | Resources can be recreated |

At student scale: GitHub Actions, Docker Compose, `.env.example`, and a clear README are enough.

---

<!-- _class: lead -->

# Part 5

## Security Changes as You Scale Out

---

# Security at Local Scale

On one small app, security often starts with basics:

- Do not commit secrets
- Validate input
- Hash passwords
- Configure CORS deliberately
- Keep dependencies updated
- Use HTTPS in deployed environments

These still matter later. Scaling adds more places where mistakes can hide.

---

# Security at Regional Scale

Once the app is deployed and reachable, the attack surface grows.

New priorities:

- Least-privilege service credentials
- Private database access
- Rate limiting and abuse prevention
- Centralized logs for suspicious behavior
- Automated dependency and secret scanning
- Backups and restore drills

Security becomes an operational habit, not a checklist before launch.

---

# Security at Multi-Region and Global Scale

Global systems face uneven laws, users, traffic, and threat patterns.

Architects must consider:

- Data residency and privacy requirements
- Regional identity and access boundaries
- Edge-level DDoS and bot protection
- Key rotation across environments
- Incident response across time zones
- Audit trails that survive regional outages

The larger the system, the more security depends on design and automation.

---

# Scaling Tradeoffs Architects Face

| Tradeoff | Why It Is Hard |
| --- | --- |
| Latency vs. consistency | Fast local reads can conflict with globally correct data |
| Reliability vs. cost | Extra regions and redundancy are not free |
| Autonomy vs. governance | Teams move faster with freedom but need shared guardrails |
| Security vs. usability | Strong controls can slow delivery if poorly designed |
| Automation vs. blast radius | Automated mistakes can scale too |

Good architecture makes these tradeoffs visible before they become emergencies.

---

<!-- _class: lead -->

# Part 6

## Applying This to Student Projects

---

# TaskBoard Evolution Path

| Stage | TaskBoard Shape |
| --- | --- |
| Local | React, Express, and Postgres run locally or in Compose |
| Regional | React on Vercel, API on Render, Postgres managed in one region |
| Multi-instance | API scales horizontally behind the platform router |
| Multi-regional | Read replicas, regional routing, replicated object storage |
| Global | CDN, edge security, global observability, compliance boundaries |

Your project probably stops at regional. Your design choices should not block the next step.

---

# Team Checklist for This Week

1. Identify anything stored on local disk or in server memory
2. Add or verify a health endpoint
3. Replace casual logging with structured logging in key routes
4. Confirm the database and uploads are external services
5. Document local setup and deployment steps
6. Set billing alerts for paid cloud accounts
7. Decide whether Docker Compose would reduce team setup friction

This is operational quality work. It makes the demo less fragile.

---

# Opus 4.7 Cross-Reference Lens

Use this review lens to check whether the deck stays aligned with the course goals.

| Lens | Deck Coverage |
| --- | --- |
| Clear student workflow | Local to regional project checklist |
| High-level architecture | Data center, cloud, regions, global systems |
| Practical engineering topics | Scaling, compute models, observability, IaC |
| AI-era verification mindset | Repeatable setup, logs, health checks, reviewable infra |
| Security as design | Secrets, least privilege, edge protection, compliance |

The deck intentionally favors architectural judgment over code mechanics.

---

# Key Takeaways

| Topic | Remember This |
| --- | --- |
| Scaling | More capacity only helps if state is designed correctly |
| Vertical vs. horizontal | Scale up for simplicity, scale out for resilience and growth |
| Compute models | VMs maximize control, PaaS maximizes speed, containers improve portability |
| Observability | Logs, metrics, and traces replace guessing |
| Security | More instances, regions, and users require stronger automation and boundaries |

Next week: security, performance, and technical debt.
