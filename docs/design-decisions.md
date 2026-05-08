# Design decisions

Managing QEMU VMs has two parts: generating the QEMU command line
(machine type, devices, disks, networking, kernel boot, passthrough)
and VM lifecycle (start, stop, restart, dependencies, logging,
resource control). systemd handles the lifecycle. Templates handle
the command line. No new daemon, no new wrapper.

Rendering and deploying the templates is left to the consumer.
The raw workflow is `minijinja-cli` + `systemctl`. Automation
layers are out of scope for this project.

The templates aim to be as unopinionated as possible. When a value
is hardcoded, it is either required by the underlying tool (QEMU,
virtiofsd, systemd), a workaround with documented reasoning, or a
sensible default that the user can override. This document lists
every such choice, the upstream reference that justifies it, and
how to change it when possible.

## Variable naming

Variable names match upstream flag or parameter names when a 1:1
mapping exists. When a variable controls a higher-level concept,
a descriptive name is used.

**`image`** — QEMU's flag is `-drive`. QEMU's own docs describe the
value as "which disk image to use with this drive." The sub-properties
(`image.file`, `image.format`, `image.cache`) match `-drive` property
names exactly. The name `image` describes what the user configures (a
disk image), while the properties map 1:1 to QEMU's `-drive`
properties.

**`cpus`** — QEMU's flag is `-smp`. The parameter inside `-smp` is
called `cpus=`. The variable matches the parameter name.

**`vsock_cid`** — QEMU's device property is `guest-cid`. The variable
adds the technology context (`vsock`) because `guest_cid` alone is
ambiguous. Maps to `-device vhost-vsock-pci,guest-cid=`.

**`ssh_port`** — No upstream equivalent. Maps to
`-nic hostfwd=tcp:127.0.0.1:<port>-:22`. The concept (SSH port
forwarding) spans multiple flag components.

**`pci_passthrough`** — No upstream equivalent. Maps to
`-device vfio-pci,host=<addr>`. Combines the bus type (PCI) with the
operation (passthrough).

**`autostart`** — No upstream equivalent. Inverted boolean: `false`
maps to `-S` (QEMU starts paused). QEMU has no "autostart" flag.

**`firmware`** — QEMU has no `-firmware` flag. Firmware selection is
implicit: pflash0 populated → UEFI, pflash0 absent → BIOS. QEMU's
own firmware specification (`docs/interop/firmware.json`) calls the
top-level concept `Firmware` and the internal function is
`pc_system_firmware_init()`. The sub-properties `code` and `vars`
match the OVMF file naming convention (`OVMF_CODE_4M.fd`,
`OVMF_VARS_4M.fd`). QEMU's spec uses the more verbose `executable`
and `nvram-template` but the OVMF file names are what users encounter
directly.

**`cloud_init.users[].password`** — Cloud-init's key is
`plain_text_passwd`. The variable uses `password` for simplicity. The
template maps it to `plain_text_passwd`.

## User-configurable (vars file)

Fully controlled by the user. See [vars.md](vars.md) for reference.

`cpu`, `accel`, `ram`, `cpus`, `machine_type`, `iommu`,
`firmware` (code, vars, vars_format), `gdb`, `autostart`,
`image` (file, format, cache, aio, discard, detect-zeroes),
`drives`, `ssh_port`, `vsock_cid`, `ssh_private_key`,
`kernel` (image, append, initrd),
`shares` (tag, mount, dir, translate_uid, translate_gid),
`share_transport`, `virtiofsd_binary`,
`cloud_init` (seed, locale, ssh_pubkey, users),
`pci_passthrough`, `nvme` (drives, subsystems),
`service` (any `[Service]` directive)

## systemd service properties

**`Documentation=`** — Set to
`man:qemu-system(1) man:systemd.service(5) man:systemd.kill(5)`.
References the upstream man pages relevant to the unit: QEMU's own
manual for the binary the service launches, systemd.service(5) for
the unit type and lifecycle directives, systemd.kill(5) for the
deliberate `KillSignal=SIGCONT` + `KillMode=mixed` choice the unit
makes. Surfaces in `systemctl status` and `systemctl help` so an
operator inspecting the unit reaches the canonical references
without leaving the CLI. Project-specific docs are not referenced
from the unit; only upstream man pages whose stability is
guaranteed by their respective maintainers. See:
`man systemd.unit`.

