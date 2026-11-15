# Cut over Vectis to libknurl 0.2.0

Status: investigated and planned, 2026-11-15. No runtime cutover is implemented.

## Recommendation and scope

Replace the private vendored Kore build and Vectis-owned server execution with
the released libknurl C SDK. Keep Vectis's public application layer, Lua facade,
outer supervisor, services, durable clients, credentials, packed assets and
metrics aggregation. The boundary is libknurl's public operations and callbacks;
the adapter translates declarations and ownership rather than rebuilding HTTP
or proxy state machines.

Make one clean backend cutover after dependency alignment. Do not introduce a
backend-selection feature flag or retain a second server implementation. Keep
the existing server-disabled product configuration supported.

The consumer work can use the C SDK now; it does not depend on Luthrum. The
0.2.0 release ships neither a Lua binding nor the standalone executable. Keep
Vectis's current Lua application API over its C adapter; do not import archived
bindings or describe native tasks as automatic Lua concurrency. See the
[released product boundary](https://github.com/c89systems/libknurl/blob/v0.2.0/docs/runtime-direction.md).

## Baselines and evidence

- Vectis: `1f06712`, branch `init`, initially clean. Its `src/`, `include/`,
  `lua/` and `vendor/` have no delta from the `45aa885` baseline of the previous
  assessment. The earlier mapping remains useful, but its dependency and
  product conclusions do not describe 0.2.0.
- Provider: libknurl **0.2.0**, released commit
  `08cc0d03d38c89483bdd5e5337358f6c9ccba5ea`, shared ABI **0**. Remote release
  metadata and the local tag identify the same candidate. Local source was
  read as evidence only; the probes use published SDK libraries and headers.
- Backend: Kore `c5fe59f521bdc1f2c486b5d8cc90227d72fcc628`, versus Vectis's
  `1ff3c0f6f7973dc7fc8bf38369790ef2d8079b3b`. All **42** Vectis patch digests
  match entries in the released `share/knurl/BACKEND`, which records 70 patches.
  Matching patch bytes establish provenance, not consumer behavior parity.
- GNU SDK: SHA-256
  `d9595c59327769ebfb4fe4bdbc8a46ae03c39f0ce86e07bb3687ed8de7e95575`.
- Musl SDK: SHA-256
  `472b6a5bbf34dbb5965d4d29d035c275b9537ffdbb35197de53b6efbfb888ce2`.
- Release CHECKSUMS: SHA-256
  `1c491879b2a92d70f0773eff7b88d4fdb2bde7d288a93e9f250a707dec829b8f`.

Both downloaded SDKs passed the release checksum manifest. The release inventory
contains all seven existing targets: x86_64/aarch64/armhf Linux with GNU and
musl, plus arm64 Darwin. Only the two x86_64 SDKs were acquired for this
investigation. [Release and assets](https://github.com/c89systems/libknurl/releases/tag/v0.2.0).

## Fresh findings and blockers

| Finding | Evidence and consequence |
| --- | --- |
| Startup preflight is delivered | `knurl_server_preflight` is present in released headers and libraries. Fresh GNU static/shared and native musl static probes reject an occupied listener and a directory as an access-log path. The same server passes preflight after releasing the occupied listener. Retire the previous missing-preflight handoff. |
| The dependency closure must advance together | Released CMake metadata requires cpkt 0.8.0, LoneJSON 0.50.0 and pslog 0.13.0. The GNU DSO has SONAME `libknurl.so.0` and needs `liblonejson.so.29`. |
| CAI is a concrete upstream blocker | The latest published CAI remains 0.6.0. Its selected SDK needs `liblonejson.so.26`. Configuring the released libknurl/provider closure followed by `find_package(cai 0.6.0 EXACT)` fails its installed ABI guard: discovered ABI 29, required ABI 26. A full Vectis cutover needs a compatible CAI release. |
| Raw Kore implementation access is not the new SDK boundary | The released `<knurl/kore.h>` provides an opaque request accessor. No raw `kore_` or `http_` definitions are exported by the inspected GNU DSO. The opaque accessor compiled and executed in the consumer probes. |
| Lua/application integration remains consumer work | libknurl's reference adapter targets Vectis `6b1e263`; it is useful evidence, not the finished Vectis adapter. Archived Lua tests and historical counts are not current shipped Lua support. |

No other required missing server operation was established by this pass.
That is a source/artifact assessment, not proof of the full consumer cutover.

### Dependency alignment

| Component | Current Vectis | Cutover requirement |
| --- | --- | --- |
| cpkt | 0.7.0 | 0.8.0, matching target SDK/digest. Review the OpenSSL allocation-failure printing fix and confirm all used public/runtime seams. |
| LoneJSON native and Lua | 0.46.0 / ABI 26 | 0.50.0 / ABI 29 together. Review every intervening public/Lua change, receiver layout, ownership, streaming, error and embedded-resource seam. Rebuild all users. |
| liblockdc core, PostgreSQL provider and Lua | 0.28.0 | Candidate 0.31.1, whose release selects LoneJSON 0.50.0. Acquire and verify the matching provider/Lua artifacts and examine the full intervening contract delta. |
| CAI native and Lua | 0.6.0 / LoneJSON ABI 26 | Obtain an upstream release supporting the selected LoneJSON ABI 29 and its public layouts. Verify all seven SDKs and matching Lua sources. Do not bypass the ABI check, mix SONAMEs, disable CAI, or use a sibling build as the product input. |
| pslog | 0.13.0 | Retain 0.13.0; the released SDK closure selects it already. |
| Remaining SDKs and Lua providers | Existing pins | Retain unless a verified dependency constraint requires alignment. Audit CMake/pkg-config and actual linked closure; do not turn the cutover into unrelated upgrades. |

The exact selected GNU provider hashes were verified against cached archives
before extracting into this checkout's ignored build directory. Core linked
component versions are curl 8.22.0, libssh2 1.11.1, libxml2 2.15.4, nghttp2
1.70.0, OpenSSL 3.6.5 and zlib 1.3.2. Reconfirm per-target dependency records
when adding production pins.

## Ownership and forwarding plan

| Surface | Vectis retains | libknurl owns / adapter work |
| --- | --- | --- |
| App lifecycle | Declaration policy, quiescence, outer supervisor, app services and VRC1 control semantics | Replace native globals and hooks with inert server construction, public preflight/run/stop/signal operations, process observations and terminal cleanup callbacks. |
| Readiness and control | Application readiness decision, parent maintenance, auth cleanup, metrics/failure frames | Translate the existing control FD through public supervision/loop operations. Parent-ready starts paused; open serving only at the current application boundary. Preserve EOF/malformed-control and parent-death cleanup. |
| Supervisor effects | Existing workflow clients, bounded process-shared receipt channels and durable recovery | Forward the existing worker-to-supervisor key wake unchanged. A library thread waker is not a substitute for this cross-process path. |
| Route declarations | Current C receivers, Lua closures, authentication/business policy and owner rooting | Register ordinary routes, methods, path kinds, groups and hooks on the public router. Preserve registration order, before/handler/after sequencing and error behavior. |
| Requests and responses | Public request/response shells and Lua lifetime/error conventions | Wrap public borrowed views and staged/live replies. Keep native owners through request terminal callbacks; retain repeated headers, raw queries, principal and identity semantics. |
| Upload, DSV and streaming | Lua/DSV callbacks and app authorization | Use public chunk uploads on the owner loop and native-reader uploads for native work. Map partial progress/AGAIN, reset, borrow/take and terminal close without whole-body collection. |
| JSON, XML, REST/OpenAPI | Application schemas, existing Lua conversions and app error policy | Delegate server mapping, parsing/writing and routing to public operations. Keep standalone Vectis-facing receivers as forwarding boundaries where the public provider covers them. Preserve output timing relative to after-hooks. |
| Static, packed and WebDAV | Packed lookup and durable-store providers, credentials, application admission | Register actual host providers; delegate ranges, conditions, file/source execution, WebDAV protocol and covered standalone filesystem operations. Preserve cancellation and interrupted replacement behavior. |
| HTTP proxy, SSE and WebSocket | Declaration policy, app callbacks and log/metric formatting | Replace local curl multi loops, relay state, buffering and framing with library execution. Preserve duplex backpressure, terminal attribution, TLS identities and application WS callbacks. |
| TLS, ACME and logging | Durable state/credential lookup, borrowed root logger and category policy | Translate public file/memory/source material and ACME persistence callbacks. Use shared logging/observations and count each outcome once. Retain callback owners until the library's documented retirement boundary. |

The [released reference contract](https://github.com/c89systems/libknurl/blob/v0.2.0/docs/vectis-integration-contract.md)
and installed `knurl/*.h` are mapping inputs. Production code must compile
against acquired public SDK headers, never the provider's private Kore tree or
the reference test structs.

## Defaults and compatibility decisions

Initialize library configuration, then explicitly translate Vectis's effective
defaults; copying zero fields is not enough. In particular:

- Preserve the existing server header, timeout-disable choices, body policy,
  hard/materialized limits, worker behavior and proxy admission budget. The
  reference adapter disables the new 30-second deferred-response deadline;
  preserve that existing Vectis behavior unless deliberately changed.
- Set `access_log_rotation_disabled` for the current append-only option.
  The library otherwise defaults to 10 MiB/three files. Its regular-file and
  final-symlink rules are stricter; recommend retaining that hardening and
  documenting the intentional difference. Public preflight supplies early
  open-error reporting, while runtime ENOSPC handling remains best effort.
- Audit new defaults and behavior for static compression, ranges, body trailers,
  WebSocket protocol validation, TLS handshake deadlines and diagnostic output.
  Use provider functionality through existing operations rather than carrying
  duplicate implementations. Record intentional externally observable changes.
- Keep `vectis_request_kore()` as an opaque borrowed accessor where useful.
  Retire installation of raw Kore headers and the installed fixture's
  `<kore/kore.h>` include. There is no established in-tree raw transport user;
  a real external commitment would require a public provider operation, not
  copied private structures.
- Preserve supported public receiver layouts where practical. If a required C
  layout change breaks ABI, derive the single required bump from the latest
  released Vectis artifact. Do not bump ABI merely for changing a private
  backend, and do not add pre-1.0 source compatibility shims.

New ACL/MMDB declarations and native task APIs need explicit Vectis application
surface decisions when used. Their C availability does not establish a Lua API
or permit foreign threads to enter a Lua VM. The minimum cutover preserves
existing workflows; adding a scheduler or redesigning supervisor execution is
outside this backend replacement.

## Implementation sequence and exit gates

1. **Close the dependency blocker.** Request the compatible CAI release and
   verify the exact selected artifacts. Align cpkt/LoneJSON/liblockdc/provider/
   Lua pins as a separately verifiable change. Complete the semantic dependency
   review, app regressions, streaming/ownership checks and one-ABI closure before
   using the new server. No mixed-ABI interim product.
2. **Add SDK acquisition and the complete adapter.** Pin libknurl 0.2.0 SDKs for
   all seven targets in the normal checksum/manifest flow. Use independent
   installed prefixes and `find_package(knurl CONFIG REQUIRED)` with imported
   `knurl::static` / `knurl::shared`. Implement the full forwarding surface with
   focused Debug tests before broad builds. Keep process-specific ownership
   explicit in the adapter; do not write transport workarounds for a missing API.
3. **Remove replaced execution in the same cutover.** Retire the
   `vectis_kore_runtime` object target, generated platform/assets/version sources,
   vendor header injection, backend feature defines, hook externs and patched
   source prerequisites. Remove the replaced bridge/proxy/event/curl execution
   and server engines, retaining only genuinely application-owned helpers.
   Delete vendoring/patch management and source-tarball reconstruction of Kore.
   Migrate all call sites and the server-disabled configuration together.
4. **Move tests to public boundaries.** Keep observable Vectis tests and port
   backend-private fixtures to public operations or wire assertions. Tests of
   provider internals belong upstream. Keep deterministic failure ordering and
   native CTest cancellation; no process-table-scanning test supervisor.
5. **Finish product/package closure.** Update exported CMake/pkg-config metadata,
   static/shared provider closure, SDK archive layout, licenses/notices, release
   sources, presets, Make/CI references and clean/extracted-source checks. Inspect
   actual compile/link commands for stale headers and double-linked server code.
   Update manuals, examples, coverage matrix, dependency audit and embedded bytes.
6. **Review and verify the completed candidate.** Finish all surfaces before the
   full matrix. Fix review findings with focused Debug tests and repeat review.
   Require the semantic integration/release review against the final candidate;
   passing provider tests cannot approve the consumer cutover.

The deletion inventory starts with [CMakeLists.txt](../CMakeLists.txt),
[the private bridge](../src/vectis_kore_bridge.c), the `src/vectis_kore_proxy*`
and `src/vectis_proxy*` execution files, the server paths in
[vectis.c](../src/vectis.c), [Makefile](../Makefile),
[source staging](../scripts/stage_release_sources.sh),
[SDK packaging](../cmake/package_archive.cmake) and `vendor/kore/`.
Classify each helper before deleting it; matching a filename is not sufficient.

## Consumer acceptance matrix

| Gate | Required observable evidence |
| --- | --- |
| Lifecycle and supervisor | Direct/supervised/service-only modes, declaration-only behavior, pre-fork quiescence, ready/serving ordering, stop/restart, occupied listeners, bad logs/spools, worker death policy, EOF/parent death and exact terminal cleanup. Direct XA/outbox receipts wake the supervisor without a recovery scan. |
| Routes and streams | Ordering, paths/methods/groups, repeated headers, raw queries, auth/error mapping, after-hook commitment, incremental upload/DSV/XML/output, callback errors, reset/close, allocation failure and cancellation. Withhold final input until incremental progress is observed. |
| Wire/protocol | HEAD/204/205/304, zero-length files, pipelining, partial bodies/trailers, ranges/conditions/encoding, proxy duplex pressure, HTTP/2 upstreams, WS/WSS validation and SSE. No duplicate reply or stale completion after disconnect. |
| Providers and policy | Real packed-asset and durable-store callbacks, interrupted WebDAV replacement/recovery, TLS trust/mTLS, local-CA ACME persistence, structured/file logging and exact metric attribution. |
| Lua product | Existing normal and Linux packed scripts, authoritative upstream modules, source-rock packaging, GC/close/reentry and fork/thread boundaries. No archived provider Lua product is treated as delivered. |
| Native/runtime | GNU/musl default worker stacks and constrained GNU Debug stacks, appropriate memory/race/cleanup checks and real service gates. Prebuilt provider libraries do not establish internal sanitizer coverage. |
| Artifacts | Seven-target build/consumer closure, ARM execution where supported, Darwin cross-build versus native evidence clearly distinguished, server-disabled mode, static CLI, relocated static/shared C89/C++98 and pkg-config consumers, source archives and embedded manual/Lua bytes. |

The older Darwin async-DNS report describes a historical bundle; it does not
establish the selected cpkt 0.8.0 artifact's behavior. Recheck actual packaged
capabilities and native kqueue/TLS/proxy behavior rather than carrying either
its failure or an upstream cross-build success forward as current runtime proof.

## Verification performed for this plan

Scratch inputs and logs are under `build/libknurl-0.2.0-cutover/`, not product
build inputs. No runtime, public headers, pins or upstream repositories changed.

| Check | Result / limit |
| --- | --- |
| Published SDK/checksum acquisition | GNU and musl archives match release CHECKSUMS and GitHub asset digests. Provider archives were separately hash-verified. |
| Released metadata and symbols | Read installed CMake/pkg-config, DEPENDENCIES, BACKEND and GNU ELF metadata; confirmed ABI 29 dependency, ABI 0 SONAME, public preflight/accessor and hidden raw Kore exports. |
| GNU installed consumers | C89 builds against imported static/shared targets pass **2/2** CTest cases for occupied listener, corrected same-server preflight, bad log destination and opaque accessor. Shared execution uses the pinned Bootlin runtime. |
| Musl installed consumers | Static/shared C89 builds pass; native static execution passes **1/1** CTest case. Shared musl runtime execution was not performed. |
| CAI negative configuration | Expected installed-package rejection of LoneJSON ABI 29 versus required ABI 26, after successful libknurl/provider discovery. This is the present dependency blocker. |
| Compile/link readback | Probe commands use the selected 0.2.0 SDK and 0.50.0 provider headers/libraries; no sibling or vendored Kore header is a probe build input. |

Logs: `configure-gnu.log`, `build-gnu.log`, `ctest-gnu.log`,
`configure-musl.log`, `build-musl.log`, `ctest-musl.log` and
`configure-cai-conflict.log`. The initial scratch package search needed an
explicit `knurl_DIR` under the target-only toolchain; that setup correction is
not a provider defect. The first shared probe also needed the normal local
runtime link policy before execution. These probes did not start the complete
Vectis app or certify its normal/packed behavior.

The [0.1.0 assessment](libknurl-migration.md) is retained as historical mapping
and probe evidence. This document supersedes its current-version recommendations,
preflight blocker, ABI targets and assumption of a shipped Knurl Lua package.
