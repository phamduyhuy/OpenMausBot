# ACP capability architecture investigation

Status: source investigation and isolated end-to-end verification, 2026-11-15. The live service and user data were not changed during this investigation. Source checkout: `8471a36df90ba2f4eef26015ce8f1acef02c0d8e`; this pins the observations below even if upstream main changes.

## Conclusion

The upstream design is a stable launch bearer with a new server-side authority record for each turn. Keeping a warm ACP process is expected. The claim that ACP resume inherently skips capability minting is **not established by source**: both direct and room dispatch build integrations before dispatch, and `agentsServerEnv` synchronously calls the mint callback. Correct diagnosis must locate where that expected refresh fails, which token the running proxy actually holds, or which event revokes the new record.

Automatically reviving arbitrary retained tokens is not equivalent to repairing that refresh. In particular, the cached grants fingerprint intentionally excludes turn-specific authority, so it cannot reconstruct a new turn's depth, handoff, or skill-authoring permission.

## Verified architecture

| Layer | Responsibility | Evidence |
| --- | --- | --- |
| Conversation orchestration | Starts a generation, builds integrations for this dispatch, sends a provider turn, binds its ID to the generation | [`server/index.ts`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/server/index.ts): `startTurn`, `beginInternalCapabilityGeneration`, `agentsIntegration`, `bindInternalCapabilityToProviderTurn` |
| Agents integration | Separates stable conversation/tool-list facts from current turn authority; mints bearer while building env | [`server/agents-turn.ts`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/server/agents-turn.ts): `AgentsConversation`, `AgentsTurn`, `agentsServerEnv` |
| Provider-to-MCP wiring | Converts integrations to ACP stdio MCP servers or owned HTTP MCP servers | [`server/drivers/turn-mcp.ts`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/server/drivers/turn-mcp.ts): `turnMcpServers`, `ownMcpOverHttp`, `acpMcpServers` |
| Shared ACP driver | Reuses processes by launch contract; reuses sessions when MCP selection contract matches; loads/creates sessions otherwise | [`server/drivers/acp/core.ts`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/server/drivers/acp/core.ts): `sendTurn`, `contractKey`, `sessionKey`, `session/load`, `session/new` |
| Agents proxy | Reads `OMB_COMMS_TOKEN` from launch env and sends tools through harness | [`server/drivers/agents-proxy.ts`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/server/drivers/agents-proxy.ts), `agents-client.ts` |
| Terminal event correlation | Prevents old/early terminal events from granting or revoking the wrong turn | [`server/turn-dispatch-guard.ts`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/server/turn-dispatch-guard.ts): `ProviderTurnGenerationRegistry` |
| Built-in Devin | Configures the shared ACP driver with `devin acp`, account/model discovery and permission-mode confirmation | [`server/drivers/acp/devin.ts`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/server/drivers/acp/devin.ts) |
| Build | Bundles server entry to `dist-server/index.js`; intermediate tsc output can coexist | [`scripts/bundle-server.mjs`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/scripts/bundle-server.mjs) |

## Authority lifecycle and invariants

All following lifecycle functions are in `server/index.ts`.

1. `beginInternalCapabilityGeneration` revokes the previous generation and installs the new one.
2. `sessionBearer` keys credentials by bot, thread and integration kind (plus connector audience). Matching `sessionGrants` reuses a token. Changed grants rotate it.
3. `sessionGrants` removes `generation`, `depth`, `roomHandoffId`, and `skillAuthoring` from the stable fingerprint and adds current approval mode. These omitted fields must come from the new dispatch, not old fingerprint JSON.
4. `mintInternalCapability` refuses inactive generations, builds a current record, and preserves counters when minted twice in the same generation.
5. `authorizedInternalCapability` accepts only a registered active record, using constant-time comparison. `internalCapabilityIsActive` also enforces generation, expiry, selected-computer/lease ownership.
6. `revokeInternalCapabilityGeneration` deletes turn records and provider ownership. `revokeEarlierTurnCapabilities` retains stable launch credentials; `revokeInternalCapabilitiesForThread` removes credentials as well. Provider reload separately removes credentials for changed engines.

Consequently a test must prove both success during the correct next turn and refusal outside a turn, after stop, after scope/approval rotation, and for stale events. A health endpoint or standalone mint test does not prove this invariant across ACP.

## Why changing completed-turn handling needs a regression case

`ProviderTurnGenerationRegistry.bind` intentionally refuses an already-completed ID. `complete` can run before `sendTurn` returns its ACK; reviving that ID would grant authority after completion. Existing tests explicitly cover this race, stopped-generation late completion, wrong-thread completion, and exact early-result consumption in [`server/turn-dispatch-guard.test.ts`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/server/turn-dispatch-guard.test.ts).