**`KillMode=`** — Set to `mixed`. Sends `KillSignal=` to the main
process. After the main process exits or `TimeoutSec=` elapses,
remaining processes in the cgroup receive `SIGKILL`. For VMs, the
main process is QEMU. After `ExecStop=` runs QMP graceful shutdown,
if QEMU exits cleanly, any leaked child processes are killed
immediately. The alternative `control-group` would send `KillSignal=`
to all processes simultaneously, which is unnecessary when only QEMU
needs the signal. See: `man systemd.kill`.

**`TimeoutSec=`** — Set to `2min`. Grace period for `ExecStop=`
before systemd sends `SIGKILL`. The systemd default
(`DefaultTimeoutStopSec=`) is 90s. VMs need longer because ACPI
powerdown triggers a full guest OS shutdown sequence (flushing
buffers, stopping services, unmounting filesystems). Override:

```yaml
service:
  TimeoutSec: 5min
```

See: `man systemd.service`.

**`Slice=`** — Set to `machine.slice`. All virtual machines and
containers registered with systemd-machined are placed in
machine.slice. Canonical cgroup placement for VMs.
See: `man systemd.special`.

**`PartOf=`**, **`Before=`**, **`WantedBy=`** — Set to
`machines.target`. Standard target for starting all containers and
virtual machines. `PartOf=` ensures VMs stop when machines.target
stops. `WantedBy=` enables auto-start. See: `man systemd.special`.

**`Type=`** — Set to `simple`. QEMU does not implement
`sd_notify()`. `notify` would be correct but requires the service to
call `sd_notify(READY=1)` after initialization. A patch adding
`sd_notify()` to QEMU was submitted (qemu-devel, 2025-12-17) but has
not been merged. When QEMU gains `sd_notify()` support, this should
change to `notify`. See: `man systemd.service`.

**`Restart=`** — Deliberately unset (systemd default `no`). Auto
restart on failure interacts badly with the BindsTo cascade
between `qemu-system@<vm>.service` and the per-share
`virtiofsd@<vm>-<tag>.service` units: a QEMU exit drops the
vhost-user sockets, virtiofsd self-exits, and the inactive deps
trip the death-link in `unit_is_bound_by_inactive`
(`src/core/unit.c`) the moment systemd tries to start the next
incarnation, so the auto-restarted VM gets stopped one second
after start. The same race the `restart_vms.yml` split
(`stop_vms.yml` -> mutate -> `start_vms.yml`) closes for explicit
restarts cannot be expressed by `Restart=`, which is a single
in-place transaction. Operators who actually want auto-restart
on a kernel-panic-style exit override via `service:
{ Restart: on-failure }`; with that they accept the race.
See: `man systemd.service`, `scripts/qemu-system-units/docs/`
"Restart cycle vs BindsTo race".

**`SyslogIdentifier=`** — Set to `qemu-system@%i`. Default journal
output (`journalctl --output=short`) prefixes each line with the
syslog identifier, which falls back to the executed process name
when this directive is unset (`man systemd.exec`). The systemd
reference templates `systemd-vmspawn@.service` and
`systemd-nspawn@.service` rely on that default because their
binary names match their service names and are informative as a
prefix. The QEMU template does not have that property: every
`qemu-system@<vm>.service` instance runs the same
`qemu-system-x86_64` binary, plus helper processes (`varlinkctl`,
`socat`, `ssh`) for `ExecStartPost=` and `ExecStop=`. Without
override, the default prefix is a mix of `qemu-system-x86_64[PID]`,
`varlinkctl[PID]`, `socat[PID]`, and parens-wrapped variants for
pre-exec failures, none of which carry the VM identity. Setting
`SyslogIdentifier=qemu-system@%i` pulls every per-service journal
record under one prefix that names both the service template and
the instance, e.g. `qemu-system@xarray[PID]`. Manager messages
(`systemd[PID]`) are emitted by systemd itself and remain
unaffected. The architecture suffix lost from the prefix is still
recoverable from the `_EXE` field (`journalctl --output=verbose`),
from `systemctl status` showing the full `ExecStart=` line, and
from QEMU's own self-identification in error messages
(`qemu-system-x86_64: -drive file=...`). See: `man systemd.exec`.

