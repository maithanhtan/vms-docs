---
name: vms-docs
description: Use the Vero Monitor Service documentation for VMS master setup, VMS agent collector rollout, and operations troubleshooting.
license: Proprietary
compatibility: Works with teams operating VMS master, collector agents, dashboards, alerting, synthetic checks, and observability components.
metadata:
  version: "1.0"
  source: "Vero Monitor Service documentation"
---

# VMS Operations Documentation Skill

Use this skill when implementing, operating, or troubleshooting Vero Monitor Service.

## Source of truth

- VMS scope, monitoring model, dashboard coverage, rollout plan, and operational constraints are documented in `README.md`.
- Published documentation focuses on three operator workflows:
  - Set up the VMS master control plane.
  - Set up VMS agent collectors near monitored systems.
  - Troubleshoot collector, dashboard, alert, synthetic check, and observability issues.

## Rules for agents

1. Keep the documentation focused on VMS operations.
2. Prefer the VMS four-layer model: Infrastructure, Service, User, and Business Flow.
3. Keep collector setup read-only by default unless an approved runbook action explicitly requires write access.
4. Use clear ownership tags for system, environment, service, owner, criticality, and scope.
5. Treat advanced observability features as conditional on approved instrumentation, credentials, and network access.
6. Do not introduce trading platform integration guidance into these docs.
