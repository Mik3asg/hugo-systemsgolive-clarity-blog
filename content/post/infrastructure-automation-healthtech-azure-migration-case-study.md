---
title: "Case Study: Infrastructure Automation Cuts Server Build Time from 4 Hours to 30 Minutes on a Regulated Healthtech Platform"
date: 2026-08-08T09:00:00+01:00
draft: false
tags: ["Case Study", "Azure", "Bicep", "Ansible", "Infrastructure as Code", "Cloud Migration", "Automation"]
categories: ["Azure", "DevOps"]
thumbnail: "images/infra-automation-azure-migration.png"
summary: "How splitting infrastructure delivery into Bicep provisioning and Ansible configuration took a regulated healthtech platform's server builds from 4 hours of manual work down to 30 minutes of automated pipeline time, ahead of a live Azure migration."
---

## Problem

As part of a live infrastructure migration for a healthtech platform (moving production services from an existing UK cloud hosting provider to Microsoft Azure), every application server was being built by hand: OS hardening, user creation, SSH configuration, Tomcat and OpenJDK installation, file-sync setup, TLS certificates, and LDAP, all configured manually, server by server. Each new application server took roughly **4 hours** of hands-on-keyboard work.

The platform is a Digital Social Care Record (DSCR) system operating in a highly regulated environment: care providers using it must be registered with the Care Quality Commission (CQC) and meet the requirements of the NHS Data Security and Protection (DSP) Toolkit. In that environment, manual builds carry real risk: undocumented configuration drift between nodes, inconsistent hardening, and no repeatable record of what was actually done to a server, all things an auditor (and an incident) will eventually surface.

---

## Approach

Working within a larger Azure migration project, I split infrastructure delivery into two layers:

- **Provisioning (Bicep + Azure Deployment Stacks):** VMs are provisioned directly into pre-existing, pre-approved subnets. Network and VNet boundaries are handled separately and were not part of this automation.
- **Configuration (Ansible):** Everything on top of the VM, OS hardening, user creation, SSH setup, Tomcat, OpenJDK, file synchronization, Let's Encrypt TLS certificates issued and auto-renewed via Certbot, and LDAP, is deployed through templated, validated playbooks. Configuration changes are checked (dry-run/diff) before being applied, and validation gates prevent broken configuration from reaching production silently.

The new Azure environment is being built in parallel with the existing production environment, not as a big-bang cutover. A site-to-site VPN connects the two environments; migration traffic between them uses private IP addressing over that encrypted VPN tunnel, and services are not directly exposed to the public internet. That private connectivity is used to copy application directories and content across ahead of time, and at cutover, rsync is used to pull across only the incremental changes rather than a full transfer, keeping the final cutover window short.

To control cost while the two environments run side by side, the new Azure infrastructure is deliberately built at a lower compute tier than production load requires. The plan is to scale out compute at the point of cutover, when traffic is redirected to Azure via a DNS (FQDN) change.

The result is a repeatable pipeline: provisioning and full service configuration for a node now happen the same way, every time, instead of depending on whoever is building it that day and what they remember to do.

---

## Outcome

- **Build time:** ~4 hours per application server manually → **~30 minutes of automated pipeline time (Bicep provisioning + Ansible configuration) to fully configure the entire cluster** (OS, security, services, TLS, LDAP).
- **Risk reduction:** Removed the main source of human error and configuration drift during an actively audited migration. Configuration is now defined in code and validated before deployment, not typed by hand under time pressure.
- **Recovery speed:** A failed or replaced node can be rebuilt to the same known-good state in minutes rather than being manually reconstructed from memory or scattered notes.
- **Time reallocation:** Removed repetitive manual build work from the migration timeline, freeing time for higher-value work on the migration itself (directory services, file-sync architecture, and cutover planning).

This work sits inside a larger, still-ongoing cloud migration for the platform; the outcomes above reflect the provisioning and configuration automation specifically, not the full migration end-to-end.

---

*Scope note: this covers the web/application infrastructure layer: VM provisioning (Bicep), OS/service configuration (Ansible), and the private connectivity (site-to-site VPN, firewall rules, rsync-based content sync) used to build it out. The database layer is a separate workstream and is not covered in this document. Final production cutover (DNS flip and compute scale-out) had not yet occurred at time of writing. Example of infrastructure automation work delivered as part of an employed role, not an independent client engagement.*
