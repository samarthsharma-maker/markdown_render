## Final 20 Interview Questions (GCP-Focused, Clubbed)

### ☁️ GCP Architecture & IAM

1. **Design a highly available SaaS application on GCP** — include VPC design, load balancing, IAM, and multi-zone strategy.

**Answer:**

**VPC Design:**
- Create a custom VPC with regional subnets across multiple regions (e.g., us-central1, us-east1, europe-west1) for geographic redundancy
- Implement subnet segmentation: separate subnets for web tier (10.0.1.0/24), application tier (10.0.2.0/24), data tier (10.0.3.0/24), and management (10.0.4.0/24)
- Enable Private Google Access on subnets to allow private communication with GCP APIs without public IPs
- Configure Cloud NAT for outbound internet access from private instances
- Use VPC firewall rules with priority-based ordering: deny-all baseline (priority 65534), then allow specific traffic (SSH from bastion, HTTPS from load balancer, inter-tier communication)
- Implement VPC Flow Logs for network monitoring and security analysis

**Load Balancing Strategy:**
- Use Global HTTP(S) Load Balancer for multi-region traffic distribution with Cloud CDN enabled for static content
- Configure backend services with instance groups in multiple zones within each region (minimum 3 zones for 99.99% SLA)
- Implement health checks: HTTP health checks every 10s with 2 consecutive successes required, 3 failures for unhealthy marking
- Set up Cloud Armor security policies at load balancer level for DDoS protection and WAF rules
- Use SSL certificates managed by Google Certificate Manager with automatic renewal
- Configure session affinity based on client IP or generated cookie for stateful applications

**Multi-Zone/Multi-Region Strategy:**
- Deploy GKE regional clusters with node pools distributed across 3+ zones within a region
- Use Regional Persistent Disks for stateful workloads with automatic replication across zones
- Implement Cloud SQL with HA configuration: primary in zone-a, synchronous replica in zone-b, read replicas in additional regions
- Deploy Cloud Spanner for globally distributed, strongly consistent database requirements
- Use Cloud Storage with multi-regional or dual-regional buckets for object storage
- Configure Traffic Director or Global Load Balancer for automatic failover between regions based on health checks

**IAM Architecture:**
- Implement organization-level policies with folder structure: Production, Staging, Development
- Use separate projects per environment with project-level IAM bindings
- Apply principle of least privilege: custom roles instead of primitive roles (Owner, Editor, Viewer)
- Service accounts for application workloads with Workload Identity binding to Kubernetes service accounts
- Use IAM Conditions for time-based or attribute-based access control (e.g., access only from corporate IP ranges)
- Implement service account impersonation for privileged operations instead of key downloads
- Enable audit logging for all IAM changes and data access
- Use VPC Service Controls to create security perimeters around sensitive resources
2. Explain **GCP project, folder, and organization hierarchy** and how you enforce access control using IAM.

**Answer:**

**Hierarchy Structure:**
- **Organization Node:** Root node representing the company (linked to Cloud Identity or Google Workspace domain). Provides centralized control and visibility across all GCP resources
- **Folders:** Organizational units under the organization node, typically representing departments, teams, environments, or business units. Folders can be nested up to 10 levels deep
- **Projects:** Fundamental resource containers where all GCP resources exist. Each project has unique project ID, project number, and project name
- **Resources:** Individual GCP services (Compute Engine instances, GKE clusters, Cloud SQL databases) exist within projects

**IAM Inheritance Model:**
- Policies are inherited downward through the hierarchy: Organization → Folder → Project → Resource
- Child resources inherit all IAM bindings from parent resources (additive inheritance, not subtractive)
- More specific bindings at lower levels combine with inherited bindings
- Use `gcloud projects get-iam-policy` to view effective permissions at any level

**Access Control Enforcement Strategies:**

**Organization-Level Policies:**
- Apply organization-wide constraints using Organization Policy Service (e.g., `constraints/compute.vmExternalIpAccess` to restrict public IPs)
- Set baseline IAM roles for security/compliance teams: Organization Admin, Security Reviewer, Billing Account Administrator
- Enable mandatory audit logging for all projects via organization policy
- Enforce domain-restricted sharing to prevent external access

**Folder-Level Policies:**
- Structure folders by environment: Production, Staging, Development folders with different access controls
- Grant environment-specific roles at folder level (e.g., developers get Editor on Development folder, Viewer on Production)
- Use folders for compliance boundaries (e.g., PCI-compliant workloads in dedicated folder with stricter policies)
- Apply cost center tags and billing labels at folder level for chargeback

**Project-Level Policies:**
- Grant application-specific access using custom roles with minimal permissions
- Use service accounts for workload identity, never user accounts for automation
- Implement separation of duties: separate projects for CI/CD, data processing, and production workloads
- Enable Data Access audit logs at project level for sensitive data access tracking

**Best Practices:**
- Use groups for IAM bindings instead of individual users for easier management
- Implement IAM Recommender to identify and remove excessive permissions
- Use IAM Conditions for context-aware access (IP address, time, resource attributes)
- Apply `constraints/iam.allowedPolicyMemberDomains` to prevent external accounts
- Regular access reviews using Cloud Asset Inventory and IAM policy analyzer
- Use service account impersonation chains for privileged operations instead of key distribution
3. How do **Service Accounts, IAM roles, and Workload Identity** work together in GCP?

**Answer:**

**Service Accounts:**
- Special type of Google account representing non-human identities (applications, VMs, services)
- Identified by email format: `SERVICE_ACCOUNT_NAME@PROJECT_ID.iam.gserviceaccount.com`
- Two types: user-managed (created explicitly) and Google-managed (created automatically by GCP services)
- Can be granted IAM roles to access GCP resources
- Support two authentication methods: service account keys (JSON/P12 files) and short-lived tokens via metadata server or Workload Identity

**IAM Roles:**
- Collections of permissions that define what actions can be performed on resources
- Three types:
  - **Primitive roles:** Owner, Editor, Viewer (broad, avoid in production)
  - **Predefined roles:** Curated by Google for specific services (e.g., `roles/container.developer`, `roles/storage.objectViewer`)
  - **Custom roles:** User-defined with specific permissions (e.g., `organizations/ORG_ID/roles/customGKEDeployer`)
- Roles are bound to service accounts using IAM policy bindings
- Permissions follow format: `service.resource.verb` (e.g., `storage.objects.get`)

**Workload Identity:**
- Recommended method for GKE pods to access GCP services without service account keys
- Eliminates security risks of storing and rotating JSON key files
- Enables Kubernetes service accounts to impersonate GCP service accounts

**How They Work Together:**

**Traditional Approach (Without Workload Identity):**
1. Create GCP service account with required IAM roles
2. Generate JSON key file for service account
3. Store key in Kubernetes Secret
4. Mount secret in pod and set `GOOGLE_APPLICATION_CREDENTIALS` environment variable
5. Application uses key to authenticate to GCP APIs
**Problems:** Key rotation burden, secret sprawl, security risk if keys leak

**Modern Approach (With Workload Identity):**

**Setup:**
1. Enable Workload Identity on GKE cluster: `--workload-pool=PROJECT_ID.svc.id.goog`
2. Create Kubernetes service account: `kubectl create serviceaccount KSA_NAME -n NAMESPACE`
3. Create GCP service account with required IAM roles (e.g., `roles/storage.objectViewer`)
4. Bind KSA to GSA using IAM policy:
```
gcloud iam service-accounts add-iam-policy-binding GSA_NAME@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]"
```
5. Annotate Kubernetes service account:
```
kubectl annotate serviceaccount KSA_NAME \
  iam.gke.io/gcp-service-account=GSA_NAME@PROJECT_ID.iam.gserviceaccount.com
```
6. Deploy pod using the Kubernetes service account

**Runtime Flow:**
1. Pod starts with Kubernetes service account
2. Application requests GCP credentials using Application Default Credentials (ADC)
3. GKE metadata server intercepts request
4. Metadata server exchanges Kubernetes token for GCP access token via token exchange
5. GCP validates that KSA is authorized to impersonate GSA (via `roles/iam.workloadIdentityUser` binding)
6. Short-lived OAuth 2.0 access token returned to application (valid for 1 hour)
7. Application uses token to authenticate to GCP APIs
8. Token automatically refreshed before expiration

**Benefits:**
- No service account keys to manage or rotate
- Automatic credential rotation (tokens expire in 1 hour)
- Fine-grained access control per Kubernetes namespace and service account
- Audit trail shows which Kubernetes workload accessed which GCP resource
- Reduced attack surface (no long-lived credentials stored in cluster)
4. Compare **Shared VPC vs VPC Peering** and give a real use case for each.

**Answer:**

**Shared VPC:**

**Architecture:**
- Centralized network management model where one host project shares subnets with multiple service projects
- Host project owns VPC network, subnets, firewall rules, and routes
- Service projects can create resources (VMs, GKE clusters) in shared subnets without managing network infrastructure
- Requires organization-level setup with Shared VPC Admin role

**Key Characteristics:**
- Single VPC network shared across multiple projects
- Centralized network administration and security policy enforcement
- Subnet-level sharing (can share specific subnets with specific projects)
- IAM-based access control using `compute.networkUser` role at subnet or project level
- Shared VPC Admin manages network, service project admins manage compute resources
- Supports hybrid connectivity (Cloud VPN, Cloud Interconnect) from host project available to all service projects
- No additional network hops or latency
- RFC 1918 IP address space managed centrally

**Use Case:**
Enterprise with multiple teams/applications requiring centralized network governance:
- Host project: `network-prod` (managed by central IT/network team)
- Service projects: `app-frontend`, `app-backend`, `data-analytics`
- Network team controls firewall rules, VPN connections, DNS configuration
- Application teams deploy resources in shared subnets without network admin privileges
- Compliance requirements mandate centralized network logging and security controls
- All projects communicate using internal IPs without VPC peering overhead

**VPC Peering:**

**Architecture:**
- Decentralized model connecting two independent VPC networks
- Each VPC maintains its own network configuration, firewall rules, and routes
- Bidirectional peering connection established between VPCs
- Can peer VPCs across projects, organizations, or even between GCP and other clouds

**Key Characteristics:**
- Two separate VPC networks with independent administration
- Private RFC 1918 connectivity between VPCs without internet gateway
- Non-transitive (if VPC-A peers with VPC-B, and VPC-B peers with VPC-C, VPC-A cannot reach VPC-C unless directly peered)
- No overlapping CIDR ranges allowed between peered VPCs
- Each VPC maintains its own firewall rules (must allow traffic from peered VPC CIDR)
- Supports cross-organization peering for partner/vendor connectivity
- No bandwidth costs for traffic between peered VPCs in same region
- Maximum 25 peering connections per VPC

