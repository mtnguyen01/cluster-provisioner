# OpenClaw Agent Platform — Kubernetes Deployment

**Goal:** Run the Agent Management Platform and all OpenClaw agents on Kubernetes.
**Date:** 2026-05-13

---

## 1. Why Kubernetes?

| Feature | Benefit for Agents |
|---------|-------------------|
| **Pod lifecycle** | Auto-restart crashed agents |
| **Health checks** | Liveness/readiness probes |
| **ConfigMaps/Secrets** | Git-managed agent configs |
| **CRDs** | Custom resources for agents |
| **HPA** | Auto-scale agents by queue depth |
| **RBAC** | Scoped permissions per agent |
| **Network policies** | Isolate agents |
| **Resource quotas** | Limit CPU/memory per agent |
| **Rolling updates** | Zero-downtime config changes |
| **Observability** | Prometheus metrics, log aggregation |

---

## 2. Architecture on Kubernetes

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                          │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     Ingress (NGINX/Traefik)                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌───────────────────────────┼───────────────────────────┐         │
│  │                           │                           │         │
│  ▼                           ▼                           ▼         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐         │
│  │   Gateway    │    │  Dashboard   │    │   Metrics    │         │
│  │  (Deployment)│    │   (React)    │    │ (Prometheus) │         │
│  └──────┬───────┘    └──────────────┘    └──────────────┘         │
│         │                                                           │
│         │ Reconciles Agent CRDs                                     │
│         │                                                           │
│  ┌──────┴───────────────────────────────────────────────────────┐  │
│  │                    Agent Controller (Operator)                │  │
│  │  - Watches Agent CRDs                                       │  │
│  │  - Creates/updates Pod Deployments                          │  │
│  │  - Manages ConfigMaps/Secrets                               │  │
│  │  - Handles health checks                                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│         ┌────────────────────┼────────────────────┐                │
│         │                    │                    │                │
│  ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐         │
│  │ ops-agent   │     │ mail-agent  │     │calendar-agent│        │
│  │ (Deployment)│     │ (Deployment)│     │ (Deployment)│         │
│  └─────────────┘     └─────────────┘     └─────────────┘         │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Shared Services (StatefulSets)                 │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │ Postgres │  │ RabbitMQ │  │  Vault   │  │  Redis   │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Custom Resource Definitions (CRDs)

### Agent CRD

```yaml
# crd-agent.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: agents.openclaw.io
spec:
  group: openclaw.io
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                role:
                  type: string
                  enum: [ops, mail, calendar, support, devops, finance]
                model:
                  type: string
                  default: "gpt-4o"
                allowedTools:
                  type: array
                  items:
                    type: string
                deniedTools:
                  type: array
                  items:
                    type: string
                requireApprovalFor:
                  type: array
                  items:
                    type: string
                bindings:
                  type: array
                  items:
                    type: object
                    properties:
                      channel:
                        type: string
                      target:
                        type: string
                replicas:
                  type: integer
                  default: 1
                resources:
                  type: object
                  properties:
                    requests:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                    limits:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                config:
                  type: object
                  additionalProperties:
                    type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: [Pending, Active, Error, Paused]
                lastHeartbeat:
                  type: string
                  format: date-time
                runsCompleted:
                  type: integer
                runsFailed:
                  type: integer
                costThisMonth:
                  type: number
```

### Example Agent Resource

```yaml
# ops-agent.yaml
apiVersion: openclaw.io/v1
kind: Agent
metadata:
  name: ops-agent
  namespace: openclaw
spec:
  role: ops
  model: "gpt-4o"
  allowedTools:
    - web_search
    - message
    - read
    - exec
    - gateway
    - cron
    - nodes
    - image
  deniedTools: []
  requireApprovalFor:
    - exec
    - gateway
  bindings:
    - channel: telegram
      target: "@ops_bot"
    - channel: slack
      target: "#ops-alerts"
  replicas: 1
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  config:
    SOUL_PATH: /config/SOUL.md
    TOOLS_PATH: /config/TOOLS.md
    HEARTBEAT_INTERVAL: "30s"
```

