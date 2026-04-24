---
name: terraform-azure
description: Use this skill for writing, reviewing, debugging, and optimizing Terraform scripts targeting Azure infrastructure. Trigger whenever the user mentions Terraform, HCL, azurerm provider, Azure resource deployment, infrastructure as code, tf files, terraform plan/apply errors, HCP Terraform, Terraform Enterprise, or asks to create/modify Azure resources like Container Apps, App Gateway, APIM, Key Vault, VNets, NSGs, Private DNS, Service Bus, PostgreSQL, or any azurerm_* resource. Also trigger for Terraform state issues, module development, variable management, backend configuration, CI/CD pipeline integration with Terraform, and converting Azure portal configurations to Terraform code.
---

# Terraform Azure Infrastructure Skill

## Purpose
Write, review, debug, and optimize Terraform configurations for Azure infrastructure following enterprise best practices. This skill captures real-world solutions to common Azure networking, security, and deployment challenges.

---

## App Gateway to Container Apps Connectivity

### Problem
App Gateway returns 502 Bad Gateway when trying to reach Container Apps.

### Root Causes and Fixes

**1. DNS Resolution Failure**
When a VNet has custom DNS servers, they won't resolve Azure Private DNS zones unless configured to forward to `168.63.129.16`. App Gateway can't resolve the Container App FQDN.

Fix options:
- Add conditional forwarder on custom DNS servers for `*.azurecontainerapps.io` → `168.63.129.16`
- Deploy a DNS Private Resolver (required when environment has no internet access)

```hcl
resource "azurerm_subnet" "dns_resolver_inbound" {
  name                 = "snet-dns-resolver-inbound"
  resource_group_name  = var.networking_rg
  virtual_network_name = var.vnet_name
  address_prefixes     = [var.dns_resolver_subnet_cidr]

  delegation {
    name = "dns-resolver-delegation"
    service_delegation {
      name    = "Microsoft.Network/dnsResolvers"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}

resource "azurerm_private_dns_resolver" "resolver" {
  name                = "dnspr-${var.environment}"
  resource_group_name = var.resource_group_name
  location            = var.location
  virtual_network_id  = data.azurerm_virtual_network.vnet.id
}

resource "azurerm_private_dns_resolver_inbound_endpoint" "inbound" {
  name                    = "dnspr-inbound"
  private_dns_resolver_id = azurerm_private_dns_resolver.resolver.id
  location                = var.location

  ip_configurations {
    private_ip_allocation_method = "Dynamic"
    subnet_id                    = azurerm_subnet.dns_resolver_inbound.id
  }
}
```

Important notes:
- The resolver needs a dedicated `/28` subnet with delegation
- After adding resolver IP to VNet DNS, the App Gateway must be reconfigured to pick up new DNS
- If VNet has multiple DNS servers, put the resolver IP first — DNS queries try servers in order, timeouts on earlier servers cause probe failures
- The private DNS zone must be linked to the VNet where the App Gateway lives, not just the Container Apps VNet

**2. Route Table Intercepting Traffic**
If a route table has a broad route like `10.0.0.0/8 → Virtual Appliance (firewall)`, it intercepts traffic to Container Apps internal IPs.

Container Apps environments get internal IPs from Azure-managed ranges (e.g., `10.34.x.x`) that may differ from your VNet address space. These IPs still fall under broad `/8` routes.

Fix: Add a more specific route for the Container Apps environment IP:

```hcl
resource "azurerm_route" "container_apps_direct" {
  name                = "ContainerApps-Direct"
  resource_group_name = var.networking_rg
  route_table_name    = var.route_table_name
  address_prefix      = var.container_apps_ip_range
  next_hop_type       = "VnetLocal"
}
```

`VnetLocal` works for both same-VNet and peered-VNet traffic.

**3. NSG Missing Required Ports**
App Gateway v2 requires inbound ports 65200-65535 from GatewayManager:

```hcl
resource "azurerm_network_security_rule" "appgw_management" {
  name                       = "AllowGatewayManager"
  priority                   = 100
  direction                  = "Inbound"
  access                     = "Allow"
  protocol                   = "Tcp"
  source_port_range          = "*"
  destination_port_range     = "65200-65535"
  source_address_prefix      = "GatewayManager"
  destination_address_prefix = "*"
  resource_group_name        = var.resource_group_name
  network_security_group_name = var.appgw_nsg_name
}
```

**4. Backend Health Shows "Unknown"**
This means Azure can't even check backend health. Causes:
- NSG blocking management ports (65200-65535)
- Route table forcing management traffic through a firewall
- FQDN in backend pool can't be resolved

Check backend health in portal: Application Gateway → Backend health.

### App Gateway Configuration