**Use Case:**
SaaS provider connecting customer-managed VPCs to multi-tenant platform:
- SaaS platform VPC: `saas-platform-prod` (10.0.0.0/16)
- Customer A VPC: `customer-a-vpc` (172.16.0.0/16) - managed by Customer A
- Customer B VPC: `customer-b-vpc` (192.168.0.0/16) - managed by Customer B
- Each customer maintains full control over their VPC (firewall rules, routing, security)
- VPC peering provides private connectivity from customer VPCs to SaaS platform APIs
- Customers isolated from each other (no transitive routing)
- SaaS provider doesn't need admin access to customer VPCs
- Customers can enforce their own security policies and compliance requirements

**Comparison Table:**

| Feature | Shared VPC | VPC Peering |
|---------|-----------|-------------|
| Network ownership | Centralized (host project) | Decentralized (each VPC independent) |
| Administration | Single network team | Multiple independent teams |
| IAM integration | Subnet-level IAM controls | No IAM integration |
| Transitive routing | Yes (all service projects share routes) | No (non-transitive) |
| IP overlap | Not allowed within same VPC | Not allowed between peered VPCs |
| Cross-organization | Requires organization node | Supported |
| Hybrid connectivity | Shared from host project | Each VPC needs own VPN/Interconnect |
| Use case | Enterprise multi-project environments | Partner/vendor connectivity, acquisitions |
5. How do you securely manage **secrets and credentials** in GCP for applications and CI/CD?

**Answer:**

**Secret Manager (Primary Solution):**

**Core Features:**
- Centralized secret storage with automatic encryption at rest using Cloud KMS
- Secret versioning with ability to enable/disable/destroy specific versions
- Automatic secret rotation support via Cloud Functions or Cloud Scheduler
- Regional and multi-regional replication for high availability
- IAM-based access control at secret or version level
- Audit logging for all secret access and modifications
- Integration with Cloud Build, GKE, Cloud Run, Cloud Functions

**Implementation:**

**Creating and Storing Secrets:**
```bash
echo -n "db-password-value" | gcloud secrets create db-password \
  --data-file=- \
  --replication-policy="automatic" \
  --labels=env=prod,app=backend
```

**Accessing from Applications:**
```python
from google.cloud import secretmanager
client = secretmanager.SecretManagerServiceClient()
name = f"projects/PROJECT_ID/secrets/db-password/versions/latest"
response = client.access_secret_version(request={"name": name})
secret_value = response.payload.data.decode('UTF-8')
```

**GKE Integration (Workload Identity + Secret Manager):**
1. Create GCP service account with `roles/secretmanager.secretAccessor`
2. Bind to Kubernetes service account via Workload Identity
3. Application uses Secret Manager API to fetch secrets at runtime
4. Alternatively, use Secret Manager CSI driver to mount secrets as volumes:
```yaml
volumes:
- name: secrets-store
  csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: "app-secrets"
```

**CI/CD Integration:**

**Cloud Build:**
```yaml
steps:
- name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    gcloud secrets versions access latest --secret=api-key > /workspace/api-key.txt
availableSecrets:
  secretManager:
  - versionName: projects/PROJECT_ID/secrets/deploy-key/versions/latest
    env: 'DEPLOY_KEY'
```

**GitHub Actions with Workload Identity Federation:**
```yaml
- uses: google-github-actions/auth@v1
  with:
    workload_identity_provider: 'projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID'
    service_account: 'github-actions@PROJECT_ID.iam.gserviceaccount.com'
- name: Access Secret
  run: |
    gcloud secrets versions access latest --secret=deployment-token
```

**Alternative Solutions:**

**Berglas (Open Source):**
- Encrypts secrets using Cloud KMS before storing in Cloud Storage
- Provides CLI and libraries for secret management
- Supports automatic decryption via environment variable injection
- Useful for legacy applications not using Secret Manager

**Kubernetes Secrets with Encryption:**
- Enable application-layer secrets encryption in GKE using Cloud KMS
- Secrets encrypted at etcd level using customer-managed encryption keys
- Use for Kubernetes-native secrets (TLS certs, image pull secrets)
- Not recommended for application secrets (use Secret Manager instead)

**Best Practices:**

**Access Control:**
- Grant `roles/secretmanager.secretAccessor` only to service accounts that need specific secrets
- Use IAM Conditions to restrict access by time, IP, or resource attributes
- Never use `roles/secretmanager.admin` for application service accounts
- Implement separate secrets per environment (dev, staging, prod)

**Secret Rotation:**
- Implement automated rotation using Cloud Scheduler + Cloud Functions
- Store multiple active versions during rotation period
- Update applications to handle version transitions gracefully
- Disable old versions after rotation validation

**Audit and Monitoring:**
- Enable Data Access audit logs for Secret Manager
- Set up Cloud Monitoring alerts for unauthorized access attempts
- Monitor secret access patterns using Log Analytics
- Implement secret usage dashboards in Cloud Monitoring

**Security Hardening:**
- Never commit secrets to version control (use .gitignore, pre-commit hooks)
- Avoid service account key files; use Workload Identity or Workload Identity Federation
- Implement secret scanning in CI/CD pipelines (tools like gitleaks, truffleHog)
- Use VPC Service Controls to restrict Secret Manager API access to specific networks
- Enable customer-managed encryption keys (CMEK) for sensitive secrets
- Implement secret expiration policies and automated cleanup of unused secrets

---

###  Kubernetes & GKE

6. Walk through the **end-to-end flow of deploying a microservice to GKE** (CI → image → deploy → expose).

**Answer:**

**Phase 1: Source Code and CI Setup**

**Repository Structure:**
```
app/
├── src/
│   └── main.py
├── Dockerfile
├── requirements.txt
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── cloudbuild.yaml
```

**Dockerfile (Multi-stage Build):**
```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY src/ .
ENV PATH=/root/.local/bin:$PATH
USER 1000
EXPOSE 8080
CMD ["python", "main.py"]
```

The multi-stage Dockerfile above demonstrates a critical optimization pattern. The builder stage installs dependencies in a temporary layer, while the final stage copies only the compiled artifacts. This separation ensures the production image contains no build tools, package managers, or source code, reducing both image size and attack surface. The USER directive switches to a non-root user (UID 1000) before the CMD instruction, ensuring the application process never runs with elevated privileges even if the container runtime defaults to root.

**Phase 2: CI Pipeline (Cloud Build)**

Cloud Build executes as a series of containerized steps, each running in isolation. The pipeline below demonstrates the four critical stages: test execution, image building, security scanning, and registry push. Each step uses a purpose-built container image from Google's cloud-builders repository, ensuring consistent and reproducible builds across environments.

**cloudbuild.yaml:**
```yaml
steps:
# 1. Run Tests
- name: 'python:3.11'
  args: ['pip install -r requirements.txt', 'pytest tests/']

# 2. Build & Tag Image
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/myapp:$SHORT_SHA', '.']

# 3. Security Scan
- name: 'gcr.io/cloud-builders/gcloud'
  args: ['container', 'images', 'scan', 'gcr.io/$PROJECT_ID/myapp:$SHORT_SHA']

# 4. Deploy to GKE
- name: 'gcr.io/cloud-builders/gke-deploy'
  args:
  - 'run'
  - '--image=gcr.io/$PROJECT_ID/myapp:$SHORT_SHA'
  - '--cluster=prod-cluster'
  - '--location=us-central1'
```

**Trigger Setup:**
```bash
gcloud builds triggers create github \
  --repo-name=myapp \
  --repo-owner=myorg \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml \
  --service-account=projects/PROJECT_ID/serviceAccounts/cloudbuild-sa@PROJECT_ID.iam.gserviceaccount.com
```

**Phase 3: Kubernetes Manifests**

Kubernetes manifests define the desired state of applications. The deployment manifest below incorporates several reliability patterns: rolling updates with zero unavailable pods ensure continuous availability during deployments, resource requests and limits enable proper scheduling and prevent resource starvation, and probes provide automatic health monitoring. The securityContext enforces non-root execution at the pod level, preventing privilege escalation attacks.

**k8s/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: myapp
        image: gcr.io/PROJECT_ID/myapp:latest
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 500m
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
```

**k8s/service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
```

The Service manifest uses the NEG (Network Endpoint Group) annotation to enable container-native load balancing. Without this annotation, traffic flows through kube-proxy using iptables rules, adding latency and reducing observability. With NEGs, the Google Cloud Load Balancer routes directly to pod IPs, bypassing the node networking layer entirely.

**k8s/ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"
    networking.gke.io/managed-certificates: "myapp-cert"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

**Phase 4: Deployment Execution**

The deployment process follows a specific sequence to ensure zero downtime. First, the new image is built and pushed to Artifact Registry with a unique SHA-based tag, guaranteeing immutability. The kubectl set image command updates the deployment spec, triggering a rolling update controlled by the strategy configuration. Kubernetes creates new pods with the updated image before terminating old ones, maintaining the minimum available replica count throughout.

**1. Image Build and Push:**
- Cloud Build triggered on git push to main branch
- Docker builds image with SHA-based tag
- Image pushed to Artifact Registry (gcr.io or us-docker.pkg.dev)
- Vulnerability scanning runs automatically

**2. Kubernetes Deployment:**
```bash
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/myapp -n production
```

**3. Service Creation:**
```bash
kubectl apply -f k8s/service.yaml
kubectl get svc myapp-service -n production
```

**4. Ingress Configuration:**
```bash
# Reserve static IP
gcloud compute addresses create myapp-ip --global

# Create managed certificate
kubectl apply -f - <<EOF
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: myapp-cert
  namespace: production
spec:
  domains:
  - myapp.example.com
EOF

# Apply ingress
kubectl apply -f k8s/ingress.yaml

# Wait for ingress provisioning
kubectl get ingress myapp-ingress -n production -w
```

**Phase 5: Verification**

**Check Deployment:**
```bash
kubectl get pods -n production -l app=myapp
kubectl logs -n production -l app=myapp --tail=50
kubectl describe deployment myapp -n production
```

**Check Service:**
```bash
kubectl get svc myapp-service -n production
kubectl get endpoints myapp-service -n production
```

**Check Ingress:**
```bash
kubectl describe ingress myapp-ingress -n production
curl -H "Host: myapp.example.com" http://INGRESS_IP/
```

**Phase 6: Monitoring and Observability**

**Setup:**
- Cloud Logging automatically collects container stdout/stderr
- Cloud Monitoring tracks CPU, memory, request latency
- Configure uptime checks for external monitoring
- Set up alerting policies for pod restarts, high error rates

**Key Metrics:**
```bash
kubectl top pods -n production -l app=myapp
kubectl get hpa -n production
```
7. Explain **GKE Autopilot vs Standard**, and when you’d choose one over the other.

**Answer:**

