# Agent Runtime Architecture

OpenClaw's agent runtime provides a sophisticated environment for AI agent execution, built on the Pi AI framework and enhanced with OpenClaw-specific features.

## Core Components

```mermaid
graph TB
    subgraph "Agent Runtime Core"
        AgentManager["Agent Manager<br/>src/agents/agents.ts"]
        SessionManager["Session Manager<br/>src/agents/context.ts"]
        IdentityManager["Identity Manager<br/>src/agents/identity.ts"]
    end

    subgraph "Execution Engine"
        PiRunner["Pi Embedded Runner<br/>src/agents/pi-embedded-runner.ts"]
        PiSubscriber["Pi Event Subscriber<br/>src/agents/pi-embedded-subscribe.ts"]
        ToolExecutor["Tool Execution<br/>src/agents/tool-policy.ts"]
        ContextManager["Context Manager<br/>src/agents/context.ts"]
    end

    subgraph "Tool System"
        ToolRegistry["Tool Registry<br/>src/agent-tools/*.ts"]
        PolicyEngine["Policy Engine<br/>src/agents/tool-policy.ts"]
        BashExecutor["Bash Tool Executor<br/>src/agents/bash-tools.exec.ts"]
        FileTools["File System Tools"]
        APITools["API Integration Tools"]
        PluginTools["Plugin-Provided Tools"]
    end

    subgraph "Context & Memory"
        ContextStore["Context Store<br/>~/.openclaw/sessions/"]
        MemorySystem["Memory System<br/>src/agents/memory.ts"]
        Pruner["Context Pruner<br/>src/agents/context-pruner.ts"]
        HistoryManager["Conversation History"]
    end

    subgraph "Authentication"
        AuthManager["Auth Manager<br/>src/auth/"]
        CredentialStore["Credential Store"]
        TokenManager["Token Manager"]
        MultiAccount["Multi-Account Support"]
    end

    AgentManager -->|Manages| SessionManager
    AgentManager -->|Manages| IdentityManager
    AgentManager -->|Executes| PiRunner

    SessionManager -->|Maintains| PiSubscriber
    SessionManager -->|Provides| ContextManager

    PiRunner -->|Uses| ToolExecutor
    PiRunner -->|Manages| ContextManager
    PiSubscriber -->|Fetches| ContextManager

    ToolExecutor -->|Registers| ToolRegistry
    ToolExecutor -->|Validates| PolicyEngine
    ToolExecutor -->|Executes| BashExecutor

    ToolRegistry -->|Contains| FileTools
    ToolRegistry -->|Contains| APITools
    ToolRegistry -->|Contains| PluginTools

    ContextManager -->|Reads/Writes| ContextStore
    ContextManager -->|Integrates| MemorySystem
    ContextManager -->|Manages| HistoryManager
    MemorySystem -- > |Optimizes | Pruner

    AgentManager -->|Authenticates| AuthManager
    AuthManager -->|Manages| CredentialStore
    AuthManager -->|Manages| TokenManager
    AuthManager -->|Supports| MultiAccount

    %% External interactions
    AgentManager -->|Communicates with| ChannelManager
    AgentManager -->|Processes via| AIProviders

    classDef core fill:#ffffff,stroke:#4da6ff,stroke-width:3px
    classDef engine fill:#e6f7ff,stroke:#0099ff,stroke-width:2px
    classDef tools fill:#fff2e6,stroke:#ff9933,stroke-width:2px
    classDef context fill:#f0fff0,stroke:#00cc66,stroke-width:2px
    classDef auth fill:#ffe6f2,stroke:#ff4da6,stroke-width:2px

    class AgentManager,SessionManager,IdentityManager core
    class PiRunner,PiSubscriber,ToolExecutor,ContextManager engine
    class ToolRegistry,PolicyEngine,BashExecutor,FileTools,APITools,PluginTools tools
    class ContextStore,MemorySystem,Pruner,HistoryManager context
    class AuthManager,CredentialStore,TokenManager,MultiAccount auth
    ChannelManager core
    AIProviders core
```

