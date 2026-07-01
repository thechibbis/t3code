## ADDED Requirements

### Requirement: omp provider driver registration

The system SHALL register an `omp` provider driver kind that wraps the `omp acp` stdio server using the existing ACP session runtime infrastructure.

#### Scenario: omp binary available

- **WHEN** the `omp` binary is found on `PATH` (or a custom `binaryPath` is configured)
- **THEN** the system SHALL report the omp provider as available in the provider snapshot
- **AND** the provider SHALL appear in the model picker sidebar

#### Scenario: omp binary not installed

- **WHEN** the `omp` binary is not found on `PATH`
- **THEN** the system SHALL report the omp provider as unavailable
- **AND** the provider SHALL NOT appear as a selectable option for new threads

#### Scenario: omp binary configured via custom path

- **WHEN** a user sets `binaryPath` to a non-default value in omp provider settings
- **THEN** the system SHALL use that path for spawning the ACP server instead of searching `PATH`

### Requirement: omp ACP session lifecycle

The omp driver SHALL support the full ACP session lifecycle: `session/new`, `session/load`, `session/resume`, `session/fork`, and `session/close`, using the shared `AcpSessionRuntime`.

#### Scenario: New session

- **WHEN** a user starts a new omp thread
- **THEN** the system SHALL spawn `omp acp`, send `initialize`, and call `session/new` with the thread's `cwd`
- **AND** the session SHALL be ready for prompt submission

#### Scenario: Resume existing session

- **WHEN** a user opens an existing omp thread with a stored `sessionId`
- **THEN** the system SHALL spawn `omp acp`, send `initialize`, and call `session/load` or `session/resume` with the stored session ID
- **AND** the conversation history SHALL be restored from omp's session state

#### Scenario: Fork session

- **WHEN** omp advertises `sessionCapabilities.fork` in its `initialize` response
- **THEN** the system SHALL support forking an omp session into a new thread

#### Scenario: Session close on thread exit

- **WHEN** an omp session is closed or the provider instance scope is released
- **THEN** the system SHALL send `session/close` to the omp ACP server
- **AND** the child process SHALL be terminated cleanly

### Requirement: omp authentication via local credentials

The omp driver SHALL authenticate using omp's `"agent"` auth method exclusively, relying on credentials pre-configured under `~/.omp`.

#### Scenario: Local credentials available

- **WHEN** `~/.omp` contains valid omp credentials
- **THEN** the `session/new` call SHALL succeed using auth method `"agent"`
- **AND** no additional authentication configuration SHALL be required from the user

#### Scenario: No local credentials

- **WHEN** `~/.omp` does not contain valid credentials
- **THEN** the first session creation SHALL fail with an `AcpError`
- **AND** the error SHALL be surfaced to the user as a provider adapter error

### Requirement: omp model selection

The omp driver SHALL discover available models dynamically from the ACP `session/new` response `configOptions` and SHALL pass the user-selected model to `session/prompt`.

#### Scenario: Dynamic model discovery

- **WHEN** an omp session is created
- **THEN** the system SHALL extract the model list from the `configOptions` entry with `category: "model"`
- **AND** the models SHALL appear in the web UI model picker

#### Scenario: Model selection persisted across sessions

- **WHEN** a user selects a model for an omp thread
- **THEN** the selected model SHALL be passed to `session/prompt` as a config option
- **AND** resuming the session SHALL preserve the model selection

#### Scenario: Default model fallback

- **WHEN** no model is explicitly selected
- **THEN** the system SHALL use `"umans/umans-glm-5.2"` as the default model for omp

### Requirement: omp mode switching

The omp driver SHALL support omp's `default` and `plan` modes, surfaced through the standard ACP `modes` mechanism.

#### Scenario: Default mode

- **WHEN** an omp session starts without a mode override
- **THEN** the session SHALL use the `default` mode (standard ACP headless mode)

#### Scenario: Plan mode

- **WHEN** a user selects plan mode for an omp thread
- **THEN** the session SHALL use omp's `plan` mode (read-only planning that drafts a plan to a markdown file)
- **AND** the mode change SHALL be communicated via ACP config options

#### Scenario: Mode change event

