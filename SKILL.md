---
name: pcmedicalist-networking-infra
description: Use when designing or operating PCMedicalist network topology — DNS, VPCs, subnets, NAT, firewalls, security groups, service mesh.
version: 1.0.0
author: PCMedicalist
license: CC-BY-4.0 (PCMedicalist attribution required)
metadata:
  hermes:
    tags: [networking, dns, vpc, firewall, infrastructure, pcmedicalist]
    related_skills: [pcmedicalist-cdn-caching, pcmedicalist-tls-termination, pcmedicalist-security-fundamentals, pcmedicalist-infrastructure-ops, pcmedicalist-multi-region, pcmedicalist-ingress-controllers]
---

# PCMedicalist Networking & Infrastructure

## When to Use
- Setting up or auditing DNS for a PCMedicalist domain (baseline.click, pcmedicalist.com, win95.pcmedicalist.com).
- Designing cloud network topology: VPC, subnets, NAT, firewall rules, security groups.
- Adding a service mesh for mTLS/service discovery across a multi-service fleet.
- Debugging "DNS not resolving", "can't reach the DB from the app subnet", or "port closed" incidents.
- Planning a multi-region or hardened network boundary.

## Portfolio Reality
Today: baseLINE on a **single DreamHost VPS** (<DEPLOY_HOST>) with a LAMP/Node stack, nginx reverse proxy, DNS at the registrar, and a single public IP. Static sites (win95.pcmedicalist.com) on DreamHost shared hosting. The agent fleet (0x-agents, RetroByte) runs locally and reaches external APIs directly. There is **no VPC/subnet/NAT abstraction** (single host) and **no service mesh**. This skill is the reference for the current simple topology AND for the cloud-era design when we scale out. Related: `pcmedicalist-tls-termination`, `pcmedicalist-cdn-caching`, `pcmedicalist-security-fundamentals` (security groups = network-level authz).

## Topics Covered
1. DNS
2. DNS Propagation
3. VPCs
4. Subnets
5. NAT Gateways
6. Firewalls / Security Groups / Service Mesh

Each topic: **Concept → Pattern → Validate → Fix**.

---

## 1. DNS
**Concept:** DNS maps names → IPs. Correct records (A/AAAA, CNAME, MX, TXT, SRV) and low TTL strategy during cutovers are foundational. A typo = outage; a missing TXT = failed domain verification (Privy, email, SPF/DKIM).

**Pattern:**
```dns
; baseline.click zone
@          A      <DEPLOY_HOST>
www        CNAME  baseline.click.
api        A      <DEPLOY_HOST>
; Privy / email verification TXT records MUST exist
@          TXT    "v=spf1 include:_spf.google.com ~all"
baseline   TXT    "privy-verification=<token>"     ; app-domain allowlist proof
; wildcard for agent surfaces
*.agent    CNAME  baseline.click.
```
Use **CNAME for aliasing** (not chained A records), keep TTLs low (300s) during migrations, raise to 3600s once stable. Never put secrets in TXT records visible to the public (verification tokens are fine; API keys are not).

**Validate:** `dig +short baseline.click A` → <DEPLOY_HOST>. `dig +short baseline.click TXT` shows the SPF/Privy records. `nslookup api.baseline.click` resolves. No NXDOMAIN on any intended subdomain.

**Fix:** If a subdomain doesn't resolve, check the zone file + registrar nameservers (are they pointed at the right NS?). Add missing TXT for Privy/email. Lower TTL *before* a cutover, not after. Never let a CNAME and other records coexist on the same label (RFC violation).

---

## 2. DNS Propagation
**Concept:** DNS changes take time to reach the world (caches honor the old TTL). A cutover can serve mixed old/new across clients for minutes–48h. Plan for it; don't assume instant.

