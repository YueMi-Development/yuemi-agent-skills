---
name: calagopus-extensions
description: |
  Extension development skill for the Calagopus Panel — covers the full extension
  API surface including backend (Rust: Extension trait, routing, settings, events,
  permissions, background tasks, shutdown handlers, CLI commands, model extensions,
  email templates, game protocol tunnels, file storage) and frontend (React/TypeScript:
  Extension class, mounting UI, translations, forms, user settings, toasts, quick actions,
  file manager slots, frontend events).

  Use when the task involves any of:
    - Building, installing, or managing Calagopus Panel extensions
    - Writing backend Rust code for an extension (routes, settings, events, permissions)
    - Writing frontend React/TypeScript for an extension (pages, components, forms)
    - Adding fields to existing Panel forms
    - Integrating with the file manager, quick actions, or toast systems
    - Registering permissions, CLI commands, or background tasks
    - Sending emails via the Panel's template system
    - Querying game server status via Wings tunnels
    - Storing files via the Panel's storage abstraction
    - Extending core models with custom columns
    - Syncing per-user preferences across devices
    - Switching to the heavy Docker image for extension support
    - Troubleshooting extension build, install, or runtime failures
metadata:
  scope: calagopus-extension-development
  targets:
    - Extension trait
    - ExtensionRouteBuilder
    - ExtensionPermissionsBuilder
    - BackgroundTaskBuilder
    - ShutdownHandlerBuilder
    - CliCommandGroupBuilder
    - ExtensionSettingsDeserializer
    - SettingsSerializeExt
    - SettingsDeserializeExt
    - ModelExtension
    - EmailTemplate
    - QueryStreamTunnel
    - QueryUdpTunnel
    - C7sZip
    - heavy Docker image
    - cardConfigurationPage
    - cardComponent
    - extensionRegistry
    - useUserSetting
    - useToast
    - useQuickActions
    - useFileManager
    - FormRegistry
    - ServerCan
    - AdminCan
    - Events
    - Activity Logging
    - Translations
    - File Manager slots
    - Frontend Events
    - Theming
---
# Calagopus Panel Extensions Skill

> Extension development skill for the Calagopus Panel. Extensions ship as `.c7s.zip` files containing both backend (Rust) and frontend (React/TypeScript) code.

