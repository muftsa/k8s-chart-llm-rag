# k8s-chart-llm-rag

To create a Helm chart for deploying large language models (LLMs) with vector-based retrieval-augmented generation (RAG) on Kubernetes, you can follow the structure below. This Helm chart will:
- Deploy an LLM inference service, possibly with a model like GPT or a similar model hosted in a container.
- Deploy a vector database (e.g., FAISS or a hosted option like Pinecone or Weaviate) for storing embeddings.
- Integrate the LLM service with the vector database for RAG capabilities.

The Helm chart’s structure assumes that you have a Docker image for your LLM model and one for the vector database (or use an available vector database Helm chart, such as Weaviate).

### Directory Structure
The Helm chart directory (`llm-rag-chart`) might look like this:
```
k8s-chart-llm-rag/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── vector-db.yaml
│   ├── ingress.yaml
└── README.md
```

### Chart Files

#### `Chart.yaml`
```yaml
apiVersion: v2
name: llm-rag-chart
description: A Helm chart for deploying an LLM with RAG capabilities on Kubernetes
version: 0.1.0
appVersion: "1.0"
```

#### `values.yaml`
The `values.yaml` file contains configuration values for deploying the LLM and vector database.

```yaml
# LLM Service
llm:
  image: "your-llm-image:latest"  # Replace with your LLM Docker image
  replicas: 1
  port: 8080
  resources:
    limits:
      cpu: "2000m"
      memory: "4Gi"
    requests:
      cpu: "1000m"
      memory: "2Gi"
  env:
    VECTOR_DB_ENDPOINT: "http://vector-db:8081"  # Points to the vector database

# Vector Database (e.g., Weaviate)
vectorDb:
  enabled: true
  image: "vector-db-image:latest"  # Replace with your vector database Docker image
  port: 8081
  replicas: 1
  resources:
    limits:
      cpu: "1000m"
      memory: "2Gi"
    requests:
      cpu: "500m"
      memory: "1Gi"

# Ingress
ingress:
  enabled: true
  annotations: {}
  hosts:
    - host: llm-rag.local
      paths:
        - path: /
  tls: []
```

#### `templates/deployment.yaml`
Defines the deployment for the LLM inference service.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "llm-rag-chart.fullname" . }}-llm
  labels:
    app: {{ include "llm-rag-chart.name" . }}
spec:
  replicas: {{ .Values.llm.replicas }}
  selector:
    matchLabels:
      app: {{ include "llm-rag-chart.name" . }}-llm
  template:
    metadata:
      labels:
        app: {{ include "llm-rag-chart.name" . }}-llm
    spec:
      containers:
        - name: llm
          image: {{ .Values.llm.image }}
          ports:
            - containerPort: {{ .Values.llm.port }}
          env:
            - name: VECTOR_DB_ENDPOINT
              value: {{ .Values.llm.env.VECTOR_DB_ENDPOINT }}
          resources:
            limits:
              cpu: {{ .Values.llm.resources.limits.cpu }}
              memory: {{ .Values.llm.resources.limits.memory }}
            requests:
              cpu: {{ .Values.llm.resources.requests.cpu }}
              memory: {{ .Values.llm.resources.requests.memory }}
```

#### `templates/vector-db.yaml`
Defines the vector database deployment. (Alternatively, you could use an existing Helm chart for Weaviate or a similar database.)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "llm-rag-chart.fullname" . }}-vector-db
  labels:
    app: {{ include "llm-rag-chart.name" . }}-vector-db
spec:
  replicas: {{ .Values.vectorDb.replicas }}
  selector:
    matchLabels:
      app: {{ include "llm-rag-chart.name" . }}-vector-db
  template:
    metadata:
      labels:
        app: {{ include "llm-rag-chart.name" . }}-vector-db
    spec:
      containers:
        - name: vector-db
          image: {{ .Values.vectorDb.image }}
          ports:
            - containerPort: {{ .Values.vectorDb.port }}
          resources:
            limits:
              cpu: {{ .Values.vectorDb.resources.limits.cpu }}
              memory: {{ .Values.vectorDb.resources.limits.memory }}
            requests:
              cpu: {{ .Values.vectorDb.resources.requests.cpu }}
              memory: {{ .Values.vectorDb.resources.requests.memory }}
```

#### `templates/service.yaml`
Defines the service resources to expose both the LLM and vector database within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "llm-rag-chart.fullname" . }}-llm
spec:
  selector:
    app: {{ include "llm-rag-chart.name" . }}-llm
  ports:
    - protocol: TCP
      port: 80
      targetPort: {{ .Values.llm.port }}

---

apiVersion: v1
kind: Service
metadata:
  name: {{ include "llm-rag-chart.fullname" . }}-vector-db
spec:
  selector:
    app: {{ include "llm-rag-chart.name" . }}-vector-db
  ports:
    - protocol: TCP
      port: 8081
      targetPort: {{ .Values.vectorDb.port }}
```

#### `templates/ingress.yaml`
Defines an optional ingress resource to expose the LLM RAG service externally.

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "llm-rag-chart.fullname" . }}
  annotations: {{ toYaml .Values.ingress.annotations | nindent 4 }}
spec:
  rules:
  - host: {{ .Values.ingress.hosts[0].host }}
    http:
      paths:
      - path: {{ .Values.ingress.hosts[0].paths[0].path }}
        pathType: Prefix
        backend:
          service:
            name: {{ include "llm-rag-chart.fullname" . }}-llm
            port:
              number: 80
{{- end }}
```

### README.md
Provide usage instructions for deploying the Helm chart.

```markdown
# llm-rag-chart

A Helm chart for deploying a large language model (LLM) with a vector database for retrieval-augmented generation (RAG) on Kubernetes.

## Installation

1. **Install Helm** if not already installed:
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

2. **Deploy the chart**:
   ```bash
   helm install my-llm-rag ./llm-rag-chart
   ```

3. **Configure Ingress (optional)**:
   - Edit `values.yaml` to enable and configure ingress.

## Configuration

- `values.yaml` allows configuration of resources, LLM model parameters, vector database settings, and other options.
