# Cisco Secure Workload → NetScaler ADC Integration Guide

A step-by-step, **beginner-friendly** guide to integrating **NetScaler ADC** (formerly **Citrix NetScaler / Citrix ADC**) with **Cisco Secure Workload (CSW)** using the **External Orchestrator (`type: Citrix Netscaler`, `orch_type: nsbalancer`)**.

CSW connects to the NetScaler **NITRO REST API** to:
1. **Import Load Balancing virtual servers** as *service inventory* → labels (`service_name`, `orchestrator_*`).
2. Optionally **enforce policy** by translating CSW policies into **NetScaler ACL rules** and deploying them to the appliance.

> **⚠ Disclaimer:** This is a **community reference guide** prepared by Cisco Solutions Engineering — not an official Cisco product document. Always refer to the [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/support/security/tetration/series.html) and the [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) for authoritative, up-to-date guidance.

---

## 0. New to CSW or NetScaler? Read this first (2 minutes)

- **Cisco Secure Workload (CSW)** discovers how applications talk and builds **least-privilege micro-segmentation** using **labels → scopes → policies**.
- **NetScaler ADC** is a **load balancer / ADC**. Clients connect to a **Load Balancing (LB) virtual server** — a **VIP** (virtual IP + port) — which distributes traffic to a backend service pool. It typically rewrites the source IP with **SNAT**.

**Why integrate them?**

| Problem NetScaler introduces | What the integration gives you |
|---|---|
| The load balancer is a "black box" — CSW doesn't know which VIPs are which services | The **orchestrator** imports each LB virtual server as a labeled **service inventory** item (`service_name`, `orchestrator_*`) |
| You want the ADC to enforce your segmentation policy | **Policy enforcement** translates CSW policies into **NetScaler ACL rules** and pushes them via the NITRO REST API |

> **Important scope note:** Unlike the F5 integration, NetScaler **does not** provide service **flow visibility** to CSW. This integration is **label import + ACL enforcement only**. For flow telemetry, use agents, NetFlow, or ERSPAN elsewhere in the estate.

---

## 1. Architecture

![CSW and NetScaler ADC Integration Architecture](csw-netscaler-architecture.png)

*The External Orchestrator uses the NetScaler NITRO REST API (HTTPS) to import LB virtual servers as service-inventory labels and to push ACL rules back to the appliance's global ACL list. There is no flow-telemetry path.*

| Path | Direction | Transport | Purpose |
|---|---|---|---|
| **External Orchestrator** | CSW ⇄ NetScaler | HTTPS NITRO REST API | Import LB virtual servers → labels; push ACL rules |

---

## 2. What you get

**Service inventory & labels** — each NetScaler LB virtual server becomes a searchable inventory item:

| Key | Value |
|---|---|
| `orchestrator_system/orch_type` | `nsbalancer` |
| `orchestrator_system/workload_type` | `service` |
| `orchestrator_system/service_name` | *(virtual server name)* |
| `orchestrator_system/cluster_id` / `cluster_name` | *(orchestrator identity)* |
| `orchestrator_annotation/snat_address` | *(SNAT address of the virtual server)* |

Use these in **inventory search**, **scopes**, and **policies** — for example, segment everything served by a given VIP.

**Policy enforcement** (optional) — CSW translates logical policies whose **provider** matches a NetScaler LB virtual server into **NetScaler ACL rules** and deploys them via NITRO. CSW becomes the source of truth for those ACLs.

---

## 3. Prerequisites

### NetScaler ADC side
- NetScaler ADC with a reachable **NITRO REST API** endpoint (HTTPS/443). Documented baseline: **v12.0.57.19+** (use a current, supported release).
- A NetScaler account for CSW:
  - **Read-only** is enough for **label import only**.
  - **Read + write** is required for **policy enforcement** (CSW writes ACL rules).
- Know which **LB virtual servers** you want CSW to see (single-address VIPs only — see §8).

### Cisco Secure Workload side
- CSW **4.x** (on-prem or SaaS). Orchestrators live under **Manage → External Orchestrators**.
- A user with **Site Admin / Root Scope Owner** rights, or an API key with the `external_integration` capability for the API path (§7).
- For **SaaS / non-routable NetScaler**: a healthy **Secure Connector** tunnel (§4.1).

### Network / firewall
- **HTTPS (TCP/443)** from the CSW source (cluster / SaaS egress / Secure Connector VM) **→ NetScaler NSIP / management endpoint**.

### 3.1 Create a least-privilege NetScaler service account (recommended)

