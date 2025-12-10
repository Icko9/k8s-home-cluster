# Ghost Chart Breakdown - Learning Guide

This document explains each part of the Ghost Helm chart, showing you exactly how it works.

## 📁 File Structure

```
apps/ghost/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Configuration values
├── templates/
│   ├── _helpers.tpl       # Reusable template functions
│   ├── deployment.yaml    # Main application deployment
│   ├── service.yaml       # Kubernetes service
│   └── pvc.yaml          # Persistent volume claim
```

---

## 1. Chart.yaml - Chart Metadata

```yaml
apiVersion: v2
name: ghost
description: A Helm chart for Ghost CMS using the official Docker image
type: application
version: 0.1.0
appVersion: "5.85.0"
```

**What it does:**
- Defines chart identity and version
- `appVersion` = version of Ghost itself
- `version` = version of this chart (increment when you change the chart)

**Why it matters:**
- Helm uses this for chart management
- Versioning helps track changes
- Used in labels and metadata

---

## 2. values.yaml - Configuration

This is where users customize the deployment. Think of it as "settings" for your chart.

### Image Configuration

```yaml
image:
  repository: ghost
  tag: "5.85.0"
  pullPolicy: IfNotPresent
```

**Used in template as:**
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**Result:** `image: "ghost:5.85.0"`

### Ghost URL

```yaml
ghost:
  url: "http://192.168.1.10:30080"
```

**Used in template as:**
```yaml
env:
- name: url
  value: {{ .Values.ghost.url | quote }}
```

**Result:** `value: "http://192.168.1.10:30080"`

**Why `| quote`?** Ensures the value is treated as a string in YAML.

### Conditional Database Config

```yaml
database:
  client: "sqlite3"  # or "mysql"
  connection:
    host: "mysql-host"
    ...
```

**Used in template as:**
```yaml
{{- if eq .Values.ghost.database.client "mysql" }}
- name: database__client
  value: {{ .Values.ghost.database.client | quote }}
{{- end }}
```

**What happens:**
- If `client: "sqlite3"` → No database env vars (uses default SQLite)
- If `client: "mysql"` → Adds MySQL connection env vars

**The `{{- if }}` pattern:** Only includes this block if condition is true.

---

## 3. _helpers.tpl - Template Helpers

These are reusable functions that prevent repetition.

### Fullname Helper

```yaml
{{- define "ghost.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}
```

**What it does:**
- Creates a consistent name for resources
- Handles overrides
- Ensures names fit Kubernetes limits (63 chars)

**Used as:**
```yaml
name: {{ include "ghost.fullname" . }}
```

**Example results:**
- Release name `ghost` → Resource name `ghost`
- Release name `my-ghost` → Resource name `my-ghost`

### Labels Helper

```yaml
{{- define "ghost.labels" -}}
helm.sh/chart: {{ include "ghost.chart" . }}
{{ include "ghost.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

**What it does:**
- Creates standard Kubernetes labels
- Includes chart version, app version
- Marks resources as managed by Helm

**Used as:**
```yaml
metadata:
  labels:
    {{- include "ghost.labels" . | nindent 4 }}
```

**The `| nindent 4` part:** Indents the output by 4 spaces (for proper YAML formatting).

---

## 4. deployment.yaml - Main Deployment

This is the core template. Let's break it down section by section.

### Metadata Section

```yaml
metadata:
  name: {{ include "ghost.fullname" . }}
  labels:
    {{- include "ghost.labels" . | nindent 4 }}
```

**What happens:**
- Name uses the helper function
- Labels are included from helper
- `| nindent 4` properly indents the labels

### Replicas

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
```

**Simple value substitution:**
- `replicaCount: 1` in values.yaml → `replicas: 1`

### Container Image

```yaml
containers:
- name: {{ .Chart.Name }}
  image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
  imagePullPolicy: {{ .Values.image.pullPolicy }}
```

**Breaking it down:**
- `{{ .Chart.Name }}` → "ghost" (from Chart.yaml)
- `{{ .Values.image.repository }}` → "ghost" (from values.yaml)
- `{{ .Values.image.tag }}` → "5.85.0" (from values.yaml)
- **Result:** `image: "ghost:5.85.0"`

### Environment Variables

#### Simple Variable

```yaml
env:
- name: url
  value: {{ .Values.ghost.url | quote }}
```

**Result:**
```yaml
env:
- name: url
  value: "http://192.168.1.10:30080"
```

#### Conditional Variables

```yaml
{{- if eq .Values.ghost.database.client "mysql" }}
- name: database__client
  value: {{ .Values.ghost.database.client | quote }}
{{- if .Values.ghost.database.connection }}
- name: database__connection__host
  value: {{ .Values.ghost.database.connection.host | quote }}
{{- end }}
{{- end }}
```

**How it works:**
1. First `{{- if }}`: Only if database client is "mysql"
2. Second `{{- if }}`: Only if connection config exists
3. `{{- end }}`: Closes each if block

**Result (if MySQL configured):**
```yaml
env:
- name: database__client
  value: "mysql"
- name: database__connection__host
  value: "mysql-host"
```

**Result (if SQLite):**
```yaml
env:
# No database env vars (uses SQLite default)
```

### Conditional Volume Mounts

```yaml
{{- if .Values.persistence.enabled }}
volumeMounts:
- name: ghost-content
  mountPath: {{ .Values.persistence.mountPath }}
{{- end }}
```

