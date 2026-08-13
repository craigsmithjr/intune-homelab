# intune-homelab
Microsoft Intune home lab: device enrollment, dynamic Entra groups, Defender policies, and LOB app deployment — with exported policy JSON, endpoint verification scripts, and documented troubleshooting of the failures along the way.

# Tools Used 
| Tool | Purpose |
|---|---|
| Microsoft Intune | Endpoint management, policy, and app deployment |
| Microsoft Entra ID (P1 trial) | Identity, device objects, dynamic group membership |
| Microsoft 365 Admin Center | User and license management, Integrated apps |
| Oracle VirtualBox 7 | Windows 11 test VM (vTPM 2.0, UEFI, Secure Boot) |
| Windows 11 Pro | Managed endpoint |

## Step 1 — Tenant Setup and User Creation

The lab runs on a Microsoft 365 Business Premium trial, which bundles Entra ID
P1 and Intune — enough to cover device management, Conditional Access, and
dynamic group membership without additional licensing.

Signing up provisions a tenant with a default `.onmicrosoft.com` domain
(`CSIT040.onmicrosoft.com` here) and a Global Administrator account. That admin
account is used for tenant configuration only; a separate standard user was
created to represent an end user and to test enrollment and policy from a
non-privileged account.

### Creating the test user

**Microsoft 365 admin center → Users → Active users → Add a user**

<img width="1386" height="901" alt="image" src="https://github.com/user-attachments/assets/5e337351-a8b2-4291-941d-faa442c26824" />


The wizard continues through **Product licenses** and **Optional settings**. A
Business Premium license was assigned here so the account could enroll a device
— **Intune enrollment fails without a license, even when the user is in scope
for automatic enrollment**, and the resulting error gives no indication that
licensing is the cause.

## Step 2 — Building the Endpoint and Enrolling It

A Windows 11 VM was built in Oracle VirtualBox to act as the managed endpoint.
Hardware settings were configured up front, since BitLocker later depends on
them:


Getting TPM and UEFI set before installing Windows avoids reconfiguring the boot
environment afterwards, which changes the measurements BitLocker relies on.

### Enrolling the device

Enrollment was performed manually from the endpoint to see the full flow rather
than abstracting it behind Autopilot.

**Settings → Accounts → Access work or school → Connect**, signing in with the
standard user account created in Step 1.

<img width="903" height="773" alt="image" src="https://github.com/user-attachments/assets/a40abb3b-20d5-424a-927d-8f62c4aa48c4" />
<img width="720" height="640" alt="image" src="https://github.com/user-attachments/assets/ed35f20b-2088-4ffe-b13d-60322fa6ba7c" />


This performs a **Microsoft Entra join** — the work account becomes the primary
identity on the device, rather than a registration layered on top of a personal
account. The distinction matters:

| | Entra registered | Entra joined |
|---|---|---|
| Device owner | The user | The organization |
| Primary sign-in | Personal account | Work account |
| Wipe scope | Selective (company data) | Full device |
| Auto-enrollment | Not by default | Yes, when MDM scope permits |

Automatic Intune enrollment on join is driven by **Entra ID → Mobility (MDM and
WIP) → Microsoft Intune → MDM user scope**, which was set to **All**. Scope
alone is not sufficient — the signing-in user must also hold an Intune license,
or enrollment fails.

### Verification

The device appeared under **Intune → Devices → All devices** and under **Entra
ID → Devices**, confirming both the directory object and MDM management.

<img width="1909" height="948" alt="image" src="https://github.com/user-attachments/assets/96b86a2c-0b17-44f7-a820-ab5e052c0223" />


### Single Sign-On

Once the device is Entra joined, signing into a Microsoft app requires no
password prompt — the app picks up the existing session and authenticates
silently.



https://github.com/user-attachments/assets/383d45a3-4e35-4ac1-9588-efa0c03f5be9



This is a property of the join type rather than of Intune itself. Entra join
issues the device a **Primary Refresh Token (PRT)** at Windows sign-in, and
Microsoft apps redeem that token for access rather than prompting for
credentials. An Entra *registered* device gets SSO only for the work account and
its apps; a *joined* device gets it device-wide.

Practical effect: fewer prompts, no stored passwords in individual apps, and
authentication that Conditional Access can evaluate centrally — the same PRT
carries device state, which is what later allows a policy to require a compliant
device before granting access.

### Confirming Intune Management

With enrollment complete, the device appears under **Intune → Devices → All
devices** as a fully managed endpoint.
<img width="1575" height="865" alt="image" src="https://github.com/user-attachments/assets/702fe995-773e-4cec-a158-0cf0ab92f40e" />

## Step 3 — Group Design

Policy in Intune is assigned to Entra groups, so group structure determines what
is manageable. A dynamic device group was created as the baseline target for all
corporate Windows endpoints.
(device.deviceTrustType -eq "AzureAD")

<img width="1643" height="927" alt="image" src="https://github.com/user-attachments/assets/7e803cd6-7a1f-4eec-bc9f-a1bb8147d1f4" />

