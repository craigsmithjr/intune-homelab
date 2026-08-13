# intune-homelab
Microsoft Intune home lab: device enrollment, dynamic Entra groups, BitLocker and Defender policies, and Win32/LOB app deployment — with exported policy JSON, endpoint verification scripts, and documented troubleshooting of the failures along the way.

# Tools Used 
| Tool | Purpose |
|---|---|
| Microsoft Intune | Endpoint management, policy, and app deployment |
| Microsoft Entra ID (P2 trial) | Identity, device objects, dynamic group membership |
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
than abstracting it behind Autopilot:

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