```yaml
# mail-agent.yaml
apiVersion: openclaw.io/v1
kind: Agent
metadata:
  name: mail-agent
  namespace: openclaw
spec:
  role: mail
  model: "gpt-4o-mini"
  allowedTools:
    - web_search
    - message
    - read
    - cron
  deniedTools:
    - exec
    - gateway
    - nodes
  requireApprovalFor:
    - message
  bindings:
    - channel: email
      target: "support@company.com"
  replicas: 1
  resources:
    requests:
      cpu: "50m"
      memory: "128Mi"
    limits:
      cpu: "200m"
      memory: "256Mi"
```

---

## 4. Agent Controller (Operator)

The controller watches Agent CRDs and manages their lifecycle.

```go
package controller

// Reconcile loop for Agent CRD
func (r *AgentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    agent := &openclawv1.Agent{}
    if err := r.Get(ctx, req.NamespacedName, agent); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 1. Ensure ConfigMap exists
    if err := r.reconcileConfigMap(ctx, agent); err != nil {
        return ctrl.Result{}, err
    }

    // 2. Ensure Secret exists (API keys, tokens)
    if err := r.reconcileSecret(ctx, agent); err != nil {
        return ctrl.Result{}, err
    }

    // 3. Ensure Deployment exists
    if err := r.reconcileDeployment(ctx, agent); err != nil {
        return ctrl.Result{}, err
    }

    // 4. Ensure Service exists (for health checks)
    if err := r.reconcileService(ctx, agent); err != nil {
        return ctrl.Result{}, err
    }

    // 5. Update status
    if err := r.updateStatus(ctx, agent); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *AgentReconciler) reconcileDeployment(ctx context.Context, agent *openclawv1.Agent) error {
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      agent.Name,
            Namespace: agent.Namespace,
            Labels:    r.agentLabels(agent),
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &agent.Spec.Replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: r.agentLabels(agent),
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: r.agentLabels(agent),
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "agent",
                            Image: "openclaw/agent:latest",
                            Env: []corev1.EnvVar{
                                {
                                    Name:  "AGENT_NAME",
                                    Value: agent.Name,
                                },
                                {
                                    Name:  "GATEWAY_URL",
                                    Value: "http://gateway.openclaw.svc.cluster.local:8080",
                                },
                                {
                                    Name:  "ALLOWED_TOOLS",
                                    Value: strings.Join(agent.Spec.AllowedTools, ","),
                                },
                                {
                                    Name:  "DENIED_TOOLS",
                                    Value: strings.Join(agent.Spec.DeniedTools, ","),
                                },
                                {
                                    Name:  "MODEL",
                                    Value: agent.Spec.Model,
                                },
                            },
                            VolumeMounts: []corev1.VolumeMount{
                                {
                                    Name:      "config",
                                    MountPath: "/config",
                                },
                                {
                                    Name:      "workspace",
                                    MountPath: "/workspace",
                                },
                            },
                            Resources: corev1.ResourceRequirements{
                                Requests: agent.Spec.Resources.Requests,
                                Limits:   agent.Spec.Resources.Limits,
                            },
                            LivenessProbe: &corev1.Probe{
                                HTTPGet: &corev1.HTTPGetAction{
                                    Path: "/health",
                                    Port: intstr.FromInt(8080),
                                },
                                InitialDelaySeconds: 10,
                                PeriodSeconds:       30,
                            },
                            ReadinessProbe: &corev1.Probe{
                                HTTPGet: &corev1.HTTPGetAction{
                                    Path: "/ready",
                                    Port: intstr.FromInt(8080),
                                },
                                InitialDelaySeconds: 5,
                                PeriodSeconds:       10,
                            },
                        },
                    },
                    Volumes: []corev1.Volume{
                        {
                            Name: "config",
                            VolumeSource: corev1.VolumeSource{
                                ConfigMap: &corev1.ConfigMapVolumeSource{
                                    LocalObjectReference: corev1.LocalObjectReference{
                                        Name: agent.Name + "-config",
                                    },
                                },
                            },
                        },
                        {
                            Name: "workspace",
                            VolumeSource: corev1.VolumeSource{
                                PersistentVolumeClaim: &corev1.PersistentVolumeClaimVolumeSource{
                                    ClaimName: agent.Name + "-workspace",
                                },
                            },
                        },
                    },
                },
            },
        },
    }

    // Create or update deployment
    existing := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{Name: agent.Name, Namespace: agent.Namespace}, existing)
    if err != nil {
        if errors.IsNotFound(err) {
            return r.Create(ctx, deployment)
        }
        return err
    }

    // Update if changed
    existing.Spec = deployment.Spec
    return r.Update(ctx, existing)
}
```

