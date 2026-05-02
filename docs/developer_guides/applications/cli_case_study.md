---
title: Case Study - Amplifier CLI Application
description: How amplifier-app-cli is built on amplifier-core
---

# Case Study: Amplifier CLI Application

Learn how **amplifier-app-cli** is built on top of amplifier-core. This case study shows real-world patterns for building applications on the Amplifier foundation.

## Overview

amplifier-app-cli is a command-line application that provides:

- **Interactive REPL** - Chat-style interface with the AI
- **Single-shot execution** - `amplifier run "prompt"`
- **Session management** - Resume previous conversations
- **Bundle system** - Pre-configured capability sets
- **Provider switching** - Easy model/provider changes

All built on amplifier-core using the libraries and following best practices.

## Architecture

```
amplifier-app-cli/
├── CLI Layer (Click)
│   ├── Command parsing
│   ├── Argument handling
│   └── Help text
│
├── Application Layer
│   ├── Configuration resolution
│   │   └── Uses: amplifier-foundation
│   ├── Bundle loading
│   │   └── Uses: amplifier-foundation
│   ├── Module resolution
│   │   └── Uses: amplifier-foundation
│   └── Session initialization (session_runner.py)
│
├── Display Layer
│   ├── Rich console formatting
│   ├── Markdown rendering
│   ├── Progress indicators
│   └── Error presentation
│
├── Agent Delegation Layer
│   └── Session spawning (session_spawner.py)
│
└── Session Layer
    └── Uses: amplifier-core
        ├── Session lifecycle
        ├── Prompt execution
        └── Event handling
```

## Key Components

### 1. Configuration Resolution

**Challenge:** Users want profiles, environment-specific configs, and easy provider switching.

**Solution:** Use amplifier-foundation's bundle system with layered composition.

```python
from amplifier_foundation import load_bundle, BundleRegistry

# Load foundation bundle
foundation = await load_bundle("git+https://github.com/microsoft/amplifier-foundation-bundle@main")

# Load user's profile
profile = await load_bundle(f"~/.amplifier/profiles/{profile_name}.md")

# Load provider
provider = await load_bundle(f"~/.amplifier/providers/{provider_name}.yaml")

# Compose: foundation → profile → provider
composed = foundation.compose(profile).compose(provider)

# Convert to mount plan for amplifier-core
mount_plan = composed.to_mount_plan()
```

**Key insight:** Composition happens at app layer, mount plan goes to core.

### 2. Bundle Loading

**Implementation pattern:**

```python
from amplifier_foundation import BundleRegistry
from pathlib import Path

class BundleManager:
    """Manages bundle discovery and loading."""
    
    def __init__(self):
        self.registry = BundleRegistry()
        
        # Register search paths
        self.search_paths = [
            Path(".amplifier/bundles"),              # Project
            Path.home() / ".amplifier" / "bundles",  # User
        ]
    
    async def load_profile(self, name: str) -> Bundle:
        """Load a user profile bundle."""
        for base in self.search_paths:
            profile_path = base / "profiles" / f"{name}.md"
            if profile_path.exists():
                return await load_bundle(str(profile_path))
        
        raise FileNotFoundError(f"Profile '{name}' not found")
    
    async def list_profiles(self) -> list[str]:
        """List available profiles."""
        profiles = set()
        for base in self.search_paths:
            profiles_dir = base / "profiles"
            if profiles_dir.exists():
                profiles.update(p.stem for p in profiles_dir.glob("*.md"))
        return sorted(profiles)
```

### 3. Module Resolution

**Challenge:** Modules need to be downloaded from git, cached locally, and resolved by ID.

**Solution:** Use amplifier-foundation's bundle preparation. `Bundle.prepare()` returns a `PreparedBundle` whose `create_session()` mounts a `module-source-resolver` capability internally before initializing the session.

```python
from amplifier_foundation import load_bundle

bundle = await load_bundle("git+https://github.com/microsoft/amplifier-foundation@main")
prepared = await bundle.prepare()

async with await prepared.create_session() as session:
    response = await session.execute("Hello!")
```

**Module source hints** come from bundle config:

```yaml
tools:
  - module: tool-filesystem
    source: git+https://github.com/microsoft/amplifier-module-tool-filesystem@main
```

### 4. Session Initialization

**Pattern: `session_runner.py`**

```python
from dataclasses import dataclass, field
from pathlib import Path

from amplifier_core import AmplifierSession
from amplifier_foundation.bundle import PreparedBundle


@dataclass
class SessionConfig:
    """All parameters needed to create and initialize a session."""

    # Required configuration
    config: dict
    search_paths: list[Path]
    verbose: bool

    # Session identity
    session_id: str | None = None  # None = generate new UUID
    bundle_name: str = "unknown"

    # Resume mode (if provided, this is a resume)
    initial_transcript: list[dict] | None = None

    # Bundle mode (required)
    prepared_bundle: PreparedBundle | None = None

    # Execution mode
    output_format: str = "text"  # text | json | json-trace


async def create_initialized_session(
    config: SessionConfig,
    console,
):
    """Create and initialize a session via the prepared bundle.

    The prepared bundle's ``create_session()`` method handles wiring
    the module-source-resolver, registering the system prompt factory,
    and initializing the session.
    """
    prepared_bundle = config.prepared_bundle
    assert prepared_bundle is not None

    session = await prepared_bundle.create_session(
        session_id=config.session_id,
        session_cwd=Path.cwd(),
        is_resumed=config.initial_transcript is not None,
    )
    return session
```

