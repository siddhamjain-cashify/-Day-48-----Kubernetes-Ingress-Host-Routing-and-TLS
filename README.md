# Day 48 --- Kubernetes Ingress, Host Routing, and TLS

## Goal

Learn how Kubernetes Ingress routes HTTP/HTTPS traffic to Services,
configure path- and host-based routing, enable TLS with cert-manager,
test routing behavior, and compare the setup with AWS ALB and
TargetGroupBinding.

## 1. Environment

-   Kubernetes: kind
-   Context: `kind-day41`
-   Ingress controller: ingress-nginx
-   Application namespace: `day48`
-   Test image: `hashicorp/http-echo:1.0.0`
-   TLS: cert-manager with a self-signed certificate

The existing kind cluster was reused.

## 2. Ingress Controller vs Ingress Resource

An **Ingress Controller** watches Ingress resources and implements their
routing rules. In this lab, ingress-nginx runs inside the cluster.

An **Ingress resource** defines rules such as routing `/api` to the API
Service, `/web` to the WEB Service, or routing different hostnames to
different Services.

**Remember:** The Ingress resource defines the rules; the controller
implements them.

## 3. Install ingress-nginx

Applied the kind-specific ingress-nginx manifest:

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/kind/deploy.yaml
```

Verified:

``` bash
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

The controller pod became `Running` and `1/1` ready. The IngressClass
was `nginx`, controller `k8s.io/ingress-nginx`.

### Port-forwarding

Kind did not expose ingress-nginx directly on the Mac's port 80, so
port-forwarding was used.

HTTP:

``` bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```

HTTPS, in another terminal:

``` bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8443:443
```

Keep the port-forward process running while testing.

## 4. Test applications

Created namespace `day48` and deployed two echo applications: - API
returns `Hello from API`. - WEB returns `Hello from WEB`.

Both use `hashicorp/http-echo:1.0.0`, container port `5678`, and a
ClusterIP Service on port `80` targeting port `5678`.

Verify:

``` bash
kubectl get deployments,pods,services -n day48
```

Both application pods were Running and ready.

## 5. Path-based routing

Created Ingress `day48-path-routing`:

  Request path   Backend
  -------------- ----------
  `/api`         `api:80`
  `/web`         `web:80`

Tested:

``` bash
curl -i http://localhost:8080/api
curl -i http://localhost:8080/web
```

Both returned HTTP 200 with the correct echo message.

### Rewrite configuration

Configured regex paths and these annotations:

``` yaml
nginx.ingress.kubernetes.io/use-regex: "true"
nginx.ingress.kubernetes.io/rewrite-target: /$2
```

Paths used:

``` yaml
- path: /api(/|$)(.*)
  pathType: ImplementationSpecific
  backend:
    service:
      name: api
      port:
        number: 80

- path: /web(/|$)(.*)
  pathType: ImplementationSpecific
  backend:
    service:
      name: web
      port:
        number: 80
```

Requests to `/api/users` and `/web/about` reached their corresponding
echo apps and returned HTTP 200. The echo app returns a fixed message,
so the rewritten URI itself was not independently visible in the
response.

## 6. Host-based routing

Created Ingress `day48-host-routing`:

  Hostname      Backend
  ------------- ----------
  `api.local`   `api:80`
  `web.local`   `web:80`

Tested:

``` bash
curl -i -H "Host: api.local" http://127.0.0.1:8080/
curl -i -H "Host: web.local" http://127.0.0.1:8080/
curl -i -H "Host: unknown.local" http://127.0.0.1:8080/
```

Known hosts matched their application routes. The unknown hostname
returned HTTP 404.

## 7. Install cert-manager

Installed cert-manager:

``` bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.2/cert-manager.yaml
```

Verified:

``` bash
kubectl get pods -n cert-manager
```

The cert-manager controller, cainjector, and webhook pods were running.

## 8. Self-signed certificate

Created ClusterIssuer `day48-selfsigned` using:

``` yaml
spec:
  selfSigned: {}
```

Created Certificate `day48-local-cert` in namespace `day48` with: -
Secret: `day48-tls` - DNS names: `api.local`, `web.local` - Duration: 90
days - Renewal window: 15 days before expiry

Verified:

``` bash
kubectl get certificate -n day48
kubectl describe certificate day48-local-cert -n day48
kubectl get secret day48-tls -n day48
```

The Certificate became `Ready=True`; the TLS Secret was created. The
certificate was valid from September 16, 2026, through December 15,
2026, with renewal around November 30, 2026.

## 9. Configure and test HTTPS

Added TLS configuration to the host Ingress:

``` yaml
tls:
  - hosts:
      - api.local
      - web.local
    secretName: day48-tls
```

Tested:

``` bash
curl -k -i --resolve api.local:8443:127.0.0.1 \
  https://api.local:8443/

curl -k -i --resolve web.local:8443:127.0.0.1 \
  https://web.local:8443/
```

Both returned HTTP/2 200: - `api.local` → `Hello from API` - `web.local`
→ `Hello from WEB`

The `-k` option skips certificate trust validation because the
certificate is self-signed. Responses included the
`strict-transport-security` header.

## 10. HTTP-to-HTTPS redirect

Tested:

``` bash
curl -i -H "Host: api.local" http://127.0.0.1:8080/
curl -i -H "Host: web.local" http://127.0.0.1:8080/
curl -i -H "Host: unknown.local" http://127.0.0.1:8080/
```

