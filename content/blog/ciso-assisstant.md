---
title: "Getting Started with CISO Assistant on Kubernetes"
description: "Deploying CISO Assistant via Helm, creating your first superuser, adding users, and how evidence storage actually works."
dateString: June 2026
draft: false
tags: ["Kubernetes", "DevOps", "Security", "Compliance", "GRC", "ISO27001"]
weight: 1
cover:
    image: "/blog/ciso/ciso.png"
---

CISO Assistant is an open source GRC (Governance, Risk, Compliance) platform. If you're doing ISO 27001, NIS2, or any compliance work and got tired of spreadsheets, it's worth a look. This post covers getting it running on Kubernetes via Helm, the first-login/superuser flow, adding users, and how evidence works.

## Installing via Helm

```bash
helm repo add intuitem https://intuitem.github.io/ca-helm-chart/
helm show values intuitem/ciso-assistant > my-values.yaml
```

Check `my-values.yaml`, specifically the `frontendOrigin` parameter, and adjust to your setup.

```bash
kubectl create ns ciso-assistant
helm install my-octopus intuitem/ciso-assistant -f my-values.yaml -n ciso-assistant
```

That's it for the install. Unlike docker-compose, there's no interactive prompt to create your first admin user - the backend just runs migrations and comes up. You create the superuser yourself.

## Creating a superuser

Find your backend pod, then run `createsuperuser` inside it directly with `kubectl exec`:

```bash
kubectl get pods -n ciso-assistant
kubectl exec -it <backend-pod-name> -n ciso-assistant -- poetry run python manage.py createsuperuser
```

`-it` attaches your terminal to that command, so it'll prompt you for email and password right there. That's your admin account.

If you ever lose the password later, run `changepassword` the same way:

```bash
kubectl exec -it <backend-pod-name> -n ciso-assistant -- poetry run python manage.py changepassword <email>
```

## Checking if a superuser already exists

Useful if you're not sure whether someone already set one up, or you're debugging why login isn't working. Open a Django shell the same way, via `kubectl exec`:

```bash
kubectl exec -it <backend-pod-name> -n ciso-assistant -- poetry run python manage.py shell
```

Then:

```python
from iam.models import User
for u in User.objects.filter(is_superuser=True):
    print(u.email, u.is_active, u.date_joined)
```

This prints out every superuser account, whether it's active, and when it was created.

## Email validation

<!-- TODO: add details on enabling email validation -->

## Adding users

Once you're logged in as admin, you don't need the CLI for this - there's a normal UI flow.

In the app: **Organization → Users → Add user**. Enter their email, assign a role (regular user, admin, etc., depending on what's configured under access control), and save.

The admin "add user" form doesn't ask you to set a password. It creates the account and fires off an invitation email with a link for the user to set their own password.

## Adding admin users

Same flow as adding a normal user, just assign the admin role/group when creating them (or edit an existing user and change their group afterward: Organization → Users → select user → update their role).

## Storing evidence

Evidence in CISO Assistant is what justifies a compliance requirement's status, or proves a control was actually applied. It's attached at the requirement level inside a compliance assessment/audit.

Each piece of evidence supports:

- A **description** - what it shows
- A **file** - upload something directly
- A **link** - point to wherever the actual artifact already lives

If your controls already live somewhere else - a git repo, a wiki, a ticketing system - you don't need to duplicate that content into CISO Assistant. You just link to it and add a short description of what it proves. Keeps a single source of truth instead of two copies that can drift out of sync.
