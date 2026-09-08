State management in `dart_agent_core` revolves around a central, serializable data object—`AgentState`—and a pluggable persistence layer that saves it to disk (on native platforms) or memory (on web). This page explains how state is structured, how it is persisted, and how to integrate file-based storage into your agent lifecycle.

## The AgentState Structure

`AgentState` is the serializable snapshot of everything that happened during an agent's lifetime. It is the single source of truth held by `StatefulAgent` and is updated after every turn of the agentic loop.

The class contains several logical groups of data:

| Category | Fields | Purpose |
|----------|--------|---------|
| **Identity** | `sessionId`, `isRunning` | Unique session ID and run status |
| **History** | `history` (AgentMessageHistory) | All messages + episodic memories |
| **Usage Tracking** | `usages`, `currentLoopUsages`, `totalLoopCount`, `currentLoopCount` | Token consumption across runs |
| **User Context** | `metadata` | Arbitrary key-value pairs (e.g., user ID, preferences) |
| **Planning** | `plan` (PlanState?) | Current todo list from the Planner |
| **Skills** | `activeSkills` (List\<String\>?) | Names of currently active skills |
| **Error State** | `lastError` | Error message from the last failed run |
| **Prompt Tracking** | `systemPromptHistory`, `toolsHistory` | Snapshot of system prompt and tool list per message index |
| **Dynamic Injection** | `systemReminders` (Map\<String, String\>) | Short notes injected into every LLM call |

`AgentMessageHistory` holds two collections: the active `messages` list (LLMMessage objects) and `episodicMemories` (compressed historical snapshots). This separation is critical—episodic memories are created by the context compressor and provide a way to recall historical details without bloating the active context window.

