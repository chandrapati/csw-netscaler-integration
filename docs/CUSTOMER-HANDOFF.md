# NetScaler ADC integration — customer handoff checklist

**Purpose:** Give this page to your customer's **NetScaler / load-balancer / network** team when scoping a Cisco Secure Workload (CSW) integration with NetScaler ADC (formerly Citrix NetScaler / Citrix ADC).

**Integration summary:** CSW connects to the NetScaler **NITRO REST API (HTTPS)** to import **Load Balancing virtual servers** as labeled service inventory and — optionally — to **deploy ACL rules** to the appliance. **Service flow visibility is not part of this integration.**

---

## Integration paths — pick what applies

| Path | What CSW does | Credential needed | Data direction |
|---|---|---|---|
| **A. Labels only** | Import LB virtual servers → `service_name` / `orchestrator_*` labels | **Read-only** NITRO | CSW → NetScaler (poll) |
| **B. Labels + enforcement** | Path A **plus** deploy **ACL rules** to the global ACL list | **Read + write** NITRO | CSW ⇄ NetScaler |

> Enforcement (B) means **CSW becomes the owner** of the NetScaler ACLs — it **replaces all existing ACLs** on the global list and reverts manual changes. Disabling it removes those ACLs.

---

## What we need from the NetScaler team

### 1. Service account

| Item | Detail |
|------|--------|
| **Account name** | e.g. `svc-csw` (dedicated service account) |
| **NetScaler version** | v12.0.57.19+ (current supported) with NITRO REST enabled |
| **Command policy (labels only)** | Read-only (built-in `read-only`) |
| **Command policy (enforcement)** | ACL configuration rights (e.g. `operator`/`sysadmin` or scoped custom policy) |

### 2. Connectivity

| Direction | Source | Destination | Port |
|-----------|--------|-------------|------|
| Outbound | CSW cluster / SaaS Secure Connector VM | NetScaler NSIP / mgmt endpoint | **443/TCP** (HTTPS) |
| Outbound (SaaS) | Secure Connector VM | CSW SaaS FQDN | **443/TCP** |

**No inbound rules** to the Secure Connector VM are required.

### 3. Endpoint(s)

Provide the **NSIP / management IP or FQDN** for the CSW **Hosts List**:

```
Example:
  netscaler-dc1.corp.example.com:443
  netscaler-dc1-node2.corp.example.com:443   (another HA member for failover)
```

- One external orchestrator in CSW = **one NetScaler** (or HA pair).
- A second appliance requires a **second** orchestrator.

### 4. Scoping decisions

- [ ] Which **LB virtual servers** matter for segmentation (single-address VIPs only — no address patterns)
- [ ] Enable **ACL enforcement**? (needs write creds; CSW owns/replaces the global ACLs)
- [ ] Confirm the team is OK with CSW managing the **global ACL list (partition default)**

---

## What CSW will produce

| NetScaler source | CSW label | Example |
|------------------|-----------|---------|
| LB virtual server | `orchestrator_system/service_name` | `web-lb-vip` |
| Orchestrator type | `orchestrator_system/orch_type` | `nsbalancer` |
| Workload type | `orchestrator_system/workload_type` | `service` |
| SNAT address | `orchestrator_annotation/snat_address` | *(address)* |

Use these labels in **inventory search**, **scopes**, and **policy**.

---

## Security & compliance talking points

- **Least privilege** — read-only account unless enforcement is required.
- **Enforcement is reversible** — disabling it removes CSW ACLs from the NetScaler.
- **Drift control** — with enforcement on, manage ACLs **in CSW only**; CSW reverts manual changes.
- **Global ACL scope** — CSW deploys to the global ACL list (partition default); confirm this is acceptable.
- **Dedicated service account** — auditable, rotatable credentials; stored encrypted in CSW.

---

## Validation (joint test)

| # | Test | Owner | Pass |
|---|------|-------|------|
| 1 | NITRO login with service account from CSW / Secure Connector | NetScaler + CSW team | ☐ |
| 2 | Orchestrator **Connection Status: Success** | CSW admin | ☐ |
| 3 | LB virtual server visible in inventory (`orchestrator_system/orch_type = nsbalancer`) | CSW admin | ☐ |
| 4 | (If enforcement) ACLs deployed to global list; enforcement status = success | Security architect | ☐ |

---

## References

- Full guide: [CSW-NetScaler-Integration-Guide.md](../CSW-NetScaler-Integration-Guide.md)
- Cisco official: [External Orchestrators — Citrix NetScaler](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-external-orchestrators-in-secure-workload.html)
- Compatibility Matrix: [Secure Workload](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html)

---

*Generic template — no customer-specific names. Customize contacts and endpoints locally before sending.*
