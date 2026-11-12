## Published branch status — 12 November 2026

This `1.45.3` branch targets Beat Saber **1.45.3_30054**. PC and Quest have
been tested together on the updated LAN UnifiedServer: both join a lobby,
start a Standard custom song, and show the failed screen. The user also tested
Quest MultiplayerExtensions. Full-song completion, late joins, reconnects and
all per-player options are not individually verified.

Runtime fixes include PC menu activation, Quest network configuration setup,
Quest platform identity injection, nonvirtual beatmap-view base calls, dedicated
server state flags, and a results window when all players fail. Validation passed:
19 dedicated-server tests, 10 HTTP tests, 7 PC tests, PC hook/signature audits,
Quest ARM64 builds, QMOD checks, and installed-library hash/load checks.

The PC BSML, SiraUtil and SongCore dependency ports are intentionally **not
published** with these branches. Reproducing the PC test requires compatible
versions of those dependencies. Quest's core pack is likewise separate; use the
versions declared by each QMOD/QPM manifest. Public BeatTogether service and
external credential verification were not validated: the LAN server disabled
authentication for the test. No general production-readiness claim is made.

The notes below preserve the port investigation/build history. Statements there
about pending tests or uncommitted changes describe earlier stages; the verified
scope above is current. The combined repository/build guide is available at
https://github.com/FrozenAlex/BeatTogether.UnifiedServer/blob/1.45.3/docs/crossplay-1.45.3.md.

---

# BeatTogether 1.45.3: PC and UnifiedServer port

Updated 12 November 2026. These are **experimental local builds**, not a multiplayer release. Source/build work is complete for this pass. The PC Plugins folder is restored to its initial empty state after correcting build-time copying; no public server was deployed, and nothing was committed or pushed.

## Target and bases

The supplied PC installation (`D:/SteamLibrary/steamapps/common/Beat Saber`) reports **1.45.3_30054** in both BeatSaberVersion.txt and globalgamemanagers. Its `CompatibilityVersion.Get()` returns **A31AF687**, matching the previously inspected Quest 1.45.3 client and the updated dedicated server.

| Component | Base | Local result |
| --- | --- | --- |
| MultiplayerCore PC | `dev-1.42.0`, `19fe842be980579957d6dc5b32114726ff49bced` | 1.6.8 DLL and BeatMods-layout ZIP |
| BeatTogether PC | `master`, `82fd4a48370ccd33e66ab5280e4c7a84447a38c0` | 2.2.6 DLL and BeatMods-layout ZIP |
| UnifiedServer | `master`, `534e4b50682538cb326a6287a5cda2cb1aadc6b8` | .NET 6 publish directory and ZIP |
| DedicatedServer inside UnifiedServer | `dev-1.42.0`, `754a67c86debb617748a3a228552092742e319eb` | Same uncommitted 1.45.3 source changes as the sibling dedicated-server port |

All three new checkouts use local branch `port/1.45.3`. UnifiedServer keeps the existing MasterServer and Status submodule revisions. Its DedicatedServer submodule was advanced to the requested base and received the actual ported files; changing only the gitlink would omit the uncommitted implementation.

## PC changes

- Rebuilt both projects against the supplied 1.45.3 managed assemblies using .NET Standard 2.1. Game and mod reference folders are separate build properties. Builds default to **not copying into the game**. BeatTogether can reference the freshly built core through `MultiplayerCorePath`.
- Ported beatmap keys to gameplay type plus characteristic enums, including previews, custom-level packets, difficulty selectors, requirement lookup and spectator transitions. Unknown serialized characteristics are rejected rather than silently replaced.
- Implemented the new packet-processed callback on the custom serializer. Custom packets still use message ID **100** and their existing type-name framing; no new custom packet schema was introduced.
- Replaced the removed `GetAuthenticationToken` path. Custom matchmaking obtains the raw platform ID and platform proof/ticket, adds `auth_user_id` and `single_use_auth_token` to the new game request payload, and preserves its other fields. Steam's SDK returns a hex web-API ticket. The official-server path delegates to the original game provider and its x-platform token exchange. No dummy-token fallback remains.
- Custom matchmaking has a bounded request lifetime, cancellable waits, and bounded cancellation cleanup. Custom HTTP requests disable cookies. Removed the old global GraphAPI async-state-machine transpiler and brittle packet-error transpiler.
- Ported `graphApiBaseUrl` and `useLocalServer` configuration. Custom status requests use the selected status URL; the game handles official status requests. Removed the obsolete status deserialization transpiler.
- Corrected static-call opcodes and binder types in dependency-injection patches. The MainSystem patch requires exactly one entitlement binding and one session-provider binding. Intro/outro replacements now return the game's `IBeatSaberConnectedPlayer` list type.
- Updated even-player placement while preserving the layout method's branches and stack. Restored local-player inclusion in the intro list.
- Restored the dev branch's disabled player-metadata sending and added event cleanup. Spectator scene transitions run on Unity's main-thread scheduler. BeatTogether removes its status listener and floating UI on disposal.

## UnifiedServer changes

- Integrates the dedicated-server compatibility string, controller/active-hand packets, identity order, announcement and late-join fixes described in the dedicated-server handoff.
- Advertises minimum version **1.45.3**, with an exact 1.45.3 matchmaking range. It no longer advertises the previous open-ended 1.40+ range.
- Rejects incompatible HTTP matchmaking requests before authentication or session allocation, including requests carrying an existing ticket: 51 for older versions, 52 for newer versions, 50 for malformed/missing versions.
- Serves status at both `/status` and the game's newer `/beat_saber_multiplayer_status` route.
- Removes the development config's incorrectly nested, obsolete version ranges. Development HTTP binding is loopback.
- Keeps the existing platform-authentication service. HTTP tests use synthetic credentials with authentication disabled; they do **not** prove external ticket verification.

