`dart_agent_core` is designed as a **mobile-first, local-first** Dart library that runs natively on all six Dart/Flutter platforms — Android, iOS, Web, Windows, macOS, and Linux. The library uses Dart's conditional export mechanism to automatically select platform-appropriate implementations at compile time, ensuring a single API surface works everywhere without runtime branching.

## Platform Support Matrix

| Platform | Dart/Flutter | File State Storage | HTTP Proxy | JavaScript Runtime | Notes |
|----------|--------------|-------------------|------------|-------------------|-------|
| **Android** | ✅ | ✅ Native filesystem | ✅ Full proxy support | ✅ Node.js via Process | Requires Node.js if using JavaScript skills |
| **iOS** | ✅ | ✅ Native filesystem | ✅ Full proxy support | ✅ Node.js via Process | Requires Node.js if using JavaScript skills |
| **macOS** | ✅ | ✅ Native filesystem | ✅ Full proxy support | ✅ Node.js via Process | Requires Node.js if using JavaScript skills |
| **Windows** | ✅ | ✅ Native filesystem | ✅ Full proxy support | ✅ Node.js via Process | Requires Node.js if using JavaScript skills |
| **Linux** | ✅ | ✅ Native filesystem | ✅ Full proxy support | ✅ Node.js via Process | Requires Node.js if using JavaScript skills |
| **Web** | ✅ | ⚠️ In-memory only | ❌ Browser-controlled | ❌ Use custom runtime | Use localStorage for persistence |
| **WASM** | ✅ | ⚠️ In-memory only | ❌ Browser-controlled | ❌ Use custom runtime | Fully compatible |

