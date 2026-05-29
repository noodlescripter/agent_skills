# AGENT.md — CCP Dev Infrastructure (App Gateway / DNS Resolver / Key Vault)

Context and operational notes for the Client Connect Portal (CCP) dev environment on Azure.
Terraform-managed via HCP Terraform (TFE). Files live under `tf/dev/` with named prefixes.

---

## Topology

```
Internet/Internal client
        │
        ▼
App Gateway (ext vnet)            vnet_clientconnectportal_azure_eus2_ext
  private frontend IP: 10.7.223.36
        │  VNet peering (Connected, forwarded traffic allowed)
        ▼
Container Apps env (dev vnet)     vnet_clientconnectportal_azure_eus2_dev
  internal load balancer IP: 10.34.47.47
  app: ccp-ui-dev
```

- ACA environment is **internal** (`internal_load_balancer_enabled = true`).
- VNet peering is **Connected**; "Allow forwarded traffic" is enabled. Gateway transit / remote gateway are NOT enabled (not needed).
- dev vnet ACA subnet has **no NSG** — only a route table (`palo-route`) with `0.0.0.0/0` → Palo Alto NVA.

---

## Application Gateway

Resource: `azurerm_application_gateway.application_gateway_dev` (`application_gateway_dev01`)
File: `tf/dev/network_appgateway_private_ui.tf`

- SKU: `Standard_v2`, capacity 1.
- Two frontends: public IP frontend and a **private** frontend (`10.7.223.36`, Static).
- Private listener (`appgw-http-listener-private`) on `http-private-port` (80) is the active path.
- Public listener / public routing rule are commented out — private only is in use.

### Backend pool
```hcl
backend_address_pool {
  name  = "appgw-backend-pool"
  fqdns = ["ccp-ui-dev.calmwater-77d9cb43.eastus2.azurecontainerapps.io"]
}
```

> **FQDN — no `.internal.` subdomain.** The private DNS zone is
> `calmwater-77d9cb43.eastus2.azurecontainerapps.io` with a wildcard `*` A record
> → `10.34.47.47`. The ACA Application URL for this environment does NOT contain
> `.internal.`, so the backend FQDN must match the wildcard zone exactly. Adding
> `.internal.` produces `Local Error: DNSResolution` on the App Gateway hop in
> Connection Troubleshoot.

### Backend HTTP settings
```hcl
backend_http_settings {
  name                  = "appgw-backend-http-settings"
  cookie_based_affinity = "Disabled"
  port                  = 443
  protocol              = "Https"
  request_timeout       = 30
  probe_name            = "appgw-ui-probe"
  # Prefer pick_host_name_from_backend_address = true so the Host header
  # always matches the backend FQDN instead of a separate local var.
}
```

### Probe
```hcl
probe {
  name                = "appgw-ui-probe"
  protocol            = "Https"
  path                = "/"
  interval            = 31
  timeout             = 31
  unhealthy_threshold = 3
  match { status_code = ["200-399"] }
  # host must match the backend FQDN (the wildcard zone name, no .internal.)
}
```

> Probe `host` and backend `host_name` previously pointed to a `local.react_ui_fqdn`
> that resolved incorrectly. That local was removed and replaced with the actual
> wildcard FQDN. Keep host header consistent across probe + backend settings.

### SSL profile
- `ssl_profile` `ccp-appgw-ssl-profile`, policy `CustomV2`, min TLS `TLSv1_2`.
- Cipher suites restricted to ECDHE AES 128/256 GCM (RSA + ECDSA).

---

## DNS resolution (the core gotcha)

The VNets do **not** use Azure-provided DNS. Custom DNS is configured on the VNet
(two internal DNS server IPs). Custom DNS servers do not natively know Azure Private
DNS zones, so `*.calmwater-77d9cb43.eastus2.azurecontainerapps.io` returns NXDOMAIN
unless explicitly forwarded.

### Confirmed working
- Private DNS zone `calmwater-77d9cb43.eastus2.azurecontainerapps.io` exists.
- Zone is **linked to both** vnets:
  - `aca-env-dns-zone-appgw-link` → ext vnet
  - `aca-env-dns-zone-dev-link` → dev vnet
