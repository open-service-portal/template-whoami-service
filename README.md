# WhoAmI Service Template

A composition-of-compositions template that creates a complete publicly accessible service by orchestrating:
- **WhoAmI Application** (Deployment, Service, Ingress)
- **Cloudflare DNS Record** (A record pointing to the Ingress)

## Overview

This template demonstrates the composition-of-compositions pattern in Crossplane v2, where one XR orchestrates other XRs to create a complete solution.

## Architecture

```
WhoAmIService (this template)
    ├── WhoAmIApp XR (from template-whoami)
    │   ├── Namespace
    │   ├── Deployment
    │   ├── Service
    │   └── Ingress
    └── CloudflareDNSRecord XR (from template-cloudflare-dnsrecord)
        └── DNS A Record
```

## Restaurant Analogy 🍽️

Think of this as ordering a **combo meal**:
- **WhoAmIService** = The combo meal (burger + fries + drink)
- **WhoAmIApp** = The burger (main application)
- **CloudflareDNSRecord** = The drink (makes it accessible)

The customer orders one thing, but gets a complete meal!

## Prerequisites

1. **Kubernetes cluster** with Crossplane v2.0+
2. **Required templates installed**:
   - `template-whoami` (application deployment)
   - `template-cloudflare-dnsrecord` (DNS management)
3. **Cloudflare configuration** (if using real DNS):
   - Cloudflare account with API token
   - Zone imported (`kubectl get zones.zone.cloudflare.upbound.io`)

## Installation

```bash
# Apply the template
kubectl apply -k https://github.com/open-service-portal/template-whoami-service

# Or clone and apply locally
git clone https://github.com/open-service-portal/template-whoami-service
kubectl apply -k template-whoami-service/
```

## Usage

### Basic Example

```yaml
apiVersion: demo.openportal.dev/v1alpha1
kind: WhoAmIService
metadata:
  name: my-service
spec:
  name: myapp  # Creates myapp.yourdomain.com
```

### Production Example

```yaml
apiVersion: demo.openportal.dev/v1alpha1
kind: WhoAmIService
metadata:
  name: prod-api
spec:
  # Application settings
  name: api
  replicas: 3
  image: "myregistry/api:v2.0.0"
  
  # DNS settings
  zone: "production-zone"
  proxied: true  # Enable Cloudflare proxy
  ttl: 300
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | string | required | Service name (becomes subdomain) |
| `replicas` | integer | 1 | Number of pod replicas |
| `image` | string | traefik/whoami:v1.10.1 | Container image |
| `zone` | string | openportal-zone | Cloudflare Zone resource |
| `proxied` | boolean | false | Enable Cloudflare proxy |
| `ttl` | integer | 1 | DNS TTL (1 = automatic) |

## Status Fields

The template provides combined status from both XRs:

```yaml
status:
  appReady: true        # Application is deployed
  dnsReady: true        # DNS record is configured
  domain: myapp.example.com  # Full domain name
  ipAddress: 203.0.113.1    # Resolved IP address
```

## How It Works

1. **User creates** a WhoAmIService resource
2. **Composition pipeline**:
   - Loads environment configuration (DNS zones, etc.)
   - Creates WhoAmIApp XR with specified parameters
   - Waits for app to be ready (function-auto-ready)
   - Extracts Ingress IP address
   - Creates CloudflareDNSRecord XR pointing to that IP
   - Updates status with combined information

## Troubleshooting

### Check XR Status

```bash
# View the service
kubectl get whoamiservices
kubectl describe whoamiservices my-service

# View underlying XRs
kubectl get whoamiapps
kubectl get cloudflarednsrecords
```

### Common Issues

1. **DNS not resolving**
   - Check if Zone exists: `kubectl get zones.zone.cloudflare.upbound.io`
   - Verify CloudflareDNSRecord is ready: `kubectl get cloudflarednsrecords`

2. **App not accessible**
   - Check WhoAmIApp status: `kubectl get whoamiapps`
   - Verify Ingress has IP: `kubectl get ingress -A`

3. **Wrong IP in DNS**
   - The composition tries to auto-detect from Ingress
   - May use placeholder initially, should update once Ingress is ready

## Development

### Testing Locally

```bash
# Apply the template
kubectl apply -k .

# Create a test service
kubectl apply -f examples/basic.yaml

# Watch the resources
watch kubectl get whoamiservices,whoamiapps,cloudflarednsrecords

# Check the DNS record
nslookup myapp.yourdomain.com
```

### Cleanup

```bash
# Delete the service (cascades to all resources)
kubectl delete whoamiservices my-service

# Remove the template
kubectl delete -k .
```

## Benefits

1. **Simplicity**: Single resource creates everything
2. **Reusability**: Leverages existing tested templates
3. **No duplication**: Reuses existing XRs completely
4. **Flexibility**: Can swap implementations without changing API
5. **Pattern**: Demonstrates composition-of-compositions

## Contributing

This template is part of the Open Service Portal project. Contributions welcome!

## License

Apache 2.0