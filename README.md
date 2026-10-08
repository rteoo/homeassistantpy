<p align="center">
  <img src="https://raw.githubusercontent.com/rteoo/homeassistantpy/main/docs/homeassistantpy-icon.svg" width="128" alt="homeassistantpy icon">
</p>

<p align="center">
  A dependency-free Python client for Home Assistant, with REST and WebSocket
  access plus explicit JSON tools for agents.
</p>

<p align="center">
  <a href="https://github.com/rteoo/homeassistantpy/actions/workflows/tests.yml"><img src="https://github.com/rteoo/homeassistantpy/actions/workflows/tests.yml/badge.svg" alt="Test status"></a>
  <a href="https://github.com/rteoo/homeassistantpy/tags"><img src="https://img.shields.io/github/v/tag/rteoo/homeassistantpy?label=stable" alt="Stable tag"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT license"></a>
</p>

Point homeassistantpy at a Home Assistant instance with a long-lived token, then read
entity states, call services, browse registries, or watch events from Python,
the command line, or an agent runtime. It returns Home Assistant's own JSON
without discarding integration-specific fields, and every device action stays
disabled until a caller explicitly enables it.

## Highlights

- REST access to entity states, services, history, logbook, calendars, cameras,
  templates, configuration checks, events, intents, and Conversation.
- Dependency-free RFC 6455 WebSocket support for area, device, and entity-registry
  discovery plus bounded event observation.
- Framework-neutral JSON function definitions and dispatch for agent runtimes.
- Explicit action policy: service calls are disabled by default and can be
  restricted to an exact service allowlist.
- TLS verification, custom CA support, bounded responses, safe errors, disabled
  redirects, ignored ambient proxies, and no automatic mutation retries.
- A JSON-only command line: results on stdout, one JSON document on stderr.
- Python 3.11 or later, using only the standard library at runtime.

## Quick start

Install the current source from GitHub:

```powershell
python -m pip install git+https://github.com/rteoo/homeassistantpy.git@v1.0.0
```

For local development:

```powershell
git clone https://github.com/rteoo/homeassistantpy.git
cd homeassistantpy
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install .
```

After the first PyPI publication, install with `python -m pip install homeassistantpy`.
The existing `v0.2.0` GitHub release predates the rename and uses the old `homepy`
package and command names.

Use `python -m pip install -e .` for an editable development install. The
`homeassistantpy` console command and `python -m homeassistantpy` are both available afterwards.

## First use

1. In Home Assistant, open your profile and create a long-lived access token.
2. Provide it as `HA_TOKEN` through a secret manager or environment variable.
3. Set `HA_URL` if Home Assistant is not at `http://homeassistant.local:8123`.
4. Check the connection, then read some state:

```powershell
$env:HA_TOKEN = "<token supplied by your secret manager>"
$env:HA_URL = "https://ha.example.ts.net"
python -m homeassistantpy health
python -m homeassistantpy states --domain light
```

Never put a real token in source code, tool arguments, shell command arguments,
or a committed file. There is no token command-line option, and homeassistantpy does not
read `.env` files automatically.

## Connect

Configure the connection with environment variables or constructor arguments:

| Setting | Purpose | Default |
| --- | --- | --- |
| `HA_TOKEN` | Required Home Assistant access token | None |
| `HA_URL` | Full URL; takes precedence over `HA_HOST` | None |
| `HA_HOST` | Hostname, IPv4, or bracketed IPv6 address | `homeassistant.local` |
| `HA_PORT` | Override the connection port | 8123 for a bare host |
| `HA_TIMEOUT` | Socket timeout in seconds | 10 |
| `HA_CA_FILE` | Custom CA file for HTTPS | System trust store |