Sources: [stateful_agent.dart](lib/src/agent/stateful_agent.dart#L223-L282), [memory.dart](lib/src/agent/memory.dart#L1-L58)

## The StateStorage Interface

`StateStorage` is a minimal abstract interface that decouples persistence from the agent core:

```dart
abstract class StateStorage {
  Future<AgentState> loadOrCreate(String sessionId, Map<String, dynamic>? initialMetadata);
  Future<void> save(AgentState state);
  Future<void> delete(String sessionId);
  Future<bool> exist(String sessionId);
}
```

The contract is simple: load an existing session or create a new one, save state when needed, and delete when done. Implementations are responsible for serialization format and storage medium.

Sources: [state_storage.dart](lib/src/agent/state_storage.dart#L1-L12)

## FileStateStorage: Platform-Aware Persistence

`FileStateStorage` is the built-in `StateStorage` implementation. It uses a conditional export pattern to provide different backends for native and web platforms:

```dart
export 'file_state_storage_web.dart'
    if (dart.library.io) 'file_state_storage_io.dart';
```

### Native (IO) Implementation

On native platforms, each session maps to a single JSON file on disk. The file is named `$sessionId.json` inside the directory specified at construction time. Key behaviors:

- **Automatic directory creation**: The save method creates the directory if it does not exist.
- **Error resilience**: Loading a corrupted file returns `null` and logs a warning, allowing the agent to create a fresh state.
- **Metadata merging**: When loading an existing session with `loadOrCreate`, any `initialMetadata` you pass is merged into the existing metadata by default.

The metadata merging behavior is controlled by the `overwrite` parameter:

| Parameter Value | Behavior |
|----------------|----------|
| `overwrite: true` (default) | New metadata keys are added; existing keys are overwritten |
| `overwrite: false` | Metadata is left untouched if the session already exists |

This is useful for multi-tenant scenarios where the same agent directory serves multiple users—each `loadOrCreate` call can inject fresh user context without disturbing other data.

Sources: [file_state_storage_io.dart](lib/src/agent/file_state_storage_io.dart#L1-L84)

### Web Implementation

On the web, the browser has no filesystem. The web implementation uses a process-wide `Map<String, String>` as an in-memory store. The `directoryPath` constructor parameter is used only to namespace entries—no filesystem operations occur. State persists for the lifetime of the page or tab but is lost on reload. The serialized JSON round-trip is preserved so behavior matches the native implementation as closely as possible.

For durable browser persistence, implement your own `StateStorage` backed by `localStorage` or IndexedDB.

Sources: [file_state_storage_web.dart](lib/src/agent/file_state_storage_web.dart#L1-L66)

## Integration with StatefulAgent

`StatefulAgent` accepts an optional `autoSaveStateFunc` callback. When provided, state is persisted automatically at specific lifecycle points:

```dart
final agent = StatefulAgent(
  name: 'my_agent',
  client: client,
  modelConfig: modelConfig,
  state: state,
  autoSaveStateFunc: (s) async => await storage.save(s),
);
```

### When State Is Saved

The `autoSaveStateFunc` is invoked after each tool batch completes, but only when the state persistence hook pipeline allows it. The hook pipeline provides three possible actions:

| Hook Action | Effect |
|-------------|--------|
| `StatePersistenceHookAction.proceed` | State is saved normally |
| `StatePersistenceHookAction.skip` | Save is skipped for this cycle |
| `StatePersistenceHookAction.abort` | Save is skipped and an error is logged |

This allows hooks to intercept state persistence—for example, to avoid saving transient intermediate states or to add validation before writing to disk.

Sources: [stateful_agent.dart](lib/src/agent/stateful_agent.dart#L466-L492), [stateful_agent.dart](lib/src/agent/stateful_agent.dart#L1221-L1240)

## Context Compression and Episodic Memory

For long-running sessions, the active context window will eventually exceed the model's token limit. `LLMBasedContextCompressor` handles this automatically by compressing older messages into episodic memories.

The compression process:

1. **Threshold check**: Compression triggers when `promptTokens` from the last `ModelUsage` exceeds `totalTokenThreshold` (default: 64,000).
2. **Message splitting**: Older messages are separated from the most recent `keepRecentMessageSize` (default: 10) messages, with care taken to keep `FunctionCall`/`FunctionExecutionResult` pairs intact.
3. **Summarization**: The older messages are sent to the LLM with a structured XML summarization prompt.
4. **Memory creation**: The summary becomes an `EpisodicMemory` (with a unique ID like `episode_aB3xYz`) and is added to `state.history.episodicMemories`.
5. **Context replacement**: The old messages are removed from the active context, replaced by a summary placeholder that includes the memory ID.

Each episodic memory stores both the XML summary and the original raw messages. The agent gains automatic access to the `retrieve_memory` tool, which can fetch verbatim past messages from any episodic memory by ID, with pagination support.

This design gives you the best of both worlds: a compact active context for the LLM, with full fidelity available on demand.

Sources: [context_compressor.dart](lib/src/agent/context_compressor.dart#L1-L219), [memory.dart](lib/src/agent/memory.dart#L59-L144)

## System Reminders: Lightweight State Injection

`AgentState.systemReminders` is a `Map<String, String>` of short notes injected into every LLM call as a `UserMessage` wrapped in `<system-reminders>` XML tags. Unlike metadata, reminders are visible to the model and are injected just before the last `UserMessage` in the request.

```dart
agent.state.systemReminders['current_time'] = DateTime.now().toIso8601String();
agent.state.systemReminders['rate_limit'] = 'User has 3 requests remaining today.';
```

Reminders persist in state until explicitly removed. They are designed for transient, per-request context that the model needs to see—such as time-sensitive information or rate-limit status.

Sources: [state_and_memory.md](doc/state_and_memory.md#L135-L147)

## State Recovery and Resumption

`AgentState.isRunning` enables crash recovery. When an agent crashes mid-run, `isRunning` remains `true`. On the next `loadOrCreate` call, the application can detect this and decide whether to resume or reset. The `lastError` field captures the error message from the failed run.

The `AgentCallToolContext` mechanism allows tools to access the live state without explicit parameters. Tools can read metadata, access the running agent, and use the shared `batchCallId` for parallel tool coordination.

```dart
String checkBalance(String currency) {
  final context = AgentCallToolContext.current;
  final userId = context?.state.metadata['user_id'] as String?;
  return fetchBalance(userId, currency);
}
```

Sources: [stateful_agent.dart](lib/src/agent/stateful_agent.dart#L335-L360), [state_and_memory.md](doc/state_and_memory.md#L40-L55)

## Complete Persistence Flow

The full lifecycle of state from creation to persistence follows this sequence:

1. **State Creation**: Either create via `AgentState.empty()` or load via `FileStateStorage.loadOrCreate()`.
2. **Agent Initialization**: Pass the state to `StatefulAgent` along with an optional `autoSaveStateFunc`.
3. **Run Execution**: During `agent.run()`, state is mutated in place as the agentic loop progresses.
4. **Context Compression**: If a `LLMBasedContextCompressor` is attached, older messages are compressed into episodic memories when the token threshold is exceeded.
5. **Auto-Save**: After each tool batch, if `autoSaveStateFunc` is set and the hook pipeline allows it, state is persisted.
6. **Session Completion**: The final state is saved when the run completes (or fails).

Sources: [simple_agent_with_state_example.dart](example/simple_agent_with_state_example.dart#L1-L86), [state_and_memory.md](doc/state_and_memory.md#L60-L78)

## Next Steps

After understanding state management and persistence, you may want to explore:

- [Streaming and Controller Events](8-streaming-and-controller-events) — how to observe agent state changes in real time via the controller and event bus.
- [AgentHook Pipeline](14-agenthook-pipeline) — how to intercept and modify state at every stage of the agentic loop, including the state persistence hook.
- [Context Compression and Episodic Memory](13-context-compression-and-episodic-memory) — a deeper look at how the `LLMBasedContextCompressor` manages long conversations.