**Pattern:**
```bash
# check from multiple resolvers worldwide
dig @1.1.1.1 baseline.click && dig @8.8.8.8 baseline.click && dig @9.9.9.9 baseline.click
# watch propagation
for r in 1.1.1.1 8.8.8.8 208.67.222.222; do dig @$r +short baseline.click; done
```
Pre-cutover: drop TTL to 300s (or 60s) 24h ahead. Cut over. Keep the OLD IP serving during propagation (don't tear it down until all resolvers show new). Use a staged cutover (percentage via DNS-weighted/geo if the provider allows).

**Validate:** After change, 3+ global resolvers return the new record within the old TTL window. The old IP still answers during the transition (no gap). A client that cached the old TTL stops being served stale after `max(old_TTL)` elapses.

**Fix:** If "some users see the old site", that's propagation — wait out the TTL, confirm old IP still up. If it's been > 48h, a resolver is ignoring TTL (rare) or the change didn't save at the authoritative NS. Verify at the authoritative server directly: `dig @<auth-ns> baseline.click`.

---

## 3. VPCs
**Concept:** A Virtual Private Cloud is an isolated network boundary. Resources inside share an address space and a controlled boundary to the internet. Foundation for all cloud networking.

**Pattern (when we move off single-VPS to cloud):**
```
VPC: 10.0.0.0/16
  ├─ public-subnet  10.0.1.0/24   (ALB, NAT GW, bastion)  -> has IGW route
  ├─ private-app    10.0.2.0/24   (baseLINE API, agents)   -> NO direct IGW
  └─ private-data   10.0.3.0/24   (Postgres, Redis)        -> NO IGW, app-only
```
Only the public subnet has an Internet Gateway route. Data tier has no public route at all — app reaches it within the VPC. Peer VPCs via VPC peering / transit gateway if multi-region.

**Validate:** `nmap` from the internet to the data subnet's PostgreSQL port → filtered/closed (not reachable). The app subnet reaches the DB over the private route; the DB cannot initiate outbound internet. A resource in private-app cannot be pinged from outside the VPC.

**Fix:** If the DB has a public IP, remove it — move to private subnet + security-group allow from app only. If two services in different VPCs can't talk, add a peering connection + route table entry. Never widen the VPC CIDR after creation without re-IP planning.

---

## 4. Subnets
**Concept:** Subnets partition a VPC's CIDR. Public vs private determined by route-table association (IGW route = public). Correct sizing avoids exhaustion and isolates tiers.

**Pattern:**
```
/16 VPC = 65536 addresses
  public   10.0.1.0/24  (254 usable)  -> ALB, bastion
  app      10.0.2.0/24  -> API, workers
  data     10.0.3.0/24  -> DB, cache
  reserved 10.0.4.0/24..10.0.255.0/24 for growth
```
Keep AZ-distinct subnets for HA (each tier spans ≥2 AZs). Use /28 minimum for tiny services; /24 default. Reserve room for autoscaling (topic 161).

**Validate:** Each tier's instances sit in the intended subnet (check ENI/subnet ID). Public-subnet instances have a public IP + IGW route; private do not. No private instance accidentally in a public subnet (route-table check).

**Fix:** If a "private" instance got a public IP, it's in a public subnet (IGW route) — move it. If a subnet is full, don't squeeze /30s; provision a new subnet in the reserved range. Span tiers across ≥2 AZs to avoid AZ failure = tier failure.

---

## 5. NAT Gateways
**Concept:** Private-subnet resources need outbound internet (apt update, API calls to Bankr/on-chain) but must not be inbound-reachable. NAT Gateway gives them egress without a public IP.

**Pattern:**
```
private-app 10.0.2.0/24  -> route 0.0.0.0/0 -> NAT GW (in public subnet) -> IGW
```
The NAT GW has a public IP; private instances share it for egress. Inbound connections cannot be initiated to private instances. For cost, one NAT per AZ (or shared if budget-bound, accepting AZ-NAT coupling).

**Validate:** From a private instance: `curl https://api.bankr.bot` works (egress OK). From the internet: `curl <private-instance-internal-ip>` fails (no inbound). The NAT GW's ENI is in the public subnet; the private instance has no public IP.

**Fix:** If private instances can't reach the internet, check the route table points 0.0.0.0/0 at the NAT GW (not IGW) and the NAT GW is in a public subnet with an EIP. If a private instance has a public IP, remove it. Monitor NAT GW throughput (it's a bottleneck / has bandwidth caps).

---

## 6. Firewalls / Security Groups / Service Mesh
**Concept:** Network-level access control. Firewalls (host/network) and Security Groups (stateful, per-instance cloud ACLs) enforce "who can talk to what on which port". A service mesh adds mTLS + discovery on top for east-west traffic.

**Pattern (defense in depth):**
```bash
# host firewall (ufw on the VPS today)
ufw default deny incoming
ufw allow 22/tcp      # SSH from bastion only (or restrict source)
ufw allow 80,443/tcp  # nginx
ufw allow from 10.0.2.0/24 to any port 5432  # app -> DB only
# cloud security group (future):
#   sg-app:  allow 443 from 0.0.0.0/0; allow 5432 from sg-app ONLY
#   sg-db:   allow 5432 from sg-app ONLY; deny 0.0.0.0/0
```
Rules: **default deny**, explicit allow, least-source (specific CIDR not 0.0.0.0/0 for DB), stateful (replies auto-allowed). Service mesh (Istio/Linkerd) for mTLS between services + traffic policy.

**Validate:** `nmap -p 5432 <db-host>` from the internet → closed. From the app subnet → open. SSH not exposed to the world (or behind bastion/VPN). A new service can't reach the DB until its SG is added to sg-db's allowlist.

**Fix:** If the DB port is world-open, lock it to the app SG/CIDR immediately. Remove `0.0.0.0/0` from any non-80/443 rule. For east-west mTLS, deploy a mesh (or at minimum wireguard/peer TLS) so intra-fleet calls are authenticated. Audit SG rules quarterly (orphaned rules = drift).

---

## Validation Checklist
- [ ] All intended subdomains resolve; SPF/Privy TXT present (1)
- [ ] TTL lowered pre-cutover; old IP kept during propagation (2)
- [ ] VPC isolates data tier (no public route) (3)
- [ ] Subnets correctly public/private via route tables; AZ-spread (4)
- [ ] Private egress via NAT, no private public-IPs (5)
- [ ] Default-deny firewall; DB locked to app SG; mTLS east-west (6)

## Fix Recipes
- **"Subdomain down"** → topics 1–2: check zone + NS + propagation, lower TTL.
- **"DB reachable from internet"** → topic 6 + 3: move to private subnet, lock SG.
- **"Private service can't call API"** → topic 5: add NAT GW + route.
- **"AZ failure killed the tier"** → topic 4: spread subnets across AZs.
- **"Lateral movement possible"** → topic 6: mesh/mTLS + tighten SGs.

## Cross-References
- `pcmedicalist-tls-termination` — encrypt the ingress the firewall allows
- `pcmedicalist-cdn-caching` — edge terminates DNS/TLS for static
- `pcmedicalist-security-fundamentals` — security groups = network authz; CSP pairs with egress rules
- `pcmedicalist-infrastructure-ops` — pod disruption budgets (194), health checks (141) in mesh
- `pcmedicalist-multi-region` — multi-VPC peering
- `pcmedicalist-ingress-controllers` — K8s ingress = L7 firewall
- `pcmedicalist-dns` (if present) — deeper DNS tooling
