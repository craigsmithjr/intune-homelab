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

![Add a user – Set up the basics](images/01-add-user-basics.png)

| Field | Value |
|---|---|
| Display name | John Smith |
| Username | `john.smith@CSIT040.onmicrosoft.com` |
| Password | Auto-generated |
| Change password at first sign-in | Enabled |

The wizard continues through **Product licenses** and **Optional settings**. A
Business Premium license was assigned here so the account could enroll a device
— **Intune enrollment fails without a license, even when the user is in scope
for automatic enrollment**, and the resulting error gives no indication that
licensing is the cause.

No admin roles were assigned. The account is a standard user by design, so that
policy and app assignment can be tested against a realistic, non-privileged
account rather than against Global Admin.

