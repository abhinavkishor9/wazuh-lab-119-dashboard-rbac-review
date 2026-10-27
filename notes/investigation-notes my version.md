# Investigation Notes 
## 1. Endpoint and Service Context

The Windows endpoint was identified as `DESKTOP-9MMM37V`. The installed PowerShell version was `7.6.8`.

The following services were observed running:

- `WazuhSvc` — Wazuh agent service.
- `Elastic Agent` — Elastic Agent service.

These observations establish the status of the local Windows services at the time of collection. They do not establish the status of every service on the Wazuh server.

The dashboard endpoint under investigation was `https://192.168.203.129`, using TCP port `443`.

## 2. Current Security Account

The request below was executed through the dashboard Dev Tools interface:

GET /_plugins/_security/api/account

The response identified the current account as:

| Field | Observed Value |
|---|---|
| Username | `admin` |
| Reserved account | `true` |
| Internal user | `true` |
| Backend roles | `admin` |
| Security roles | `own_index`, `all_access` |
| Global tenant | `true` |
| Admin tenant | `true` |
| Tenant named `admin` | `true` |

The response confirms that the investigation was performed using the internal `admin` account. This is an important limitation: administrator access cannot demonstrate what a standard analyst account is permitted or denied.

The account response is evidence of the current account's reported mappings and tenant access. It is not a substitute for inspecting the effective permissions and testing actions under other accounts.

## 3. Internal Users

The following request was used:

GET /_plugins/_security/api/internalusers

The dashboard's internal-user view showed these accounts:

| Account | Observed Backend Role or Information |
|---|---|
| `admin` | `admin` |
| `logstash` | `logstash` |
| `snapshotrestore` | `snapshotrestore` |
| `kibanaserver` | No backend role displayed |
| `kibanaro` | `kibanauser`, `readall`; attributes were displayed |
| `readall` | `readall` |
| `anomalyadmin` | No backend role displayed |

The API response and dashboard table provide an inventory of visible internal accounts and selected attributes. The available output does not establish the full effective permissions of each account.

The presence of an account or backend role does not automatically demonstrate that the account is actively used, improperly configured, or excessively privileged. Each account must be assessed against its intended function and associated role mappings.

## 4. Security Roles

The role inventory was retrieved using:

GET /_plugins/_security/api/roles

The dashboard displayed 45 roles.

One inspected role was:

GET /_opendistro/_security/api/roles/observability_read_access

The response showed:

- `reserved`: `true`
- Cluster permission: `cluster:admin/opensearch/observability/get`
- Index permissions: none listed
- Tenant permissions: none listed

Another role visible in the inventory was `security_analytics_ack_alerts`, with the cluster permission `cluster:admin/opensearch/securityanalytics/alerts/*`.

These examples demonstrate that roles can contain different permission scopes. The observed role names alone do not prove what a particular analyst can do, and the selected examples are not a complete review of the 45 roles.

## 5. Role Mappings

The role-mapping inventory was queried using:

GET /_plugins/_security/api/rolesmapping

The captured output included mappings such as:

- `manage_snapshots`, mapped to backend role `snapshotrestore`.
- `logstash`, mapped to backend role `logstash`.
- `manage_wazuh_index`, which appeared in the captured response.

A separate dashboard view showed reserved Wazuh UI mappings, including:

- `wui_elastic_admin`
- `wui_open_search_admin`

The displayed mapping table associated these entries with the `administrator` role and `wazuh-wui`.

The available screenshots do not contain the complete role-mapping response. Therefore, they do not support a complete mapping inventory or a definitive conclusion about all users' effective permissions.

## 6. Index Access and Health

The following requests were used to inspect index access:

GET /wazuh-alerts-*/_count

GET /_cat/indices/wazuh-*?v

The captured index listing showed green health for the displayed indices. Examples included Wazuh alert indices dated `2026.08.18` and `2026.08.19`, monitoring indices, and a local inventory packages index.

The available screenshot contains only part of the listing. It does not establish a complete inventory of indices, and the captured output does not provide sufficient evidence to report the result of the `_count` request independently.

Green index health describes index availability and shard health. It does not prove that role-based restrictions are correctly enforced.

## 7. HTTPS Connectivity and Certificate Validation

The initial request used:

`https://192.168.203.129`

PowerShell returned an SSL connection error with these validation failures:

- `RemoteCertificateNameMismatch`
- `RemoteCertificateChainErrors`

A separate TCP test to port `443` returned `TcpTestSucceeded : True`, confirming that the port was reachable from the Windows endpoint.

The presented certificate was inspected and had these properties:

| Property | Observed Value |
|---|---|
| Subject | `CN=wazuh-dashboard, OU=Wazuh, O=Wazuh, L=California, C=US` |
| Issuer | `L=California, O=Wazuh, OU=Wazuh` |
| Subject Alternative Name | `IP Address=127.0.0.1` |
| Thumbprint | `104BBC3DE72C8CE9B86D67F64A14E16EA80038D6` |
| Signature algorithm | `sha256RSA` |
| Validity start | 1 August 2026 |
| Validity end | 29 July 2036 |

The certificate did not list `192.168.203.129` in its SAN.

`certutil -verify` reported `CERT_E_CHAINING`, `CERT_TRUST_IS_PARTIAL_CHAIN`, and an unavailable revocation check. A search of the Windows Local Machine Root and CA stores returned no certificates matching `Wazuh`.

These results support the conclusion that hostname validation and chain validation were failing. The complete issuing CA and the intended certificate deployment were not established.

A later request using `Invoke-WebRequest -SkipCertificateCheck` returned HTTP `200`. This was a diagnostic request with certificate validation bypassed. It must not be interpreted as successful certificate validation or as evidence of correct RBAC enforcement.

## 8. Access-Test Status

The test-results file was created with these initial entries:

| Test | Expected | Observed | Status |
|---|---|---|---|
| Analyst alert access | Allowed | Not tested | Inconclusive |
| Role administration | Restricted | Not tested | Inconclusive |

These entries are placeholders, not test results. The investigation did not establish whether a standard analyst account could access the intended alert data or whether it was prevented from performing administrative operations.

## 9. Findings and Limitations

### Confirmed observations

- The current Security API account was `admin`.
- The account response listed `own_index` and `all_access`.
- The dashboard displayed seven internal-user entries in the captured view.
- The role inventory displayed 45 roles.
- Selected role and role-mapping details were retrieved.
- The displayed indices had green health.
- TCP port `443` was reachable from Windows.
- Normal HTTPS validation failed because of a certificate name mismatch and chain errors.
- The diagnostic HTTPS request returned HTTP `200` when certificate validation was bypassed.

### Unverified questions

- Whether a standard analyst has the intended effective permissions.
- Whether administrative actions are properly restricted for non-administrators.
- Whether every role mapping is appropriate for its intended purpose.
- Whether the missing issuer certificate is available on the Wazuh server.
- Whether the certificate should be replaced or reissued by an existing lab CA.
- Whether index-level access restrictions match the intended data scope.