| Property | Value |
|---|---|
| Name | `SG-DEV-WIN-Corporate` |
| Type | Security |
| Membership type | Dynamic Device |
| Source | Cloud |

### Naming convention

`SG-DEV-WIN-Corporate` breaks down as:

| Segment | Meaning |
|---|---|
| `SG` | Security group — distinguishes it from M365 groups and distribution lists |
| `DEV` | Contains **devices**, not users |
| `WIN` | Windows platform |
| `Corporate` | Corporate-owned, as opposed to personal |

### Why dynamic

Membership is rule-driven rather than manual. A newly enrolled device that
matches the rule joins automatically and receives the baseline policy set with
no administrative action. **Dynamic rules processing status: Succeeded**
confirms the rule evaluated without error.

## Step 4 — Endpoint Security: Defender Antivirus

With the target group in place, a Defender Antivirus policy was created to serve
as the security baseline for all corporate Windows devices.

**Endpoint security → Antivirus → Create Policy → Windows → Microsoft Defender
Antivirus**

<img width="1641" height="876" alt="image" src="https://github.com/user-attachments/assets/0d6e049b-1867-40f7-8643-b2f2ab42d6c6" />


| Property | Value |
|---|---|
| Name | Security baseline / Defender AV policy |
| Platform | Windows |
| Profile | Microsoft Defender Antivirus |
| Assigned to | `SG-DEV-WIN-Corporate` |

### Why Endpoint Security rather than Settings Catalog

Defender settings exist in both places — Endpoint Security policies are built on
the same underlying configuration engine. Endpoint Security was chosen because
it provides a curated view with dedicated per-setting reporting, and it is where
a reviewer expects to find the workload.

**Configuring the same setting in both places creates a conflict**, and when two
policies set the same value differently, neither applies — the setting reverts
to unmanaged. Security Baselines are a common source of this, since they set
hundreds of Defender values on their own. One owner per setting.

### Verification

The console reported **Succeeded: 1** with no errors, conflicts, or
non-applicable devices.

<img width="1642" height="942" alt="image" src="https://github.com/user-attachments/assets/37799ebb-77e1-4726-96a9-311eca4acb48" />

## Step 5 — Application Deployment

Zoom was deployed as the test application. Three approaches were attempted; the
first two failed for different reasons, and the failures are more instructive
than the eventual success.

### Attempt 1 — Microsoft Store app (new)

**Apps → Windows → Add → Microsoft Store app (new)**

The app creation failed immediately with:

> The selected app does not have a valid latest package version.

The Store (new) app type is backed by winget and requires the Store catalog
entry to carry an installable package. Zoom's listing points at the vendor's own
installer rather than a packaged Store app, so there is nothing for Intune to
retrieve. Nothing was misconfigured — this app type simply cannot deploy this
app.

### Attempt 2 — Line-of-business app, assigned to a user group

**Apps → Windows → Add → Line-of-business app**, uploading `ZoomInstallerFull.msi`
(202 MiB, the per-machine MSI from Zoom's IT admin download page — not the
per-user `.exe` from the standard download page).

<img width="1629" height="865" alt="image" src="https://github.com/user-attachments/assets/27d8c8a2-0970-40cd-af57-88f38d7d527e" />


Assigned as **Required** to a group containing the test *user*. The install
reported Pending, then Failed. Nothing appeared in Program Files, Program Files
(x86), or any Uninstall registry key.

### Attempt 3 — Same app, assigned to a device group

The assignment was moved from the user group to `SG-DEV-WIN-Corporate`. After a
sync, the install completed and Zoom appeared on the endpoint.

<img width="1632" height="828" alt="image" src="https://github.com/user-attachments/assets/b727d732-21e5-4378-918e-d35cc137c150" />

<img width="1008" height="757" alt="image" src="https://github.com/user-attachments/assets/10d9e884-c2e7-48e2-8703-ef06592d3306" />


## Conclusion

This lab covers the core of cloud-native Windows endpoint management: tenant and
user setup, Entra join and Intune enrollment, dynamic group design, an Endpoint.
Security baseline, and application deployment — each verified from the endpoint
rather than from the admin console alone.

The most useful thing it produced was the failures. The Microsoft Store (new) app
type could not deploy Zoom at all. A line-of-business MSI install started in the
correct context and silently never completed, with no error surfaced beyond
"Failed" and no entry in the log that most guidance points to
A recurring theme: **the console is not the endpoint.** Intune reporting lags the
device by up to an hour, a policy showing "Succeeded" only confirms delivery
rather than effect.

### Next steps

- **Compliance policies and Conditional Access** — the highest-value addition.
  A compliance policy keyed on BitLocker and Defender state, enforced by a
  Conditional Access rule that blocks non-compliant devices from Microsoft 365.
- **Windows Update rings** — deferrals, deadlines, and update reporting.
- **Windows Autopilot** — zero-touch provisioning with an Enrollment Status Page.
- **Cross-browser policy** — importing Chrome and Firefox ADMX templates, since
  Edge-only hardening leaves an obvious bypass.