### 5. Display Layer

**Challenge:** Present AI responses, tool calls, errors in user-friendly format.

**Solution:** Rich console with custom formatting.

```python
from rich.console import Console
from rich.markdown import Markdown
from rich.panel import Panel

class DisplaySystem:
    """Handles all CLI output formatting."""
    
    def __init__(self):
        self.console = Console()
    
    def show_response(self, text: str):
        """Display AI response as formatted markdown."""
        md = Markdown(text)
        self.console.print(md)
    
    def show_error(self, error: Exception):
        """Display error in red panel."""
        self.console.print(
            Panel(
                str(error),
                title="Error",
                border_style="red",
            )
        )
    
    def show_tool_call(self, tool_name: str, args: dict):
        """Show tool being executed."""
        self.console.print(f"🔧 Using tool: [bold]{tool_name}[/bold]")
```

### 6. Agent Delegation

**Challenge:** Delegate tasks to specialized agents while maintaining conversation context.

**Solution: Session Spawning (session_spawner.py)**

The CLI implements agent delegation by registering `session.spawn` and `session.resume` capabilities on the coordinator. Tool modules (e.g., `tool-task`) call these capabilities, which delegate to `spawn_sub_session()` and `resume_sub_session()` in `session_spawner.py`.

#### CLI-Specific Search Paths for Agents

**First-match-wins resolution** (highest → lowest priority):

1. **Environment Variables** - `AMPLIFIER_AGENT_<NAME>=~/test-agent.md` (for testing)
2. **User Directory** - `~/.amplifier/agents/zen-architect.md` (personal overrides)
3. **Project Directory** - `.amplifier/agents/project-reviewer.md` (project-specific)
4. **Bundle Agents** - Agents bundled with loaded bundles

**Environment variable format**: `AMPLIFIER_AGENT_<NAME>` (uppercase, dashes → underscores)

```bash
# Testing agent changes
export AMPLIFIER_AGENT_ZEN_ARCHITECT=~/test-zen.md
amplifier run "design system"  # Uses test version
```

#### Session Spawning Capability

`register_session_spawning()` exposes spawn/resume to tools via the coordinator:

```python
# In session_runner.py
def register_session_spawning(session: AmplifierSession) -> None:
    from .session_spawner import resume_sub_session, spawn_sub_session

    async def spawn_capability(
        agent_name: str,
        instruction: str,
        parent_session: AmplifierSession,
        agent_configs: dict[str, dict],
        sub_session_id: str | None = None,
        tool_inheritance: dict[str, list[str]] | None = None,
        hook_inheritance: dict[str, list[str]] | None = None,
        orchestrator_config: dict | None = None,
        parent_messages: list[dict] | None = None,
        provider_preferences: list | None = None,
        self_delegation_depth: int = 0,
        session_metadata: dict | None = None,
        use_subprocess: bool = False,
    ) -> dict:
        return await spawn_sub_session(
            agent_name=agent_name,
            instruction=instruction,
            parent_session=parent_session,
            agent_configs=agent_configs,
            # ... other args
        )

    async def resume_capability(sub_session_id: str, instruction: str) -> dict:
        return await resume_sub_session(sub_session_id=sub_session_id, instruction=instruction)

    session.coordinator.register_capability("session.spawn", spawn_capability)
    session.coordinator.register_capability("session.resume", resume_capability)
```

#### Spawn Tool Policy

Bundles control which tools spawned agents inherit:

```yaml
# In bundle.md
tools:
  - module: tool-task      # Parent can delegate
  - module: tool-filesystem
  - module: tool-bash

spawn:
  exclude_tools: [tool-task]  # But agents can't delegate further
  # OR
  tools: [tool-a, tool-b]     # Agents get ONLY these tools
```

**Default behavior:** If no `spawn` section, agents inherit all parent tools.

#### Multi-Turn Sub-Session Resumption

Sub-sessions support multi-turn conversations through automatic state persistence:

```python
# After sub-session execution, before cleanup
from amplifier_app_cli.session_store import SessionStore

# Capture current state
context = child_session.coordinator.get("context")
transcript = await context.get_messages() if context else []

metadata = {
    "session_id": sub_session_id,
    "parent_id": parent_session.session_id,
    "agent_name": agent_name,
    "created": datetime.now(UTC).isoformat(),
    "config": merged_config,
    "agent_overlay": agent_config,
}

# Persist to storage
store = SessionStore()  # ~/.amplifier/projects/{project}/sessions/
store.save(sub_session_id, transcript, metadata)
```

### 7. Session Storage

**Pattern: Project-scoped sessions**

