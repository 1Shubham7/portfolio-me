---
title: "How to Make Pods Restart When a ConfigMap Changes"
description: "Why a changed ConfigMap restarts nothing, the Helm checksum annotation done properly, the two mistakes that make it silently do nothing, and when an immutable ConfigMap, Reloader, or a reload sidecar is the better tool. Written after two years of writing and managing Helm packages in production."
dateString: September 2026
draft: false
tags: ["Helm", "Kubernetes", "SRE", "ConfigMap", "GitOps"]
weight: 1
cover:
    image: "/blog/configmap-restart/cover.png"
---

You change a value, run `helm upgrade`, the ConfigMap in the cluster updates, and the pods carry on running the old config until something unrelated restarts them: a node drain, an OOM kill, an image bump three days later. Kubernetes is not broken and neither is Helm; as far as the Deployment controller can see, nothing about the Deployment changed. Here is the fix, the two ways people get it wrong, and the three cases where it is the wrong fix.

## The problem, in one screen

A Deployment's controller watches exactly one thing for "should I roll out": `spec.template`. Change the image, an env var, a label on the template, an annotation on the template, and it creates a new ReplicaSet and shifts pods over. Change anything else, `replicas` say, and it does not. The Kubernetes docs put it in one sentence: a rollout is triggered only if the Deployment's Pod template is changed.

The pod template does not contain the ConfigMap. It contains a pointer:

```yaml
volumes:
  - name: config
    configMap:
      name: api-config
```

Edit `api-config` all you like. The string `api-config` in the template is unchanged, so the template is unchanged, so the Deployment does nothing. Kubernetes has no dependency link between the two objects.

## "It's mounted as a volume, it updates itself" does not save you

This is the objection that comes up every time, and it is a quarter true.

A ConfigMap mounted as a volume does get updated in the running pod. The kubelet syncs it on its own schedule: the total delay can be as long as the kubelet sync period, one minute by default, plus a cache propagation delay that depends on the kubelet's change detection strategy. So roughly a minute, sometimes two, and the new file appears under the mount path.

That is where the good news ends. The app has to notice. Most applications read their config file once at startup and never look at it again, so a file that changed underneath them is a file they will read on the next restart, which is the exact situation you were in already.

Two mount shapes never update at all. A `subPath` mount, the thing you reach for to put one file into a directory that already has other files in it, is a bind mount of the file as it was at container creation. The kubelet updates a ConfigMap volume by writing a new directory and swapping the volume's `..data` symlink to point at it, and the bind mount stays pointed at the old target, so the container never sees the new file. And a ConfigMap consumed as environment variables, through `envFrom` or `valueFrom.configMapKeyRef`, is read when the container is created and stays fixed for the container's lifetime. The docs are explicit that env vars require a pod restart.

So for a volume mount with an app that watches its files, the automatic update works. For everything else, and that is almost every app, a config change needs a restart.

## The checksum pattern

The Helm idiom is a hash of the rendered ConfigMap, placed in the pod template as an annotation.

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

Three pieces, each doing one job.

`$.Template.BasePath` is the namespaced path to the current chart's `templates/` directory, something like `api/templates`. The `$` matters inside a `range` or `with` block, where `.` has been rebound and `.Template` would not resolve. `print` appends the filename, so the argument becomes `api/templates/configmap.yaml`.

`include` takes that path and renders the template with the context you pass, `.` here, and returns the output as a string instead of writing it into the manifest. What you get back is the full ConfigMap YAML exactly as Helm is about to send it to the cluster, with every value substituted.

`sha256sum` hashes the string. The result is a 64-character hex digest, safe to drop into an annotation without quoting.

The consequence is mechanical. A value that feeds the ConfigMap changes, the rendered ConfigMap changes, the hash changes, the annotation in the pod template changes, the Deployment rolls. Nothing that feeds the ConfigMap changes, the hash is identical to last time, the pod template is byte-for-byte the same, nothing rolls. You have made the config part of the pod template's identity, which is the thing Kubernetes was missing.

It costs one line and depends on nothing outside Helm. It has been in the Helm docs under "Chart Development Tips and Tricks" for years and works on any Helm 3 or Helm 4 release.

## The complete example

A chart with one ConfigMap, one Deployment, the values that feed them, and a helpers file cut down to the two labels the example needs. Your chart's `helm create` scaffold emits more labels than this; that only shifts line numbers in the diff below, not what it shows.

```yaml
# values.yaml
image:
  repository: registry.example.com/api
  tag: "1.4.2"
config:
  logLevel: info
  upstreamTimeout: 5s
```

