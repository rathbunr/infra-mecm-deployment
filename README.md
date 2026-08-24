# MECM Standalone Primary Site – Ansible Automation Platform 2.7 Playbook

Production-quality, fully unattended installation of a **standalone primary site** of Microsoft Endpoint Configuration Manager (MECM / SCCM) Current Branch on a single Windows Server 2025 host.

## Architecture Summary

| Component     | Details                                                                |
| ------------- | ---------------------------------------------------------------------- |
| Control plane | Ansible Automation Platform 2.7 (containerized Execution Environments) |
| Connection    | WinRM over HTTPS (5986) + Kerberos                                     |
| Target OS     | Windows Server 2025 (domain-joined)                                    |
| SQL           | SQL Server 2022 Standard (local, default instance)                     |
| WSUS          | Update Services role using local SQL (not WID)                         |
| Certificates  | Enterprise subordinate CA (no self-signed)                             |
| Site type     | Standalone Primary Site                                                |

## AAP Custom Credential Types

Create the following **Custom Credential Types** in AAP (Administration → Credential Types).

### 1. MECM Service Accounts

**Input Configuration**

```yaml
fields:
  - id: sccm_install_account
    type: string
    label: MECM Install Account (DOMAIN\user)
  - id: sccm_install_password
    type: string
    label: MECM Install Account Password
    secret: true
  - id: sccm_naa_account
    type: string
    label: Network Access Account (DOMAIN\user)
  - id: sccm_naa_password
    type: string
    label: Network Access Account Password
    secret: true
  - id: sql_sa_password
    type: string
    label: SQL SA Password
    secret: true
required:
  - sccm_install_account
  - sccm_install_password
  - sccm_naa_account
  - sccm_naa_password
  - sql_sa_password
```

**Injector Configuration**

```yaml
extra_vars:
  sccm_install_account: "{{ sccm_install_account }}"
  sccm_install_password: "{{ sccm_install_password }}"
  sccm_naa_account: "{{ sccm_naa_account }}"
  sccm_naa_password: "{{ sccm_naa_password }}"
  sql_sa_password: "{{ sql_sa_password }}"
```

### 2. AD Certificate Services

**Input Configuration**

```yaml
fields:
  - id: ca_server
    type: string
    label: Enterprise CA Server FQDN (e.g. ca01.example.com)
  - id: ca_name
    type: string
    label: CA Common Name (e.g. Contoso Issuing CA 01) — NOT the hostname
  - id: ca_template_name
    type: string
    label: Certificate Template Name (e.g. WebServer or custom MECM template)
required:
  - ca_server
  - ca_name
  - ca_template_name
```

**Injector Configuration**

```yaml
extra_vars:
  ca_server: "{{ ca_server }}"
  ca_name: "{{ ca_name }}"
  ca_template_name: "{{ ca_template_name }}"
```

### 3. Domain Join / Admin

**Input Configuration**

```yaml
fields:
  - id: domain_admin_user
    type: string
    label: Domain Admin User (DOMAIN\user)
  - id: domain_admin_password
    type: string
    label: Domain Admin Password
    secret: true
  - id: domain_name
    type: string
    label: DNS Domain Name (e.g. example.com)
  - id: domain_ou
    type: string
    label: Target OU DN (optional)
required:
  - domain_admin_user
  - domain_admin_password
  - domain_name
```

**Injector Configuration**

```yaml
extra_vars:
  domain_admin_user: "{{ domain_admin_user }}"
  domain_admin_password: "{{ domain_admin_password }}"
  domain_name: "{{ domain_name }}"
  domain_ou: "{{ domain_ou | default('') }}"
```

## Inventory & Variables

- `inventory/hosts.yml` – target host and connection settings.
- `inventory/group_vars/mecm_servers.yml` – all site-specific values.

**Critical variables you must set before launch:**

| Variable               | Description                                                      |
| ---------------------- | ---------------------------------------------------------------- |
| `mecm_site_code`       | 3-character site code (e.g. P01)                                 |
| `mecm_site_name`       | Friendly site name                                               |
| `mecm_product_id`      | Product key or `EVAL` for evaluation                             |
| `sql_media_unc`        | UNC path to SQL Server 2022 media                                |
| `mecm_media_unc`       | UNC path to MECM Current Branch baseline media                   |
| `mecm_prereq_unc`      | UNC path to MECM redistributable prerequisites                   |
| `adk_media_unc`        | UNC path to Windows ADK installer directory                      |
| `adk_winpe_media_unc`  | UNC path to Windows ADK WinPE add-on installer directory         |
| `odbc_driver_msi_unc`  | UNC path to `msodbcsql18.msi` (>= 18.4.1.1)                     |
| `vcredist_x64_unc`     | UNC path to VC++ 2015-2022 Redistributable (x64)                |
| `vcredist_x86_unc`     | UNC path to VC++ 2015-2022 Redistributable (x86)                |
| `domain_name`          | AD DNS domain                                                    |
| `domain_netbios`       | NetBIOS domain name                                              |
| `mecm_boundary_groups` | List of boundary groups + AD sites                               |

