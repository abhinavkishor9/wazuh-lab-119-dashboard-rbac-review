# wazuh-lab-119-dashboard-rbac-review
## Overview
Role-Based Access Control (RBAC) in Wazuh determines what authenticated users can view and which actions they can perform within the dashboard. Access permissions should align with each user's responsibilities, following the principle of least privilege.

A SOC analyst may need access to security alerts and investigation dashboards, while an administrator may require additional permissions to manage users, roles, and security settings. Excessive permissions can expose sensitive telemetry or allow unauthorized configuration changes.

This lab reviews the security configuration of a Wazuh deployment, focusing on dashboard users, role definitions, role mappings, index access, and administrative permissions. The objective is to understand how access is assigned and identify what additional testing is required to verify that users have only the permissions needed for their responsibilities.

The investigation also encountered an HTTPS certificate validation problem while accessing the dashboard from a Windows endpoint. This issue was investigated alongside the RBAC review because reliable, certificate-validated access is important when administering and assessing a security platform.

## Lab Objectives

- Examine Wazuh Dashboard's internal users and identify their configured backend roles and attributes.
- Review security role definitions to understand the scope of cluster, index, and tenant permissions.
- Analyze role mappings to determine how users and backend roles are associated with security permissions.
- Assess the relationship between the current administrator account and the permissions available to standard SOC analysts.
- Inspect Wazuh index availability and evaluate what index-level query results reveal about access.
- Investigate HTTPS certificate validation failures encountered while accessing the dashboard from Windows PowerShell.
- Document the difference between successful HTTP connectivity with certificate validation bypassed and trusted HTTPS communication.
- Identify gaps in the available evidence and define the additional access tests required to validate role-based restrictions.
- Produce an evidence-based assessment without assuming that configured roles automatically enforce the intended access boundaries.

## Lab Scenario

A security team is reviewing access controls in a Wazuh environment to understand whether dashboard users receive permissions appropriate to their responsibilities. The review focuses on internal user accounts, security roles, role mappings, and access to Wazuh alert data.

The investigation begins with the administrator account and expands to the available user and role configurations. The OpenSearch Security API and Wazuh Dashboard provide evidence of configured accounts, backend roles, reserved roles, and role mappings. However, the presence of a role or permission in the configuration does not, by itself, confirm that access restrictions work as intended.

During the review, the analyst encounters an HTTPS certificate validation failure when accessing the dashboard from Windows PowerShell. Although TCP connectivity to port 443 succeeds, the certificate reports a hostname mismatch and an incomplete trust chain. A diagnostic request succeeds when certificate validation is bypassed, but this does not resolve the underlying trust problem or establish that role-based access controls are functioning correctly.

The review therefore aims to distinguish confirmed configuration details from assumptions and unverified access behavior.

Key investigation areas include:

- Identifying configured internal users and their associated backend roles.
- Reviewing role permissions and mappings to understand potential access boundaries.
- Examining available Wazuh indices without assuming that their presence proves a user's access.
- Documenting the HTTPS certificate issues affecting normal PowerShell connectivity.
- Recording access tests as inconclusive where actual analyst and administrative permissions have not been verified.

The final assessment must clearly separate observed evidence, unresolved issues, and tests still required. No successful role-based access restriction should be claimed without corresponding access-test results.

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