## machined registration

Each VM registers itself with machined at start-up via
`ExecStartPost=` running
`varlinkctl call <socket> io.systemd.Machine.Register <json>`.
Socket path: `/run/user/%U/systemd/machine/io.systemd.Machine`
(user scope) or `/run/systemd/machine/io.systemd.Machine`
(system scope). Required JSON fields: `name`, `class=vm`,
`service=qemu-system`, `leader=${MAINPID}`. Optional:
`vSockCid` when the vars file sets `vsock_cid`; `id` when the
vars file sets `uuid`.

### `id` correlates QEMU's UUID with machined's record

When `uuid` is set in the vars file, the same value is passed to
QEMU's `-uuid` flag (rendered into vm.env's `QEMU_ARGS`) and to
the Register call's `id` field
(`sd_json_dispatch_id128` in `src/machine/machine-varlink.c:133`).
The host sees it as `Id=` in `machinectl show <vm_name>`; the
guest sees it as `dmidecode -s system-uuid` and
`/sys/class/dmi/id/product_uuid`. A single user-supplied UUID
threads through both sides, so tooling that correlates by
machine UUID resolves to the same VM regardless of which side
queries. No auto-derivation: omit `uuid` and QEMU passes its
default (all-zeros) while the Register call omits `id`.

Registering `vSockCid` is what makes `ssh machine/<vm>` route
over AF_VSOCK. The shipped
`/etc/ssh/ssh_config.d/20-systemd-ssh-proxy.conf` hands the
`machine/*` pattern to `systemd-ssh-proxy`, which looks the
machine up over Varlink and reads the registered CID. Without
it, `systemd-ssh-proxy` exits with "Machine has no AF_VSOCK
CID assigned" (`src/ssh-generator/ssh-proxy.c`).

### Why two `ExecStartPost=` lines across two templates

The shared `qemu-system@.service.j2` renders once for all
instances and has no per-VM variables at that render time, so
its Jinja cannot branch on `vsock_cid`. It emits one
`ExecStartPost=` that registers without CID - correct for any
VM, always safe.

The per-VM `qemu-system-override.conf.j2` renders with
`vsock_cid` in scope. When set, it emits `ExecStartPost=`
(empty) followed by `ExecStartPost=-varlinkctl ... vSockCid:N`.
Systemd treats the empty directive as a reset of the
accumulated list, so the drop-in's call replaces the shared
template's call rather than running alongside. Net effect:
every VM makes one registration call, with CID if configured,
without otherwise. VMs that never set `vsock_cid` hit only the
shared template's call and behave exactly as before.

### Why Varlink and not `busctl`

The legacy DBus `RegisterMachine` method
(`src/machine/machined-dbus.c`) has a fixed `sayssus`
signature with no `vSockCid` field. `RegisterMachineEx`
accepts `VSockCID` but requires the leader identified by
`LeaderPIDFD` or `LeaderPID+LeaderPIDFDID`, neither of which
a shell script can synthesise. Varlink's
`io.systemd.Machine.Register` leader dispatcher accepts a
bare integer PID and machined acquires the pidfd daemon-side
(`json_dispatch_pidref` in
`src/libsystemd/sd-json/json-util.c`).

### Quoting and expansion

`${MAINPID}` uses the brace form because systemd's exec parser
expands `${VAR}` (but not `$VAR`) inside escape-quoted
arguments, letting the JSON body ride as one argv entry
without a `/bin/sh -c` wrapper. The drop-in inlines
`{{ vsock_cid }}` at Jinja render time rather than pulling
`${VSOCK_CID}` from the environment because the value is
already known per-VM.

The `-` prefix on `ExecStartPost=` keeps the VM running if
machined is unreachable. Registration is informational.

### `machinectl list` columns OS, VERSION, ADDRESSES are unsupported for VMs

