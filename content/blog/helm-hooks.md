---
title: "Helm Hooks: What They Actually Do, When to Use Them, and When Not To"
description: "What Helm does with a hook step by step, the three things hooks do not do that everyone assumes they do, a migration Job that survives a rollback, and when an initContainer or an Argo CD sync wave is the better tool. Written after two years of writing and managing Helm packages in production."
dateString: September 2026
draft: false
tags: ["Helm", "Kubernetes", "SRE", "DevOps", "GitOps", "ArgoCD", "Platform Engineering"]
weight: 1
cover:
    image: "/blog/helm-hooks/cover.png"
---

Hooks are the most copy-pasted and least understood part of Helm. Most people have a migration hook that works, and keeps working, until the day a rollback does not. Here is the full model, from what Helm actually does when it sees the annotation to what Argo CD does with the same YAML.

## The mental model

Without hooks, Helm renders every template, sorts the resulting objects by kind (Namespace and the policy kinds at the front, then ServiceAccount, Secret, ConfigMap, and so on down to Deployment, Job, and Ingress), and sends them to the API server one after another. There is no waiting between them. A Job in `templates/` is created in the same pass as the Deployment that needs it, a couple of API calls later, and the two race.

A hook is a resource you pull out of that pass. The `helm.sh/hook` annotation tells Helm to hold the object back, create it at a named moment in the release lifecycle, and wait for it before doing anything else. The moments are `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`, `pre-rollback`, `post-rollback`, and `test`, which only fires under `helm test`. One object can carry several, comma-separated, and a migration Job usually carries two: `pre-install,pre-upgrade`.

That is the whole feature. Everything else is detail about the waiting and the cleanup.

## What Helm does with a hook, step by step

Take a Job annotated `helm.sh/hook: pre-upgrade` and run `helm upgrade`.

1. Helm renders every template, then splits the output. Objects with a hook annotation go into the release's hook list; everything else becomes the release manifest.
2. If an object with the hook's name and kind already exists in the cluster, Helm deletes it and waits for the deletion. That is the default delete policy, covered below.
3. Helm creates the hook and waits for it to complete.
4. Helm applies the main manifest: the three-way merge between the previous manifest, the new one, and the live objects.
5. `post-upgrade` hooks run after that, and after `--wait` if you passed it.

"Complete" depends on the kind. A Job is complete when its `Complete` condition is true, and the hook fails the moment a `Failed` condition appears. A Pod is complete when its phase is `Succeeded`, failed at `Failed`. Any other kind is complete as soon as the API server has accepted it: a Secret or ConfigMap hook is done the instant it exists, and Helm's legacy waiter says in a comment that for kinds like ReplicaSet it does nothing special, so do not expect a Deployment hook to wait for pods. Each hook's wait is bounded by `--timeout`, default `5m0s`.

If the hook fails, the release is marked `failed` and step 4 never happens. The old pods keep serving, the old manifest is untouched, and `helm history` shows a failed revision sitting on top of the deployed one. With `--atomic` (`--rollback-on-failure` in Helm 4) Helm rolls back to the previous revision, which here changes nothing in the cluster, because nothing had changed yet.

That block-and-abort behaviour is the entire reason hooks exist. A plain Job in `templates/` is created alongside the Deployment with no ordering, and if it fails, the release is still `deployed` unless you passed `--wait-for-jobs`. A hook Job runs first, on its own, and a failure stops everything behind it.

## Ordering with `helm.sh/hook-weight`

Two hooks at the same point run in weight order. The weight is a string holding an integer, negatives allowed, ascending, `"0"` when unset. Helm creates the lowest weight first and waits for it to complete before creating the next.

```yaml
# templates/wait-for-db.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "app.fullname" . }}-wait-for-db
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-weight: "-5"
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  backoffLimit: 20
  activeDeadlineSeconds: 240
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: wait
          image: postgres:16
          command: ["pg_isready", "-h", "{{ .Values.database.host }}", "-p", "5432", "-t", "5"]
```

```yaml
# templates/migrate.yaml, annotations only; the full Job is further down
metadata:
  name: {{ include "app.fullname" . }}-migrate
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-weight: "0"
```

