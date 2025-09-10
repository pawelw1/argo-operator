This OpenSearch Operator autoscaling demo will include the following configs

1. [GitHub/GitLab repository](#githubgitlab-repository)
2. k8s cluster - for demo will use Docker Desktop k8s
3. [[#ArgoCD]]
4. [[#OpenSearch Operator]]
5. OpenSearch cluster manifest 
6. OpenSearch Prometheus exporter
7. Prometheus operator [[#Monitoring]]
8. Grafana [[#Monitoring]]
9. Alertmanager [[#Monitoring]]
10. [[#Alert Manager]]
11. [[#ArgoEvents]]
12. [[#Argo Workflows]]
13. [[#Manual testing]]

### GitHub/GitLab repository 

##### Installation steps:
1. Add Gitlab repo 
```bash
helm repo add gitlab https://charts.gitlab.io/
helm repo update
```

2. Deploy Gitlab CE 
```bash
helm upgrade --install gitlab gitlab/gitlab --timeout 600s --set global.hosts.domain=gitlab.pablo.local --set global.edition=ce -f values.yaml -n gitlab
```

3. Initial Gitlab password is located in the secret 
```bash
kubectl get secret <name>-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ; echo
```

4. Use Opensearch cluster manifest with Prometheus exporter and EventService object for Prometheus operator
   Use OpenSearch Prometheus exporter plugin from OpenSearch project GitHub. 
```yaml
apiVersion: opensearch.opster.io/v1
kind: OpenSearchCluster
metadata:
  name: my-first-cluster
  namespace: default
spec:
  security:
    config:
    tls:
      http:
        generate: true
      transport:
        generate: true
        perNode: true
 confMgmt:
    smartScaler: true
  general:
    httpPort: 9200
    serviceName: my-first-cluster
    version: "3.2.0"
    pluginsList: ["repository-s3"]
    drainDataNodes: true
    monitoring:
      enable: true # Enable or disable the monitoring plugin
      labels: # The labels add for ServiceMonitor
        monitoring: prometheus
      scrapeInterval: 30s # The scrape interval for Prometheus
      pluginUrl: https://github.com/opensearch-project/opensearch-prometheus-exporter/releases/download/3.2.0.0/prometheus-exporter-3.2.0.0.zip
      tlsConfig:
        insecureSkipVerify: true
  dashboards:
    service:
      type: NodePort
    tls:
      enable: true
      generate: true
    version: 3.2.0
    enable: true
    replicas: 1
    resources:
      requests:
        memory: "512Mi"
        cpu: "200m"
      limits:
        memory: "512Mi"
        cpu: "200m"
  nodePools:
    - component: masters
      replicas: 3
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "2Gi"
          cpu: "1000m"
      roles:
        - "cluster_manager"
      persistence:
        emptyDir: {}
    - component: data
      replicas: 2
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "2Gi"
          cpu: "1000m"
      roles:
        - "data"
      persistence:
        pvc:
          storageClass: managed-nfs-storage
          accessModes:
            - ReadWriteOnce

```   


### ArgoCD 

1. Install ArgoCD using the following commands 
```bash
kubectl apply -f https://github.com/argoproj/argo-cd/blob/bc493296910496a957c47f9f14dbec9925a9f425/manifests/install.yaml
```

2. Install argocd command line 
```bash
brew install argocd 
```   

3. Get initial password 
```bash
kubectl get secret <name>-initial-admin-secret -ojsonpath='{.data.password}' | base64 --decode ; echo
```

4. Reset password with argocd cli and initial password 
```bash
argocd login localhost:<argocd-server_service_port> --username admin --password <initial_admin_password>
```


### OpenSearch Operator

1. Install OpenSearch Operator with ArgoCD 
   ![[Pasted image 20250719020058.png]]


### Opensearch Cluster 

1. Create OpenSearch cluster using OpenSearchCluster manifest from Gitlab 
   ![[Pasted image 20250719020249.png]]
### Monitoring

1. Clone argo-operator repo from local gitlab to install Prometheus Operator
   Original link https://github.com/prometheus-operator/prometheus-operator.git
```bash
git clone https://gitlab.pablo.local/pablo/argo-operator.git
```

2. Navigate to the folder and create CRDs 
```bash
kubectl create -f setup/
```

3. Create monitoring namespace 
```bash
kubectl create ns monitoring
```

4. Create Monitoring App with ArgoCD 
   ![[Pasted image 20250719015803.png]]
### Argo Events and Workflows Service Account 

1. Create roles and rolebinding for Service Account argo and argo-workflows

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argo-workflow-trigger-binding
  namespace: argo
subjects:
  - kind: ServiceAccount
    name: default
    namespace: argo-events
roleRef:
  kind: Role
  name: argo-workflow-trigger
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argo-workflow-trigger
  namespace: argo
rules:
  - apiGroups: ["argoproj.io"]
    resources: ["workflowtemplates"]
    verbs: ["get"]
  - apiGroups: ["argoproj.io"]
    resources: ["workflows"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: argo   # match the Role namespace
  name: argo-rolebinding
subjects:
- kind: ServiceAccount
  name: argo         # or the name of your Argo workflow service account
  namespace: argo
roleRef:
  kind: Role
  name: argo-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: argo   # change to your workflow namespace
  name: argo-role
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list", "create", "delete"]
- apiGroups: ["argoproj.io"]
  resources: ["workflows"]
  verbs: ["get", "watch", "list", "create", "delete", "update", "patch"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["create", "delete", "get", "list"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "watch", "list", "create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
    meta.helm.sh/release-name: argo-workflows
    meta.helm.sh/release-namespace: argo
  creationTimestamp: "2025-07-06T23:45:45Z"
  labels:
    app: workflow-controller
    app.kubernetes.io/component: workflow-controller
    app.kubernetes.io/instance: argo-workflows
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: argo-workflows-workflow-controller
    app.kubernetes.io/part-of: argo-workflows
    helm.sh/chart: argo-workflows-0.45.19
  name: argo-workflows-workflow
  namespace: argo
  resourceVersion: "19086620"
  uid: 28d5a10d-5f0a-48df-9d60-c80d88d1d52a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argo-workflows-workflow
subjects:
- kind: ServiceAccount
  name: argo-workflow
  namespace: argo
```


### ArgoEvents

##### Installation with helm charts. 
1. Add repository 
```bash
helm repo add argo https://argoproj.github.io/argo-helm
```

2. Install ArgoEvents in argo-events namespace
```bash
helm install argo-events argo/argo-events -n argo-events --create-namespace
```

3. Install Event bus
```bash
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml
```

##### Once completed 

1. Create Webhook using EventSource object
``` yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: alertmanager-events-scale
  namespace: argo-events
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  webhook:
    scale-up:
      port: "12000"
      endpoint: /scale-up
      method: POST
    scale-down:
      port: "12000"
      endpoint: /scale-down
      method: POST

```

2. Create sensors 
   
   Scale up sensor 
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: trigger-scale-up-os-cluster
  namespace: argo-events
spec:
  dependencies:
    - name: alert-up
      eventSourceName: alertmanager-events-scale
      eventName: scale-up
  triggers:
    - template:
        name: trigger-workflow
        argoWorkflow:
          group: argoproj.io
          version: v1alpha1
          resource: workflows
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: scale-cluster-
                namespace: argo
              spec:
                workflowTemplateRef:
                  name: scale-up-opensearch-template

```

   Scale down sensor 
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: trigger-scale-down-os-cluster
  namespace: argo-events
spec:
  dependencies:
    - name: alert-down
      eventSourceName: alertmanager-events-scale
      eventName: scale-down
  triggers:
    - template:
        name: trigger-workflow
        argoWorkflow:
          group: argoproj.io
          version: v1alpha1
          resource: workflows
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: scale-cluster-
                namespace: argo
              spec:
                workflowTemplateRef:
                  name: scale-down-opensearch-template
```
3. Create Workflow templates 
   
   Scale up Workflow template 
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: scale-up-opensearch-template
  generateName: scale-opensearch-up-
  namespace: argo
spec:
  entrypoint: scale-opensearch-up
  serviceAccountName: argo-workflow

  templates:
    - name: scale-opensearch-up
      container:
        image: alpine/git
        command: [sh, -c]
        args:
          - |
            apk add --no-cache git curl yq bash;
            git clone http://pablo:Eliatra123@gitlab-webservice-default.gitlab.svc:8181/pablo/argo-operator.git;
            cd argo-operator;
            echo "Scaling replicas...";
            yq -i '.spec.nodePools[] |= (select(.component == "data") .replicas += 1)' main/cluster.yml;
            git config user.name "Argo Bot";
            git config user.email "argo@local";
            git add main/cluster.yml;
            git commit -m "🔁 Argo scaled up data node replicas";
            git push origin main;

```

   Scale down Workflow template 
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: scale-down-opensearch-template
  generateName: scale-opensearch-down-
  namespace: argo
spec:
  entrypoint: scale-opensearch-down
  serviceAccountName: argo-workflow

  templates:
    - name: scale-opensearch-down
      container:
        image: alpine/git
        command: [sh, -c]
        args:
          - |
            apk add --no-cache git curl yq bash;
            git clone http://pablo:Eliatra123@gitlab-webservice-default.gitlab.svc:8181/pablo/argo-operator.git;
            cd argo-operator;
            echo "Scaling replicas...";
            yq -i '.spec.nodePools[] |= (select(.component == "data") .replicas -= 1)' main/cluster.yml;
            git config user.name "Argo Bot";
            git config user.email "argo@local";
            git add main/cluster.yml;
            git commit -m "🔁 Argo scaled down data node replicas";
            git push origin main;

```



### Argo Workflows 

1. Create additional values.yml file
```yaml
server:
  extraArgs:
  - --auth-mode=server
  serviceType: NodePort
  serviceNodePort: 32746
  secure: true
workflow:
  serviceAccount:
    create: true
    name: "argo-workflow"
  rbac:
    create: true
controller:
  workflowNamespaces:
    - default
    - argo
```
   
   
2. Install Argo Workflows 
```bash
helm install argo-workflows argo/argo-workflows  -f server-auth.yaml -n argo
```


### Alert Manager

Get current AlertManager config from a secret 

```bash
kubectl get secret alertmanager-main -n monitoring -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d > alertmanager.yaml
```

Update alertmanager.yml with OpenSearch scale up and scale down

```
"global":
  "resolve_timeout": "5m"

"inhibit_rules":
- "equal":
  - "namespace"
  - "alertname"
  "source_matchers":
  - "severity = critical"
  "target_matchers":
  - "severity =~ warning|info"
- "equal":
  - "namespace"
  - "alertname"
  "source_matchers":
  - "severity = warning"
  "target_matchers":
  - "severity = info"
- "equal":
  - "namespace"
  "source_matchers":
  - "alertname = InfoInhibitor"
  "target_matchers":
  - "severity = info"

"receivers":
- "name": "Default"
- "name": "Watchdog"
- "name": "Critical"
- "name": "null"
- "name": "argo-events-scale-up"
  "webhook_configs":
    - "url": "http://alertmanager-events-scale.argo-events.svc.cluster.local:12000/scale-up"
- "name": "argo-events-scale-down"
  "webhook_configs":
    - "url": "http://alertmanager-events-scale.argo-events.svc.cluster.local:12000/scale-down"

"route":
  "group_by":
  - "namespace"
  "group_interval": "5m"
  "group_wait": "30s"
  "receiver": "Default"
  "repeat_interval": "12h"
  "routes":
  - "matchers":
    - "alertname = Watchdog"
    "receiver": "Watchdog"
  - "matchers":
    - "alertname = InfoInhibitor"
    "receiver": "null"
  - "matchers":
    - "severity = critical"
    "receiver": "Critical"
  - "matchers":
    - "alertname = OpenSearchScaleUp"
    "receiver": "argo-events-scale-up"
  - "matchers":
    - "alertname = OpenSearchScaleDown"
    "receiver": "argo-events-scale-down"

```

Apply new configuration to the secret. 


```bash
kubectl create secret generic alertmanager-main -n monitoring \
  --from-file=alertmanager.yaml=alertmanager.yaml \
  --dry-run=client -o yaml | kubectl apply -f -
```

Delete AlerManager pod to apply the configuration. 

```bash
kubectl delete pod -n monitoring -l alertmanager=main
```


### Manual testing

Manually scale down or the cluster. 

```bash 
curl -X POST http://localhost:12000/scale-down  -H "Content-Type: application/json"   -d '{"status": "scale-down", "message": "Cluster under pressure"}'
curl -X POST http://localhost:12000/scale-up  -H "Content-Type: application/json"   -d '{"status": "scale-up", "message": "Cluster is fine"}'
```