```yaml
# templates/_helpers.tpl
{{- define "app.fullname" -}}
{{ .Release.Name }}
{{- end -}}
{{- define "app.labels" -}}
app.kubernetes.io/name: api
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
{{- define "app.selectorLabels" -}}
app.kubernetes.io/name: api
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "app.fullname" . }}-config
  labels:
    {{- include "app.labels" . | nindent 4 }}
data:
  app.conf: |
    log_level = {{ .Values.config.logLevel }}
    upstream_timeout = {{ .Values.config.upstreamTimeout }}
    metrics_prefix = {{ .Release.Name }}
```

The `metrics_prefix` line is there on purpose. It comes from `.Release.Name`, not from values, and it is going to matter in the next section.

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          volumeMounts:
            - name: config
              mountPath: /etc/api
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: {{ include "app.fullname" . }}-config
```

Render it twice, once with the defaults and once with a changed value, and diff the output.

```bash
helm template api ./charts/api > before.yaml
helm template api ./charts/api --set config.logLevel=debug > after.yaml
diff -u before.yaml after.yaml
```

```diff
--- before.yaml
+++ after.yaml
@@ -10,7 +10,7 @@
     app.kubernetes.io/instance: api
 data:
   app.conf: |
-    log_level = info
+    log_level = debug
     upstream_timeout = 5s
     metrics_prefix = api
 ---
@@ -34,7 +34,7 @@
         app.kubernetes.io/name: api
         app.kubernetes.io/instance: api
       annotations:
-        checksum/config: cbddf2ae50da531b848f9d9ba64f3f18142cefc9077b401af6ffd1920531749e
+        checksum/config: b4a5298d915a31afe843a065ba7578092b33b07b673ce2930fad0526041ec723
     spec:
       containers:
         - name: api
```

Two hunks: the first is the ConfigMap data you changed, the second is the pod template annotation you did not touch, changed anyway because it is derived from the first. That second hunk is the rollout. The hash values depend on everything in the rendered ConfigMap, labels and helper output included, so yours will not match these; what matters is that the two differ from each other.

Run the first command twice with the same values and `diff` prints nothing. Same input, same rendered ConfigMap, same hash, and a `helm upgrade` with no config change leaves the pods alone.

## Two ways it goes wrong

Both produce a chart that looks right, passes `helm lint`, installs cleanly, and does not restart anything.

### The annotation is on the wrong object

A Deployment has two `metadata` blocks. The one at the top is the Deployment's own. The one under `spec.template` is the pod template's. The checksum has to be on the second.

```yaml
# Wrong. This is the Deployment's metadata.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
spec:
  template:
    metadata:
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
```

```yaml
# Right. This is the pod template's metadata.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
spec:
  template:
    metadata:
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

The wrong version is not a no-op. The hash changes, Helm patches the Deployment, `kubectl get deploy -o yaml` shows the new annotation, and everything looks like it worked. But the Deployment's own annotations are not part of `spec.template`, so the controller sees no template change and creates no ReplicaSet. This is the most common shape of "I added the checksum and it still does not restart", and the quickest way to confirm it is `kubectl rollout history deploy/<name>`: if the revision number did not move, the template did not change.

[NEED: one line on a checksum you found on the wrong object in a real chart, if there is one]

### It hashes the wrong thing

The other widespread version of the pattern is this:

```yaml
checksum/config: {{ toYaml .Values.config | sha256sum }}
```

It is shorter, it avoids the `$.Template.BasePath` incantation, and it is subtly incomplete. It hashes the values that go into the ConfigMap, not the ConfigMap. Anything the template computes from somewhere else is invisible to it. Look back at the example: `metrics_prefix = {{ .Release.Name }}` is in the ConfigMap and not in `.Values.config`. Rename the release and the ConfigMap changes while the hash does not. The same goes for a `tpl` expansion inside the ConfigMap, a `.Values.global` reference, a helper that reads `.Chart.AppVersion`, and, easy to forget, a change to the template file itself. Fix a typo in `configmap.yaml`, ship the chart, and `.Values.config` is the same as it was, so nothing rolls.

Hash the rendered template file. It is the only thing that captures every input.

There is one exception, and Helm's docs call it out: inside a library chart, `$.Template.BasePath` points at the consuming chart, not the library, so the file is not there. Hash the named template instead: `include "lib.configmap" . | sha256sum`.

