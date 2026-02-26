# TrustViz — Domain Trust Mapping Visualizer

A standalone HTML tool for visualising Active Directory domain trust relationships from PowerView / PowerSploit output. No installation, no server — open the HTML file in any browser.

---

## Quick Start

1. Open `trust-visualizer.html` in a browser
2. Paste PowerView output into the input box **or** drop a CSV file onto the drop zone
3. Click **▶ Render** (or press `Ctrl+Enter`)
4. Click **Sample** to see it working immediately with demo data

---

## Collecting Trust Data (PowerView Queries)

### Recommended — Full Recursive Mapping

Maps all trusts across every reachable domain in the forest, recursively following trust chains.

```powershell
# Requires PowerView (PowerSploit)
Import-Module .\PowerView.ps1

# Full recursive trust map — best starting point
Get-DomainTrustMapping | Format-List
```

### Export to CSV (recommended for large environments)

```powershell
# Export to CSV — drag and drop directly into TrustViz
Get-DomainTrustMapping | Export-Csv -NoTypeInformation -Path trusts.csv

# With explicit credential (useful from non-domain machine)
$cred = Get-Credential
Get-DomainTrustMapping -Credential $cred | Export-Csv -NoTypeInformation -Path trusts.csv
```

### Current Domain Only

```powershell
# Trusts for the current domain only (no recursion)
Get-DomainTrust | Format-List

# Specific domain
Get-DomainTrust -Domain corp.local | Format-List

# Forest-level trusts only
Get-ForestTrust | Format-List
```

### Targeting a Specific Forest or DC

```powershell
# Against a specific domain controller
Get-DomainTrust -Domain corp.local -Server dc01.corp.local | Format-List

# Map trusts from a specific forest
Get-ForestDomain -Forest partner.com | Get-DomainTrust | Format-List
```

### Native Windows (no PowerView required)

```powershell
# Built-in nltest — useful when PowerView is unavailable
nltest /domain_trusts /all_trusts /v

# netdom — enumerate trusts for a domain
netdom query trust /domain:corp.local

# dsquery — pull trust objects directly from AD
dsquery * -filter "(objectClass=trustedDomain)" -attr name trustDirection trustType trustAttributes
```

### Built-in PowerShell (ActiveDirectory module)

```powershell
# Requires RSAT / AD module
Get-ADTrust -Filter * | Select-Object Name,Direction,TrustType,TrustAttributes | Format-List
```

---

## Input Formats Supported

### 1. Raw PowerView `Format-List` output

Paste directly from terminal. Blank lines between records are used as delimiters.

```
SourceName      : corp.local
TargetName      : child.corp.local
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : WITHIN_FOREST
TrustDirection  : Bidirectional
WhenCreated     : 1/15/2019 9:00:00 AM
WhenChanged     : 3/12/2024 2:15:00 PM

SourceName      : corp.local
TargetName      : partner-ext.com
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : FOREST_TRANSITIVE
TrustDirection  : Bidirectional
```

### 2. CSV (from `Export-Csv`)

Standard CSV with headers. Required columns: `SourceName`, `TargetName`. Optional but useful: `TrustDirection`, `TrustAttributes`, `TrustType`, `WhenCreated`, `WhenChanged`.

```csv
SourceName,TargetName,TrustType,TrustAttributes,TrustDirection
corp.local,child.corp.local,WINDOWS_ACTIVE_DIRECTORY,WITHIN_FOREST,Bidirectional
corp.local,partner-ext.com,WINDOWS_ACTIVE_DIRECTORY,FOREST_TRANSITIVE,Bidirectional
```

### 3. TSV (tab-separated)

Same structure as CSV but tab-delimited. Auto-detected.

---

## Understanding the Output

### Node Types

| Icon | Colour | Meaning |
|------|--------|---------|
| 🏛 | Cyan | Root domain (most outbound trusts, no parent) |
| 🏰 | Cyan | Forest root involved in cross-forest trust |
| 🏢 | Green | Child domain (same forest, `WITHIN_FOREST`) |
| 🌐 | Yellow | External domain (non-transitive or unknown) |
| 🏰 | Orange | Cross-forest trust target (`FOREST_TRANSITIVE`) |
| 🔑 | Red | MIT / cross-realm Kerberos trust |

### Edge Types

| Style | Meaning |
|-------|---------|
| Solid line, arrows both ends | Bidirectional trust |
| Solid line, single arrow | Outbound trust (this domain trusts the target) |
| Dashed line | Inbound trust (target trusts this domain) |
| Green | Within-forest child trust |
| Cyan | External trust |
| Orange | Forest transitive trust |
| Red | MIT / cross-realm |

### Trust Direction (from the source domain's perspective)

- **Outbound** — the source domain *trusts* the target. Users in the target can authenticate to resources in the source.
- **Inbound** — the target domain trusts the source. Users in the source can authenticate to resources in the target.
- **Bidirectional** — both directions apply.

---

## Example Attack Scenarios to Look For

**1. Unconstrained delegation + forest trust**
Look for computers marked as unconstrained delegation in a domain that has a bidirectional forest trust. Coerce DC auth across the trust → capture cross-forest DC TGT.

**2. SID filtering disabled on external trusts**
If `TREAT_AS_EXTERNAL` or `QUARANTINED_DOMAIN` attributes are *absent* on an external trust, SID history may be honoured. Accounts with forged SID history could escalate across the trust boundary.

**3. Inbound-only trusts from high-value domains**
A domain you've compromised that has an inbound trust from a more sensitive domain means users from that sensitive domain may be authenticating to your resources — NTLM relay, credential capture.

**4. Old / unexpected trusts**
Check `WhenCreated` values in CSV exports. Trusts created years ago and never reviewed (especially `NON_TRANSITIVE` or `FOREST_TRANSITIVE` to external domains) are frequently forgotten and under-monitored.

---

## Tips

- **Large environments:** Use `Export-Csv` and drop the file rather than pasting raw text — cleaner parsing and handles thousands of records.
- **Reposition nodes:** Drag any node to fix its position; the rest of the graph continues to settle around it.
- **Zoom to fit:** Click `⊡` after the graph stabilises to auto-frame everything.
- **Filter by node:** Click a domain node to see all of its trust relationships listed in the sidebar.
- **Tune layout:** Use the force/repulsion sliders if nodes are overlapping in dense environments. Increase repulsion (more negative) to spread nodes apart.

---

## Requirements

- Any modern browser (Chrome, Firefox, Edge, Safari)
- No internet connection required after initial load (D3.js loaded from cdnjs.cloudflare.com on first open — save the page locally or self-host D3 for fully offline use)
- No installation, no dependencies, no server
