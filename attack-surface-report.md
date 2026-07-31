# External Attack Surface Reconnaissance Report

## Target

**Domain:** dodopayments.tech

---

## Assessment Type

**Passive OSINT Reconnaissance**

---

# Executive Summary

This report documents the external attack surface reconnaissance performed against `dodopayments.tech`.

The objective was to identify publicly exposed assets from an external attacker perspective using only publicly available information.

The assessment focused on:

- Certificate Transparency logs
- Passive subdomain enumeration
- DNS intelligence
- Live host discovery
- Technology fingerprinting

No exploitation, vulnerability scanning, brute force, fuzzing, or intrusive testing was performed.

---

# 1. Reconnaissance Methodology

The following passive reconnaissance techniques were used:

| Tool | Purpose |
|---|---|
| crt.sh | Certificate Transparency based subdomain discovery |
| Subfinder | Passive subdomain enumeration |
| Assetfinder | Passive asset discovery |
| Amass | Passive DNS intelligence gathering |
| httpx | Live host discovery and technology fingerprinting |

---

# 2. Subdomain Discovery Results

Multiple independent passive intelligence sources were combined to build the external asset inventory.

## Discovery Summary

| Source | Method | Results |
|---|---|---:|
| crt.sh | Certificate Transparency Logs | 75 |
| Subfinder | Passive OSINT Sources | 123 |
| Assetfinder | Passive OSINT Sources | 53 |
| Amass | Passive DNS Enumeration | 224 |
| Combined Unique Assets | Merged Inventory | 348 |

---

# 3. Certificate Transparency Findings

Certificate Transparency logs revealed multiple publicly registered subdomains.

Examples:

```
app.dodopayments.tech
checkout.dodopayments.tech
customer.dodopayments.tech
docs.dodopayments.tech
api.lago.dodopayments.tech
api.marble.dodopayments.tech
grafana.dodopayments.tech
keycloak.dodopayments.tech
sonarqube.dodopayments.tech
wordpress.dodopayments.tech
```

---

## Security Observation

Certificate Transparency information can provide attackers with:

- Public service discovery
- Application naming patterns
- Development environment exposure
- API endpoint discovery
- Third-party service identification

This information helps attackers build an external attack map before attempting further actions.

---

# 4. DNS and Infrastructure Observations

Passive DNS analysis identified publicly reachable infrastructure.

Observed technologies and providers:

- Cloudflare CDN
- Vercel hosting infrastructure

Examples:

```
checkout.dodopayments.tech
        |
        ▼
       Vercel
```

```
website.dodopayments.tech
        |
        ▼
     Cloudflare
```

---

## DNS Records Observed

Collected record types:

- A records
- AAAA records
- CNAME records

Example:

```
104.18.x.x
2606:4700::xxxx
```

These IP ranges indicate Cloudflare edge infrastructure.

---

# 5. Live Host Discovery

Tool used:

```
httpx
```

Result:

```
348 discovered subdomains
```

Live HTTP/HTTPS services:

```
48 hosts
```

---

## Observation

Only a subset of discovered DNS assets responded over HTTP/HTTPS.

The remaining assets may represent:

- Inactive services
- Internal services
- Non-web endpoints
- Protected resources

---

# 6. Technology Fingerprinting

Technology detection was performed using:

- httpx fingerprints
- HTTP headers
- Public application responses

---

## Identified Technologies

| Technology | Observation |
|---|---|
| Cloudflare | CDN/WAF protection |
| Vercel | Application hosting |
| Next.js | Web framework |
| React | Frontend framework |
| Node.js | Backend runtime |
| Astro | Website framework |
| Webpack | Frontend tooling |
| HSTS | Security header |

---

# 7. Interesting Discovered Assets

## checkout.dodopayments.tech

Detected technologies:

```
Next.js
React
Node.js
Webpack
Vercel
HSTS
```

Observation:

Payment-related services are high-value assets because attackers commonly investigate:

- Payment workflows
- Authentication flows
- Business logic
- API communication

---

## website.dodopayments.tech

Detected technologies:

```
Astro
Cloudflare
Cloudflare Bot Management
```

Observation:

The use of CDN protection reduces direct origin exposure, but public technology information remains visible.

---

# 8. Additional Interesting Services

| Host | Observation |
|---|---|
| api.lago.dodopayments.tech | Public API-style endpoint |
| api.marble.dodopayments.tech | Backend service |
| grafana.dodopayments.tech | Monitoring platform |
| keycloak.dodopayments.tech | Authentication service |
| kafka.dodopayments.tech | Messaging infrastructure |
| sonarqube.dodopayments.tech | Development/security tooling |
| wordpress.dodopayments.tech | CMS platform |

---

# 9. Risk Analysis

## Public API Exposure

Examples:

```
api.*
partner-api.*
events.*
```

Potential risks:

- Authentication weaknesses
- Authorization issues
- Data exposure
- Rate limit weaknesses

Recommendations:

- Implement strong authentication
- Validate authorization
- Apply API rate limiting
- Monitor suspicious requests

---

# Development and Testing Environments

Examples:

```
dev.*
test.*
demo.*
```

Potential risks:

- Reduced security controls
- Debug information exposure
- Configuration leakage

Recommendations:

- Restrict public access
- Maintain production security standards
- Remove unnecessary public exposure

---

# Monitoring and Administrative Services

Examples:

```
grafana
keycloak
sonarqube
kafka
```

Potential risks:

- Information disclosure
- Administrative exposure
- Infrastructure mapping

Recommendations:

- Restrict access through private networks
- Enable strong authentication
- Apply least privilege access

---

# 10. Technology Fingerprinting Limitations

## WhatWeb

WhatWeb testing was attempted.

Issue:

```
cannot load such file -- /usr/bin/lib/messages
```

Reason:

Local installation/environment issue.

Alternative fingerprinting was completed using:

- httpx
- HTTP response headers
- Public information

---

# 11. TLS Assessment Limitation

TLS posture review was planned using:

```
testssl.sh
```

Attempted:

```
testssl.sh dodopayments.tech
```

Result:

```
Not completed
```

Reason:

- Tool unavailable in the environment
- Time limitation

---

# 12. Assessment Boundaries

This assessment was strictly passive.

## Completed

✅ Certificate Transparency analysis  
✅ DNS discovery  
✅ Subdomain enumeration  
✅ Live host detection  
✅ Technology identification  

---

## Not Performed

❌ Exploitation  
❌ Vulnerability scanning  
❌ Credential attacks  
❌ Fuzzing  
❌ Brute force  
❌ Active attacks  

---

# 13. Part B — Penetration Testing Status

## Status

```
Not Completed
```

Reason:

- No designated authorized target was provided
- Time constraints

No penetration testing activity was performed against:

```
dodopayments.tech
```

---

# 14. Conclusion

The passive reconnaissance phase successfully mapped the external attack surface of:

```
dodopayments.tech
```

The assessment identified:

- 348 unique subdomains
- 48 live HTTP/HTTPS services
- Public application technologies
- Cloud infrastructure providers
- Potential attacker-interest areas

The findings demonstrate how publicly available information can be used by an external attacker to understand an organization's technology landscape.

Further penetration testing should only be conducted after explicit authorization and target confirmation.
