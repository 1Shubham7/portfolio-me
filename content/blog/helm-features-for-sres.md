---
title: "25 Helm Features You Didn't Know About That Make You a Better SRE"
description: "Twenty-five Helm flags, template functions, chart features and tools that most daily users never touch, each with the operational problem it solves. Written after two years of writing and managing Helm packages in production."
dateString: September 2026
draft: false
tags: ["Helm", "Kubernetes", "SRE", "DevOps", "GitOps", "Platform Engineering"]
weight: 1
cover:
    image: "/blog/helm-features-for-sres/cover.png"
---

Most people learn `helm install`, `helm upgrade`, and `values.yaml`, and stop. That is enough to get a chart onto a cluster. Helm has a second layer of features built for the people who then have to run that chart at scale: flags that make a bad upgrade fail safely, template functions that refuse to render garbage, and tooling that catches a broken chart before a cluster does. I learned this after writing and managing Helm packages in prod for two years, and it is the list I wish I had started with.

Minimum versions are given where they matter. Helm 4 renamed a few of these flags; where it did, both names are here.

## CLI flags you're not using

All of these are on `helm install` or `helm upgrade`, and none of them are the default.

### 1. `--dry-run=server`

Plain `--dry-run` renders templates without ever talking to the cluster, so it cannot tell you the manifest is invalid.

Since Helm 3.13, `--dry-run` takes a value. `client` is the old behaviour. `server` sends the rendered manifests to the API server as a Kubernetes dry-run request. You get real schema validation, your admission webhooks run against the objects, and `lookup` works because there is a cluster to look in. Helm 4 makes the three values explicit: `none`, `client`, `server`.

```bash
helm upgrade --install api ./charts/api -f prod.yaml --dry-run=server
```

It needs cluster credentials. A webhook that has not declared itself side-effect free will reject the dry-run request; better to find that out here than during the real one.

### 2. `--atomic` and `--cleanup-on-fail`

A failed upgrade leaves the release in `failed` state with a half-applied manifest and a human to sort it out.

`--atomic` rolls the release back to the last successful revision when the upgrade fails. For "fails" to mean anything, Helm has to wait for the rollout, so `--atomic` sets `--wait` automatically; without it an upgrade is "successful" the moment the API server accepts the objects. `--cleanup-on-fail`, on upgrade and rollback but not install, deletes resources that the failed operation newly created. Rollback alone does not, because the previous revision never had them. `--timeout` bounds the wait and defaults to `5m0s`. Set it longer than your slowest healthy rollout, or a slow rollout gets rolled back for being slow.

```bash
helm upgrade api ./charts/api -f prod.yaml --atomic --cleanup-on-fail --timeout 10m
```

Helm 4 renamed `--atomic` to `--rollback-on-failure`, which defaults `--wait` to the new `watcher` strategy. `--atomic` is deprecated on `helm upgrade`, and early Helm 4 releases rejected it outright on `helm install`.

### 3. `--wait-for-jobs`

`--wait` returns once Pods, PVCs, Services and the minimum replicas of each workload are ready, and a Job is none of those, so a data migration that is still running, or failing, does not hold up the release.

`--wait-for-jobs` (Helm 3.5) extends `--wait` to every Job in the manifest, within the same `--timeout`. Paired with `--atomic`, a failed Job now fails the upgrade and triggers the rollback instead of leaving a green release with a red Job next to it.

```bash
helm upgrade api ./charts/api -f prod.yaml --atomic --wait-for-jobs --timeout 15m
```

This is for Jobs in the regular manifest. Hook Jobs (item 20) already block on their own. Helm 4 keeps the flag, and `--wait` there takes a strategy: `--wait` alone means `watcher`, built on kstatus, the Kustomize status library; `legacy` is the Helm 3 behaviour; `hookOnly`, the default when the flag is omitted, waits for hooks and nothing else.

### 4. `--set-file`, `--set-json`, `--set-string`

