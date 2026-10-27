# Troubleshooting Notes

## 1. PowerShell HTTPS Request Failed

### Symptom

A normal HTTPS request to the dashboard failed:

```powershell
$DashboardUrl = "https://192.168.203.129"

Invoke-WebRequest `
    -Uri $DashboardUrl `
    -Method Get `
    -TimeoutSec 10
```

The exception reported:

- `RemoteCertificateNameMismatch`
- `RemoteCertificateChainErrors`

### Investigation

The endpoint was reachable on TCP port `443`:

```powershell
Test-NetConnection 192.168.203.129 -Port 443
```

The output returned:

`TcpTestSucceeded : True`

The presented certificate contained the SAN `IP Address=127.0.0.1`, not `192.168.203.129`.

### Interpretation

TCP connectivity was available, but certificate validation failed. A reachable TCP port does not prove that the HTTPS certificate is valid or trusted.

### Resolution Status

Not permanently resolved during this investigation. The certificate and trust configuration still require remediation.

## 2. Dashboard Responded When Certificate Validation Was Bypassed

### Diagnostic Request

```powershell
Invoke-WebRequest `
    -Uri "https://192.168.203.129" `
    -Method Get `
    -TimeoutSec 10 `
    -SkipCertificateCheck
```

### Observed Result

The response returned HTTP status `200`.

### Interpretation

The endpoint returned a successful HTTP response when PowerShell skipped certificate validation. This confirms that the original request's certificate validation failure was preventing the normal request from completing.

It does not establish that the certificate is trusted, that the server's certificate configuration is correct, or that RBAC restrictions are working.

### Safety Note

`-SkipCertificateCheck` must remain a diagnostic-only workaround. Do not use it for routine authenticated dashboard access or sensitive operations.

## 3. Dashboard Hostname Did Not Resolve

### Commands

```powershell
Resolve-DnsName wazuh-dashboard -ErrorAction SilentlyContinue

Select-String `
    -Path "$env:SystemRoot\System32\drivers\etc\hosts" `
    -Pattern "wazuh-dashboard"
```

### Observed Result

Neither command returned a matching result.

### Interpretation

The hostname did not resolve through the tested Windows DNS lookup, and no matching entry was found in the inspected hosts file.

This does not prove that the hostname is absent from the Linux VM's configuration or from every DNS server.

### Next Action

Confirm the intended dashboard hostname and address from the server configuration. If a hostname is selected, configure name resolution appropriately and ensure the certificate SAN contains that hostname.

## 4. Certificate Chain Could Not Be Built

### Certificate Export

The public certificate was exported to:

`C:\Users\Dell\AppData\Local\Temp\wazuh-dashboard.cer`

The export did not include a private key.

### Verification Command

```powershell
certutil -verify "$env:TEMP\wazuh-dashboard.cer"
```

### Observed Results

The output included:

- `CERT_TRUST_IS_PARTIAL_CHAIN`
- `CERT_E_CHAINING`
- `CERT_TRUST_REVOCATION_STATUS_UNKNOWN`
- `CERT_TRUST_IS_OFFLINE_REVOCATION`

It also reported that the issuer certificate could not be found.

### Interpretation

Windows could not construct a complete trusted chain from the presented certificate. The revocation error means the revocation check could not be completed; it does not establish that the certificate was revoked.

The certificate's issuer name differs from its subject name, so the certificate should not be assumed to be self-signed.

### Next Action

Obtain the intended issuing CA certificate from the authorized server configuration or administrator. Verify its provenance and certificate relationships before importing it into a Windows trust store.

Do not import the dashboard leaf certificate into the Trusted Root store as a shortcut.

## 5. No Matching Wazuh Certificate Found in Windows Stores

### Command

```powershell
$IssuerText = "Wazuh"

Get-ChildItem Cert:\LocalMachine\Root, Cert:\LocalMachine\CA |
    Where-Object {
        $_.Subject -match $IssuerText -or
        $_.Issuer -match $IssuerText
    } |
    Select-Object Subject, Issuer, Thumbprint, NotAfter, PSParentPath |
    Format-List
```

### Observed Result

No matching certificates were returned.

### Interpretation

No certificate matching the selected text was found in the two Local Machine stores inspected. This does not prove that the issuing certificate is unavailable elsewhere or that no other trust path exists.

### Next Action

Identify the actual issuing CA and check the intended trust chain. Install only the verified CA certificate in the appropriate store if required.

## 6. Certificate Details

The exported certificate showed:

- Subject: `CN=wazuh-dashboard, OU=Wazuh, O=Wazuh, L=California, C=US`
- Issuer: `L=California, O=Wazuh, OU=Wazuh`
- SAN: `IP Address=127.0.0.1`
- Thumbprint: `104BBC3DE72C8CE9B86D67F64A14E16EA80038D6`
- Signature algorithm: `sha256RSA`
- Validity: 1 August 2026 to 29 July 2036

The certificate was within its stated validity period during the investigation. Its validity dates do not resolve the hostname mismatch or incomplete trust chain.

## 7. RBAC Test Results Were Not Completed

### Initial Results File

The initial CSV contained:

| Test | Expected | Observed | Status |
|---|---|---|---|
| Analyst alert access | Allowed | Not tested | Inconclusive |
| Role administration | Restricted | Not tested | Inconclusive |

### Interpretation

These are placeholders. They must not be reported as successful or failed access tests.

### Next Action

After resolving certificate validation and confirming the intended test accounts:

1. Sign in with an authorized analyst account.
2. Test access to the alert data required for the analyst's duties.
3. Test whether role administration is restricted.
4. Record the actual response for each test.
5. Compare the observed behavior with the configured permissions.
6. Retain screenshots or sanitized API responses as evidence.

Do not modify production roles or save configuration changes merely to test restrictions.

