# Data & State Architecture

OpenClaw's state management system provides comprehensive handling of configuration, session state, and memory across all components. This document details the data flow, storage patterns, and state synchronization mechanisms.

## Configuration Flow

### Configuration Management Overview

```mermaid
graph TB
    subgraph "Configuration Sources"
        ConfigFiles["Configuration Files"]
        EnvVars["Environment Variables"]
        CLIArgs["CLI Arguments"]
        DefaultConfig["Default Configuration"]
        PluginConfig["Plugin Configuration"]
    end

    subgraph "Configuration Processing"
        ConfigLoader["Configuration Loader"]
        SchemaValidator["Schema Validator (Zod)"]
        MergeConfig["Configuration Merger"]
        SubstituteVars["Variable Substitutor"]
        NormalizePaths["Path Normalizer"]
    end

    subgraph "Configuration Types"
        MainConfig["Main Config"]
        ChannelConfig["Channel Configs"]
        AgentConfig["Agent Configs"]
        ModelConfig["Model Configs"]
        RoutingConfig["Routing Config"]
        HooksConfig["Hooks Config"]
    end

    subgraph "Runtime State"
        RuntimeConfig["Runtime Configuration"]
        ConfigCache["Configuration Cache"]
        HotReloadWatcher["Hot Reload Watcher"]
    end

    ConfigFiles -->|Load| ConfigLoader
    EnvVars -->|Load| ConfigLoader
    CLIArgs -->|Load| ConfigLoader
    DefaultConfig -->|Load| ConfigLoader
    PluginConfig -->|Load| ConfigLoader

    ConfigLoader -->|Validate| SchemaValidator
    SchemaValidator -->|Merge| MergeConfig
    MergeConfig -->|Substitute| SubstituteVars
    SubstituteVars -->|Normalize| NormalizePaths

    NormalizePaths -->|Create| MainConfig
    NormalizePaths -->|Create| ChannelConfig
    NormalizePaths -->|Create| AgentConfig
    NormalizePaths -->|Create| ModelConfig
    NormalizePaths -->|Create| RoutingConfig
    NormalizePaths -->|Create| HooksConfig

    MainConfig -->|Expose| RuntimeConfig
    ChannelConfig -->|Expose| RuntimeConfig
    AgentConfig -->|Expose| RuntimeConfig
    ModelConfig -->|Expose| RuntimeConfig
    RoutingConfig -->|Expose| RuntimeConfig
    HooksConfig -->|Expose| RuntimeConfig

    RuntimeConfig -->|Cached in| ConfigCache
    RuntimeConfig -->|Monitored by| HotReloadWatcher

    HotReloadWatcher -->|Trigger on change| ConfigCache
    ConfigCache -->|Updates| RuntimeConfig

    classDef source fill:#f0f0f0,stroke:#999999,stroke-width:1px
    classDef process fill:#e6f7ff,stroke:#0099ff,stroke-width:2px
    classDef type fill:#ffffff,stroke:#4da6ff,stroke-width:2px
    classDef runtime fill:#ffe6f2,stroke:#ff4da6,stroke-width:2px

    class ConfigFiles,EnvVars,CLIArgs,DefaultConfig,PluginConfig source
    class ConfigLoader,SchemaValidator,MergeConfig,SubstituteVars,NormalizePaths process
    class MainConfig,ChannelConfig,AgentConfig,ModelConfig,RoutingConfig,HooksConfig type
    class RuntimeConfig,ConfigCache,HotReloadWatcher runtime
```

### Configuration Loading Sequence