- Record sets: `*  A  10.34.47.47` (wildcard) and `@  SOA`. The SOA `@` has no A record — that is normal.

### The fix
Add a **conditional forwarder** on the custom DNS servers (both IPs) for the zone:

```
Zone:        calmwater-77d9cb43.eastus2.azurecontainerapps.io   (NO .internal.)
Forward to:  <target depends on where custom DNS lives>
```

Forwarder target rule:
- Custom DNS is an **Azure VM** → forward to `168.63.129.16` (Azure DNS). No Private DNS Resolver required.
- Custom DNS is **on-prem / outside Azure** → forward to the **Private DNS Resolver inbound endpoint IP**, since `168.63.129.16` is only reachable inside an Azure VNet.

> Open item: confirm with the network team whether the two custom DNS IPs are
> Azure VMs or on-prem. That decides whether the Private DNS Resolver is needed at all.

### Private DNS Resolver (only if custom DNS is on-prem)
File: `tf/dev/network_dns_resolver.tf`

- Deployed in dev vnet; inbound endpoint needs a delegated subnet (min `/28`,
  delegation `Microsoft.Network/dnsResolvers`).
- The inbound endpoint private IP is the forwarder target for the custom DNS.
- Verify after the forwarder is added:
  ```bash
  nslookup ccp-ui-dev.calmwater-77d9cb43.eastus2.azurecontainerapps.io <inbound-endpoint-ip>
  # expect 10.34.47.47
  ```

---

## Key Vault

File: `tf/dev/_keyvault.tf`, policy in `keyvault_policy.tf`

- App Gateway SSL cert is sourced from Key Vault (PFX → `azurerm_key_vault_certificate`).
- CA certs (no private key) are stored as **secrets**, not certificates — a CA cert
  without a private key cannot be imported as a Key Vault certificate.
- If Terraform (running externally via TFE) hits 403 on Key Vault: public network
  access is likely disabled. Either whitelist the runner IP via `network_acls.ip_rules`
  or run from inside the VNet via private endpoint. Confirm the deploying identity has
  an access policy / RBAC role.
- Use `lifecycle { ignore_changes = [tags] }` on KV resources — tags are enforced by Azure Policy.

---

## Diagnosis playbook (App Gateway can't reach ACA)

Work top-down; each was checked for this environment:

1. **Peering** — Connected + forwarded traffic on. ✅
2. **Private DNS zone links** — linked to both vnets. ✅
3. **Wildcard A record** — `* → 10.34.47.47` present. ✅
4. **Backend FQDN** — must equal the wildcard zone name, **no `.internal.`**. ✅ (was the first bug)
5. **Custom DNS forwarder** — required because VNet uses custom DNS, not Azure DNS. ← network team action
6. **Connection Troubleshoot** — `application_gateway_dev01` hop showing
   `Local Error: DNSResolution` = forwarder/FQDN issue. Destination `Healthy` =
   the IP `10.34.47.47` is reachable, so peering/routing are fine.
7. **Container port** — once DNS resolves, a crashing container with the wrong
   `ingress.target_port` looks like a backend failure. App listens on **3000**, so
   `ingress { target_port = 3000 }` (was previously 8080 → crash on port mismatch). ✅ (was the second bug)

### Two root causes found
- App Gateway backend FQDN had `.internal.` → DNS resolution error.
- Container App `target_port` mismatch (8080 vs actual 3000) → container crash, looked like backend down.

---

## Windows service / port helpers (runner / WildFly hosts)

```powershell
# Find PID holding a port
netstat -ano | findstr :<port>

# Kill by PID
taskkill /PID <pid> /F

# Map PID → process name
tasklist /FI "PID eq <pid>"

# Map PID → Windows service name
Get-WmiObject Win32_Service | Where-Object { $_.ProcessId -eq <pid> }
```

---

## Notes
- Backend `Destination: Healthy` in Connection Troubleshoot only means the IP is
  reachable — it does not mean the app responds. Check container logs for crashes.
- After changing VNet DNS settings, the App Gateway may need to re-pick DNS; re-run
  Backend Health / Connection Troubleshoot to confirm.