---

## 5. Gateway Deployment

```yaml
# gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: openclaw
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      containers:
        - name: gateway
          image: openclaw/gateway:latest
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: grpc
          env:
            - name: DB_HOST
              value: "postgres.openclaw.svc.cluster.local"
            - name: RABBITMQ_URL
              value: "amqp://guest:guest@rabbitmq.openclaw.svc.cluster.local:5672/"
            - name: VAULT_ADDR
              value: "http://vault.openclaw.svc.cluster.local:8200"
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: gateway
  namespace: openclaw
spec:
  selector:
    app: gateway
  ports:
    - port: 8080
      targetPort: 8080
      name: http
    - port: 9090
      targetPort: 9090
      name: grpc
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gateway
  namespace: openclaw
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt"
spec:
  tls:
    - hosts:
        - api.openclaw.internal
      secretName: gateway-tls
  rules:
    - host: api.openclaw.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gateway
                port:
                  number: 8080
```

---

## 6. Agent Pod Spec

```yaml
# agent-pod-template.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ops-agent-xyz
  namespace: openclaw
  labels:
    app: ops-agent
    agent-role: ops
    managed-by: openclaw-controller
spec:
  containers:
    - name: agent
      image: openclaw/agent:v1.2.3
      env:
        - name: AGENT_NAME
          value: "ops-agent"
        - name: GATEWAY_URL
          value: "http://gateway.openclaw.svc.cluster.local:8080"
        - name: ALLOWED_TOOLS
          value: "web_search,message,read,exec,gateway,cron,nodes,image"
        - name: DENIED_TOOLS
          value: ""
        - name: MODEL
          value: "gpt-4o"
        - name: SUBSCRIPTION_MODE
          value: "plus"  # or "api" for production
      volumeMounts:
        - name: config
          mountPath: /workspace/config
        - name: memory
          mountPath: /workspace/memory
        - name: tools
          mountPath: /workspace/tools
        - name: secrets
          mountPath: /workspace/secrets
          readOnly: true
      resources:
        requests:
          cpu: "100m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 30
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
  volumes:
    - name: config
      configMap:
        name: ops-agent-config
    - name: memory
      persistentVolumeClaim:
        claimName: ops-agent-memory
    - name: tools
      configMap:
        name: ops-agent-tools
    - name: secrets
      secret:
        secretName: ops-agent-secrets
```

---

## 7. Horizontal Pod Autoscaler (HPA)

```yaml
# hpa-mail-agent.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mail-agent-hpa
  namespace: openclaw
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mail-agent
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Pods
      pods:
        metric:
          name: openclaw_pending_runs
        target:
          type: AverageValue
          averageValue: "10"
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

---

## 8. Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-isolation
  namespace: openclaw
spec:
  podSelector:
    matchLabels:
      managed-by: openclaw-controller
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: gateway
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: gateway
      ports:
        - protocol: TCP
          port: 8080
    - to:
        - namespaceSelector:
            matchLabels:
              name: openclaw
      ports:
        - protocol: TCP
          port: 5432  # Postgres
        - protocol: TCP
          port: 5672  # RabbitMQ
        - protocol: TCP
          port: 8200  # Vault
    - to: []  # Internet
      ports:
        - protocol: TCP
          port: 443  # HTTPS for APIs
```

---

## 9. Resource Quotas

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: agent-quota
  namespace: openclaw
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    persistentvolumeclaims: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: agent-limits
  namespace: openclaw
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: 2Gi
      min:
        cpu: "50m"
        memory: "64Mi"
      type: Container
```

---

## 10. Helm Chart Structure

```
openclaw-platform/
├── Chart.yaml
├── values.yaml
├── values-production.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── crds/
│   │   └── agent-crd.yaml
│   ├── controller/
│   │   ├── deployment.yaml
│   │   ├── rbac.yaml
│   │   └── serviceaccount.yaml
│   ├── gateway/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   ├── agents/
│   │   └── _agent-template.tpl
│   ├── shared/
│   │   ├── postgres.yaml
│   │   ├── rabbitmq.yaml
│   │   ├── vault.yaml
│   │   └── redis.yaml
│   ├── monitoring/
│   │   ├── servicemonitor.yaml
│   │   └── networkpolicy.yaml
│   └── hpa.yaml
```

### values.yaml

```yaml
# values.yaml
replicaCount: 1