The `-5` Job retries `pg_isready` until the database answers, then the `0` Job migrates it. If the database never answers, the wait Job hits its deadline, the hook fails, and the migration is never created.

One thing the docs leave implicit: hooks with the same weight do not run in parallel. Helm sorts the list, weight first, and runs it one hook at a time. Equal weights mean you did not care about the order, not that they overlap.

## `helm.sh/hook-delete-policy`, explained properly

The annotation says when Helm deletes the hook resource. Three values, combinable with commas.

`before-hook-creation` is what you get when you set nothing. Before creating the hook, Helm deletes any existing object with the same name and kind, then waits for it to be gone. This matters because Helm creates hooks, it does not patch them, and a Job's pod template is immutable after creation anyway. Without this policy the second upgrade fails with `jobs.batch "api-migrate" already exists`, and if Helm did try to update the Job in place, the API server would refuse the new image tag with `field is immutable`. Either way the leftover Job has to go before the next one can exist.

`hook-succeeded` deletes the hook after it completes. `hook-failed` deletes it after it fails.

The combination I use everywhere is `before-hook-creation,hook-succeeded`. A successful migration cleans up after itself. A failed one stays, so `kubectl logs job/api-migrate` still works and `kubectl describe` still shows why the pod died. The next upgrade attempt deletes the failed Job first, then creates a fresh one. You get evidence when you need it and nothing lying around when you do not.

Never use `hook-failed` on its own. The one case where you need the Job's logs is the one case where that policy has already deleted them. Helm 3.18 added `helm.sh/hook-output-log-policy: hook-failed`, which prints a failed Job's or Pod's logs in the `helm` output before any cleanup. Older Helm 3 releases ignore the annotation, so it does not make `hook-failed` safe there.

Setting the annotation replaces the default rather than adding to it. `hook-delete-policy: hook-succeeded` alone means no `before-hook-creation`, and the next run after a failure collides with the failed Job.

## The three things hooks don't do

This is the part the copy-pasted Job does not tell you.

### Hooks are not part of the release

The release record stores the rendered hook YAML, which is what `helm get hooks` prints, but the live objects a hook creates are not tracked or managed as part of the release. The Helm docs say this in so many words. The release manifest does not contain them. The three-way merge on upgrade never sees them. `helm uninstall` runs your `pre-delete` and `post-delete` hooks and deletes everything in the manifest, and a hook Job still sitting in the namespace because it failed last Tuesday is not in the manifest, so it stays.

Cleanup is entirely on you: a delete policy, or `ttlSecondsAfterFinished` on the Job, or both.

### `helm rollback` does not undo what a hook did

Rollback runs the target revision's `pre-rollback` hooks, applies the target revision's manifest over the current one, runs its `post-rollback` hooks, and marks the release `deployed`. It does not run anyone's `pre-upgrade` hook, and it has no concept of reversing one.

So the sequence that bites is: the `pre-upgrade` migration runs and succeeds, the new Deployment fails to roll out, `--atomic` or a human runs a rollback, and the old pods come back in front of a schema that has already moved on. Nothing in Helm complains. Nothing can, because Helm has no idea what the Job did.

Two ways out. The one that works is making every migration that runs from a hook backward-compatible: expand first (add the column, add the table, backfill), ship the code that uses it, and contract (drop the old column) in a later release, once nothing old can come back. The other is a `pre-rollback` hook that runs the down migration, which almost nobody writes, because a down migration for anything that touched data is mostly fiction. Design for the first.

[NEED: one line on a rollback that put the old app in front of a migrated schema, if there is one]

### `pre-upgrade` runs while the old pods are still serving

Look at the step list again. The hook completes before the manifest is applied, which is before the new Deployment exists, which means the previous version handles traffic for the whole duration of the migration. The migration runs against a live database, under load, from code that does not know about the change being made under it.

This is the rollback constraint seen from the other side. An `ALTER TABLE` that takes a lock the old version cannot tolerate takes the site down before the new version is even created. Expand/contract covers this too, and so does knowing what your database does on each statement: on PostgreSQL 11 and later, `ADD COLUMN` with a constant default is a catalog change; a column type change rewrites the table under a lock.