```mermaid
sequenceDiagram
    participant CLI as "CLI Program"
    participant Loader as "ConfigLoader"
    participant FS as "File System"
    participant Validator as "SchemaValidator"
    participant Merger as "ConfigMerger"
    participant Runtime as "RuntimeConfig"

    CLI->>CLI: Parse CLI arguments
    CLI->>CLI: Build config paths

    Note over CLI, FS: Load Configuration Files
    CLI->>Loader: loadConfig(paths)
    Loader->>FS: Load config.json
    FS-->>Loader: Config content
    Loader->>FS: Load config.local.json (if exists)
    FS-->>Loader: Local overrides
    Loader->>FS: Load channel configs
    FS-->>Loader: Channel configs

    Loader->>Loader: Add environment variables
    Loader->>Loader: Merge CLI arguments

    Note over Loader, Validator: Validate Configuration
    Loader->>Validator: validate(config)
    Validator->>Validator: Apply Zod schemas
    alt Validation Success
        Validator-->>Loader: Validated config
    else Validation Error
        Validator-->>CLI: Error details
        CLI-->>CLI: Exit with error
    end

    Note over Loader, Merger: Process Configuration
    Loader->>Merger: merge(config)
    Merger->>Merger: Apply defaults
    Merger->>Merger: Substitute environment variables
    Merger->>Merger: Normalize file paths
    Merger->>Merger: Parse connection strings
    Merger->>Merger: Validate interdependencies

    Merger-->>Loader: Processed config

    Note over Loader, Runtime: Update Runtime
    Loader->>Runtime: update(config)
    Runtime->>CLI: Config ready

    Note over Runtime, FS: Watch for Changes
    Runtime->>FS: Setup file watchers
    FS-->>Runtime: File change events
    Runtime->>Loader: Reload config
    Loader->>Runtime: Updated config
    Runtime->>CLI: Notify config change
```

### Configuration Schema Structure

```typescript
interface OpenClawConfig {
  // Global Settings
  logLevel: 'trace' | 'debug' | 'info' | 'warn' | 'error'
  timezone: string
  locale: string

  // Gateway Configuration
  gateway: {
    enabled: boolean
    mode: 'local' | 'remote'
    host: string
    port: number
    auth: {
      secret: string
      timeout: number
    }
  }

  // Agent Configuration
  agents: AgentConfig[]

  // Channel Configuration
  channels: {
    [key: string]: ChannelConfig
  }

  // Model Configuration
  models: {
    [key: string]: ModelConfig
  }

  // Routing Configuration
  routing?: RoutingConfig

  // Hook Configuration
  hooks?: {
    [key: string]: HookConfig[]
  }

  // Auto-reply Configuration
  autoReply?: AutoReplyConfig

  // Advanced Configuration
  plugins?: PluginConfig[]
  profiles?: ProfileConfig[]
}
```

## State Management

### State Components

```mermaid
graph TB
    subgraph "Session State"
        SessionData["Session Data"]
        ConversationState["Conversation State"]
        AgentState["Agent Runtime State"]
        ChannelState["Channel Connection State"]
        TempState["Temporary Processing State"]
    end

    subgraph "Persistent Storage"
        SessionStore["~/.openclaw/sessions/*.jsonl"]
        ConfigStore["~/.openclaw/config.json"]
        CredentialStore["~/.openclaw/credentials/"]
        MessageQueue["~/.openclaw/queues/"]
        MemoryStore["~/.openclaw/memory/"]
    end

    subgraph "In-Memory Caches"
        ConfigCache["Configuration Cache"]
        SessionCache["Session Cache"]
        AuthorizeCache["Authorization Cache"]
        ToolResultCache["Tool Result Cache"]
        ProviderCache["Provider Token Cache"]
    end

    subgraph "Synchronization"
        FileWatcher["File System Watcher"]
        StateSync["State Synchronizer"]
        EventEmitter["Event Emitter"]
        StateLock["State Lock Manager"]
    end

    SessionData -->|Saved to| SessionStore
    ConversationState -->|Saved to| SessionStore
    AgentState -->|Saved to| SessionStore
    ChannelState -->|Saved to| SessionStore
    TempState -->|May persist| MessageQueue

    SessionStore -->|Loaded to| SessionCache
    ConfigStore -->|Loaded to| ConfigCache
    CredentialStore -->|Loaded to| AuthorizeCache
    MessageQueue -->|Processed to| ToolResultCache
    MemoryStore -->|Used by| SessionCache

    ConfigCache -->|Invalidated by| FileWatcher
    SessionCache -->|Updated by| StateSync
    SessionCache -->|Events via| EventEmitter
    SessionStore -->|Locked by| StateLock

    TempState -.->|Monitored by| SessionData
    ToolResultCache -.->|Shared with| AgentState
    ProviderCache -.->|Used by| ChannelState

    classDef state fill:#ffffff,stroke:#4da6ff,stroke-width:2px
    classDef storage fill:#e6f7ff,stroke:#0099ff,stroke-width:2px
    classDef cache fill:#fff2e6,stroke:#ff9933,stroke-width:2px
    classDef sync fill:#f0fff0,stroke:#00cc66,stroke-width:1px

    class SessionData,ConversationState,AgentState,ChannelState,TempState state
    class SessionStore,ConfigStore,CredentialStore,MessageQueue,MemoryStore storage
    class ConfigCache,SessionCache,AuthorizeCache,ToolResultCache,ProviderCache cache
    class FileWatcher,StateSync,EventEmitter,StateLock sync
```

