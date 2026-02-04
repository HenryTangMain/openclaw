# Plugin Architecture

OpenClaw's plugin system enables extensibility without modifying the core codebase. This document details the plugin architecture, lifecycle, and extension mechanisms.

## Overview

The plugin system provides:
- **Dynamic Loading**: Plugins are discovered and loaded at runtime
- **Runtime Isolation**: Plugins run in isolated contexts with controlled APIs
- **Extensibility**: Multiple extension points (channels, tools, commands, hooks, services)
- **Dependency Management**: Automatic dependency resolution within plugins

## Plugin System Components

```mermaid
graph TB
    %% Core Components
    PluginRegistry["Plugin Registry<br/>src/plugins/registry.ts"]
    PluginLoader["Plugin Loader<br/>src/plugins/loader.ts"]
    PluginRuntime["Plugin Runtime<br/>src/plugins/runtime/"]
    ManifestValidator["Manifest Validator<br/>src/plugins/types.ts"]

    %% Extension Points
    ChannelExtensions["Channel Extensions<br/>New messaging platforms"]
    ToolExtensions["Tool Extensions<br/>Agent capabilities"]
    CommandExtensions["Command Extensions<br/>CLI commands"]
    HookExtensions["Hook Extensions<br/>Event callbacks"]
    ServiceExtensions["Service Extensions<br/>Shared utilities"]

    %% External Components
    ExtensionsDir["extensions/ Directory"]
    CoreSystem["Core System Components"]
    AgentRuntime["Agent Runtime"]
    ChannelManager["Channel Manager"]
    CLI["CLI Application"]

    Discovery[("Plugin Discovery")]
    Loading[("Plugin Loading")]
    Registration[("Extension Registration")]
    Runtime[("Runtime Execution")]

    ExtensionsDir -->|Contains| Discovery
    Discovery -->|Found plugins| PluginLoader
    PluginLoader -->|Loads| Loading
    Loading -->|Validated manifest| ManifestValidator
    ManifestValidator -->|Extensions| Registration
    Registration -->|Registered| PluginRegistry
    PluginRegistry -->|Manages| Runtime
    Runtime -->|Executes| PluginRuntime

    PluginRuntime -->|Provides| ChannelExtensions
    PluginRuntime -->|Provides| ToolExtensions
    PluginRuntime -->|Provides| CommandExtensions
    PluginRuntime -->|Provides| HookExtensions
    PluginRuntime -->|Provides| ServiceExtensions

    ChannelExtensions -->|Extends| ChannelManager
    ToolExtensions -->|Extends| AgentRuntime
    CommandExtensions -->|Extends| CLI
    HookExtensions -->|Hooks into| CoreSystem
    ServiceExtensions -->|Provides| CoreSystem

    PluginRegistry -.->|API surface| CoreSystem

  %%  Styling
    classDef component fill:#ffffff,stroke:#4da6ff,stroke-width:2px
    classDef extension fill:#e6f7ff,stroke:#0099ff,stroke-width:2px
    classDef core fill:#ffe6f2,stroke:#ff4da6,stroke-width:2px
    classDef external fill:#f0f0f0,stroke:#999999,stroke-width:1px
    classDef process fill:#fff2e6,stroke:#ff9933,stroke-width:1px

    class PluginRegistry,PluginLoader,PluginRuntime,ManifestValidator component
    class ChannelExtensions,ToolExtensions,CommandExtensions,HookExtensions,ServiceExtensions extension
    class CoreSystem,AgentRuntime,ChannelManager,CLI core
    class ExtensionsDir external
    class Discovery,Loading,Registration,Runtime process
```

## Plugin Lifecycle

### Plugin Registration Sequence

```mermaid
sequenceDiagram
    participant FileSystem as "File System"
    participant Loader as "PluginLoader"
    participant Validator as "ManifestValidator"
    participant Registry as "PluginRegistry"
    participant Runtime as "PluginRuntime"
    participant CoreSystem as "Core System"

    FileSystem->>Loader: Discover plugin directories
    Loader->>FileSystem: Load package.json
    FileSystem-->>Loader: package.json contents

    Note over Loader: Check plugin requirements<br/>Node.js version<br/>dependencies

    Loader->>FileSystem: Load plugin manifest<br/>clawd.plugin.json
    FileSystem-->>Loader: manifest data

    Loader->>Validator: Validate manifest schema
    Validator-->>Loader: Validation result

    alt Validation Success
        Loader->>Loader: Determine extension types<br/>channels, tools, commands, hooks

        Loader->>Runtime: Create isolated context
        Runtime-->>Loader: Context created

        Loader->>Runtime: Load plugin entry point
        Runtime-->>Loader: Plugin module loaded

        Loader->>Registry: Register extensions

        loop For each extension type
            Registry->>CoreSystem: Register extension
            CoreSystem-->>Registry: Extension registered
        end

        Registry-->>Loader: Plugin fully registered
    else Validation Failed
        Loader-->>FileSystem: Log error, skip plugin
    end
```

### Plugin Runtime Dependency Injection