Sources: [README.md](README.md#L104-L120), [CHANGELOG.md](CHANGELOG.md#L83-L95)

## How Cross-Platform Support Works

The library uses Dart's conditional export mechanism to resolve platform-specific implementations at compile time. This means the same `dart_agent_core` package automatically selects the correct implementation based on your target platform.

### Platform-Specific Implementation Files

The library has three main platform-specific areas, each with an `_io.dart` (native) and `_web.dart` (browser) implementation:

```mermaid
graph TB
    subgraph "Compile-Time Platform Resolution"
        A[Public API Interface] --> B[Conditional Export]
        B --> C{dart.library.io?}
        C -->|Yes| D[Native Implementation]
        C -->|No| E[Web Implementation]
    end
    
    subgraph "Platform-Specific Areas"
        F[File State Storage]
        G[HTTP Utilities]
        H[Filesystem Helpers]
        I[JavaScript Runtime]
    end
    
    D --> J[file_state_storage_io.dart]
    D --> K[http_util_io.dart]
    D --> L[fs_io.dart]
    D --> M[node_javascript_runtime_io.dart]
    
    E --> N[file_state_storage_web.dart]
    E --> O[http_util_web.dart]
    E --> P[fs_stub.dart]
    E --> Q[node_javascript_runtime_web.dart]
```

**Sources:** [file_state_storage.dart](lib/src/agent/file_state_storage.dart#L1-L11), [http_util.dart](lib/src/core/http_util.dart#L1-L8), [fs.dart](lib/src/core/fs.dart#L1-L13)

### Platform-Specific Files Overview

| Area | Native File | Web File | Purpose |
|------|------------|----------|---------|
| **State Storage** | `file_state_storage_io.dart` | `file_state_storage_web.dart` | Persist agent session state |
| **HTTP** | `http_util_io.dart` | `http_util_web.dart` | Configure HTTP proxies |
| **Filesystem** | `fs_io.dart` | `fs_stub.dart` | File/directory operations |
| **JavaScript** | `node_javascript_runtime_io.dart` | `node_javascript_runtime_web.dart` | Execute JavaScript scripts |

Sources: [file_state_storage_io.dart](lib/src/agent/file_state_storage_io.dart#L1-L84), [file_state_storage_web.dart](lib/src/agent/file_state_storage_web.dart#L1-L66)

## Setup Instructions by Platform

### Native Platforms (Android, iOS, macOS, Windows, Linux)

For native platforms, the setup is straightforward. The library automatically uses filesystem-based state storage and supports all features including JavaScript runtime.

#### Step 1: Add Dependency

Add `dart_agent_core` to your `pubspec.yaml`:

```yaml
dependencies:
  dart_agent_core: ^2.0.4
```

#### Step 2: Configure API Keys

For native platforms, use `dart:io` to access environment variables:

```dart
import 'dart:io';

void main() async {
  final apiKey = Platform.environment['OPENAI_API_KEY'] ?? '';
  // ... rest of setup
}
```

Sources: [simple_agent_example.dart](example/simple_agent_example.dart#L1-L56), [simple_agent_with_gemini_example.dart](example/simple_agent_with_gemini_example.dart#L1-L50)

#### Step 3: State Storage Configuration

On native platforms, `FileStateStorage` automatically persists state to JSON files on disk:

```dart
final agent = StatefulAgent(
  name: 'my_agent',
  client: client,
  modelConfig: modelConfig,
  state: AgentState.empty(),
  stateStorage: FileStateStorage('/path/to/state/directory'),
  // ... other configuration
);
```

Sources: [file_state_storage_io.dart](lib/src/agent/file_state_storage_io.dart#L14-L84)

### Web Platform Setup

Web platform requires special considerations due to the absence of `dart:io` and filesystem access.

#### Step 1: Add Web Package

Add the `web` package for WASM-safe browser APIs:

```yaml
dependencies:
  dart_agent_core: ^2.0.4
  web: ^1.0.0
```

**Important:** Avoid the legacy `dart:html` package, which is not WASM-compatible.

#### Step 2: Configure API Keys

Use `package:web` for WASM-safe browser API access:

```dart
import 'package:web/web.dart' as web;
import 'package:dart_agent_core/dart_agent_core.dart';

void main() async {
  // Persist the key once from your app (e.g., a settings screen):
  // web.window.localStorage.setItem('OPENAI_API_KEY', '<key>');
  
  final apiKey = web.window.localStorage.getItem('OPENAI_API_KEY') ?? '';
  final client = OpenAIClient(apiKey: apiKey);
  // ... same StatefulAgent setup as Quick Start
}
```

Sources: [README.md](README.md#L113-L125)

#### Step 3: State Storage for Web

On web, `FileStateStorage` operates as an in-memory store. For durable persistence, use `localStorage` or `IndexedDB`:

```dart
// In-memory storage (default web behavior)
final agent = StatefulAgent(
  stateStorage: FileStateStorage('session_data'),
  // ... other configuration
);

// For durable web storage, implement your own StateStorage
class LocalStorageStateStorage implements StateStorage {
  @override
  Future<AgentState> loadOrCreate(String sessionId, Map<String, dynamic>? initialMetadata, {bool overwrite = true}) async {
    // Use localStorage or IndexedDB
    final stored = web.window.localStorage.getItem(sessionId);
    if (stored != null) {
      return AgentState.fromJson(jsonDecode(stored));
    }
    return AgentState(sessionId: sessionId, metadata: initialMetadata);
  }
  
  @override
  Future<void> save(AgentState state) async {
    web.window.localStorage.setItem(state.sessionId, jsonEncode(state.toJson()));
  }
  
  @override
  Future<void> delete(String sessionId) async {
    web.window.localStorage.removeItem(sessionId);
  }
  
  @override
  Future<bool> exist(String sessionId) async {
    return web.window.localStorage.getItem(sessionId) != null;
  }
}
```

Sources: [file_state_storage_web.dart](lib/src/agent/file_state_storage_web.dart#L14-L66)

#### Step 4: JavaScript Runtime for Web

The default `NodeJavaScriptRuntime` doesn't work on web. Use a custom implementation like `flutter_js`:

```yaml
dependencies:
  flutter_js: ^0.8.7
```

```dart
import 'package:dart_agent_core/dart_agent_core.dart';
import 'package:flutter_js/flutter_js.dart' as flutter_js;

class FlutterJavaScriptRuntime implements JavaScriptRuntime {
  final flutter_js.JavaScriptRuntime runtime;
  
  FlutterJavaScriptRuntime({required this.runtime});
  
  @override
  Future<JavaScriptExecutionResult> executeFile({
    required String scriptPath,
    Map<String, dynamic>? args,
    Duration? timeout,
    required JavaScriptBridgeRegistry bridgeRegistry,
    required JavaScriptBridgeContext bridgeContext,
  }) async {
    // Implement using flutter_js
    // ...
  }
}

void main() async {
  final agent = StatefulAgent(
    skillDirectoryPath: '/skills',
    javaScriptRuntime: FlutterJavaScriptRuntime(
      runtime: flutter_js.getJavascriptRuntime(),
    ),
    // ... other configuration
  );
}
```

Sources: [README.md](README.md#L206-L235), [node_javascript_runtime_web.dart](lib/src/agent/node_javascript_runtime_web.dart#L1-L29)

## Platform-Specific Feature Details

### HTTP Proxy Configuration

| Platform | Proxy Support | Implementation |
|----------|--------------|----------------|
| **Native** | ✅ Full support | Configures `HttpClient` with proxy settings |
| **Web** | ❌ Not supported | No-op (browsers manage proxying themselves) |

**Native Proxy Configuration:**

```dart
final client = OpenAIClient(
  apiKey: apiKey,
  proxyUrl: 'http://proxy.example.com:8080',
  // Supports authentication: 'http://user:pass@proxy.example.com:8080'
);
```

Sources: [http_util_io.dart](lib/src/core/http_util_io.dart#L1-L39), [http_util_web.dart](lib/src/core/http_util_web.dart#L1-L11)

### Filesystem Operations

| Operation | Native | Web |
|-----------|--------|-----|
| `fsDirectoryExistsSync()` | ✅ Uses `dart:io` | ❌ Always returns `false` |
| `fsFileExistsSync()` | ✅ Uses `dart:io` | ❌ Always returns `false` |
| `fsReadAsString()` | ✅ Uses `dart:io` | ❌ Throws `UnsupportedError` |
| `fsFindFiles()` | ✅ Recursively searches | ❌ Returns empty list |
| `fsAbsolutePath()` | ✅ Resolves to absolute path | ❌ Returns path unchanged |

Sources: [fs_io.dart](lib/src/core/fs_io.dart#L1-L64), [fs_stub.dart](lib/src/core/fs_stub.dart#L1-L38)

### JavaScript Runtime

| Feature | Native | Web |
|---------|--------|-----|
| **NodeJavaScriptRuntime** | ✅ Spawns Node.js process | ❌ Returns error |
| **Custom Runtime** | ✅ Can override | ✅ Required for JS skills |
| **Bridge Communication** | ✅ Via stdio JSON protocol | ✅ Via custom implementation |

**Native JavaScript Runtime (Node.js):**

```dart
final agent = StatefulAgent(
  skillDirectoryPath: '/path/to/skills',
  javaScriptRuntime: NodeJavaScriptRuntime(), // Uses 'node' command
  // ... other configuration
);
```

Sources: [node_javascript_runtime_io.dart](lib/src/agent/node_javascript_runtime_io.dart#L1-L256)

## Testing Platform Support

### Dart Unit Testing

Test platform-specific behavior using conditional imports in tests:

```dart
// test/platform_test.dart
import 'package:test/test.dart';
import 'package:dart_agent_core/dart_agent_core.dart';

void main() {
  group('Platform Support', () {
    test('FileStateStorage works on current platform', () async {
      final storage = FileStateStorage('test_sessions');
      final state = AgentState.empty();
      
      await storage.save(state);
      final loaded = await storage.loadOrCreate(state.sessionId, null);
      
      expect(loaded.sessionId, equals(state.sessionId));
    });
  });
}
```

### Flutter Integration Testing

For Flutter apps, test across platforms using integration tests:

```dart
// integration_test/platform_test.dart
import 'package:integration_test/integration_test.dart';
import 'package:dart_agent_core/dart_agent_core.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  testWidgets('Agent works on current platform', (tester) async {
    // Test agent functionality on the current platform
    final agent = StatefulAgent(
      name: 'test_agent',
      client: OpenAIClient(apiKey: 'test_key'),
      modelConfig: ModelConfig(model: 'test_model'),
      state: AgentState.empty(),
    );
    
    // ... test agent behavior
  });
}
```

## Troubleshooting Platform Issues

### Common Web Platform Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `dart:io` import error | Using `dart:io` directly in web code | Use conditional exports or `package:web` |
| `FileStateStorage` not persisting | In-memory storage on web | Implement custom `StateStorage` with localStorage |
| JavaScript skills not working | `NodeJavaScriptRuntime` on web | Use `flutter_js` or custom runtime |
| API key not accessible | `Platform.environment` unavailable | Use `localStorage` via `package:web` |

### Common Native Platform Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `node: command not found` | Node.js not installed | Install Node.js or provide custom runtime |
| Permission denied on state directory | Insufficient filesystem permissions | Check app sandbox permissions |
| Proxy not working | Incorrect proxy URL format | Use `http://user:pass@host:port` format |

## Next Steps

After setting up platform support, you're ready to explore the core features:

1. **[Supported LLM Providers](4-supported-llm-providers)** - Configure different AI providers
2. **[StatefulAgent and the Agentic Loop](5-statefulagent-and-the-agentic-loop)** - Build your first agent
3. **[State Management and File Persistence](7-state-management-and-file-persistence)** - Implement state persistence across platforms

The platform-agnostic design means you can write code once and deploy it across all supported platforms with minimal configuration changes.