**Pattern:** Only include volume mount if persistence is enabled.

### Resources

```yaml
{{- if .Values.resources }}
resources:
  {{- toYaml .Values.resources | nindent 10 }}
{{- end }}
```

**What `toYaml` does:**
- Converts the values.yaml structure to YAML
- `| nindent 10` indents it properly

**Input (values.yaml):**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
```

**Output (in template):**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
```

### Conditional Volumes

```yaml
{{- if .Values.persistence.enabled }}
volumes:
- name: ghost-content
  persistentVolumeClaim:
    claimName: {{ include "ghost.fullname" . }}
{{- end }}
```

**Key point:** Uses the helper function for consistent naming.

---

## 5. service.yaml - Service Template

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "ghost.fullname" . }}
  labels:
    {{- include "ghost.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      {{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
  selector:
    {{- include "ghost.selectorLabels" . | nindent 4 }}
```

### Key Parts:

**Service Type:**
```yaml
type: {{ .Values.service.type }}
```
Result: `type: NodePort`

**Conditional NodePort:**
```yaml
{{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
nodePort: {{ .Values.service.nodePort }}
{{- end }}
```

**Breaking down `{{- if and ... }}`:**
- `and` = both conditions must be true
- `eq .Values.service.type "NodePort"` = service type is NodePort
- `.Values.service.nodePort` = nodePort value exists (not null)

**Result:**
- If NodePort + nodePort set → includes nodePort
- If ClusterIP → no nodePort
- If NodePort + nodePort null → no nodePort (auto-assigned)

**Selector:**
```yaml
selector:
  {{- include "ghost.selectorLabels" . | nindent 4 }}
```

Matches pods with the same labels.

---

## 6. pvc.yaml - Persistent Volume Claim

```yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "ghost.fullname" . }}
  labels:
    {{- include "ghost.labels" . | nindent 4 }}
spec:
  accessModes:
    {{- toYaml .Values.persistence.accessModes | nindent 4 }}
  resources:
    requests:
      storage: {{ .Values.persistence.size }}
  {{- if .Values.persistence.storageClass }}
  {{- if (eq "-" .Values.persistence.storageClass) }}
  storageClassName: ""
  {{- else }}
  storageClassName: {{ .Values.persistence.storageClass }}
  {{- end }}
  {{- end }}
{{- end }}
```

### Key Patterns:

**Conditional Resource:**
Only creates PVC if `persistence.enabled: true`

**Storage Class Handling:**
```yaml
{{- if .Values.persistence.storageClass }}
  {{- if (eq "-" .Values.persistence.storageClass) }}
  storageClassName: ""  # Explicitly empty
  {{- else }}
  storageClassName: {{ .Values.persistence.storageClass }}  # Use specified class
  {{- end }}
{{- end }}
```

**Why this pattern?**
- If storageClass is `"-"` → set to empty string (use default)
- If storageClass is `"longhorn"` → use Longhorn
- If storageClass is null → don't set (use cluster default)

---

## 🎯 Template Syntax Cheat Sheet

### Basic Substitution
```yaml
name: {{ .Values.name }}
```

### With Default
```yaml
name: {{ .Values.name | default "default" }}
```

### String Quoting
```yaml
value: {{ .Values.url | quote }}
```

### Conditionals
```yaml
{{- if .Values.enabled }}
enabled: true
{{- end }}

{{- if eq .Values.env "prod" }}
replicas: 3
{{- else }}
replicas: 1
{{- end }}
```

### Include Helpers
```yaml
name: {{ include "ghost.fullname" . }}
labels:
  {{- include "ghost.labels" . | nindent 4 }}
```

### Convert to YAML
```yaml
{{- toYaml .Values.config | nindent 4 }}
```

### Whitespace Control
```yaml
{{- }}  # Trim before
{{ -}}  # Trim after
{{- -}} # Trim both
```

---

## 🔍 How I Built This Chart

### Step 1: Research
- Checked Ghost Docker image documentation
- Found required environment variables
- Identified port (2368)
- Found volume mount path (`/var/lib/ghost/content`)

### Step 2: Structure
- Created Chart.yaml with metadata
- Defined values.yaml with all configurable options
- Created helper templates for reusability

### Step 3: Templates
- Started with deployment.yaml (main resource)
- Added service.yaml for networking
- Added pvc.yaml for storage
- Used conditionals for optional features

### Step 4: Testing
- Used `helm template` to see rendered output
- Used `helm lint` to validate
- Tested with `--dry-run --debug`

### Step 5: Documentation
- Added comments to values.yaml
- Created README
- Wrote setup guide

---

## 💡 Key Takeaways

1. **Start with values.yaml** - Define what users can configure
2. **Use helpers** - Avoid repetition, ensure consistency
3. **Use conditionals** - Make features optional
4. **Test frequently** - Use `helm template` and `helm lint`
5. **Study examples** - Look at well-maintained charts
6. **Document well** - Comments and README help users

---

## 🚀 Next Steps

1. **Modify the chart:**
   - Add an Ingress template
   - Add a ConfigMap for Ghost config
   - Add health check customization

2. **Create your own:**
   - Start with a simple app (nginx)
   - Add features one at a time
   - Test each addition

3. **Study more:**
   - Look at Bitnami charts
   - Check Artifact Hub
   - Read Helm documentation

Remember: Practice makes perfect! Start simple and build up complexity.

