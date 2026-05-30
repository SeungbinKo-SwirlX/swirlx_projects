# Per-user DCV — GUI access model for AWS-dispatched Fluent

**Status (2026-05-30)**: head node multi-session **chosen** for now. Login node
documented as the graduation path (not adopted). Thin DCV helper **spec'd, not
yet implemented**.

**Context**: ~5 colleagues. The heavy Fluent solve is dispatched to AWS hpc6a
(`cfd-aws-cluster` + `cfd-solver-fluent` plugins). This doc covers the *GUI
access* side — how each colleague visually inspects (monitoring + BC checks)
their dispatched Fluent without fighting over one shared workstation.

---

## 1. Why per-user DCV (and why the Web UI was demoted)

**Original problem**: 5 people share one workstation → one NICE DCV console
session per host → mouse/keyboard collision blocks Fluent use.

**Two solution attempts:**

1. **Fluent native Web UI (:5000)** — multi-browser concurrent view. Was the
   headline of the first plugin extraction (M2). **Demoted 2026-05-30** to an
   opt-in fallback because (a) per-user DCV gives each colleague a full GUI
   environment anyway, and (b) the Fluent FTM (Fault-Tolerant Meshing) workflow
   does not support the Web UI, so DCV is needed regardless. Web UI niche now =
   "quick browser peek from a DCV-less environment."
2. **per-user DCV** (chosen) — each colleague gets their own DCV session, so no
   input collision and a full Fluent GUI. Contention is solved at the infra
   level by giving everyone their own GUI env, not by sharing one.

**Boundary principle**: "how a human views the GUI" (DCV vs Web UI) is an
access/infra concern, kept separate from "Fluent control" (launch + PyFluent
drive). The plugins demote the Web UI from default; they do **not** absorb DCV
provisioning. DCV setup is infra/ops.

---

## 2. The key architectural fact: where the GUI renders

In the AWS path, Fluent's **cortex** (the host process that owns the GUI + scheme
interpreter + coordinates the MPI ranks) runs on whatever node you launch
`fluent` on. The MPI compute ranks spread to the hpc6a compute nodes via
`-cnf hosts`. The heavy numerical solve is on hpc6a; the **GUI rendering load is
tied to the cortex node** (software OpenGL / Mesa `llvmpipe`, since head and
compute are both **non-GPU** instances).

So "per-user DCV" = each of the 5 users needs a DCV session on a host where their
cortex renders.

---

## 3. Chosen model (2026-05-30): head node multi-session DCV

- NICE DCV on the head node, **Linux multi-session**: each colleague gets their
  own virtual DCV session (`dcv create-session --owner <user>`), each with an
  independent virtual display → **no mouse/keyboard collision** (exactly the
  goal). NICE DCV is free to use on EC2.
- Each colleague drives Fluent from their own session.
- **Why the head (not a separate login node) for now**: 5 users, light use
  (monitoring + BC checks), minimize cost/infra. Accepted tradeoff: the head
  also runs `slurmctld`, so GUI/DCV load is coupled to the scheduler (see Risk).

### Guardrails (because the GUI shares the host with slurmctld)
- **Cap concurrent sessions** ≈ Fluent license seats (concurrency is
  license-bound anyway).
- **Size the head** for N concurrent cortex + N software-GL renders.
- **Monitor slurmctld under load**: while a heavy op runs, on the head check
  `top` (CPU/mem) and `scontrol ping` (controller responsiveness). If
  `scontrol ping` lags or jobs stop scheduling → that's the signal to move to a
  login node.

### Risk (the reason a login node exists)
`slurmctld` is a **cluster-wide** resource. If the head bogs down under GUI/DCV
load, the failure is not "my mesh rotation is laggy" — it is **everyone's job
scheduling/execution** stalling. Higher severity class than per-user DCV
smoothness. Recoverable, but cluster-wide when it happens.

---

## 4. Current launch mechanism (interim) — `salloc` + `ssh -Y`

Today (before NICE DCV multi-session is stood up) GUI Fluent is launched via a
bash script run from the head node. Mechanics (from the colleague's script,
2026-05-30):