1. On NetScaler: **System → User Administration → Users → Add**.
2. Create a dedicated user (e.g. `svc-csw`) with a strong, unique password.
3. Bind a **command policy**:
   - **Label import only:** a **read-only** command policy (e.g. built-in `read-only`).
   - **Policy enforcement:** a policy allowing ACL configuration (e.g. `operator`/`sysadmin` or a scoped custom policy) — CSW must create/replace ACLs.

> **Security:** Store this credential in your secrets manager. Never commit it to source control. CSW stores it encrypted; you must **re-enter the password every time you edit** the orchestrator.

---

## 4. Configure the NetScaler External Orchestrator (UI)

**Path:** `Manage → External Orchestrators → Create New Configuration`

### Step 1 — Basic tab

Select **Type = Citrix Netscaler**, then fill the common fields:

| Field | Required | Value / guidance |
|---|---|---|
| **Name** | ✅ | Unique per tenant, e.g. `netscaler-dc1` |
| **Type** | ✅ | `Citrix Netscaler` (not editable after creation) |
| **Description** | ❌ | e.g. "DC1 NetScaler — service labels + ACL enforcement" |
| **Username** | ✅ | The NetScaler service account (e.g. `svc-csw`) |
| **Password** | ✅ | The service account password (re-enter on every edit) |
| **Full Snapshot Interval (s)** | ✅ | How often to re-poll NetScaler |
| **Delta Interval (s)** | ❌ | Incremental poll (default 60s). First full snapshot completes in ~60s |
| **Accept Self-signed Cert** | ❌ | Check only if the ADC presents a self-signed cert (lab). Prefer a CA-signed cert in production |
| **Secure Connector Tunnel** | ❌ | Check for **SaaS / non-routable** NetScaler (see §4.1) |

### Step 2 — Hosts List tab

Add the NetScaler NITRO endpoint:

| Column | Value |
|---|---|
| **Host Name / IP** | NetScaler NSIP / management IP or FQDN |
| **Port** | `443` |

- For an **HA pair**, enter **another member node** so the orchestrator **fails over** to the currently-active node.
- A **different NetScaler** requires a **separate orchestrator**.

### Step 3 — NetScaler-specific field

| Field | Default | What it does |
|---|---|---|
| **Enable Enforcement** | Off | If checked, CSW may deploy **ACL rules** to this NetScaler (**requires write credentials**). Start **off** — turn on later (§6) |

### Step 4 — (Optional) Alerts tab
- Check **Alert enabled**; set **Severity** and **Disconnect Duration (m)**.
- Also enable **Connector Alerts** under **Manage → Workloads → Alert Configs**.

### Step 5 — Create
Click **Create**. Allow ~1 minute for the connection status to update. The first full snapshot of virtual servers typically completes in ~60 seconds.

### 4.1 (SaaS only) Secure Connector tunnel

Skip if CSW is **on-prem** and can reach the NetScaler directly. For **SaaS/TaaS** or a non-routable ADC:

1. In CSW: **Manage → Secure Connector** → download/register the client on a Linux VM that **can** reach the NetScaler.
2. Size per the [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) (2 vCPU / 4 GB RAM; RHEL/Rocky/Alma).
3. Confirm the Secure Connector is **Active/Connected**, then check **Secure Connector Tunnel** on the orchestrator.

---

## 5. Verify label import

### 5.1 Connection status
Open the orchestrator row — the status should be healthy. If it isn't, read the error (bad credentials, unreachable host, or cert error).

### 5.2 Find your NetScaler services in inventory
**Path:** `Investigate → Inventory Search`

```
orchestrator_system/orch_type = nsbalancer
```

```
orchestrator_system/workload_type = service and orchestrator_system/service_name = my-lb-vip
```

Each LB virtual server appears as a **service inventory** item (VIP + protocol + port) with `service_name` and `orchestrator_*` labels.

> **Caveat:** Only a VIP given as a **single address** is imported — a VIP defined as an **address pattern** is **not** supported.

---

## 6. (Optional) Enable ACL policy enforcement to NetScaler

> Only do this once you understand the impact: **CSW becomes the owner** of the NetScaler's ACLs.

**How it works:**
- CSW translates logical policies whose **provider** matches a labeled NetScaler LB virtual server into **NetScaler ACL rules** and deploys them via the NITRO REST API.
- **All existing ACL rules are replaced** by CSW-generated ACL rules.
- ACLs are always deployed to the **global ACL list** (partition **default**).
- CSW **detects drift** — manual ACL changes are reverted. **Manage these ACLs in CSW only.**

**Steps:**
1. Edit the orchestrator → check **Enable Enforcement** (credentials must have **write** access). This alone deploys nothing yet.
2. In the **workspace** that contains at least one policy whose provider is the NetScaler service, run **Enforce**.
3. Verify on the NetScaler that the global ACL list now contains CSW-generated allow/drop ACLs. Use the OpenAPI *policy enforcement status for external orchestrator* to confirm success/failure.

