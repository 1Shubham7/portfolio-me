---
title: "Site Reliability Engineer (Obmondo)"
dateString: Apr 2025 – Present
draft: false
tags: ["Kubernetes", "SRE", "Go", "Observability", "Security"]
showToc: false
weight: 5
cover:
    image: "experience/obmondo.jpeg"
---

## Outline
Worked as an SRE, handling production systems and managing Kubernetes clusters and Linux servers across both internal and customer environments. As the youngest SRE at Obmondo, regularly took ownership of production alerts and incidents, performing root cause analysis and implementing long-term fixes. Led the rewrite of the most business-critical internal tool, improving its reliability, testing, and integration into GitOps workflows. Authored and improved a large part of the company’s ISMS, strengthening both the company’s and customers’ information security. Deployed and maintained multiple applications across staging and production environments.

- Rewrote the incident automation tool that routes alerts from Alertmanager to Gitea, automatically creating and managing issues, comments, labels, and resolution tracking based on alert type. Implemented a cache-based alert history with timelines, allowing SREs to view up to 7 days of alert history. Wrote a Cron to synchronize the cache with Gitea issues, making the cache the single source of truth. Developed an end-to-end test suite for the tool and deployed a GitOps-based staging environment for testing. This tool became critical for pitching to customers how we handle their cluster alerts and incidents.
- Migrated from Grafana OnCall to a self-hosted Incident Response Management (IRM) system, reducing operational costs by $4,000/year. Deployed the IRM using custom Helm charts and GitOps workflows. Designed and implemented a webhook-driven incident pipeline connecting Alertmanager to the IRM and Telegram API, routing alerts to SREs’ Telegram based on schedules and escalation chains.
- Desiged and Wrote most of the company’s ISMS (Information Security Management System) to achieve ISO 27001 compliance, covering everything from technical to organizational controls. Worked directly with the CEO and CTO to define and implement security policies. Deployed security tooling and wrote automation scripts to improve and automate information security practices. Played a key role in making the organization ISO 27001 compliant and supported customers through consulting and pitching compliance services.
- Built a Prometheus metric exporter that generates compliance metrics when repositories in VCS violate ISO security policies or when commits are not signed using YubiKeys. Implemented pre-commit hooks enforcing GPG-signed commits using approved YubiKey keys and alerting on git history rewrites in protected branches.
- Wrote scripts for SBOM generation and end-to-end testing of the customer onboarding system. Designed and built a panic recovery mechanism with centralized error-handling middleware in Go, which was critical for the reliability of an agent.
- Resolved critical incidents in customer Kubernetes clusters and Linux servers by performing root cause analysis (RCA) and designing long-term, preventative solutions.
- Designed and built Go APIs for internal web applications and wrote optimized SQL queries using CTEs. Used DBeaver for query design, analysis, and optimization.
- Deployed a GRC system and implemented Velero backups to preserve ISMS evidence and ensure disaster recovery.