`--set` types its argument, so `true` arrives in the template as a boolean and `20240101` as an int64, and a values file has the same problem one level up, where YAML reads `tag: 1.10` as the float 1.1 before any template sees it.

`--set-string` skips the typing: a numeric tag stays a string and `true` stays a word. Plain `--set` only ever types `true`, `false`, `null` and integers without a leading zero. `1.0` and `0123` are already strings there; it is the values file that needs `"1.10"` in quotes. `--set-file key=path` reads the file and uses its contents as the value: a certificate, or a config rendered in CI, without a values file. `--set-json` (Helm 3.10) takes a JSON literal, and a list of maps becomes one flag rather than a chain of `key[0].name=` entries.

```bash
helm upgrade api ./charts/api \
  --set-string image.tag=20240101 \
  --set-file tls.cert=./certs/tls.crt \
  --set-json 'tolerations=[{"key":"dedicated","operator":"Equal","value":"api","effect":"NoSchedule"}]'
```

`--set-json` values follow JSON rules: a bare string needs its own quotes inside the shell quotes, and a JSON number is a float64, so `--set-json 'version=1.0'` really does arrive as `1`.

### 5. `--reuse-values` vs `--reset-values`

"Keep the values from last time" means three different things depending on what you pass.

With no flags and no `-f` or `--set`, `helm upgrade` copies the previous release's user-supplied values forward. Pass any `-f` or `--set` and it does not merge: what you pass is now the whole set, and anything set last time and not passed this time reverts to the chart default. `--reuse-values` merges your new overrides on top of the previous release's values, but it also keeps the previous chart version's defaults, so defaults added by the new chart version are ignored. `--reset-values` discards the old values. `--reset-then-reuse-values` (Helm 3.14) is what most people think `--reuse-values` does: new chart defaults, then the last release's values, then your overrides.

```bash
helm get values api -o yaml            # what is actually set right now
helm upgrade api ./charts/api --version 2.0.0 --reset-then-reuse-values -f prod.yaml
```

`--reuse-values` in a GitOps flow means the values in git have stopped being the truth.

[NEED: one line on a value that silently reverted on you after an upgrade, if there is one]

### 6. Reading the release record

Guessing what is deployed from a values file in git, when the cluster may disagree.

Helm stores every revision as a Secret in the release namespace, and `helm get` reads it. `helm get values` shows the user-supplied values; `--all` shows the computed values after chart defaults were merged, which is what the templates actually saw. `helm get manifest` is the YAML Helm applied, `helm get hooks` the hook resources, and `--revision N` on any of them reads an older revision. `helm history` lists revisions with status and chart version. `--history-max` (default 10) caps how many revisions are kept, and `helm uninstall --keep-history` retains the record so `helm history` and `helm rollback` still work after an uninstall.

```bash
helm history api
helm get values api --revision 12 --all
helm get manifest api --revision 12 | kubectl diff -f -
```

`helm get manifest` is what Helm applied, not what is in the cluster now; anything edited by hand or by a controller is not in it. helm diff's `--three-way-merge` (item 25) closes that gap.

### 7. `helm template --validate`, `--api-versions`, `--kube-version`

`helm template` renders against a fake cluster: `.Capabilities.KubeVersion` is whatever Helm was built with and `.Capabilities.APIVersions` is a built-in list, so any template that branches on either renders the wrong branch offline.

`--kube-version 1.30.0` sets the version. `--api-versions monitoring.coreos.com/v1/ServiceMonitor`, repeatable, adds to the list that `.Capabilities.APIVersions.Has` checks. `--validate` goes further and validates the rendered manifests against the cluster you are pointed at, the same validation `install` does. ArgoCD renders everything with `helm template` and never runs `helm install`. It passes `--kube-version` and `--api-versions` from the destination cluster automatically; that is the only reason capability branches work there. `spec.source.helm.kubeVersion` and `apiVersions` override what it detects.