GKE offers two distinct operational modes that fundamentally differ in management responsibility and flexibility. Understanding the architectural differences helps determine which mode aligns with specific workload requirements and organizational capabilities.

**GKE Autopilot:**

**Architecture:**
Autopilot represents a fully managed Kubernetes experience where Google assumes responsibility for node infrastructure, security hardening, and capacity management. The control plane and data plane are both managed, eliminating operational overhead for node provisioning, patching, and scaling. This mode implements a pay-per-pod model where billing is based on actual resource requests rather than provisioned node capacity.
- Fully managed Kubernetes service where Google manages the entire cluster infrastructure
- Google provisions and manages nodes, node pools, and cluster configuration
- Pre-configured with security best practices and optimized settings
- Pay-per-pod resource requests (CPU/memory) rather than per-node

**Key Characteristics:**
- **Node Management:** Fully automated, no node pool configuration required
- **Scaling:** Automatic node provisioning based on pod resource requests
- **Security:** Hardened by default (Shielded GKE nodes, Workload Identity enforced, no SSH access to nodes)
- **Networking:** VPC-native clusters mandatory, uses GKE Dataplane V2 (eBPF-based)
- **Pod Spec Restrictions:** Enforces resource requests/limits, no privileged containers, no hostPath volumes
- **Upgrades:** Automatic node and control plane upgrades managed by Google
- **Pricing:** Billed based on pod resource requests (vCPU-hours and GB-hours)
- **SLA:** 99.9% uptime SLA for regional clusters (99.5% for zonal)

**Limitations:**
- Cannot SSH into nodes or run DaemonSets with privileged access
- No support for GPUs, Windows nodes, or specialized machine types
- Cannot use local SSDs or hostPath volumes
- Limited customization of node configuration
- Cannot modify kubelet flags or node OS settings

**GKE Standard:**

**Architecture:**
- User-managed Kubernetes where you configure and manage node pools
- Full control over node machine types, OS, networking, and cluster settings
- Requires manual capacity planning and node pool management

**Key Characteristics:**
- **Node Management:** Manual node pool creation, sizing, and configuration
- **Scaling:** Configure Cluster Autoscaler with min/max node counts per pool
- **Security:** Configurable security settings (can enable/disable features)
- **Networking:** Supports routes-based or VPC-native clusters
- **Pod Spec Flexibility:** No restrictions on pod specifications
- **Upgrades:** Manual control over upgrade timing and maintenance windows
- **Pricing:** Billed per node (machine type pricing) plus cluster management fee
- **SLA:** 99.95% uptime SLA for regional clusters with multiple zones

**Advanced Features:**
- Support for GPUs, TPUs, Windows nodes, ARM-based nodes
- Custom machine types and sole-tenant nodes
- Local SSDs for high-performance storage
- DaemonSets with privileged access for monitoring/security agents
- Node taints, tolerations, and advanced scheduling
- Custom CNI plugins and network policies

**Comparison Table:**

| Feature | Autopilot | Standard |
|---------|-----------|----------|
| Node management | Fully automated | Manual configuration |
| Pricing model | Pay-per-pod (resource requests) | Pay-per-node |
| Capacity planning | Automatic | Manual |
| Security posture | Hardened by default | Configurable |
| Pod restrictions | Enforced (no privileged) | None |
| GPU/TPU support | No | Yes |
| Windows containers | No | Yes |
| Node customization | Limited | Full control |
| Operational overhead | Minimal | High |
| Best for | Standard workloads, startups | Custom requirements, ML/AI |

**When to Choose Autopilot:**

**Use Cases:**
- Stateless web applications and microservices with standard resource requirements
- Development and staging environments where operational simplicity is prioritized
- Teams with limited Kubernetes expertise or small DevOps teams
- Cost optimization for variable workloads (pay only for pod resources used)
- Startups and small companies wanting managed infrastructure
- Compliance-focused environments requiring hardened security by default

**Example Scenario:**
SaaS application with multiple microservices running standard containerized workloads. Team wants to focus on application development rather than cluster management. No special hardware requirements (GPUs) or privileged access needed. Variable traffic patterns benefit from automatic scaling without over-provisioning nodes.

**When to Choose Standard:**

**Use Cases:**
- Machine learning workloads requiring GPUs or TPUs
- Windows container workloads
- Applications requiring privileged containers or hostPath volumes
- Custom networking requirements (specific CNI plugins, advanced network policies)
- Batch processing with specific node configurations or local SSDs
- Legacy applications with specific OS or kernel requirements
- Multi-tenant environments requiring node isolation and taints
- Compliance requirements needing specific node hardening or audit logging

**Example Scenario:**
ML platform running TensorFlow training jobs on GPU nodes. Requires privileged DaemonSets for monitoring agents (Datadog, Prometheus node exporter). Uses local SSDs for high-performance data caching. Needs fine-grained control over node pool scaling and maintenance windows to avoid disrupting long-running training jobs.

**Migration Considerations:**
- Cannot convert Autopilot to Standard or vice versa (requires new cluster)
- Autopilot enforces resource requests/limits (must update pod specs before migration)
- Standard to Autopilot migration requires removing privileged workloads and DaemonSets
- Cost modeling needed: Autopilot can be cheaper for low-utilization clusters, more expensive for high-density workloads
8. How do **Ingress, Services, and Google Load Balancer** integrate in GKE?

**Answer:**

The integration between Kubernetes networking primitives and Google Cloud infrastructure follows a hierarchical model. Understanding this architecture is essential for troubleshooting connectivity issues and optimizing application performance.

**Component Overview:**

The three components form a layered abstraction: Services provide stable internal endpoints for pod discovery, Ingress defines external routing rules, and Google Cloud Load Balancers handle the actual traffic distribution. Each layer adds functionality while abstracting complexity from the layer above.

**Services:**
- Kubernetes abstraction providing stable endpoint for pod access
- Types: ClusterIP (internal), NodePort (node-level), LoadBalancer (external L4), ExternalName
- Label selectors route traffic to matching pods
- Maintains endpoint list dynamically as pods scale

**Ingress:**
- Kubernetes resource for HTTP(S) routing and load balancing
- Defines rules for routing external traffic to services
- Supports host-based and path-based routing
- Manages SSL/TLS termination

**Google Cloud Load Balancer:**
- GCP-native load balancing infrastructure
- Created automatically when Ingress is deployed in GKE
- Types: Global HTTP(S) LB, Internal HTTP(S) LB, Network LB

**Integration Flow:**

**1. Service Creation (ClusterIP):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```
- Service creates endpoint list of pod IPs matching selector
- Annotation `cloud.google.com/neg: '{"ingress": true}'` enables Network Endpoint Groups (NEGs)
- NEGs provide container-native load balancing (direct pod IP routing, bypassing kube-proxy)

**2. Ingress Resource:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "app-ip"
    networking.gke.io/managed-certificates: "app-cert"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1/*
        pathType: ImplementationSpecific
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

**3. GKE Ingress Controller Actions:**
- GKE Ingress controller watches for Ingress resources
- Provisions Google Cloud HTTP(S) Load Balancer components:
  - **Global Forwarding Rule:** Entry point with static IP
  - **Target HTTP(S) Proxy:** SSL termination and routing logic
  - **URL Map:** Path-based routing rules from Ingress spec
  - **Backend Service:** Maps to Kubernetes Service
  - **Network Endpoint Group (NEG):** Container-native endpoints (pod IPs)
  - **Health Checks:** Derived from readiness probes

**4. Traffic Flow:**
```
Client Request (api.example.com/v1/users)
  ↓
Global Forwarding Rule (Static IP: 34.120.x.x)
  ↓
Target HTTPS Proxy (SSL termination, cert validation)
  ↓
