# Telegram Bot Platform - Project Documentation

A multi-service Telegram bot platform built around a hand-crafted Layer 7 load balancer.
Every user message becomes a real HTTP webhook that flows through the load balancer,
giving a genuine production traffic to observe and monitor.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Overview](#2-architecture-overview)
3. [File Structure](#3-file-structure)
4. [Component Reference](#4-component-reference)
   - 4.1 [Load Balancer](#41-load-balancer)
   - 4.2 [Service Handlers](#42-service-handlers)
   - 4.3 [Shared State Store](#43-shared-state-store)
   - 4.4 [Live Dashboard](#44-live-dashboard)
5. [How Traffic Flows](#5-how-traffic-flows)
6. [How Routing Works](#6-how-routing-works)
7. [Health Checking](#7-health-checking)
8. [TLS Termination](#8-tls-termination)
9. [Configuration System](#9-configuration-system)
10. [Deployment Layout on Oracle Cloud](#10-deployment-layout-on-oracle-cloud)
11. [Observability and Metrics](#11-observability-and-metrics)
12. [Known Edge Cases](#12-known-edge-cases)
13. [Build Order](#13-build-order)

---

## 1. Project Overview

This project is a hobby platform that teaches load balancing, reverse proxying, and
distributed systems through direct, hands-on experience with real internet traffic.

The core learning vehicle is the **load balancer**, which you build entirely from scratch.
It is not a wrapper around Nginx, HAProxy, or any existing proxy software. Every byte of
routing logic, every health check, every header rewrite, and every connection management
decision is code you write yourself.

The platform gives the load balancer something meaningful to protect: a Telegram bot with
real utility commands. Because the bot is live on Telegram's network, real users generate
real, unpredictable traffic. You do not write traffic simulators. You share your bot in a
Telegram group, and the traffic comes.

**What you learn:**

- How a Layer 7 load balancer differs from Layer 4 - concretely, in code
- How TLS termination works and why it must happen before routing
- How to parse application-layer content (HTTP + JSON) to make routing decisions
- How health checking keeps traffic away from dead backends
- How stateful services (mid-conversation bots) create routing problems that stateless
  services do not
- How connection draining allows you to take backends down without dropping live requests
- How shared state makes horizontal scaling possible
- How real traffic behaves differently from simulated traffic - retries, bursts, duplicates

**What you do not learn here (intentionally out of scope):**

- How to use Nginx, Caddy, or any managed proxy - you are building the equivalent
- How Kubernetes ingress controllers work internally - but after this project you will
  understand what they are doing
- How to operate a production system at scale - this is a learning project, not an SRE course

---

## 2. Architecture Overview

```
Telegram Users
     │
     │  (send messages to bot)
     ▼
Telegram Servers  ──── HTTP POST webhook ────►  Load Balancer  (your VM, public IP, port 443)
                                                      │
                              ┌───────────────────────┼───────────────────────┐
                              │                       │                       │
                     reads JSON body          routes by command        forwards to backend
                              │                       │                       │
                    ┌─────────▼──────────────────────────────────────────┐   │
                    │              Routing Table                          │   │
                    │  /weather    ──► Weather Handler Pool               │   │
                    │  /convert    ──► Currency Handler Pool              │   │
                    │  /summarize  ──► Summarize Handler Pool             │   │
                    │  (default)   ──► General Handler Pool               │   │
                    └─────────────────────────────────────────────────────┘   │
                                                                              │
               ┌──────────────────────────────────────────────────────────────┘
               │
     ┌─────────▼──────────────────────────────────────────────┐
     │                  Service Handler Pools                   │
     │                                                          │
     │  Weather Pool      Currency Pool   Summarize Pool  General Pool
     │  [instance A]      [instance A]    [instance A]    [instance A]
     │  [instance B]      [instance B]    [instance B]    [instance B]
     │                                                          │
     └──────────────────────────┬───────────────────────────────┘
                                │  (all pools read/write)
                                ▼
                       Shared State Store
                       (Redis or equivalent)
                       - sessions
                       - rate limit counters
                       - dedup records
                       - metrics
                       - cache

                                         Live Dashboard
                                         (reads state store, served through LB)
```

All components inside the dashed Oracle Cloud boundary run on your free VMs.
Telegram's servers are external and outside your control.

---

## 3. File Structure

```
telegram-bot-platform/
│
├── README.md                          ← this file
├── .env.example                       ← environment variable template (no secrets committed)
├── .gitignore
│
├── docs/                              ← extended documentation
│   ├── architecture.md                ← deep-dive on component design decisions
│   ├── load-balancer-internals.md     ← how your LB works, algorithm by algorithm
│   ├── routing-rules.md               ← routing table format and how rules are evaluated
│   ├── health-checking.md             ← health check design, thresholds, failover logic
│   ├── tls-setup.md                   ← how TLS termination is configured
│   ├── deployment.md                  ← step-by-step Oracle Cloud VM setup guide
│   ├── state-store.md                 ← what lives in the store and why
│   └── observability.md               ← metrics, dashboard, what to watch for
│
├── load-balancer/                     ← the core of the project - hand-built Layer 7 LB
│   ├── main                           ← entry point, starts the LB process
│   ├── config/
│   │   └── config                     ← loads and validates config at startup
│   ├── listener/
│   │   └── listener                   ← accepts incoming TCP connections, hands off to handler
│   ├── tls/
│   │   └── terminator                 ← performs TLS handshake, exposes plain HTTP upstream
│   ├── http/
│   │   ├── parser                     ← reads raw HTTP request into structured form
│   │   └── writer                     ← serialises a response back to the client
│   ├── router/
│   │   ├── router                     ← evaluates routing rules, selects a pool
│   │   ├── rules                      ← rule types: exact match, prefix match, fallback
│   │   └── extractor                  ← navigates JSON body to extract routing signal
│   ├── pool/
│   │   ├── pool                       ← manages a named group of backend instances
│   │   ├── instance                   ← represents one backend instance (address + state)
│   │   └── selector                   ← implements balancing algorithms (round-robin, least-conn)
│   ├── health/
│   │   ├── checker                    ← runs periodic health checks against each instance
│   │   └── state                      ← tracks healthy/unhealthy/draining state per instance
│   ├── proxy/
│   │   ├── forwarder                  ← opens connection to backend, forwards request
│   │   └── headers                    ← rewrites and injects headers (X-Forwarded-For, etc.)
│   ├── middleware/
│   │   ├── ratelimit                  ← per-user rate limiting, reads user ID from JSON body
│   │   ├── dedup                      ← rejects already-processed Telegram message IDs
│   │   └── logger                     ← structured request/response logging
│   └── metrics/
│       └── recorder                   ← writes per-request metrics to the state store
│
├── services/                          ← the backend handler services your LB routes to
│   │
│   ├── shared/                        ← code shared across all services
│   │   ├── telegram/
│   │   │   ├── client                 ← makes outbound calls to Telegram's send-message API
│   │   │   └── types                  ← structs/types matching Telegram's JSON update format
│   │   ├── state/
│   │   │   └── client                 ← reads and writes to the shared state store
│   │   └── config/
│   │       └── config                 ← shared config loader
│   │
│   ├── weather/                       ← handles /weather commands
│   │   ├── main                       ← entry point, starts HTTP server
│   │   ├── handler                    ← receives forwarded webhook, calls weather API, replies
│   │   ├── parser                     ← extracts city name from message text
│   │   └── provider/
│   │       └── openweather            ← talks to OpenWeatherMap (or equivalent free API)
│   │
│   ├── currency/                      ← handles /convert commands
│   │   ├── main
│   │   ├── handler
│   │   ├── parser                     ← extracts amount, source currency, target currency
│   │   ├── session                    ← manages multi-step conversation state
│   │   └── provider/
│   │       └── exchangerate           ← talks to a free exchange rate API
│   │
│   ├── summarize/                     ← handles /summarize commands
│   │   ├── main
│   │   ├── handler
│   │   ├── parser                     ← extracts text or URL from message
│   │   └── provider/
│   │       └── ai                     ← calls an AI summarization API (e.g. free tier)
│   │
│   └── general/                       ← catch-all for unrecognised commands and mid-conv messages
│       ├── main
│       ├── handler
│       └── session                    ← checks state store for active session context
│
├── dashboard/                         ← live web UI for monitoring your LB in real time
│   ├── main                           ← serves the dashboard as a web app (behind the LB)
│   ├── api/
│   │   └── metrics                    ← HTTP endpoints the frontend polls for live data
│   ├── state/
│   │   └── reader                     ← reads metrics and health state from the state store
│   └── ui/
│       ├── index.html                 ← dashboard page
│       ├── styles                     ← styling
│       └── scripts/
│           ├── chart                  ← request rate graph
│           ├── instances              ← per-instance health and request count panels
│           └── feed                  ← live request feed (command → instance → response time)
│
├── state-store/                       ← configuration and schema for the shared state store
│   ├── config                         ← Redis / equivalent config
│   └── schema.md                      ← documents every key pattern, TTL, and data shape
│
├── config/                            ← runtime configuration files (not secrets)
│   ├── lb-config.yaml                 ← routing rules, pool definitions, algorithm choice
│   ├── health-config.yaml             ← check interval, failure thresholds, drain settings
│   └── tls-config.yaml                ← certificate paths, renewal hooks
│
├── scripts/                           ← operational scripts
│   ├── register-webhook               ← calls Telegram API to register your LB's URL as webhook
│   ├── deregister-webhook             ← removes the webhook registration
│   ├── health-check                   ← manually hits the /health endpoint of each instance
│   ├── rolling-restart                ← gracefully restarts service instances one at a time
│   └── generate-certs                 ← runs Let's Encrypt certificate issuance
│
└── tests/
    ├── load-balancer/
    │   ├── router-tests               ← unit tests for routing rule evaluation
    │   ├── parser-tests               ← unit tests for HTTP and JSON parsing
    │   ├── selector-tests             ← unit tests for round-robin and least-conn algorithms
    │   └── health-tests               ← unit tests for health state transitions
    ├── integration/
    │   ├── webhook-flow               ← sends a fake Telegram webhook and asserts correct routing
    │   ├── failover                   ← kills a backend instance, asserts traffic reroutes
    │   └── dedup                      ← sends the same message ID twice, asserts second is dropped
    └── fixtures/
        ├── telegram-message.json      ← sample Telegram webhook payloads for testing
        ├── telegram-callback.json
        └── telegram-inline.json
```

---

## 4. Component Reference

### 4.1 Load Balancer

The load balancer is the heart of the project. It is a single long-running process that
listens on port 443 (HTTPS) and is the only component directly reachable from the public
internet.

**What it is not:**
It is not a framework. It is not a wrapper around an existing proxy. Every part of it -
the TCP listener, the TLS handshake, the HTTP parser, the routing engine, the connection
to the backend - is code you write.

**Responsibilities, in the order they happen per request:**

| Step | Module | What happens |
|------|--------|--------------|
| 1 | `listener` | Accepts the raw TCP connection from Telegram |
| 2 | `tls/terminator` | Performs TLS handshake, decrypts the stream |
| 3 | `http/parser` | Reads the HTTP method, path, headers, and body into memory |
| 4 | `middleware/dedup` | Checks if this Telegram message ID was already processed |
| 5 | `middleware/ratelimit` | Checks if this user has exceeded their request quota |
| 6 | `router/extractor` | Navigates the JSON body to extract the command string |
| 7 | `router/router` | Evaluates routing rules, selects a destination pool |
| 8 | `pool/selector` | Picks a specific healthy instance from that pool |
| 9 | `proxy/headers` | Rewrites and injects headers before forwarding |
| 10 | `proxy/forwarder` | Opens a connection to the backend, forwards the request |
| 11 | `metrics/recorder` | Writes request metadata and timing to the state store |
| 12 | `http/writer` | Relays the backend's response back to Telegram |

**Sub-components:**

`config/` - Reads the lb-config.yaml at startup. Holds the full routing table,
pool definitions (which addresses are in each pool), algorithm selection, timeout
values, and middleware settings. The config is the single source of truth for the
LB's behaviour. Changing routing rules does not require recompiling - only a config
reload or restart.

`listener/` - Opens a TCP socket on port 443 and accepts incoming connections in a
loop. Each accepted connection is handed off to a goroutine (or thread, depending
on your language) so the listener never blocks. The listener itself holds no state.

`tls/terminator` - Performs the TLS handshake using the certificate provisioned by
Let's Encrypt. After a successful handshake, it exposes a plain, decrypted byte
stream to the rest of the pipeline. The backend services never see TLS - they speak
plain HTTP on the internal network. This is called TLS termination at the edge.

`http/parser` - Reads bytes from the decrypted stream and constructs a structured
representation of the HTTP request: method, path, headers map, and body bytes. The
body must be fully buffered before the router can inspect it. This parser is a
learning exercise in its own right - HTTP/1.1 framing has subtle rules around
Content-Length, chunked transfer encoding, and header line endings.

`router/extractor` - Given the buffered body bytes, navigates the JSON structure to
find the value at `update.message.text`. Extracts the first whitespace-delimited
token. If that token begins with `/`, it is a bot command and becomes the routing
key. If the field is absent or the body is not a recognised Telegram update type,
the extractor returns a sentinel value that causes the router to use the fallback rule.

`router/router` - Holds the routing table loaded from config. Evaluates rules in
order. The first matching rule wins. Returns the name of the destination pool.

`pool/pool` - A named collection of backend instances. The pool knows which
instances are currently healthy (via health state) and delegates instance selection
to the selector.

`pool/selector` - Implements the balancing algorithm. Round-robin maintains a
counter and cycles through instances. Least-connections tracks the active request
count per instance and always selects the instance with the lowest count. The
selector is the only component that differs between balancing strategies.

`health/checker` - Runs a background loop for each pool. Periodically sends an
HTTP GET to each instance's `/health` endpoint. Tracks consecutive successes and
failures. After a configurable number of consecutive failures, marks the instance
as unhealthy and removes it from the eligible set. After a configurable number of
consecutive successes, restores it. This is entirely separate from the request
handling path - health checks run on their own schedule.

`proxy/forwarder` - Opens a new TCP connection to the selected backend instance,
writes the forwarded HTTP request (with rewritten headers), waits for the backend
response, and streams it back to the original caller. Handles timeouts: if the
backend does not respond within the configured deadline, the forwarder closes the
connection and returns a 504 to Telegram.

`middleware/dedup` - Before routing, checks whether the incoming Telegram message ID
already exists in the state store. If it does, returns 200 immediately without
forwarding to any backend. If it does not, writes the message ID to the store with
a short TTL (long enough to cover Telegram's retry window) and proceeds. This
prevents double-processing when Telegram retries a delivery.

`middleware/ratelimit` - Reads the user ID from the JSON body. Checks a counter in
the state store for that user. If the counter exceeds the configured threshold for
the current window, returns 429 without forwarding. Otherwise, increments the
counter and proceeds. The counter has a TTL equal to the rate limit window.

---

### 4.2 Service Handlers

Each service handler is an independent HTTP server. It has no knowledge of the load
balancer - it simply listens on an internal port, receives forwarded requests, processes
them, and returns an HTTP response. Multiple instances of each handler can run
simultaneously because all shared state lives in the state store, not inside the process.

**Weather Handler**

Receives webhooks for `/weather` commands. Extracts the city name from the message text.
Calls an external weather API using a free-tier API key. Formats a human-readable response.
Makes an outbound HTTP call to Telegram's `sendMessage` API to deliver the reply to the
user. Returns 200 to the load balancer. Does not store any session state - each request
is fully self-contained.

**Currency Handler**

The most stateful of the handlers because `/convert` requires a multi-step conversation.
The user sends `/convert`, the bot asks for an amount, the user replies with a number, the
bot asks for a target currency, the user replies with a currency code. Each reply from the
user arrives as a separate webhook with no command prefix.

This handler reads and writes session state from the shared state store. When it receives a
message from a user who has an active `/convert` session, it reads the session to understand
what step the conversation is on, processes the reply accordingly, and either advances or
completes the session. The session has a TTL so abandoned conversations expire automatically.

**Summarize Handler**

Receives webhooks for `/summarize` commands. The message text either contains the content to
summarize directly or a URL to fetch and summarize. Calls an external AI API (free tier).
Because AI API calls can be slow, this handler is the most likely to bump against Telegram's
webhook response timeout. The handler acknowledges the webhook immediately with 200, then
processes the request asynchronously and delivers the reply through Telegram's API when ready.
This non-blocking pattern is important to understand because it demonstrates that the webhook
response and the bot reply are independent events.

**General Handler**

The catch-all. Receives any request that did not match a specific command rule. Has two jobs.

First, check whether the sender has an active session with another handler (stored in the
state store). If they do - for example they are mid-conversation with the currency handler
- the general handler delegates to the appropriate logic rather than treating the message
as unknown.

Second, if there is truly no context, respond with a help message listing available commands.

---

### 4.3 Shared State Store

A Redis instance (or equivalent) running on one of the Oracle VMs. Every service handler
and the load balancer itself read from and write to this store. It is the only component
that has global visibility across all processes.

**Key namespaces and what lives in each:**

`session:{user_id}` - The current conversation state for a user who is mid-command.
Contains which command is active, what step the conversation is on, and any partial
data collected so far (e.g. the amount entered for a currency conversion). Has a TTL
of several minutes so abandoned sessions clean themselves up.

`ratelimit:{user_id}:{window}` - The request counter for a specific user within the
current time window. The window is a timestamp truncated to the window size (e.g. the
current minute). Has a TTL equal to the window size. The load balancer increments this
on every request and checks it before forwarding.

`dedup:{message_id}` - A flag indicating this Telegram message ID has been processed.
Has a TTL equal to Telegram's retry window (a few minutes). The load balancer sets this
on first receipt and checks it on subsequent arrivals of the same ID.

`metrics:requests:{timestamp}` - A counter of total requests processed in a given
second or minute. Written by the load balancer. Read by the dashboard.

`metrics:instance:{pool}:{instance}:latency` - A rolling list of response times for a
specific backend instance. Written by the load balancer after each forwarded request.
Read by the dashboard to compute per-instance average latency.

`metrics:instance:{pool}:{instance}:count` - Total requests handled by a specific
instance since startup. Used by the dashboard and also by the least-connections
selector for real-time load awareness.

`cache:weather:{city}:{hour}` - Cached weather API response for a city, keyed by city
name and the current hour. Has a TTL of one hour. The weather handler writes this on
first fetch and reads it on subsequent requests for the same city within the hour.

---

### 4.4 Live Dashboard

A small web application that reads from the state store and presents real-time information
about your platform's behaviour. It runs as its own process on one of the Oracle VMs and
is itself served through the load balancer - meaning you can observe the LB's behaviour
while the LB is simultaneously serving your request to the dashboard.

**What the dashboard shows:**

Requests per second graph - a rolling time-series of incoming webhook volume. You can
watch this spike when you share your bot in a busy group.

Per-instance request distribution - a bar chart showing how many requests each backend
instance has handled. With round-robin, this should be nearly even. With least-connections
and a slow summarize handler, you will see the distribution skew.

Per-instance latency - average response time per instance. Useful for spotting when a
particular instance is slower than its peers.

Health status panel - a grid showing each instance as green (healthy), yellow (degraded),
or red (down). Updates in real time as the health checker marks instances up or down.

Live request feed - a rolling list of the most recent requests showing: which command was
received, which pool was selected, which instance handled it, and how long it took. This
is the most instructive view when you are actively testing routing behaviour.

Failover event log - a timestamped record of every time an instance was marked unhealthy
or restored. Lets you correlate health events with traffic pattern changes in the graphs.

---

## 5. How Traffic Flows

This section traces a single `/weather Nairobi` message from the moment a user sends it
to the moment they receive a reply.

**1.** The user sends `/weather Nairobi` in a Telegram chat with your bot.

**2.** Telegram's servers receive the message and look up the registered webhook URL for
your bot. That URL points to your load balancer's public IP on Oracle Cloud.

**3.** Telegram opens a TCP connection to your LB on port 443 and initiates a TLS handshake.

**4.** Your LB's `listener` accepts the connection. The `tls/terminator` completes the
handshake and exposes a decrypted stream.

**5.** The `http/parser` reads the incoming bytes and buffers the full HTTP POST request.
The body is a JSON object containing the message text, sender details, chat ID, and
a unique message ID.

**6.** The `middleware/dedup` extracts the message ID and checks the state store. This is
the first time this message has arrived, so no dedup record exists. The middleware writes
the message ID to the store with a TTL and proceeds.

**7.** The `middleware/ratelimit` extracts the user ID and checks the rate limit counter
in the state store. The user is under their limit. The middleware increments the counter
and proceeds.

**8.** The `router/extractor` navigates the JSON body to `message.text`, finds the string
`/weather Nairobi`, and extracts the first token: `/weather`.

**9.** The `router/router` evaluates the routing table. The rule `exact:/weather →
weather-pool` matches. The router returns `weather-pool`.

**10.** The `pool/selector` for `weather-pool` applies round-robin and selects instance B
(because instance A handled the previous request).

**11.** The `proxy/headers` module rewrites the request: adds `X-Forwarded-For` with
Telegram's IP, adds `X-Command: /weather`, preserves all original headers.

**12.** The `proxy/forwarder` opens a TCP connection to instance B's internal address
and writes the forwarded HTTP request.

**13.** Instance B's weather handler receives the request. It parses `Nairobi` from the
message text. It checks the state store cache for `cache:weather:nairobi:{current-hour}`.
No cache entry exists. It calls the external weather API.

**14.** The weather API responds. The handler writes the result to the cache with a one-hour
TTL. It formats the reply text. It makes an outbound HTTP call to Telegram's `sendMessage`
API, which delivers the weather report to the user's chat.

**15.** The handler returns HTTP 200 to the load balancer.

**16.** The `proxy/forwarder` receives the 200. The `metrics/recorder` writes the request
duration and instance selection to the state store.

**17.** The LB's `http/writer` relays the 200 back to Telegram on the original TLS connection.

**18.** Telegram's webhook delivery is complete. The user sees the weather report.

Total time: typically 200–800ms depending on weather API latency. The user experiences
only the Telegram-side delay. From Telegram's perspective, your system responded in time.

---

## 6. How Routing Works

The routing table is defined in `config/lb-config.yaml`. It is a list of rules evaluated
in order. The first matching rule wins. If no rule matches, the fallback rule always matches.

**Rule types:**

`exact` — The extracted command must exactly equal the configured string.
Example: `exact: /weather` matches `/weather` but not `/weatherforecast`.

`prefix` — The extracted command must start with the configured string.
Example: `prefix: /convert` matches `/convert`, `/converteur`, `/convert_usd`.
Useful if you want to route a family of related commands to one pool.

`fallback` — Matches everything. There must always be exactly one fallback rule,
and it must be the last rule in the table.

**What happens when the routing signal is absent:**

When the incoming webhook is not a text message - for example it is a button tap
(callback_query) or an inline query - the `extractor` cannot find `message.text`.
It returns a sentinel value. The router treats any sentinel as a non-match for all
typed rules and falls through to the fallback, sending the request to the General
pool. The General handler knows how to inspect callback_query and inline_query fields
and respond appropriately.

**What happens when routing fails entirely:**

If the backend instance selected by the pool is unreachable (connection refused, timeout),
the forwarder does not return an error to Telegram. Instead it marks the instance as
suspect (increments its failure counter for the health checker to evaluate) and retries
with the next available instance in the pool. Only if all instances in the pool are
unreachable does the LB return a 503 to Telegram.

---

## 7. Health Checking

The health checker runs as a background goroutine (or thread) per pool. It is entirely
independent of the request handling path - it runs on a timer, not triggered by requests.

**The check itself:**

Every N seconds (configurable per pool), the health checker sends an HTTP GET to
`{instance-address}/health`. A healthy instance returns HTTP 200 with a JSON body
containing its current status. The checker records the result - success or failure -
in the instance's state object.

**State transitions:**

An instance starts in the `healthy` state. After K consecutive failures (configurable),
it transitions to `unhealthy`. The pool selector never selects an unhealthy instance.
After M consecutive successes (configurable), it transitions back to `healthy`. This
hysteresis (different thresholds for failure and recovery) prevents flapping — an
instance that is borderline does not oscillate rapidly between healthy and unhealthy.

**Draining:**

When you want to take an instance down for maintenance, you set it to `draining` state
(via a script or admin endpoint). A draining instance receives no new requests from the
pool selector, but the forwarder does not immediately close existing connections. It
waits until all in-flight requests to that instance complete before terminating. This
is graceful draining - no request is dropped mid-flight.

**The health endpoint contract:**

Each service handler exposes `/health`. That endpoint returns 200 if the handler
considers itself operational - it can reach the state store, it has capacity to accept
new requests, and its external API dependencies are reachable. It returns 503 if any
of those conditions fail. This makes health checking application-aware rather than
just TCP-aware. A handler whose external weather API is down should report unhealthy
so the LB stops sending it traffic, even though its HTTP server is technically running.

---

## 8. TLS Termination

Your load balancer is the only component with a public IP address and the only component
that handles TLS. This is the standard edge-termination pattern.

**Why TLS must terminate at the LB:**

The JSON body routing depends on reading the HTTP body. The HTTP body is inside the
TLS layer. Therefore TLS must be decrypted before routing can happen. There is no
way to route on application-layer content without first decrypting the transport layer.

**Certificate provisioning:**

Let's Encrypt issues free TLS certificates for your domain. The `scripts/generate-certs`
script runs the ACME challenge to prove you control the domain and writes the certificate
and private key to the paths referenced in `config/tls-config.yaml`. Certificates expire
every 90 days. The script also sets up automatic renewal.

**What the backend services see:**

Because TLS terminates at the LB, backend services communicate over plain HTTP on the
internal Oracle Cloud private network. They never handle certificates. The LB injects
`X-Forwarded-Proto: https` so backends know the original connection was encrypted even
though their leg of the journey is not.

---

## 9. Configuration System

All behaviour is driven by YAML files in `config/`. No routing logic, pool membership,
health check thresholds, or algorithm choice is hardcoded.

`lb-config.yaml` - Defines pools (names, instance addresses), routing rules (in evaluation
order), balancing algorithm per pool, timeout values (backend connect timeout, read timeout,
write timeout), and middleware settings (rate limit window, rate limit threshold, dedup TTL).

`health-config.yaml` - Defines health check interval per pool, failure threshold (how many
consecutive failures before marking unhealthy), recovery threshold (how many consecutive
successes before restoring), drain timeout (how long to wait for in-flight requests before
forcibly closing a draining instance).

`tls-config.yaml` - Defines certificate path, private key path, and minimum TLS version.

Changes to config require a process restart to take effect. A future enhancement could
add a config reload signal so the LB picks up routing changes without dropping connections.

---

## 10. Deployment Layout on Oracle Cloud

Oracle Cloud's Always Free ARM tier provides 4 CPU cores and 24GB RAM total. These are
sliced into separate VMs so components are genuinely isolated - a crash in one VM does
not affect others.

```
VM 1 (1 core, 6GB)   - Load balancer process + TLS cert + state store (Redis)
VM 2 (1 core, 6GB)   - Weather handler (2 instances) + Currency handler (2 instances)
VM 3 (1 core, 6GB)   - Summarize handler (2 instances) + General handler (2 instances)
VM 4 (1 core, 6GB)   - Live dashboard
```

All VMs are connected on Oracle's internal private network. Backend services listen on
internal IPs only - they are not reachable from the public internet. Only the LB's public
IP is exposed. The state store also listens on the internal network only.

This layout means your load balancer is making real network calls to real separate machines
when it forwards requests - not loopback calls on localhost. That distinction matters for
understanding connection latency, connection pooling, and what happens when a VM is
rebooted.

---

## 11. Observability and Metrics

The platform is designed to be watched. The goal is not just to build a load balancer but
to observe it behaving - routing, distributing, health-checking, failing over - with real
traffic.

**What to watch when testing routing:**

Send several `/weather` commands and observe the live feed. You should see the requests
alternating between weather handler instances if round-robin is configured. Switch the
algorithm to least-connections and then send a mix of `/weather` (fast) and `/summarize`
(slow) commands. You should see the weather requests pile onto whichever instance is not
blocked by a slow summarize call.

**What to watch when testing failover:**

Stop one backend instance manually. Watch the health checker detect the failure (after K
consecutive failed checks). Watch the live feed stop showing that instance receiving traffic.
Restart the instance. Watch it recover after M successful checks. No requests should fail
from Telegram's perspective - the LB should have retried on a healthy instance.

**What to watch during a spike:**

Share your bot in a Telegram group and ask people to try it simultaneously. Watch the
requests-per-second graph spike. Watch the per-instance distribution. If one instance
falls behind, observe whether least-connections compensates.

**What to watch during a rolling restart:**

Run `scripts/rolling-restart`. Watch each instance enter the draining state, see zero
new requests assigned to it in the live feed, then come back as healthy once restarted.
The bot should remain responsive throughout — no user should see an error.

---

## 12. Known Edge Cases

**Telegram retries on slow summarize requests.**
The summarize handler calls an external AI API which can be slow. If it exceeds Telegram's
timeout, Telegram retries. The dedup middleware handles this at the LB level — the retry
arrives with the same message ID and is dropped immediately with a 200. The original request
continues processing and delivers the reply when the AI API responds.

**Mid-conversation messages have no routing signal.**
A user in a multi-step `/convert` conversation sends `EUR` with no command prefix. The LB
routes this to the General pool (fallback rule). The General handler checks the state store,
finds an active `/convert` session for this user, and delegates to the currency session logic.
The LB has no awareness of this — it correctly treats the lack of a command as a reason to
use the fallback.

**All instances in a pool are unhealthy.**
If every instance in a pool is marked unhealthy - for example all currency handler instances
are down - the LB returns 503 for any `/convert` request. Telegram receives a non-200 and
retries. Those retries also receive 503 and are logged. Once an instance recovers and passes
health checks, the retries begin routing successfully. Because of dedup, each unique message
ID is processed exactly once when a healthy instance becomes available.

**The state store itself goes down.**
This is the single point of failure in the current architecture. If Redis goes down, the
dedup middleware cannot check for duplicates (safe to proceed - risk of double-processing),
the rate limiter cannot check quotas (safe to proceed - risk of over-allowing), and session
state is unavailable (multi-step conversations break). Service handlers can still process
single-step commands like `/weather` that require no session state. The dashboard goes dark.
Addressing this is a future scaling concern - a replicated state store or a fallback to
in-memory state per instance.

---

## 13. Build Order

Build the platform in this sequence. Each step gives you something working and observable
before adding the next layer of complexity.

**Step 1.** Provision the Oracle Cloud VMs. Confirm they can reach each other on the
internal network. Confirm VM 1 has a public IP reachable from the internet.

**Step 2.** Build a single weather handler instance. Register a Telegram webhook pointing
directly at it (no LB yet). Send `/weather` messages to your bot and confirm end-to-end
flow works - message in, weather reply out.

**Step 3.** Build the load balancer - listener, TLS terminator, HTTP parser, and a
minimal forwarder with no routing logic (just forwards everything to the weather handler).
Change the Telegram webhook to point at the LB. Confirm end-to-end flow still works
through the LB.

**Step 4.** Spin up a second weather handler instance. Implement round-robin selection
in the LB. Add the metrics recorder and connect the LB to the state store. Observe
requests alternating between the two instances in the state store data.

**Step 5.** Implement the health checker. Kill one weather handler instance manually.
Confirm the LB stops routing to it. Restart it. Confirm it recovers.

**Step 6.** Build the currency handler with session state. Add the JSON body extractor
and routing table to the LB. Add the routing rule so `/convert` goes to the currency pool
and everything else goes to the weather pool (temporary). Test that commands route correctly.

**Step 7.** Build the summarize and general handlers. Add their pools and routing rules.
Add the dedup and rate limit middleware. Test all four command types.

**Step 8.** Build the live dashboard. Connect it to the state store. Route `/dashboard`
path requests through the LB to the dashboard service. Watch real metrics while sending
test commands.

**Step 9.** Share your bot. Observe real traffic. Experiment with algorithm settings and
watch the distribution change. Try a rolling restart. Break things on purpose and watch
the system recover.
