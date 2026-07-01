## 1. Contracts & Schema

- [x] 1.1 Add `ProviderDriverKind.make("omp")` constant to `packages/contracts/src/model.ts`, with `DEFAULT_MODEL_BY_PROVIDER`, `PROVIDER_DISPLAY_NAMES`, `MODEL_SLUG_ALIASES_BY_PROVIDER`, and `DEFAULT_GIT_TEXT_GENERATION_MODEL_BY_PROVIDER` entries
- [x] 1.2 Create `OmpSettings` schema in `packages/contracts/src/settings.ts` using `makeProviderSettingsSchema` with fields: `enabled` (default true), `binaryPath` (default "omp"), `customModels` (hidden, default [])
- [x] 1.3 Add `OmpSettingsPatch` schema and `ServerSettings.providers.omp` entry to `packages/contracts/src/settings.ts`
- [x] 1.4 Export `OmpSettings` type from the contracts package index

## 2. ACP Support Layer

- [x] 2.1 Create `apps/server/src/provider/acp/OmpAcpSupport.ts` with `buildOmpAcpSpawnInput` (command: binaryPath || "omp", args: ["acp"])
- [x] 2.2 Add `resolveOmpAuthMethodId` (constant `"agent"`) to `OmpAcpSupport.ts`
- [x] 2.3 Add `makeOmpAcpRuntime` to `OmpAcpSupport.ts` — wraps `AcpSessionRuntime.layer` with spawn input and auth method, no extension wrapper
- [x] 2.4 Add `resolveOmpAcpBaseModelId` (fallback `"umans/umans-glm-5.2"`) and `currentOmpModelIdFromSessionSetup` to `OmpAcpSupport.ts`
- [x] 2.5 Add `applyOmpAcpModelSelection` helper to `OmpAcpSupport.ts` (mirrors Grok's apply helper for config option selection)

## 3. Provider Adapter

- [x] 3.1 Create `apps/server/src/provider/Layers/OmpAdapter.ts` with `makeOmpAdapter(ompSettings, options)` — copy Grok adapter structure, strip xAI extension wiring (no prompt-completion fallback, no askUserQuestion request handling)
- [x] 3.2 Implement session context management (session map, active turn tracking, pending approvals, pending user inputs)
- [x] 3.3 Wire ACP `session/request_permission` → provider approval UI via `mapAcpToAdapterError` and `acpPermissionOutcome` from `AcpAdapterSupport.ts`
- [x] 3.4 Wire ACP `session/update` notifications → `ProviderRuntimeEvent` stream using `AcpRuntimeModel.parseSessionUpdateEvent`
- [x] 3.5 Implement `session/prompt` dispatch with model/thinking/mode config options
- [x] 3.6 Implement session new/load/resume/fork/close lifecycle methods
- [x] 3.7 Verify `available_commands_update` session update type is silently ignored (no parse warning) — add fallthrough if needed

## 4. Provider Snapshot & Status

- [x] 4.1 Create `apps/server/src/provider/Layers/OmpProvider.ts` with `buildInitialOmpProviderSnapshot`, `checkOmpProviderStatus`, and `enrichOmpSnapshot`
- [x] 4.2 `checkOmpProviderStatus` — check binary existence via `resolveSpawnCommand`, no auth probing
- [x] 4.3 `buildInitialOmpProviderSnapshot` — return available/unavailable snapshot based on binary check
- [x] 4.4 `enrichOmpSnapshot` — no-op or minimal enrichment (no update-check endpoint for omp)

## 5. Driver Registration

- [x] 5.1 Create `apps/server/src/provider/Drivers/OmpDriver.ts` — implement `ProviderDriver<OmpSettings, OmpDriverEnv>` with driverKind, metadata, configSchema, defaultConfig, and create() wiring adapter + textGen + snapshot
- [x] 5.2 Create `apps/server/src/provider/Services/OmpAdapter.ts` — thin registration (1-liner, mirrors GrokAdapter service file)
- [x] 5.3 Add `OmpDriver` to `BUILT_IN_DRIVERS` array and `OmpDriverEnv` to `BuiltInDriversEnv` union in `builtInDrivers.ts`

## 6. Text Generation

- [x] 6.1 Create `apps/server/src/textGeneration/OmpTextGeneration.ts` with `makeOmpTextGeneration(ompSettings, environment)` — copy Grok text generation structure
- [x] 6.2 Implement `runOmpJson` helper — spawn one-shot omp ACP session, send prompt, parse JSON output
- [x] 6.3 Implement `generateCommitMessage`, `generatePrContent`, `generateBranchName`, `generateThreadTitle` using the shared prompt builders from `TextGenerationPrompts.ts`

## 7. Web UI Registration

- [x] 7.1 Add omp entry to `PROVIDER_OPTIONS` in `apps/web/src/session-logic.ts` (label: "Oh My Pi", available: true, pickerSidebarBadge: "new")
- [x] 7.2 Add omp icon mapping to `PROVIDER_ICON_BY_PROVIDER` in `apps/web/src/components/chat/providerIconUtils.ts`
- [x] 7.3 Create or import an omp icon component for the provider icon

## 8. Documentation

- [x] 8.1 Create `docs/providers/omp.md` — document single-account setup, custom binary path, and how omp's local credentials work

## 9. Testing

- [ ] 9.1 Write `OmpAcpSupport.test.ts` — spawn input construction, auth method resolution, model ID resolution
- [ ] 9.2 Write `OmpAdapter.test.ts` — session lifecycle, prompt dispatch, permission handling, event stream mapping
- [ ] 9.3 Write `OmpProvider.test.ts` — snapshot building, status checking with binary present/absent
- [ ] 9.4 Write `OmpTextGeneration.test.ts` — commit message, PR content, branch name, thread title generation
- [ ] 9.5 Write `OmpDriver.test.ts` — driver create/snapshot wiring, config decoding, default config
- [x] 9.6 Run `vp check` and `vp run typecheck` — must pass with zero errors
- [x] 9.7 Run full prompt cycle through `omp acp` to confirm no private notifications appear during prompt execution (validates the no-extension-layer assumption from design D1)

## 10. Validation

- [x] 10.1 Run `vp check` and `vp run typecheck` after all implementation is complete
- [ ] 10.2 Verify omp provider appears in the model picker sidebar when omp binary is available
- [ ] 10.3 Verify a new omp session can be created and a prompt can be sent
- [ ] 10.4 Verify session resume works for an existing omp thread