## A migration Job done right

Every line has a job. The comments say which.

```yaml
# templates/migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  # A fixed name, not generateName: before-hook-creation needs a stable name to find and delete.
  name: {{ include "app.fullname" . }}-migrate
  labels:
    {{- include "app.labels" . | nindent 4 }}
  annotations:
    # pre-install so a fresh install gets a schema; pre-upgrade so every upgrade migrates first.
    helm.sh/hook: pre-install,pre-upgrade
    # After the -5 wait-for-db Job, before anything at 1 or higher.
    helm.sh/hook-weight: "0"
    # Clear last run's Job before creating this one; delete on success; keep on failure for kubectl logs.
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
    # Helm 3.18 and later: print the failed Job's logs in the helm output. Older releases ignore it.
    helm.sh/hook-output-log-policy: hook-failed
spec:
  # One retry covers a dropped connection. More than that is retrying a real failure.
  backoffLimit: 1
  # Kill a stuck migration and fail the Job, so the upgrade fails cleanly instead of hanging
  # until Helm's --timeout. Keep this below --timeout, or Helm gives up first and you get a
  # timeout error instead of the Job's own failure reason.
  activeDeadlineSeconds: 240
  template:
    metadata:
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
    spec:
      # Never: a failed pod is a failed attempt, counted by backoffLimit, and its logs stay readable.
      restartPolicy: Never
      # Same ServiceAccount as the app: same RBAC, same workload identity towards the database.
      serviceAccountName: {{ include "app.serviceAccountName" . }}
      # Same pull secrets, or the image the Deployment can pull is one the Job cannot.
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: migrate
          # Same repository and tag as the Deployment, from the same values, so the migration
          # and the code that needs it are always the same build.
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          command: ["/app/migrate", "up"]
          env:
            # From the same values the Deployment's ConfigMap is rendered from, not from the
            # ConfigMap itself: on a fresh install that ConfigMap does not exist yet when
            # pre-install runs, and on an upgrade it is still the previous release's copy.
            - name: DB_HOST
              value: {{ .Values.database.host | quote }}
            - name: DB_NAME
              value: {{ .Values.database.name | quote }}
          envFrom:
            # A Secret that exists before the chart does, the same one the Deployment mounts.
            # Never a second copy of the credentials rendered just for the Job.
            - secretRef:
                name: {{ required "database.existingSecret is required" .Values.database.existingSecret }}
          # Omitted entirely when migration.resources is unset, rather than rendering resources: null.
          {{- with .Values.migration.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
```

The ConfigMap comment is the one people get wrong. A `pre-install` hook runs before the manifest, so anything the chart itself creates, a ConfigMap or a Secret rendered from values, does not exist yet on first install. Referenced through `env` or `envFrom`, as above, the pod sits in `CreateContainerConfigError` until the deadline kills it; mounted as a volume, it sits in `ContainerCreating` with a `FailedMount` event instead. On upgrade it is worse in a quieter way: the object exists, but it is last release's version. A hook may only reference what existed before the chart did, or what it rendered from values itself.

`--wait-for-jobs` is not needed for any of this. Helm always waits on a hook Job; `--wait` and `--wait-for-jobs` are about the main manifest. In Helm 4, where `--wait` takes a strategy, the default when the flag is omitted is `hookOnly`, which is exactly this: wait for hooks, nothing else.

## Three more uses beyond migrations

### Seed an admin user once the app is up

The app's API is the only supported way to create a user, and it needs the app running.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "app.fullname" . }}-seed-admin
  annotations:
    helm.sh/hook: post-install
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  backoffLimit: 6
  activeDeadlineSeconds: 240
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: seed
          image: curlimages/curl
          envFrom:
            - secretRef:
                name: {{ .Values.admin.existingSecret }}
          command: ["sh", "-c"]
          args:
            - >-
              curl -fsS -X POST
              http://{{ include "app.fullname" . }}:{{ .Values.service.port }}/api/v1/users
              -H "Content-Type: application/json"
              -d "{\"email\":\"$ADMIN_EMAIL\",\"password\":\"$ADMIN_PASSWORD\",\"role\":\"admin\"}"