```bash
helm template api ./charts/api -f prod.yaml \
  --kube-version 1.30.0 \
  --api-versions monitoring.coreos.com/v1/ServiceMonitor \
  --validate
```

`--validate` needs cluster access, so it turns offline rendering into a round trip. Helm 4 deprecates it in favour of `helm template --dry-run=server`, the same round trip under the item 1 name. `--api-versions` adds to the built-in list rather than replacing it.

### 8. `--post-renderer`

A third-party chart needs one patch, a sidecar or a label or a `securityContext`, and forking the chart to add it means maintaining the fork forever.

Helm renders the chart, pipes the complete manifest to the post-renderer's stdin, and installs whatever comes back on stdout. The usual choice is Kustomize: a `kustomization.yaml` with the patch, and a two-line script that writes stdin to a file and runs `kustomize build`. `--post-renderer-args` (Helm 3.9) passes arguments. The release record stores the patched output; `helm get manifest` and `helm diff` see the patch too.

```bash
#!/bin/sh
# kustomize-post-renderer
cat > all.yaml
kustomize build .
```

```bash
helm upgrade --install grafana grafana/grafana -f prod.yaml --post-renderer ./kustomize-post-renderer
```

Helm 4 changed this: `--post-renderer` takes the name of a `postrenderer` plugin, not an executable path. The script above has to be wrapped as a plugin. Either way, the patch is invisible to anyone reading the chart. Keep the script next to the values file it belongs to.

## Template functions that do real work

These run at render time, and most of them exist so a bad values file fails before it reaches the API server.

### 9. `required` and `fail`

A missing value renders as an empty string, and the result is a Deployment with `image: :latest` that Kubernetes rejects with an error nowhere near the cause.

`required "msg" .Values.x` aborts the render with your message when the value is empty. `fail "msg"` aborts unconditionally, so you wrap it in the condition `required` cannot express: two values that must agree, say, or a size below a floor.

```yaml
image: "{{ .Values.image.repository }}:{{ required "image.tag must be set; this chart does not default to latest" .Values.image.tag }}"
{{- if and .Values.persistence.enabled (not .Values.persistence.storageClass) }}
{{- fail "persistence.enabled requires persistence.storageClass" }}
{{- end }}
```

`required` only fails on nil and the empty string. `0` and `false` pass it. If those are wrong for your field, write the check yourself and `fail`.

### 10. `lookup`

The chart needs something that already exists in the cluster, a generated password in a Secret say, without the user copying it into values.

`lookup "v1" "Secret" .Release.Namespace "db-creds"` returns the object as a dict at render time, or an empty dict if it is not there, so `if $s` branches on existence. The classic use is a credential generated on first install and reused on every upgrade instead of regenerated. Checking that a namespace exists, or reading a ConfigMap another chart wrote, is the same shape.

```yaml
{{- $existing := lookup "v1" "Secret" .Release.Namespace "db-creds" }}
{{- if $existing }}
password: {{ index $existing.data "password" }}
{{- else }}
password: {{ randAlphaNum 32 | b64enc }}
{{- end }}
```

Under `helm template` and client-side `--dry-run`, `lookup` returns an empty dict. ArgoCD renders with `helm template`, so under ArgoCD the snippet above regenerates the password on every sync. `--dry-run=server` (item 1) is the only dry run where it works.

### 11. `tpl`

Users want to reference `.Release.Name` or another value inside a values file, and values are plain strings.

`tpl` renders a string as a template with the context you pass. `{{ tpl .Values.ingress.host . }}` turns a value of `"{{ .Release.Name }}.{{ .Values.global.domain }}"` into a real hostname. The same function renders config files shipped in `files/` that contain template syntax: `tpl (.Files.Get "config.toml") .`.

```yaml
# values.yaml
ingress:
  host: "{{ .Release.Name }}.{{ .Values.global.domain }}"
```