Both `machine_get_os_release` and `machine_get_addresses` in
machined return `-EOPNOTSUPP` for `MACHINE_VM`
(`src/machine/machined-core.c:418` and `:323`). They handle
`MACHINE_HOST` directly and `MACHINE_CONTAINER` via
`namespace_fork` into the container's mnt+pid+root or net
namespace; neither path applies to a VM whose kernel and netstack
live behind a hypervisor boundary. The varlink protocol exposes
no per-machine update method either - `Register / List /
Unregister / Terminate / Kill / Open / OpenRootDirectory /
MapFrom / MapTo / BindMount / CopyFrom / CopyTo`
(`src/machine/machined-varlink.c:789-800`) - so a registered VM
has no way to push os-release or current addresses back to
machined post-Register. The `ifIndices` field in the Register
dispatch table (`machine-varlink.c:141`) is captured into the
`Machine` struct but `machine_get_addresses` for `MACHINE_VM`
short-circuits with `EOPNOTSUPP` before any per-class logic, so
it is currently inert.

Operator-side configuration cannot fill those columns. Closing
the gap requires an upstream patch adding push-style varlink
methods plus a guest-side notifier service over `AF_VSOCK`.

## QEMU workarounds

**`KillSignal=`** — Set to `SIGCONT`. QEMU on `SIGTERM` calls
`qemu_system_killed()` which sets `force_shutdown=true` and exits
immediately without ACPI powerdown. `SIGCONT` is a no-op signal that
keeps QEMU alive, giving `ExecStop=` time to send QMP
system_powerdown for proper ACPI shutdown. After `TimeoutSec=`,
systemd sends `SIGKILL` (via `KillMode=mixed`). Not
user-configurable. See: `man systemd.kill`.

**`ExecStop=`** — Sends QMP system_powerdown via socat for graceful
guest shutdown. QEMU does not translate `SIGTERM` to ACPI powerdown.
When `ssh_private_key` is defined, `ExecStop=` first attempts SSH
shutdown (guest-initiated poweroff) with QMP as fallback. Not
user-configurable (the mechanism is fixed; the SSH path is enabled by
setting `ssh_private_key` and `vsock_cid`).

## QEMU firmware

**`-drive if=pflash`** — OVMF firmware via pflash drives. QEMU's
firmware specification (`docs/interop/firmware.json`) defines the
`split` mode: the executable (CODE) is read-only and shared, the
NVRAM template (VARS) is cloned per-VM and configured read-write.
pflash0 holds the executable, pflash1 holds the per-VM NVRAM file.

QEMU selects firmware mode based on pflash0 presence: if pflash0 is
populated, QEMU enters pflash mode (UEFI); if absent, ROM mode
(SeaBIOS). There is no explicit firmware flag; the presence of pflash
drives is the entire mechanism. See: `hw/i386/pc_sysfw.c`
`pc_system_firmware_init()`.