The shared ACP driver ordinarily creates `turnId = retryOf ?? newId()`; its IDs are harness-generated rather than ACP session IDs. Reuse is therefore not a generic property of normal ACP resume. A retry-specific collision, if suspected, must be reproduced without weakening early-completion protection.

## Devin causality: not established

The upstream built-in integration is [PR #5880](https://github.com/milind-soni/OpenMausBot/pull/5880). Its provider-specific behavior plugs into the shared ACP core. This alone does not prove that the user's earlier Devin enhancements match upstream, are present in the installed bundle, or caused capability failures. Compare installed-version source, locally altered files and the user's Devin patch history before attributing causality.

## Next diagnostic checks

- Pin the installed package version to its source commit and compare its ACP core, integration construction, and event correlation with the checkout.
- In an isolated fixture, run two turns in one ACP session with a tools call in each. Record redacted token fingerprints, generation transitions, mint/revoke cause and provider event IDs.
- Cover approval/scope changes, room handoffs, stopped turn, early completion before provider ACK and a late event from an older generation.
- Verify through the repository's isolated fixture protocol in [`docs/verification/README.md`](https://github.com/milind-soni/OpenMausBot/blob/8471a36df90ba2f4eef26015ce8f1acef02c0d8e/docs/verification/README.md); do not mutate the user's live data to test.


## Installed package and local Devin adapter

The installed npm package is `openmausbot@0.1.155`, source tag `v0.1.155` (`8b17baf6fabfeb52615bc5b78127e04e797934c0`). A downloaded npm tarball's `dist-server/index.js` exactly matches the supplied `/tmp/idx-cap.js` snapshot. Installed split ACP `core.js` and `custom.js` also match the published package. These comparisons establish provenance of those files only; they do not establish that every installed file is unmodified.

The local Devin instance uses `customAcp` with `~/.local/bin/devin-acp`. The inspected Python adapter forwards incoming protocol messages unchanged to `devin acp`, implements `--version`, and trims advertised model/config-option lists above 128 entries. No capability-token rewrite was found in this wrapper. This does not establish the behavior of Devin's own MCP caching or identify the earlier enhancement task's history. Upstream's built-in Devin integration is newer than the installed source tag and is absent from the original installed bundle.

The supplied current snapshot contains session bearer slot/retired-token changes and a lazy fallback in authorization. It does not contain the advertised begin-generation remint or completed-ID-unretire changes. Patch-script intentions must not be treated as installed behavior.

## Isolated verification results

Runtime: Node 24 installed in a temporary test toolchain. Fixtures use temporary homes/data directories and free ports; no real Devin account or live service mutation is involved.

| Target | Result |
| --- | --- |
| Checkout `8471a36df`: existing session-credentials, Devin driver and dispatch-guard suites | 29 passed, 1 platform skip |
| New custom ACP credential test on checkout `8471a36df` | Pass |
| Same new test against `v0.1.155` source | Pass |
| Final combined lifecycle, Devin and dispatch-guard suites including the new test | 30 passed, 1 platform skip |
| Same test restarting its isolated fixture with the installed hotfixed bundle | Fail: after approval rotation, old token returned **200**, expected **401** |

The added test proves that two prompts use one ACP process and session; the stable token is refused between turns, succeeds in the next active turn, and must be refused after approval changes and Stop. The installed hotfix demonstrably breaks the approval-rotation boundary. This is a confirmed regression introduced by retaining/reactivating retired tokens, not proof of the original 401 cause.

Run the source checks:

```sh
pnpm exec vitest run server/turn-dispatch-guard.test.ts server/drivers/acp/devin.test.ts server/session-credentials.e2e.test.ts
```

To check a built bundle, the test-only entry override restarts only the isolated fixture with that entry (the bundle must retain its adjacent runtime assets):

```sh
OMB_CREDENTIALS_TEST_SERVER_ENTRY=/absolute/path/to/dist-server/index.js \
  pnpm exec vitest run server/session-credentials.e2e.test.ts -t 'remints a custom ACP'
```

The original resumed-turn 401 remains unreproduced. A source fix for that issue requires a captured failing tool call and its server-side generation/mint/revoke sequence, or a deterministic reproduction involving the actual Devin MCP session behavior. Neither a healthy process nor a successful synthetic ACP test is enough to close it. The earlier enhancement task/PR is still needed to attribute changes to that task rather than to unrelated upstream or local work.