The dedicated server still uses compatibility **A31AF687** for its actual transport gate. Protocol number 9 alone is not sufficient to identify this packet layout. The detailed packet field-order findings are in the earlier `BeatTogether-1.40.8-to-1.45.3-findings.md` report.

## Validation

- Both PC Release builds pass and ZIP payloads match the built DLLs; embedded manifests target 1.45.3.
- **6 PC tests pass**: credential JSON preservation/escaping and rejection of empty credentials, metadata checks for **62 Harmony/Affinity targets**, and checks for binder/intro/outro replacement types, the layout increment pattern and the actual game compatibility marker.
- **10 UnifiedServer HTTP tests pass** using ASP.NET TestServer: both status routes, older/newer/malformed versions and resumed-ticket rejection, plus Steam (platform 3) and Quest (platform 2) credential payloads reaching matchmaking. Dedicated instance allocation is mocked in these HTTP tests.
- **15 dedicated-server packet/session/state tests pass** against the nested source used by UnifiedServer. Wire fixtures are hand-authored from serializer analysis, not captured network traffic.
- The published server started successfully on `127.0.0.1:18989`; both status routes returned 1.45.3 and a 1.42.0 matchmaking request returned error 51. That test process was stopped and the packaged configuration restored.
- Publish output includes Windows `enet.dll` and Linux `libenet.so`. Native lobby creation and ENet exchange were not exercised by the HTTP smoke test.
- Build logs still contain existing nullable warnings and a System.Net.Http reference-version warning; they are not zero-warning builds.

## Lobby-state follow-up

The server state-mask writer now retains the result of `SetBits`; previously `dedicated_server` and `terminating` writes were discarded. The new regression test failed before the fix. Both DedicatedServer checkouts were updated, 15 dedicated and 10 HTTP tests pass, and the rebuilt local server responds to status requests. A separate loopback PC test server is running; the public service has not been changed. See the dependency handoff for the test configuration and pending in-game validation.

## Dependency and runtime limits

The PC dependencies have now been ported and rebuilt: **BSML 1.14.3+dev, SiraUtil 3.4.2+dev and SongCore 3.16.1**. Both PC mods were rebuilt against them, with updated manifest minimums. BSIPA 4.3.7 is supported using a restored, version-gated SongCore config-watcher workaround. The new crossplay bundle contains these dependencies and the game-independent BeatSaverSharp/compression libraries with their manifests. See [dependency implementation, validation and first-test instructions](multiplayer-pc-dependencies-1.45.3.md). Runtime compatibility remains unverified; plugin-defined custom characteristics remain a known port gap.

Also still unverified:

1. Actual Harmony/Affinity application, Zenject initialization and UI behavior inside Unity (metadata validation does not execute patches).
2. Steam ticket acceptance by the deployed BeatSaver/BeatTogether authentication service. The game's Steam ticket is requested for `oculus-xplat-backend`; acceptance by the legacy verifier must be checked with a real account. Quest proof verification also needs a live test. No real credentials were read or used here.
3. Create/join/rejoin, custom-song download, per-player difficulty/modifiers, spectator transitions, results, and switching back to official servers.
4. PC/Quest crossplay, active-hand updates, late-join avatars and 2/4/6+ player layouts through real ENet traffic.

The server's existing `AuthenticateClients: false` default remains unchanged. This port does not constitute an authentication-service review or a deployment.

## Rebuilding

PC, from the MultiplayerCore.PC checkout (requires .NET 9 SDK for tests):

```powershell
./build-port.ps1 -BeatSaberDir 'D:/SteamLibrary/steamapps/common/Beat Saber' -ModRefsDir 'PATH_TO_MOD_REFERENCE_ROOT' -BeatTogetherRepo '../BeatTogether.PC'
```

`ModRefsDir` contains `Plugins/SongCore.dll`, `Plugins/SiraUtil.dll` and `Plugins/BSML.dll`; it defaults to the game folder. Other references come from the game's Managed and Libs folders. The script builds core, builds BeatTogether against that exact DLL, packages both, and runs the PC tests. It does not install them.

UnifiedServer, from its checkout with the ported submodule files present:

```powershell
dotnet test BeatTogether.UnifiedServer.Tests -c Release
dotnet test dependencies/BeatTogether.DedicatedServer/BeatTogether.DedicatedServer.Tests -c Release
dotnet publish BeatTogether.UnifiedServer -c Release --self-contained false -o artifacts/1.45.3/unified-server
```

The framework-dependent server requires the .NET 6 / ASP.NET Core 6 runtime. Run `dotnet BeatTogether.UnifiedServer.dll` from the publish directory after configuring `appsettings.json` for the intended host. The supplied default is not a public deployment configuration prepared for your network.

## Handoff contents

The share bundle contains the two PC mod ZIPs, the UnifiedServer publish ZIP, this report, the previous packet findings, validation hashes, and source patches. It contains no proprietary PC game assemblies or old dependency DLLs.

For source patches: clone each named repository at the base SHA above. For UnifiedServer, initialize its pinned submodules first. Apply its root patch, then check out DedicatedServer at `754a67c86debb617748a3a228552092742e319eb` and apply `DedicatedServer.patch` inside that submodule. Apply the other patches in their corresponding PC repositories. `git apply` is sufficient; no commit is required. Every patch was checked in reverse against the final local source. Preserve the submodule's source changes when sharing or committing the port.