```python
import os

from homeassistantpy import HomeAssistant

token = os.environ["HA_TOKEN"]
ha = HomeAssistant(token, host="homeassistant.local")
print(ha.health())

lan = HomeAssistant(token, host="192.168.1.50")
tailnet = HomeAssistant(token, host="100.101.102.103")
https = HomeAssistant(token, host="https://ha.example.ts.net")
custom_port = HomeAssistant(token, host="192.168.1.50", port=8124)
```

A bare host uses HTTP port 8123. Full URLs retain their scheme and explicit
port; without one, the normal scheme port is used: HTTP 80 or HTTPS 443.
Reverse-proxy path prefixes are supported, and a trailing `/api` is accepted.

Use a reachable LAN address, Tailscale IP, MagicDNS hostname, or existing HTTPS
endpoint. Tailscale must already provide routing and does not replace Home
Assistant authentication.

## Read Home Assistant

The client exposes direct Python methods that return Home Assistant data without
discarding integration-specific fields:

```python
from homeassistantpy import HomeAssistant

ha = HomeAssistant.from_env()

states = ha.get_states(domain="light")
one_state = ha.get_state("light.desk")
services = ha.get_services()
areas = ha.get_areas()
devices = ha.get_devices()
entities = ha.get_entity_registry()
```

History requires a nonempty list of entity IDs. Timestamp arguments accept
timezone-aware Python `datetime` values or ISO strings with an explicit
timezone. Availability and permissions depend on the integrations enabled in
your Home Assistant instance and the token user.

For asynchronous runtimes, run synchronous calls in a worker thread:

```python
import asyncio

state = await asyncio.to_thread(ha.get_state, "light.desk")
```

Cancelling that coroutine does not cancel a request already running in the
worker thread or undo a device action.

## Control devices

Discover states and services before choosing an action:

```python
ha.call_service(
    "light",
    "turn_on",
    {"brightness_pct": 40},
    target={"entity_id": "light.desk"},
)

ha.call_service("scene", "turn_on", target={"entity_id": "scene.evening"})

forecast = ha.call_service(
    "weather",
    "get_forecasts",
    {"type": "daily"},
    target={"entity_id": "weather.home"},
    return_response=True,
)
```

`target` accepts entity, device, area, floor, and label selectors. These are
placed at the top level of the REST service body; duplicate selectors are
rejected rather than overwritten.

**`set_state()` does not operate a physical device.** It changes the state
representation held by Home Assistant. Use `call_service()` for device control.
The ordinary service-call result is a list of changed states; with
`return_response=True`, it also includes service response data. Read the target
state again when your workflow needs confirmation of its final state.

## Use from an agent

`AgentTools` exposes a narrow, framework-neutral set of JSON function
descriptors:

```python
from homeassistantpy import HomeAssistant
from homeassistantpy.agent import AgentTools

ha = HomeAssistant.from_env()
agent = AgentTools(
    ha,
    allow_actions=True,
    allowed_services={"light.turn_on", "light.turn_off", "scene.turn_on"},
    include_discovery=True,
)

tool_definitions = agent.tools
result = agent.dispatch("ha_get_state", {"entity_id": "light.desk"})
```

