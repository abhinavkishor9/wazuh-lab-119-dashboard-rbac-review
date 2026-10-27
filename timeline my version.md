# Investigation Timeline 
### 07:30 — Evidence Directory Created

Created the local evidence directory:

`C:\Users\Dell\Desktop\Lab119-Wazuh-RBAC-Review`

The directory was confirmed to exist. An initial CSV was prepared to track analyst alert access and role-administration tests. Both entries were marked `Not tested` and `Inconclusive`.

### Environment Inventory — Time Not Recorded

Collected the Windows endpoint and local service details.

- Endpoint: `DESKTOP-9MMM37V`
- PowerShell version: `7.6.8`
- Wazuh agent service: `WazuhSvc`, Running
- Elastic Agent service: Running

These observations describe the Windows endpoint and do not establish the status of every service on the Wazuh server.

### Security Account Review — Time Not Recorded

Queried the current account through the OpenSearch Security API:

`GET /_plugins/_security/api/account`

The response identified the current user as the reserved internal account `admin`, with backend role `admin` and security roles `own_index` and `all_access`.

This established that the configuration review was conducted using an administrator account rather than a standard SOC analyst account.

### Internal User and Role Review — Time Not Recorded

Queried the internal users and role definitions:

- `GET /_plugins/_security/api/internalusers`
- `GET /_plugins/_security/api/roles`

The dashboard displayed seven internal-user entries in the captured view and 45 configured roles.

The available output included accounts such as `logstash`, `snapshotrestore`, `kibanaserver`, `kibanaro`, `readall`, and `anomalyadmin`.

The role inventory included `observability_read_access` and `security_analytics_ack_alerts`. The available evidence was not a complete review of every role's effective permissions.

### Role-Mapping Review — Time Not Recorded

Queried:

`GET /_plugins/_security/api/rolesmapping`

The captured response included mappings for `manage_snapshots`, `logstash`, and `manage_wazuh_index`.

A separate dashboard view showed the reserved Wazuh UI mappings `wui_elastic_admin` and `wui_open_search_admin`, associated with the `administrator` role and `wazuh-wui`.

The captured mapping output was incomplete, so a complete account-to-permission mapping could not be established.

### Index Inspection — Time Not Recorded

Queried:

- `GET /wazuh-alerts-*/_count`
- `GET /_cat/indices/wazuh-*?v`

The displayed index listing included Wazuh alert, monitoring, and inventory indices with green health. Some alert indices were dated August 2026.

The evidence does not establish that index health proves correct role enforcement, and the captured output does not independently establish the result of the count request.

### HTTPS Connectivity Test — Time Not Recorded

A normal PowerShell request to `https://192.168.203.129` failed with certificate validation errors.

A TCP test to port `443` returned `TcpTestSucceeded : True`, confirming TCP reachability.

### Certificate Inspection — Time Not Recorded

Inspected the certificate presented by the dashboard endpoint.

The certificate subject was `CN=wazuh-dashboard`, and its SAN contained only `IP Address=127.0.0.1`.

The certificate did not identify `192.168.203.129` in its SAN. Windows also reported certificate chain validation errors.

### Hostname Resolution Check — Time Not Recorded

Checked whether `wazuh-dashboard` resolved through Windows DNS and whether a matching hosts-file entry existed.

Neither command returned a matching result.

### Certificate Export and Verification — Time Not Recorded

Exported the presented public certificate to:

`C:\Users\Dell\AppData\Local\Temp\wazuh-dashboard.cer`

Ran `certutil -verify` against the exported certificate. The output reported an incomplete certificate chain, a missing issuer certificate, and an unavailable revocation check.

The Windows Local Machine Root and CA stores were searched for certificates matching `Wazuh`. No matching certificates were returned.

The issuer's provenance and the intended certificate chain remained unverified.

### Diagnostic HTTPS Request — Time Not Recorded

Retried the dashboard request using PowerShell's `-SkipCertificateCheck` option.

The request returned HTTP `200`.

This demonstrated that the endpoint returned a successful HTTP response when certificate validation was bypassed. It did not resolve the certificate problem or verify RBAC behavior.

