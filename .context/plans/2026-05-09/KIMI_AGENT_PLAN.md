---
createdAt: "2026-05-09T14:14:12Z"
implementedAt: "2026-05-09T14:45:00Z"
reviewedAt: null
---

# Plan: Add Kimi Code CLI Agent Support

## Overview

Add full Level 5 support for Kimi Code CLI agent to Agent of Empires. This includes:
- Agent registry entry with metadata
- Hook-based status detection (Level 3) with TOML config
- Session resume support via `--session` flag
- Docker sandbox support with config mounting
- Full test coverage following existing patterns

Target platforms: Mac/Linux only.

## Target Structure

No new directories. All changes modify existing files in `src/`.

## Files to Create

None. All changes modify existing files.

## Files to Modify

### 1. `src/agents.rs`

Add `AgentDef` entry for kimi to the `AGENTS` array (after kiro):
- `name`: "kimi"
- `binary`: "kimi"
- `aliases`: `&[]`
- `detection`: `DetectionMethod::Which("kimi")`
- `yolo`: `Some(YoloMode::CliFlag("--yolo"))`
- `instruction_flag`: `None`
- `set_default_command`: `false`
- `detect_status`: `status_detection::detect_kimi_status`
- `container_env`: `&[("KIMI_CONFIG_DIR", "/root/.kimi")]`
- `hook_config`: `None` (custom TOML format, uses `install_kimi_hooks`)
- `resume_strategy`: `ResumeStrategy::Flag("--session")`
- `host_only`: `false`
- `send_keys_enter_delay_ms`: `0`
- `install_hint`: `"curl -LsSf https://code.kimi.com/install.sh | bash"`

Update test functions:
- `test_get_agent_known`: Add `assert_eq!(get_agent("kimi").unwrap().binary, "kimi");`
- `test_agent_names`: Add `"kimi"` to expected list at end
- `test_resolve_tool_name`: Add `assert_eq!(resolve_tool_name("kimi"), Some("kimi"));`
- `test_settings_index_roundtrip`: Add kimi at index 13 (after kiro at 12)
- `test_install_hint_lookup`: Add kimi install hint verification

### 2. `src/tmux/status_detection.rs`

Add stub status detection function (like settl, hermes, kiro):
```rust
/// Kimi Code CLI status is detected via hooks (TOML-based), not tmux pane parsing.
/// This stub exists so the agent registry has a valid function pointer.
pub fn detect_kimi_status(_content: &str) -> Status {
    Status::Idle
}
```

### 3. `src/hooks/mod.rs`

Add Kimi hook events constant and installer/uninstaller (like settl):

```rust
/// Kimi Code CLI hook events and the AoE status they map to.
/// Kimi uses TOML config with `[[hooks]]` array entries.
const KIMI_HOOKS: &[(&str, &str)] = &[
    ("PreToolUse", "running"),
    ("Stop", "idle"),
    ("Notification", "waiting"),
];

/// Install AoE status hooks into Kimi's `~/.kimi/config.toml`.
///
/// Kimi uses TOML config with `[[hooks]]` array entries.
pub fn install_kimi_hooks(config_path: &Path) -> Result<()> {
    let home = dirs::home_dir().ok_or_else(|| anyhow::anyhow!("No home directory"))?;
    let config_path = home.join(".kimi").join("config.toml");

    let mut config: toml::Value = if config_path.exists() {
        let content = std::fs::read_to_string(&config_path)?;
        toml::from_str(&content).unwrap_or_else(|e| {
            tracing::warn!("Failed to parse {}: {}", config_path.display(), e);
            toml::Value::Table(toml::map::Map::new())
        })
    } else {
        toml::Value::Table(toml::map::Map::new())
    };

    let table = config
        .as_table_mut()
        .ok_or_else(|| anyhow::anyhow!("Config root is not a TOML table"))?;

    let hooks = table
        .entry("hooks")
        .or_insert_with(|| toml::Value::Array(Vec::new()));
    let hooks_arr = hooks
        .as_array_mut()
        .ok_or_else(|| anyhow::anyhow!("hooks key is not a TOML array"))?;

    hooks_arr.retain(|hook| {
        !hook
            .get("command")
            .and_then(|c| c.as_str())
            .is_some_and(is_aoe_hook_command)
    });

    for (event, status) in KIMI_HOOKS {
        let mut entry = toml::map::Map::new();
        entry.insert("event".into(), toml::Value::String((*event).into()));
        entry.insert("command".into(), toml::Value::String(hook_command(status)));
        hooks_arr.push(toml::Value::Table(entry));
    }

    if let Some(parent) = config_path.parent() {
        std::fs::create_dir_all(parent)?;
    }
    let formatted = toml::to_string_pretty(&config)?;
    std::fs::write(&config_path, formatted)?;

    tracing::info!("Installed AoE hooks in {}", config_path.display());
    Ok(())
}

/// Remove AoE hooks from Kimi's `~/.kimi/config.toml`.
pub fn uninstall_kimi_hooks(config_path: &Path) -> Result<bool> {
    let home = dirs::home_dir().ok_or_else(|| anyhow::anyhow!("No home directory"))?;
    let config_path = home.join(".kimi").join("config.toml");

    if !config_path.exists() {
        return Ok(false);
    }

    let content = std::fs::read_to_string(&config_path)?;
    let mut config: toml::Value = toml::from_str(&content).unwrap_or_else(|e| {
        tracing::warn!("Failed to parse {}: {}", config_path.display(), e);
        toml::Value::Table(toml::map::Map::new())
    });

    let Some(hooks_arr) = config.get_mut("hooks").and_then(|h| h.as_array_mut()) else {
        return Ok(false);
    };

    let before = hooks_arr.len();
    hooks_arr.retain(|hook| {
        !hook
            .get("command")
            .and_then(|c| c.as_str())
            .is_some_and(is_aoe_hook_command)
    });

    if hooks_arr.len() == before {
        return Ok(false);
    }

    let formatted = toml::to_string_pretty(&config)?;
    std::fs::write(&config_path, formatted)?;
    tracing::info!("Removed AoE hooks from {}", config_path.display());
    Ok(true)
}
```