```yaml
# templates/ingress.yaml
- host: {{ tpl .Values.ingress.host . | quote }}
```

`tpl` parses the string on every call, so it is slow inside loops, and a value the user never meant as a template (a `{{` inside a regex or a Prometheus query) is now one.

### 12. `dig`, `hasKey`, `kindIs`

`.Values.a.b.c` panics the render with a nil pointer when `a` or `b` is unset, and `if .Values.replicas` cannot tell "unset" from "set to 0".

`dig "a" "b" "c" "default" $map` (Helm 3.5) walks the path and returns the default if any key is missing, no panic. `hasKey $map "replicas"` answers "was it set at all", which is the only way to honour an explicit `0` or `false`. `kindIs "string" .Values.x` (or `"map"`, `"slice"`) lets one key accept either a string or a structured block.

```yaml
{{- $cpu := dig "resources" "limits" "cpu" "500m" .Values.AsMap }}
{{- if hasKey .Values "replicas" }}
replicas: {{ .Values.replicas }}
{{- end }}
{{- if kindIs "string" .Values.config }}
{{ .Values.config | nindent 2 }}
{{- else }}
{{ toYaml .Values.config | nindent 2 }}
{{- end }}
```

`dig` type-asserts a plain map, and `.Values` is Helm's own `chartutil.Values` type, so `dig ... .Values` fails with an interface conversion error. Pass `.Values.AsMap`, or a nested map like `.Values.resources`, which is plain.

### 13. The config checksum annotation

Changing a ConfigMap does not restart the pods that mount it, so the config is updated and nothing reads it.

Put a hash of the rendered ConfigMap into the pod template's annotations. Any change to the ConfigMap changes the hash, which changes the pod template, which triggers a rollout. Hashing the rendered template file via `include (print $.Template.BasePath "/configmap.yaml") .` catches everything that affects the output, including changes to the template itself; hashing `.Values.config` only catches value changes.

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

`$.Template.BasePath` is the current chart's path, so inside a library chart (item 18) hash the named template instead: `include "lib.configmap" . | sha256sum`. And anything in the ConfigMap that changes on every render, a timestamp or a `randAlphaNum`, rolls the Deployment on every upgrade.

[NEED: one line on a config change that shipped and did not roll the pods, if there is one]

### 14. `.Capabilities.APIVersions.Has` and `semverCompare` on `.Capabilities.KubeVersion`

A chart that emits a ServiceMonitor on a cluster without the Prometheus Operator fails the install with `no matches for kind`.

`.Capabilities.APIVersions.Has "monitoring.coreos.com/v1/ServiceMonitor"` is true only if the cluster serves that resource. Wrap the manifest in it. `.Capabilities.KubeVersion.Version` is the cluster version, and `semverCompare` against it picks an `apiVersion` or a field per cluster generation.

```yaml
{{- if .Capabilities.APIVersions.Has "monitoring.coreos.com/v1/ServiceMonitor" }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
# ...
{{- end }}
{{- if semverCompare ">=1.29.0-0" .Capabilities.KubeVersion.Version }}
# field that exists from 1.29
{{- end }}
```

The `-0` suffix is not decoration. Managed clusters report versions like `v1.30.5-gke.1234` or `v1.30.2-eks-abc123`, semver reads the suffix as a prerelease, and a constraint without `-0` never matches one. Offline, both checks are only as right as what you passed to `--kube-version` and `--api-versions` (item 7).

### 15. `.Release.IsInstall` / `.Release.IsUpgrade`

Some things should happen once, on first install: seed a credential, say, or run a bootstrap Job.

`.Release.IsInstall` is true during `helm install`; `.Release.IsUpgrade` is true during upgrade and also during rollback. Wrap the one-time resource in the branch. Combined with `lookup` and `helm.sh/resource-policy: keep`, this is how you generate a credential on install and never touch it again.

```yaml
{{- if .Release.IsInstall }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-bootstrap
# ...
{{- end }}
```