```mermaid
graph LR
    subgraph "Runtime Isolation Layer"
        PluginCode["Plugin Code"]
        RuntimeContext["Runtime Context"]
        APIProxy["API Proxy/Bridge"]
    end

    subgraph "Core System APIs"
        ChannelAPI["Channel API"]
        AgentAPI["Agent API"]
        ToolAPI["Tool API"]
        ConfigAPI["Config API"]
        LoggerAPI["Logger API"]
    end

    PluginCode -->|Uses| RuntimeContext
    RuntimeContext -->|Mediates| APIProxy
    APIProxy -->|Exposes| ChannelAPI
    APIProxy -->|Exposes| AgentAPI
    APIProxy -->|Exposes| ToolAPI
    APIProxy -->|Exposes| ConfigAPI
    APIProxy -->|Exposes| LoggerAPI

    %% API Restrictions
    APIProxy -.->|Validates calls| ChannelAPI
    APIProxy -.->|Validates calls| AgentAPI
    APIProxy -.->|Validates calls| ToolAPI
    APIProxy -.->|Validates calls| ConfigAPI
    APIProxy -.->|Validates calls| LoggerAPI
```

## Extension Points

### 1. Channel Extensions

Add support for new messaging platforms.

```typescript
// Example: MyChannel extension
export default {
  name: 'mychannel',
  version: '1.0.0',
  channels: [{
    name: 'mychannel',
    // Channel implementation
    async sendMessage(channelId, message) {
      // Implementation
    },
    async start() {
      // Connection logic
    }
  }]
}
```

**Registration Flow:**
```
Plugin Discovery -> Manifest Parsing -> Channel Registration ->
Runtime Setup -> Channel Initialization -> Message Routing Integration
```

### 2. Tool Extensions

Add new capabilities to AI agents.

```typescript
// Example: Custom Tool
export default {
  name: 'mytool',
  version: '1.0.0',
  tools: [{
    name: 'my_custom_tool',
    description: 'Does something useful',
    parameters: z.object({
      input: z.string()
    }),
    handler: async (args) => {
      // Tool implementation
      return { result: 'success' };
    }
  }]
}
```

**Registration Flow:**
```
Plugin Discovery -> Manifest Parsing -> Tool Registration ->
Agent Runtime Integration -> Capability Registration -> Execution Context Setup
```

### 3. Command Extensions

Add new CLI commands.

```typescript
// Example: Custom Command
export default {
  name: 'mycommand',
  version: '1.0.0',
  commands: [{
    name: 'my-custom-command',
    description: 'My custom functionality',
    handler: async (args) => {
      // Command implementation
    }
  }]
}
```

### 4. Hook Extensions

Add event-driven functionality.

```typescript
// Example: Custom Hooks
export default {
  name: 'myhooks',
  version: '1.0.0',
  hooks: {
    'message:received': async (message) => {
      // Pre-process message
    },
    'agent:response': async (response) => {
      // Post-process response
    }
  }
}
```

### 5. Service Extensions

Provide shared utilities.

```typescript
// Example: Shared Service
export default {
  name: 'myservice',
  version: '1.0.0',
  services: {
    myUtility: {
      helperFunction: () => { /* ... */ }
    }
  }
}
```

## Plugin Manifest Structure

Each plugin requires a `clawd.plugin.json` manifest:

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "extensions": {
    "channels": ["./channels/*.js"],
    "tools": ["./tools/*.js"],
    "commands": ["./commands/*.js"],
    "hooks": ["./hooks/*.js"]
  },
  "dependencies": {
    "@openclaw/plugin-sdk": "^3.0.0"
  },
  "engines": {
    "node": ">=22.0.0"
  }
}
```

## Runtime Isolation

Plugins run in isolated contexts with:
- **Limited API Surface**: Only documented APIs are exposed
- **Permission Checking**: All operations are validated
- **Error Boundaries**: Plugin errors don't crash the core system
- **Resource Limits**: CPU and memory usage can be capped

## Plugin Development Workflow

1. **Setup**: Create plugin directory with `package.json`
2. **Manifest**: Define `clawd.plugin.json` with extensions
3. **Implementation**: Build extension code following SDK patterns
4. **Testing**: Test with `openclaw plugin test`
5. **Distribution**: Publish to npm or distribute as package

## Extension Registry

The plugin registry maintains:
- **Active Plugins**: Currently loaded plugins
- **Extension Mappings**: What each plugin provides
- **API Bindings**: Runtime API access control
- **Dependency Graph**: Inter-plugin dependencies
- **Lifecycle State**: Loading, active, error states

## Best Practices

1. **Minimal Dependencies**: Keep plugin dependencies lightweight
2. **Error Handling**: Always handle errors gracefully
3. **Type Safety**: Use TypeScript and proper types
4. **Documentation**: Document all extensions clearly
5. **Testing**: Include comprehensive tests
6. **Isolation**: Don't depend on internal APIs
7. **Versioning**: Follow semantic versioning

## Security Considerations

- **Code Review**: All plugins should be reviewed
- **Signature Verification**: Consider plugin signing
- **Sandbox Limitations**: Runtime isolation is not foolproof
- **API Auditing**: Log all plugin API calls
- **Permission Model**: Plugins request specific permissions

## Related Documentation

- [System Architecture](./system-architecture.md) - High-level system overview
- [Channel Provider Architecture](./channel-provider-architecture.md) - Messaging integrations
- [Plugin SDK Documentation](../channels/plugins.md) - Building plugins
- [Source Code](../src/plugins/) - Plugin system implementation

## Source Code References

| Component | File |
|-----------|------|
| Plugin Registry | `src/plugins/registry.ts` |
| Plugin Loader | `src/plugins/loader.ts` |
| Manifest Types | `src/plugins/types.ts` |
| Runtime Context | `src/plugins/runtime/context.ts` |
| SDK Interface | `src/plugin-sdk/index.ts` |