```

`post-install` runs after the manifest is applied, but "applied" means accepted by the API server, not ready. Pass `--wait` and Helm holds the hook until the Deployment is ready; without it, `backoffLimit: 6` is what gives the Service time to get an endpoint. It is `post-install` only, not `post-upgrade`, because the admin should exist once. Argo CD will run it on every sync anyway, so a `409 Conflict` from the API should count as success. More on that below.

### Deregister something external before delete

The app registered itself somewhere outside the cluster: a DNS record, a service-registry entry. Uninstalling the chart should unregister it.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "app.fullname" . }}-deregister
  annotations:
    helm.sh/hook: pre-delete
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  backoffLimit: 2
  activeDeadlineSeconds: 120
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: {{ include "app.serviceAccountName" . }}
      containers:
        - name: deregister
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["/app/deregister"]
          envFrom:
            - secretRef:
                name: {{ .Values.registry.existingSecret }}
```

`pre-delete`, not `post-delete`. At `pre-delete` the manifest is still in the cluster: the ServiceAccount and its RoleBinding, the chart's Secrets, the running pods. At `post-delete` all of it is gone, and a Job that references any of it fails on a missing object, leaving you with a `failed` uninstall and an orphaned DNS record. A `pre-delete` hook that fails also fails the uninstall before anything is removed, which is the behaviour you want from a step called "clean up after yourself".

### `helm test` as a smoke test that ships with the chart

```yaml
# templates/tests/smoke.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "app.fullname" . }}-smoke
  annotations:
    helm.sh/hook: test
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: smoke
      image: curlimages/curl
      command: ["curl", "-fsS", "http://{{ include "app.fullname" . }}:{{ .Values.service.port }}/healthz"]
```

Nothing creates this Pod on install. `helm test <release> --logs` creates it, waits for it to exit, prints its output, and fails if the pod did, so it slots in right after `helm upgrade` in a pipeline. The test lives in the chart and is versioned with it, which is the point: whoever changes the chart changes the check. `before-hook-creation` is there because a failed test pod is kept, and without it the next `helm test` fails on `already exists` before it runs anything.

## When not to use a hook

A hook is the wrong tool more often than the right one, and the tell is the shape of the work.

| The work is | Use |
| :-- | :-- |
| Per-pod, idempotent, and needed on every start: wait for a dependency, warm a cache | An `initContainer` |
| Ordered against nothing, and you want it deleted on `helm uninstall` | A plain Job in `templates/`, with `--wait-for-jobs` if it must gate the release |
| Ongoing: a schema that follows a custom resource, a recurring backup, anything that reconciles | An operator, or a CronJob |
| Do this once per release, wait for it, then continue, and abort if it fails | A hook |

The `initContainer` migration is the common wrong answer. It runs in every replica, on every restart, and ten replicas rolling out is ten migration attempts racing for the same lock. Migration tools with advisory locks make that survivable, not correct. A hook runs once per release, before any replica exists, and a failure means no replica ever does.

The plain Job is the other one. It is fine when nothing depends on the ordering, and it has one advantage over a hook: it is in the manifest, so uninstall removes it and the three-way merge tracks it. The moment you need "before the Deployment", it stops being fine.

Hooks are for "do this, wait, then continue, abort if it fails". Nothing else in Helm gives you that, and nothing else needs it.

## Hooks under Argo CD

Argo CD renders the chart with `helm template` and never runs `helm install`, so none of the machinery above runs. It reads the same annotations and maps them onto its own hook system.