image:
  repository: openclaw
  tag: latest
  pullPolicy: IfNotPresent

gateway:
  replicas: 2
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  ingress:
    enabled: true
    host: api.openclaw.internal
    tls: true

controller:
  replicas: 1
  resources:
    requests:
      cpu: 100m
      memory: 128Mi

agents:
  ops-agent:
    enabled: true
    role: ops
    model: gpt-4o
    replicas: 1
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
    tools:
      allowed:
        - web_search
        - message
        - read
        - exec
        - gateway
        - cron
        - nodes
        - image
    bindings:
      - channel: telegram
        target: "@ops_bot"

  mail-agent:
    enabled: true
    role: mail
    model: gpt-4o-mini
    replicas: 1
    resources:
      requests:
        cpu: 50m
        memory: 128Mi
    tools:
      allowed:
        - web_search
        - message
        - read
        - cron
      denied:
        - exec
        - gateway

postgres:
  enabled: true
  persistence:
    size: 10Gi

rabbitmq:
  enabled: true
  persistence:
    size: 5Gi

vault:
  enabled: true
  devMode: true  # Set to false in production

autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

---

## 11. Deployment Commands

```bash
# 1. Install CRDs
kubectl apply -f crds/

# 2. Install platform
helm install openclaw ./openclaw-platform \
  --namespace openclaw \
  --create-namespace \
  --values values.yaml

# 3. Check status
kubectl get agents -n openclaw
kubectl get pods -n openclaw
kubectl get svc -n openclaw

# 4. Scale an agent
kubectl scale agent mail-agent --replicas=3 -n openclaw

# 5. Update agent config
kubectl edit agent ops-agent -n openclaw
# Controller automatically rolls out changes

# 6. View logs
kubectl logs -n openclaw -l app=ops-agent --tail=100

# 7. Port forward for local testing
kubectl port-forward svc/gateway 8080:8080 -n openclaw

# 8. Upgrade platform
helm upgrade openclaw ./openclaw-platform -n openclaw

# 9. Uninstall
helm uninstall openclaw -n openclaw
```

---

## 12. Monitoring

### Prometheus ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: openclaw-metrics
  namespace: openclaw
spec:
  selector:
    matchLabels:
      managed-by: openclaw-controller
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

### Key Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `openclaw_agents_total` | Gauge | Number of agents |
| `openclaw_agent_runs_total` | Counter | Total runs per agent |
| `openclaw_agent_run_duration_seconds` | Histogram | Run duration |
| `openclaw_agent_run_cost_usd` | Counter | Cumulative cost |
| `openclaw_agent_heartbeat_timestamp` | Gauge | Last heartbeat |
| `openclaw_pending_runs` | Gauge | Queue depth |
| `openclaw_approvals_pending` | Gauge | Pending approvals |
| `openclaw_tool_calls_total` | Counter | Tool calls by type |

---

## 13. Multi-Environment Setup

```bash
# Development
helm install openclaw-dev ./openclaw-platform \
  --namespace openclaw-dev \
  --values values.yaml \
  --set vault.devMode=true \
  --set agents.ops-agent.replicas=1

# Staging
helm install openclaw-staging ./openclaw-platform \
  --namespace openclaw-staging \
  --values values-staging.yaml \
  --set vault.devMode=false

# Production
helm install openclaw-prod ./openclaw-platform \
  --namespace openclaw-prod \
  --values values-production.yaml \
  --set vault.devMode=false \
  --set gateway.replicas=3 \
  --set postgres.persistence.size=100Gi
```

---

## 14. Backup and DR

### Velero Backup

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: openclaw-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - openclaw
    includedResources:
      - agents.openclaw.io
      - configmaps
      - secrets
      - persistentvolumeclaims
    storageLocation: default
    volumeSnapshotLocations:
      - aws-default
```

### Agent Workspace Backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: agent-backup
  namespace: openclaw
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: busybox
              command:
                - /bin/sh
                - -c
                - |
                  tar czf /backup/agents-$(date +%Y%m%d).tar.gz /workspace/
                  aws s3 cp /backup/ s3://openclaw-backups/agents/ --recursive
              volumeMounts:
                - name: workspace
                  mountPath: /workspace
          restartPolicy: OnFailure
```
