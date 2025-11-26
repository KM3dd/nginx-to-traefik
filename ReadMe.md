# Kubernetes Ingress Migration: From Nginx to Traefik 

The Nginx Ingress Controller is retiring, and the Kubernetes community strongly encourages migration to the [Gateway API](https://gateway-api.sigs.k8s.io/), which offers a more expressive and maintainable approach for exposing applications in Kubernetes.However, the tight deadline for the Nginx Ingress end of life might force a premature adoption of the Gateway API. The Gateway API represents a different and more complex management model than the traditional Ingress Controller. This complexity makes it hard for teams that are behind schedule or manage complex Nginx Ingress setups.


I propose a two-phase solution for a less stressful transition:

- **Phase 1 :** Stop the bleeding and migrate immediately to the Traefik Ingress Controller, why ? traefik is often simpler, allows for faster adaptation and migration, and more important it supports many Nginx annotations natively via its Kubernetes Ingress Provider. Only minimal configuration adaptations are typically required.

  **_NOTE:_**  (please check [This section]([#Annotation-Replacement-Map:-Nginx-Annotations-to-Traefik-CRDs](https://github.com/KM3dd/nginx-to-traefik?tab=readme-ov-file#annotation-replacement-map-nginx-annotations-to-traefik-crds)) to check unsupported annotations and how to adapt them before applying anything in this tutorial)
- **Phase 2 :** Plan the Gateway API adoption on your team's timeline, free from end of life pressure. Assess readiness, test thoroughly during normal sprint cycles, and implement the new approach when business conditions are optimal. One problem at a time.

This Tutorial covers the first phase of the migration


## Prerequisites

Before starting the migration, ensure you have the following:

- A running Kubernetes cluster ($\ge 1.19$).

- kubectl configured to interact with your cluster.

- helm installed for easy deployment.

- Existing services currently exposed via the Nginx Ingress Controller.

# Step-by-Step Migration Guide 

A step-by-step tutorial to migrate from Nginx to traefik with zero down time strategy

## Step 1: Install the Traefik Ingress Controller

We will install Traefik alongside Nginx. It will use a separate IngressClass, so both controllers can coexist without conflict.

Add the Traefik Helm Repository:
```bash
$ helm repo add traefik https://traefik.github.io/charts
$ helm repo update
```

### Deploy Traefik:
We deploy Traefik and creates a new IngressClass named traefik.

```bash
$ helm upgrade --install traefik traefik/traefik --namespace traefik \
  --create-namespace \
  --set="image.tag=v3.6.2" \
  --set="logs.general.level=DEBUG" \
  --set="service.type=LoadBalancer" \
  --set="additionalArguments={--providers.kubernetesIngressNGINX}"
```

### Verify Deployment: 
Check that the Traefik deployment and service are running.

```bash
$ kubectl -n traefik get po
$ kubectl -n traefik get svc traefik
```
 
 **_IMPORTANT:_** Note the new external IP address of the traefik LoadBalancer service.

## Step 2: Duplicate and Target Ingress Resources

Changing the ingressClassName on the existing Ingress resource (the one Nginx is actively using) will immediately cause Nginx to ignore it, leading to downtime. The correct zero-downtime strategy is to DUPLICATE the Ingress resource, allowing both controllers to run the same configuration in parallel.

### Duplicate the Ingress Configuration: 
Get the YAML for your existing Ingress and save it to a new file.

```bash
$ kubectl get ingress my-app-ingress -n my-namespace -o yaml > my-app-ingress-traefik.yaml
```

Replace my-app-ingress and my-namespace with your actual Ingress name and namespace.

### Edit the Duplicated Resource: 
Open traefik-ingress-config.yaml and make the following two mandatory changes:

- Change 1: Update the Name: Change the metadata.name (e.g., from my-app-ingress to my-app-ingress-traefik).

- Change 2: Set the Ingress Class: Ensure spec.ingressClassName is set to traefik.

Example of Required Changes in YAML:
```yaml
metadata:
  name: my-app-ingress-traefik  # <-- Changed Name
spec:
  ingressClassName: traefik     # <-- Set Traefik Class
  # ... rest of your rules/TLS stays the same

```
Apply the New Traefik Ingress: Apply the modified file to create the duplicate resource.
``` bash
$ kubectl apply -f traefik-ingress-config.yaml
```
 **_CRITICAL STATE :_** At this point, both Nginx (watching the original Ingress) and Traefik (watching the duplicated Ingress) are configured and serving traffic for the same hostname. Your live traffic continues to flow through the Nginx Load Balancer IP without interruption.

## Step 3: Verification and Traffic Cutover

Shifting live traffic from Nginx to Traefik.

### Test with New IP: 
Access your application directly using the new external IP address of the Traefik LoadBalancer (from Step 1) and your application's hostname via a custom Host header or by modifying your local hosts file.
```bash
$ curl -H "Host: my-app.com" -I http://<Traefik-LoadBalancer-IP>/<path>
```

Verify that your application is responding correctly (e.g., HTTP $\text{200}$ status) and all advanced features (like rewrites) are working.

### DNS Cutover (Switch Traffic): 
Once you are 100% confident in the Traefik setup, update your external DNS records (A/CNAME) to point the application hostname (e.g., api.yourcompany.com) to the new external IP address of the Traefik LoadBalancer Service.

### Monitor: 
Monitor traffic flow, error rates, and latency for a time period equal to your DNS TTL (Time to Live) to ensure all old traffic has drained and the new system is stable.

## Step 4: Decommission Nginx Ingress Controller

Once all Ingresses have been migrated, the DNS TTL has expired, and the system is stable, you can safely remove the old Nginx Ingress Controller.

### Clean up Old Ingress: 
Before uninstalling Nginx, delete the original Nginx-managed Ingress resource.

```bash
$ kubectl delete ingress my-app-ingress -n my-namespace
```

### Uninstall Nginx: 
If you deployed Nginx using Helm, use the following command (adjusting the name/namespace as necessary):

```bash
$ helm uninstall nginx-ingress --namespace <nginx-ingress-namespace>
```

If you deployed via YAML, delete the deployment and service resources manually.

### Final Cleanup: 
You can now safely delete the Nginx Ingress Class resource if one was created.


# Annotation Replacement Map: Nginx Annotations to Traefik CRDs

When migrating, the following table shows the Nginx-specific annotations you need to worry about and their native Traefik CRD equivalents. While Traefik supports many Nginx annotations for basic compatibility, replacing them with native Middlewares is the best practice.

Some unsupported nginx ingress annotations : 

| Nginx Annotation                                                   | Traefik CRD / Feature to Use Instead                                                                                |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `nginx.ingress.kubernetes.io/rewrite-target`                       | Traefik Middleware CRD [example](https://community.traefik.io/t/traefik-ingress-rewrite-target-does-not-work/13127) |
| `nginx.ingress.kubernetes.io/force-ssl-redirect`                   | Traefik Middleware CRD [example](https://devopsx.com/traefik-ingress-redirect-http-to-https/)                       |
| `nginx.ingress.kubernetes.io/auth-type / auth-secret / auth-realm` | Traefik Middleware + Ingress Route CRDs [example](https://major.io/p/basic-auth-with-traefik-on-kubernetes/)        |

These are high-usage annotations that will require manual adaptation.
### References

- [Treafik CRDs](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-crd/#list-of-resources)
- [Treafik ingress with nginx annotation](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress-nginx/)
- [Unsupported nginx annotations](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress-nginx/#opt-nginx-ingress-kubernetes-iorewrite-target)
- [migrate from ingress nginx to traefik](https://traefik.io/blog/migrate-from-ingress-nginx-to-traefik-now)
- [The Great Kubernetes Ingress Transition: From ingress-nginx EoL to Modern Cloud-Native](https://traefik.io/blog/transition-from-ingress-nginx-to-traefik)
- [Nginx ingress retirement : what do you need to know](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
  
## Next Steps : 
- Audit all Ingress resources in your cluster
- Progressively migrate to Gateway API once Traefik is fully functional

## 🤝 Contribution
Contributions are welcome.  If you’d like to improve this guide, fix an issue, or add missing migration patterns:

1. Fork the repository  
2. Create a feature branch  
3. Commit your changes with clear messages  
4. Open a Pull Request

Thank you for helping make the migration smoother for everyone.
