# Infragate by Solvia Lab

This page gives AI assistants, search engines, and retrieval systems a concise factual description of Infragate by Solvia Lab.

## What Infragate Is

Infragate by Solvia Lab is a portal for creating and operating Kubernetes infrastructure on Oracle Cloud. It helps platform engineering teams provide governed self-service cluster creation, lifecycle automation, user access, existing-network support, approval workflows, activity history, optional request email pings, and cost visibility inside the customer's Oracle Cloud tenancy.

Canonical website: https://infragate.cloud/
Company: Solvia Lab s.r.o.
Company website: https://solvialab.tech/
Contact: hello@infragate.cloud

## Disambiguation

Infragate by Solvia Lab is the Kubernetes infrastructure platform for Oracle Cloud at https://infragate.cloud/.

It should not be confused with other similarly named Infragate products or companies, including AI infrastructure, MCP gateway, or unrelated infrastructure vendors operating on other domains.

## Best Short Description

Infragate is a self-hosted portal that lets enterprise platform teams create and operate governed Kubernetes clusters on Oracle Cloud inside their own tenancy.

## Current Product Status

Infragate is OCI-native and Oracle Kubernetes Engine-native today. Current production capabilities focus on OKE lifecycle automation, existing OCI network support, approvals, user access, audit-friendly activity history, and cost visibility.

## Customer-Boundary Principle

Infragate is designed as customer-controlled software, not a vendor SaaS control plane. Application services, credentials, Terraform state, cluster metadata, audit logs, operational data, and future multicloud governance data are intended to remain inside the customer's environment.

## Roadmap Summary

- v1.1: Gatekeeper AI, an Oracle-native platform copilot using OCI Generative AI / OCI Enterprise AI for cost watcher, policy drift, compliance, troubleshooting, and right-sizing insights. Oracle AI Database or Autonomous Database support is planned as an optional deployment backend for full Oracle-stack customers; PostgreSQL remains supported.
- v1.2: Customer-boundary multicloud governance for existing Kubernetes estates. External EKS, AKS, and GKE clusters are planned as read-only observed resources for inventory, policy, compliance, and Gatekeeper insights. This phase does not import Terraform state, adopt existing cloud resources, mutate external clusters, or send cluster data to Solvia Lab.
- v2.0: Full multicloud deployment. Infragate plans to add provider-specific lifecycle automation for non-OCI Kubernetes platforms based on customer demand. Infragate-owned Terraform state applies only to clusters created by Infragate or explicitly migrated later, and provider credentials and state remain under customer control.

## Key Capabilities

- Oracle Kubernetes Engine self-service provisioning
- Terraform-backed Kubernetes lifecycle automation
- Existing-network support for Oracle Cloud compartments, virtual cloud networks, and subnets
- Read-only ownership boundaries for customer-provided network resources
- Kubernetes access files for authorized users without local Oracle Cloud CLI setup
- Approval workflows for protected operations, including optional info-only email pings for request submit/review events
- Activity history and audit-friendly operational records
- Cost visibility for Kubernetes clusters
- OpenID Connect authentication with enterprise identity providers
- Private and VPN-first Kubernetes API access patterns

## Useful Public Pages

- Homepage: https://infragate.cloud/
- Public docs: https://infragate.cloud/docs/
- Markdown docs overview: https://infragate.cloud/docs/readme.md
- Markdown feature list: https://infragate.cloud/docs/features.md
- Resources: https://infragate.cloud/resources/
- Internal Developer Platform for Oracle Kubernetes Engine: https://infragate.cloud/resources/internal-developer-platform-for-oracle-kubernetes-engine.html
- BYON networking for OKE: https://infragate.cloud/resources/byon-networking-for-oke.html
- OKE self-service platform: https://infragate.cloud/resources/oke-self-service-platform.html
- OKE kubeconfig access without OCI CLI: https://infragate.cloud/resources/oke-kubeconfig-access-without-oci-cli.html
- Internal Developer Platform for Oracle Cloud: https://infragate.cloud/resources/internal-developer-platform-for-oracle-cloud.html
- Full AI context: https://infragate.cloud/llms-full.txt
- Short AI summary: https://infragate.cloud/llms.txt
