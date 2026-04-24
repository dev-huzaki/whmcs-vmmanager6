# WHMCS × VMmanager 6 modules

WHMCS integration for ISPsystem VMmanager 6, based on the official
`ispsystem_official` addon plus the `vmmanager6` server module, with a
small set of customizations on top.

## What's in this repo

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

The server module implements the full WHMCS service lifecycle against
the VMmanager 6 REST API (`/vm/v3/*`):

- `CreateAccount`   — provisions a user (if missing) and a VM
- `SuspendAccount` / `UnsuspendAccount` — `host_stop` / `host_start`
- `TerminateAccount` — deletes the VM
- `ChangePackage`   — CPU / RAM / disk / traffic / OS reinstall / IP
- `ChangePassword`  — root password update
- `AdminSingleSignOn` / `ServiceSingleSignOn` — key-based SSO
- `MetricProvider`  — usage statistics
- Power/reboot actions from both admin and client area
- `statistic` / `ip` / `ip_change` client-area subviews

## Customizations on top of the upstream module

### 1. Block SMTP port 25 on every newly created VM
Applied as part of `POST /vm/v3/host` during `CreateAccount`, so the
anti-spam policy is in place before the VM ever boots. Two drop rules
(inbound + outbound, TCP+UDP, port 25) are added to the host's
`firewall_rules` — enforced on the cluster node via nftables.

### 2. Admin-side toggle for SMTP port 25
Two entries in `AdminCustomButtonArray`:

- **Block SMTP port 25** — `vmmanager6_BlockPort25`
- **Unblock SMTP port 25** — `vmmanager6_UnblockPort25`

Both handlers:

- read the current `firewall_rules` via `GET /vm/v3/host?where=(id EQ ...)`,
- strip any existing port-25 entries (so block is idempotent and unblock
  only touches our own rules),
- push the resulting set back via `POST /vm/v3/host/{id}/resource`
  (the correct `HostResourceParams` endpoint — the root
  `/vm/v3/host/{id}` does not accept `firewall_rules`),
- wait for the resulting VMmanager task to reach `complete`, bailing
  out with a `LogicError` on `fail`.

All unrelated firewall rules the operator may have set on the host are
preserved across block/unblock cycles.

### 3. VxLAN-count mode parity
Every admin action (including the new port-25 toggles) short-circuits
to `success` when the product is configured with
`It is VxLAN count` (`configoption14 = on`), matching the existing
behavior of `SuspendAccount` / `ChangePackage` / etc.

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

## License

Proprietary — ISPsystem LLC. See file headers for details.