`helm template` sets `IsInstall` unless you pass `--is-upgrade`, and ArgoCD always renders as an install, so under ArgoCD an `IsInstall` branch is live on every sync. A resource that appears only in the install branch is absent from the next upgrade's manifest. Helm deletes it then, unless it carries `resource-policy: keep`.

## Chart structure

Things that live in the chart itself and change how it behaves for everyone who installs it.

### 16. `values.schema.json`

A typo'd key (`replicaCount` where the chart reads `replicas`) is silently ignored, and the chart deploys with the default.

A JSON Schema file at the chart root is validated against the final `.Values`, after defaults and overrides are merged, on `helm install`, `helm upgrade`, `helm lint` and `helm template`. Types catch `replicas: "3"`, `enum` catches an unsupported mode, `required` catches a missing key, and `additionalProperties: false` catches the typo, because an unknown key is now an error instead of a no-op.

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "additionalProperties": false,
  "required": ["image"],
  "properties": {
    "global": { "type": "object" },
    "replicas": { "type": "integer", "minimum": 1 },
    "image": {
      "type": "object",
      "required": ["tag"],
      "properties": { "repository": { "type": "string" }, "tag": { "type": "string" } }
    },
    "logLevel": { "enum": ["debug", "info", "warn", "error"] }
  }
}
```

Helm injects `global` into every chart's values, and a parent chart's values contain a key per subchart. With `additionalProperties: false` you must list `global` and every dependency name under `properties`, or the schema rejects a perfectly valid install. `--skip-schema-validation` is the escape hatch when a third-party chart gets this wrong.

### 17. `helm.sh/resource-policy: keep`

`helm uninstall` deletes the PVC holding the database, and an upgrade that drops a resource from the chart deletes it from the cluster too.

The annotation tells Helm to skip deleting the resource in any operation that would otherwise remove it: uninstall, an upgrade where the resource left the chart, a rollback to a revision that did not have it. The object stays, and because Helm's ownership annotations are still on it, a reinstall under the same release name adopts it. Put it on PVCs, on Secrets holding generated credentials, on anything whose loss is not recoverable from git.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ .Release.Name }}-data
  annotations:
    helm.sh/resource-policy: keep
```

"Kept" means orphaned. Helm forgets about it, so an uninstall followed by an install under a different name leaves the old PVC behind for someone to find on a bill.

[NEED: one line on a PVC you lost, or found orphaned, if there is one]

### 18. Library charts, `alias`, `condition`, `tags`, and `global`

Forty charts each with their own copy of `_helpers.tpl`, and a bundled database you cannot turn off.

`type: library` in `Chart.yaml` makes a chart that renders nothing and is not installable, but whose named templates are available to any chart that lists it as a dependency: one definition of `common.labels`, forty consumers, no copy-paste. On the dependency side, `condition: postgresql.enabled` turns a subchart off from the parent's values; `tags` does the same for a group of subcharts at once; `alias` lets you list one chart twice under different names: two Redis instances from one chart. `global` values are the one channel that reaches every subchart without being addressed to it by name.

```yaml
# Chart.yaml
dependencies:
  - name: common
    version: 1.x.x
    repository: oci://registry.example.com/charts
  - name: redis
    version: 19.x.x
    repository: oci://registry.example.com/charts
    alias: redis-cache
    condition: redis-cache.enabled
  - name: postgresql
    version: 15.x.x
    repository: oci://registry.example.com/charts
    condition: postgresql.enabled
    tags: [database]
```

`condition` wins over `tags` when both are set. A subchart sees only `.Values.<its-name>` and `.Values.global`; nothing else from the parent crosses the boundary.

### 19. `NOTES.txt` as an operational message

The person who ran `helm install` does not know how to reach the thing, or that they installed it with persistence off.

`templates/NOTES.txt` is rendered with full template access and printed after install and upgrade, and by `helm get notes` later. It can compute the real URL from `.Values.ingress` and print the `kubectl port-forward` command with the actual release name and namespace. More usefully, it can warn: persistence is disabled, or the default password is in use.