Results: - `api.local` → `308 Permanent Redirect`, Location points to
`https://api.local`. - `web.local` → `308 Permanent Redirect`, Location
points to `https://web.local`. - `unknown.local` → HTTP 404.

Controller logs showed backend matches `day48-api-80` and
`day48-web-80`.

**Distinction:** The 308 response redirects HTTP to HTTPS. HSTS tells
compatible browsers to use HTTPS for future requests; it is not itself
the redirect.

## 11. Troubleshooting lessons

### `curl localhost:80` failed

Kind was not configured to expose the controller directly on host port
80. Port-forwarding to local port 8080 worked.

### HTTPS port 8443 refused the connection

The HTTPS port-forward was not running. Starting and keeping this
command open fixed it:

``` bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8443:443
```

### NGINX welcome page appeared during an earlier HTTP test

Later tests showed expected 308 redirects for the known hosts and 404
for an unknown host. Controller logs confirmed the intended backend
matches.

### cert-manager briefly appeared to have no resources

A later check confirmed the current context was `kind-day41`,
cert-manager pods were Running, the Certificate was Ready, and the TLS
Secret existed.

## 12. AWS ALB and TargetGroupBinding comparison

### Conceptual traffic flow

User → AWS Application Load Balancer (ALB) → Target Group → Kubernetes
Service/Pod targets.

The AWS Load Balancer Controller watches Kubernetes resources and
coordinates AWS load-balancing resources.

  -----------------------------------------------------------------------
  kind lab                            AWS/EKS concept
  ----------------------------------- -----------------------------------
  ingress-nginx controller            AWS Load Balancer Controller

  Ingress resource                    Ingress interpreted by AWS
                                      controller

  NGINX routing rules                 ALB listener rules

  Kubernetes Service                  Application backend

  Local port-forward                  External ALB endpoint

  Self-signed TLS Secret              ALB HTTPS listener can use an ACM
                                      certificate
  -----------------------------------------------------------------------

### TargetGroupBinding

A TargetGroupBinding connects an existing AWS target group to a
Kubernetes Service.

Illustrative example only; it was not applied:

``` yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: api-target-binding
  namespace: day48
spec:
  serviceRef:
    name: api
    port: 80
  targetGroupARN: <existing-aws-target-group-arn>
```

The target group and Service must exist, and the AWS controller must be
installed and configured.

### Target types

  Target type   Registered target   Typical traffic path
  ------------- ------------------- -------------------------------------
  `instance`    EC2 worker node     ALB → node NodePort → Service → Pod
  `ip`          Pod IP              ALB → Pod IP

No AWS ALB, target group, or EKS resources were created during this lab.

## 13. Completion checklist

  Task                                             Status
  ------------------------------------------------ -----------------------------------
  Explain Ingress Controller vs Ingress resource   Completed
  Install ingress-nginx                            Completed
  Deploy API and WEB apps                          Completed
  Configure `/api` and `/web` routing              Completed
  Configure rewrite rules                          Applied; backend routing verified
  Configure host routing                           Completed
  Test unknown hostname                            Completed --- HTTP 404
  Install cert-manager                             Completed
  Create self-signed certificate                   Completed --- Ready
  Configure HTTPS for both hosts                   Completed
  Verify HTTP-to-HTTPS redirect                    Completed --- HTTP 308
  Compare AWS ALB and TargetGroupBinding           Conceptually covered
  Include journal and review questions             Included below

## 14. Journal

Today I learned that an Ingress resource defines routing rules and an
Ingress Controller implements them. I installed ingress-nginx in my
existing kind cluster and deployed two echo applications behind separate
Services.

I verified path-based routing with `/api` and `/web`, then configured
host-based routing for `api.local` and `web.local`. I applied regex
rewrite rules and tested requests against the backend apps.

I installed cert-manager, created a self-signed ClusterIssuer and
Certificate, and configured HTTPS for both hostnames. Both HTTPS
requests returned HTTP 200 with the correct app response. I also
verified that HTTP requests to known hosts return 308 redirects and
unknown hosts return 404.

A practical lesson was that a local port-forward must remain running
during testing. When port 8443 was not listening, HTTPS failed even
though the Ingress and certificate configuration were still valid.

Finally, I compared the kind NGINX setup with AWS ALB. The AWS Load
Balancer Controller integrates Kubernetes resources with AWS
load-balancing resources, and TargetGroupBinding can associate an
existing target group with a Kubernetes Service.

## 15. Review questions

1.  What is the difference between an Ingress resource and an Ingress
    Controller?
2.  Why did this kind lab require port-forwarding?
3.  How does host-based routing differ from path-based routing?
4.  What does the rewrite-target annotation do?
5.  Why did curl require `-k` for the self-signed certificate?
6.  What is the difference between HTTP 308 and the HSTS header?
7.  What does cert-manager do in this lab?
8.  What is the role of an AWS TargetGroupBinding?
9.  What is the difference between ALB `instance` and `ip` target types?
10. Why should an unknown hostname return 404 rather than reach an
    unintended application?

## Key takeaway

**Ingress routes traffic to Kubernetes Services. TLS secures the
connection. In AWS, the Load Balancer Controller integrates Kubernetes
routing configuration with AWS load-balancing resources, and
TargetGroupBinding can connect an existing target group to a Kubernetes
Service.**