pflash is created by `pc_system_flash_create()` which is part of the
PC machine initialization path. microvm does not call this function;
it calls `x86_bios_rom_init()` directly, even with `pcie=on`. The
`pcie=on` property on microvm only enables the GPEX PCIe host bridge
for PCI devices; it does not change the firmware initialization path.
ARM virt machines create pflash independently (256 KB sectors vs
x86's 4 KB).

The NVRAM file stores UEFI boot order, Secure Boot key databases
(PK, KEK, db, dbx), and guest-written variables. Each VM needs its
own writable NVRAM file for independent variable state. QEMU enforces
this with file-level locking. A second VM attempting to open the same
writable NVRAM file will fail to start. Create a per-VM copy from the
template:

```shell
cp /usr/share/OVMF/OVMF_VARS_4M.fd images/test-ovmf-vars.fd
```

The firmware spec also supports qcow2 format. An alternative to
copying is a qcow2 overlay with the template as backing file:

```shell
qemu-img create --format qcow2 \
  --backing /usr/share/OVMF/OVMF_VARS_4M.fd \
  --backing-format raw \
  images/test-ovmf-vars.qcow2
```

This saves disk space (only changed variables are stored) and keeps
the system template untouched. Use `format: qcow2` in the vars file
when using a qcow2 NVRAM file.

When the `firmware` section is absent from vars, no pflash drives are
rendered and QEMU defaults to SeaBIOS. Backward compatible.
User-configurable: set `firmware.code` and `firmware.vars`.

## QEMU device choices

**`-device virtio-*-pci-non-transitional`** — All virtio devices use
non-transitional variants. These only require `CONFIG_VIRTIO_PCI` in
the guest kernel. Transitional devices additionally require
`CONFIG_VIRTIO_PCI_LEGACY`, which custom kernels often disable. On
microvm, devices use the `-device virtio-*-device` suffix
(virtio-mmio transport). vhost-user-fs-pci has no non-transitional
variant (modern-only device). Not user-configurable (determined by
`machine_type`). See: `<qemu_binary> -device help`.

**`-device virtio-rng-*`** — Always present. Provides entropy to the
guest. Without it, `/dev/random` may block and boot can stall waiting
for entropy. Not user-configurable.

**`-nographic`**, **`-serial mon:stdio`** — Headless operation with
serial console on stdio. Standard interface for kernel development
VMs. The serial output goes to the systemd journal via stdout
capture. VGA, SPICE, and VNC are not supported. Not
user-configurable.

**`-nic user`** — User-mode networking (SLIRP). No host privileges
required. Port forwarding controlled via `ssh_port`. Alternative
host-guest transport via VSOCK (`vsock_cid`). TAP and bridge
networking are not supported (require root or network namespace
setup). The NIC model is determined by `machine_type`.
See: `man qemu-system`.

**`-object memory-backend-memfd,share=on`** — Required by virtiofs.
virtiofsd accesses guest memory via shared memory mapping. Without
`share=on`, virtiofsd cannot map guest memory. Only emitted when
virtiofs shares are defined. Not user-configurable.

**`LimitMEMLOCK=`** — Auto-computed as `<ram+256>M` when
`pci_passthrough` is defined. VFIO DMA mapping requires locked
memory. The 256M overhead covers QEMU allocations beyond guest RAM.
Override:

```yaml
service:
  LimitMEMLOCK: 8G
```

See: `man systemd.exec`.

## virtiofsd

**`--sandbox=`** — Set to `namespace` with `--uid-map :0:%U:1:` and
`--gid-map :0:%G:1:`. Provides PID, mount, network, and user
namespace isolation as an unprivileged user. The `--uid-map` maps
root inside the namespace to the service user outside (`%U`/`%G` are
systemd specifiers expanded at runtime). Without `--uid-map`,
virtiofsd warns "Couldn't set the process uid as root" because the
default 1-to-1 UID mapping does not include UID 0 (virtiofsd calls
`setresuid(0)` after creating the namespace). Override via per-share
env file: set `VIRTIOFSD_SANDBOX_ARGS=--sandbox=none`. See:
`/usr/libexec/virtiofsd --help`.

**`--xattr`** — Extended attribute support. Required for correct
POSIX semantics (security labels, capabilities, ACLs). Not
user-configurable. See: `/usr/libexec/virtiofsd --help`.

**`--no-announce-submounts`** — QEMU does not support submounts.
Prevents virtiofsd from announcing submount boundaries that the VMM
cannot handle. Not user-configurable. See:
`/usr/libexec/virtiofsd --help`.

**`--fd=3`** — Socket activation. systemd passes the listening socket
as FD 3 per the `sd_listen_fds` protocol. Not user-configurable.
See: `man sd_listen_fds`.

**`StopWhenUnneeded=yes`** — Set on `virtiofsd@.service`. Makes the
service auto-stop when no QEMU instance pins it. The per-VM drop-in
(`templates/qemu-system-override.conf.j2`) emits
`BindsTo=virtiofsd@%i-<tag>.service` for every share, which creates an
implicit reverse `BoundBy=` dependency on the virtiofsd unit. `BoundBy`
carries the `UNIT_ATOM_PINS_STOP_WHEN_UNNEEDED` atom
(`src/core/unit-dependency-atom.c:60`); the moment all pinning
dependencies go inactive, `unit_is_unneeded()`
(`src/core/unit.c:2179-2207`) returns true and systemd queues the
service for stop. The listening socket is not pinned by QEMU's
`BindsTo=`, so it stays active across stop/start cycles and
re-activates virtiofsd on the next QEMU connection. The stop side is
symmetric with the start side: socket activation handles start,
`StopWhenUnneeded=` handles stop, with no explicit lifecycle plumbing
between QEMU and virtiofsd in either direction. Without this directive,
virtiofsd survives QEMU and keeps stale share-directory bindings alive,
requiring an explicit `systemctl stop` of each per-share service to
force re-binding on the next QEMU connection. See: `man systemd.unit`.

**`Before=qemu-system@<vm>.service`** — Set on every per-instance
`virtiofsd@<vm>-<share>.service.d/override.conf` rendered from
`templates/virtiofsd-override.conf.j2`. Inverts the stop ordering
of the `BindsTo=virtiofsd@%i-<tag>.service` cascade emitted by the
per-VM `qemu-system-override.conf`. Without it, the stop
transaction `systemctl --user stop qemu-system@<vm>` enqueues both
qemu's stop and virtiofsd's `BoundBy=`-cascaded stop and runs them
in parallel: qemu's `ExecStop=ssh root@vsock/<cid> systemctl
poweroff` triggers a guest shutdown, but virtiofsd has already
torn down its vhost-user socket. The guest kernel logs
`virtio-fs: response too short (0)` on every outstanding fs
request, the unmount step in the guest's shutdown sequence hangs
in D-state on `/nix/store`, `/lib/modules`, and the data shares,
ACPI powerdown is never delivered, and qemu hits
`TimeoutSec=2min` followed by `SIGKILL`. Adding `Before=qemu` on
the virtiofsd side reverses to `After=virtiofsd` in the stop
direction (`man systemd.unit`: *"the inverse of the start-up order
is applied"*). The cascade still queues virtiofsd's stop in the
same transaction (`UNIT_ATOM_PROPAGATE_STOP`,
`src/core/unit-dependency-atom.c:60`), so failure-time
propagation is preserved; the ordering only changes when the
queued stop runs. Socket activation is unaffected because
`Before=` orders but does not pull starts; virtiofsd@.service
still starts only when qemu connects to its socket.
See: `man systemd.unit`.

## Restart cycle vs `BindsTo=` race

Reload the unit definition with stop, mutate, `daemon-reload`, start.
Never `systemctl restart` the VM service when the per-VM drop-in or
`vm.env` was just re-rendered. The reason is a race in
`JOB_RESTART`'s interaction with `BindsTo=virtiofsd@%i-<tag>.service`
that does not exist for plain `JOB_START`.

A `systemctl start` builds a transaction. The transaction walks
`UNIT_ATOM_PULL_IN_START` dependencies of the VM service
(`src/core/transaction.c:1055`) and queues `JOB_START` on each. The
atom mapping `[UNIT_BINDS_TO] = UNIT_ATOM_PULL_IN_START | ...`
(`src/core/unit-dependency-atom.c:31`) means every
`BindsTo=virtiofsd@%i-<tag>.service` line in the per-VM drop-in is
pulled in. If those services were inactive at construction time, an
actual `JOB_START` job sticks on each. While the VM transitions to
`UNIT_ACTIVE`, `unit_submit_to_stop_when_bound_queue()`
(`src/core/unit.c:2871`) fires, and the queue handler walks the
`BindsTo=` deps via `unit_is_bound_by_inactive()`
(`src/core/unit.c:2233`); any dep with `other->job != NULL` is
skipped, so the death-link does not trigger.

A `systemctl restart` is different in exactly one place. Per
`src/core/job.c:204`, `JOB_RESTART` is `JOB_STOP` followed by an
in-place change to `JOB_START` (`job_change_type(j, JOB_START)` at
`src/core/job.c:1027`). The change happens after the stop phase
finishes; transaction construction is **not** re-run. So the
`UNIT_ATOM_PULL_IN_START` walk only happens once, at the original
restart's transaction-construction time, when virtiofsd@ services
were `ACTIVE` because the prior VM was running. `JOB_START`
on an already-active dep is a no-op, no job sticks. During the stop
phase, QEMU's exit drops every vhost-user socket, virtiofsd self-
exits ("Client disconnected, shutting down"), and the
`virtiofsd@<vm>-<tag>.service` units transition to `inactive` with
no job pending. The patched-in `JOB_START` on the VM then runs.
When the VM reaches `UNIT_ACTIVE`, the death-link evaluator finds
`BindsTo=` deps in `(inactive, no job)` state and queues `JOB_STOP`
on the VM. The VM is killed two minutes later via
`TimeoutStopSec=`, mid-workload.

Splitting the cycle into `systemctl stop` and a separate
`systemctl start` makes each side a fresh transaction. The start-
side transaction sees virtiofsd@ services in `inactive` state, queues
`JOB_START` on each via `UNIT_ATOM_PULL_IN_START`, and the death-
link evaluator skips them. This is the canonical systemd shape for
"swap a unit definition and restart": stop, mutate, `daemon-reload`,
start. Anything tighter than that races. See:
`playbooks/roles/qemu_system_units/tasks/start_vms.yml`,
`man systemd.unit` (`BindsTo=`),
`man systemctl` (`restart`).

## Cloud-init

### Network configuration gap

Debian generic and genericcloud images ship with an empty
`/etc/netplan/` directory. They depend on cloud-init's fallback
network detection to generate netplan YAML at first boot. The
fallback scans all physical NICs and generates DHCP config for the
first one. This works when cloud-init boots with the image's own
distro kernel and initramfs.

With direct kernel boot and a custom initramfs, the fallback is
unreliable. The symptom: `/etc/netplan/` stays empty,
`systemd-networkd-wait-online.service` hangs, the guest has no
network connectivity.

The fix: provide a `network-config` file in the seed ISO. The
NoCloud datasource processes `network-config` before the fallback,
so it works regardless of boot mode. `files/network-config` is a
static file (no template rendering needed). Pass it to
`cloud-localds`:

```shell
cloud-localds --network-config=files/network-config \
  images/seed.iso /tmp/user-data /tmp/meta-data
```

The network config matches both predictable interface names (`en*`)
and classic names (`eth*`). Custom kernels without full PCI sysfs
support or with `net.ifnames=0` on the command line leave interfaces
as `eth0`. Both patterns are needed.

Images that do NOT need this fix:
nocloud (ships `/etc/netplan/90-default.yaml` baked in),
mkosi (ships `/etc/systemd/network/80-dhcp.network` baked in),
imageless (debootstrap includes network config).

See: https://cloudinit.readthedocs.io/en/latest/reference/network-config-format-v2.html

### user-data

Infrastructure requirements for the VM to be usable with the
service templates. Apply only when `cloud_init` is defined.

**`disable_root`**, **`PermitRootLogin`**,
**`PasswordAuthentication`** — Set to `false`, `yes`, `yes`. Root SSH
access required for non-interactive VM management (`ExecStop=` SSH
shutdown, automated provisioning). Not user-configurable.

**`locale`** — The `locale:` directive uses cloud-init's built-in
locale module, which handles per-distro differences automatically.
Debian: writes `/etc/locale.gen` and runs `locale-gen`. Fedora:
writes `/etc/locale.conf`. No distro-specific `write_files` or
`packages` needed.

**`growpart`** and **`resizefs`** — Cloud-init's built-in `growpart`
and `resizefs` modules run automatically (enabled in
`cloud_init_modules` by default on both Debian and Fedora). They
detect the root partition from the mount table and resize it to fill
the disk. No `runcmd` needed. Verified against upstream Fedora Cloud
image `/etc/cloud/cloud.cfg` which lists both modules.

**`runcmd`** — Runs `touch /etc/cloud/cloud-init.disabled` only.
Disables cloud-init after first run to prevent re-provisioning on
reboot. Not user-configurable. See: `man cloud-init`.

**`ssh_pwauth`** — Set to `true`. Password authentication fallback
for console access when no SSH key is configured. Not
user-configurable.

## 9P

**`security_model=`** — Set to `none`. Passes through file
permissions without UID/GID mapping. Correct for sharing host
directories where the user already owns the files. Other models
(mapped-xattr, passthrough) require root or change on-disk xattrs.
Not user-configurable. See: `man qemu-system`.

**`multidevs=`** — Set to `remap`. Remaps device/inode numbers for
filesystems spanning multiple host devices. Prevents inode collisions
when sharing directories that contain mount points. Not
user-configurable. See: `man qemu-system`.