1. On head, ensure `DISPLAY=:0` (head's X server / DCV session).
2. `salloc -p queue1 -N <nodes> -J FLU<cores>` reserves hpc6a nodes; the wrapper
   re-runs itself (on head) inside the allocation.
3. Parse allocated nodes → build Fluent `-cnf node0:96,node1:54` from the core
   distribution. `MASTER_NODE = node0`.
4. On MASTER_NODE, runtime-patch sshd: `apt-get install` X11 libs +
   `sed -i 'X11Forwarding yes' / 'X11UseLocalhost no'` + `systemctl restart ssh`.
   (`X11UseLocalhost no` opens the forwarded display off loopback so the other
   compute node's ranks can reach it too.)
5. `ssh -Y <MASTER_NODE> "cd <wd>; fluent -t<cores> -cnf=<cnf>"` — launches
   Fluent on the compute node, GUI forwarded back.

### What this actually offloads (precise)

| Work | Runs on | Loads head? |
|---|---|---|
| Heavy solve (e.g. 150 cores) | node0 + node1 (hpc6a) | No |
| **Cortex CPU ops** (msh→fmd convert, case save/load) | node0 (hpc6a) | **No — moved off head** ✓ |
| **X server `:0` + DCV serving + forwarded display** | **head** | **Yes — still on head** |
| slurmctld | head | Yes |

- `node0` is **host(cortex) + 96 compute ranks** — not wasted. (Only an
  allocated-but-0-core node, e.g. the `0` in `"96 54 0"`, sits idle.)
- **`echo $DISPLAY`**: on head = `:0`; inside Fluent on the compute node =
  `localhost:10.0` (set by `ssh -Y` X11 forwarding).

### Is this already "cross-node X forwarding"? — Yes.
The `ssh -Y <MASTER_NODE> "fluent ..."` line **is** cross-node X11 forwarding:
the X client (Fluent) is on the compute node, the X server is on the head
(`:0`), traffic tunnels over ssh. Implication: the head is still the X server /
display endpoint, so moving the cortex to the compute node offloaded the
**cortex CPU ops** but **not** the display serving — and heavy 3D over *indirect*
GLX may still render on the head's `:0`. ⚠️ Exact GL rendering location depends
on direct/indirect GLX + `iglx` config; verify on the real setup if 3D
interactivity gets heavy.

### Fragility to retire
The per-launch `apt-get install … + sed sshd_config + systemctl restart ssh` on
the master node is a workaround for **ephemeral** compute nodes (X11 deps + sshd
config don't persist across allocations), so it re-patches every run (network
apt + sudo + sshd restart mid-flow). A **persistent** interactive host (login
node, X11 baked into the AMI) eliminates it.

---

## 5. Graduation path: login node (documented, not adopted yet)

A ParallelCluster **login node** (or a standalone small EC2, e.g. `m6a.*`) is a
persistent interactive host, **separate from the slurmctld head**, that mounts
the same FSx and submits to the same Slurm. It is still "one shared box, N
sessions, not per-user EC2" — just not the scheduler host.

### What it buys
- **(a) Decouple slurmctld**: head becomes Slurm-only; GUI/DCV load can't stall
  scheduling. *This is the main reason.*
- **(b) Kill the apt/sshd hack**: persistent host, X11 configured once into the
  AMI.
- **(c) Clean GL** (if cortex runs on the login node): local Mesa `llvmpipe`
  render in the user's own DCV session — no `ssh -Y`, no indirect-GLX fragility.
- hpc6a is then allocated **only** for the parallel solve.

### Two variants
- **V1 — minimal change**: keep the current compute-node-cortex model (cortex on
  hpc6a `node0`, which also computes), but point the DCV/X-server endpoint at the
  login node instead of the head. Head freed; still uses `ssh -Y` + the sshd
  patch.
- **V2 — fully clean**: cortex on the login node, compute ranks on hpc6a via
  `-cnf`. No forwarding, no sshd patch, local GL. Loses the "host node also
  computes" efficiency (marginal).

### Cost
One shared modest instance (e.g. `m6a`), stoppable when the team is offline.
hpc6a usage unchanged (solve only). For 5 light users this is small; weigh it
against the slurmctld-stability insurance.

### When to switch
Adopt a login node when the §3 head-node guardrail test fails — `scontrol ping`
lags or job scheduling stalls while users interact — or when the per-launch
apt/sshd patching becomes a maintenance burden.

---

## 6. Thin DCV helper — spec (deferred, not yet implemented)

Once NICE DCV multi-session is stood up (head or login node), a thin convenience
wrapper belongs in **`cfd-aws-cluster`** (infra layer). It lives in its own
module so `ClusterClient` stays focused on compute, and it manages session
**lifecycle only** — provisioning (instance / AMI / account) stays infra.

Proposed `cfd_aws_cluster/dcv.py`:
```python
@dataclass
class DcvSession:
    session_id: str
    owner: str
    host: str                        # DCV server ssh alias
    @property
    def web_url(self) -> str:        # https://{host}:8443/#{session_id}  (hint)

def list_dcv_sessions(cluster, *, host=None) -> list[DcvSession]
def ensure_dcv_session(cluster, *, session_id="cfd", owner=None, host=None) -> DcvSession
```
- Reuses `ClusterClient.ssh()` as transport; if the DCV host ≠ compute head, the
  caller passes a `ClusterClient(head_alias=dcv_host)` → **topology-agnostic**.
- Safety: `dcv list-sessions` / `dcv create-session` only — **no kill**. Composes
  with the package's "never pattern-kill" rule.
- 8443 reachability/tunnel is infra's concern; use
  `TunnelManager(cluster, remote_port=8443)` if a tunnel is needed.
- ⚠️ **Parser unverified**: the `dcv list-sessions` output format has not been
  seen on the real cluster. Finalize the parse against a real sample before
  shipping; until then, best-effort + flagged.

---

## 7. Decision log
- **2026-05-30**: Web UI demoted to opt-in (`web_ui` default `false`) across
  `cfd-solver-fluent/params.py`, `cfd_agent/.../aws/session.py`, and
  `cfd_agent/configs/t_merge_aws_demo.yaml` (flags/code preserved, never
  deleted; the `runtime.interactive` 5-phase `proceed.ok` pauses are untouched —
  they share the file but are an independent mechanism). Per-user DCV chosen as
  the GUI access model. **head multi-session** chosen for now (cost/scale);
  login node documented as the graduation path. Thin DCV helper spec'd;
  implementation deferred until DCV multi-session is live (need a real
  `dcv list-sessions` sample).

---

## 8. Setup procedure + manual workflow (deferred — colleagues actively using Fluent)

Setup is **not** executed yet (the cluster is in active use). This section is the
runbook to follow when it's free.

### Locked decisions (2026-05-30)
- **A — OS accounts**: shared **`ubuntu`** **now**. Collision-freedom comes from
  separate *virtual sessions* (separate displays), **not** from separate accounts —
  so one account + N virtual sessions already gives "no input collision." Per-user
  accounts are deferred as a later *isolation / security / audit* upgrade. The
  shared→per-user migration is **additive (no teardown)** — see the Migration
  subsection — so starting shared and moving later costs no rework.
- **B — session lifecycle**: **(i) admin pre-creates** the per-user virtual
  sessions, hardened with a **boot systemd unit** so they survive head reboots.
  Rationale: most colleagues drive Fluent **manually** (DCV in → run the salloc
  script by hand) without going through cfd_agent/cfd-aws-cluster, so there is no
  code path to trigger an on-demand helper — sessions must exist independently.
  The (iii) `ensure_dcv_session` helper (§6) stays a *later* convenience for the
  cfd_agent-driven path only.
- **C — client**: **browser default** (zero install, fine for monitoring). Native
  DCV client noted as an option for heavy interactive work (better keyboard/mouse
  fidelity for Fluent's middle-button rotate + shortcuts; its QUIC/UDP speed edge
  is **moot over an SSM TCP tunnel**).

### Pre-req discovery (read-only, safe while colleagues work) — run on the head
```bash
cat /etc/os-release          # distro + version (e.g. Ubuntu 22.04)
uname -r                     # kernel
dcv version 2>/dev/null || dcvserver --version   # DCV server version
dpkg -l | grep -iE 'nice-dcv|nice-xdcv'          # installed DCV pkgs (xdcv = virtual-session support)
systemctl is-active dcvserver                     # is the DCV server running
dcv list-sessions            # current sessions — capture this output: it finalizes the §6 helper parser
```

### Discovery result (2026-05-30) — multi-session already provisioned
Head: **Ubuntu 24.04.4 LTS** (kernel `6.8.0-1029-aws`), **Amazon DCV 2024.0
(r18131)**. Installed: `nice-dcv-server` + `nice-dcv-web-viewer` +
**`nice-xdcv`** (2024.0.631-1, virtual-session support). `dcvserver` is
**active**, and the current session is already `type:virtual` (owner `ubuntu`).
→ The server/packages are done; "going multi-session" is just **creating more
virtual sessions**. The install steps below are mostly N/A.

`dcv list-sessions` output format (confirmed → finalizes the §6 helper parser):
`Session: 'ID' (owner:OWNER type:TYPE)`.

### Setup steps (mostly already satisfied)
1. ~~Install `nice-dcv-server` + `nice-xdcv` + WM~~ — already installed (verify a
   WM/desktop exists for the virtual sessions). No `nice-dcv-gl` (no GPU → Mesa
   software GL).
2. ~~`systemctl enable --now dcvserver`~~ — already active. (DCV free + auto-
   licensed on EC2.)
3. **Boot systemd unit** (`After=dcvserver.service`) that creates one virtual
   session per colleague, owned by `ubuntu`:
   `dcv create-session --type virtual --owner ubuntu <name>`. (This is the only
   remaining setup task for sessions.)
4. Expose port 8443 — see the access-posture decision below.

### Manual user workflow (per colleague)
1. **Reach the DCV server** — `https://localhost:8443` after an SSM/SSH tunnel of
   8443 (matches today's private posture), **or** `https://<head-public-ip>:8443`
   if a public IP + SG-restricted 8443 is chosen.
2. **Log in as `ubuntu`** (shared password, DCV PAM auth).
3. **Open your assigned session URL** `…:8443/#<your-session>` → your own virtual
   session (own display). *Separation is by convention under shared ubuntu — use
   your own session URL; opening someone else's session-id shares their screen.*
4. Inside your session: terminal → run the salloc + `ssh -Y` script → Fluent GUI
   forwards to **your** session's display. Drive Fluent by hand. (`$DISPLAY` in the
   session is your virtual display, so the script's `:0` fallback never triggers.)
5. Disconnect anytime (close browser) → session + Fluent keep running; reconnect
   to the same URL later.

### ⚠️ Open decision — access posture (decide before relying on any SG rule)
Head public IP: `3.129.124.163`. **SG confirmed (2026-05-30): 8443/TCP open to
`0.0.0.0/0`** — the DCV login is reachable from the entire internet, and with
shared `ubuntu` the *effective firewall is a single shared password*. Colleagues
roam office/home (dynamic IPs), so SG IP-restriction is impractical. **Stopgap
regardless of the path chosen: a strong unique `ubuntu` password + `fail2ban` now;
don't leave weak-shared-password + `0.0.0.0/0`.**

- **Public IP + open 8443**: zero colleague setup (`https://3.129.124.163:8443`).
  BUT with **shared `ubuntu`**, the *effective firewall is one shared password* —
  the only gate between the internet and a Fluent desktop that holds AWS creds +
  FSx + cluster access. Single shared credential = single point of failure; the
  session shell can reach AWS; DCV's default cert is self-signed; audit logs all
  say "ubuntu". This re-opens the §8-A account decision: **if you go public,
  reconsider per-user OS accounts** (individual credential + revoke + audit), and
  at minimum **restrict the SG to known office IPs** + a strong unique password.
- **SSM/SSH tunnel 8443 → localhost**: no public head exposure (most secure);
  consistent with the current bridge. Each user runs a port-forward then browses
  `https://localhost:8443`. Friction = each colleague needs AWS creds + SSM plugin
  (small if they already use the cfd_agent path; larger for pure-DCV users).
- **Tailscale (recommended — a tailnet is already in use for the Mac-mini dashboard)**:
  install Tailscale on the head (joins the tailnet, gets a `100.x` IP), drop the
  public 8443 SG rule, colleagues reach the head's tailnet IP. WireGuard mesh that's
  **always-on** (no per-use connect → manual workflow truly unchanged), works from any
  roaming network, authenticates each device by **Tailscale identity** (a per-user gate
  at the network door, stronger than a shared password), and carries UDP so DCV QUIC
  works (native-client perf bonus). Scope with **Tailscale ACLs** (who may reach
  `head:8443`). Durability: install via a ParallelCluster **custom action + tagged auth
  key** so the head re-joins after reboot/replacement.
- **Generic VPN (AWS Client VPN / WireGuard)**: same idea if not using Tailscale —
  one-time client install + **per-use connect (1 click)**; workflow otherwise unchanged;
  `0.0.0.0/0` dropped, head reached on its private IP.
- **Decision driver (roaming team)**: SG IP-restriction is out, so it's **VPN**
  (most secure, +1-click-per-use) vs **public + harden** (`fail2ban` + real TLS +
  strong per-user passwords; zero per-use friction, residual internet exposure).
  Security-first → VPN; lowest-friction → public+harden. Either way, per-user
  accounts are the auth-layer upgrade and the `0.0.0.0/0` stopgap applies now.

### Migration: shared `ubuntu` → per-user accounts (documented, deferred)
**Additive — no teardown** of the DCV server, `nice-xdcv`, the multi-session setup,
the session-creating systemd unit *pattern*, the salloc+`ssh -Y` workflow, or the
network/SG. So starting shared now and migrating later costs no rework. Steps:

1. `sudo useradd -m -s /bin/bash <user>` + `sudo passwd <user>` × N.
2. Verify each account can run the workflow: **Slurm submit** (munge is host-level,
   so new users generally can submit — check for accounting associations); **`/fsx`
   access** (group/permissions); **Fluent license env** (`AWP_ROOT261` etc. set
   globally via `/etc/profile.d`). The manual DCV+salloc path needs **no** AWS creds
   (that's the cfd_agent dispatch path), so don't replicate `~/.aws`.
3. Retarget the session systemd unit: `dcv create-session --owner <user>` per user
   instead of all `--owner ubuntu`.
4. Colleagues re-bookmark their new session URL + log in as themselves.
5. **Durability**: manually-added users may not survive a ParallelCluster head
   update/replacement — keep the `useradd` loop in a reprovision script (same
   concern as the session-creation unit). LDAP/AD is the "proper" but
   overkill-for-5 alternative.

Benefit of migrating: individual revoke (offboarding = delete the account), per-user
audit, no shared password to rotate. This is the natural trigger if shared-account
**tracking/attribution** becomes a problem.

### Admin / vendor access is orthogonal (do not break it)
The AWS-setup vendor administers the cluster (adding compute nodes, Slurm config) via
**IAM + SSH/SSM to the head** — a *different door* than DCV 8443. Closing the public
8443 rule and adding Tailscale for DCV changes only the **GUI door**:
- Keep the change **scoped to 8443**; leave the SSH/SSM rules untouched. Do **not**
  tighten the whole SG to tailnet-only, or the vendor's admin path breaks.
- The vendor's **IAM role** (EC2 / ParallelCluster / Slurm via AWS APIs) is AWS-level
  and unaffected by any network/DCV change.
- `ubuntu` stays the admin/system account (per-user DCV accounts are *added*, not a
  replacement), so the vendor's `ubuntu` SSH access persists.
- If the vendor ever needs the DCV GUI itself (unlikely — their work is CLI/Slurm),
  invite their device to the tailnet with a scoped ACL.
