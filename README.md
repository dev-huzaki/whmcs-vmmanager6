# WHMCS × VMmanager 6 modules — unofficial fork

> **Disclaimer.** This repository is **not an official ISPsystem
> release**. It is a community-maintained fork of the upstream
> `ispsystem_official` addon and the `vmmanager6` WHMCS server module,
> with a few practical additions for hosting providers. ISPsystem is not
> affiliated with this repo and does not provide support for it.

## What we added on top of the upstream module

1. **Block SMTP port 25 on every newly created VM.**
   A pair of `drop` firewall rules (TCP+UDP, inbound + outbound) is
   injected into the `POST /vm/v3/host` payload during `CreateAccount`,
   so the anti-spam policy is enforced on the cluster node (via
   nftables) from the moment the VM boots — no extra API round-trip and
   no reliance on the guest OS firewall.

2. **Admin-side toggle for port 25.**
   Two new entries in the service's *Module Commands* list:

   - **Block SMTP port 25** — `vmmanager6_BlockPort25`
   - **Unblock SMTP port 25** — `vmmanager6_UnblockPort25`

   The handlers read the current `firewall_rules` of the VM, strip any
   existing port-25 entries, and (for *Block*) append the canonical two
   drop rules. Operations are **idempotent** and preserve any other
   firewall rules set on the host by other tools.

3. **Correct `firewall_rules` endpoint.**
   The upstream docs and several third-party integrations get this
   wrong. `firewall_rules` belongs to `HostResourceParams`, so updates
   must go to `POST /vm/v3/host/{id}/resource` — sending them to the
   root `/vm/v3/host/{id}` fails with `PROXY-3008: Unexpected property`.

4. **VxLAN-count mode parity.**
   The new admin actions short-circuit to `success` when the product is
   configured with `It is VxLAN count` (`configoption14 = on`), matching
   the existing behavior of `SuspendAccount` / `ChangePackage` / etc.

## Repository layout

```
addons/ispsystem_official/        WHMCS addon — creates the mod_ispsystem
                                  mapping table (serviceid <-> external_id)
includes/ispsystem/6/             Shared API + utility library used by the
                                  server module (HTTP client, auth helpers,
                                  safeCall wrapper, task poller, etc.)
servers/vmmanager6/               WHMCS server module for VMmanager 6
  vmmanager6.php                    — main module contract
  lib/vmmanager6.php                — VM_API host_action helpers
  lib/VMMetricProvider.php          — usage metrics provider
  clientarea.tpl / statistic.tpl /
  ip.tpl / ip_change.tpl            — client-area Smarty templates
```

## What the module provides

The server module implements the full WHMCS service lifecycle against
the VMmanager 6 REST API (`/vm/v3/*`):

- `CreateAccount` — provisions a user (if missing) and a VM
- `SuspendAccount` / `UnsuspendAccount` — `host_stop` / `host_start`
- `TerminateAccount` — deletes the VM
- `ChangePackage` — CPU / RAM / disk / traffic / OS reinstall / IP
- `ChangePassword` — root password update
- `AdminSingleSignOn` / `ServiceSingleSignOn` — key-based SSO
- `MetricProvider` — usage statistics
- Power / reboot actions from both admin and client area
- `statistic` / `ip` / `ip_change` client-area subviews

## Requirements

- WHMCS with the ISPsystem global addon (`addons/ispsystem_official`)
  activated — it creates and migrates the `mod_ispsystem` table used by
  the server module to persist the `serviceid → external_id` mapping.
- A reachable VMmanager 6 installation with an administrator account
  (username = email, `@admin` role).
- PHP version supported by your WHMCS release.

## Installation

1. Drop the three directories (`addons/`, `includes/`, `servers/`)
   into the matching locations of your WHMCS root.
2. In WHMCS admin → **System Settings → Addon Modules**, activate
   **ISPsystem global module**.
3. Create a server of type **VMmanager 6** (hostname/IP, admin email,
   password) and attach it to the product.
4. Configure the product's module settings (cluster, source OS/image,
   vCPU, RAM, disk, traffic, IP pool, recipe, etc.).

## Contact

Found a bug, want an extra feature, or need help wiring this into your
WHMCS? Reach out on Telegram:

**[@huzaki_team](https://t.me/huzaki_team)**

Pull requests and issues are welcome.

## License

Original code is proprietary to **ISPsystem LLC** — see the headers of
the individual files for their license terms. The customizations in
this fork are provided as-is, without warranty of any kind.