The mirror-image mistake is hashing too much. If the ConfigMap template contains anything that changes on every render, the hash changes on every render, and the Deployment rolls on every `helm upgrade` whether or not anything changed. The usual culprits are `now`, `randAlphaNum`, `uuidv4`, and a `{{ .Release.Revision }}` someone added for "traceability". Under Argo CD it is worse than an unwanted rollout. Argo CD renders the chart with `helm template` on every comparison, so a value that regenerates each render is a permanent diff: the application is `OutOfSync` the moment a sync finishes, and with auto-sync on, every sync is another rollout. Argo CD's own docs describe exactly this for `randAlphaNum`. The ConfigMap has to be deterministic for the hash to mean anything.

## Secrets too

A chart-rendered Secret gets the same treatment, with its own annotation so the two do not collide:

```yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
  checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
```

A Secret mounted as a volume updates on the same kubelet schedule as a ConfigMap and has the same `subPath` and env var limits, so the case for restarting is identical.

The limit is that the checksum works because the chart renders the Secret and can therefore hash it. The moment the Secret comes from somewhere else, that is gone. A `database.existingSecret` value that names a Secret someone created by hand, a Secret managed by External Secrets Operator that syncs from Vault, a Sealed Secret decrypted by its controller: none of these pass through the chart's templates. There is nothing to `include`. A rotated credential lands in the Secret, the kubelet projects it into the volume within a minute or so, and the pod keeps using the connection it opened with the old password until it restarts for some other reason.

That gap is what the last option below is for. The checksum cannot close it.

## Immutable ConfigMaps as the alternative

The checksum pattern keeps one ConfigMap with a fixed name and mutates it in place. The other approach flips that: a new ConfigMap on every change, named after its content, and a pod template that references the new name.

```yaml
# templates/_helpers.tpl
{{- define "app.configData" -}}
app.conf: |
  log_level = {{ .Values.config.logLevel }}
  upstream_timeout = {{ .Values.config.upstreamTimeout }}
  metrics_prefix = {{ .Release.Name }}
{{- end }}

{{- define "app.configName" -}}
{{ include "app.fullname" . }}-config-{{ include "app.configData" . | sha256sum | trunc 8 }}
{{- end }}
```

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "app.configName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
immutable: true
data:
  {{- include "app.configData" . | nindent 2 }}
```

```yaml
# templates/deployment.yaml, the volume only; no checksum annotation
      volumes:
        - name: config
          configMap:
            name: {{ include "app.configName" . }}
