# Mihomo Party → Hiddify Fork: Feature & UI Parity Plan

Goal: build one compiled client (Mac, Windows, iOS) for personal + small friend-group use,
reusing the existing subscription (SS-1, SS-2, V2Ray US-LA/JP/Canada, rate-limited node) and
reproducing Mihomo Party's functionality and UI, not just its routing rules.

## What Mihomo Party gives us today

- Multiple **named proxy groups**, each independently selectable from a curated subset of
  subscription nodes: default/GEOIP group, JP group, Telegram group, WeChat group.
- Per-group node selection persists independently — e.g. WeChat can pin SS-1 while the main
  group runs URL-test across V2Ray nodes.
- Rule engine bypassing TUN/system proxy entirely for: school intranet (`ykpaoschool.cn`,
  `10.x.x.x`), specific game servers (Arknights CN, Minecraft/Modrinth), LAN.
- JS override scripts layered on top of the base subscription (renaming nodes, injecting
  groups, tweaking rule providers) without editing the subscription itself.
- A single-window group-tabs UI: switch between groups via tabs/dropdown, see delay per node,
  tap to select.

## What Hiddify has today (as of this fork point, commit 276a7eff)

Findings from reading the source directly:

- **Core is capable of multi-group**: `ProxyRepository.watchActiveProxies()` returns
  `List<OutboundGroup>` and `selectProxy(groupTag, outboundTag)` already takes an explicit
  group tag — the sing-box core and repository layer support multiple named selector groups.
- **UI is currently single-group only**: `ProxiesOverviewNotifier.build()` calls
  `watchProxies()`, which was refactored down to `Stream<OutboundGroup?>` (singular). The old
  multi-group sorting logic is present but commented out in
  `lib/features/proxy/overview/proxies_overview_notifier.dart`. `ProxiesOverviewPage` renders
  exactly one `GridView` for one group with no tab/group switcher.
  → **This is the main gap.** The plumbing exists; the page just needs to iterate groups.
- **Route rules module** (`lib/features/route_rules/`) is a flat rule list per profile, each
  rule has one `Outbound` target from a closed enum: `proxy | direct | direct_with_fragment |
  block`. There is no per-rule "which named group" concept — a rule can only send traffic to
  proxy/direct/block, not to "the WeChat group" specifically.
  → To reproduce Mihomo Party's per-app proxy-group routing (e.g. "WeChat domains use the
  WeChat group, not whatever the main group is set to"), we need either:
    (a) extend the generated config schema so `Rule.outbound` can reference a named outbound
        group tag (requires touching the protobuf schema + hiddify-core, the Go backend), or
    (b) emulate it at the sing-box config-generation layer by writing a custom JSON template
        with multiple `selector` outbounds and per-domain routing rules that target them by
        tag — bypassing Hiddify's rule builder UI for this part and injecting a hand-written
        sing-box rule block, similar in spirit to a Mihomo Party override script.
  Option (b) is much lower risk and matches how Mihomo Party overrides already work
  conceptually (JS override → generated config), so it's the recommended starting point.
- **Profile parser** (`lib/features/profile/data/profile_parser.dart`) already handles `ss://`,
  `ssconf://`, and other standard subscription link formats, and the project README lists
  Clash / Clash Meta / V2Ray / sing-box subscription formats as supported. The existing
  6-node subscription URL should import with no server-side changes.
- **No JS override system.** Mihomo Party's override scripts are a Clash-Meta-client feature
  with no Hiddify equivalent. Closest existing surface: `lib/features/profile/details/
  json_editor.dart`, which lets a profile's generated config be hand-edited/inspected. We'll
  build a lightweight override layer on top of this (see Phase 3).
- **Per-app proxy** exists (`lib/features/per_app_proxy/`) — Android/package-based only per a
  first read; needs a follow-up check for whether it's meaningful on desktop/iOS, since
  Mihomo Party's per-app-adjacent use case here is really domain-based (WeChat/QQ/game
  bypass), not app-based, which route rules already cover on desktop.

## Phase plan

1. **Baseline build** — get the unmodified fork building and running on macOS first (fastest
   inner loop), import the real subscription, confirm nodes show up and basic proxying works.
2. **Multi-group proxy UI** — restore `List<OutboundGroup>` end-to-end:
   - `ProxyRepository.watchProxies()` → emit all groups again.
   - `ProxiesOverviewNotifier` → drop the singular collapse, keep a `List<OutboundGroup>`.
   - `ProxiesOverviewPage` → add a group switcher (tabs or segmented control) above the grid,
     one grid per selected group tab. This is the direct Mihomo Party group-tabs equivalent.
   - Wire `changeProxy` to pass the active tab's group tag.
3. **Named groups from the subscription** — define our own selector groups at config-generation
   time (default/GEOIP, JP, Telegram, WeChat), each pre-populated with the node subset we
   already use in Mihomo Party. Implemented as a post-processing step on the parsed profile
   (outbound-group injection), not a UI feature users configure by hand — mirrors how the
   Mihomo Party override script pre-wires groups today.
4. **Rule parity via custom sing-box rule injection** — port the actual bypass rules (school
   intranet domain + `10.x.x.x`, Arknights CN, Minecraft/Modrinth, WeChat/QQ domain sets) as a
   hand-written rule block merged into the generated config at the same injection point as
   step 3, targeting our named groups instead of the generic `proxy` outbound.
5. **UI pass for visual parity** — theme, group tab styling, delay-color coding, and layout
   tweaks so the app reads like a natural evolution of Mihomo Party rather than stock Hiddify.
   Low priority relative to 2–4; cosmetic only.
6. **Cross-platform builds** — macOS and Windows first (unrestricted distribution), iOS last
   (needs paid Apple Developer account + Network Extension entitlement + TestFlight, per prior
   discussion in this project).

## Immediate next step

Get a macOS debug build running with the real subscription imported, before writing any
group/rule injection code, so every later change has a working baseline to diff against.