## Session Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Initializing: Start Agent

    state Initializing {
        [*] --> LoadingConfig: Load Configuration
        LoadingConfig --> SettingUpIdentity: Create Identity
        SettingUpIdentity --> InitializingTools: Initialize Tools
        InitializingTools --> LoadingContext: Load Context
        LoadingContext --> [*]
    }

    Initializing --> Active: Initialization Complete

    state Active {
        [*] --> Processing: New Message

        Processing --> AwaitingModel: Send to AI Provider
        AwaitingModel --> RunningTools: AI Requests Tool
        RunningTools --> Processing: Tool Result
        RunningTools --> Processing: Tool Error
        AwaitingModel --> Generating: AI Generating Response
        Generating --> Streaming: Stream Response
        Streaming --> Processing: Continue Generation
        Streaming --> Complete: Response Complete

        Complete --> Idle: Wait for Next Message
        Processing --> Error: Processing Error
        Error --> Idle: Log Error
    }

    Active --> Suspended: User Command
    Suspended --> Active: Resume
    Suspended --> Idle: Timeout

    Idle --> Active: New Message
    Idle --> Suspended: Manual Suspend

    Active --> Terminating: Stop Command
    Suspended --> Terminating: Stop Command
    Idle --> Terminating: Stop Command

    state Terminating {
        [*] --> SavingContext: Save Session State
        SavingContext --> CleanupTools: Cleanup Tools
        CleanupTools --> ClosingConnections: Close Connections
        ClosingConnections --> [*]
    }

    Terminating --> [*]: Session Ended

    note right of Active : Heartbeat active
    note right of Idle : Timeout monitoring
    note right of Suspended : Paused execution
```

## Agent Execution Flow

### Message Processing Sequence

```mermaid
sequenceDiagram
    participant Channel as "Channel Adapter"
    participant Router as "Message Router"
    participant AgentManager as "Agent Manager"
    participant Context as "Context Manager"
    participant PiRunner as "Pi Runner"
    participant AIProvider as "AI Provider"
    participant ToolRegistry as "Tool Registry"

    Note over Channel, AIProvider: Incoming Message Processing
    Channel->>Router: Incoming message
    Router->>AgentManager: Route to agent

    AgentManager->>Context: Load/create session
    Context->>Context: Load conversation history
    Context->>Context: Apply context pruning
    Context-->>AgentManager: Session context

    AgentManager->>PiRunner: Start execution
    PiRunner->>Context: Get context
    Context-->>PiRunner: Full context

    PiRunner->>AIProvider: Send context
    AIProvider->>AIProvider: Process with model

    Note over PiRunner, AIProvider: Tool Request
    AIProvider-->>PiRunner: Tool call request
    PiRunner->>ToolRegistry: Find tool
    ToolRegistry-->>PiRunner: Tool implementation

    PiRunner->>ToolRegistry: Execute tool
    Note over ToolRegistry: Policy check<br/>Parameter validation<br/>Execution<br/>Result formatting
    ToolRegistry-->>PiRunner: Tool result

    PiRunner->>AIProvider: Send tool result
    AIProvider->>AIProvider: Continue generation

    AIProvider-->>PiRunner: Final response
    PiRunner->>Context: Update context
    Context->>Context: Save conversation

    PiRunner-->>AgentManager: Response ready
    AgentManager->>Router: Send response
    Router->>Channel: Format for platform
    Channel->>Channel: Send to platform

    Note over PiRunner, Context: Async tool execution allows<br/>parallel processing
```

## Tool Execution System

### Tool Architecture

```mermaid
graph TB
    subgraph "Tool Layer"
        ToolDefinition["Tool Definition<br/>Name, Description, Schema"]
        ToolHandler["Tool Handler<br/>Implementation"]
        ToolValidator["Parameter Validator<br/>Zod Schema"]
        ToolRegistry["Tool Registry<br/>Registration & Discovery"]
    end

    subgraph "Policy Layer"
        PolicyManager["Policy Manager"]
        ApprovalSystem["Approval System<br/>User Confirmation"]
        RateLimiter["Rate Limiter"]
        ExecutionGuard["Execution Guard<br/>Safety Checks"]
    end

    subgraph "Execution Layer"
        BashTool["Bash Tool<br/>Command Execution"]
        FileTool["File System Tool"]
        APITool["API Integration Tool"]
        PluginTool["Plugin Tools"]
        CustomTool["Custom Tools"]
    end

    subgraph "Result Processing"
        Formatter["Result Formatter"]
        Truncator["Text Truncator"]
        ErrorHandler["Error Handler"]
        RetryLogic["Retry Logic"]
    end

    ToolDefinition -->|Defines| ToolHandler
    ToolDefinition -->|Uses| ToolValidator
    ToolHandler -->|Registers with| ToolRegistry

    ToolRegistry -->|Lookup| PolicyManager
    PolicyManager -->|Checks| ApprovalSystem
    PolicyManager -->|Enforces| RateLimiter
    RateLimiter -->|Validates| ExecutionGuard

    ExecutionGuard -->|Sends to| BashTool
    ExecutionGuard -->|Sends to| FileTool
    ExecutionGuard -->|Sends to| APITool
    ExecutionGuard -->|Sends to| PluginTool
    ExecutionGuard -->|Sends to| CustomTool

    BashTool -->|Returns| Formatter
    FileTool -->|Returns| Formatter
    APITool -->|Returns| Formatter
    PluginTool -->|Returns| Formatter
    CustomTool -->|Returns| Formatter

    Formatter -->|Formats| Truncator
    Truncator -->|Optionally| ErrorHandler
    ErrorHandler -->|May trigger| RetryLogic

    RetryLogic -->|Retries| ExecutionGuard

    classDef tool fill:#ffffff,stroke:#4da6ff,stroke-width:2px
    classDef policy fill:#fff2e6,stroke:#ff9933,stroke-width:2px
    classDef exec fill:#e6f7ff,stroke:#0099ff,stroke-width:2px
    classDef result fill:#f0fff0,stroke:#00cc66,stroke-width:1px

    class ToolDefinition,ToolHandler,ToolValidator,ToolRegistry tool
    class PolicyManager,ApprovalSystem,RateLimiter,ExecutionGuard policy
    class BashTool,FileTool,APITool,PluginTool,CustomTool exec
    class Formatter,Truncator,ErrorHandler,RetryLogic result
