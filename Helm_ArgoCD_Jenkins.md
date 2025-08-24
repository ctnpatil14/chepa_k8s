Bhai Chetan, Bunty Dada full swing mode ON! चल, एकदम neat and reusable Helm chart तयार करूया — जे production-ready आहे आणि ArgoCD, probes, NetworkPolicy, आणि CI/CD साठी तयार आहे.

---

## 📦 Helm Chart: `myapp-chart`

### 🗂️ Folder Structure

```
myapp-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── networkpolicy.yaml
│   └── _helpers.tpl
```

---

## 🧾 `Chart.yaml`

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for deploying myapp
version: 0.1.0
appVersion: "1.0"
```

---

## 🧾 `values.yaml`

```yaml
replicaCount: 2

image:
  repository: cdtsbikaner/devopstgmay2025
  tag: tomcat-v1
  pullPolicy: IfNotPresent

containerPorts:
  - name: tomcat
    port: 8080
  - name: web
    port: 80

probes:
  enabled: true
  tomcat:
    path: /health
    port: 8080
  web:
    path: /
    port: 80

service:
  type: ClusterIP

networkPolicy:
  enabled: true
  allowedPorts:
    - 8080
    - 80
```

---

## 📄 `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: tomcat
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.containerPorts[0].port }}
          {{- if .Values.probes.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.tomcat.path }}
              port: {{ .Values.probes.tomcat.port }}
            initialDelaySeconds: 5
            periodSeconds: 10
          {{- end }}
        - name: web
          image: "{{ .Values.image.repository }}:webserver-v1.2"
          ports:
            - containerPort: {{ .Values.containerPorts[1].port }}
          {{- if .Values.probes.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.web.path }}
              port: {{ .Values.probes.web.port }}
            initialDelaySeconds: 5
            periodSeconds: 10
          {{- end }}
```

---

## 📄 `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}-svc
spec:
  selector:
    app: {{ .Chart.Name }}
  type: {{ .Values.service.type }}
  ports:
    - name: tomcat
      port: {{ .Values.containerPorts[0].port }}
      targetPort: {{ .Values.containerPorts[0].port }}
    - name: web
      port: {{ .Values.containerPorts[1].port }}
      targetPort: {{ .Values.containerPorts[1].port }}
```

---

## 📄 `networkpolicy.yaml`

```yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-myapp-traffic
spec:
  podSelector:
    matchLabels:
      app: {{ .Chart.Name }}
  policyTypes:
    - Ingress
  ingress:
    - ports:
        {{- range .Values.networkPolicy.allowedPorts }}
        - protocol: TCP
          port: {{ . }}
        {{- end }}
{{- end }}
```

---

## 🚀 ArgoCD Integration

Push this chart to GitHub, then create an ArgoCD `Application`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  project: default
  source:
    repoURL: https://github.com/chetanpatil/myapp-chart
    targetRevision: HEAD
    path: myapp-chart
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 🛠️ Jenkins CI/CD Pipeline (Declarative)

```groovy
pipeline {
  agent any
  stages {
    stage('Lint Helm') {
      steps {
        sh 'helm lint myapp-chart'
      }
    }
    stage('Package & Push') {
      steps {
        sh 'helm package myapp-chart'
        sh 'helm push myapp-chart-*.tgz oci://myrepo'
      }
    }
    stage('Trigger ArgoCD Sync') {
      steps {
        sh 'argocd app sync myapp'
      }
    }
  }
}
```

---

भाई, हे chart तू customize करू शकतोस — values.yaml मधून environment-specific configs, probes, ports, आणि even RBAC जोडता येतील.  
सांग, पुढे काय पाहिजे — Helm testing with `unittest`, chart versioning strategy, की sealed secrets integration? 😎
