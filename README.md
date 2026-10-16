# Docker VPS: what specs you actually need, why KVM matters, and how to run a full Compose stack for around $45 a year

A Docker VPS is a virtual private server with a real kernel and full root access, where you install Docker Engine yourself and run your containers on your own terms. Picking one comes down to three things: the host must use KVM virtualization, the RAM has to match what your stack actually uses, and the billing model decides what you'll really pay in year two. On the first two, the requirements are well settled. On the third, the market splits into monthly cloud providers charging $4–9 forever, and budget annual hosts — RackNerd being a prominent example — that sell comparable KVM machines for roughly one monthly fee per year.

> **Quick pick:** most people searching for a Docker VPS want a cheap, reliable box for self-hosting. That's the RackNerd KVM 4 GB tier — full root, real kernel, promo pricing around **$44/year** (roughly $3.70/month). [👉 Check RackNerd's current promo lineup and datacenter locations](https://bit.ly/my-rn) — the specials rotate through the year, so the order page is always the source of truth. If you need managed tooling or EU-first latency, the comparison further down covers when paying more makes sense.

## Why shared hosting can't run Docker (and what exit code 137 means)

It fails for a structural reason, and the failure has a signature you may have already seen. Shared hosting pools physical RAM across all tenants under a single kernel, with each tenant's memory capped through cgroups — Linux control groups. When your containers collectively cross that ceiling, the kernel's OOM killer terminates them, and Docker reports exit code 137. It will happen again on the next deploy no matter how you tune the compose file, because the ceiling sits above anything you control.

Shared hosting also commonly restricts the kernel-level features Docker depends on: user namespaces, network namespaces, seccomp enforcement. Even where installation technically succeeds, a multi-container setup starves itself against every other tenant on the box.

The fix is a VPS with dedicated RAM and full root — specifically a KVM VPS. KVM gives the virtual machine its own kernel with real cgroups, namespaces, and overlay filesystem support, which is everything Docker wants underneath it. OpenVZ-based VPS plans share the host kernel with other tenants, and Docker either gets blocked or runs degraded on them. This is why every serious comparison for this topic filters for KVM first, and why a plain Linux box with root beats any "container platform" that wedges you into someone else's kernel.

## What specs a Docker VPS actually needs

RAM is the primary constraint. A single application container runs in 512 MB–2 GB. A realistic Docker Compose stack — your app plus PostgreSQL, Redis, and a reverse proxy — needs 4–8 GB once you stop counting optimism as memory. Current comparisons converge on these ranges:

| Workload | Min RAM | Recommended RAM | Storage |
| --- | --- | --- | --- |
| Single container (dev/staging) | 1 GB | 2 GB | 20 GB SSD |
| Docker Compose, 2–4 services | 2 GB | 4 GB | 40 GB NVMe |
| Compose stack in production (5+ services) | 4 GB | 8 GB | 80 GB NVMe |
| K3s single-node Kubernetes | 4 GB | 8 GB | 60 GB NVMe |

Storage matters, but differently than you might expect. Docker volumes for databases write small, random chunks constantly — NVMe handles that, SATA SSD is the floor, and a spinning disk will make a loaded database miserable.

Two details most spec tables skip:

- **Image layers and build cache grow fast.** If you build images on the server itself, budget 160 GB or more and prune unused images regularly. A 40 GB disk fills up quicker than you'd think.
- **CPU steal is the hidden tax on oversold hosts.** The symptom is inconsistent build times and random latency spikes. One detailed comparison in this space specifically flags IOPS caps on some budget providers — Contabo's ~1,300 IOPS cap, for instance, visibly degrades a loaded PostgreSQL container. Fine for stateless services; painful for database volumes.

## RackNerd plans and pricing: what a Docker-ready VPS costs

RackNerd's model is deliberately plain: KVM virtualization, full root access, a 1 Gbps port, multi-TB monthly transfer, and annual promo pricing that starts near $10 a year. There's no managed Docker dashboard and no per-container billing — you get a Linux box with your own kernel, which is exactly what the last section said Docker wants. Datacenters cover several US locations (Los Angeles, San Jose, Seattle, Dallas, Chicago, New York, and Atlanta among them) plus Amsterdam in Europe.

Here's the current shape of the lineup:

| Plan | RAM | Storage | Typical promo price | Fits | Order |
| --- | --- | --- | --- | --- | --- |
| KVM 768 MB | 768 MB | ~10 GB SSD | ~$9–10/yr | Learning Docker, one tiny service | [ Order KVM 768 MB](https://bit.ly/my-rn) |
| KVM 1 GB | 1 GB | ~20 GB SSD | ~$11/yr | A single container with headroom | [ Order KVM 1 GB](https://bit.ly/my-rn) |
| KVM 2.5 GB | 2.5 GB | ~45 GB SSD | ~$26/yr | Light Compose stack, 2–3 services | [ Order KVM 2.5 GB](https://bit.ly/my-rn) |
| KVM 4 GB | 4 GB | ~65 GB SSD | ~$44/yr | Realistic stack: app + Postgres + Redis + proxy | [ Order KVM 4 GB](https://bit.ly/my-rn) |
| KVM 6.5 GB | 6.5 GB | ~100 GB SSD | ~$69/yr | Heavier stack or a first K3s node | [ Order KVM 6.5 GB](https://bit.ly/my-rn) |
| KVM 8 GB | 8 GB | ~130 GB SSD | ~$86/yr | Production Compose or K3s with room | [ Order KVM 8 GB](https://bit.ly/my-rn) |

*Figures reflect RackNerd's typical recent promo tiers; exact SSD sizes and transfer allowances shift a bit between promos and locations. Black Friday and New Year bring their best-known specials, occasionally with flash tiers under $10. The order page always shows the current table.*

The trade-off for these prices: you pay a year upfront, and the headline deals are annual promotions. One thing that separates RackNerd from first-term-teaser hosts is that the specials have historically renewed at the signup rate rather than jumping after year one — the recurring amount is printed at checkout, and it's worth a ten-second read before you pay. [👉 See RackNerd's current VPS pricing and locations](https://bit.ly/my-rn)

### Which RackNerd plan for which Docker workload

- **768 MB / 1 GB:** learning, experiments, one small service. A single app container needs 512 MB–2 GB, so 1 GB is the honest floor.
- **2.5 GB:** two or three small services without a hungry database. Workable, but tight for PostgreSQL under real load.
- **4 GB:** the tier most self-hosters actually want. A stack with a database sits in the 4–8 GB band, and 4 GB is its entry point. It also happens to cost about what DigitalOcean charges *per month* for a 1 GB droplet.
- **6.5–8 GB:** production Compose with monitoring, or single-node K3s. K3s itself only needs ~1 GB minimum, but real workloads want the recommended 8 GB.

If you're just experimenting, buy the smallest tier. You can always order a bigger plan later; the reverse is harder on your wallet.

## How RackNerd compares to the usual monthly-priced Docker hosts

On price per year, RackNerd is difficult to beat. Where it gives ground is managed tooling and European latency. The honest picture:

| Provider | Entry price | What you get | Billing | Best for |
| --- | --- | --- | --- | --- |
| RackNerd | ~$11/yr (1 GB) · ~$44/yr (4 GB) | KVM, full root, 1 Gbps, multi-TB transfer | Annual promo | Lowest yearly cost for self-hosting |
| Hetzner | CX22, €3.79/mo | 2 vCPU, 4 GB RAM, 40 GB NVMe, 20 TB traffic | Monthly | EU latency; the default pick in self-hosting communities |
| Contabo | Core VPS 4, €5.24/mo | 4 vCPU, 8 GB RAM, 200 GB SSD, flat-rate traffic | Monthly | Most RAM per euro; mind the ~1,300 IOPS cap for DB-heavy work |
| DigitalOcean | $6/mo | 1 GB RAM, 25 GB NVMe | Monthly | Quick test spins, polished API and docs |
| Hostinger | KVM 1, $6.49/mo (renews $11.99/mo) | 1 vCPU, 4 GB RAM, 50 GB NVMe, managed Docker dashboard | Billed upfront | Beginners who want visual container management |

A few notes that matter more than the sticker prices:

- **Hetzner's CX22 lands within a couple of euros per year of RackNerd's 4 GB tier.** If your users are in Europe, latency alone can justify it. Hetzner is also the name that comes up most consistently across r/selfhosted, LowEndTalk, and Hacker News threads, with mature community tooling if your Compose setup grows toward Kubernetes.
- **Hostinger charges for convenience** — a visual Docker manager, one-click updates, weekly backups, and an AI assistant for server tasks. Note the renewal structure though: $6.49 first term, $11.99 after. The renewal line is where teaser pricing lives in this market.
- **Contabo gives you the most raw hardware per euro** with flat-rate traffic, and that's a genuine fit for RAM-hungry stateless workloads. The IOPS cap is the catch for database volumes.
- **DigitalOcean at $6/month for 1 GB works out to $72/year** — more than RackNerd's 4 GB tier costs per year, for a quarter of the RAM. You're paying for platform polish, not hardware. Vultr fills a similar niche with hourly billing, which is genuinely useful for short-lived test environments.

## Setting up Docker on a RackNerd VPS: the short version

Nothing special is required — a fresh KVM VPS with root becomes a Docker host in about fifteen minutes.

1. **Pick the OS.** Ubuntu 24.04 or Debian 13 both ship current kernels and both have official Docker apt repositories.
2. **Install Docker Engine from Docker's own repository** (download.docker.com), including the compose plugin — skip the distribution packages, which lag behind current releases.
3. **Describe your stack in one compose file** — services, volumes, networks — and bring it up with `docker compose up -d`.
4. **Put a reverse proxy in front.** Caddy if you want a two-line config with automatic TLS; Traefik if you prefer defining routes with labels on the containers. Portainer on top if you want a web UI.
5. **Set up storage hygiene now:** prune unused images and build cache on a schedule, and dump your volumes with cron so a bad deploy never costs you data.
6. **Basic hardening:** SSH keys over passwords, and a firewall that only exposes 80, 443, and your SSH port.

[👉 Order a RackNerd KVM plan and follow along](https://bit.ly/my-rn) — pick the datacenter closest to whoever will actually use the containers.

## Docker VPS FAQ

### Can I run Docker on shared hosting?

No, not reliably. The cgroup memory ceiling triggers the kernel's OOM killer (exit code 137), and the kernel namespaces Docker requires are usually restricted. Any plan advertising Docker on shared infrastructure is fighting its own architecture.

### How many containers fit on a 4 GB VPS?

It's a question of combined resources, not container count. As a reference point, a 16 GB machine typically runs 10–20 mid-sized services, so 4 GB comfortably covers a 3–5 service Compose stack with a small database. Run `docker stats` for a week and let actual usage decide.

### Does the RackNerd promo price go up at renewal?

Historically, no — the specials renew at the signup rate for as long as you keep the service. That's the opposite of the first-term-discount pattern (see Hostinger's $6.49 → $11.99 jump), and it's the main reason the annual math holds up over multiple years. The recurring amount is stated at checkout; confirm it there.

### Do I need a managed Docker host?

Only if you never want to open a terminal. Hostinger's Docker manager handles visual deploys and updates starting at $6.49/month. If you're comfortable with SSH, a root KVM VPS does the same job for a fraction of the yearly cost.

### Can I run Kubernetes on a budget VPS?

Yes — K3s, rather than full Kubernetes. K3s is a certified lightweight distribution that ships as a single ~70 MB binary, includes Traefik and CoreDNS by default, and runs real workloads with 4 GB. Full kubeadm wants 2 GB per node just for the control plane. On RackNerd, that points you at the 6.5–8 GB tiers.

## So which one should you get?

- **Learning Docker or one small service** → RackNerd KVM 1 GB, around $11 a year. Don't buy more until a container asks for it.
- **A real stack with a database** → KVM 4 GB at around $44/year. App + Postgres + Redis + reverse proxy stops feeling cramped right about there.
- **Several stacks, monitoring, or K3s** → KVM 8 GB at around $86/year, still less than most monthly hosts charge for 2 GB.
- **European audience, or you want community-blessed tooling** → Hetzner's CX22 at €3.79/month is worth the premium. You want a dashboard instead of a terminal → Hostinger's KVM plans with the Docker manager.

Before you pay, two checks worth ten seconds each: confirm the datacenter location matches where your users actually are, and read the recurring amount on the order page — RackNerd's promo lineup rotates, and the page you're ordering from is always the current truth. [👉 Pick your RackNerd plan here](https://bit.ly/my-rn)