```

### Tool Sequence Diagram

```mermaid
sequenceDiagram
    participant Agent as "Agent Runtime"
    participant Policy as "Policy Engine"
    participant Registry as "Tool Registry"
    participant Validator as "Parameter Validator"
    participant Guard as "Execution Guard"
    participant Tool as "Tool Implementation"
    participant System as "System/External API"

    Agent->>Policy: Tool call request
    Policy->>Policy: Check approval policy
    alt User approval required
        Policy->>Agent: Request approval
        Agent->>Policy: User approved
    end
    Policy->>Policy: Check rate limits
    Policy->>Policy: Global safety checks
    Policy-->>Registry: Move request

    Registry->>Registry: Lookup tool
    Registry-->>Validator: Tool found

    Validator->>Validator: Validate parameters
    alt Validation failed
        Validator-->>Agent: Validation error
    else Validation passed
        Validator-->>Guard: Validated params
    end

    Guard->>Guard: Pre-execution checks
    Guard->>Tool: Execute with params

    Tool->>System: Make API call
    System-->>Tool: Return result
    Tool->>Tool: Process result
    Tool-->>Guard: Raw result

    Guard->>Guard: Validate result
    Guard->>Guard: Format result
    Guard->>Guard: Truncate if needed
    Guard-->>Agent: Formatted result

    Note over Agent, System: Async execution allows<br/>agent to continue<br/>while tool runs
```

## Memory and Context Management

### Memory Integration

```mermaid
graph TB
    subgraph "Memory System"
        MemoryManager["Memory Manager"]
        VectorStore["Vector Store<br/>Semantic Search"]
        KnowledgeBase["Knowledge Base"]
        SessionMemory["Session Memory"]
        LongTermMemory["Long-term Memory"]
    end

    subgraph "Context Management"
        ContextStore["Context Store"]
        ConversationHistory["Conversation History"]
        SystemContext["System Context"]
        RoleContext["Role-based Context"]
    end

    subgraph "Optimization"
        ContextPruner["Context Pruner"]
        TokenOptimizer["Token Optimizer"]
        RelevanceScorer["Relevance Scoring"]
        CompressionEngine["Text Compression"]
    end

    MemoryManager -->|Manages| VectorStore
    MemoryManager -->|Manages| SessionMemory
    MemoryManager -->|Manages| LongTermMemory
    VectorStore -->|Stores| KnowledgeBase

    ContextStore -->|Contains| ConversationHistory
    ContextStore -->|Contains| SystemContext
    ContextStore -->|Contains| RoleContext

    ContextPruner -->|Operates on| ConversationHistory
    ContextPruner -->|Operates on| SessionMemory
    TokenOptimizer -->|Optimizes| ContextStore
    RelevanceScorer -->|Scores| ConversationHistory
    CompressionEngine -->|Compresses| ContextStore

    MemoryManager -->|Searches| VectorStore
    MemoryManager -->|Retrieves| LongTermMemory

    %% Connections
    ContextStore -.->|Accessed by| MemoryManager
    ConversationHistory -.->|Scored by| RelevanceScorer
    SessionMemory -.->|Archived to| LongTermMemory

    classDef memory fill:#ffffff,stroke:#4da6ff,stroke-width:3px
    classDef context fill:#e6f7ff,stroke:#0099ff,stroke-width:2px
    classDef optimize fill:#fff2e6,stroke:#ff9933,stroke-width:2px

    class MemoryManager,VectorStore,KnowledgeBase,SessionMemory,LongTermMemory memory
    class ContextStore,ConversationHistory,SystemContext,RoleContext context
    class ContextPruner,TokenOptimizer,RelevanceScorer,CompressionEngine optimize