URL Map (matches /v1/* → backend-service)
  ↓
Backend Service (load balancing algorithm)
  ↓
Network Endpoint Group (pod IPs: 10.4.1.5:8080, 10.4.2.3:8080)
  ↓
Pod (direct connection, no kube-proxy)
```

**NEG vs Instance Group Backends:**

**With NEGs (Container-Native):**
- Load balancer sends traffic directly to pod IPs
- Bypasses kube-proxy and iptables overhead
- Health checks directly against pods
- Better performance and observability
- Enabled via `cloud.google.com/neg: '{"ingress": true}'` annotation

**Without NEGs (Legacy):**
- Load balancer sends to node IPs (instance groups)
- kube-proxy forwards to pods via iptables/IPVS
- Additional network hop and latency
- Health checks against nodes, not pods

**Advanced Features:**

**Multi-Cluster Ingress:**
```yaml
apiVersion: networking.gke.io/v1
kind: MultiClusterIngress
metadata:
  name: global-ingress
spec:
  template:
    spec:
      backend:
        serviceName: default-backend
        servicePort: 80
```
- Single global load balancer across multiple GKE clusters
- Traffic distribution based on proximity and health
- Disaster recovery and global load distribution

**Internal Ingress:**
```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: "gce-internal"
```
- Creates Internal HTTP(S) Load Balancer
- Only accessible within VPC or via VPN/Interconnect
- No public IP exposure

**Cloud Armor Integration:**
```yaml
metadata:
  annotations:
    cloud.google.com/armor-config: '{"security-policy-name": "my-policy"}'
```
- DDoS protection and WAF rules
- Rate limiting and geo-blocking
- Applied at load balancer level

**Troubleshooting:**
```bash
# Check Ingress status
kubectl describe ingress app-ingress

# View NEG status
kubectl get svcneg -A

# Check backend health
gcloud compute backend-services get-health BACKEND_NAME --global

# View load balancer components
gcloud compute forwarding-rules list
gcloud compute url-maps list
```
9. Explain **pod lifecycle, probes, and autoscaling (HPA)** and how they impact reliability.

**Answer:**

Understanding pod lifecycle and health monitoring is fundamental to building reliable Kubernetes applications. These mechanisms work together to ensure applications start correctly, remain healthy during operation, and scale appropriately under load.

**Pod Lifecycle:**

A pod transitions through distinct phases from creation to termination. Each phase represents a specific state in the pod's existence, and understanding these transitions is critical for debugging startup issues and implementing graceful shutdown logic.

**Phases:**
1. **Pending:** Pod accepted by cluster, waiting for scheduling and image pull
2. **Running:** Pod bound to node, at least one container running
3. **Succeeded:** All containers terminated successfully (exit 0)
4. **Failed:** All containers terminated, at least one failed (non-zero exit)
5. **Unknown:** Pod state cannot be determined (node communication failure)

**Container States:**
- **Waiting:** Container not running (pulling image, waiting for init containers)
- **Running:** Container executing without issues
- **Terminated:** Container finished execution or was killed

**Init Containers:**
- Run sequentially before app containers
- Use cases: database migrations, config file generation, waiting for dependencies
```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nslookup db-service; do sleep 2; done']
```

**Probes (Health Checks):**

Kubernetes probes enable the kubelet to monitor container health and make automated decisions about traffic routing and container lifecycle. Properly configured probes are essential for zero-downtime deployments and automatic recovery from failures. Each probe type serves a distinct purpose and misconfiguration is a common cause of deployment issues.

**Liveness Probe:**
The liveness probe determines whether a container is functioning correctly. When the probe fails, kubelet terminates the container and initiates a restart based on the pod's restart policy. This mechanism handles scenarios where the application process is running but deadlocked or otherwise unable to process requests.
- Determines if container is alive and healthy
- If fails, kubelet kills container and restarts based on restart policy
- Use for detecting deadlocks or hung processes
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```
**Impact:** Prevents traffic to unhealthy pods, automatic recovery from failures

**Readiness Probe:**
- Determines if container is ready to accept traffic
- If fails, pod removed from Service endpoints (no traffic routed)
- Does NOT restart container
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  successThreshold: 1
  failureThreshold: 3
```
**Impact:** Prevents traffic during startup, deployments, or temporary unavailability

**Startup Probe:**
- For slow-starting containers (legacy apps, large initialization)
- Disables liveness/readiness checks until startup succeeds
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # 300 seconds total
```
**Impact:** Prevents premature restarts of slow-starting applications

**Probe Types:**
- **HTTP GET:** HTTP request to endpoint (200-399 = success)
- **TCP Socket:** TCP connection attempt
- **Exec:** Command execution in container (exit 0 = success)
- **gRPC:** gRPC health check protocol

**Horizontal Pod Autoscaler (HPA):**

HPA provides automatic scaling based on observed metrics. The controller runs a control loop that queries the Metrics Server (or custom metrics adapter) every 15 seconds by default, calculates the desired replica count using the formula: desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)], and updates the deployment's replica count. HPA prevents both under-provisioning during traffic spikes and over-provisioning during quiet periods.

**Mechanism:**
- Automatically scales replica count based on metrics
- Queries Metrics Server for resource utilization
- Calculates desired replicas: `ceil[currentReplicas * (currentMetric / targetMetric)]`
- Evaluates every 15 seconds (default)

**Resource-Based HPA:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

**Custom Metrics HPA:**
```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"
```

**External Metrics (Cloud Monitoring):**
```yaml
metrics:
- type: External
  external:
    metric:
      name: pubsub.googleapis.com|subscription|num_undelivered_messages
      selector:
        matchLabels:
          resource.labels.subscription_id: "my-subscription"
    target:
      type: AverageValue
      averageValue: "100"
```

**Reliability Impact:**

**Probes Best Practices:**
- Set `initialDelaySeconds` > application startup time
- `timeoutSeconds` should be < `periodSeconds`
- Liveness threshold higher than readiness (3 vs 1) to avoid flapping
- Probe endpoints should be lightweight (no heavy DB queries)
- Separate liveness (process health) from readiness (traffic readiness)

**HPA Best Practices:**
- Set `minReplicas` ≥ 2 for high availability
- Configure `scaleDown.stabilizationWindowSeconds` to prevent flapping
- Use multiple metrics (CPU + memory + custom) for better scaling decisions
- Ensure resource requests are set accurately (HPA uses requests as baseline)
- Monitor HPA events: `kubectl describe hpa app-hpa`

**Common Pitfalls:**
- Liveness probe too aggressive → constant pod restarts
- No readiness probe → traffic sent to starting pods → 502 errors
- HPA without resource requests → cannot calculate utilization
- Scaling too fast → overwhelming downstream dependencies
- Not accounting for startup time → CrashLoopBackOff
10. You see `CrashLoopBackOff` and high latency in production pods—**how do you debug this in GKE?**

**Answer:**

Debugging production issues requires a systematic approach that combines metric analysis, log investigation, and cluster introspection. Rather than random exploration, effective debugging follows a hypothesis-driven process: observe symptoms, form hypotheses, gather evidence, and iterate until the root cause is identified. The techniques below address the two most common production issues: pods failing to start (CrashLoopBackOff) and applications responding slowly.

**Phase 1: Initial Assessment**

Begin with broad observation before diving into specifics. Cluster-wide metrics reveal whether the issue is isolated to specific pods or indicates systemic problems like resource exhaustion or networking failures.

**Check Pod Status:**
```bash
kubectl get pods -n production
kubectl describe pod POD_NAME -n production
```
Look for:
- Restart count (high = recurring issue)
- Last State: Terminated (exit code, reason)
- Events section (image pull errors, scheduling failures, probe failures)

**Common Exit Codes:**
- Exit 0: Normal termination
- Exit 1: Application error
- Exit 137: SIGKILL (OOMKilled - memory limit exceeded)
- Exit 143: SIGTERM (graceful shutdown)
- Exit 255: Exit status out of range

**Phase 2: CrashLoopBackOff Debugging**

**1. Check Container Logs:**
```bash
# Current container logs
kubectl logs POD_NAME -n production -c CONTAINER_NAME

# Previous container logs (after crash)
kubectl logs POD_NAME -n production -c CONTAINER_NAME --previous

# Stream logs
kubectl logs -f POD_NAME -n production

# All pods with label
kubectl logs -n production -l app=myapp --tail=100
```

**2. Check Resource Limits:**
```bash
kubectl describe pod POD_NAME -n production | grep -A 5 "Limits\|Requests"
kubectl top pod POD_NAME -n production
```
If OOMKilled:
- Increase memory limits
- Check for memory leaks in application
- Review memory usage patterns in Cloud Monitoring

**3. Check Liveness/Readiness Probes:**
```bash
kubectl describe pod POD_NAME -n production | grep -A 10 "Liveness\|Readiness"
```
Issues:
- `initialDelaySeconds` too short → probe fails before app ready
- `timeoutSeconds` too short → slow responses marked as failures
- Probe endpoint has bugs or dependencies

**4. Check Image and Configuration:**
```bash
# Verify image exists and is pullable
kubectl describe pod POD_NAME -n production | grep Image:

# Check environment variables and secrets
kubectl describe pod POD_NAME -n production | grep -A 20 "Environment"

# Verify ConfigMaps and Secrets exist
kubectl get configmap -n production
kubectl get secret -n production
```

**5. Exec into Running Container (if possible):**
```bash
kubectl exec -it POD_NAME -n production -- /bin/sh
# Check file permissions, environment, connectivity
```

**6. Check Events:**
```bash
kubectl get events -n production --sort-by='.lastTimestamp' | grep POD_NAME
```

**Phase 3: High Latency Debugging**

**1. Check Application Metrics:**
```bash
# CPU and memory usage
kubectl top pods -n production -l app=myapp

# HPA status
kubectl get hpa -n production
kubectl describe hpa HPA_NAME -n production
```

**2. Cloud Monitoring Queries:**
```
# P99 latency
fetch k8s_container
| metric 'kubernetes.io/container/restart_count'
| filter resource.namespace_name == 'production'

# Request latency from load balancer
fetch https_lb_rule
| metric 'loadbalancing.googleapis.com/https/request_count'
| group_by [response_code_class]
```

**3. Check Network Performance:**
```bash
# Service endpoints
kubectl get endpoints SERVICE_NAME -n production

# Network policies blocking traffic
kubectl get networkpolicies -n production

# DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup SERVICE_NAME.production.svc.cluster.local
```

**4. Check Downstream Dependencies:**
```bash
# Database connection pool exhaustion
# External API rate limiting
# Cloud SQL proxy issues
kubectl logs POD_NAME -n production | grep -i "connection\|timeout\|error"
```

**5. Trace Requests (Cloud Trace):**
- Enable Cloud Trace in application
- Identify slow spans (database queries, external API calls)
- Check for N+1 queries or missing indexes

**6. Check Load Balancer Health:**
```bash
# Backend health status
gcloud compute backend-services get-health BACKEND_SERVICE_NAME --global

# Ingress status
kubectl describe ingress INGRESS_NAME -n production
```

**Phase 4: Advanced Debugging**

**1. Enable Debug Logging:**
```bash
# Update deployment with debug env var
kubectl set env deployment/myapp LOG_LEVEL=debug -n production
```

**2. Profile Application:**
```bash
# Port-forward to profiling endpoint
kubectl port-forward POD_NAME 6060:6060 -n production
# Access pprof: http://localhost:6060/debug/pprof/
```

**3. Check Node Health:**
```bash
kubectl get nodes
kubectl describe node NODE_NAME

# Node resource pressure
kubectl top nodes
```

**4. Review Recent Changes:**
```bash
# Deployment history
kubectl rollout history deployment/myapp -n production

# Rollback if needed
kubectl rollout undo deployment/myapp -n production
```

**5. Cloud Logging Queries:**
```
resource.type="k8s_container"
resource.labels.namespace_name="production"
resource.labels.pod_name=~"myapp-.*"
severity>=ERROR
```

**Common Root Causes:**

**CrashLoopBackOff:**
- Missing environment variables or secrets
- Database connection failures (wrong credentials, network issues)
- OOMKilled (memory limits too low)
- Application bugs (panic, unhandled exceptions)
- Liveness probe misconfiguration
- Missing dependencies (init containers not completing)

**High Latency:**
- Insufficient replicas (HPA not scaling fast enough)
- Resource throttling (CPU limits too low)
- Slow database queries (missing indexes, connection pool exhaustion)
- External API timeouts
- Network policies blocking traffic
- Cold start issues (JVM warmup, connection pool initialization)
- Inefficient code paths (N+1 queries, synchronous processing)

**Resolution Steps:**
1. Immediate: Rollback to last known good version
2. Short-term: Increase resources, adjust probe settings
3. Long-term: Fix application bugs, optimize queries, implement caching

###  Docker & Containers

11. Explain **Dockerfile best practices**, multi-stage builds, non-root containers, and image optimization.

**Answer:**

Dockerfiles define the build process for container images. Optimizing Dockerfiles directly impacts image size, build time, security posture, and deployment performance. The practices below address common inefficiencies and security vulnerabilities observed in production environments.

**Multi-Stage Builds:**

**Purpose:**
Multi-stage builds address the fundamental tension between build-time requirements (compilers, package managers, development tools) and runtime requirements (minimal base, reduced attack surface). By separating these concerns into distinct stages, the final image contains only what is necessary to run the application.
- Separate build-time dependencies from runtime dependencies
- Reduce final image size by excluding build tools, compilers, and intermediate artifacts
- Improve security by minimizing attack surface

**Example:**
```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Stage 2: Runtime
FROM alpine:3.19
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
EXPOSE 8080
CMD ["./main"]
```
**Benefits:**
- Build stage: 800MB (includes Go compiler, build tools)
- Runtime stage: 15MB (only binary and minimal OS)
- 98% size reduction

**Non-Root Containers:**

**Security Rationale:**
- Running as root increases risk if container is compromised
- Attacker gains root privileges on container filesystem
- Potential for container escape exploits
- Compliance requirements (PCI-DSS, SOC 2) mandate least privilege

**Implementation:**
```dockerfile
FROM python:3.11-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set working directory and ownership
WORKDIR /app
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

EXPOSE 8080
CMD ["python", "app.py"]
```

**Kubernetes Enforcement:**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
```

**Image Optimization Best Practices:**

**1. Use Minimal Base Images:**
```dockerfile
# Bad: Full OS (1.2GB)
FROM ubuntu:22.04

# Better: Slim variant (200MB)
FROM python:3.11-slim

# Best: Distroless (50MB) or Alpine (5MB)
FROM gcr.io/distroless/python3-debian11
```

**2. Layer Caching Optimization:**
```dockerfile
# Bad: Invalidates cache on any code change
COPY . .
RUN pip install -r requirements.txt

# Good: Dependencies cached separately
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

**3. Minimize Layers:**
```dockerfile
# Bad: Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean

# Good: Single layer with cleanup
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**4. Use .dockerignore:**
```
.git
.github
node_modules
*.md
.env
tests/
__pycache__
*.pyc
.DS_Store
```

**5. Avoid Installing Unnecessary Packages:**
```dockerfile
# Use --no-install-recommends to avoid suggested packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl && \
    rm -rf /var/lib/apt/lists/*
```

**6. Pin Versions:**
```dockerfile
# Bad: Unpredictable builds
FROM python:3
RUN pip install flask

# Good: Reproducible builds
FROM python:3.11.6-slim
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# requirements.txt: flask==3.0.0
```

**7. Health Checks:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

**8. Use COPY Instead of ADD:**
```dockerfile
# Bad: ADD has implicit tar extraction and URL fetching
ADD app.tar.gz /app

# Good: COPY is explicit and predictable
COPY app/ /app/
```

**9. Leverage Build Arguments:**
```dockerfile
ARG VERSION=1.0.0
LABEL version="${VERSION}"
LABEL maintainer="team@example.com"
```

**10. Distroless Images for Maximum Security:**
```dockerfile
# Multi-stage with distroless
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o app .

FROM gcr.io/distroless/static-debian11
COPY --from=builder /app/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```
**Benefits:**
- No shell, package manager, or utilities
- Minimal attack surface
- Only application and runtime dependencies

**Complete Optimized Example:**
```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-alpine AS builder

WORKDIR /app

# Install dependencies (cached layer)
COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application code
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine

# Security: Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy only necessary files
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

# Switch to non-root user
USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD node healthcheck.js || exit 1

CMD ["node", "dist/server.js"]
```

**Image Size Comparison:**
- node:20 (full): 1.1GB
- node:20-slim: 240MB
- node:20-alpine: 180MB
- Multi-stage optimized: 120MB
12. How do you **scan container images**, prevent vulnerable images from deploying, and enforce policies?

**Answer:**

Container image security operates on the principle that base images, application dependencies, and configuration can all introduce vulnerabilities. A layered scanning approach addresses each attack vector: base image vulnerabilities are detected through OS package analysis, application dependency vulnerabilities through language-specific scanners, and misconfigurations through policy enforcement. The goal is to shift security left, catching issues before they reach production.

**Container Image Scanning:**

**Vulnerability Scanning Approaches:**

Scanning should occur at multiple stages: during development (IDE integration), in CI/CD pipelines (blocking builds), in registries (continuous monitoring), and at runtime (admission control). Each stage provides different coverage and acts as a defense layer.

**1. Artifact Registry Vulnerability Scanning (GCP Native):**

**Automatic Scanning:**
- Enabled by default on Artifact Registry
- Scans images on push and continuously rescans for new vulnerabilities
- Uses vulnerability databases: CVE, Debian Security Tracker, Alpine SecDB, Ubuntu Security Notices

**Enable and Configure:**
```bash
# Enable vulnerability scanning
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us-central1 \
  --enable-vulnerability-scanning

# View scan results
gcloud artifacts docker images list us-central1-docker.pkg.dev/PROJECT_ID/my-repo

gcloud artifacts docker images describe \
  us-central1-docker.pkg.dev/PROJECT_ID/my-repo/myapp:v1.0 \
  --show-all-metadata
```

**Vulnerability Severity Levels:**
- CRITICAL: Immediate action required
- HIGH: Fix soon
- MEDIUM: Plan remediation
- LOW: Monitor
- MINIMAL: Informational

**Binary Authorization (Policy Enforcement):**

**Policy Example:**
```yaml
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
  - projects/PROJECT_ID/attestors/prod-attestor

clusterAdmissionRules:
  us-central1-a.prod-cluster:
    evaluationMode: REQUIRE_ATTESTATION
    enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
    requireAttestationsBy:
    - projects/PROJECT_ID/attestors/prod-attestor
    - projects/PROJECT_ID/attestors/security-attestor
```

**3. CI/CD Integration:**

**Cloud Build with Vulnerability Scanning:**
```yaml
steps:
# Build image
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'us-docker.pkg.dev/$PROJECT_ID/repo/app:$SHORT_SHA', '.']

# Push to Artifact Registry
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'us-docker.pkg.dev/$PROJECT_ID/repo/app:$SHORT_SHA']

# Wait for scan results
- name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    set -e
    echo "Waiting for vulnerability scan..."
    sleep 30
    
    # Get scan results
    SCAN_RESULT=$(gcloud artifacts docker images describe \
      us-docker.pkg.dev/$PROJECT_ID/repo/app:$SHORT_SHA \
      --format='value(image_summary.vulnerability_counts)' || echo "")
    
    # Parse critical and high vulnerabilities
    CRITICAL=$(echo $SCAN_RESULT | jq -r '.CRITICAL // 0')
    HIGH=$(echo $SCAN_RESULT | jq -r '.HIGH // 0')
    
    if [ "$CRITICAL" -gt 0 ] || [ "$HIGH" -gt 5 ]; then
      echo "CRITICAL: $CRITICAL, HIGH: $HIGH vulnerabilities found"
      exit 1
    fi

# Create attestation (if scan passes)
- name: 'gcr.io/cloud-builders/gcloud'
  args:
  - 'beta'
  - 'container'
  - 'binauthz'
  - 'attestations'
  - 'sign-and-create'
  - '--artifact-url=us-docker.pkg.dev/$PROJECT_ID/repo/app:$SHORT_SHA'
  - '--attestor=prod-attestor'
  - '--attestor-project=$PROJECT_ID'
```

**4. Third-Party Scanning Tools:**

**Trivy (Open Source):**
```bash
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Scan image
trivy image --severity HIGH,CRITICAL us-docker.pkg.dev/PROJECT_ID/repo/app:v1.0

# Fail build on critical vulnerabilities
trivy image --exit-code 1 --severity CRITICAL us-docker.pkg.dev/PROJECT_ID/repo/app:v1.0

# Generate SARIF report for GitHub
trivy image --format sarif --output trivy-results.sarif myimage:tag
```

**Integrate in CI:**
```yaml
# Cloud Build
- name: 'aquasec/trivy'
  args:
  - 'image'
  - '--exit-code'
  - '1'
  - '--severity'
  - 'CRITICAL,HIGH'
  - '--no-progress'
  - 'us-docker.pkg.dev/$PROJECT_ID/repo/app:$SHORT_SHA'
```

**Snyk:**
```bash
# Scan image
snyk container test us-docker.pkg.dev/PROJECT_ID/repo/app:v1.0 \
  --severity-threshold=high

# Monitor image for new vulnerabilities
snyk container monitor us-docker.pkg.dev/PROJECT_ID/repo/app:v1.0
```

**5. Policy Enforcement Strategies:**

**OPA (Open Policy Agent) Gatekeeper:**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockImages
metadata:
  name: block-unscanned-images
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - production
  parameters:
    allowedRegistries:
    - us-docker.pkg.dev/PROJECT_ID/
    requireAttestation: true
```

**Admission Webhook:**
```go
// Custom admission controller
func validateImage(image string) error {
    // Check if image is from allowed registry
    if !strings.HasPrefix(image, "us-docker.pkg.dev/PROJECT_ID/") {
        return fmt.Errorf("image not from approved registry")
    }
    
    // Query Artifact Registry for vulnerability scan
    vulns := getVulnerabilities(image)
    if vulns.Critical > 0 || vulns.High > 10 {
        return fmt.Errorf("image has %d critical and %d high vulnerabilities", 
            vulns.Critical, vulns.High)
    }
    
    return nil
}
```

**6. Runtime Security:**

**Falco (Runtime Threat Detection):**
```yaml
# Detect suspicious container behavior
- rule: Unexpected outbound connection
  desc: Detect unexpected outbound connections from container
  condition: >
    outbound and container and not allowed_outbound_destinations
  output: >
    Unexpected outbound connection (user=%user.name command=%proc.cmdline 
    connection=%fd.name container=%container.name)
  priority: WARNING
```

**GKE Security Posture Dashboard:**
- Provides visibility into workload vulnerabilities
- Identifies misconfigurations (privileged containers, missing resource limits)
- Recommends security improvements

**Best Practices:**

**1. Shift Left Security:**
- Scan images in CI pipeline before deployment
- Fail builds on critical vulnerabilities
- Provide developers with scan results early

**2. Continuous Monitoring:**
- Rescan images regularly for newly discovered CVEs
- Set up alerts for new critical vulnerabilities in deployed images
- Automate patching and redeployment

**3. Base Image Management:**
- Maintain curated set of approved base images
- Regularly update base images with security patches
- Use minimal base images (Alpine, Distroless) to reduce attack surface

**4. Vulnerability Remediation Workflow:**
```
1. Scan detects vulnerability
2. Create Jira ticket with CVE details
3. Update base image or dependency
4. Rebuild and rescan image
5. Deploy patched image
6. Verify vulnerability resolved
```

**5. Compliance and Audit:**
- Enable audit logging for Binary Authorization decisions
- Track which images are deployed and their attestation status
- Generate compliance reports showing vulnerability trends
- Implement SLAs for patching critical vulnerabilities (e.g., 7 days)

---

###  CI/CD (Jenkins / GitHub Actions)

13. Design a **CI/CD pipeline for a GKE-based SaaS application** with rollback and zero-downtime deployments.

**Answer:**

**Pipeline Architecture:**

**GitHub Actions Workflow:**
```yaml
jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    # 1. Build & Push
    - name: Build & Push
      run: |
        docker build -t $IMAGE:$GITHUB_SHA .
        docker push $IMAGE:$GITHUB_SHA
    
    # 2. Deploy to GKE
    - name: Deploy
      run: |
        kubectl set image deployment/$DEPLOYMENT_NAME \
          $DEPLOYMENT_NAME=$IMAGE:$GITHUB_SHA
        kubectl rollout status deployment/$DEPLOYMENT_NAME
```

**Zero-Downtime Deployment Configuration:**

**Deployment Manifest:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Allow 1 extra pod during update
      maxUnavailable: 0  # Never reduce available pods
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: "{{ .Values.version }}"
    spec:
      containers:
      - name: myapp
        image: us-docker.pkg.dev/my-project/repo/myapp:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # Grace period for connection draining
      terminationGracePeriodSeconds: 30
```

**Key Zero-Downtime Settings:**
- `maxUnavailable: 0`: Ensures minimum replica count maintained
- `maxSurge: 1`: Creates new pod before terminating old one
- `readinessProbe`: New pods only receive traffic when ready
- `preStop hook`: Allows graceful connection draining
- `terminationGracePeriodSeconds`: Time for pod to finish requests

**Rollback Strategy:**

**Automatic Rollback Script:**
```bash
if ! kubectl rollout status deployment/$DEPLOYMENT_NAME --timeout=5m; then
  echo "Deployment failed, rolling back"
  kubectl rollout undo deployment/$DEPLOYMENT_NAME
  exit 1
fi
```

**Manual Rollback:**
```bash
# View deployment history
kubectl rollout history deployment/myapp

# Rollback to previous version
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=3

# Check rollback status
kubectl rollout status deployment/myapp
```

**Advanced: Progressive Delivery with Flagger**

**Canary Deployment with Automatic Rollback:**
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
    targetPort: 8080
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
    webhooks:
    - name: load-test
      url: http://flagger-loadtester/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://myapp-canary/"
```

**Rollback triggers:**
- Request success rate < 99%
- Request duration > 500ms
- Failed health checks
- Manual intervention

**Database Migration Handling:**

**Pre-Deployment Job:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-{{ .Values.version }}
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: us-docker.pkg.dev/my-project/repo/myapp:{{ .Values.version }}
        command: ["npm", "run", "migrate"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
      restartPolicy: Never
  backoffLimit: 3
```

**Pipeline with Migration:**
```yaml
    - name: Run Database Migration
      run: |
        kubectl apply -f k8s/migration-job.yaml
        kubectl wait --for=condition=complete --timeout=5m job/db-migration-$GITHUB_SHA
    
    - name: Deploy Application
      run: |
        kubectl set image deployment/$DEPLOYMENT_NAME \
          $DEPLOYMENT_NAME=$IMAGE:$GITHUB_SHA
```

**Monitoring & Notifications:**

**Slack Notification:**
```yaml
    - name: Notify Slack
      if: always()
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: |
          Deployment ${{ job.status }}
          Version: ${{ github.sha }}
          Environment: Production
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

**Best Practices:**
- Use immutable image tags (SHA, not `latest`)
- Implement comprehensive health checks
- Set appropriate resource requests/limits
- Use PodDisruptionBudgets to prevent disruption
- Monitor deployment metrics (error rates, latency)
- Implement automated rollback on metric degradation
- Maintain deployment history for audit trail
- Test rollback procedures regularly
14. How do you **secure CI/CD pipelines** when deploying to GCP (credentials, secrets, approvals)?

**Answer:**

Securing CI/CD pipelines requires addressing multiple attack vectors: credential theft, unauthorized access, pipeline manipulation, and supply chain attacks. The strategies below implement defense in depth, ensuring that compromise of any single component does not grant full system access. The focus is on eliminating long-lived credentials, enforcing least privilege, and maintaining comprehensive audit trails.

**1. Workload Identity Federation (No Service Account Keys)**

Workload Identity Federation enables external identity providers (GitHub, GitLab, CircleCI) to authenticate to GCP without storing service account keys. The mechanism uses OIDC token exchange: the CI system provides a signed JWT token asserting its identity, and GCP exchanges this for a short-lived access token scoped to specific permissions.

**GitHub Actions Integration:**
```bash
# Create Workload Identity Pool
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --display-name="GitHub Actions Pool"

# Create Provider
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository_owner=='myorg'"

# Grant permissions to service account
gcloud iam service-accounts add-iam-policy-binding github-actions@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/myorg/myrepo"
```

**GitHub Actions Workflow:**
```yaml
jobs:
  deploy:
    permissions:
      contents: read
      id-token: write  # Required for OIDC
    steps:
    - uses: google-github-actions/auth@v1
      with:
        workload_identity_provider: 'projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
        service_account: 'github-actions@PROJECT_ID.iam.gserviceaccount.com'
        token_format: 'access_token'
```

**Benefits:**
- No long-lived service account keys
- Automatic token rotation
- Scoped to specific repositories
- Audit trail via Cloud Logging

**2. Secret Management**

**GitHub Secrets with Secret Manager:**
```yaml
steps:
- uses: google-github-actions/auth@v1
  with:
    workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
    service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}

- name: Access Secrets
  run: |
    DB_PASSWORD=$(gcloud secrets versions access latest --secret=db-password)
    API_KEY=$(gcloud secrets versions access latest --secret=api-key)
    
    # Use secrets in deployment
    kubectl create secret generic app-secrets \
      --from-literal=db-password=$DB_PASSWORD \
      --from-literal=api-key=$API_KEY \
      --dry-run=client -o yaml | kubectl apply -f -
```

**Cloud Build with Secret Manager:**
```yaml
availableSecrets:
  secretManager:
  - versionName: projects/PROJECT_ID/secrets/db-password/versions/latest
    env: 'DB_PASSWORD'
  - versionName: projects/PROJECT_ID/secrets/api-key/versions/latest
    env: 'API_KEY'

steps:
- name: 'gcr.io/cloud-builders/kubectl'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    kubectl create secret generic app-secrets \
      --from-literal=db-password=$$DB_PASSWORD \
      --from-literal=api-key=$$API_KEY \
      --dry-run=client -o yaml | kubectl apply -f -
  secretEnv: ['DB_PASSWORD', 'API_KEY']
```

**3. Least Privilege IAM**

**Service Account Permissions:**
```bash
# Create dedicated CI/CD service account
gcloud iam service-accounts create cicd-deployer \
  --display-name="CI/CD Deployer"

# Grant minimal required permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:cicd-deployer@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/container.developer"  # GKE deployment only

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:cicd-deployer@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"  # Read artifacts

# Secret Manager access (specific secrets only)
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:cicd-deployer@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

**Custom Role for CI/CD:**
```yaml
title: "CI/CD Deployer"
description: "Minimal permissions for CI/CD pipeline"
stage: "GA"
includedPermissions:
- container.clusters.get
- container.clusters.getCredentials
- container.deployments.create
- container.deployments.update
- container.deployments.get
- container.pods.get
- container.pods.list
- storage.objects.get
- storage.objects.list
- secretmanager.versions.access
```

**4. Approval Gates**

**Manual Approval for Production:**
```yaml
# GitHub Actions Environment Protection
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to Staging
      run: ./deploy.sh staging
  
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
    - name: Deploy to Production
      run: ./deploy.sh production
```

**GitHub Environment Settings:**
- Required reviewers: 2 approvers from DevOps team
- Wait timer: 5 minutes before deployment
- Deployment branches: only `main` branch

**Cloud Build Approval:**
```yaml
steps:
- name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    # Request approval via Pub/Sub
    gcloud pubsub topics publish deployment-approval \
      --message='{\"build_id\": \"$BUILD_ID\", \"env\": \"production\"}'\n    \n    # Wait for approval (via Cloud Function updating build metadata)\n    while true; do\n      STATUS=$(gcloud builds describe $BUILD_ID --format='value(substitutions.APPROVAL_STATUS)')\n      if [ \"$STATUS\" = \"approved\" ]; then\n        break\n      elif [ \"$STATUS\" = \"rejected\" ]; then\n        exit 1\n      fi\n      sleep 30\n    done\n\n- name: 'gcr.io/cloud-builders/kubectl'\n  args: ['apply', '-f', 'k8s/production/']\n```

**5. Secure Build Environment**

**Isolated Build Environments:**
```yaml\n# Cloud Build with private pool\nsteps:\n- name: 'gcr.io/cloud-builders/docker'\n  args: ['build', '-t', 'gcr.io/$PROJECT_ID/app', '.']\n\noptions:\n  pool:\n    name: 'projects/PROJECT_ID/locations/us-central1/workerPools/secure-pool'\n  logging: CLOUD_LOGGING_ONLY\n  machineType: 'N1_HIGHCPU_8'\n```

**Private Pool Configuration:**\n- VPC network with no external IP\n- Access to Artifact Registry via Private Google Access\n- Network policies restricting outbound connections\n- Dedicated service account per pool

**6. Audit Logging**

**Enable Comprehensive Logging:**\n```bash\n# Enable Data Access audit logs for Secret Manager\ngcloud projects get-iam-policy PROJECT_ID > policy.yaml\n\n# Add audit config\ncat >> policy.yaml <<EOF\nauditConfigs:\n- service: secretmanager.googleapis.com\n  auditLogConfigs:\n  - logType: DATA_READ\n  - logType: DATA_WRITE\n  - logType: ADMIN_READ\nEOF\n\ngcloud projects set-iam-policy PROJECT_ID policy.yaml\n```

**Monitor CI/CD Activities:**\n```\n# Cloud Logging query for deployments\nresource.type=\"k8s_cluster\"\nprotoPayload.methodName=\"io.k8s.apps.v1.deployments.update\"\nprotoPayload.authenticationInfo.principalEmail=\"cicd-deployer@PROJECT_ID.iam.gserviceaccount.com\"\n\n# Secret access audit\nresource.type=\"secretmanager.googleapis.com/Secret\"\nprotoPayload.methodName=\"google.cloud.secretmanager.v1.SecretManagerService.AccessSecretVersion\"\n```

**7. Code Signing and Verification**

**Binary Authorization:**\n```bash\n# Sign images in CI/CD\ngcloud beta container binauthz attestations sign-and-create \\\n  --artifact-url=\"us-docker.pkg.dev/PROJECT_ID/repo/app:$TAG\" \\\n  --attestor=\"cicd-attestor\" \\\n  --attestor-project=\"PROJECT_ID\" \\\n  --keyversion=\"projects/PROJECT_ID/locations/global/keyRings/binauthz/cryptoKeys/attestor-key/cryptoKeyVersions/1\"\n```

**Policy Enforcement:**\n```yaml\nadmissionWhitelistPatterns:\n- namePattern: gcr.io/google-containers/*\n\ndefaultAdmissionRule:\n  requireAttestationsBy:\n  - projects/PROJECT_ID/attestors/cicd-attestor\n  - projects/PROJECT_ID/attestors/security-scan-attestor\n  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG\n```

**8. Network Security**

**VPC Service Controls:**\n```bash\n# Create service perimeter\ngcloud access-context-manager perimeters create cicd-perimeter \\\n  --title=\"CI/CD Perimeter\" \\\n  --resources=\"projects/PROJECT_NUMBER\" \\\n  --restricted-services=\"storage.googleapis.com,artifactregistry.googleapis.com,secretmanager.googleapis.com\" \\\n  --policy=\"POLICY_ID\"\n\n# Allow access only from authorized networks\ngcloud access-context-manager perimeters update cicd-perimeter \\\n  --add-access-levels=\"corporate_network,cicd_runners\"\n```

**9. Dependency Scanning**

**Scan Dependencies in Pipeline:**
```yaml
steps:
- name: Dependency Check
  run: |
    npm audit --audit-level=high
    
- name: SAST Scanning
  uses: github/codeql-action/analyze@v2

- name: Container Scanning
  run: |
    trivy image --severity CRITICAL,HIGH --exit-code 1 $IMAGE
```

**10. Best Practices Summary**

**Credentials:**
- Use Workload Identity Federation (no service account keys)
- Rotate secrets regularly via Secret Manager
- Use short-lived tokens (1 hour max)
- Never commit credentials to version control
- Implement secret scanning in pre-commit hooks

**Access Control:**
- Principle of least privilege for service accounts
- Separate service accounts per environment
- Use custom IAM roles instead of primitive roles
- Implement IAM Conditions for time/IP-based access
- Regular access reviews and permission audits

**Approvals:**
- Require manual approval for production deployments
- Implement 4-eyes principle (2+ approvers)
- Use deployment windows for production changes
- Maintain audit trail of all approvals
- Automated rollback on failed health checks

**Monitoring:**
- Enable audit logging for all CI/CD activities
- Alert on unauthorized access attempts
- Monitor for anomalous deployment patterns
- Track deployment frequency and success rates
- Implement security scanning at every stage
15. Explain **blue-green vs canary deployments** and how you'd implement one on GKE.

**Answer:**

Blue-green and canary deployments represent different risk management strategies for releasing new code. Blue-green provides atomic cutover with instant rollback capability, while canary enables gradual exposure with metric-based progression. The choice depends on application characteristics, team capabilities, and risk tolerance. Neither strategy is universally superior; each suits different scenarios.

**Blue-Green Deployment:**

**Concept:**
Blue-green deployment maintains two identical production environments at all times. The active environment serves all traffic while the inactive environment receives the new deployment. After validation, a load balancer configuration change (typically DNS or service mesh routing) switches all traffic to the new environment. Rollback requires only reversing the routing change.
- Two identical production environments: Blue (current) and Green (new)
- Traffic switches entirely from Blue to Green after validation
- Instant rollback by switching traffic back to Blue
- Zero downtime during switch

**Architecture:**
```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     ┌────────▼────────┐         ┌─────────▼────────┐
     │  Blue (v1.0)    │         │  Green (v2.0)    │
     │  3 replicas     │         │  3 replicas      │
     │  ACTIVE         │         │  STANDBY         │
     └─────────────────┘         └──────────────────┘
```

**GKE Implementation with Service Selector:**

**Blue Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: us-docker.pkg.dev/project/repo/myapp:v1.0
```

**Green Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: us-docker.pkg.dev/project/repo/myapp:v2.0
```

**Service (Traffic Switch):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Change to 'green' to switch traffic
  ports:
  - port: 80
    targetPort: 8080
```

**Switch Traffic:**
```bash
# Validate green deployment
kubectl get pods -l app=myapp,version=green
curl http://myapp-green-test-service/health

# Switch production traffic to green
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback to blue if issues
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

**Canary Deployment:**

**Concept:**
- Gradually route percentage of traffic to new version
- Monitor metrics during rollout
- Increase traffic incrementally (5% -> 25% -> 50% -> 100%)
- Automatic rollback on metric degradation

**Architecture:**
```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │ 90%                    10%  │
     ┌────────▼────────┐         ┌─────────▼────────┐
     │  Stable (v1.0)  │         │  Canary (v2.0)   │
     │  9 replicas     │         │  1 replica       │
     └─────────────────┘         └──────────────────┘
```

**GKE Implementation with Istio:**

**VirtualService for Traffic Split:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: myapp-canary
        port:
          number: 80
  - route:
    - destination:
        host: myapp-stable
        port:
          number: 80
      weight: 90
    - destination:
        host: myapp-canary
        port:
          number: 80
      weight: 10
```

**Progressive Rollout with Flagger:**
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
    targetPort: 8080
  analysis:
    interval: 1m
    threshold: 5         # Max failed checks before rollback
    maxWeight: 50        # Max canary traffic percentage
    stepWeight: 10       # Traffic increment per step
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
    webhooks:
    - name: smoke-test
      type: pre-rollout
      url: http://flagger-loadtester/
      timeout: 60s
      metadata:
        cmd: "curl -s http://myapp-canary:8080/health"
```

**Comparison Table:**

| Feature | Blue-Green | Canary |
|---------|-----------|--------|
| Traffic switch | Instant (100%) | Gradual (%) |
| Rollback speed | Instant | Gradual or instant |
| Resource usage | 2x (both versions running) | 1.1x (small canary) |
| Risk exposure | All users at once | Small subset first |
| Validation | Pre-switch testing | Production metrics |
| Complexity | Simple | Requires traffic management |
| Best for | Databases, breaking changes | Stateless services, A/B testing |

**When to Use Each:**

**Blue-Green:**
- Database schema changes requiring atomic switch
- Major version upgrades with breaking changes
- Regulatory requirements for full environment validation
- Simple infrastructure without service mesh
- Quick rollback is critical

**Canary:**
- Frequent deployments of stateless services
- Feature flags and A/B testing
- Metrics-driven release validation
- Risk mitigation for large user bases
- Gradual rollout with automatic rollback

---

### ️ Infrastructure as Code (Terraform)

16. How do you structure **Terraform for GCP** (projects, environments, modules, state management)?

**Answer:**

Terraform project structure significantly impacts maintainability, team collaboration, and blast radius of changes. A well-designed structure separates concerns across environments, enables code reuse through modules, and implements proper state isolation. The patterns below represent production-tested approaches for GCP infrastructure management.

**Directory Structure:**
```
terraform/
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/       # main.tf, variables.tf, backend.tf
├── modules/
│   ├── gke-cluster/
│   ├── vpc/
│   └── cloud-sql/
└── global/         # Shared VPC, IAM
```

**Environment Configuration:**

**environments/prod/main.tf:**
```hcl
module "vpc" {
  source       = "../../modules/vpc"
  network_name = "prod-vpc"
}

module "gke" {
  source       = "../../modules/gke-cluster"
  cluster_name = "prod-cluster"
  network      = module.vpc.network_name
  node_count   = 3
}
```

**environments/prod/backend.tf:**
```hcl
terraform {
  backend "gcs" {
    bucket = "mycompany-terraform-state"
    prefix = "prod"
  }
}
```

**Module Design:**

**modules/gke-cluster/main.tf:**
```hcl
resource "google_container_cluster" "primary" {
  name     = var.cluster_name
  location = var.region
  
  remove_default_node_pool = true
  initial_node_count       = 1
  
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
}

resource "google_container_node_pool" "primary" {
  cluster    = google_container_cluster.primary.name
  node_count = var.node_count
  
  node_config {
    machine_type = var.machine_type
    workload_metadata_config { mode = "GKE_METADATA" }
  }
}
```

**State Management:**

**Remote Backend (GCS):**
```hcl
terraform {
  backend "gcs" {
    bucket = "company-terraform-state"
    prefix = "environments/prod"
    
    # Enable state locking
    # GCS uses object versioning for locking
  }
}
```

**State Bucket Setup:**
```bash
# Create state bucket with versioning
gsutil mb -l us-central1 gs://company-terraform-state

# Enable versioning for state recovery
gsutil versioning set on gs://company-terraform-state

# Set lifecycle policy for old versions
cat > lifecycle.json << EOF
{
  "rule": [
    {
      "action": {"type": "Delete"},
      "condition": {"numNewerVersions": 10}
    }
  ]
}
EOF
gsutil lifecycle set lifecycle.json gs://company-terraform-state

# Restrict access
gsutil iam ch -d allUsers gs://company-terraform-state
```

**State Isolation per Environment:**
```
gs://company-terraform-state/
├── dev/
│   └── default.tfstate
├── staging/
│   └── default.tfstate
├── prod/
│   └── default.tfstate
└── global/
    └── default.tfstate
```

**Best Practices:**

**1. Module Versioning:**
```hcl
module "gke" {
  source  = "git::https://github.com/company/terraform-modules.git//gke-cluster?ref=v1.2.0"
}

# Or using Terraform Registry
module "gke" {
  source  = "terraform-google-modules/kubernetes-engine/google"
  version = "~> 27.0"
}
```

**2. Environment Variables:**
```hcl
# terraform.tfvars (per environment)
project_id   = "mycompany-prod"
region       = "us-central1"
environment  = "prod"
node_count   = 3
machine_type = "e2-standard-4"
```

**3. Resource Naming:**
```hcl
locals {
  resource_prefix = "${var.project_id}-${var.environment}"
}

resource "google_compute_network" "vpc" {
  name = "${local.resource_prefix}-vpc"
}
```
17. How do you manage **Terraform state safely in teams** and prevent accidental resource deletion?

**Answer:**

**State Locking:**

**GCS Backend with Locking:**
```hcl
terraform {
  backend "gcs" {
    bucket = "company-terraform-state"
    prefix = "prod"
  }
}
```
- GCS provides automatic state locking via object locking
- Prevents concurrent modifications
- Lock released automatically on completion or timeout

**Preventing Accidental Deletion:**
```hcl
resource "google_container_cluster" "primary" {
  lifecycle {
    prevent_destroy = true
  }
}

resource "google_sql_database_instance" "main" {
  deletion_protection = true
}
```

**3. State Bucket Policies:**
```bash
# Object versioning for recovery
gsutil versioning set on gs://terraform-state

# Object lock for immutability
gsutil retention set 7d gs://terraform-state

# IAM restrictions
gcloud storage buckets add-iam-policy-binding gs://terraform-state \
  --member="group:terraform-admins@company.com" \
  --role="roles/storage.objectAdmin"
```

**4. Plan Review Process:**
```bash
# Always run plan before apply
terraform plan -out=plan.tfplan

# Review plan output
terraform show plan.tfplan

# Apply only approved plan
terraform apply plan.tfplan
```

**5. CI/CD Safeguards:**
```yaml
# GitHub Actions with approval
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
    - name: Terraform Plan
      run: terraform plan -out=plan.tfplan
    - uses: actions/upload-artifact@v3
      with:
        name: plan
        path: plan.tfplan

  apply:
    needs: plan
    environment: production  # Requires approval
    steps:
    - uses: actions/download-artifact@v3
    - name: Apply
      run: terraform apply plan.tfplan
```

**6. Moved Blocks (Terraform 1.1+):**
```hcl
moved {
  from = google_container_cluster.old_name
  to   = google_container_cluster.new_name
}
```

**7. Import Existing Resources:**
```bash
terraform import google_container_cluster.primary \
  projects/my-project/locations/us-central1/clusters/prod-cluster
```

**8. State Backup and Recovery:**
```bash
# Pull state for backup
terraform state pull > backup.tfstate

# Recover from backup
terraform state push backup.tfstate

# List state contents
terraform state list

# Remove resource from state (without destroying)
terraform state rm google_container_cluster.temp
```

**Team Workflow Best Practices:**
- Require PR reviews for all Terraform changes
- Use separate state files per environment
- Implement drift detection in CI/CD
- Maintain change log of infrastructure modifications
- Regular state audit and cleanup

---

###  Monitoring, Reliability & Cost

18. How do you implement **monitoring, logging, alerting, and SLOs** for a production GKE platform?

**Answer:**

Production monitoring requires a layered approach combining metrics collection, centralized logging, intelligent alerting, and SLO-based reliability measurement. The goal is not merely to detect failures but to provide actionable insights that enable rapid response and continuous improvement. The stack described below integrates native GCP services with Kubernetes primitives.

**Monitoring Stack:**

Cloud Monitoring (formerly Stackdriver) automatically collects metrics from GKE clusters without additional configuration. These metrics cover container resource usage, pod lifecycle events, and cluster health. However, application-specific metrics require instrumentation using Prometheus client libraries or OpenTelemetry SDKs.

**Cloud Monitoring (Stackdriver):**
```yaml
# GKE metrics automatically collected:
- container/cpu/usage_time
- container/memory/used_bytes
- container/restart_count
- container/uptime
- kubernetes.io/pod/volume/used_bytes
```

**Custom Metrics (Prometheus):**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
```

**Logging:**

**Cloud Logging Configuration:**
```yaml
# GKE clusters automatically send logs to Cloud Logging
# Configure log exclusions for cost optimization
resource "google_logging_project_exclusion" "debug_logs" {
  name        = "exclude-debug"
  description = "Exclude debug logs"
  filter      = "severity=DEBUG"
}
```

**Structured Logging:**
```json
{
  "severity": "ERROR",
  "message": "Request failed",
  "httpRequest": {
    "requestMethod": "POST",
    "status": 500,
    "latency": "0.234s"
  },
  "labels": {
    "app": "myapp",
    "version": "v1.2.0"
  }
}
```

**Alerting:**

**Alert Policy (Terraform):**
```hcl
resource "google_monitoring_alert_policy" "high_error_rate" {
  display_name = "High Error Rate"
  combiner     = "OR"
  conditions {
    display_name = "Error Rate > 1%"
    condition_threshold {
      filter     = "resource.type=\"k8s_container\" AND metric.type=\"logging.googleapis.com/user/error_count\""
      duration   = "60s"
      comparison = "COMPARISON_GT"
      threshold_value = 100
    }
  }
}
```

**SLO Configuration:**

**Availability SLO (Terraform):**
```hcl
resource "google_monitoring_slo" "availability" {
  service      = "myapp-service"
  display_name = "99.9% Availability"
  goal         = 0.999
  rolling_period_days = 30
  request_based_sli {
    good_total_ratio {
      good_service_filter  = "metric.labels.response_code_class=\"200\""
      total_service_filter = "resource.type=\"https_lb_rule\""
    }
  }
}
```

**Latency SLO:**
```yaml
resource "google_monitoring_slo" "latency" {
  service      = google_monitoring_service.myapp.service_id
  slo_id       = "latency-slo"
  display_name = "P99 Latency < 500ms"
  goal         = 0.99
  
  request_based_sli {
    distribution_cut {
      distribution_filter = "metric.type=\"loadbalancing.googleapis.com/https/total_latencies\""
      range {
        max = 500  # milliseconds
      }
    }
  }
}
```

**SLO Burn Rate Alert:**
```yaml
resource "google_monitoring_alert_policy" "slo_burn" {
  display_name = "SLO Burn Rate Alert"
  
  conditions {
    display_name = "Fast Burn (2%/hour)"
    condition_threshold {
      filter = "select_slo_burn_rate(\"projects/PROJECT/services/myapp/serviceLevelObjectives/availability-slo\", \"1h\")"
      threshold_value = 14.4
      duration = "0s"
    }
  }
}
```

**Dashboard:**
```yaml
resource "google_monitoring_dashboard" "gke_overview" {
  dashboard_json = jsonencode({
    displayName = "GKE Platform Overview"
    gridLayout = {
      widgets = [
        {
          title = "Request Rate"
          xyChart = {
            dataSets = [{
              timeSeriesQuery = {
                timeSeriesFilter = {
                  filter = "metric.type=\"loadbalancing.googleapis.com/https/request_count\""
                }
              }
            }]
          }
        },
        {
          title = "Error Rate"
          xyChart = {...}
        },
        {
          title = "P99 Latency"
          xyChart = {...}
        }
      ]
    }
  })
}
```

**Key Metrics to Monitor:**
- Request rate, error rate, latency (RED method)
- CPU/memory utilization
- Pod restart count
- Replica availability
- SLO burn rate
- Error budget remaining
19. Cloud cost suddenly spikes. How do you investigate and optimize costs on GCP?

**Answer:**

Cloud cost investigation requires correlating billing data with resource usage patterns to identify the specific resources driving increased spend. The process combines billing export analysis, resource inventory review, and metric correlation. Once the cost drivers are identified, optimization follows established patterns: right-sizing, commitment-based discounts, and architectural improvements.

**Investigation Phase:**

Start with the billing export in BigQuery, which provides granular cost attribution by service, SKU, project, and time period. The queries below identify rapid cost increases and their sources. Note that billing data has a delay of several hours, so real-time monitoring requires combining billing analysis with resource metrics.

**1. Identify Spikes using BigQuery:**
```sql
SELECT
  project.id AS project_id,
  service.description AS service,
  SUM(cost) AS total_cost
FROM `billing_export`
WHERE _PARTITIONTIME >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY 1, 2 ORDER BY 3 DESC
```

**2. Identify Top Cost Drivers:**
```sql
# Cost by project and service
SELECT
  project.id AS project_id,
  service.description AS service,
  SUM(cost) AS cost_today,
  LAG(SUM(cost)) OVER (PARTITION BY project.id, service.description ORDER BY DATE(_PARTITIONTIME)) AS cost_yesterday
FROM billing_export
GROUP BY project_id, service, DATE(_PARTITIONTIME)
HAVING cost_today > cost_yesterday * 1.5  -- 50% spike
```

**3. Resource-Level Investigation:**
```bash
# Find oversized VMs
gcloud compute instances list --format="table(name,zone,machineType,scheduling.preemptible)" \
  --filter="status=RUNNING"

# Check GKE node utilization
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**Common Spike Causes:**
- Undeleted resources (test VMs, old clusters)
- Egress traffic spikes
- Oversized machine types
- Runaway autoscaling
- Storage growth (snapshots, logs)
- Expensive queries (BigQuery on-demand)

**Optimization Strategies:**

**1. Compute Optimization:**
```hcl
# Right-size VMs based on recommendations
resource "google_compute_instance" "app" {
  machine_type = "e2-medium"  # vs n1-standard-4
  
  scheduling {
    preemptible         = true  # 60-91% discount
    automatic_restart   = false
    on_host_maintenance = "TERMINATE"
  }
}
```

**2. Committed Use Discounts:**
```bash
# 1-year: 37% discount, 3-year: 55% discount
gcloud compute commitments create commitment-1 \
  --region=us-central1 \
  --resources=vcpu=100,memory=400GB \
  --plan=36-month
```

**3. GKE Optimization:**
```yaml
# Enable cluster autoscaler
resource "google_container_node_pool" "primary" {
  autoscaling {
    min_node_count = 1
    max_node_count = 10
  }
}

# Use spot VMs for non-critical workloads
node_config {
  spot = true
}
```

**4. Storage Optimization:**
```bash
# Lifecycle policies for Cloud Storage
gsutil lifecycle set lifecycle.json gs://bucket-name

# lifecycle.json
{
  "rule": [
    {"action": {"type": "SetStorageClass", "storageClass": "NEARLINE"}, "condition": {"age": 30}},
    {"action": {"type": "SetStorageClass", "storageClass": "COLDLINE"}, "condition": {"age": 90}},
    {"action": {"type": "Delete"}, "condition": {"age": 365}}
  ]
}
```

**5. Network Optimization:**
- Use Cloud CDN for static content
- Enable Private Google Access (avoid egress)
- Co-locate resources in same region
- Use internal load balancers where possible

**6. Logging Cost Reduction:**
```hcl
resource "google_logging_project_exclusion" "exclude_debug" {
  name   = "exclude-debug-logs"
  filter = "severity < WARNING"
}
```

**7. Budget Alerts:**
```hcl
resource "google_billing_budget" "budget" {
  billing_account = var.billing_account
  display_name    = "Monthly Budget"
  amount {
    specified_amount {
      currency_code = "USD"
      units         = "10000"
    }
  }
  threshold_rules {
    threshold_percent = 0.5
  }
  threshold_rules {
    threshold_percent = 0.9
  }
  all_updates_rule {
    pubsub_topic = google_pubsub_topic.budget_alerts.id
  }
}
```

**Quick Wins:**
- Delete unused resources (VMs, disks, IPs)
- Switch to regional storage from multi-regional
- Use sustained use discounts (automatic)
- Resize oversized instances
- Enable autoscaling with aggressive scale-down

---

###  Security, Linux & Real-World Scenarios

20. Production goes down after a deployment. Walk through your incident response, debugging steps, rollback, and prevention strategy.

**Answer:**

Effective incident response follows a structured process that prioritizes service restoration over root cause analysis. The first priority is always reducing customer impact, typically through rollback, followed by stabilization and only then investigation. The phases below represent a battle-tested framework for handling production incidents with minimal mean time to recovery (MTTR).

**Incident Response Flow:**

1. **Acknowledge Alert:** PagerDuty/Slack.
2. **Initial Triage:** Check `kubectl get events` and `kubectl top pods`.
3. **Diagnosis:** Check recent deployments via `kubectl rollout history`.
4. **Rollback:** `kubectl rollout undo deployment/myapp`.
5. **Post-Mortem:** Identify root cause (e.g., config error, OOM) and implement prevention (Canary, Tests).

**Post-Incident Review:**
1. Timeline of events
2. Root cause analysis (5 Whys)
3. Impact assessment
4. Action items with owners and deadlines
5. Process improvements
6. Update runbooks and playbooks

---