Update `uninstall_all_hooks()` to include kimi:
```rust
pub fn uninstall_all_hooks() {
    match uninstall_settl_hooks() {
        // ... existing
    }
    if let Some(home) = dirs::home_dir() {
        let hermes_config = home.join(".hermes").join("config.yaml");
        match uninstall_hermes_hooks(&hermes_config) {
            // ... existing
        }
        let kiro_config = home.join(crate::hooks::KIRO_HOOKS_AGENT_FILE);
        match uninstall_kiro_hooks(&kiro_config) {
            // ... existing
        }
        let kimi_config = home.join(".kimi").join("config.toml");
        match uninstall_kimi_hooks(&kimi_config) {
            Ok(modified) => {
                if modified {
                    tracing::info!("Uninstalled Kimi hooks");
                }
            }
            Err(e) => tracing::warn!("Failed to uninstall Kimi hooks: {}", e),
        }
    }
}
```

Add tests for kimi hooks (follow settl test patterns):
- `test_install_kimi_hooks_creates_new_file`
- `test_install_kimi_hooks_idempotent`
- `test_install_kimi_hooks_preserves_user_hooks`
- `test_uninstall_kimi_hooks_removes_aoe_entries`
- `test_uninstall_kimi_hooks_preserves_user_hooks`
- `test_uninstall_kimi_hooks_nonexistent_file`

### 4. `src/session/instance.rs`

In `install_agent_status_hooks()` (after kiro block):
```rust
} else if self.tool == "kimi" && !self.is_sandboxed() {
    if let Some(home) = dirs::home_dir() {
        let config_path = home.join(".kimi").join("config.toml");
        if let Err(e) = crate::hooks::install_kimi_hooks(&config_path) {
            tracing::warn!("Failed to install kimi hooks: {}", e);
        }
    }
}
```

In `status_hook_env_prefix()`:
```rust
let has_hooks = agent.and_then(|a| a.hook_config.as_ref()).is_some()
    || tool == "settl"
    || tool == "hermes"
    || tool == "kiro"
    || tool == "kimi";
```

Update `test_status_hook_env_prefix_includes_hermes`:
```rust
#[test]
fn test_status_hook_env_prefix_includes_hermes() {
    // ... existing hermes, settl, kiro assertions
    assert!(status_hook_env_prefix("abc123", "kimi", crate::agents::get_agent("kimi")).contains("AOE_INSTANCE_ID"));
}
```

### 5. `src/session/container_config.rs`

Add to `AGENT_CONFIG_MOUNTS` (after kiro):
```rust
AgentConfigMount {
    tool_name: "kimi",
    host_rel: ".kimi",
    container_suffix: ".kimi",
    skip_entries: &["sandbox", "sessions", "credentials", "logs"],
    seed_files: &[],
    copy_dirs: &[],
    keychain_credential: None,
    home_seed_files: &[],
    preserve_files: &[],
    clean_files: &[],
},
```

