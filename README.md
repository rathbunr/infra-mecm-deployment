# MECM Standalone Primary Site – Ansible Automation Platform 2.7 Playbook

Production-quality, fully unattended installation of a **standalone primary site** of Microsoft Endpoint Configuration Manager (MECM / SCCM) Current Branch on a single Windows Server 2025 host.

## Architecture Summary

| Component              | Details                                                                 |
|------------------------|-------------------------------------------------------------------------|
| Control plane          | Ansible Automation Platform 2.7 (containerized Execution Environments) |
| Connection             | WinRM over HTTPS (5986) + Kerberos                                      |
| Target OS              | Windows Server 2025 (domain-joined)                                     |
| SQL                    | SQL Server 2022 Standard (local, default instance)                      |
| WSUS                   | Update Services role using local SQL (not WID)                          |
| Certificates           | Enterprise subordinate CA (no self-signed)                              |
| Site type              | Standalone Primary Site                                                 |

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
    label: Enterprise CA Server FQDN
  - id: ca_template_name
    type: string
    label: Certificate Template Name (e.g. WebServer or custom MECM template)
required:
  - ca_server
  - ca_template_name
```

**Injector Configuration**

```yaml
extra_vars:
  ca_server: "{{ ca_server }}"
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

- `inventory/hosts.yml` – define the target host and connection settings.
- `inventory/group_vars/mecm_servers.yml` – all site-specific values (site code, media UNC paths, SQL directories, boundary groups, etc.).

**Critical variables you must set before launch:**

| Variable              | Description                                      |
|-----------------------|--------------------------------------------------|
| `mecm_site_code`      | 3-character site code (e.g. P01)                 |
| `mecm_site_name`      | Friendly site name                               |
| `sql_media_unc`       | UNC path to SQL Server 2022 media                |
| `mecm_media_unc`      | UNC path to MECM Current Branch baseline media   |
| `mecm_prereq_unc`     | UNC path to MECM redistributable prerequisites   |
| `domain_name`         | AD DNS domain                                    |
| `mecm_boundary_groups`| List of boundary groups + AD sites               |

## Execution Environment

Build the EE from the provided `execution-environment.yml`:

```bash
ansible-builder build -f execution-environment.yml -t mecm-ee:latest
# Push to your private automation hub / registry
```

## Operator Guide – AAP 2.7

1. **Import Project**
   - Projects → Add → Source Control (Git) or Manual.
   - Point at this repository / upload the `mecm-standalone` directory.

2. **Create Credentials**
   - Create instances of the three Custom Credential Types defined above.
   - Also create a standard **Machine** credential (or use the domain service account) that can authenticate via Kerberos to the target.

3. **Build / Publish Execution Environment**
   - Use the supplied `execution-environment.yml`.
   - Publish to Automation Hub or a private registry.
   - Create an Execution Environment object in AAP that points at the image.

4. **Inventory**
   - Create an inventory and add the Windows host.
   - Attach the `group_vars` values (or use an inventory source that populates them).

5. **Job Template**
   - Name: `MECM Standalone Primary Install`
   - Inventory: the inventory containing the target.
   - Project: the imported project.
   - Playbook: `site.yml`
   - Execution Environment: the MECM EE.
   - Credentials: Machine + the three Custom Credential Types.
   - Extra variables (optional overrides).
   - Enable **Privilege Escalation** if required by your WinRM configuration (usually not needed with Kerberos + domain admin rights).

6. **Launch**
   - Run the job template.
   - Use tags (`prereqs`, `certs`, `sql`, `wsus`, `mecm`, `postconfig`) to re-run individual phases if needed.
   - Monitor the job output; on MECM setup failure the playbook automatically surfaces the last 200 lines of `C:\ConfigMgrSetup.log`.

## Idempotency & Safety

- Every role uses `creates:`, registry checks, `win_stat`, or `when:` guards.
- Schema extension and System Management container creation are performed only when missing.
- SQL and MECM setup are skipped when the respective services / registry keys already exist.
- Handlers restart only the services that actually need it.
- Secrets are never logged (`no_log: true` on sensitive tasks).

## Post-Installation Notes

- After the playbook completes, open the Configuration Manager console on the site server (or a remote admin workstation) and verify:
  - Management Point, Distribution Point, and Software Update Point roles.
  - Certificate binding on the Default Web Site.
  - Network Access Account.
  - Boundary Groups.
  - Client Push settings.
- Review logs:
  - `C:\ConfigMgrSetup.log`
  - `%ProgramFiles%\Microsoft Configuration Manager\Logs\*.log`
  - SQL setup logs under the SQL media folder / `%ProgramFiles%\Microsoft SQL Server\...\Setup Bootstrap\Log`

## References

- Microsoft Docs – MECM Setup command-line options and unattended INI reference
- Microsoft Docs – SQL Server ConfigurationFile.ini
- Microsoft Docs – WSUS post-install with SQL
- Red Hat – Ansible Automation Platform 2.7 Execution Environments
- ansible.windows / microsoft.ad collection documentation

---

**Disclaimer**: This playbook follows official Microsoft and Red Hat guidance. Always test in a non-production lab before production use. Ensure the install account has the necessary rights (local admin on the site server, schema admin / enterprise admin for the one-time schema extension, permissions on the System Management container, etc.).