```
{{- if not .Values.persistence.enabled }}
WARNING: persistence.enabled is false. All data is lost when the pod restarts.
{{- end }}

Connect with:
  kubectl -n {{ .Release.Namespace }} port-forward svc/{{ include "app.fullname" . }} 8080:80
```

Nobody sees NOTES under ArgoCD, or in a CI log that scrolls past. It is for a human at a terminal, not a substitute for `fail`.

## Lifecycle: hooks and tests

Two features that let the chart run something, in order, and block on it.

### 20. Hooks with `hook-weight` and `hook-delete-policy`

The migration has to finish before the new pods start, and the second run of the same Job fails because a Job's pod template is immutable once created.

`helm.sh/hook: pre-upgrade` (or `pre-install`, `post-install`, `pre-delete`, and the rest) runs a resource at that point in the lifecycle. For a Job or Pod, Helm waits for it to complete and fails the release if it fails. That is the blocking a migration needs. `hook-weight` orders multiple hooks, ascending. `hook-delete-policy` says when to remove the hook resource. `before-hook-creation`, the default, deletes the previous instance before creating the new one, and that is the immutable-Job collision solved. `hook-succeeded` deletes it after success; a failed Job stays behind for `kubectl logs`.

```yaml
metadata:
  name: {{ .Release.Name }}-migrate
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-weight: "-5"
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
```

Hooks are not part of the release manifest, so an `--atomic` rollback does not undo what a hook did. ArgoCD maps Helm hooks onto its own sync phases with its own semantics; check what it does with yours.

### 21. `helm test` and the `test` hook

The install says `deployed` and nothing has checked whether the service answers.

A Pod in `templates/tests/` with `helm.sh/hook: test` is not created on install. `helm test <release>` creates it, waits for it to exit, and reports pass or fail on the exit code. The `helm create` scaffold gives you a busybox `wget` at the Service, which proves the Service exists. The one worth writing proves the deploy worked: a `psql` against the bundled database, or a request at a real endpoint with the credential the chart generated. `--logs` prints the pod's output; you want that in CI.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-smoke
  annotations:
    helm.sh/hook: test
    helm.sh/hook-delete-policy: hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: smoke
      image: curlimages/curl
      envFrom:
        - secretRef:
            name: {{ .Release.Name }}-api-token
      command: ["curl", "-fsS", "-H", "Authorization: Bearer $(API_TOKEN)",
                "http://{{ include "app.fullname" . }}:{{ .Values.service.port }}/v1/me"]
```

It only runs when something runs `helm test`. `ct install` does (item 24); your deploy pipeline does not unless you add the line.

## Testing and quality tooling

None of these need a cluster except one, and all of them belong in CI.

### 22. `helm lint --strict` plus kubeconform

`helm lint` checks the chart's structure and that the templates render. It does not know that `spec.replica` is not a field, or that `batch/v1beta1` CronJob is gone in 1.25.

`helm lint --strict` fails on warnings rather than only errors, and `--kube-version` runs the deprecated-API check for a specific version. kubeconform takes rendered manifests on stdin and validates every object against the real Kubernetes JSON schemas for the version you name, offline, with `-strict` rejecting unknown fields. For CRDs, point `-schema-location` at a schema catalog.

```bash
helm lint ./charts/api --strict -f ci/prod-values.yaml

helm template api ./charts/api -f ci/prod-values.yaml \
  | kubeconform -kubernetes-version 1.30.0 -strict -summary \
      -schema-location default \
      -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