| Option | Default | Effect |
| --- | --- | --- |
| `allow_actions` | `False` | Shows and permits the `ha_call_service` tool |
| `allowed_services` | `None` | Exact `DOMAIN.SERVICE` names; empty denies every service |
| `include_discovery` | `False` | Adds area, device, and entity-registry tools |
| `include_events` | `False` | Adds bounded event collection |
| `allow_conversation` | `False` | Adds Conversation; see [Events and Conversation](#events-and-conversation) |

`allowed_services=None` permits any service only when actions are explicitly
enabled. Each allowlist entry must use the `DOMAIN.SERVICE` form; an entry that
could never match raises `TypeError`.

The allowlist limits service names, not individual entities or payloads. The
agent orchestrator remains responsible for user authorization and target
selection; tool schemas are not a security sandbox.

See [MCP compatibility](docs/mcp-compatibility.md) for the boundary between
homeassistantpy and Home Assistant's native MCP/Assist surfaces.

## JSON command line

Every command writes JSON. Successful output goes to stdout. Stderr holds at
most one JSON object: an `error` key on failure, with a nonzero exit status,
and a `warning` key for plain-HTTP connections.

```powershell
python -m homeassistantpy health
python -m homeassistantpy states --domain light
python -m homeassistantpy state light.desk
python -m homeassistantpy services
python -m homeassistantpy areas
python -m homeassistantpy devices
python -m homeassistantpy entity-registry
python -m homeassistantpy tools
python -m homeassistantpy --host 100.101.102.103 health
python -m homeassistantpy call light turn_on --target '{"entity_id":"light.desk"}' --allow-actions
python -m homeassistantpy tool ha_get_state --arguments '{"entity_id":"light.desk"}'
```

`python -m homeassistantpy tools` lists tool schemas without a token or network
connection. Add `--allowed-service DOMAIN.SERVICE` repeatedly to restrict an
enabled action tool. Use `--help` on a command for all options.

## Events and Conversation

WebSocket registry reads and event observation use separate connections. Event
streams are finite, closeable, and bounded:

```python
with ha.watch_events("state_changed", max_events=10, duration=15) as events:
    for event in events:
        print(event)

print(events.stop_reason)  # max_events, duration, or closed
```

`watch_events()` uses one connection, a 16 MiB message limit, a cumulative event
payload limit, and a one-second cleanup allowance. It does not reconnect,
retry, redirect, use ambient proxies or compression, or promise gap-free
delivery. The CLI emits one flushed NDJSON event per line:

```powershell
python -m homeassistantpy watch --event-type state_changed --max-events 10 --duration 15
python -m homeassistantpy tools --include-discovery --include-events
python -m homeassistantpy tool ha_collect_events --include-events --arguments '{"event_type":"state_changed","duration":5}'
```

Python callers may set either event limit to `None`, but at least one limit must
remain active. Agent event collection is always capped at 100 events and 30
seconds. A registered entity need not have a current state, and entity and
device area assignments can differ.

Conversation access is separately authorized:

```python
reply = ha.process_conversation("What time is it?", language="en")
```

At the agent and CLI boundaries, Conversation requires both
`allow_actions`/`--allow-actions` and `allow_conversation`/`--allow-conversation`.
It cannot be combined with an `allowed_services` collection, including an empty
one. The built-in `home_assistant` agent is selected at those boundaries;
direct Python callers may provide `agent_id`.

```powershell
python -m homeassistantpy conversation --text "What time is it?" --allow-actions --allow-conversation
```

An HTTP 200 response can still contain a structured Assist domain error; that
response is preserved. A timeout leaves the outcome unknown and is never
retried.

## API surface

| Methods | Purpose |
| --- | --- |
| `health`, `get_config`, `get_components` | Availability and configuration |
| `get_states`, `get_state` | Current entity states |
| `set_state`, `delete_state` | State representation changes |
| `get_services`, `call_service` | Discover and invoke integration actions |
| `get_events`, `fire_event` | Event discovery and firing |
| `get_history`, `get_logbook` | Recorded state changes and activity |
| `get_error_log`, `get_camera_image` | Error text and camera bytes |
| `get_calendars`, `get_calendar_events` | Calendars and events |
| `render_template`, `check_config`, `handle_intent` | Template, config, and intent operations |
| `get_areas`, `get_devices`, `get_entity_registry` | WebSocket registry discovery |
| `watch_events`, `process_conversation` | Bounded events and Conversation |

## Data safety and privacy

homeassistantpy talks only to the Home Assistant host you configure. It has no
telemetry, stores nothing on disk, and never logs tokens or response bodies.

- TLS certificate verification is enabled by default; use `ca_file` or
  `HA_CA_FILE` for a private CA.
- Plain HTTP to a non-loopback host emits `homeassistantpy.InsecureTransportWarning`
  because the bearer token is unencrypted. Prefer an HTTPS URL; if using a
  tunnel, verify the actual route. A private or Tailscale-looking address alone
  does not prove encryption. After verifying a tunnel, silence it with
  `warnings.simplefilter("ignore", homeassistantpy.InsecureTransportWarning)`. The CLI
  reports it as `{"warning": {"code": "insecure_transport", ...}}` on stderr.
- HTTP redirects are rejected, ambient HTTP proxies are ignored, and failed
  actions are never retried automatically.
- Errors omit credentials, request bodies, response bodies, URLs, and raw
  server/OS details. Response bodies are capped at 16 MiB.
- If a mutation times out or its response is lost, its outcome is unknown.
  Check the device state before deciding whether to issue another action.
- `TransportError.category` distinguishes DNS, refused-connection, timeout, TLS,
  and other network failures. HTTP errors expose `status_code`.

## Platform status and limitations

homeassistantpy is pure Python and runs anywhere Python 3.11 or later does. CI runs the
suite on Windows and Linux with Python 3.11 and 3.14.

- **Scope:** homeassistantpy does not automate browser clicks, edit dashboards or
  integration registries, or manage Supervisor. Those surfaces need separate
  clients.
- **Timeouts:** REST timeouts apply per socket operation, not per request.
  Synchronous OS DNS resolution can exceed the WebSocket setup deadline.
- **Live verification:** Live Home Assistant connectivity, Tailscale routing,
  household actions, and static type checking are outside the local test
  suite's verification boundary.

## Develop and build

Run the complete local verification gates from the repository root:

```powershell
python -m unittest discover -s tests -v
python -m compileall -q homeassistantpy tests
python -m homeassistantpy --help
```

Build the source archive and wheel when the existing build tools are available:

```powershell
python -m build --no-isolation
```

Artifacts are written to `dist/`. Tests use synthetic data and loopback
HTTP/WebSocket/TLS servers; they do not contact or operate a real Home Assistant
installation. See [PLAN.md](PLAN.md) for architecture and verification scope.

Release history is documented in [CHANGELOG.md](CHANGELOG.md).

## License

homeassistantpy is released under the [MIT License](LICENSE).

## Releasing

The `pyproject.toml` version must already be merged to `main`. From a clean checkout matching `origin/main`, run `python release.py --tag vX.Y.Z --dry-run`, then rerun without `--dry-run` and type the tag to confirm.

The release helper runs these gates locally: `python -m unittest discover -s tests -v`, `python -m compileall -q homeassistantpy tests`, and `python -S -m homeassistantpy --help`. It never bumps, commits, or deploys. A retry is safe only for the same tag when the existing tag points at the exact merged commit and no GitHub release exists; there is no service rollback because this project has no deployment target. The helper uses the current Python runtime; CI remains the authoritative cross-platform check.

### PyPI publishing

`.github/workflows/publish.yml` runs when a GitHub release is published. It tests
Python 3.11 and 3.14, requires a `vX.Y.Z` tag matching `pyproject.toml`, builds and
checks the source distribution and wheel, and smoke-tests the installed wheel
before uploading to PyPI. The publishing job uses OIDC Trusted Publishing;
no long-lived PyPI token is stored in GitHub.

Before the first publication, register a pending publisher on
[PyPI](https://pypi.org/manage/account/publishing/) with these exact values:

- PyPI project: `homeassistantpy`
- GitHub owner: `rteoo`
- GitHub repository: `homeassistantpy`
- Workflow filename: `publish.yml`
- GitHub environment: `pypi`

See [PyPI's pending publisher guide](https://docs.pypi.org/trusted-publishers/creating-a-project-through-oidc/).
Configure the repository's `pypi` environment with any desired reviewer and tag
restrictions before publishing a release. Existing releases do not automatically
rerun the workflow when it is added. Publish a new release from a tested commit
containing this workflow; the tag must match the manifest version. PyPI does not
allow replacing an already published version.