```python
class SessionStore:
    """Manages session persistence."""
    
    def get_session_dir(self, session_id: str) -> Path:
        """Get session directory."""
        project_slug = self._get_project_slug(os.getcwd())
        return Path.home() / ".amplifier" / "projects" / project_slug / "sessions" / session_id
    
    def _get_project_slug(self, cwd: str) -> str:
        """Convert working directory to project slug."""
        # /home/user/projects/my-app → -home-user-projects-my-app
        return cwd.replace(os.sep, "-")
    
    def save(self, session_id: str, transcript: list, metadata: dict):
        """Save session to disk."""
        session_dir = self.get_session_dir(session_id)
        session_dir.mkdir(parents=True, exist_ok=True)
        
        # Save transcript
        with open(session_dir / "transcript.jsonl", "w") as f:
            for msg in transcript:
                f.write(json.dumps(sanitize_message(msg)) + "\n")
        
        # Save metadata
        with open(session_dir / "metadata.json", "w") as f:
            json.dump(metadata, f, indent=2)
    
    def load(self, session_id: str) -> tuple[list, dict]:
        """Load session from disk."""
        session_dir = self.get_session_dir(session_id)
        
        # Load transcript
        transcript = []
        with open(session_dir / "transcript.jsonl") as f:
            for line in f:
                transcript.append(json.loads(line))
        
        # Load metadata
        with open(session_dir / "metadata.json") as f:
            metadata = json.load(f)
        
        return transcript, metadata
```

## Command Structure

The CLI uses Click for command structure:

```python
import click

@click.group(cls=AmplifierGroup)
def cli():
    """Amplifier CLI - Your AI coding assistant."""
    pass

@cli.command()
@click.argument("prompt", required=False)
def run(prompt: str):
    """Run a single prompt."""
    asyncio.run(run_async(prompt))

@cli.group()
def session():
    """Session management commands."""
    pass

@session.command()
def list():
    """List all sessions."""
    # Implementation

@session.command()
@click.argument("session_id")
def resume(session_id: str):
    """Resume a session."""
    # Implementation
```

## Error Handling

**Pattern: LLM Error Filtering**

The CLI filters duplicate LLM errors from console output:

```python
from logging import Filter

class LLMErrorLogFilter(Filter):
    """Suppress duplicate LLM error lines from console output."""
    
    def filter(self, record):
        # Suppress provider-level and session-level LLM errors
        # (CLI renders these as Rich panels instead)
        if "Anthropic API error" in record.getMessage():
            return False
        if "Execution failed" in record.getMessage():
            return False
        return True

# Attach to console handler
for handler in logging.root.handlers:
    if isinstance(handler, logging.StreamHandler):
        handler.addFilter(LLMErrorLogFilter())
```

**Pattern: Rich Error Display**

```python
from amplifier_core.llm_errors import LLMError

try:
    response = await session.execute(prompt)
except LLMError as e:
    # Display as formatted panel
    console.print(
        Panel(
            f"[red]{e.message}[/red]\n\n"
            f"Provider: {e.provider}\n"
            f"Error type: {e.error_type}",
            title="LLM Error",
            border_style="red",
        )
    )
```

## Key Takeaways

### 1. Thin Application Layer

The CLI is ~2000 lines of application code. Most functionality comes from:
- **amplifier-foundation** - Bundle loading, composition, agent resolution
- **amplifier-core** - Session management, module lifecycle
- **Rich** - Display formatting

### 2. Clear Separation of Concerns

```
CLI commands → Configuration → Session → Core
     ↓             ↓              ↓         ↓
   Click     amplifier-     amplifier-  Modules
             foundation        core
```

### 3. Composition Over Configuration

Users don't edit YAML files - they compose bundles:

```bash
# Not this
vim ~/.amplifier/config.yaml

# This
amplifier provider add anthropic
# → Composes provider bundle with foundation
```

### 4. Protocol Boundaries

The app speaks **mount plans** to core:

```python
# App layer: compose bundles
bundle = foundation.compose(profile).compose(provider)

# Protocol: mount plan
mount_plan = bundle.to_mount_plan()

# Core: accepts mount plan
session = AmplifierSession(config=mount_plan)
```

## Implementation Checklist

When building your own application:

- [ ] **Configuration**: How do users select providers, tools, profiles?
- [ ] **Bundle Loading**: Where do bundles live? How are they discovered?
- [ ] **Module Resolution**: How are module sources resolved and cached?
- [ ] **Session Creation**: How do you create and initialize sessions?
- [ ] **Display**: How are responses, errors, and progress shown?
- [ ] **Session Storage**: Where do sessions persist? How do users resume?
- [ ] **Error Handling**: How are LLM errors, validation errors presented?

## Next Steps

- **Study the code**: [amplifier-app-cli on GitHub](https://github.com/microsoft/amplifier-app-cli)
- **Build your app**: Use patterns from this case study
- **Module development**: Create custom providers, tools, hooks

## Resources

- **amplifier-core** - Session and module system
- **amplifier-foundation** - Bundle composition layer
- **amplifier-app-cli** - Full source code