```

### Context Pruning Strategy

```typescript
interface ContextPruningStrategy {
  // Token-based limits
  maxTokens: number
  targetTokens: number
  bufferTokens: number

  // Message history limits
  maxMessages: number
  keepRecent: number
  keepSystem: boolean

  // Semantic filters
  relevanceThreshold: number
  importanceDecay: number

  // Selective retention
  keepUserMessages: boolean
  keepToolResults: boolean
  truncateLongMessages: boolean
}
```

## Multi-Account Support

### Authentication Architecture

```mermaid
graph TB
    subgraph "Account Management"
        AccountManager["Account Manager"]
        AccountRegistry["Account Registry"]
        ProfileStore["Profile Store"]
    end

    subgraph "Authentication"
        AuthHandlers["Auth Handlers"]
        TokenManager["Token Manager"]
        OAuthFlows["OAuth 2.0 Flows"]
        APITokens["API Token Storage"]
    end

    subgraph "Per-Agent Identity"
        AgentIdentity["Agent Identity"]
        PersonaManager["Persona Manager"]
        ChannelIdentity["Channel-specific Identity"]
    end

    subgraph "Credentials"
        CredentialVault["Credential Vault"]
        Encryption["Encryption Layer"]
        RotationManager["Key Rotation"]
    end

    AccountManager -->|Manages| AccountRegistry
    AccountManager -->|Stores| ProfileStore
    AccountRegistry -->|Lookups| ProfileStore

    AuthHandlers -->|Uses| TokenManager
    AuthHandlers -->|Implements| OAuthFlows
    OAuthFlows -->|Manages| APITokens
    TokenManager -->|Rotates| RotationManager

    AgentIdentity -->|Associated with| AccountRegistry
    AgentIdentity -->|Uses| PersonaManager
    AgentIdentity -->|Has| ChannelIdentity

    CredentialVault -->|Secures| APITokens
    CredentialVault -->|Protected by| Encryption
    RotationManager -->|Updates| CredentialVault

    AccountRegistry -.->|Used by| AgentIdentity
    TokenManager -.->|Referenced by| ChannelIdentity

    classDef account fill:#ffffff,stroke:#4da6ff,stroke-width:3px
    classDef auth fill:#e6f7ff,stroke:#0099ff,stroke-width:2px
    classDef identity fill:#fff2e6,stroke:#ff9933,stroke-width:2px
    classDef creds fill:#f0fff0,stroke:#00cc66,stroke-width:2px

    class AccountManager,AccountRegistry,ProfileStore account
    class AuthHandlers,TokenManager,OAuthFlows,APITokens auth
    class AgentIdentity,PersonaManager,ChannelIdentity identity
    class CredentialVault,Encryption,RotationManager creds
```

## Scaling Considerations

### Concurrent Sessions
- Multi-threaded execution where possible
- Connection pooling for external APIs
- Rate limit management across sessions
- Resource usage monitoring

### Performance Optimizations
- Tool result caching
- Provider response streaming
- Context pruning for memory efficiency
- Non-blocking I/O operations

## Related Documentation

- [System Architecture](./system-architecture.md) - High-level system overview
- [Plugin Architecture](./plugin-architecture.md) - Extensibility mechanisms
- [Channel Provider Architecture](./channel-provider-architecture.md) - Messaging integration
- [Data & State Architecture](./data-state-architecture.md) - Configuration and state management
- [Tool Development](../channels/tools.md) - Building custom tools

## Source Code References

| Component | File |
|-----------|------|
| Agent Manager | `src/agents/agents.ts` |
| Session Manager | `src/agents/context.ts` |
| Pi Runner | `src/agents/pi-embedded-runner.ts` |
| Tool Executor | `src/agents/tool-policy.ts` |
| Context Pruner | `src/agents/context-pruner.ts` |
| Bash Tools | `src/agents/bash-tools.exec.ts` |
| Memory System | `src/agents/memory.ts` |
| Identity Management | `src/agents/identity.ts` |
| Auth System | `src/auth/*.ts` |