- **WHEN** omp sends a `current_mode_update` session update
- **THEN** the system SHALL emit a `ModeChanged` runtime event
- **AND** the web UI SHALL reflect the new mode

### Requirement: omp thinking level configuration

The omp driver SHALL support omp's thinking-level config option (`off`, `auto`, `high`, `xhigh`) via the standard ACP config options mechanism.

#### Scenario: Thinking level selection

- **WHEN** an omp session is created and the `thinking` config option is present in `configOptions`
- **THEN** the user SHALL be able to select a thinking level in the traits picker
- **AND** the selected level SHALL be passed to `session/prompt` as a config option

### Requirement: omp prompt execution

The omp driver SHALL execute prompts via the standard ACP `session/prompt` method and SHALL stream `session/update` notifications as `ProviderRuntimeEvent`s.

#### Scenario: Prompt with text content

- **WHEN** a user submits a text prompt to an omp session
- **THEN** the system SHALL call `session/prompt` with the text content
- **AND** streaming `session/update` notifications SHALL be converted to `ProviderRuntimeEvent`s

#### Scenario: Prompt with image content

- **WHEN** omp advertises `promptCapabilities.image` in its `initialize` response
- **THEN** the system SHALL support sending image content blocks in `session/prompt`

#### Scenario: Prompt with embedded context

- **WHEN** omp advertises `promptCapabilities.embeddedContext` in its `initialize` response
- **THEN** the system SHALL support sending embedded context blocks in `session/prompt`

### Requirement: omp permission handling

The omp driver SHALL handle ACP `session/request_permission` notifications and SHALL route permission decisions through T3 Code's approval UI.

#### Scenario: Permission requested

- **WHEN** omp sends a `session/request_permission` notification during prompt execution
- **THEN** the system SHALL surface an approval request in the web UI
- **AND** the user's decision (accept, accept-for-session, decline) SHALL be sent back to omp

### Requirement: omp text generation

The omp driver SHALL provide text generation for git commit messages, PR content, branch names, and thread titles by spawning a short-lived omp ACP session and parsing JSON output.

#### Scenario: Generate commit message

- **WHEN** the system needs a commit message for changes made in an omp thread
- **THEN** the system SHALL spawn a one-shot omp ACP session with a commit-message prompt
- **AND** the response SHALL be parsed as JSON and sanitized into a commit subject

#### Scenario: Generate PR content

- **WHEN** the system needs PR title and body for an omp thread
- **THEN** the system SHALL spawn a one-shot omp ACP session with a PR-content prompt
- **AND** the response SHALL be parsed and sanitized into a PR title and body

#### Scenario: Generate branch name

- **WHEN** the system needs a branch name for an omp thread
- **THEN** the system SHALL spawn a one-shot omp ACP session with a branch-name prompt
- **AND** the response SHALL be sanitized into a valid git branch name

#### Scenario: Generate thread title

- **WHEN** the system needs a thread title for an omp conversation
- **THEN** the system SHALL spawn a one-shot omp ACP session with a thread-title prompt
- **AND** the response SHALL be sanitized into a display title

### Requirement: omp provider settings

The system SHALL expose omp provider settings with a configurable binary path, registered under `ServerSettings.providers.omp`.

#### Scenario: Default settings

- **WHEN** no omp provider settings are configured
- **THEN** the system SHALL use `binaryPath: "omp"` and `enabled: true` as defaults

#### Scenario: Custom binary path

- **WHEN** a user sets a custom `binaryPath` in omp provider settings
- **THEN** the system SHALL use that path for spawning `omp acp`

#### Scenario: Provider disabled

- **WHEN** a user sets `enabled: false` for the omp provider
- **THEN** the provider SHALL NOT appear as a selectable option for new threads
- **AND** existing omp sessions SHALL continue until closed

### Requirement: omp UI registration

The web UI SHALL register omp in the provider picker sidebar and provider icon map.

#### Scenario: Provider appears in picker sidebar

- **WHEN** the omp driver is registered and available
- **THEN** the model picker sidebar SHALL list "Oh My Pi" as a selectable provider

#### Scenario: Provider icon displayed

- **WHEN** an omp model is shown in the model picker
- **THEN** the omp provider icon SHALL be rendered next to the model name