```

kubeconform validates shape, not admission. A valid manifest can still be refused by a policy webhook; `--dry-run=server` (item 1) is the check for that.

### 23. helm-unittest

A chart with twenty `if` branches has twenty ways to render wrong, and the only test is installing it.

A Helm plugin. Test files in `tests/*_test.yaml` name a template, set values, and assert on paths in the rendered output. `failedTemplate` asserts that the render fails with a given message. The `required` and `fail` guards from item 9 get tested rather than hoped for. A `capabilities` block sets the fake Kubernetes version and API versions; the `.Capabilities` branches from item 14 are testable too. It renders locally and runs in seconds. No cluster.

```yaml
suite: deployment
templates:
  - deployment.yaml
tests:
  - it: sets replicas from values
    set:
      replicas: 3
    asserts:
      - equal:
          path: spec.replicas
          value: 3
  - it: refuses persistence without a storage class
    set:
      persistence.enabled: true
    asserts:
      - failedTemplate:
          errorMessage: persistence.enabled requires persistence.storageClass
```

```bash
helm plugin install https://github.com/helm-unittest/helm-unittest.git
helm unittest ./charts/api
```

It tests the template, not the cluster, so it sits next to `ct install`, not in place of it.

### 24. chart-testing with `ci/*-values.yaml`

The chart only ever gets installed with default values, and the PR that changed the templates forgot to bump the version.

`ct lint` runs `helm lint`, yamllint, and a `Chart.yaml` schema check, and refuses a changed chart whose version was not incremented against the target branch. `ct install` does a real `helm install` and `helm test` for each changed chart, once per file matching `ci/*-values.yaml` in the chart directory. Each non-default path (persistence on, ingress on, HA mode) is actually installed. `--upgrade` also tests the upgrade from the previous chart version. The standard pairing in a GitHub workflow is `helm/kind-action` for a throwaway cluster and `helm/chart-testing-action` for `ct`.

```
charts/api/ci/
  default-values.yaml
  ingress-values.yaml
  ha-values.yaml
```

```bash
ct lint --target-branch main
ct install --target-branch main --upgrade
```

A kind cluster has no cloud storage class, no load balancer, and none of your admission webhooks, so what passes there is "installs and the test pod exits 0", not "works in prod".

### 25. helm-docs and helm diff

The README says one thing and `values.yaml` says another, and nobody knows what the upgrade will change until it has.

helm-docs generates the values table in the README from comments in `values.yaml`: a `# --` comment above a key is its description, and `# @default --` overrides the default shown. Run it as a pre-commit hook and the README cannot drift. helm-diff is a plugin: `helm diff upgrade` renders the chart with your values and diffs it against the deployed release's manifest, before you run the upgrade. `helm diff rollback` and `helm diff revision` do the same for those operations. `--three-way-merge` diffs against what is actually in the cluster rather than the stored manifest, and catches hand edits.

```yaml
# -- Number of API replicas. Ignored when autoscaling.enabled is true.
replicas: 2
# -- Storage class for the data PVC. Required when persistence.enabled is true.
# @default -- cluster default
storageClass: ""
```

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade api ./charts/api -f prod.yaml --three-way-merge
```

helm-diff renders offline by default, so `lookup` is empty in it unless you pass `--dry-run=server`, the same limit as item 10.

## Links

- [helm install](https://helm.sh/docs/helm/helm_install/) and [helm upgrade](https://helm.sh/docs/helm/helm_upgrade/) flag reference
- [Template functions and pipelines](https://helm.sh/docs/chart_template_guide/functions_and_pipelines/), including `lookup`
- [Chart hooks](https://helm.sh/docs/topics/charts_hooks/) and [chart tests](https://helm.sh/docs/topics/chart_tests/)
- [Chart tips and tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/), where the checksum annotation and `resource-policy` live
- [Argo CD's Helm docs](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/), for what `helm template` means there
- [kubeconform](https://github.com/yannh/kubeconform), [helm-unittest](https://github.com/helm-unittest/helm-unittest), [chart-testing](https://github.com/helm/chart-testing), [helm-docs](https://github.com/norwoodj/helm-docs), [helm-diff](https://github.com/databus23/helm-diff)