```hcl
resource "azurerm_application_gateway" "appgw" {
  name                = var.appgw_name
  resource_group_name = var.resource_group_name
  location            = var.location

  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
    capacity = 1
  }

  backend_address_pool {
    name  = "backend-pool"
    fqdns = [local.container_app_fqdn]
  }

  backend_http_settings {
    name                                = "https-settings"
    cookie_based_affinity               = "Disabled"
    port                                = 443
    protocol                            = "Https"
    request_timeout                     = 30
    pick_host_name_from_backend_address = true
    probe_name                          = "health-probe"
  }

  probe {
    name                                      = "health-probe"
    protocol                                  = "Https"
    path                                      = "/"
    interval                                  = 30
    timeout                                   = 30
    unhealthy_threshold                       = 3
    pick_host_name_from_backend_http_settings = true

    match {
      status_code = ["200-399"]
    }
  }
}
```

Key points:
- Use FQDN in backend pool, not IP addresses
- `pick_host_name_from_backend_address = true` is critical — Container Apps rejects requests with wrong Host header
- Minimum recommended subnet size is `/24`
- Each App Gateway needs its own public IP
- Use multi-site listeners or path-based routing for multiple services

---

## Key Vault Access and Certificates

### 403 Forbidden When Uploading Certificates
When Key Vault has `public_network_access_enabled = false` and Terraform runs from outside the VNet:

```hcl
resource "azurerm_key_vault" "vault" {
  public_network_access_enabled = true

  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
    ip_rules       = [var.cicd_runner_ip]
  }
}
```

### CA Certificates Without Private Keys
Key Vault certificates require a private key. CA certs used as trust anchors don't have private keys. Upload as secrets:

```hcl
resource "azurerm_key_vault_secret" "ca_cert" {
  name         = "ca-cert-name"
  value        = file("${path.module}/certs/ca-cert.pem")
  key_vault_id = azurerm_key_vault.vault.id
  content_type = "application/x-pem-file"
}
```

Application code using `getCertificateClient().getCertificate()` won't find secrets — code must use `SecretClient` for CA certs.

### Extracting CA Cert from PFX
```bash
openssl pkcs12 -in cert.pfx -legacy -nokeys -cacerts -passin pass:password -out ca-cert.pem
```

### Stripping PEM Headers
```hcl
locals {
  raw_pem    = file("${path.module}/certs/cert.pem")
  cert_clean = replace(replace(replace(replace(local.raw_pem,
    "-----BEGIN CERTIFICATE-----", ""),
    "-----END CERTIFICATE-----", ""),
    "\n", ""), "\r", "")
}
```

---

## mTLS Certificate Chain Validation

### "Path does not chain with any of the trust anchors"

The trust anchor must be the **CA certificate that signed** the client cert, not another end-entity cert signed by the same CA.

```
CA Certificate (MUST be trust anchor)
    ├── Service A cert (end-entity — cannot be trust anchor)
    └── Service B cert (end-entity — cannot be trust anchor)
```

Even if two certs share the same issuer, one cannot validate the other. Only the CA can.

### Verify Cert Relationships
```bash
# Check if cert is a CA
keytool -list -v -keystore cert.pfx -storetype PKCS12 -storepass password
# Look for: BasicConstraints: [CA:true]

# Check issuer/subject match
openssl x509 -in cert.pem -noout -issuer -subject
# Trust anchor subject must match client cert issuer
```

### Key Vault getCer() Returns Leaf Cert Only
PFX with a cert chain — `getCer()` returns only the leaf, not CA certs. Upload CA cert separately where it is the primary cert.

### Container Apps Client Certificate Mode
```hcl
ingress {
  target_port             = 8080
  transport               = "auto"
  client_certificate_mode = "accept"  # or "require"
}
```

- `ignore` — won't forward client certs (breaks mTLS)
- `accept` — forwards cert if present
- `require` — rejects requests without client cert

---

## Container Apps Configuration

### Prevent Cold Starts
```hcl
template {
  min_replicas = 1
  max_replicas = 5
}
```

### User-Assigned Managed Identity
```hcl
identity {
  type         = "UserAssigned"
  identity_ids = [azurerm_user_assigned_identity.uai.id]
}

template {
  container {
    env {
      name  = "AZURE_CLIENT_ID"
      value = azurerm_user_assigned_identity.uai.client_id
    }
  }
}
```

Without `AZURE_CLIENT_ID`, `DefaultAzureCredential` throws `CredentialUnavailableException`.

### Environment Variables from Maps
```hcl
locals {
  app_env = {
    SETTING_ONE = "value1"
    SETTING_TWO = "value2"
  }
  app_secrets = {
    DB_PASSWORD = "secret-name"
  }
}

dynamic "env" {
  for_each = local.app_env
  content {
    name  = env.key
    value = env.value
  }
}

dynamic "env" {
  for_each = local.app_secrets
  content {
    name        = env.key
    secret_name = env.value
  }
}
```

### Docker Base Image
- Alpine (`eclipse-temurin:21-jre-alpine`) — smaller but may need `gcompat`/`libc6-compat`, permission issues common
- Debian (`eclipse-temurin:21-jre-jammy`) — larger but works out of the box
- `UnsatisfiedLinkError` for native libraries (Netty) → switch from Alpine to Debian