| Helm annotation | Argo CD |
| :-- | :-- |
| `helm.sh/hook: pre-install`, `pre-upgrade` | `argocd.argoproj.io/hook: PreSync` |
| `helm.sh/hook: post-install`, `post-upgrade` | `argocd.argoproj.io/hook: PostSync` |
| `helm.sh/hook: pre-delete` | `argocd.argoproj.io/hook: PreDelete` (Argo CD 3.3 and later) |
| `helm.sh/hook: post-delete` | `argocd.argoproj.io/hook: PostDelete` (Argo CD 2.10 and later) |
| `helm.sh/hook-weight` | `argocd.argoproj.io/sync-wave` |
| `helm.sh/hook-delete-policy: before-hook-creation`, `hook-succeeded`, `hook-failed` | `BeforeHookCreation`, `HookSucceeded`, `HookFailed` |
| `helm.sh/hook: pre-rollback`, `post-rollback`, `test` | Ignored: the object is never created |

Five consequences follow.

Argo CD cannot tell an install from an upgrade; every operation is a sync. `pre-install` and `pre-upgrade` are the same `PreSync` hook, so a `pre-install` hook runs on every sync and had better be idempotent. The seed-admin Job above becomes a `PostSync` hook that runs after every sync, which is why it has to treat "user already exists" as success.

Rollback hooks do not exist. An Argo rollback is a sync to an older revision of the Application, and that older revision's `PreSync` hooks run, with that revision's image. A migration tool that is at or beyond the requested version and does nothing is fine here. One that tries to go backwards is not.

`test` hooks are dropped entirely, not skipped: an object with a Helm hook type Argo does not recognise is filtered out before the sync, so it is never created and never shown. `PreDelete` and `PostDelete` fire only when the Application itself is deleted, not when a resource is pruned from the chart.

Delete policies map one to one and the default is the same, `BeforeHookCreation`. The difference is when they apply: at Argo's sync phase boundaries rather than at Helm's per-hook moment. Hook resources are created by the sync operation and tracked with it, so a hook Job shows up in the Application's resource tree for as long as it exists, which is more visibility than Helm gives you.

And if an object carries an `argocd.argoproj.io/hook` annotation, its `helm.sh/hook` annotation is ignored. Pick one.

That leaves the question of sync waves. A wave is an ordering over resources that should exist permanently and be reconciled forever: the Namespace at `-3`, the CRDs at `-2`, the operator at `-1`, the custom resources at `0`. Argo applies a wave, waits for everything in it to be healthy, pauses two seconds so other controllers can react, and moves to the next. A hook is for something that runs and finishes. Put a migration Job in a wave with no hook annotation and it becomes a permanent resource: applied once, healthy when complete, and then the next sync that changes its image tag gets `field is immutable` back from the API server, because a Job's pod template cannot be patched. Give it the hook annotation and `BeforeHookCreation` and the problem disappears, because a hook is replaced rather than patched.

Waves order things that stay. Hooks are for things that run and finish. Helm only has the second; `hook-weight` orders hooks against each other, never against the manifest. Argo CD has both, and the Helm annotation lands you in the right one.

## Links

- [Chart hooks](https://helm.sh/docs/topics/charts_hooks/), the page with the lifecycle, the weights, and the line about hook resources not being managed with the release
- Helm's hook executor, [pkg/action/hooks.go](https://github.com/helm/helm/blob/main/pkg/action/hooks.go), and the legacy waiter in [pkg/kube/wait.go](https://github.com/helm/helm/blob/main/pkg/kube/wait.go), which defines "complete" per kind
- [HIP-0019](https://helm.sh/community/hips/hip-0019/), the `hook-output-log-policy` proposal, shipped in Helm 3.18 and Helm 4
- Kubernetes [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) docs for `backoffLimit`, `activeDeadlineSeconds`, and the `restartPolicy` restriction
- Argo CD's [Helm](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/) page for the hook mapping table, and [sync phases and waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/) for the rest
- The mapping itself: [hook/helm](https://github.com/argoproj/gitops-engine/tree/master/pkg/sync/hook/helm) and [syncwaves/waves.go](https://github.com/argoproj/gitops-engine/blob/master/pkg/sync/syncwaves/waves.go) in gitops-engine for the sync-phase hooks and `hook-weight`, and [controller/hook.go](https://github.com/argoproj/argo-cd/blob/master/controller/hook.go) in argo-cd for `pre-delete` and `post-delete`
