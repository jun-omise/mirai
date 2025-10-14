# Void/Mirai Complete System Architecture

**Version:** 2.0 - Unified Documentation
**Date:** January 2025
**Coverage:** VSCode OSS Fork + Void AI Layer + Mirai Backend Proxy

---

## 📋 Table of Contents

1. [Executive Overview](#executive-overview)
2. [Complete System Architecture](#complete-system-architecture)
3. [Repository Structure](#repository-structure)
4. [Integration Architecture](#integration-architecture)
5. [Request Flow (End-to-End)](#request-flow-end-to-end)
6. [Void Core Components](#void-core-components)
7. [Mirai Backend Components](#mirai-backend-components)
8. [Provider Implementations](#provider-implementations)
9. [Tool System](#tool-system)
10. [Message Translation](#message-translation)
11. [Development Workflow](#development-workflow)
12. [Deployment & Operations](#deployment--operations)
13. [Troubleshooting](#troubleshooting)

---

## Executive Overview

### What is Void/Mirai?

Void (branded as "Mirai") is a **production-grade, open-source AI code editor** built as a fork of VSCode, with a unified architecture spanning:

1. **VSCode OSS Fork** (`/desktop/void`) - The editor foundation
2. **Void AI Layer** (within VSCode) - AI-powered features
3. **Mirai Backend Proxy** (`/desktop/mirai`) - Centralized LLM gateway

```
┌──────────────────────────────────────────────────────────────┐
│                        User's Machine                         │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │            Void Editor (VSCode OSS Fork)                │  │
│  │  /Users/.../Desktop/void                                │  │
│  │                                                          │  │
│  │  ┌────────────────┐     ┌──────────────────────────┐   │  │
│  │  │ Monaco Editor  │     │  Void AI Contribution    │   │  │
│  │  │  (Code UI)     │     │  - Chat Interface        │   │  │
│  │  │                │     │  - Code Editing          │   │  │
│  │  └────────────────┘     │  - Autocomplete          │   │  │
│  │                         │  - Tool Orchestration    │   │  │
│  │                         └──────────────────────────┘   │  │
│  └─────────────────────────────┬──────────────────────────┘  │
│                                │                              │
│                                │ HTTP POST                    │
│                                ▼                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         Mirai Backend Proxy Server                      │  │
│  │  /Users/.../Desktop/mirai                               │  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │  │
│  │  │  Message    │→ │   Provider   │→ │ LLM Provider  │  │  │
│  │  │ Translator  │  │   Adapter    │  │ Impls (SDKs)  │  │  │
│  │  └─────────────┘  └──────────────┘  └───────────────┘  │  │
│  │                                                          │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │  Auth, Credits, Token Tracking, Database         │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └─────────────────────────┬──────────────────────────────┘  │
└────────────────────────────┼─────────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │   External LLM APIs          │
              │  • Anthropic (Claude)        │
              │  • OpenAI (GPT)              │
              │  • Google (Gemini)           │
              │  • Ollama (Local)            │
              │  • Mistral, Groq, DeepSeek   │
              └──────────────────────────────┘
```

### Key Features

**Void Editor (Client)**:
- 🤖 Multi-provider AI integration (10+ providers, 100+ models)
- 🛠️ AI tool calling (file operations, code search, terminal, Git)
- ⚡ Fast/Slow Apply code editing
- 🔌 MCP (Model Context Protocol) support
- 🎯 Smart autocomplete
- 💬 Multi-threaded chat

**Mirai Backend (Server)**:
- 🔒 Centralized authentication & credit management
- 📊 Token usage tracking & analytics
- 🔄 Format translation (VSCode ↔ Internal ↔ Provider)
- 🛡️ Rate limiting & error handling
- 🎛️ Provider abstraction layer
- 🔧 XML tool extraction for non-native providers

---

## Complete System Architecture

### Full Stack Overview

```mermaid
graph TB
    subgraph "VSCode Foundation Layer"
        A[Monaco Editor Core]
        B[Workbench Shell]
        C[Platform Services]
        D[Base Utilities]
    end

    subgraph "Void AI Contribution"
        E1[Sidebar UI<br/>sidebarPane.ts]
        E2[Code Editor<br/>editCodeService.ts]
        E3[Autocomplete<br/>autocompleteService.ts]
        E4[Tools<br/>toolsService.ts]
        E5[Chat Threads<br/>chatThreadService.ts]
        E6[Settings<br/>voidSettingsService.ts]
    end

    subgraph "Void LLM Layer"
        F1[sendLLMMessageService<br/>Browser]
        F2[sendLLMMessage<br/>Electron Main]
        F3[Model Capabilities<br/>Database]
    end

    subgraph "Backend Proxy Option"
        G1[Backend Proxy Client<br/>_sendBackendProxyChat]
        G2{Use Backend<br/>Proxy?}
    end

    subgraph "Mirai Backend Server"
        H1[Message Translator<br/>VSCode ↔ Internal]
        H2[Provider Adapter<br/>Orchestration]
        H3[Provider Implementations<br/>SDK Integrations]
        H4[Auth & Credits<br/>Management]
        H5[XML Tool Extractor<br/>Fallback Tools]
    end

    subgraph "External Services"
        I1[Anthropic API]
        I2[OpenAI API]
        I3[Google AI API]
        I4[Ollama Local]
        I5[Mistral, Groq, etc.]
    end

    A --> B
    B --> C
    C --> D

    B --> E1
    E1 --> E5
    E5 --> F1
    E2 --> F1
    E3 --> F1
    E4 --> E5
    E1 --> E6

    F1 --> F2
    F2 --> G2
    F2 --> F3

    G2 -->|Yes| G1
    G2 -->|No| I1

    G1 -->|HTTP POST| H1
    H1 --> H2
    H2 --> H3
    H2 --> H4
    H3 --> H5

    H3 --> I1
    H3 --> I2
    H3 --> I3
    H3 --> I4
    H3 --> I5

    F2 -->|Direct| I1
    F2 -->|Direct| I2
    F2 -->|Direct| I3
    F2 -->|Direct| I4

    style E1 fill:#ff6b6b
    style E2 fill:#ff6b6b
    style E4 fill:#ff6b6b
    style F1 fill:#4ecdc4
    style G1 fill:#ffe66d
    style H2 fill:#95e1d3
```

### Architecture Layers

| Layer | Location | Purpose | Technology |
|-------|----------|---------|------------|
| **UI Layer** | Void/browser | User interface, React components | React, Monaco Editor |
| **Service Layer** | Void/browser | Business logic, state management | TypeScript Services |
| **LLM Integration** | Void/electron-main | Direct LLM calls OR proxy calls | Node.js, Native SDKs |
| **Backend Proxy** | Mirai server | Centralized LLM gateway (optional) | Node.js, Express/Remix |
| **Provider Layer** | Both | Actual SDK integrations | @anthropic-ai/sdk, openai, etc. |
| **External APIs** | Cloud/Local | LLM providers | REST/SSE APIs |

---

## Repository Structure

### Dual Repository Setup

```
/Users/srinivaskattimani/Desktop/
├── void/                           # VSCode OSS + Void AI
│   ├── src/vs/
│   │   ├── base/                   # Foundation utilities
│   │   ├── platform/               # Platform services
│   │   ├── editor/                 # Monaco Editor
│   │   └── workbench/
│   │       ├── browser/            # UI layer
│   │       ├── services/           # Core services
│   │       └── contrib/
│   │           └── void/           # 🎯 VOID AI FEATURES
│   │               ├── browser/    # UI & Services
│   │               │   ├── sidebarPane.ts
│   │               │   ├── editCodeService.ts
│   │               │   ├── toolsService.ts
│   │               │   └── react/  # React components
│   │               ├── common/     # Shared code
│   │               │   ├── sendLLMMessageService.ts
│   │               │   ├── modelCapabilities.ts
│   │               │   ├── prompts.ts
│   │               │   └── mcpService.ts
│   │               └── electron-main/  # Main process
│   │                   └── llmMessage/
│   │                       ├── sendLLMMessage.ts
│   │                       └── sendLLMMessage.impl.ts
│   ├── extensions/                 # Built-in extensions
│   ├── build/                      # Build system
│   └── cli/                        # Rust CLI
│
└── mirai/                          # Backend Proxy Server
    ├── app/
    │   ├── routes/                 # API endpoints
    │   │   ├── api.vscode.chat.ts
    │   │   └── api.vscode.fim.ts
    │   └── lib/
    │       └── vscode-proxy/       # 🎯 PROXY LAYER
    │           ├── message-translator.ts
    │           ├── provider-adapter.ts
    │           ├── provider-implementations.ts
    │           ├── xml-tool-extractor.ts
    │           └── model-capabilities.ts
    ├── prisma/                     # Database schema
    └── package.json
```

### Key Directories

| Path | Purpose | Lines of Code |
|------|---------|---------------|
| **Void** | | |
| `void/src/vs/workbench/contrib/void/` | Core AI features | ~15,000 |
| `void/src/vs/workbench/services/` | Workbench services | ~50,000 |
| `void/extensions/` | Built-in extensions | ~100,000 |
| **Mirai** | | |
| `mirai/app/lib/vscode-proxy/` | Backend proxy layer | ~5,000 |
| `mirai/app/lib/llm/` | LLM provider database | ~3,000 |
| `mirai/app/lib/credits/` | Credit management | ~2,000 |

---

## Integration Architecture

### Communication Flow

```mermaid
sequenceDiagram
    participant User
    participant VoidUI as Void UI<br/>(React/Browser)
    participant VoidService as Void Services<br/>(Browser Process)
    participant IPC as Electron IPC
    participant VoidMain as Void Main<br/>(Main Process)
    participant MiraiBackend as Mirai Backend<br/>(Node.js Server)
    participant ProviderSDK as Provider SDK<br/>(Anthropic/OpenAI)

    User->>VoidUI: Type message & send
    VoidUI->>VoidService: chatThreadService.addMessage()
    VoidService->>VoidService: sendLLMMessageService.sendMessage()
    VoidService->>IPC: Proxy to main process
    IPC->>VoidMain: sendLLMMessage.ts

    alt Backend Proxy Mode
        VoidMain->>MiraiBackend: HTTP POST /api/vscode/chat
        MiraiBackend->>MiraiBackend: Validate auth token
        MiraiBackend->>MiraiBackend: Check credits
        MiraiBackend->>MiraiBackend: Translate message format
        MiraiBackend->>ProviderSDK: Call provider SDK
        ProviderSDK-->>MiraiBackend: Stream response (SSE)
        MiraiBackend-->>VoidMain: Forward stream
    else Direct Mode
        VoidMain->>ProviderSDK: Direct SDK call
        ProviderSDK-->>VoidMain: Stream response
    end

    VoidMain-->>IPC: Stream chunks
    IPC-->>VoidService: Forward to browser
    VoidService-->>VoidUI: Update UI
    VoidUI-->>User: Display response
```

### Backend Proxy vs Direct Mode

#### Backend Proxy Mode (Recommended for Production)

```typescript
// Void configuration
modelSelection = {
  providerName: 'backendProxy',
  modelName: 'claude-3-5-sonnet-20241022'
}

settingsOfProvider = {
  backendProxy: {
    endpoint: 'http://localhost:3002'  // or production URL
  }
}
```

**Advantages**:
- ✅ Centralized authentication
- ✅ Credit tracking & usage limits
- ✅ Token usage analytics
- ✅ Rate limiting
- ✅ Unified error handling
- ✅ Cost monitoring

**Use cases**: Multi-user deployments, managed services, corporate environments

#### Direct Mode

```typescript
// Void configuration
modelSelection = {
  providerName: 'anthropic',  // or 'openAI', 'gemini', etc.
  modelName: 'claude-3-5-sonnet-20241022'
}

settingsOfProvider = {
  anthropic: {
    apiKey: 'sk-ant-...'
  }
}
```

**Advantages**:
- ✅ Lower latency (no proxy hop)
- ✅ Simpler setup
- ✅ Works offline (with Ollama)
- ✅ No external dependencies

**Use cases**: Individual developers, offline work, local models

---

## Request Flow (End-to-End)

### Complete Message Journey

```mermaid
graph TB
    subgraph "1. User Input"
        A[User types in Void sidebar]
    end

    subgraph "2. Browser Process"
        B1[React Component captures input]
        B2[chatThreadService.addMessage]
        B3[sendLLMMessageService.sendMessage]
        B4[Prepare request with context]
    end

    subgraph "3. IPC Transfer"
        C[Electron IPC Channel]
    end

    subgraph "4. Main Process"
        D1{Provider Check}
        D2[Backend Proxy?]
        D3[Direct Provider?]
    end

    subgraph "5a. Backend Proxy Path"
        E1[HTTP POST to Mirai]
        E2[Message Translator]
        E3[Provider Adapter]
        E4[Credit Deduction]
        E5[Provider Implementation]
    end

    subgraph "5b. Direct Path"
        F1[Provider Implementation<br/>in Void]
        F2[Direct SDK Call]
    end

    subgraph "6. LLM Provider"
        G1[Anthropic/OpenAI/etc.]
        G2[Stream Response]
    end

    subgraph "7. Response Flow"
        H1[Parse streaming chunks]
        H2[Extract tool calls]
        H3[Format for display]
    end

    subgraph "8. UI Update"
        I1[Update thread state]
        I2[Re-render React]
        I3[Display to user]
    end

    A --> B1
    B1 --> B2
    B2 --> B3
    B3 --> B4
    B4 --> C
    C --> D1

    D1 --> D2
    D1 --> D3

    D2 --> E1
    E1 --> E2
    E2 --> E3
    E3 --> E4
    E4 --> E5
    E5 --> G1

    D3 --> F1
    F1 --> F2
    F2 --> G1

    G1 --> G2
    G2 --> H1
    H1 --> H2
    H2 --> H3
    H3 --> I1
    I1 --> I2
    I2 --> I3

    style E3 fill:#95e1d3
    style F1 fill:#ffe66d
    style I2 fill:#ff6b6b
```

### Detailed Request Flow

#### Phase 1: User Input → Browser Services (Void)

```typescript
// 1. User types in sidebar
User.types("Refactor this function")

// 2. React component captures
<ChatInput onSubmit={handleSubmit} />

// 3. Thread service adds message
chatThreadService.addMessage({
  role: 'user',
  content: "Refactor this function",
  timestamp: Date.now()
})

// 4. LLM service prepares request
sendLLMMessageService.sendMessage({
  messagesType: 'chatMessages',
  messages: conversationHistory,
  modelSelection: { providerName, modelName },
  onText: (params) => updateUI(params),
  onFinalMessage: (params) => finalizeMessage(params),
  onError: (params) => showError(params)
})
```

#### Phase 2: Browser → Main Process (Void)

```typescript
// 5. IPC channel transfer
const channel = mainProcessService.getChannel('void-channel-llmMessage')

channel.call('sendLLMMessage', {
  requestId: uuid(),
  messages,
  modelSelection,
  settingsOfProvider,
  // ... other params
})

// 6. Main process receives
// In sendLLMMessage.ts (electron-main)
export const sendLLMMessage = async (params) => {
  const { providerName, modelName } = params.modelSelection

  if (providerName === 'backendProxy') {
    // Route to backend proxy
    await _sendBackendProxyChat(params)
  } else {
    // Route to direct implementation
    const impl = sendLLMMessageToProviderImplementation[providerName]
    await impl.sendChat(params)
  }
}
```

#### Phase 3: Void → Mirai Backend (if using proxy)

```typescript
// 7. HTTP request to Mirai
const response = await fetch(`${endpoint}/api/vscode/chat`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${miraiToken}`
  },
  body: JSON.stringify({
    messages: messages,
    providerName: 'anthropic',  // actual provider
    modelName: 'claude-3-5-sonnet-20241022',
    temperature: 0.7,
    stream: true,
    tools: tools,
    chatMode: 'agent'
  })
})

// 8. Mirai backend receives
// In Mirai: app/routes/api.vscode.chat.ts
export async function POST({ request }) {
  // Validate auth
  const token = request.headers.get('Authorization')
  const user = await validateToken(token)

  // Translate format
  const internalRequest = await translateVSCodeChatRequest(body, user.id)

  // Check credits
  await creditManager.checkAndDeduct(user.id, estimatedCost)

  // Route to provider
  await providerAdapter.handleChatCompletion(internalRequest, callbacks)
}
```

#### Phase 4: Mirai → Provider SDK

```typescript
// 9. Provider implementation
// In Mirai: app/lib/vscode-proxy/provider-implementations.ts
const sendAnthropicChat = async (params) => {
  const anthropic = new Anthropic({
    apiKey: config.apiKey,
    dangerouslyAllowBrowser: true
  })

  const stream = anthropic.messages.stream({
    system: systemMessage,
    messages: messages,
    model: modelName,
    max_tokens: 4096,
    tools: tools
  })

  // 10. Stream response
  stream.on('text', (textDelta) => {
    fullText += textDelta
    onText({ fullText, fullReasoning: '' })
  })

  stream.on('finalMessage', (response) => {
    onFinalMessage({
      fullText,
      fullReasoning: '',
      tokenUsage: response.usage
    })
  })
}
```

#### Phase 5: Response Back to User

```typescript
// 11. Stream flows back through layers
Provider → Mirai → Void Main → IPC → Void Browser → React → User

// 12. UI updates incrementally
onText: ({ fullText }) => {
  setMessages(prev => [...prev.slice(0, -1), {
    ...prev[prev.length - 1],
    content: fullText
  }])
}
```

---

## Void Core Components

### 1. Sidebar & Chat Interface

**File**: `src/vs/workbench/contrib/void/browser/sidebarPane.ts`

**Purpose**: Main chat UI and thread management

**Key Features**:
- Multi-threaded conversations
- Message history
- Tool call visualization
- Streaming response display
- Context management

**React Components**:
```
void/browser/react/src/
├── sidebar-view/
│   ├── Sidebar.tsx          # Main sidebar container
│   ├── ChatInput.tsx        # Message input with shortcuts
│   ├── MessageList.tsx      # Conversation display
│   └── ThreadSwitcher.tsx   # Thread navigation
└── components/
    ├── CodeBlock.tsx        # Syntax-highlighted code
    ├── ToolCallDisplay.tsx  # Tool execution visualization
    └── DiffView.tsx         # Code diff display
```

### 2. Code Editing Service

**File**: `src/vs/workbench/contrib/void/browser/editCodeService.ts`

**Purpose**: Handle all AI-powered code editing operations

**Key Concepts**:

```mermaid
stateDiagram-v2
    [*] --> CreateDiffZone: User triggers Apply/Cmd+K
    CreateDiffZone --> StreamLLM: Prepare file context
    StreamLLM --> ParseResponse: Receive LLM output
    ParseResponse --> FastApply: If Fast Apply mode
    ParseResponse --> SlowApply: If Slow Apply mode
    FastApply --> ComputeDiff: Search/replace blocks
    SlowApply --> ComputeDiff: Full file rewrite
    ComputeDiff --> ShowRedGreen: Display diff zones
    ShowRedGreen --> WaitApproval: User reviews changes
    WaitApproval --> Accept: Approve
    WaitApproval --> Reject: Decline
    WaitApproval --> AcceptAll: Accept all
    Accept --> ApplyChanges: Update ITextModel
    AcceptAll --> ApplyChanges
    ApplyChanges --> [*]
    Reject --> [*]
```

**DiffZone Lifecycle**:
- **DiffZone**: Region showing red (removed) and green (added) lines
- **DiffArea**: Generic line range tracker
- **Streaming**: Real-time diff updates as LLM generates

### 3. LLM Message Service

**File (Browser)**: `src/vs/workbench/contrib/void/common/sendLLMMessageService.ts`
**File (Main)**: `src/vs/workbench/contrib/void/electron-main/llmMessage/sendLLMMessage.ts`

**Purpose**: Unified interface for all LLM calls

**Architecture**:

```typescript
// Browser process (service)
class LLMMessageService implements ILLMMessageService {
  async sendLLMMessage(params: ServiceSendLLMMessageParams): Promise<string | null> {
    // 1. Validate auth
    if (!this.globalAuthService.isAuthenticated) {
      return null
    }

    // 2. Smart tool selection
    const tools = await this.aiToolRegistryService.getRelevantTools({
      userPrompt,
      maxTools: 5
    })

    // 3. Send to main process via IPC
    const requestId = generateUuid()
    this.channel.call('sendLLMMessage', {
      requestId,
      messages,
      modelSelection,
      settingsOfProvider,
      tools,
      miraiToken
    })

    return requestId
  }
}

// Main process (implementation)
export const sendLLMMessage = async (params, metricsService) => {
  const { providerName, modelName } = params.modelSelection

  if (providerName === 'backendProxy') {
    // Route to Mirai backend
    return await _sendBackendProxyChat(params)
  } else {
    // Direct provider call
    const impl = sendLLMMessageToProviderImplementation[providerName]
    return await impl.sendChat(params)
  }
}
```

### 4. Model Capabilities Database

**File**: `src/vs/workbench/contrib/void/common/modelCapabilities.ts`
**Size**: 1,675 lines

**Purpose**: Central configuration for 100+ models across 10+ providers

**Structure**:
```typescript
interface ModelCapabilities {
  modelName: string
  recognizedModelName: string
  contextWindow: number           // 128K, 200K, etc.
  reservedOutputTokenSpace: number
  supportsSystemMessage: 'system-role' | 'developer-role' | 'separated' | false
  specialToolFormat?: 'openai-style' | 'anthropic-style' | 'gemini-style'
  supportsFIM: boolean
  reasoningCapabilities?: {
    canIOReasoning: boolean
    openSourceThinkTags?: { start: string; end: string }
  }
}
```

**Why Critical**:
- Determines available features per model
- Validates token limits
- Enables smart model selection
- Supports 100+ models: GPT-4, Claude, Gemini, Llama, Mixtral, etc.

### 5. Tools Service

**File**: `src/vs/workbench/contrib/void/browser/toolsService.ts`
**Size**: 675 lines

**Purpose**: Orchestrate AI tool execution

**Tool Categories**:

| Category | Tools | Description |
|----------|-------|-------------|
| **File** | `read_file`, `write_file`, `list_dir`, `search_files` | File system operations |
| **Code** | `apply_edit`, `search_symbol`, `get_definition` | Code manipulation |
| **Terminal** | `run_command`, `get_output` | Shell execution |
| **Git** | `git_status`, `git_commit`, `git_diff` | Version control |
| **MCP** | Dynamic from MCP servers | External integrations |

**Tool Execution Flow**:
```typescript
async executeTool(name: string, args: any): Promise<ToolResult> {
  // 1. Validate parameters
  const tool = this.getToolByName(name)
  tool.validateParams(args)

  // 2. Execute with appropriate service
  switch(name) {
    case 'read_file':
      const content = await this.fileService.readFile(args.path)
      return { success: true, content }

    case 'apply_edit':
      await this.editCodeService.applyEdit(args)
      return { success: true }

    case 'run_command':
      const output = await this.terminalService.executeCommand(args.command)
      return { success: true, output }
  }
}
```

---

## Mirai Backend Components

### 1. Message Translator

**File**: `mirai/app/lib/vscode-proxy/message-translator.ts`

**Purpose**: Translate between VSCode format and internal format

**Provider Name Mapping**:
```typescript
const providerNameMapping = {
  // VSCode → Mirai Internal
  'anthropic' → 'Anthropic',
  'openAI' → 'OpenAI',
  'gemini' → 'Google',
  'mistral' → 'Mistral',
  'groq' → 'Groq',
  'deepseek' → 'Deepseek',
  'xAI' → 'xAI',
  'ollama' → 'Ollama'
}
```

**Message Format Translation**:
```typescript
// VSCode format
{
  role: 'user',
  content: 'Hello'
}

// Internal format (adds metadata)
{
  role: 'user',
  content: 'Hello',
  userId: 'user_123',
  timestamp: Date.now(),
  requestId: 'req_abc'
}
```

### 2. Provider Adapter

**File**: `mirai/app/lib/vscode-proxy/provider-adapter.ts`

**Purpose**: Orchestrate provider calls with auth, credits, and error handling

**Request Lifecycle**:
```typescript
class VSCodeProxyService {
  async handleChatCompletion(request, callbacks) {
    // 1. Validate request
    const errors = validateVSCodeChatRequest(request)
    if (errors.length > 0) throw new Error(errors.join(', '))

    // 2. Check authentication
    const user = await authBridge.validateToken(request.token)
    if (!user) throw new Error('Invalid token')

    // 3. Load provider settings
    const settings = await db.getProviderSettings(user.id, request.providerName)
    if (!settings) throw new Error('Provider not configured')

    // 4. Check credits
    const estimatedCost = estimateTokens(request.messages) * 0.01
    const hasCredits = await creditManager.hasCredits(user.id, estimatedCost)
    if (!hasCredits) throw new Error('Insufficient credits')

    // 5. Deduct credits (pre-charge)
    await creditManager.deductCredits(user.id, estimatedCost, {
      provider: request.providerName,
      model: request.modelName
    })

    // 6. Execute provider call
    const impl = providerImplementations[request.providerName]
    await impl.sendChat({
      ...request,
      settings,
      onText: callbacks.onText,
      onFinalMessage: callbacks.onFinal,
      onError: async (error) => {
        // Refund on error
        await creditManager.refundCredits(user.id, estimatedCost, 'error')
        callbacks.onError(error)
      }
    })

    // 7. Report actual usage & refund difference
    const actualCost = /* calculated from token usage */
    if (actualCost < estimatedCost) {
      await creditManager.refundCredits(user.id, estimatedCost - actualCost, 'overcharge')
    }
  }
}
```

### 3. Provider Implementations

**File**: `mirai/app/lib/vscode-proxy/provider-implementations.ts`
**Size**: 744 lines

**Purpose**: Direct SDK integrations with LLM providers

**Implementations**:

| Provider | SDK | Chat | FIM | Native Tools |
|----------|-----|------|-----|--------------|
| Anthropic | `@anthropic-ai/sdk` | ✅ | ❌ | ✅ anthropic-style |
| OpenAI | `openai` | ✅ | ❌ | ✅ openai-style |
| Google | `@google/genai` | ✅ | ❌ | ✅ gemini-style |
| Ollama | `ollama` | ✅ | ✅ | ❌ XML fallback |
| Mistral | `@mistralai/mistralai` | ✅ | ✅ | ✅ openai-style |
| Groq | `openai` (compatible) | ✅ | ❌ | ✅ openai-style |
| DeepSeek | `openai` (compatible) | ✅ | ❌ | ✅ openai-style |

**Example Implementation**:
```typescript
const sendAnthropicChat = async (params: SendChatParams) => {
  const { messages, separateSystemMessage, tools, onText, onFinalMessage, onError,
          settingsOfProvider, modelName, _setAborter, reportTokenUsage } = params

  // 1. Initialize SDK
  const anthropic = new Anthropic({
    apiKey: settingsOfProvider.anthropic.apiKey,
    dangerouslyAllowBrowser: true
  })

  // 2. Prepare request
  const stream = anthropic.messages.stream({
    system: separateSystemMessage,
    messages: messages as AnthropicLLMChatMessage[],
    model: modelName,
    max_tokens: 4096,
    tools: tools && tools.length > 0 ? tools : undefined,
    tool_choice: { type: 'auto' }
  })

  // 3. Handle streaming
  let fullText = ''
  stream.on('text', (textDelta) => {
    fullText += textDelta
    onText({ fullText, fullReasoning: '' })
  })

  stream.on('finalMessage', (response) => {
    // Report token usage
    if (response.usage) {
      reportTokenUsage(response.usage, 'chat')
    }

    // Check for tool calls
    const toolCalls = response.content.filter(c => c.type === 'tool_use')
    const toolCall = toolCalls[0] ? parseToolCall(toolCalls[0]) : undefined

    onFinalMessage({
      fullText,
      fullReasoning: '',
      anthropicReasoning: response.content.filter(c => c.type === 'thinking'),
      toolCall
    })
  })

  stream.on('error', (error) => {
    if (error instanceof Anthropic.APIError && error.status === 401) {
      onError({ message: 'Invalid Anthropic API key', fullError: error })
    } else {
      onError({ message: error.toString(), fullError: error })
    }
  })

  // 4. Set aborter
  _setAborter(() => stream.controller.abort())
}
```

### 4. XML Tool Extractor

**File**: `mirai/app/lib/vscode-proxy/xml-tool-extractor.ts`

**Purpose**: Extract tool calls from XML-formatted text (fallback for providers without native tool support)

**Supported XML Tags**:
```xml
<bash>npm install axios</bash>
<read_file>/path/to/file.ts</read_file>
<write_file>/path/to/new_file.ts
content here
</write_file>
<web_search>how to use React hooks</web_search>
<list_files>/src/components</list_files>
```

**How It Works**:
```typescript
export function extractXMLToolsWrapper(onText, onFinalMessage, chatMode) {
  let fullText = ''
  let trueFullText = '' // includes XML tags
  let toolBuffer = ''
  let foundToolTag = null

  const newOnText = (params) => {
    trueFullText = params.fullText

    // Check if tool tag is opening
    const toolMatch = findToolTag(trueFullText)
    if (toolMatch) {
      foundToolTag = toolMatch
      fullText = trueFullText.substring(0, toolMatch.idx)
    } else {
      fullText = trueFullText
    }

    // Parse tool if found
    let toolCall = null
    if (foundToolTag) {
      toolCall = parseXMLTool(foundToolTag.toolName, trueFullText.substring(foundToolTag.idx))
    }

    // Call original onText with cleaned text
    onText({
      fullText,  // WITHOUT tool XML
      fullReasoning: params.fullReasoning,
      toolCall   // Parsed tool call object
    })
  }

  return { newOnText, newOnFinalMessage }
}
```

**Before/After Example**:

Input (from LLM):
```
I'll read the file for you.
<read_file>/src/app.ts</read_file>
Let me check the contents.
```

Output:
```typescript
// Visible text
"I'll read the file for you.\nLet me check the contents."

// Tool call object
{
  id: 'tool_123',
  name: 'read_file',
  rawParams: { path: '/src/app.ts' },
  doneParams: ['path'],
  isDone: true
}
```

---

## Provider Implementations

### Supported Providers (Full List)

| Provider | Models | Tool Support | FIM Support | Location |
|----------|--------|--------------|-------------|----------|
| **Anthropic** | Claude 3.5 Sonnet, Opus, Haiku | Native (anthropic-style) | ❌ | Both |
| **OpenAI** | GPT-4, GPT-4o, GPT-3.5 | Native (openai-style) | ❌ | Both |
| **Google** | Gemini 1.5 Pro, Flash, 2.0 | Native (gemini-style) | ❌ | Both |
| **Mistral** | Large, Medium, Codestral | Native (openai-style) | ✅ | Mirai only |
| **Groq** | Llama 3, Mixtral | Native (openai-style) | ❌ | Both |
| **DeepSeek** | Coder, Chat | Native (openai-style) | ❌ | Both |
| **xAI** | Grok models | Native (openai-style) | ❌ | Both |
| **Ollama** | Any local model | XML fallback | ✅ | Both |
| **OpenRouter** | Aggregator (100+ models) | Varies | ❌ | Void only |
| **Together** | Open source models | Native (openai-style) | ❌ | Void only |

### Tool Format Comparison

#### OpenAI-Style Tools
```json
{
  "type": "function",
  "function": {
    "name": "read_file",
    "description": "Read contents of a file",
    "parameters": {
      "type": "object",
      "properties": {
        "path": {
          "type": "string",
          "description": "File path"
        }
      },
      "required": ["path"]
    }
  }
}
```

#### Anthropic-Style Tools
```json
{
  "name": "read_file",
  "description": "Read contents of a file",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "File path"
      }
    },
    "required": ["path"]
  }
}
```

#### Gemini-Style Tools
```typescript
{
  functionDeclarations: [{
    name: "read_file",
    description: "Read contents of a file",
    parameters: {
      type: "OBJECT",
      properties: {
        path: {
          type: "STRING",
          description: "File path"
        }
      },
      required: ["path"]
    }
  }]
}
```

---

## Tool System

### Tool Execution Architecture

```mermaid
graph TB
    subgraph "Tool Request"
        A[LLM Generates Tool Call]
        B{Native Tools<br/>Supported?}
    end

    subgraph "Native Tool Path"
        C1[Provider Returns<br/>Structured Tool Call]
        C2[Parse Tool Parameters]
        C3[toolsService.executeTool]
    end

    subgraph "XML Tool Path"
        D1[XML Extractor Detects Tag]
        D2[Parse XML Content]
        D3[toolsService.executeTool]
    end

    subgraph "Tool Execution"
        E1{Tool Type}
        E2[File Operation]
        E3[Code Edit]
        E4[Terminal Command]
        E5[Git Operation]
        E6[MCP Tool]
    end

    subgraph "Result"
        F1[Format Result]
        F2[Return to LLM]
        F3[Continue Generation]
    end

    A --> B
    B -->|Yes| C1
    B -->|No| D1

    C1 --> C2
    C2 --> C3
    D1 --> D2
    D2 --> D3

    C3 --> E1
    D3 --> E1

    E1 --> E2
    E1 --> E3
    E1 --> E4
    E1 --> E5
    E1 --> E6

    E2 --> F1
    E3 --> F1
    E4 --> F1
    E5 --> F1
    E6 --> F1

    F1 --> F2
    F2 --> F3

    style C3 fill:#4ecdc4
    style D3 fill:#4ecdc4
    style E1 fill:#ffe66d
```

### Available Tools

#### File Tools
```typescript
// Read file
{
  name: 'read_file',
  description: 'Read the contents of a file',
  parameters: { path: string }
}

// Write file
{
  name: 'write_file',
  description: 'Write content to a file',
  parameters: { path: string, content: string }
}

// List directory
{
  name: 'list_dir',
  description: 'List files in a directory',
  parameters: { path: string, recursive?: boolean }
}

// Search files
{
  name: 'search_files',
  description: 'Search for files matching a pattern',
  parameters: { pattern: string, path?: string }
}
```

#### Code Tools
```typescript
// Apply edit (Fast/Slow Apply)
{
  name: 'apply_edit',
  description: 'Apply code changes to a file',
  parameters: {
    file: string,
    changes: Array<{ old: string, new: string }>
  }
}

// Search symbol
{
  name: 'search_symbol',
  description: 'Find symbols in the codebase',
  parameters: { symbol: string, type?: 'function' | 'class' | 'variable' }
}

// Get definition
{
  name: 'get_definition',
  description: 'Get the definition of a symbol',
  parameters: { symbol: string, file?: string }
}
```

#### Terminal Tools
```typescript
// Run command
{
  name: 'run_command',
  description: 'Execute a shell command',
  parameters: { command: string, workingDirectory?: string }
}

// Get output
{
  name: 'get_output',
  description: 'Get output from last command',
  parameters: {}
}
```

#### Git Tools
```typescript
// Git status
{
  name: 'git_status',
  description: 'Get current git status',
  parameters: {}
}

// Git commit
{
  name: 'git_commit',
  description: 'Create a git commit',
  parameters: { message: string, files?: string[] }
}

// Git diff
{
  name: 'git_diff',
  description: 'Show git diff',
  parameters: { file?: string, staged?: boolean }
}
```

---

## Message Translation

### VSCode → Internal Format

```typescript
// VSCode Request
POST /api/vscode/chat
{
  "messages": [
    { "role": "system", "content": "You are a helpful assistant" },
    { "role": "user", "content": "Explain React hooks" }
  ],
  "providerName": "anthropic",
  "modelName": "claude-3-5-sonnet-20241022",
  "temperature": 0.7,
  "max_tokens": 4000,
  "stream": true,
  "tools": [/* MCP tools */],
  "chatMode": "agent"
}

// Internal Format (after translation)
{
  "messages": [
    { "role": "user", "content": "Explain React hooks" }
  ],
  "systemMessage": "You are a helpful assistant",  // Extracted
  "providerName": "Anthropic",  // Normalized
  "modelName": "claude-3-5-sonnet-20241022",
  "userId": "user_abc123",  // From auth token
  "temperature": 0.7,
  "maxTokens": 4000,
  "stream": true,
  "tools": [/* Converted to provider format */],
  "chatMode": "agent",
  "requestId": "req_xyz789",
  "timestamp": 1234567890
}
```

### Streaming Response Format

```typescript
// Server-Sent Events (SSE) format
data: {"id":"chunk_1","object":"chat.completion.chunk","created":1234567890,"model":"claude-3-5-sonnet-20241022","choices":[{"index":0,"delta":{"role":"assistant","content":"React"}}]}

data: {"id":"chunk_2","object":"chat.completion.chunk","created":1234567890,"model":"claude-3-5-sonnet-20241022","choices":[{"index":0,"delta":{"content":" hooks"}}]}

// ... more chunks ...

data: {"id":"chunk_final","object":"chat.completion.chunk","created":1234567890,"model":"claude-3-5-sonnet-20241022","choices":[{"index":0,"delta":{},"finish_reason":"stop"}],"usage":{"prompt_tokens":50,"completion_tokens":200,"total_tokens":250}}

data: [DONE]
```

### Tool Call Translation

```typescript
// Provider-specific tool call (Anthropic)
{
  type: 'tool_use',
  id: 'toolu_123',
  name: 'read_file',
  input: { path: '/src/app.ts' }
}

// Normalized tool call (internal)
{
  id: 'toolu_123',
  name: 'read_file',
  rawParams: { path: '/src/app.ts' },
  doneParams: ['path'],
  isDone: true
}

// VSCode format (for display)
{
  id: 'toolu_123',
  type: 'function',
  function: {
    name: 'read_file',
    arguments: '{"path":"/src/app.ts"}'
  }
}
```

---

## Development Workflow

### Local Development Setup

#### 1. Start Mirai Backend

```bash
# Terminal 1: Mirai Backend Server
cd /Users/srinivaskattimani/Desktop/mirai

# Install dependencies
npm install

# Set up environment
cp .env.example .env
# Edit .env with database URL, etc.

# Run database migrations
npm run prisma:migrate

# Start development server
npm run dev  # Runs on http://localhost:3002
```

#### 2. Start Void Editor

```bash
# Terminal 2: Void Editor
cd /Users/srinivaskattimani/Desktop/void

# Install dependencies (first time only)
npm install

# Build React components
npm run buildreact

# Start watch mode (auto-rebuild on changes)
npm run watch

# Terminal 3: Run the editor
./scripts/code.sh  # Or code.bat on Windows
```

#### 3. Configure Integration

In Void editor (once running):
1. Open Settings (Cmd+,)
2. Search for "Void"
3. Set Backend Proxy endpoint: `http://localhost:3002`
4. Select provider: `backendProxy`
5. Select model: `claude-3-5-sonnet-20241022`

Or edit settings JSON directly:
```json
{
  "void.backendProxy.endpoint": "http://localhost:3002",
  "void.chat.modelSelection": {
    "providerName": "backendProxy",
    "modelName": "claude-3-5-sonnet-20241022"
  }
}
```

### Development Commands

#### Void (VSCode)

| Command | Purpose | Terminal |
|---------|---------|----------|
| `npm run watch` | Watch core + extensions | Keep running |
| `npm run watchreact` | Watch React components | Keep running |
| `npm run compile` | Full compilation | One-time |
| `npm run buildreact` | Build React once | One-time |
| `./scripts/code.sh` | Launch editor | New terminal |
| `npm run test-node` | Run Node tests | One-time |
| `npm run test-browser` | Run browser tests | One-time |

#### Mirai Backend

| Command | Purpose | Terminal |
|---------|---------|----------|
| `npm run dev` | Start dev server | Keep running |
| `npm run build` | Build for production | One-time |
| `npm run prisma:migrate` | Run DB migrations | One-time |
| `npm run prisma:studio` | Open DB GUI | Keep running |
| `npm run test` | Run tests | One-time |
| `npm run lint` | Lint code | One-time |

### Testing Integration

#### End-to-End Test

```bash
# 1. Ensure both servers running
# Mirai: http://localhost:3002
# Void: Launched via ./scripts/code.sh

# 2. Test backend proxy health
curl http://localhost:3002/health
# Should return: {"status":"ok"}

# 3. Test chat endpoint (with auth token)
curl -X POST http://localhost:3002/api/vscode/chat \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Hello"}],
    "providerName": "anthropic",
    "modelName": "claude-3-5-sonnet-20241022",
    "stream": false
  }'

# 4. In Void editor, open sidebar (Cmd+L)
# 5. Type a message and verify:
#    - Message sent to backend
#    - Response streamed back
#    - Credits deducted (check DB)
#    - Token usage logged
```

### Debug Mode

#### Void Debug

```bash
# Launch with DevTools
./scripts/code.sh --inspect-brk

# Launch with verbose logging
./scripts/code.sh --verbose

# View console logs
# Open Help → Toggle Developer Tools in the running editor
```

#### Mirai Debug

```bash
# Enable debug logs
DEBUG=vscode-proxy:* npm run dev

# Or set in .env
LOG_LEVEL=debug
DEBUG=*
```

### Hot Reload Workflow

**Void Changes**:
1. Edit TypeScript file in `src/vs/workbench/contrib/void/`
2. Watch mode auto-compiles (~2-5 seconds)
3. Reload Void window (Cmd+R or Ctrl+R)
4. Test changes

**React Changes**:
1. Edit React component in `void/browser/react/src/`
2. `npm run watchreact` auto-rebuilds
3. Reload Void sidebar (close and reopen)
4. Test changes

**Mirai Changes**:
1. Edit file in `mirai/app/lib/vscode-proxy/`
2. Dev server auto-restarts (nodemon/remix)
3. No manual restart needed
4. Test changes

---

## Deployment & Operations

### Production Architecture

```
┌────────────────────────────────────────────────────────┐
│                    Users (Multiple)                     │
│  Void Editor (Desktop Apps) on Mac/Windows/Linux       │
└───────────────────────┬────────────────────────────────┘
                        │
                        │ HTTPS (443)
                        ▼
          ┌─────────────────────────────┐
          │   Load Balancer / CDN       │
          │   (Nginx / Cloudflare)      │
          └──────────────┬──────────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
┌─────────────────┐           ┌─────────────────┐
│ Mirai Backend 1 │           │ Mirai Backend 2 │
│  (Node.js)      │           │  (Node.js)      │
│  Port 3002      │           │  Port 3002      │
└────────┬────────┘           └────────┬────────┘
         │                              │
         └──────────────┬───────────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  PostgreSQL Database  │
            │  (User data, credits, │
            │   API keys, logs)     │
            └───────────────────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  Redis Cache          │
            │  (Sessions, rate      │
            │   limiting)           │
            └───────────────────────┘
```

### Environment Configuration

#### Production `.env` (Mirai)

```env
# Environment
NODE_ENV=production
PORT=3002

# Database
DATABASE_URL=postgresql://user:pass@prod-db.example.com:5432/mirai
DATABASE_POOL_MIN=2
DATABASE_POOL_MAX=10

# Redis
REDIS_URL=redis://prod-redis.example.com:6379
REDIS_CACHE_TTL=3600

# Security
CORS_ORIGINS=https://voideditor.com
RATE_LIMIT_ENABLED=true
RATE_LIMIT_MAX_REQUESTS=100
RATE_LIMIT_WINDOW_MS=60000

# Logging
LOG_LEVEL=info
LOG_FORMAT=json

# Monitoring
SENTRY_DSN=https://...@sentry.io/...
ENABLE_METRICS=true
METRICS_PORT=9090

# Credits
DEFAULT_CREDIT_BALANCE=1000
CREDIT_REFILL_RATE=100
CREDIT_REFILL_INTERVAL=86400000  # 24 hours
```

### Scaling Strategies

#### Horizontal Scaling
- Multiple Mirai backend instances behind load balancer
- Stateless design (no in-memory sessions)
- Shared PostgreSQL database
- Shared Redis for caching and rate limiting

#### Vertical Scaling
- Increase server RAM for caching
- More CPU cores for concurrent requests
- Faster database storage (SSD/NVMe)

#### Database Optimization
- Index on `user_id`, `provider_name`, `created_at`
- Partition large tables (token_usage_logs by month)
- Archive old data (> 6 months) to cold storage
- Read replicas for analytics queries

### Monitoring & Observability

#### Key Metrics

**Backend Health**:
- Request rate (req/sec)
- Response time (p50, p95, p99)
- Error rate (%)
- Active connections
- Memory usage
- CPU usage

**Business Metrics**:
- Token usage by provider
- Credit consumption rate
- User active sessions
- Tool call frequency
- Cost per request

**Provider Metrics**:
- Provider response time
- Provider error rate
- Provider rate limit hits
- Token usage by model

#### Logging

```typescript
// Structured logging
logger.info('Chat completion started', {
  requestId: 'req_123',
  userId: 'user_abc',
  provider: 'Anthropic',
  model: 'claude-3-5-sonnet-20241022',
  messageCount: 5,
  hasTools: true,
  timestamp: Date.now()
})

logger.info('Chat completion completed', {
  requestId: 'req_123',
  duration: 2345,  // ms
  tokens: {
    input: 150,
    output: 320,
    total: 470
  },
  creditsUsed: 4.7,
  success: true
})
```

#### Alerting

**Critical Alerts**:
- Backend server down
- Database connection lost
- Error rate > 5%
- Provider API failures

**Warning Alerts**:
- Response time > 10s (p95)
- Credit balance low (< 100)
- Rate limit approaching
- High memory usage (> 80%)

### Backup & Recovery

#### Database Backups
```bash
# Daily automated backups
0 2 * * * pg_dump $DATABASE_URL > /backups/mirai_$(date +%Y%m%d).sql

# Retention: 30 days
# Encryption: AES-256
# Storage: S3/GCS
```

#### Disaster Recovery
1. Restore database from latest backup
2. Deploy new backend instances
3. Update DNS to point to new instances
4. Verify functionality
5. Monitor for errors

**RTO (Recovery Time Objective)**: < 1 hour
**RPO (Recovery Point Objective)**: < 24 hours

---

## Troubleshooting

### Common Issues & Solutions

#### Issue: Backend Proxy Not Responding

**Symptoms**:
- Void shows "Connection failed" error
- Requests timeout
- Chat doesn't respond

**Debug Steps**:
```bash
# 1. Check if backend is running
curl http://localhost:3002/health
# Expected: {"status":"ok"}

# 2. Check backend logs
cd /Users/srinivaskattimani/Desktop/mirai
npm run dev  # Check console for errors

# 3. Verify endpoint in Void settings
# Open Void Settings → Search "Backend Proxy"
# Endpoint should be: http://localhost:3002

# 4. Check firewall/network
telnet localhost 3002
```

**Solutions**:
- Start Mirai backend: `cd mirai && npm run dev`
- Verify endpoint URL matches
- Check firewall rules
- Ensure port 3002 is not in use

#### Issue: Authentication Failed

**Symptoms**:
- "Invalid authentication token" error
- 401 Unauthorized responses

**Debug Steps**:
```bash
# 1. Check token in Void
# Open Command Palette (Cmd+Shift+P)
# Type: "Developer: Inspect Context Keys"
# Look for Mirai auth token

# 2. Verify token in database
psql $DATABASE_URL -c "SELECT * FROM users WHERE id='user_abc'"

# 3. Check token expiration
# Tokens expire after 30 days by default
```

**Solutions**:
- Re-authenticate in Void (Accounts section)
- Clear token cache: Delete `~/.mirai-editor/auth-tokens.json`
- Check database for user record
- Verify JWT secret matches between client/server

#### Issue: Tool Calls Not Working

**Symptoms**:
- LLM mentions tools but doesn't execute
- Tool results not returned to LLM
- XML tags visible in output

**Debug Steps**:
```typescript
// 1. Check model capabilities
// In Void DevTools console:
console.log(modelCapabilities.getModelCapabilities('anthropic', 'claude-3-5-sonnet-20241022'))
// Should show: specialToolFormat: 'anthropic-style'

// 2. Verify chat mode
// Chat mode must be 'agent' or 'gather' for tools
console.log(voidSettingsService.state.chatMode)

// 3. Check tool definitions
console.log(mcpService.getMCPTools())
```

**Solutions**:
- Verify `chatMode` is set to 'agent' or 'gather'
- Check model supports tools (has `specialToolFormat`)
- For Ollama: Ensure XML extraction is enabled
- Verify tool definitions are loaded

#### Issue: Streaming Stops Mid-Response

**Symptoms**:
- Response cuts off abruptly
- No final message received
- Chat appears hung

**Debug Steps**:
```bash
# 1. Check network stability
ping api.anthropic.com

# 2. Check token limits
# Response may stop if hitting output token limit

# 3. Check abort signals
# Look for "Aborted" in logs

# 4. Verify streaming configuration
# stream: true in request
```

**Solutions**:
- Increase request timeout in backend
- Check provider token limits
- Verify network is stable
- Ensure abort signal not triggered

#### Issue: High Credit Usage

**Symptoms**:
- Credits depleting faster than expected
- Unexpected cost increases

**Debug Steps**:
```sql
-- Check token usage
SELECT
  provider_name,
  model_name,
  SUM(total_tokens) as total_tokens,
  COUNT(*) as request_count
FROM token_usage_logs
WHERE user_id = 'user_abc'
  AND created_at > NOW() - INTERVAL '24 hours'
GROUP BY provider_name, model_name
ORDER BY total_tokens DESC;

-- Check expensive requests
SELECT *
FROM token_usage_logs
WHERE user_id = 'user_abc'
  AND total_tokens > 10000
ORDER BY created_at DESC
LIMIT 10;
```

**Solutions**:
- Review prompts for unnecessary verbosity
- Set lower `max_tokens` in requests
- Use cheaper models for simple tasks
- Implement client-side token estimation
- Add budget alerts

#### Issue: Provider API Errors

**Symptoms**:
- "Rate limit exceeded" errors
- "API key invalid" errors
- Provider-specific error codes

**Debug Steps**:
```bash
# 1. Check provider API status
curl https://status.anthropic.com
curl https://status.openai.com

# 2. Verify API key
# Test with curl
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-3-5-sonnet-20241022","max_tokens":1024,"messages":[{"role":"user","content":"Hello"}]}'

# 3. Check rate limits
# Look for 429 status codes in logs
```

**Solutions**:
- **Rate Limit**: Implement exponential backoff
- **Invalid API Key**: Regenerate key in provider dashboard
- **Provider Down**: Switch to fallback provider
- **Quota Exceeded**: Upgrade provider plan

### Debug Tools

#### Void DevTools Console

```typescript
// Access Void's internal state
// Open Help → Toggle Developer Tools

// Get current settings
voidSettingsService.state

// Get active thread
chatThreadService.currentThread

// Test LLM call
sendLLMMessageService.sendMessage({
  messagesType: 'chatMessages',
  messages: [{role: 'user', content: 'test'}],
  modelSelection: {providerName: 'backendProxy', modelName: 'claude-3-5-sonnet-20241022'},
  // ... other params
})

// View model capabilities
modelCapabilities.getModelCapabilities('anthropic', 'claude-3-5-sonnet-20241022')
```

#### Mirai Backend Logs

```bash
# Real-time logs
npm run dev | tee logs/debug.log

# With debug flags
DEBUG=vscode-proxy:* LOG_LEVEL=debug npm run dev

# Filter logs
npm run dev | grep "ERROR"
npm run dev | grep "userId=user_abc"
```

#### Database Queries

```sql
-- Active sessions
SELECT user_id, COUNT(*) as session_count
FROM active_sessions
WHERE last_activity > NOW() - INTERVAL '5 minutes'
GROUP BY user_id;

-- Recent errors
SELECT *
FROM error_logs
WHERE created_at > NOW() - INTERVAL '1 hour'
ORDER BY created_at DESC
LIMIT 50;

-- Credit balance
SELECT user_id, balance, last_updated
FROM credit_balances
WHERE user_id = 'user_abc';
```

### Performance Troubleshooting

#### Slow Responses

**Check**:
1. Network latency to provider
2. Database query performance
3. Token estimation accuracy
4. Message context size

**Solutions**:
- Use CDN for static assets
- Add database indexes
- Implement result caching
- Reduce message history length

#### Memory Leaks

**Check**:
```bash
# Monitor memory usage
watch -n 1 "ps aux | grep node"

# Heap dump (Node.js)
node --expose-gc --inspect server.js
# Open chrome://inspect
```

**Solutions**:
- Dispose event listeners properly
- Clear caches periodically
- Use weak references where appropriate
- Restart servers daily

---

## Quick Reference

### Key Files

| Component | File | Location | Purpose |
|-----------|------|----------|---------|
| **Void Core** | | | |
| Sidebar UI | `sidebarPane.ts` | `void/browser/` | Main chat interface |
| Code Editor | `editCodeService.ts` | `void/browser/` | AI code editing |
| LLM Service | `sendLLMMessageService.ts` | `void/common/` | LLM abstraction layer |
| LLM Impl | `sendLLMMessage.ts` | `void/electron-main/` | Provider SDKs |
| Model DB | `modelCapabilities.ts` | `void/common/` | 100+ model configs |
| Tools | `toolsService.ts` | `void/browser/` | Tool orchestration |
| **Mirai Backend** | | | |
| Translator | `message-translator.ts` | `mirai/lib/vscode-proxy/` | Format conversion |
| Adapter | `provider-adapter.ts` | `mirai/lib/vscode-proxy/` | Request orchestration |
| Providers | `provider-implementations.ts` | `mirai/lib/vscode-proxy/` | SDK integrations |
| XML Tools | `xml-tool-extractor.ts` | `mirai/lib/vscode-proxy/` | Tool extraction |
| Model DB | `model-capabilities.ts` | `mirai/lib/vscode-proxy/` | Model specs |

### Important Commands

```bash
# Development
cd void && npm run watch && ./scripts/code.sh
cd mirai && npm run dev

# Testing
curl http://localhost:3002/health
npm run test-node  # Void tests
npm run test       # Mirai tests

# Building
cd void && npm run compile
cd mirai && npm run build

# Database
cd mirai && npm run prisma:migrate
cd mirai && npm run prisma:studio
```

### Environment Variables

```env
# Mirai Backend
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
PORT=3002
NODE_ENV=development
LOG_LEVEL=debug

# Void (in settings.json)
"void.backendProxy.endpoint": "http://localhost:3002"
"void.chat.modelSelection": {
  "providerName": "backendProxy",
  "modelName": "claude-3-5-sonnet-20241022"
}
```

### API Endpoints

```bash
# Health check
GET /health

# Chat completion
POST /api/vscode/chat
Authorization: Bearer <token>
Body: { messages, providerName, modelName, stream }

# Fill-in-middle (autocomplete)
POST /api/vscode/fim
Body: { prefix, suffix, providerName, modelName }

# Token usage
POST /api/vscode/token-usage
Body: { model, provider, tokens }
```

---

## Resources

### Documentation
- [VSCode Source Organization](https://github.com/microsoft/vscode/wiki/Source-Code-Organization)
- [Monaco Editor API](https://microsoft.github.io/monaco-editor/)
- [Electron Docs](https://www.electronjs.org/docs)

### Repositories
- [Void GitHub](https://github.com/voideditor/void)
- [VSCode OSS](https://github.com/microsoft/vscode)

### Provider APIs
- [Anthropic](https://docs.anthropic.com)
- [OpenAI](https://platform.openai.com/docs)
- [Google AI](https://ai.google.dev/docs)
- [Ollama](https://ollama.ai/docs)
- [MCP Specification](https://modelcontextprotocol.io)

### Community
- [Discord](https://discord.gg/RSNjgaugJs)
- [Twitter/X](https://twitter.com/voideditor)
- [Website](https://voideditor.com)

---

**Document Version:** 2.0 - Unified Architecture
**Last Updated:** January 2025
**Maintained By:** Void/Mirai Team

---

*This unified documentation covers the complete Void/Mirai system including the VSCode OSS fork, Void AI contribution, and Mirai backend proxy server. For contribution guidelines, see [HOW_TO_CONTRIBUTE.md](./HOW_TO_CONTRIBUTE.md).*