### State Management Component Diagram

```mermaid
graph TB
    subgraph "State Management Layer"
        StateController["State Controller<br/>src/state/controller.ts"]
        StateValidator["State Validator<br/>Zod Schemas"]
        StateMigrator["State Migrator<br/>Version Management"]
        StateAccessor["State Accessor<br/>API Interface"]
    end

    subgraph "Session Management"
        SessionManager["Session Manager"]
        ConversationManager["Conversation Manager"]
        ContextManager["Context Manager"]
        AgentStateManager["Agent State Manager"]
    end

    subgraph "Storage Backend"
        JSONLStore["JSONL Store<br/>src/state/jsonl-store.ts"]
        KeyValueStore["Key-Value Store"]
        FileStore["File Store"]
        EncryptedStore["Encrypted Store"]
    end

    subgraph "Data Flow"
        WritePath["Write Path"]
        ReadPath["Read Path"]
        MigratePath["Migration Path"]
        CacheHit["Cache Hit"]
        CacheMiss["Cache Miss"]
    end

    StateController -->|Manages| SessionManager
    StateController -->|Manages| ConversationManager
    StateController -->|Manages| ContextManager
    StateController -->|Manages| AgentStateManager

    StateController -->|Validates| StateValidator
    StateController -->|Manages| StateMigrator
    StateController -->|Provides| StateAccessor

    SessionManager -->|Uses| JSONLStore
    ConversationManager -->|Uses| JSONLStore
    ContextManager -->|Uses| JSONLStore
    AgentStateManager -->|Uses| JSONLStore

    JSONLStore -->|Alternative| KeyValueStore
    JSONLStore -->|Alternative| FileStore
    EncryptedStore -->|Securing| KeyValueStore

    StateAccessor -->|Initiates| WritePath
    StateAccessor -->|Initiates| ReadPath
    StateMigrator -->|Uses| MigratePath

    ReadPath -->|Checks| CacheHit
    CacheHit -->|Returns| StateAccessor
    ReadPath -->|If not in cache| CacheMiss
    CacheMiss -->|Reads from| JSONLStore
    CacheMiss -->|Populates| CacheHit

    WritePath -->|Updates| JSONLStore
    WritePath -->|Invalidates| CacheHit

    classDef manager fill:#ffffff,stroke:#4da6ff,stroke-width:3px
    classDef storage fill:#e6f7ff,stroke:#0099ff,stroke-width:2px
    classDef flow fill:#fff2e6,stroke:#ff9933,stroke-width:1px

    class StateController,StateValidator,StateMigrator,StateAccessor manager
    class SessionManager,ConversationManager,ContextManager,AgentStateManager manager
    class JSONLStore,KeyValueStore,FileStore,EncryptedStore storage
    class WritePath,ReadPath,MigratePath,CacheHit,CacheMiss flow
```

## Data Persistence Patterns

### JSONL Session Storage