> **Stopping enforcement:** Disabling enforcement on the orchestrator (or deleting it) **immediately removes all CSW ACLs** from the NetScaler, leaving the ACL list **empty**. Plan for this.

---

## 7. (Alternative) Configure the orchestrator via OpenAPI

For automation. The API key needs the `external_integration` capability.

`POST /openapi/v1/orchestrator/{rootScopeName}`

```python
import os, json

req_payload = {
    "name": "netscaler-dc1",
    "type": "netscaler",
    "description": "DC1 NetScaler - service labels + ACL enforcement",
    "hosts_list": [
        {"host_name": "netscaler-dc1.corp.example.com", "port_number": 443}
    ],
    "username": "svc-csw",
    # Pull the secret from env / a secrets manager - never hardcode credentials
    "password": os.environ["NETSCALER_CSW_PASSWORD"],
    "enable_enforcement": False,   # set True only when ready to push ACLs (needs write creds)
    "full_snapshot_interval": 3600,
    "delta_interval": 60,
    "insecure": False,             # True only to accept self-signed certs (lab)
    "use_secureconnector_tunnel": False   # True for SaaS / non-routable NetScaler
}

resp = restclient.post("/orchestrator/Default", json_body=json.dumps(req_payload))
print(resp.status_code, resp.text)
```

> **Security note (applied per repo policy):** the sample reads the password from an environment variable (`os.environ`) rather than embedding it. Never commit credentials or API keys; store them in a secrets manager and inject at runtime.

Useful endpoints: `GET/POST/PUT/DELETE /openapi/v1/orchestrator/{scope}[/{id}]`, and *policy enforcement status for external orchestrators* to confirm ACL deployment.

---

## 8. Caveats & troubleshooting

| Symptom / topic | Guidance |
|---|---|
| **Connectivity / auth failure** | Allow HTTPS from the CSW source (cluster / SaaS egress / Secure Connector VM) → NetScaler NSIP. Verify credentials; enforcement needs **write** access. |
| **Certificate error** | NetScaler presents a self-signed cert → install a CA-signed cert (preferred) or check **Accept Self-signed Cert** / set `insecure: True` (lab only). |
| **ACL rules not found after enforcement** | Ensure the LB virtual server is **enabled / up**; only then are ACLs applied. |
| **VIP not imported** | Only **single-address** VIPs are supported (not address-pattern VIPs). |
| **No service visibility** | Expected — NetScaler service **flow visibility is not supported** by this integration. Use agents / NetFlow / ERSPAN for flow data. |
| **ACLs applied to wrong partition** | Expected — CSW always deploys to the **global ACL list (partition default)**. |
| **Enforcement removed unexpectedly** | Disabling enforcement or deleting the orchestrator **clears all CSW ACLs** from the NetScaler — expected behavior. |

---

## 9. Video references

> **There is currently no dedicated Cisco Secure Workload + NetScaler video** in the Cisco/community catalog. The videos below cover the **same external-orchestrator + load-balancer enforcement pattern** (F5 BIG-IP) and general CSW concepts — useful analogs while configuring NetScaler.

| Video | Why it's relevant |
|---|---|
| [F5 BIG-IP IPFIX Configuration](https://www.youtube.com/watch?v=aJZEcZtUXDg) | Load-balancer + Secure Workload integration pattern (F5 analog) |
| [F5 BIG-IP & Cisco Tetration: APM Visibility](https://www.youtube.com/watch?v=dqbWhvFNsso&t=90s) | ADC + Secure Workload visibility concepts |
| [Cisco Tetration & F5 BIG-IP AFM](https://www.youtube.com/watch?v=HcF3yQHmeXc) | ADC firewall/ACL enforcement concepts |
| [CSW-User-Education video library](https://github.com/chandrapati/CSW-User-Education) | Full curated CSW learning path (scopes, labels, policy, enforcement) |

> *Tetration* is the former product name for Cisco Secure Workload — the concepts apply directly. If Cisco publishes a NetScaler-specific demo, this section will be updated.

---

## 10. References

- [CSW — External Orchestrators, incl. Citrix NetScaler (On-Prem 4.0)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-external-orchestrators-in-secure-workload.html)
- [CSW — External Orchestrators (SaaS 4.0)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-external-orchestrators.html)
- [CSW — OpenAPI (Orchestrators & enforcement status)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
- [CSW Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html)
- [NetScaler NITRO REST API documentation](https://developer-docs.netscaler.com/)
