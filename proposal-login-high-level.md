# Proposed login flow: confined login manager and session snaps

## Status and scope

**Under discussion, not selected or implemented.** Overview from 2026-11-24
of the brokered alternative in
[proposal-login-manager.md](proposal-login-manager.md#model-b-broker-launched-leader-in-its-own-snap).
The earlier [leader-inside-login-manager model](proposal-login-manager.md#model-a-leader-inside-login-manager-confinement)
and the [host-managed login-provider alternative](proposal-base-integration.md#login-provider-packaging-alternatives)
remain alternatives.

This document owns the overview diagrams, not a separate API or design.
`login-manager`, `session-manager` and `desktop-session` are provisional
interface names. The diagrams assume stock `core26`, a confined GDM and a
GNOME session provider; they are not observations of the running VM.
The greeter, initial setup and non-GNOME desktops need their own validation.

Supporting source revisions and inspection limits are in the
[login-manager evidence](proposal-login-manager.md#status-and-evidence).
The existing PAM and leader/monitor behavior is recorded in
[login research](research-session-login.md); the proposed broker has not been
implemented or tested. The 2026-11-24 SSH attempt did not authenticate, so
there is no fresh VM verification behind these diagrams.

## Connections and roles

Arrows mean **plug connects to slot**, not process parentage or data flow.
The login provider handles privileged authentication; the session provider's
app runs as the authenticated user with separate, narrower authority.

```text
 LOGIN-MANAGER SNAP                         HOST / SNAPD
 +-----------------------------+           +-----------------------------+
 | login-manager plug ---------+---------->| login-manager slot          |
 |                             |           | PAM, credentials, seat,     |
 |                             |           | launch-service API access   |
 |                             |           +-----------------------------+
 | desktop-session plug -------+-----+
 +-----------------------------+     |
                                     v
 SESSION PROVIDER SNAP
 +-----------------------------------------------------------------------+
 | desktop-session slot                                                  |
 |   Session files associated with a registered bootstrap app             |
 |                                                                       |
 | Bootstrap app: gnome-session -> gnome-session-init-worker               |
 |   session-manager plug ------------------------------+                 |
 |                                                     |                 |
 | Desktop apps and unit definitions                    |                 |
 |   gnome-session-service, gnome-shell, monitor, ...     |                 |
 +-----------------------------------------------------+-----------------+
                                                       |
                                                       v
                                           HOST / SNAPD
                                           +-----------------------------+
                                           | session-manager slot        |
                                           | User-desktop startup,       |
                                           | state and lifetime control  |
                                           +-----------------------------+
```

The provider may be the session snap or a runtime snap exposing a real snap
app. Executing its content inside GDM would not activate that app's plugs.
The greeter UI does not automatically receive `login-manager` authority.
See [interface roles](proposal-login-manager.md#interface-roles) and
[trust concessions](proposal-login-manager.md#purpose-and-trust).

## Session publication and registration

The new publication interface would mount individual session files read-only
in the login-manager snap's namespace. It would not merge whole provider
directories or expose their user units to the host manager automatically.

```text
 Provider's desktop-session slot
       |
       +-- per-file mount profile ----------------------------------+
       |                                                            v
       |   LOGIN-MANAGER SNAP NAMESPACE
       |   /usr/share/wayland-sessions/<provider-session>.desktop
       |     Exec=<generic launch client> <registered-session-id>
       |
       +-- trusted app registration --------------------------------+
                                                                    v
           HOST LAUNCH-SERVICE LOOKUP
           <registered-session-id> -> <provider snap.app + revision>

 Separate registration:
 provider's user units / activation metadata
       |
       +----> host user-manager / bus registration, before startup
```

The entry above is illustrative; whether the provider supplies it or snapd
generates it is open. Missing mount targets, collisions, revision ownership
and helper packaging belong to
[base integration](proposal-base-integration.md#session-provider-publication-and-launch-integration).
Generic client/service packaging is undecided; desktop names and provider
paths must not be hard-coded into stock `core26`.

## Authentication and startup

This is a process/IPC sketch, not a final cgroup layout. `(U)` identifies
the same user manager in both places. GDM opens the PAM
session before launching the client. The independent launch service then
starts the bootstrap with the login user's credentials and its own snap
sandbox; it does not perform a second PAM login.

```text
 LOGIN-MANAGER SNAP                         HOST LOGIND
 confined                                  independent service
 +---------------------------------+       +------------------------------+
 | GDM                             |       | Registers login/seat/VT      |
 |  |                              |       |                              |
 |  +-- privileged PAM worker      |       | Starts or reuses             |
 |       +-- authenticate          |       | user@UID.service (U)         |
 |       +-- establish groups      |       |                              |
 |       +-- pam_open_session()    +------>|                              |
 |       |                         |       +------------------------------+
 |       +-- fork; change UID      |
 |            |                    |
 |            v                    |
 |       gdm-wayland-session       |
 |            |                    |
 |            +-- execute Exec=    |
 |                 |               |
 |                 v               |
 |       launch client / proxy     |
 |       login user; STAYS ALIVE   |
 +-----------------+---------------+
                   |
                   | system D-Bus: launch(session-id, login context)
                   | AppArmor gates authorized login-manager snap apps
                   v
 +---------------------------------------+
 | HOST SESSION-LAUNCH SERVICE            |
 | system-D-Bus activated                 |
 |                                       |
 | Resolve registered app/revision       |
 | Establish login user's credentials    |
 | Launch through normal snap entrypoint |
 | Return handle; watch launched process |
 +------------------+--------------------+
                    |
                    | independent snap sandbox
                    v
 SESSION PROVIDER SNAP                              HOST USER MANAGER (U)
 +---------------------------------------+          +----------------------+
 | Confined bootstrap app                |          | systemd --user       |
 |                                       |          |                      |
 | gnome-session                         |          | Registered desktop   |
 |   prepare local environment           |          | units already known  |
 |   exec init-worker                    |          |                      |
 |              |                        |          |                      |
 |              v                        | StartUnit| gnome-session@...    |
 | gnome-session-init-worker              +--------->|   target             |
 |   start desktop target                |          +----------+-----------+
 |   monitor lifetime; STAY ALIVE         |                     |
 +---------------------------------------+                     | snap
                                                               | entrypoints
                                                               v
                                                    +----------------------+
                                                    | Confined desktop apps|
                                                    | Monitor, session     |
                                                    | service, Shell, ...  |
                                                    | Each app's policy    |
                                                    +----------------------+
```

The proxy remains GDM's session command; the bootstrap is **not its child**.
A successful launch or `StartUnit` reply is not session completion. The
service returns a handle/lifetime connection rather than holding one D-Bus
method call open for the whole login. See the
[launch contract](proposal-login-manager.md#launch-service-authorization-and-lifetime).

Provider registration and shared environment preparation must precede the
services that consume them. The [session-setup proposal](proposal-session-setup.md#proposed-separation)
describes registration, preparation and later display/audio publication
using the earlier model's final-exec choice; adapting that ordering to the
brokered launch remains required. A new user manager is not guaranteed:
SSH, linger or another login may already have started it.

## Two lifetime connections

Keep the generic login-lifetime channel separate from GNOME's existing FIFO.
`gnome-session-init-worker` is the bootstrap leader, not GDM's PAM worker or
the separate `gnome-session-service` desktop process.

```text
 GENERIC LOGIN LIFETIME

 GDM worker
     | watches child
     v
 gdm-wayland-session
     | watches child
     v
 launch proxy <===== broker lifecycle API =====> launch service
                                                    | watches process
                                                    v
                                            GNOME init-worker

 GNOME-INTERNAL DESKTOP LIFETIME

 GNOME init-worker <===== existing FIFO =====> GNOME monitor
                                                   |
                                                   | user-systemd service
                                                   v
                                            desktop unit graph
```

No inherited GNOME FIFO is required for this first version. A passed FD could
implement the separate broker connection, but FD passing does not bypass
AppArmor receive/use checks. Duplicate endpoints must not prevent hangup.
Details are in [runtime and lifetime requirements](proposal-login-manager.md#runtime-environment-and-session-lifetime).

## Stop and logout

Termination must travel from GDM's process tree to the independent leader.
The broker can send the signal; the proxy need not gain general cross-snap
signal-sending authority.

```text
 GDM / login-scope termination
          |
          | SIGTERM reaches helper/proxy
          v
     launch proxy -- stop request --> launch service
                                           |
                                           | SIGTERM
                                           v
                                    GNOME init-worker
                                           |
          +--------------------------------+-------------------------+
          |                                                          |
          v                                                          v
 D-Bus: start shutdown target                               FIFO notification
          |                                                          |
          v                                                          v
    systemd --user                                             GNOME monitor
          |                                                          |
          +--------------------> desktop teardown <------------------+
```

Normal desktop logout travels back as completion, not as an explicit Unix
signal from the leader to GDM. FIFO hangup is an FD event, not Unix `SIGHUP`.

```text
 Desktop logout / user-systemd teardown
      |
      v
 GNOME monitor exits, closing FIFO
      |
      | FIFO hangup
      v
 GNOME init-worker waits for teardown, then exits
      |
      v
 Launch service observes exit and reports completion
      |
      v
 Launch proxy exits -> Wayland helper exits -> GDM closes PAM / cleans up
```

Proxy disappearance must also trigger teardown if it cannot request a stop.
Broker failure must not orphan the desktop or leave a proxy waiting forever.
The exact cgroup/logind association, crash recovery and FD protocol remain
open. Acceptance checks belong to
[login-manager validation](proposal-login-manager.md#validation-before-claiming-support),
including generated policy for every bootstrap/monitor app, fresh and reused
user managers, refresh, disconnect and recovery.