```typescript
// Example: Session data stored as NDJSON
interface SessionRecord {
  sessionId: string
  agentId: string
  channelId: string
  timestamp: Date
  type: 'message' | 'response' | 'tool' | 'error' | 'state'
  data: any
}

// Storage pattern:
// ~/.openclaw/sessions/{sessionId}.jsonl
// {"sessionId": "abc123", "type": "message", "data": {...}}
// {"sessionId": "abc123", "type": "response", "data": {...}}
// {"sessionId": "abc123", "type": "tool", "data": {...}}
```

### Memory System Architecture

```mermaid
graph TB
    subgraph "Memory System Components"
        MemoryManager["Memory Manager"
        SemanticMemory["Semantic Memory"
            EpisodicMemory["Episodic Memory"
                WorkingMemory["Working Memory"
                    ConsolidationEngine["Consolidation Engine"
                        VectorIndex["Vector Index"
                            SearchEngine["Semantic Search"

    MemoryManager -->|Manages| SemanticMemory
    MemoryManager -->|Manages| EpisodicMemory
    MemoryManager -->|Manages| WorkingMemory

    WorkingMemory -->|Periodically| ConsolidationEngine
    ConsolidationEngine -->|Creates| EpisodicMemory
    EpisodicMemory -->|Enhances| SemanticMemory

    SemanticMemory -->|Indexed in| VectorIndex
    EpisodicMemory -->|Indexed in| VectorIndex

    VectorIndex -->|Enables| SearchEngine
    SearchEngine -->|Retrieves| MemoryManager

    subgraph "Memory Operations"
        Encode["Encode Experience"]
        Store["Store in Memory"]
        Retrieve["Retrieve Relevant"]
        Consolidate["Consolidate Memories"]
        Forget["Forget Old/Unimportant"]
    end

    MemoryManager -->|Performs| Encode
    Encode -->|Results in| Store
    Store -->|Read by| Retrieve
    Retrieve -->|May trigger| Consolidate
    Consolidate -->|May trigger| Forget

    classDef memory fill:#ffffff,stroke:#4da6ff,stroke-width:2px
    classDef operation fill:#e6f7ff,stroke:#0099ff,stroke-width:2px

    class MemoryManager,SemanticMemory,EpisodicMemory,WorkingMemory,ConsolidationEngine,VectorIndex,SearchEngine memory
    class Encode,Store,Retrieve,Consolidate,Forget operation
}
```

## Key Features

### 1. Hot Configuration Reload
- File watchers on configuration files
- Immediate update of runtime configuration
- Partial reconfiguration (only changed values)
- Validation before applying changes

### 2. State Consistency
- Atomic state updates with locks
- Version management for migrations
- Rollback capability for failed updates
- Event-driven state change notifications

### 3. High Performance
- In-memory caching for frequently accessed data
- Lazy loading of session data
- Batched writes to reduce I/O
- Optimistic updates with eventual consistency

### 4. Security & Privacy
- Encrypted storage for sensitive data
- Credential rotation support
- Secure credential vault
- Temporary state cleanup

## Best Practices

### Configuration Management
1. Use environment variables for secrets
2. Keep configuration files under version control (except secrets)
3. Validate configuration at startup
4. Use profiles for different environments

### State Management
1. Minimize state size with pruning
2. Use appropriate cache TTLs
3. Implement graceful degradation when state is unavailable
4. Monitor state growth and performance

### Session Handling
1. Clean up old sessions regularly
2. Implement session timeouts
3. Archive inactive sessions
4. Use separate sessions for isolation

## Related Documentation

- [System Architecture](./system-architecture.md) - High-level system overview
- [Agent Runtime Architecture](./agent-runtime-architecture.md) - AI agent orchestration
- [Configuration Guide](../configuration/) - Configuration file documentation
- [State Management API](../api/state.md) - State management API reference

## Source Code References

| Component | File |
|-----------|------|
| Configuration Loading | `src/config/config.ts` |
| Configuration Schemas | `src/config/zod-schema.config.ts` |
| State Controller | `src/state/controller.ts` |
| Session Manager | `src/state/session-manager.ts` |
| JSONL Store | `src/state/jsonl-store.ts` |
| Memory System | `src/agents/memory.ts` |
| Context Management | `src/agents/context.ts` |