In `build_container_config()`, wire hook installation for sandbox:
```rust
let hermes_hooks = tool == "hermes";
let kiro_hooks = tool == "kiro";
let kimi_hooks = tool == "kimi";
if hermes_hooks || kiro_hooks || kimi_hooks || agent.hook_config.is_some() {
    // ... existing hermes/kiro code
    } else if kimi_hooks {
        let sandbox_dir = home.join(".kimi").join(SANDBOX_SUBDIR);
        let config_file = sandbox_dir.join("config.toml");
        if let Err(e) = crate::hooks::install_kimi_hooks(&config_file) {
            tracing::warn!("Failed to install kimi hooks in sandbox: {}", e);
        }
    }
}
```

## Diagrams

### State Transition Diagram

```
   [Idle] --PreToolUse--> [Running] --Stop--> [Idle]
                                 |
                                 |Notification (permission_prompt)
                                 v
                            [Waiting]
```

Status transitions via hooks:
- **Idle → Running**: PreToolUse hook fires
- **Running → Waiting**: Notification hook fires (matcher: permission_prompt)
- **Waiting → Running**: Approval granted
- **Running → Idle**: Stop hook fires

## Test Cases

### TC-001: Agent Registration

**Priority:** P0
**Type:** Functional

#### Objective
Verify kimi agent is registered and discoverable.

#### Preconditions
- AoE compiled with changes

#### Test Steps
1. Run `aoe agents`
   **Expected:** kimi appears in the agent list
2. Run `aoe add kimi`
   **Expected:** Kimi session starts successfully

#### Post-conditions
- Kimi agent is usable

### TC-002: Status Detection via Hooks

**Priority:** P0
**Type:** Functional

#### Objective
Verify hook-based status detection works.

#### Preconditions
- Kimi installed on host
- AoE hooks installed in ~/.kimi/config.toml

#### Test Steps
1. Start kimi session via AoE
2. Trigger tool use in kimi
   **Expected:** Status changes to Running
3. Tool completes
   **Expected:** Status returns to Idle
4. Trigger action requiring approval
   **Expected:** Status changes to Waiting

#### Post-conditions
- Hook status files created in /tmp/aoe-hooks/

### TC-003: Session Resume

**Priority:** P0
**Type:** Functional

#### Objective
Verify session resume works.

#### Preconditions
- Previous kimi session exists with ID

#### Test Steps
1. Run `aoe add kimi --resume <session-id>`
   **Expected:** Session resumes with prior context
2. Verify conversation history present
   **Expected:** Previous messages visible

#### Post-conditions
- Resume functionality works

### TC-004: Docker Sandbox

**Priority:** P0
**Type:** Integration

#### Objective
Verify kimi works in Docker sandbox.

#### Preconditions
- Docker running

#### Test Steps
1. Create sandboxed kimi session
   **Expected:** Session starts in container
2. Verify status detection
   **Expected:** Status updates correctly via hooks
3. Verify session resume
   **Expected:** Can resume sandboxed session

#### Post-conditions
- Docker sandbox functional

### TC-005: YOLO Mode

**Priority:** P1
**Type:** Functional

#### Objective
Verify YOLO/auto-approve works.

#### Preconditions
- Kimi session created with yolo mode enabled

#### Test Steps
1. Trigger action that normally requires approval
   **Expected:** Action auto-approved, no Waiting status

#### Post-conditions
- YOLO mode functional

### TC-006: Agent Detection

**Priority:** P1
**Type:** Functional

#### Objective
Verify kimi detection works.

#### Preconditions
- Kimi installed on PATH

#### Test Steps
1. Run `aoe agents`
   **Expected:** kimi shows as available
2. Run with kimi not installed
   **Expected:** kimi shows as unavailable with install hint

#### Post-conditions
- Detection functional

### TC-007: Hook Installation Tests

**Priority:** P1
**Type:** Unit

#### Objective
Verify kimi hook install/uninstall functions work.

#### Test Steps
1. Run `cargo test --lib hooks`
   **Expected:** All kimi hook tests pass (install_kimi_hooks, uninstall_kimi_hooks)

#### Post-conditions
- Hook management functional

## Verification Commands

```bash
# Build and test
cargo build
cargo test --lib agents
cargo test --lib status_detection
cargo test --lib hooks
cargo clippy -- -D warnings

# Manual verification
./target/debug/aoe agents
./target/debug/aoe add kimi
```

## Expected Outcome

- [ ] `aoe agents` shows kimi as available agent
- [ ] Kimi sessions launch successfully (host)
- [ ] Status detection works (Running/Waiting/Idle) via hooks
- [ ] Session resume works (`--session` flag)
- [ ] Docker sandbox works
- [ ] All tests pass
- [ ] No clippy warnings
- [ ] Code follows existing patterns exactly

## Rollback Plan

All changes are additive to existing files. To rollback:
1. `git checkout -- src/agents.rs src/tmux/status_detection.rs src/hooks/mod.rs src/session/instance.rs src/session/container_config.rs`
2. No database migrations or breaking changes involved
