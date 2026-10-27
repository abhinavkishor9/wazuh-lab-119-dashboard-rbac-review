# Lab 119 — Wazuh Dashboard Role-Based Access Review

## Overview

This lab reviews the security configuration of a Wazuh deployment, focusing on dashboard users, role definitions, role mappings, index access, and administrative permissions. The objective is to understand how access is assigned and identify what additional testing is required to verify that users have only the permissions needed for their responsibilities.

The investigation also encountered an HTTPS certificate validation problem while accessing the dashboard from a Windows endpoint. This issue was investigated alongside the RBAC review because reliable, certificate-validated access is important when administering and assessing a security platform.

## Objectives

- Identify available internal users and their assigned backend roles.
- Inspect configured security roles and their permissions.
- Review role mappings and administrative access assignments.
- Examine available Wazuh indices and their health status.
- Test dashboard access with certificate validation enabled and compare it with a diagnostic request that bypasses validation.
- Document access-testing limitations and distinguish configuration evidence from verified user behavior.
- Establish the additional evidence required to complete the RBAC assessment.

## Environment

| Component | Observed Details |
|---|---|
| Windows endpoint | `DESKTOP-9MMM37V` |
| PowerShell | `7.6.8` |
| Wazuh agent service | `WazuhSvc` — Running |
| Elastic Agent service | Running |
| Dashboard endpoint | `https://192.168.203.129` |
| Dashboard port | `443` |
| Current security account | `admin` |
| Dashboard certificate subject | `CN=wazuh-dashboard, OU=Wazuh, O=Wazuh, L=California, C=US` |

## Investigation Summary

The OpenSearch Security API was used through the dashboard's Dev Tools interface to inspect the current account, internal users, roles, role mappings, and index information.

The current account was identified as the reserved internal user `admin`, with backend role `admin` and security roles `own_index` and `all_access`. The internal-user listing also showed accounts including `logstash`, `snapshotrestore`, `kibanaserver`, `kibanaro`, `readall`, and `anomalyadmin`.

The role inventory contained 45 roles. The reviewed role examples included `observability_read_access` and `security_analytics_ack_alerts`. The available role-mapping output also contained mappings for service-related and administrative roles. These observations establish that security configuration exists, but they do not independently demonstrate the effective permissions of every account.

The index listing showed green health for the displayed Wazuh indices, including alert, monitoring, and inventory indices. The observed alert indices included dates from August 2026. Index health alone does not establish whether the expected analyst permissions are correctly enforced.

## HTTPS Certificate Findings

A normal PowerShell HTTPS request to `https://192.168.203.129` failed with:

- `RemoteCertificateNameMismatch`
- `RemoteCertificateChainErrors`

The presented certificate had the subject `CN=wazuh-dashboard` and a Subject Alternative Name containing only `IP Address=127.0.0.1`. It did not list `192.168.203.129`.

Certificate verification with `certutil` reported an incomplete chain, a missing issuer certificate, and an unavailable revocation check. The exported certificate did not contain a private key, as expected for a public certificate.

A separate diagnostic request using `-SkipCertificateCheck` returned HTTP `200`. This confirms that the endpoint returned a successful HTTP response when certificate validation was bypassed. It does not establish that the certificate is trusted or that the dashboard's RBAC controls are correct.

## Outcome

**Overall status: Inconclusive**

The security configuration was partially inspected, and the HTTPS certificate validation issue was reproduced and characterized. However, access tests using a standard analyst account and administrative restrictions were not completed.

The next steps are to correct the dashboard certificate and trust configuration, then validate permitted and restricted actions using authorized test accounts.

## Evidence and Limitations

- The current account and selected security configuration were observed through the Security API.
- Only portions of the role and role-mapping responses were captured in the available evidence.
- No complete permission review was performed for all 45 roles.
- No successful analyst-versus-administrator access comparison was established.
- The certificate was inspected, but its intended issuing CA and complete provenance were not established.
- No claim of confirmed excessive access or successful least-privilege enforcement is made.

## Safety Notes

All testing should remain within an authorized lab environment. Certificate validation should not be disabled for routine access, and certificates should not be trusted until their provenance and intended purpose have been verified.
