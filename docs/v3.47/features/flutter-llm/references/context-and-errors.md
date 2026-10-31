---
title: Flutter LLM context pruning and error states
author: MOZGII, Inc.
lang: en
tags: [flutter, llm, context, errors]
description: Pruning conversation history to a token budget, and a sealed error type that gives each AI failure its own message.
---
# Flutter LLM Context Pruning and Error States

Two small pieces keep a chat honest over a long session: a budget for what's sent, and a clear message for each way a turn can fail.

## The AI-Generated Label

`AiResponseLabel` sits above each AI reply, so nobody takes model output for a person's words or a checked fact. It takes its colour from the theme and needs no state.

```dart
// lib/features/ai/presentation/ai_response_label.dart
import 'package:gap/gap.dart';
import 'package:material_ui/material_ui.dart';

class AiResponseLabel extends StatelessWidget {
  const AiResponseLabel({super.key});

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final color = theme.colorScheme.tertiary;
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        Icon(Icons.auto_awesome, size: 12, color: color),
        const Gap(4),
        Text(
          'AI-generated',
          style: theme.textTheme.labelSmall?.copyWith(color: color),
        ),
      ],
    );
  }
}
```

## Conversation Context Pruning

Every model has a finite context window, so old messages have to go. This keeps the system prompt and as many recent messages as fit. Four characters per token is a rough estimate for English.

```dart
// lib/features/ai/data/context_manager.dart
import 'chat_data_source.dart';

class ConversationContextManager {
  static const int _maxTokenEstimate = 3000;

  List<ChatMessage> pruneForContext(
    List<ChatMessage> messages, {
    ChatMessage? systemPrompt,
  }) {
    var tokenCount = (systemPrompt?.content.length ?? 0) ~/ 4;
    final retained = <ChatMessage>[];
    for (final message in messages.reversed) {
      final tokens = message.content.length ~/ 4;
      if (tokenCount + tokens > _maxTokenEstimate) break;
      tokenCount += tokens;
      retained.add(message);
    }
    return [?systemPrompt, ...retained.reversed];
  }
}
```

## Responsible AI Labels and Errors

Label AI output where it appears, with a small "AI-generated" mark beside the reply. Show a content filter refusal differently from a network error, since retrying won't fix the first one. A sealed type lets the compiler check that every failure has a message:

```dart
// lib/features/ai/domain/ai_error.dart
sealed class AiError {}

class AiContentFiltered extends AiError {}

class AiRateLimited extends AiError {}

class AiNetworkError extends AiError {}

String messageFor(AiError error) => switch (error) {
      AiContentFiltered() => "This request can't be answered.",
      AiRateLimited() => 'Too many requests. Wait a moment and try again.',
      AiNetworkError() => 'No connection. Check your network and try again.',
    };
```
