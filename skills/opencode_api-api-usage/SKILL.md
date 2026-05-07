---
name: opencode_api-api-usage
description: Use when working with opencode_api package to send prompts to opencode server and interpret the message response structure (text, reasoning, tool parts).
---

# opencode_api Api Usage

## When to Use

Use this skill when:
- Integrating with opencode.ai server API
- Sending prompts to a session and processing responses
- Building tools that interact with opencode programmatically

## How It Works

The `opencode_api` package wraps the opencode server HTTP API using retrofit. Key workflow:

1. **Create client** with authentication
2. **Get or create session** to send messages to
3. **Send prompt** via `sendMessage()` 
4. **Parse response** — response contains `MessageWithParts` with `info` (message metadata) and `parts` (the actual content)

## Examples

### Basic Setup

```dart
import 'package:opencode_api/opencode_api.dart';

void main() async {
  final dio = OpencodeClient.createDio(
    username: 'opencode',
    password: 'your-password',
    baseUrl: 'http://localhost:4096',
  );
  final client = OpencodeClient(dio);
  // ... use client
}
```

### Send a Prompt and Read Response

```dart
// Get an existing session or create new one
final sessions = await client.getSessions();
final session = sessions.isEmpty 
    ? await client.createSession({}) 
    : sessions.first;

// Send a prompt
final response = await client.sendMessage(
  session.id!,  // session ID required
  {
    'parts': [
      {'type': 'text', 'text': 'What is 2+2?'},
    ],
  },
);

// Response contains message parts with different types
print('Message ID: ${response.info?.id}');
print('Role: ${response.info?.role}');

for (final part in response.parts ?? []) {
  print('Part type: ${part.type}');
  
  // Text part contains the actual response text
  if (part.type == 'text' && part.text != null) {
    print('Text: ${part.text}');  // e.g., "4"
  }
  
  // Reasoning part contains model's thinking
  if (part.type == 'reasoning' && part.text != null) {
    print('Reasoning: ${part.text}');
  }
}
```

### Understanding Part Types

The `MessageWithParts` response contains these common part types:

| Type | Field | Description |
|------|-------|-------------|
| `text` | `part.text` | Actual response text |
| `reasoning` | `part.text` | Model's reasoning/thinking |
| `step-start` | - | Marks start of processing step |
| `step-finish` | - | Marks end with cost/tokens info |
| `tool` | - | Tool call results (see below) |

### Handling Tool Parts

```dart
for (final part in response.parts ?? []) {
  if (part.type == 'tool') {
    // Tool parts have state: pending, running, completed, error
    // The actual output is in part.content as JSON string
    final toolOutput = part.content;
    print('Tool: $toolOutput');
  }
}
```

### Sending Multi-part Messages

```dart
await client.sendMessage(session.id!, {
  'parts': [
    {'type': 'text', 'text': 'Explain this code:'},
  ],
  // Optional: specify model/agent
  // 'model': 'anthropic/claude-3',
  // 'agent': 'general',
});
```

## Response Structure Reference

```dart
// MessageWithParts contains:
class MessageWithParts {
  final Message? info;      // Message metadata (id, role, timestamps)
  final List<MessagePart>? parts;  // Actual content
}

// Each part has:
class MessagePart {
  final String? id;      // Unique part ID
  final String? type;    // 'text', 'reasoning', 'tool', 'step-start', etc.
  final String? text;    // Content for text/reasoning types
  final String? content; // JSON string for tool parts
}
```

## Common Patterns

### Async Message (no wait)

```dart
// Send without waiting for response (returns immediately)
await client.sendMessageAsync(session.id!, {
  'parts': [{'type': 'text', 'text': 'Continue processing...'}],
});
```

### Error Handling

```dart
try {
  final response = await client.sendMessage(sessionId, body);
} on OpencodeException catch (e) {
  // Use e.userMessage for safe, non-technical error display
  print('Error: ${e.userMessage}');
}
```

### List Recent Messages

```dart
final messages = await client.getMessages(sessionId, limit: 10);
for (final msg in messages) {
  print('${msg.info?.role}: ${msg.parts?.firstOrNull?.text}');
}
```

## Notes

- Session ID is required for all message operations
- Response parts arrive in order of generation
- Text may be streamed in chunks; use all parts for complete response
- Tool calls appear as separate parts with their outputs in `content` field as JSON string