**Sources:**
- [Installing Extensions](https://calagopus.com/docs/panel/extensions/installing-extensions)
- [Uninstalling Extensions](https://calagopus.com/docs/panel/extensions/uninstalling-extensions)
- [Disabling Extensions](https://calagopus.com/docs/panel/extensions/disabling-extensions)
- [Switching to the Heavy Image](https://calagopus.com/docs/panel/extensions/switching-to-the-heavy-image)
- [Extension File Structure](https://calagopus.com/docs/panel/extensions/extension-file-structure)
- [Development Environment](https://calagopus.com/docs/panel/extensions/development-environment)
- [Theming](https://calagopus.com/docs/panel/extensions/concepts/theming) — **not yet fetched**
- [Events](https://calagopus.com/docs/panel/extensions/concepts/events)
- [Settings](https://calagopus.com/docs/panel/extensions/concepts/settings)
- [Routing](https://calagopus.com/docs/panel/extensions/concepts/routing)
- [Permissions](https://calagopus.com/docs/panel/extensions/concepts/permissions)
- [CLI Commands](https://calagopus.com/docs/panel/extensions/concepts/cli-commands)
- [Background Tasks & Shutdown Handlers](https://calagopus.com/docs/panel/extensions/concepts/background-tasks-and-shutdown-handlers)
- [Update Checks & Extension Calls](https://calagopus.com/docs/panel/extensions/concepts/update-checks-and-extension-calls)
- [Frontend API Calls](https://calagopus.com/docs/panel/extensions/concepts/frontend-api-calls)
- [Activity Logging](https://calagopus.com/docs/panel/extensions/concepts/activity-logging)
- [Translations](https://calagopus.com/docs/panel/extensions/concepts/translations)
- [Mounting UI](https://calagopus.com/docs/panel/extensions/concepts/mounting-ui)
- [Quick Actions](https://calagopus.com/docs/panel/extensions/concepts/quick-actions)
- [File Manager](https://calagopus.com/docs/panel/extensions/concepts/file-manager)
- [Toasts](https://calagopus.com/docs/panel/extensions/concepts/toasts)
- [Frontend Events](https://calagopus.com/docs/panel/extensions/concepts/frontend-events)
- [Extending Models](https://calagopus.com/docs/panel/extensions/concepts/extending-models)
- [Email Templates](https://calagopus.com/docs/panel/extensions/concepts/email-templates)
- [Speaking Game Protocols](https://calagopus.com/docs/panel/extensions/concepts/speaking-game-protocols)
- [File Storage](https://calagopus.com/docs/panel/extensions/concepts/file-storage)
- [Forms](https://calagopus.com/docs/panel/extensions/concepts/forms)
- [User Settings](https://calagopus.com/docs/panel/extensions/concepts/user-settings)
- [Patching & Adding Translations](https://calagopus.com/docs/panel/extensions/patching-and-adding-translations)

---

## Getting Started

### The Heavy Image

Extension installation requires the Panel to compile new code. The regular `:latest` and `:nightly` Docker images do not include the toolchain. Switch to the `:heavy` (or `:nightly-heavy`) variant. See [Switching to the Heavy Image](https://calagopus.com/docs/panel/extensions/switching-to-the-heavy-image) for the compose.yml changes (image tag + four volume mounts: `./build/binaries`, `./build/translations`, `./build/extensions`, `./build/extension-migrations`).

Web uninstall/rebuild/restart controls are unavailable when `APP_DISABLE_EXTENSION_MANAGEMENT` is enabled, but installed extensions still run and can be toggled. The extension CLI remains available.

### Installing an Extension

Once on `:heavy` or `:nightly-heavy`:

**Option 1 — Admin UI:** Drop the `.c7s.zip` into the upload area on the Panel's extension management page. The Panel handles install, compile, and load.

**Option 2 — File drop + restart:** Copy the `.c7s.zip` into the Panel's `extensions/` data directory (default heavy compose: `./build/extensions`), named `<identifier>.c7s.zip`, then `docker compose restart web`. The Panel detects the new file on startup. To update, overwrite the existing archive. A second archive under a different filename for the same package causes a "already installed" build failure.

### Uninstalling

- **Admin UI:** Extension management page → Uninstall.
- **File removal:** Delete the `.c7s.zip` from `./build/extensions`, then `docker compose restart web`.

### Disabling

Flipping the switch on an extension's card keeps it installed but stops it from running. No recompile needed — works on all deployment types. The Panel picks up the change on next start (shows a "Pending restart" badge until then). On heavy images the UI has a "Restart the panel" button; elsewhere restart manually.

Disabling skips the extension's backend and frontend entrypoints, so its routes, permissions, background tasks, model extensions, and pages are absent. Its data (settings, migrations, permissions) is preserved and restored on re-enable.

Key gotcha: if the extension added a `NOT NULL` column without a default to a core table, model creation fails while the extension is disabled.

CLI commands remain registered when an extension is disabled — they are set up before the Panel reads which extensions are disabled.

---

## Extension Architecture

### File Structure

Extensions follow a standardized multi-part structure:

- **Frontend** — React-based UI layer (TypeScript)
- **Backend** — Rust-based logic layer
- **Database migrations** — optional, applied on startup

See [Extension File Structure](https://calagopus.com/docs/panel/extensions/extension-file-structure) for the complete breakdown of directories, required files, package naming conventions, and entrypoints.

### The Extension Trait (Rust)

The Rust side is a class implementing `Extension`:

```rust
use shared::{State, extensions::Extension};

#[derive(Default)]
pub struct ExtensionStruct;

#[async_trait::async_trait]
impl Extension for ExtensionStruct {
    async fn initialize(&mut self, state: State) { /* ... */ }
    async fn initialize_router(&mut self, state: State, builder: ExtensionRouteBuilder) -> ExtensionRouteBuilder { builder }
    async fn initialize_permissions(&mut self, state: State, builder: ExtensionPermissionsBuilder) -> ExtensionPermissionsBuilder { builder }
    async fn initialize_background_tasks(&mut self, state: State, builder: BackgroundTaskBuilder) -> BackgroundTaskBuilder { builder }
    async fn initialize_shutdown_handlers(&mut self, state: State, builder: ShutdownHandlerBuilder) -> ShutdownHandlerBuilder { builder }
    async fn settings_deserializer(&self, state: State) -> ExtensionSettingsDeserializer { Arc::new(...) }
    async fn check_for_updates(&self, state: State, current_version: &semver::Version) -> Result<Option<ExtensionUpdateInfo>, anyhow::Error> { Ok(None) }
    async fn process_call(&self, name: &str, args: &[ExtensionCallValue]) -> Option<ExtensionCallValue> { None }
    async fn initialize_cli(&mut self, env: Option<&Arc<Env>>, builder: CliCommandGroupBuilder) -> CliCommandGroupBuilder { builder }
    async fn initialize_email_templates(&mut self, state: State, builder: ExtensionEmailTemplateBuilder) -> ExtensionEmailTemplateBuilder { builder }
}
```

### The Extension Class (TypeScript/React)

The frontend entry point is a class extending `Extension`:

```ts
import { Extension, ExtensionContext } from 'shared';

class MyExtension extends Extension {
  public cardConfigurationPage: React.FC | null = MyConfigPage;
  public cardComponent: React.FC | null = MyCardSummary;
  public cardIcon: React.ReactNode = <FontAwesomeIcon icon={faRocket} />;

  public initialize(ctx: ExtensionContext): void {
    // Register UI through ctx.extensionRegistry
  }
}

export default new MyExtension();
```

### Package Naming

Package names follow the `dev_<author>_<name>` identifier convention (from Extension File Structure). This convention is enforced — extensions that don't follow it may not work correctly, especially those adding server pages that depend on an egg feature flag.

---

## Events (Backend)

Extensions listen to model lifecycle events. There are four event emitter types:

| Trait | When it fires |
|---|---|
| `EventEmittingModel` | Model-specific events (e.g. `ServerEvent::InstallStarted`) |
| `CreatableModel` | After a model is created |
| `UpdatableModel` | After a model is updated |
| `DeletableModel` | After a model is deleted |
| `DuplicableModel` | After a model is duplicated |

### EventEmittingModel

```rust
Server::register_event_handler(async |_state, event| {
    match &*event {
        ServerEvent::InstallStarted { server, .. } => { /* ... */ }
        ServerEvent::InstallCompleted { server, successful } => { /* ... */ }
        _ => {}
    }
    Ok(())
});
```

Returns an `EventHandlerHandle`; call `.disconnect()` to remove the listener.

### CreatableModel / UpdatableModel / DeletableModel

Each exposes two hooks — a **before** hook (runs before the DB operation, can modify options or cancel) and an **after** hook (runs after, inside the same transaction):

```rust
Server::register_create_handler(ListenerPriority::Normal, |options, query_builder, state, transaction| {
    Box::pin(async move {
        tracing::info!("creating server with name: {}", options.name);
        Ok(())
    })
});

Server::register_after_create_handler(ListenerPriority::Normal, |result, options, state, transaction| {
    Box::pin(async move {
        tracing::info!("server created with name: {} (result available now)", options.name);
        Ok(())
    })
});
```

`ListenerPriority` determines call order. If a listener returns an error, listeners with lower priority are not called.

---

## Settings

Settings persist arbitrary data per-extension in the Panel's database. Three steps: define a struct, implement `SettingsSerializeExt` + `SettingsDeserializeExt`, and return the deserializer from `settings_deserializer()`.

### Defining the Struct

```rust
#[derive(ToSchema, Serialize, Deserialize)]
pub struct ExtensionSettingsData {
    pub api_key: compact_str::CompactString,
    pub collect_telemetry: bool,
}
```

### Serialization Patterns

**Pattern 1 — One key per field** (use when you need per-field control like encryption):

```rust
impl SettingsSerializeExt for ExtensionSettingsData {
    async fn serialize(&self, serializer: SettingsSerializer) -> Result<SettingsSerializer, anyhow::Error> {
        Ok(serializer
            .write_raw_encrypted_setting("api_key", self.api_key.clone()).await?
            .write_raw_setting("collect_telemetry", self.collect_telemetry.to_compact_string()))
    }
}

impl SettingsDeserializeExt for ExtensionSettingsDataDeserializer {
    async fn deserialize_boxed(&self, mut deserializer: SettingsDeserializer<'_>) -> Result<ExtensionSettings, anyhow::Error> {
        Ok(Box::new(ExtensionSettingsData {
            api_key: deserializer.read_raw_encrypted_setting("api_key").await?.unwrap_or_default(),
            collect_telemetry: deserializer.take_raw_setting("collect_telemetry").and_then(|s| s.parse().ok()).unwrap_or(false),
        }))
    }
}
```

**Pattern 2 — One key for the whole struct** (use for complex types like `Vec<T>`, nested structs):

```rust
impl SettingsSerializeExt for ExtensionSettingsData {
    async fn serialize(&self, serializer: SettingsSerializer) -> Result<SettingsSerializer, anyhow::Error> {
        Ok(serializer.write_serde_setting("item_groups", &self.item_groups)?)
    }
}
```

### Reading and Writing

```rust
// Read
let settings = state.settings.get().await?;
let ext_settings: &ExtensionSettingsData = settings.find_extension_settings()?;

// Write
let mut settings = state.settings.get_mut().await?;
let ext_settings: &mut ExtensionSettingsData = settings.find_mut_extension_settings()?;
ext_settings.api_key = "new_key".into();
settings.save().await?; // Must call save()!
```

### Admin Form

The Panel doesn't auto-build a settings UI. Ship a `cardConfigurationPage` component that calls your own GET/PUT routes. Standard stack: `@mantine/form` + Zod + custom inputs from `@/elements/input/`.

---

## Routing

Routes are registered through `ExtensionRouteBuilder` in `initialize_router`. They are just axum routes wrapped in `utoipa_axum` for automatic OpenAPI docs.

### The Seven Routers

| Method | Mount point | Auth | Use for |
|---|---|---|---|
| `add_global_router` | `/` | None | Public endpoints, webhooks |
| `add_auth_api_router` | `/api/auth` | None | Custom auth flows |
| `add_admin_api_router` | `/api/admin` | Admin session + permission check | Admin-only endpoints |
| `add_client_api_router` | `/api/client` | User session | User-scoped, non-server |
| `add_client_server_api_router` | `/api/client/servers/{server}` | User session + server access | User actions on their servers |
| `add_remote_api_router` | `/api/remote` | Node token | Calls from Wings |
| `add_remote_server_api_router` | `/api/remote/servers/{server}` | Node token + server scope | Wings about a specific server |

### Registering Routes

```rust
async fn initialize_router(&mut self, state: State, builder: ExtensionRouteBuilder) -> ExtensionRouteBuilder {
    builder
        .add_admin_api_router(|routes| {
            routes.nest("/extensions/dev.yourname.test", routes::admin::router(&state))
        })
        .add_client_server_api_router(|routes| {
            routes.nest("/my-feature", routes::server::router(&state))
        })
}
```

### File-System Convention

Mirror your URL tree in your file tree:

```
backend/src/
  lib.rs
  routes/
    mod.rs
    admin/
      mod.rs
      settings.rs
    server/
      mod.rs
      items/
        mod.rs
        _item_/
          mod.rs
          _variant_.rs   # {item}/{variant} path params
```

### Route Handler Pattern

```rust
mod get {
    #[utoipa::path(get, path = "/", responses((status = OK, body = inline(Response))))]
    pub async fn route(state: GetState, permissions: GetPermissionManager) -> ApiResponseResult {
        permissions.has_admin_permission("settings.read")?;
        let settings = state.settings.get().await?;
        let ext_settings: &ExtensionSettingsData = settings.find_extension_settings()?;
        ApiResponse::new_serialized(Response { settings: ext_settings }).ok()
    }
}

pub fn router(state: &State) -> OpenApiRouter<State> {
    OpenApiRouter::new()
        .routes(routes!(get::route))
        .with_state(state.clone())
}
```

### Request Bodies and Validation

```rust
#[derive(ToSchema, Validate, Deserialize)]
pub struct Payload {
    #[garde(url, length(min = 1, max = 255))]
    #[schema(format = "url")]
    api_url: Option<compact_str::CompactString>,
}

#[utoipa::path(put, path = "/", request_body = inline(Payload), responses((status = OK))]
pub async fn route(state: GetState, permissions: GetPermissionManager, shared::Payload(data): shared::Payload<Payload>) -> ApiResponseResult {
    if let Err(errors) = shared::utils::validate_data(&data) {
        return ApiResponse::new_serialized(ApiError::new_strings_value(errors))
            .with_status(StatusCode::BAD_REQUEST).ok();
    }
    // ...
}
```

### Response Types

```rust
// Success
ApiResponse::new_serialized(Response { count: 42 }).ok()

// Error with status
ApiResponse::new_serialized(ApiError::new_strings_value(errors))
    .with_status(StatusCode::BAD_REQUEST)
    .ok()

// Simple error shortcut
return Err(ApiResponse::error("username is already taken"));

// Bubbling with DisplayError (from deep in call stack)
find_item(&id).ok_or_else(|| DisplayError::new("item not found").with_status(StatusCode::NOT_FOUND).into())?;
```

### Hiding Routes from API Docs

Use plain axum routing instead of `routes!()`:

```rust
.add_global_router(|routes| {
    routes.route("/my-extension/callback", axum::routing::get(callback::route))
})
```

---

## Permissions

Permissions are declared through `ExtensionPermissionsBuilder` in `initialize_permissions`. The Panel exposes them in the permission-picker UI.

### The Three Scopes

| Scope | Method | Checked with | Applied to |
|---|---|---|---|
| User | `add_user_permission_group` | `has_user_permission` | Client routes (non-server-scoped) |
| Server | `add_server_permission_group` | `has_server_permission` | Client-server routes |
| Admin | `add_admin_permission_group` | `has_admin_permission` | Admin routes |

### Registering Permissions

```rust
async fn initialize_permissions(&mut self, state: State, builder: ExtensionPermissionsBuilder) -> ExtensionPermissionsBuilder {
    builder.add_server_permission_group(
        "my-feature",
        PermissionGroup {
            description: "Permissions for the My Feature extension.",
            permissions: IndexMap::from([
                ("read", "Allows viewing My Feature data on this server."),
                ("update", "Allows changing My Feature settings on this server."),
            ]),
        },
    )
}
```

Use **kebab-case** for group and permission names (e.g. `egg-changer`, not `eggChanger`).

### Mutating Core Permission Groups

Add a permission to an existing core group instead of creating a new one when it fits naturally:

```rust
builder.mutate_server_permission_group("settings", |group| {
    group.add_permission("egg-changer", "Allows updating the egg of a server.");
});
```

Check with `permissions.has_server_permission("settings.egg-changer")?`.

### Adding Permission Icons (Frontend)

```ts
ctx.extensionRegistry.permissionIcons.addServerPermissionIcon('my-feature', <FontAwesomeIcon icon={faCube} />);
```

### Frontend Permission Checks

Use `ServerCan` / `AdminCan` components to hide UI users can't access:

```tsx
<ServerCan action='settings.egg-changer'>
  <TitleCard title='Change Egg'>...</TitleCard>
</ServerCan>

<AdminCan action='extensions.manage' cantSave>
  <Button onClick={doSave}>Save</Button>
</AdminCan>
```

Hooks for imperative checks: `useServerPermissions(action)` + `useCan(canMatrix, false)`.

> **Security rule:** Backend permission checks are the real security boundary. Frontend `<Can>` components are a UX layer only — never skip the backend `has_*_permission` check.

---

## CLI Commands

Extensions add top-level subcommands to `panel-rs`. Registration is in `initialize_cli`, which runs before the Panel reads which extensions are disabled (so a disabled extension's commands stay available).

```rust
async fn initialize_cli(&mut self, env: Option<&Arc<Env>>, builder: CliCommandGroupBuilder) -> CliCommandGroupBuilder {
    builder.add_group("my-tool", "Tools for the my-tool extension.", commands::commands)
}
```

Users run: `panel-rs my-tool version`, `panel-rs my-tool disable-2fa --user alice`.

### Writing a Command

```rust
use clap::Args;

#[derive(Args)]
pub struct VersionArgs;

pub struct VersionCommand;

impl CliCommand<VersionArgs> for VersionCommand {
    fn get_command(&self, command: clap::Command) -> clap::Command { command }
    fn get_executor(self) -> Box<ExecutorFunc> {
        Box::new(|_env, _arg_matches| {
            Box::pin(async move {
                println!("version {}", env!("CARGO_PKG_VERSION"));
                Ok(0) // 0 = success
            })
        })
    }
}
```

### Accessing State from CLI

```rust
let state = shared::AppState::new_cli(env).await?;
```

### Interactive Fallback

```rust
if std::io::stdout().is_terminal() {
    let user: String = Input::with_theme(&ColorfulTheme::default())
        .with_prompt("Username, Email or UUID")
        .interact_text()?;
} else {
    eprintln!("user arg is required when not running in an interactive terminal");
    return Ok(1);
}
```

Return `Ok(0)` for success, `Ok(non_zero)` for expected failures (with an error printed to stderr). Only return `Err(...)` for unexpected internal errors.

---

## Background Tasks

Background tasks run in a loop for the process lifetime. Registered in `initialize_background_tasks`. **Only runs on the primary instance** — silently no-ops on non-primary instances.

```rust
async fn initialize_background_tasks(&mut self, state: State, builder: BackgroundTaskBuilder) -> BackgroundTaskBuilder {
    builder
        .add_task("refresh-cache", |state| async move {
            crate::cache::refresh(&state).await?;
            tokio::time::sleep(Duration::from_secs(300)).await;
            Ok(())
        })
        .await;
    builder
}
```

- The loop function runs immediately again after it returns — **you must sleep inside it** or it hot-spins.
- Errors are logged and retried at the next iteration; panics terminate the loop permanently.
- Task names must be unique Panel-wide (they share the namespace with core tasks).

### Scheduled (Cron) Tasks

```rust
use std::str::FromStr;

builder
    .add_cron_task(
        "daily-cleanup",
        croner::Cron::from_str("0 0 0 * * *").unwrap(), // midnight UTC
        |state| async move { crate::cleanup::run_daily(&state).await?; Ok(()) }
    )
    .await;
```

No sleep needed — the Panel handles pacing. Missed ticks are skipped, not backlogged.

---

## Shutdown Handlers

Run once during graceful shutdown, before the process exits. Registered in `initialize_shutdown_handlers`. **Runs on every instance**, not just the primary.

```rust
async fn initialize_shutdown_handlers(&mut self, state: State, builder: ShutdownHandlerBuilder) -> ShutdownHandlerBuilder {
    builder
        .add_handler("flush-pending-writes", |state| async move {
            crate::buffer::flush_all(&state).await?;
            Ok(())
        })
        .await;
    builder
}
```

- Handlers run sequentially; one failing handler doesn't block the others.
- Not triggered by SIGKILL, crashes, or OOM — design handlers as best-effort cleanup, not a durability guarantee.
- A background task that accumulates data + a shutdown handler that flushes it is the canonical pairing.

---

## Update Checks & Extension Calls

### Update Checks

```rust
async fn check_for_updates(&self, state: State, current_version: &semver::Version) -> Result<Option<ExtensionUpdateInfo>, anyhow::Error> {
    let release_info: ReleaseInfo = state.client
        .get("https://api.github.com/repos/you/your-extension/releases/latest")
        .send()?.json().await?;
    let latest = semver::Version::parse(&release_info.tag_name.trim_start_matches('v'))?;
    if latest > *current_version {
        Ok(Some(ExtensionUpdateInfo {
            version: latest,
            changes: release_info.body.lines().map(compact_str::CompactString::from).collect(),
        }))
    } else {
        Ok(None)
    }
}
```

Cache rate-limited sources with `state.cache.cached("my-extension::latest-release", 60 * 60, || async { ... })`.

### Extension Calls (In-Process)

Extensions call functions on other loaded extensions via `state.extensions.call()`:

```rust
// Caller side
let result: Option<ExtensionCallValue> = state
    .extensions
    .call("dev.yourname.my-ext:get-thing", &[Box::new("some-id".to_string())])
    .await;

// Callee side
async fn process_call(&self, name: &str, args: &[ExtensionCallValue]) -> Option<ExtensionCallValue> {
    match name {
        "dev.yourname.my-ext:get-thing" => {
            let id = args.get(0)?.downcast_ref::<String>()?;
            let thing = lookup_thing(id).await.ok()?;
            Some(Box::new(thing))
        }
        _ => None,
    }
}
```

- Values are passed as `Box<dyn Any + Send + Sync>` — types must match exactly on both sides.
- The Panel iterates extensions in registration order; the first to return `Some` wins.
- Call names are a shared namespace — prefix with your package name.
- For complex shared data, use a serde JSON intermediate representation to avoid type mismatches.

---

## Activity Logging

Every mutating route should log an activity entry.

### The Three Loggers

| Logger | Import | Use from |
|---|---|---|
| `GetAdminActivityLogger` | `shared::models::admin_activity` | Admin routes |
| `GetServerActivityLogger` | `shared::models::server` | Server routes |
| `GetUserActivityLogger` | `shared::models::user_activity` | Client routes (non-server) |

```rust
pub async fn route(state: GetState, permissions: GetPermissionManager, activity_logger: GetAdminActivityLogger, ...) -> ApiResponseResult {
    // ... do work ...
    activity_logger
        .log(
            "settings:extensions:update",
            serde_json::json!({
                "extension": "dev.yourname.test",
                "changed_fields": ["api_url"],
            }),
        )
        .await;
    ApiResponse::new_serialized(Response {}).ok()
}
```

Event name convention: `scope:subscope.action` (e.g. `server:backup.create`, `settings:extensions:update`). Colons namespace, dots sub-action.

Payload should answer "what happened and what was touched?" — include IDs, before/after values, decision points. Don't duplicate automatic fields (IP, timestamp, user ID are added by the Panel).

Log: mutations, security-relevant actions, anything an operator would need to debug later. Don't log: pure reads, idempotent no-ops.

---

## Translations

### Defining Translations

```ts
import { defineTranslations, defineEnglishItem } from 'shared';

const translations = defineTranslations({
  items: {
    file: defineEnglishItem('File', 'Files'),
  },
  translations: {
    helloWorld: 'Hello World',
    fileCount: 'You have {files}',
    pages: {
      home: { title: 'My Extension', welcome: 'Welcome back, {username}!' },
    },
  },
});

export const useExtTranslations = translations.useTranslations.bind(translations);
export const getExtTranslations = translations.getTranslations.bind(translations);
```

`{braces}` are interpolation variables, type-checked at the call site. Use `defineEnglishItem(singular, plural)` for count-attached keys.

### Using in Components

```tsx
const { t: tExt, tReact: tExtReact, tItem: tExtItem } = useExtTranslations();

return (
  <div>
    <h1>{tExt('helloWorld', {})}</h1>
    <p>{tExt('fileCount', { files: tExtItem('file', 5) })}</p>
  </div>
);
```

For non-React contexts use `getExtTranslations()`.

### Markdown in Translations

```tsx
// Plain markdown
{tExt('myMarkdownTranslation', {}).md()}

// With React node interpolation
{tReact('userMessage', { user: <strong>{name}</strong>, link: <a href="/help">help</a> })}
```

Use `tReact` when any interpolation value is a React node; `t(...).md()` for plain markdown without React.

### Shipping Other Languages

1. Run `pnpm build:translations` in the Panel's frontend directory to generate an English starter file at `frontend/public/translations/en/dev.yourname.extension.json`.
2. Copy it to your extension's `public/translations/<language>/dev.yourname.extension.json` (e.g. `es/dev.yourname.extension.json`).
3. Replace English values with translations.

JSON structure is flat — items and translations at top level, keys as defined in `translations.ts`. Fill in all six CLDR plural categories (`zero`, `one`, `two`, `few`, `many`, `other`) for non-English item translations.

### Using Base Panel Translations

```tsx
import { useTranslations } from '@/providers/TranslationProvider.tsx';

const { t } = useTranslations();
<Button>{t('common.button.save', {})}</Button>
```

Alias clearly when mixing: `const { t } = useTranslations()` + `const { t: tExt } = useExtTranslations()`.

---

## Mounting UI

### The Extension Class Fields

```ts
class MyExtension extends Extension {
  public cardConfigurationPage: React.FC | null = MyConfigPage; // mounts at /admin/extensions/<id>
  public cardComponent: React.FC | null = MyCardSummary;         // summary on the admin extension card
  public cardIcon: React.ReactNode = <FontAwesomeIcon icon={faRocket} />;
  public initialize(ctx: ExtensionContext): void { /* more UI */ }
}
```

### Slotting Into Existing Pages

```ts
ctx.extensionRegistry.pages.server.settings.enterSettingContainers((containers) =>
  containers.appendComponent(MyServerSettingsCard),
);
```

Slot methods: `appendComponent` (end), `prependComponent` (beginning). For container slots: `prependContentComponent`, `appendContentComponent`.

Use CSS `order-` utility classes to land between existing entries (stock entries use `order-10`, `order-20`, etc.).

Action bar slots: `enterActionBar` on `server.schedules`, `server.backups`, `server.databases`, etc.

### Adding New Routes

```ts
ctx.extensionRegistry.enterRoutes((routes) =>
  routes.addServerRoute({
    name: () => getExtTranslations().t('pages.server.versions.title', {}),
    icon: faCube,
    path: '/minecraft/versions',
    element: MinecraftVersionsPage,
    permission: 'settings.my-feature',
    filter: () => getUserSetting('dev.example.myfeature::enabled', z.boolean(), true),
  }),
);
```

Route types: `addGlobalRoute`, `addAuthenticationRoute`, `addAccountRoute`, `addAdminRoute`, `addServerRoute`.

### Container Wrappers

Route components must wrap themselves in the appropriate container:

| Route type | Wrapper |
|---|---|
| `addGlobalRoute` / `addAuthenticationRoute` | `ContentContainer` |
| `addAccountRoute` | `AccountContentContainer` |
| `addAdminRoute` (top-level) | `AdminContentContainer` |
| `addAdminRoute` (under SubNavigation) | `AdminSubContentContainer` |
| `addServerRoute` | `ServerContentContainer` |

```tsx
import ServerContentContainer from '@/elements/containers/ServerContentContainer.tsx';

export default function MyServerPage() {
  return (
    <ServerContentContainer title={tExt('pages.server.myfeature.title', {})}>
      {/* page content */}
    </ServerContentContainer>
  );
}
```

### Interceptors

Reorder routes or replace a stock page's element:

```ts
routes.addServerRouteInterceptor((items) => {
  // reorder, or replace by path
  const idx = items.findIndex(i => i.path === '/server/~/files');
  if (idx >= 0) items[idx].element = MyReplacementComponent;
});
```

### Reading Panel State

```tsx
import { useServerStore } from '@/stores/server.ts';
import { useGlobalStore } from '@/stores/global.ts';

const server = useServerStore((state) => state.server);
const { permissions } = useGlobalStore((state) => ({ permissions: state.permissions }));
```

Outside React: `getGlobalStore()` from `@/stores/global.ts`.

---

## Quick Actions

The quick actions palette (Mod+Space) aggregates everything the user can do. Extensions contribute actions, categories, and modes.

### Registering a Global Action

```ts
ctx.extensionRegistry.enterQuickActions((quickActions) =>
  quickActions
    .addCategory({ id: 'dev.0x7d8.cleanup', label: () => tExt('quickAction.category', {}), icon: faBroom, order: 25 })
    .addAction({
      id: 'dev.0x7d8.cleanup.purgeLogs',
      category: 'dev.0x7d8.cleanup',
      label: () => tExt('quickAction.purgeLogs', {}),
      keywords: ['clean', 'logs'],
      icon: <FontAwesomeIcon icon={faBroom} />,
      scopes: ['server'],
      permission: 'files.delete',
      perform: () => purgeLogs(),
    }),
);
```

### Page-Scoped Actions (use a hook instead)

```tsx
import { useQuickActions } from '@/plugins/quick-actions/useQuickActions.ts';

export default function MyServerPage() {
  useQuickActions([{
    id: 'dev.0x7d8.cleanup.purgeSelected',
    category: CORE_QUICK_ACTION_CATEGORIES.page,
    label: () => tExt('quickAction.purgeSelected', {}),
    icon: faFileCirclePlus,
    isVisible: () => selected.length > 0,
    perform: () => purge(selected),
  }]);
  return /* ... */;
}
```

### Modes

Modes activate on a prefix (e.g. `%`) and replace the palette's item list with computed results:

```ts
const { useQuickActionModes, useQuickActionTerm } = '@/plugins/quick-actions/useQuickActions.ts';
const term = useQuickActionTerm('%'); // null when not active, '' when active without query

useQuickActionModes([{
  id: 'dev.0x7d8.players',
  prefix: '%',
  hint: () => tExt('quickAction.playerHint', {}),
  items: players.items.map((player) => ({
    key: `player:${player.uuid}`,
    category: 'dev.0x7d8.cleanup',
    label: player.name,
    description: player.uuid,
    icon: faUser,
    onSelect: () => { setOpen(false); kickPlayer(player.uuid); },
  })),
}]);
```

Prefixes core doesn't use: `%`, `~`, `&`, `:`, `!`.

### Guidelines

- Prefix ids with your package identifier.
- Pass getter functions for `label` so they re-resolve on language switch.
- Add `keywords` for alternative search terms.
- Reserve `danger: true` for destructive actions.
- Confirm destructive actions — have `perform` open a modal instead of acting directly.

---

## File Manager

The file manager has extensive extension surface via `ctx.extensionRegistry.pages.server.files`.

### Component Slots

| Slot | Renders | Visible |
|---|---|---|
| `enterContainer` | Around whole `/files` page | Always (cannot use `useFileManager`) |
| `enterEditorContainer` | Around editor route | Editor page only |
| `enterFileToolbar` | Either side of Connect/New | Both views |
| `enterFileTreeToolbar` | Icon row at top of tree sidebar | Tree view only |
| `enterFileActionBar` | Floating bar when rows selected | While selected |
| `enterFileOperationsProgress` | Inside operations popover | During archive/extract/upload |
| `enterFileSettings` | Gear popover next to page title | Both views |
| `enterFileEditorSettings` | Gear popover in editor | Edit/new only |
| `enterFileImageViewerSettings` | Gear popover in image viewer | Image only |

### Context Menus

```ts
ctx.extensionRegistry.pages.server.files.enterFileContextMenu((menu) =>
  menu.addItemInterceptor((items, { file, directory, surface }) => {
    if (!file.name.endsWith('.yml')) return;
    items.push(
      { type: 'divider' },
      { type: 'action', icon: faWandMagicSparkles, label: 'Format YAML', color: 'gray',
        onClick: () => formatYaml(join(directory, file.name)) },
    );
  }),
);
```

For mass context menu (no props passed to interceptor), use `addComponentItemInterceptor` so you can call `useFileManager` to read `selectedFiles`.

### Custom File Icons

```ts
ctx.extensionRegistry.pages.server.files
  .addFileIconHandler((file) => file.name.endsWith('.mcfunction') ? faScroll : undefined)
  .addFileOpenableHandler((file) =>
    file.name.endsWith('.mcfunction')
      ? { openable: true, handleOpen: ({ handleFileOpen }) => handleFileOpen(file.name, 'mcfunction', {}) }
      : { openable: false },
  );
```

Icon handlers run before built-in checks; openable handlers run in the middle. You cannot claim directories, archives, or files the Panel already handles.

### Custom Editors

```ts
ctx.extensionRegistry.pages.server.files.addFileEditorAction({
  name: 'mcfunction',
  title: (file) => `Editing ${file}`,
  contentType: 'string', // 'string' works in tree pane, 'blob' never does
  content: McFunctionEditor,
  header: { rightSection: McFunctionSaveButton },
});
```

This creates the route `/server/<id>/files/mcfunction?directory=...&file=...`. `contentType: 'string'` passes decoded text + `setContent`; `'blob'` passes raw `Blob` with no Panel persistence (supply your own save via `header.rightSection`).

### The Trash Page

Trash page slots via `ctx.extensionRegistry.pages.server.files.trash`:
- `enterContainer` — around whole trash page
- `enterActionBar` — Restore/Delete Permanently buttons
- `enterEntryContextMenu` — row menu

### Reading File Manager State

```tsx
import { useFileManager } from '@/providers/contexts/fileManagerContext.ts';

const directory = useFileManager((state) => state.browsingDirectory);
const selectedFiles = useFileManager((state) => state.selectedFiles);
const browsingWritableDirectory = useFileManager((state) => state.browsingWritableDirectory);
```

`useFileManager` only works inside `FileManagerProvider` — which excludes the `container` slot.

### Permissions

The file manager surfaces `files.read-content`, `files.create`, `files.update`, `files.archive`, `files.delete`, `files.query-raw`. Context menu items take a `canAccess` boolean.

---

## Toasts

Transient feedback cards in the corner. Use `useToast` from `@/providers/ToastProvider.tsx`:

```tsx
const { addToast } = useToast();
addToast('Settings saved.', 'success'); // type defaults to 'success'
addToast('Node is unreachable.', 'error');
addToast('Saved, but restart needed.', 'warning');
addToast('Import started in background.', 'info');
```

### With Actions

```tsx
addToast('Import finished.', 'success', [
  { name: 'Show files', icon: faFolderOpen, onClick: () => navigate(`/server/${uuid}/files?directory=${path}`) },
]);
```

### Progress Toasts

```tsx
const { addProgressToast, updateToast, dismissToast } = useToast();
const id = addProgressToast('Uploading world.zip', { progress: 0 });

await upload(file, (percent) => updateToast(id, { progress: percent }));
dismissToast(id);
addToast('Upload complete.', 'success');
```

**Critical:** `dismissToast` must be called on every path (success, error, cancel) — progress toasts have no auto-dismiss. Don't convert a finished progress toast to a completion toast with `updateToast` — it keeps the missing timeout and close button.

### Undoable Toasts

```tsx
import { useUndoableToast } from '@/plugins/toast/useUndoableToast.ts';

const addUndoableToast = useUndoableToast(`server:${server.uuid}:my-extension`);
addUndoableToast('Renamed 3 files.', () => renameFiles(...).then(({ renamed }) => {
  if (renamed < 1) { addToast('Could not undo.', 'error'); return; }
  addToast('Rename undone.', 'success');
  invalidate();
}).catch(err => addToast(httpErrorToHuman(err), 'error')));
```

Pass `null` instead of a function for non-reversible operations. Scope strings are used by `runLastUndoEntry()` for the Ctrl+Z keyboard shortcut.

### Toasting API Errors

```tsx
updateThing(values)
  .then(() => addToast('Saved.', 'success'))
  .catch(err => addToast(httpErrorToHuman(err), 'error'))
  .finally(() => setSaving(false));
```

### Intercepting Toasts (Frontend Events)

```ts
ctx.extensionRegistry.enterEvents((events) =>
  events.enterToast((toast) =>
    toast
      .addShowHandler((event) => { /* edit event.data.message */ })
      .addUpdateHandler((event) => { /* intercept updates */ })
      .addDismissHandler((event) => { if (event.reason === 'timeout' && event.toast.type === 'error') event.cancel(); }),
  ),
);
```

---

## Frontend Events

The frontend event system lets extensions intercept or modify Panel behavior before it happens. All registered in `initialize()` via `enterEvents`.

All events are `CancellableEvent<D>` — call `event.cancel()` to stop the action, `event.uncancel()` to undo a prior cancel.

Priority: `highest` → `high` → `normal` → `low` → `lowest`. `receiveCancelled: true` on a handler lets it run after a prior cancel and potentially uncancel.

### Event Groups

| Group | Enter with | Events |
|---|---|---|
| Toasts | `enterToast` | show, update, dismiss |
| Server Websocket | `enterServerWebsocket` | message (from Wings), send (to Wings) |
| API | `enterApi` | request, response |
| Auth | `enterAuth` | logout, impersonate |
| Clipboard | `enterClipboard` | copy, paste |
| Quick Actions | `enterQuickActions` | select |

### Server Websocket

```ts
ctx.extensionRegistry.enterEvents((events) =>
  events.enterServerWebsocket((socket) =>
    socket
      .addMessageHandler((event) => {
        if (event.data.event === SocketEvent.CONSOLE_OUTPUT) {
          event.data.args = event.data.args.map(line => line.replaceAll('secret-token', '********'));
        }
      })
      .addSendHandler((event) => {
        if (event.data.event === SocketRequest.SEND_COMMAND && event.data.args[0]?.startsWith('op ')) {
          event.cancel();
        }
      }),
  ),
);
```

Cancelled incoming messages never reach Panel listeners. Cancelled outgoing frames never leave the browser. Auth/setup frames bypass handlers entirely.

### API

```ts
events.enterApi((api) =>
  api
    .addRequestHandler((event) => { event.data.headers.set('X-My-Extension', '1'); })
    .addResponseHandler((event) => { if (event.data.config.url === '/api/client/account') { /* ... */ } }),
);
```

Cancelled requests reject with an `AxiosError` bearing code `API_REQUEST_CANCELLED_CODE`. Only successful responses trigger `addResponseHandler`. The very first Panel request (settings load before extension init) is not covered.

### Auth

```ts
events.enterAuth((auth) =>
  auth.addLogoutHandler((event) => {
    if (event.kind === 'logout' && hasUnsavedWork()) {
      event.cancel();
      showUnsavedWorkWarning();
    }
  }),
);
```

Login is not an event (the session already exists by the time the frontend learns about it). An expired session doesn't emit `AuthLogoutEvent` — the Panel just drops to the login page.

### Clipboard

```ts
events.enterClipboard((clipboard) =>
  clipboard.addCopyHandler((event) => { event.data = redactSecrets(event.data); }),
);
```

Cancelling `copyToClipboard` / `pasteFromClipboard` causes it to reject with `ClipboardCancelledError`.

---

## Extending Models

Model extensions add columns to core tables, hook into create/update/delete flows, and extend API structs so columns appear in JSON.

### The Migration

```sql
-- up.sql
ALTER TABLE "servers" ADD COLUMN "subdomains" integer NOT NULL DEFAULT 0;
-- down.sql
ALTER TABLE "servers" DROP COLUMN "subdomains";
```

### Model Extension

```rust
#[derive(Serialize, Deserialize)]
pub struct ServerExtensionData { pub subdomains: i32 }

pub struct ServerExtension;

impl SafeModelExtension for ServerExtension {
    type Value = ServerExtensionData;
    fn name() -> &'static str { ServerExtension.extension_name() }
}

impl ModelExtension for ServerExtension {
    fn extension_name(&self) -> &'static str { "dev.yourname.subdomains" }
    fn extended_columns(&self, prefix: &str) -> BTreeMap<&'static str, compact_str::CompactString> {
        BTreeMap::from([("servers.subdomains", compact_str::format_compact!("{prefix}subdomains"))])
    }
    fn map_extended(&self, prefix: &str, row: &PgRow) -> Result<ModelExtensionMapType, DatabaseError> {
        Ok(Box::new(ServerExtensionData {
            subdomains: row.try_get(compact_str::format_compact!("{prefix}subdomains").as_str())?,
        }))
    }
}
```

### Registering Everything

```rust
async fn initialize(&mut self, _state: State) {
    // 1. Register SELECT hook
    Server::register_model_extension(model::ServerExtension);

    // 2. Hook CREATE
    Server::register_create_handler(ListenerPriority::Normal, |options, query_builder, _state, _transaction| {
        Box::pin(async move {
            if let Ok(extended) = options.feature_limits.parse_extended::<model::ExtendedApiServerFeatureLimits>()
                && let Some(value) = extended.subdomains
            {
                query_builder.set("subdomains", value);
            }
            Ok(())
        })
    });

    // 3. Hook UPDATE
    Server::register_update_handler(ListenerPriority::Normal, |_server, options, query_builder, _state, _transaction| {
        Box::pin(async move {
            if let Some(feature_limits) = &options.feature_limits
                && let Ok(extended) = feature_limits.parse_extended::<model::ExtendedApiServerFeatureLimits>()
                && let Some(value) = extended.subdomains
            {
                query_builder.set("subdomains", value);
            }
            Ok(())
        })
    });

    // 4. Extend API struct
    ApiServerFeatureLimits::extend_validated(
        |server, _state| Box::pin(async move { Ok(server.parse_model_extension::<model::ServerExtension>()?) }),
        |_limits, extension, _state| model::ExtendedApiServerFeatureLimits { subdomains: Some(extension.subdomains) },
    );
}
```

### Extended API Struct

```rust
#[derive(ToSchema, Validate, Serialize, Deserialize)]
pub struct ExtendedApiServerFeatureLimits {
    #[garde(range(min = 0))]
    #[schema(minimum = 0)]
    pub subdomains: Option<i32>, // Option is required for backward compatibility
}
```

Use `Option<T>` — existing API clients don't know about your field, and `None` means "don't touch the column."

### Reading from Frontend

```tsx
import { useServerStore } from '@/stores/server.ts';
import { z } from 'zod';

const schema = z.object({ subdomains: z.number() });
const { subdomains } = schema.parse(server.featureLimits);
```

Values ride on the API response object untyped. Parse with your own Zod schema to get typed access.

---

## Email Templates

### Registering a Template

```rust
async fn initialize_email_templates(&mut self, _state: State, builder: ExtensionEmailTemplateBuilder) -> ExtensionEmailTemplateBuilder {
    builder.add_template(EmailTemplate {
        identifier: "dev.0x7d8.test.welcome",
        available_variables: vec!["user", "invite_link"],
        default_subject: "{{ settings.app.name }} - Welcome",
        default_content: include_str!("../mails/welcome.html"),
        default_enabled: true,
    })
}
```

Identifier must be globally unique — prefix with your package name.

### Writing Template Content

Templates use MiniJinja syntax. Variables are `{{ variable }}`. Operators can override subject and body from the admin UI independently.

```html
<p>Hello {{ user.name }},</p>
<p>Click <a href="{{ invite_link }}">here</a> to set a password.</p>
```

Use variables for wording that changes per language — keep the template itself in English.

### Variables and Translations

Ship variable defaults with `include_dir`:

```rust
const MAIL_VARIABLES: include_dir::Dir<'_> = include_dir::include_dir!("$CARGO_MANIFEST_DIR/mails/variables");
builder.add_template_variables(&MAIL_VARIABLES)
```

JSON structure: `{ "dev.0x7d8.test.welcome": { "subject": "Welcome", "greeting": "Hello <strong>{{ user.username }}</strong>," } }`.

### Sending Email

```rust
state.mail.send_template(state, "dev.0x7d8.test.welcome", user, minijinja::context! {
    user => user,
    invite_link => invite_link,
}).await;
```

- `send_template` is fire-and-forget (spawns a task, failures logged internally).
- `send_template_foreground` awaits and propagates errors.
- If the template is disabled, send is silently skipped.
- `settings` and `language` are implicit template globals — don't pass them yourself.
- A 15-second cache applies to template content and enabled state.

### Overriding Core Templates

```rust
builder.mutate_template("password_reset", |template| {
    template.default_subject = "Password Reset - My Brand";
    template.default_content = include_str!("../mails/branded_password_reset.html");
})
```

---

## Speaking Game Protocols

Query tunnels open a raw TCP/UDP socket to a game server's port, proxied through Wings. This lets extensions speak the game's own wire protocol.

### Getting a Tunnel

```rust
let client = server.node.fetch_cached(&state.database).await?
    .api_client(&state.database).await?;

let mut tunnel = client.open_tunnel_tcp(server.uuid, port).await
    .map_err(|err| std::io::Error::other(anyhow::Error::from(err)))?;
```

Methods: `open_tunnel_tcp` (returns `QueryStreamTunnel` — `AsyncRead + AsyncWrite`), `open_tunnel_udp` (datagram socket), `open_tunnel_unix` (unix socket in server files).

UDP `recv` has a built-in 5-second timeout. A refused UDP connection returns `ConnectionRefused` immediately.

### Example: Minecraft Server List Ping (TCP)

```rust
// Write handshake (VarInt-encoded packets, then JSON status request)
write_varint(&mut tunnel, body.len() as i32).await?;
tunnel.write_all(&body).await?;
write_varint(&mut tunnel, 1).await?;
write_varint(&mut tunnel, 0x00).await?; // empty status request
tunnel.flush().await?;

// Read response
let json_len = read_varint(&mut tunnel).await? as usize;
let mut json = vec![0u8; json_len];
tunnel.read_exact(&mut json).await?;
let status: serde_json::Value = serde_json::from_slice(&json)?;

let motd = match &status["description"] {
    serde_json::Value::String(s) => s.clone(),
    serde_json::Value::Object(_) => status["description"]["text"].as_str().unwrap_or_default().to_string(),
    _ => String::new(),
};
```

VarInt encoding helpers are needed — write a `write_varint` / `read_varint` pair using `AsyncWriteExt` / `AsyncReadExt`.

### Warning on Hot Paths

Querying game servers hits the network and can take seconds. Don't call query tunnels on every server in a list on every page load — cache results with a short TTL or refresh known servers via a background task.

---

## File Storage

Extensions store files via `state.storage`, which routes to either the local filesystem or S3 depending on operator config.

### The Three Core Operations

```rust
// Store a file
let bytes_written = state.storage
    .store("publicdata/extensions/dev.yourname.my-feature/uuid.bin", body, "image/png")
    .await?;

// Get a public URL
let urls = state.storage.retrieve_urls().await?;
let url = urls.get_url("publicdata/extensions/dev.yourname.my-feature/uuid.bin");

// Delete
state.storage.remove("publicdata/extensions/dev.yourname.my-feature/uuid.bin").await?;
```

`store` accepts any `AsyncRead + Unpin` — stream directly from multipart uploads or HTTP responses without buffering the whole file.

### Directory Prefixes

| Prefix | Public? | Use for |
|---|---|---|
| `assets/` | Yes | Admin branding — don't write here unless you mean it |
| `avatars/` | Yes | User avatars — managed by core |
| `publicdata/` | Yes | Extension files that can be publicly linked |
| `privatedata/` | No | Extension files requiring authentication |

Extensions should use `publicdata/extensions/<package-id>/` or `privatedata/extensions/<package-id>/`.

### Cache Headers

Filesystem backend sets `Cache-Control` automatically: `avatars/` → `public, max-age=31536000, immutable`; `publicdata/` → `no-cache`. On S3, use `store_with_cache_control` to set a `Cache-Control` header.

### Listing and Searching

```rust
// Shallow listing
let page = state.storage.list("publicdata/extensions/dev.yourname.my-feature", "", 1, 50).await?;

// Name search (walks entire subtree, expensive on S3)
let results = state.storage.search("publicdata/extensions/dev.yourname.my-feature", "", "report", 50).await?;
```

`search` is O(subtree) — on S3 it issues sequential `ListObjectsV2` calls and filters client-side. Debounce user-driven searches, scope narrowly, and don't use it as a general lookup mechanism.

### Temporary Files

Use the `tempfile` crate for ephemeral files — `state.storage` is persistent storage:

```rust
let mut tempfile = NamedTempFile::new()?;
// drops on scope exit, file deleted automatically
```

### Cap Filesystem Escape Hatch

For cases the high-level API doesn't cover (random-access reads, complex traversal):

```rust
let cap_fs = settings.storage_driver
    .get_cap_filesystem("publicdata/extensions/dev.yourname.my-feature").await?;
```

Only works on filesystem deployments; returns `None` on S3. The root argument must not contain user input — `get_cap_filesystem` itself must use a hardcoded path.

---

## Forms

Extensions add fields to existing Panel forms via `enterForms`:

```ts
ctx.extensionRegistry.enterForms((forms) =>
  forms.extend('admin.servers.create', {
    zodShape: {
      customIdentifier: z.string().max(64),
      monitoringEnabled: z.boolean(),
    },
    initialValues: {
      customIdentifier: '',
      monitoringEnabled: false,
    },
    transform: (fields) => [
      ...insertFieldsAfter(fields, 'description', {
        type: 'text',
        name: 'customIdentifier',
        label: 'Provisioning Tag',
        description: 'Passed to the provisioning system on first start.',
      } satisfies FieldDef),
      { type: 'switch', name: 'monitoringEnabled', label: 'Enable Monitoring', advanced: true } satisfies FieldDef,
    ],
  }),
);
```

Form IDs are a typed union (`FormId`) — typos are compile errors.

### Transform Helpers

- `insertFieldsAfter(fields, name, ...insert)` — insert after a named field
- `insertFieldsBefore(fields, name, ...insert)` — insert before
- `updateField(fields, name, fn)` — replace a field
- `removeField(fields, name)` — drop a field

### Field Types

All field types (`FieldDef` discriminated union): `text`, `password`, `textarea`, `number`, `switch`, `checkbox`, `select`, `multiselect`, `multiselectgroup`, `autocomplete`, `date`, `tags`, `numberTags`, `size`, `localizedtext`, `localizedtextarea`, `divider`, `section`, `custom`.

Common options on most types: `name`, `label`, `description`, `tooltip`, `required`, `advanced`, `colSpan` (`'full'` or `1`), `when` ( `(values) => boolean` — show/hide).

### Custom Fields

```ts
{ type: 'custom', name: 'myField', label: 'Custom', render: (form) => <MyCustomInput {...form.getInputProps('myField')} /> }
```

### Conditional Fields

```ts
{ type: 'text', name: 'driverPath', label: 'Driver Path', when: (values) => values.driver === 'custom' }
```

---

## User Settings

Per-user key-value store that syncs across devices. Unlike global settings, no backend code needed — pick a key, read/write from the frontend.

### Naming

Namespace with `::` and your extension identifier: `dev.example.myextension::view_mode`. Don't write into bare namespaces like `app::`, `file_manager::`, `server::`.

### Frontend API

```ts
import { useUserSetting } from '@/lib/userSettings.ts';

// In a component — reactive, auto-syncs
const [viewMode, setViewMode] = useUserSetting(
  'dev.example.myextension::view_mode',
  z.enum(['list', 'grid']),
  'list',
);

// Outside components
import { getUserSetting, setUserSetting, removeUserSetting } from '@/lib/userSettings.ts';
const mode = getUserSetting('dev.example.myextension::view_mode', z.enum(['list', 'grid']), 'list');
setUserSetting('dev.example.myextension::view_mode', 'grid');
removeUserSetting('dev.example.myextension::view_mode');
```

The Zod schema validates on read; a malformed or missing value falls back to the default. The backend deliberately doesn't validate shapes.

### Device-Local Values

Some preferences shouldn't sync — use `setUserSettingLocal` instead. Reads through `useUserSetting` / `getUserSetting` prefer the device-local value.

```ts
setUserSettingLocal('dev.example.myextension::audio-device', 'builtin-speaker');
```

Users can pin any synced setting to one device via the scope menu. Helpers: `overrideUserSettingLocally(key, value)`, `clearUserSettingOverride(key)`, `pushUserSettingToAccount(key)`.

### Backend Access

```rust
let settings = user.get_settings(&state.database).await?;
let prefs: Option<MyPrefs> = settings.get("dev.example.myextension::prefs");
```

Reads are cached 60 seconds per user. For writes: `user.get_settings_mut(&state.database)` + `settings.save(&state.database)`.

### Rules

- JSON `null` deletes the key — not storable.
- Operators control max keys (512) and max value size (16 KiB) per user.
- Store preferences, not documents.
- For per-item flags (dismissals, expanded groups), store one map-valued key and prune stale entries on write.

---

## Theming

> **Not yet fetched.** See [Theming](https://calagopus.com/docs/panel/extensions/concepts/theming).

Theming lets extensions customize the look and feel of the Panel interface. See the source documentation for how to override Mantine theme configuration, register custom themes, and hook into the Panel's styling system.

---

## Patching & Adding Translations

> **Not yet fetched.** See [Patching & Adding Translations](https://calagopus.com/docs/panel/extensions/patching-and-adding-translations).

On the heavy image, operators can override the Panel's built-in translations or add new languages by placing JSON files in specific directories. See the source documentation for the exact file locations, naming conventions, and how the override system works.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `/app/binaries is missing` | Add the four heavy image volume mounts to compose.yml |
| Old version after `docker compose pull` | `docker compose down && rm -r ./build/binaries && docker compose up -d` |
| Build fails or 504 during install | Check build logs at Admin → Extensions → View build logs. If OOM, add RAM or swap |
| Build stuck on "Compiling..." | Cancel build + Retry, or `docker compose down && docker compose up -d` |
| Build failed, alert names an extension | Remove or update the named extension, then retry |
| Frontend broken after extension update | Stop stack, remove archive from `./build/extensions`, remove `./build/binaries`, `docker compose up -d --force-recreate` |
| Extension declares panel version >=1.0.0 (allows older than 1.1.0) | Extension author needs to update to declare `>=1.1.0` |
| Extension builds but does nothing | Check the package name follows `dev_<author>_<name>` convention |
| Binary install can't install extensions | Build the panel yourself with extensions included — see Development Environment |

---

## Extension File Structure & Development Environment

Not yet fetched. See:
- [Extension File Structure](https://calagopus.com/docs/panel/extensions/extension-file-structure) — complete breakdown of directories, required files, package naming, and entrypoints
- [Development Environment](https://calagopus.com/docs/panel/extensions/development-environment) — setting up a local environment for building extensions