---

## APIM Configuration

### Custom Domain
```hcl
resource "azurerm_api_management_custom_domain" "domain" {
  api_management_id = azurerm_api_management.apim.id

  gateway {
    host_name                = "api.yourdomain.com"
    key_vault_certificate_id = azurerm_key_vault_certificate.cert.versionless_secret_id
  }
}
```

`key_vault_id` is deprecated — use `key_vault_certificate_id` with `versionless_secret_id`.

### Unique API Paths
Each API needs unique `path`. "One or more fields contain incorrect values" usually means path collision:

```hcl
resource "azurerm_api_management_api" "service_a" {
  path = "api/service-a"
}
resource "azurerm_api_management_api" "service_b" {
  path = "api/service-b"
}
```

### Path Rewriting
```xml
<rewrite-uri template="/api/service/{path}" copy-unmatched-params="true" />
```

### Wildcard Operations
```hcl
resource "azurerm_api_management_api_operation" "wildcard" {
  for_each     = toset(["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"])
  operation_id = "wildcard-${lower(each.key)}"
  method       = each.key
  url_template = "/{path}"

  template_parameter {
    name     = "path"
    required = false
    type     = "string"
  }
}
```

---

## Private DNS and VNet Peering

### Cross-VNet DNS Resolution
When App Gateway and Container Apps are in different peered VNets:
1. Private DNS zone must be linked to **both** VNets
2. Custom DNS servers must forward Azure private zone queries
3. Route tables must allow cross-VNet traffic

```hcl
resource "azurerm_private_dns_zone_virtual_network_link" "link" {
  name                  = "link-to-vnet"
  resource_group_name   = var.dns_zone_rg
  private_dns_zone_name = var.dns_zone_name
  virtual_network_id    = azurerm_virtual_network.vnet.id
  registration_enabled  = false
}
```

### Adding VNet Address Space
```hcl
address_space = [
  "10.7.223.32/28",  # existing
  "10.7.224.0/28"    # new range
]
```

---

## CI/CD with HCP Terraform

### GitHub Actions — Update Variable and Trigger Run
```yaml
- name: Deploy
  env:
    TFE_TOKEN: ${{ secrets.TFE_TOKEN }}
  run: |
    WS_ID=$(curl -s \
      -H "Authorization: Bearer ${TFE_TOKEN}" \
      "https://${TFE_HOST}/api/v2/organizations/${ORG}/workspaces/${WS}" \
      | jq -r '.data.id')

    VAR_ID=$(curl -s \
      -H "Authorization: Bearer ${TFE_TOKEN}" \
      "https://${TFE_HOST}/api/v2/workspaces/${WS_ID}/vars" \
      | jq -r '.data[] | select(.attributes.key == "image_tag") | .id')

    curl -s -X PATCH \
      -H "Authorization: Bearer ${TFE_TOKEN}" \
      -H "Content-Type: application/vnd.api+json" \
      -d "{\"data\":{\"attributes\":{\"value\":\"${{ github.sha }}\"}}}" \
      "https://${TFE_HOST}/api/v2/workspaces/${WS_ID}/vars/${VAR_ID}"

    curl -s -X POST \
      -H "Authorization: Bearer ${TFE_TOKEN}" \
      -H "Content-Type: application/vnd.api+json" \
      -d "{\"data\":{\"attributes\":{\"message\":\"Deploy\",\"auto-apply\":true},\"relationships\":{\"workspace\":{\"data\":{\"type\":\"workspaces\",\"id\":\"${WS_ID}\"}}}}}" \
      "https://${TFE_HOST}/api/v2/runs"
```

---

## Debugging Checklist

### App Gateway 502
1. Portal → App Gateway → Backend health → read exact error
2. "Unknown" → NSG blocking 65200-65535 or FQDN resolution failure
3. "Unhealthy" → probe failing (wrong path, timeout, status code)
4. Check route table for NVA routes intercepting backend traffic
5. Check private DNS zone links
6. Check custom DNS servers resolve private DNS zones

### Container App Not Starting
1. Check logs: portal → Container App → Log stream
2. `UnsatisfiedLinkError` → switch Alpine to Debian base image
3. `CredentialUnavailableException` → add `AZURE_CLIENT_ID` env var
4. Key Vault 403 → verify managed identity access policy

### mTLS Failures
1. Enable debug: `LOGGING_LEVEL_COM_<PACKAGE>=TRACE`
2. Check certs loaded from Key Vault (cert name and size in logs)
3. Verify trust anchor is CA cert (`CA:true`) not end-entity
4. Verify client cert issuer matches trust anchor subject
5. Check `client_certificate_mode` is not `ignore`

### PostgreSQL Private Endpoint
- `nslookup` returns public IP → private DNS zone not linked or custom DNS not forwarding
- Can't ping → normal, Azure PaaS ignores ICMP, test with `telnet <ip> 5432`
- Bypass DNS for testing: connect directly to private endpoint IP