```

The data lives in a named template rather than the file because the file's name now depends on a hash of the data, and you cannot hash a file whose output includes the hash. The named template is the content; the file is the wrapper.

The name change is the pod template change. No annotation, nothing to put on the wrong object. `immutable: true` has been stable since Kubernetes 1.21 and means the API server rejects any update to `data` or `binaryData`; to change it you delete and recreate, which the content-addressed name does for you.

Two benefits the mutable pattern does not have.

The first is load. The kubelet on every node that runs one of these pods keeps a watch open on the ConfigMap so it can refresh the volume. For an immutable ConfigMap the kubelet closes that watch, because nothing can change. The Kubernetes docs list this as the reason the feature exists: for clusters with tens of thousands of ConfigMap-to-pod mounts it significantly reduces load on the API server.

The second is rollback correctness. With the mutable pattern, `helm rollback` rewrites the single ConfigMap to the old content and the Deployment to the old checksum in the same pass. The ConfigMap change is instant; the Deployment change is a rollout. For the duration of the rollout, the pods that have not been replaced yet are running the new code with the old config file appearing under their mount path. Whether that hurts depends on whether the app re-reads the file, but the window exists on every upgrade and every rollback. With immutable ConfigMaps, the old pods keep the old ConfigMap and the new pods get the new one. Config and code move together, pod by pod, and there is no moment where a pod sees a config it was not started with.

Now the cost. Every config change leaves a ConfigMap behind. On the next `helm upgrade`, the previous ConfigMap is no longer in the rendered manifest, so Helm deletes it. That is the correct default and it has two consequences. Old pods still mid-rollout have it mounted and keep running, but any of them that gets rescheduled during the rollout fails to start on the missing ConfigMap. And `kubectl rollout undo`, which flips the Deployment to the previous ReplicaSet, points at a ConfigMap that no longer exists. `helm rollback` is fine, because the old revision's manifest still contains the old ConfigMap and Helm recreates it.

`helm.sh/resource-policy: keep` on the ConfigMap stops Helm deleting it, which makes `kubectl rollout undo` work and removes the mid-rollout hazard. It also means the ConfigMaps accumulate forever, one per distinct config, and Helm has forgotten about all but the current one. You own the cleanup: a label selector and a scheduled `kubectl delete`, or a policy of living with the orphans. Pick one before adopting the pattern rather than after the namespace fills up.

Kustomize does this natively. `configMapGenerator` appends a content hash suffix to the name and rewrites every reference to it, so the generated ConfigMap is new on every content change without any template work. Helm has no equivalent built in, which is why the checksum annotation is the Helm default and the immutable pattern is the one you reach for deliberately.

## Reloader for the cases the checksum cannot cover

Stakater's Reloader is a controller that watches ConfigMaps and Secrets and rolls the workloads that reference them. One annotation on the Deployment, on its own `metadata` this time and not the pod template's, because Reloader reads the Deployment and patches the template itself:

```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```

With that, any change to any ConfigMap or Secret the pod template references, through `envFrom` or a volume, triggers a rollout. It works on Deployments, StatefulSets, DaemonSets, and Argo Rollouts. It does not care where the change came from, which is the point: the Secret rotated by External Secrets Operator, the ConfigMap edited by hand, the Secret named in `existingSecret` that the chart never rendered, all of them roll the pod. `configmap.reloader.stakater.com/reload: "name"` and `secret.reloader.stakater.com/reload: "name"` narrow it to specific objects if `auto` is too broad. A Secrets Store CSI mount is not a Secret reference, so `auto` does not see it; that needs the controller run with `--enable-csi-integration=true` and `secretproviderclass.reloader.stakater.com/auto: "true"` on the workload.

```bash
helm repo add stakater https://stakater.github.io/stakater-charts
helm install reloader stakater/reloader
```

How it triggers the rollout is worth knowing. The default `--reload-strategy=env-vars` patches a dummy environment variable into every container that references the changed object, which is a pod template change. The alternative, `--reload-strategy=annotations`, adds a `reloader.stakater.com/last-reloaded-from` annotation to the pod template instead, and the README recommends it for Argo CD because it avoids unwanted sync diffs. Choose at install time, because it is a flag on the controller, not on the workload.

The first cost is that it is an operator, with RBAC to read every Secret in every namespace it watches, and someone has to keep it upgraded. The second is that restarts happen on someone else's schedule. A credential that Vault rotates at 3am is a rollout at 3am, for every workload referencing it, at once. `deployment.reloader.stakater.com/pause-period: "5m"` batches a burst of changes into one rollout, but it does not move the rollout to business hours. If that is not acceptable, the checksum pattern's property of only rolling when someone runs `helm upgrade` is a feature rather than a limitation.

### The lighter option: reload without restarting

For an app that can reload its own config on a signal, none of the above is necessary. Prometheus reloads on a `SIGHUP` or an HTTP `POST` to `/-/reload` when started with `--web.enable-lifecycle`, and if the new file is not well-formed the change is not applied, so a broken config is rejected rather than crashing the process.

The sidecar pattern: run `configmap-reload` next to the app, pointed at the same mount.

```yaml
      containers:
        - name: prometheus
          args:
            - --config.file=/etc/prometheus/prometheus.yml
            - --web.enable-lifecycle
          volumeMounts:
            - name: config
              mountPath: /etc/prometheus
        - name: config-reloader
          image: ghcr.io/jimmidyson/configmap-reload:v0.15.0
          args:
            - --volume-dir=/etc/prometheus
            - --webhook-url=http://localhost:9090/-/reload
          volumeMounts:
            - name: config
              mountPath: /etc/prometheus
              readOnly: true
```

The kubelet updates the mounted file within a minute or so, the sidecar sees the directory change and POSTs the reload endpoint, and the app picks up the new config with no pod restart and no connection drops. No checksum annotation on this Deployment, because a checksum would restart it, which is what you were trying to avoid. The constraint is the app: it has to support reloading, and the sidecar has to watch a volume mount, not a `subPath` and not env vars, for the same reasons as before.

`kiwigrid/k8s-sidecar`, which Grafana's chart uses for dashboards, works the other way round. It does not watch a mount. It watches the API for ConfigMaps and Secrets carrying a label, writes their contents into a directory itself, and calls a reload URL when they change. That fits config that is spread across many ConfigMaps discovered by label, where there is no single file to mount, rather than the one-file case above.

One thing to say out loud before adopting any of the restart-based patterns on a stateful service: with `strategy.type: Recreate`, which is what a single-replica service with a `ReadWriteOnce` volume is usually stuck with, Kubernetes kills every existing pod before creating the new one. Every checksum-triggered rollout is a short outage, so a config change has the same cost as a deploy. That is often fine. It should be a decision rather than a surprise.
