---
title: "Deploying Grafana OnCall in an Air-Gapped Kubernetes Cluster"
description: "A step-by-step guide to running Grafana OnCall without any egress to grafana.com — using OCI artifacts, init containers, and Kubernetes secrets to deliver plugins in restricted environments."
dateString: April 2026
draft: false
tags: ["Grafana", "Kubernetes", "DevOps", "Air-Gapped", "Helm", "OCI", "Harbor"]
weight: 101
cover:
    image: "blog/grafana-airgap.png"
---

## The Problem

By default, Grafana downloads the `grafana-oncall-app` UI plugin directly from `grafana.com` every time a pod starts. In a security-hardened or air-gapped Kubernetes cluster — where outbound internet access is restricted — this causes the plugin to fail on every restart.

This post walks through how to make Grafana OnCall work completely offline, with no egress to `grafana.com`, by pushing the plugin to a private OCI-compatible registry (such as Harbor) and using an init container to install it at pod startup.

---

## Overview

The approach has five parts:

1. Push the plugin to your private registry as an OCI artifact
2. Mirror the `oras` CLI image to your private registry
3. Create Kubernetes secrets for registry authentication
4. Configure your Helm values to use the init container
5. Block egress to `grafana.com` at the network level

---

## Step 1 — Push the Plugin to Your Private Registry

First, get the plugin ZIP. You can download it from the Grafana plugin page:

```
https://grafana.com/api/plugins/grafana-oncall-app/versions/<YOUR_PLUGIN_VERSION>/download
```

Or copy it directly from an existing Grafana pod:

```sh
kubectl cp <GRAFANA_POD>:/var/lib/grafana/plugins/grafana-oncall-app ./grafana-oncall-app
```

Then zip it up:

```sh
zip -r grafana-oncall-app.zip grafana-oncall-app/
```

Now push it to your registry using [`oras`](https://oras.land), which stores files as OCI artifacts:

```bash
oras push <REGISTRY_URL>/<PROJECT>/grafana-oncall-app-plugin:<VERSION> \
  grafana-oncall-app.zip:application/zip
```

> The init container will use `oras pull` to retrieve this artifact at pod startup.

---

## Step 2 — Mirror the `oras` Image

The init container runs the `oras` CLI inside a container. Mirror its image into your private registry so it can be pulled without internet access:

```bash
docker pull ghcr.io/oras-project/oras:<VERSION>
docker tag  ghcr.io/oras-project/oras:<VERSION> <REGISTRY_URL>/<PROJECT>/oras:<VERSION>
docker push <REGISTRY_URL>/<PROJECT>/oras:<VERSION>
```

---

## Step 3 — Create Kubernetes Secrets

Two secrets are needed in the OnCall namespace.

### Image pull secret

This allows Kubernetes to pull the `oras` container image from your private registry:

```bash
kubectl create secret docker-registry registry-pull-secret \
  --docker-server=<REGISTRY_URL> \
  --docker-username=<robot-account> \
  --docker-password=<robot-token> \
  --dry-run=client -o yaml > registry-pull-secret.yaml
```

### Registry credentials for `oras pull`

This secret is used inside the init container script to authenticate when pulling the plugin artifact:

```bash
kubectl create secret generic registry-credentials \
  --from-literal=username=<robot-account> \
  --from-literal=password=<robot-token> \
  --dry-run=client -o yaml > registry-credentials.yaml
```

Apply both secrets to your cluster (or manage them via your preferred secrets workflow, e.g. Sealed Secrets, External Secrets Operator, etc.).

---

## Step 4 — Configure Helm Values

Update your Grafana Helm values to wire everything together. The key changes are:

- Disable the default `grafana.com` plugin download
- Prevent Grafana from syncing the plugin registry on startup
- Add an init container that pulls and installs the plugin from your registry
- Mount the install script via a ConfigMap

```yaml
oncall:
  grafanaPluginInstall:
    enabled: true   # Renders a ConfigMap with the plugin install script

  grafana:
    plugins: []     # Disable the grafana.com plugin download list

    grafana.ini:
      plugins:
        preinstall_sync_enabled: false  # Prevent plugin registry sync

    env:
      GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS: grafana-oncall-app

    image:
      pullSecrets:
        - registry-pull-secret

    extraInitContainers:
      - name: install-oncall-plugin
        image: <REGISTRY_URL>/<PROJECT>/oras:<VERSION>
        command: [sh, /scripts/install-plugin.sh]
        env:
          - name: REGISTRY_USER
            valueFrom:
              secretKeyRef:
                name: registry-credentials
                key: username
          - name: REGISTRY_PASSWORD
            valueFrom:
              secretKeyRef:
                name: registry-credentials
                key: password
          - name: PLUGIN_REF
            valueFrom:
              secretKeyRef:
                name: registry-credentials
                key: plugin
        volumeMounts:
          - name: storage
            mountPath: /var/lib/grafana
          - name: plugin-install-script
            mountPath: /scripts

    extraVolumes:
      - name: plugin-install-script
        configMap:
          name: oncall-grafana-plugin-install
          defaultMode: 0755
      - name: provisioning
        configMap:
          name: helm-testing-grafana-plugin-provisioning
```

> **Note:** `extraInitContainers` and `extraVolumes` replace (rather than merge with) subchart defaults in most Helm setups. Make sure to re-include any volumes that were previously defined by the subchart, such as the provisioning volume shown above.

---

## Step 5 — Block Egress to grafana.com

Once everything is in place, enforce that the Grafana pod cannot reach `grafana.com` at all. This ensures the air-gapped setup remains intact and no accidental downloads occur.

The exact mechanism depends on your cluster's networking stack. Common options include:

- **Kubernetes NetworkPolicy** — blocks traffic at the namespace/pod level using standard Kubernetes primitives
- **CNI-level policy** (e.g. Cilium, Calico) — offers more granular egress controls including FQDN-based rules
- **Egress proxies or firewalls** — enforce at the infrastructure level outside the cluster

Choose whichever fits your environment. The important thing is that `grafana.com` is unreachable from the Grafana pod after this change.

---

## Summary

With this setup, Grafana OnCall runs fully offline:

- The plugin ZIP lives in your private registry as an OCI artifact
- An init container pulls and installs the plugin before Grafana starts
- The default `grafana.com` download is disabled via Helm and `grafana.ini`
- Network-level controls prevent any accidental egress

This pattern generalises beyond OnCall — you can use the same approach for any Grafana plugin in an air-gapped environment.