## Prerequisite Media Staging

Stage these on a UNC share accessible by the install account:

1. **SQL Server 2022 Standard** — full media (ISO extracted)
2. **MECM Current Branch baseline** — extracted from ISO or VLSC
3. **MECM redistributable prerequisites** — via `setupdl.exe` or manual download
4. **Windows ADK** — `adksetup.exe` and payload
5. **Windows ADK WinPE add-on** — separate download
6. **Microsoft ODBC Driver 18 for SQL Server** — `msodbcsql18.msi` (x64) version >= 18.4.1.1
7. **Visual C++ 2015-2022 Redistributable** — both `vc_redist.x64.exe` and `vc_redist.x86.exe`

## Execution Environment

```bash
ansible-builder build -f execution-environment.yml -t mecm-ee:latest
podman push mecm-ee:latest <registry>/mecm-ee:latest
```

## Operator Guide – AAP 2.7

1. **Import Project** — Projects → Add → Git or Manual; point at this repo.
2. **Create Credentials** — instances of the three Custom Credential Types above, plus a Machine credential for Kerberos.
3. **Build / Publish EE** — `ansible-builder` with the supplied `execution-environment.yml`.
4. **Inventory** — create inventory, add the Windows host, set `group_vars`.
5. **Job Template** — playbook `site.yml`, attach all four credentials, select the EE.
6. **Launch** — use tags (`prereqs`, `certs`, `sql`, `wsus`, `mecm`, `postconfig`) to re-run phases.

## Prerequisite Installation Sequence

| Step | Phase        | What it does                                            |
| ---- | ------------ | ------------------------------------------------------- |
| 1    | `prereqs`    | Validate connectivity, DNS, domain membership           |
| 2    | `prereqs`    | Install Windows Features (single transaction)           |
| 3    | `prereqs`    | Disable IE ESC, configure firewall rules                |
| 4    | `prereqs`    | Verify .NET Framework 4.8+                              |
| 5    | `prereqs`    | Install Windows ADK + WinPE add-on                      |
| 6    | `prereqs`    | Install Microsoft ODBC Driver 18 (>= 18.4.1.1)         |
| 7    | `prereqs`    | Install Visual C++ 2015-2022 (x64 + x86)               |
| 8    | `prereqs`    | Extend AD schema (`extadsch.exe`)                       |
| 9    | `prereqs`    | Create System Management container + delegate           |
| 10   | `certs`      | Request PKI certificate (with SAN) from enterprise CA   |
| 11   | `certs`      | Bind certificate to IIS (HTTPS 443)                     |
| 12   | `sql`        | Install SQL Server 2022 unattended                      |
| 13   | `sql`        | Post-config: TCP/IP, memory, firewall, cleanup INI      |
| 14   | `wsus`       | Install WSUS role with SQL backend                      |
| 15   | `wsus`       | Run `wsusutil.exe postinstall`                          |
| 16   | `mecm`       | Stage prereqs, template INI, `setup.exe /script`        |
| 17   | `postconfig` | Configure MP, DP, SUP, NAA, boundaries, client push    |

## Required Permissions

- **Local Administrator** on the site server
- **Schema Admin + Enterprise Admin** for the one-time AD schema extension
- **Full Control** on the System Management container (playbook creates this)
- **sysadmin** on the SQL instance (granted via ConfigurationFile.ini)

## References

- [MECM prerequisites for installing sites](https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/install/prerequisites-for-installing-sites)
- [Site and site system prerequisites](https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/configs/site-and-site-system-prerequisites)
- [Prerequisite checks](https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/install/list-of-prerequisite-checks)
- [Unattended setup script reference](https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/install/command-line-script-file)
- [AAP 2.7 Execution Environments](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/)
- [ansible.windows collection](https://docs.ansible.com/ansible/latest/collections/ansible/windows/)
- [microsoft.ad collection](https://docs.ansible.com/ansible/latest/collections/microsoft/ad/)

---

**Disclaimer**: Always test in a non-production lab before production use.
