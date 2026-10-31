---
name: flutter-llm
description: Adds LLM features to Flutter apps safely - choosing between flutter_edge_ai on-device models, Firebase AI Logic and your own backend proxy, keeping API keys out of the client, decoding server-sent events across chunks, streaming chat UI with cancel and error states, context pruning, AI labels, and a voice pipeline with speech_to_text, record, flutter_tts and just_audio. Use when asked to add an AI chat or assistant, call OpenAI, Anthropic or Gemini from Flutter, run a model on the phone, stream tokens to the screen, or build voice input and output.
metadata:
  tags:
  - flutter
  - llm
  - ai
  - gemma
  - openai
  - gemini
  - voice
---
# Flutter LLM

An LLM feature in an app needs three things to go right: the key stays safe, tokens reach the screen as they arrive, and a failure leaves the person with a clear next step. This skill picks where the model runs, streams its answer properly, and labels the result as AI-generated.

## Step 0: Read the Project's Versions

```bash
flutter --version
grep -nE "^\s+(material_ui|firebase_ai|firebase_app_check|flutter_edge_ai|flutter_gemma|google_generative_ai|dio|speech_to_text):" pubspec.yaml
```

Checked against Flutter 3.47.9, `material_ui` 1.9.0, `firebase_ai` 5.1.0, `flutter_edge_ai` 2.5.4, `flutter_edge_ai_litertlm` 1.19.0, `dio` 5.11.1, `speech_to_text` 7.5.0, `record` 7.1.2 and `just_audio` 0.10.6. Two lines there mean work before anything else: `flutter_gemma` is discontinued in favor of `flutter_edge_ai`, and `google_generative_ai` should move to `firebase_ai`. Both are covered below.

## Where the Model Should Run

| Your need | Use | Why the others lose here |
|---|---|---|
| Private or offline work, no cost per call | `flutter_edge_ai` | Cloud calls need a network and send the data away |
| Gemini, no server of your own | Firebase AI Logic through `firebase_ai` | A proxy is a service you'd have to run |
| OpenAI, Anthropic, or several providers | Your own backend proxy | They have no client SDK that hides the key |
| Rate limits, billing or logging per user | Your own backend proxy | Client code can't enforce them |
| Text recognition or translation only | `google_mlkit_*` packages | A general model is slower and larger |

## Never Put API Keys in Flutter

Anything compiled into an app can be extracted, including `--dart-define` values. A key in the client is a key on the internet. There are three safe paths, and all of them keep the secret away from the device:

1. **Your own backend.** The app calls your endpoint, and your server holds the OpenAI or Anthropic key: `Flutter app → your-backend.com/api/ai/chat → OpenAI API`. Use Firebase Functions, Supabase Edge Functions or any server.
2. **Firebase AI Logic.** `firebase_ai` calls Gemini directly from the client, with no key in the app: Google holds it. App Check is what stops strangers from using your project, so treat it as required.
3. **On-device inference.** `flutter_edge_ai` has no API key at all.

Model ids retire on the provider's schedule, so the client shouldn't hold them either. Send a purpose such as `chat` and let the proxy map it to the current id, or read the id from Remote Config with Firebase.

## Worked Example: Streaming Chat from Your Backend

Server-sent events arrive in network chunks, and a chunk can end inside a line or inside a multi-byte character. Calling `utf8.decode(chunk).split('\n')` on each chunk loses both. Decode the byte stream instead, split lines across chunks, and emit an event only at the blank line that ends it. The example needs these packages:

```yaml
dependencies:
  dio: ^5.11.1
  flutter_riverpod: ^3.4.3
  gap: ^3.0.1
  material_ui: ^1.9.0
```

```dart
// lib/features/ai/data/sse_data.dart
import 'dart:convert';

/// Yields the data payload of each server-sent event in [bytes].
Stream<String> sseData(Stream<List<int>> bytes) async* {
  final lines = utf8.decoder
      .bind(bytes.cast<List<int>>())
      .transform(const LineSplitter());
  final data = <String>[];
  await for (final line in lines) {
    if (line.isEmpty) {
      if (data.isNotEmpty) yield data.join('\n');
      data.clear();
    } else if (line.startsWith('data:')) {
      final value = line.substring('data:'.length);
      data.add(value.startsWith(' ') ? value.substring(1) : value);
    }
  }
  if (data.isNotEmpty) yield data.join('\n');
}
```

The data source posts to your proxy and reads OpenAI's delta format. It passes a `CancelToken` so the person can stop a reply and stop paying for it. Bad JSON raises a `FormatException`, and valid JSON in the wrong shape a `TypeError`, rather than disappearing; the notifier below turns both into a failed state. So does a connection cut mid-reply, which Dio passes through the body stream raw, as an `HttpException` rather than a `DioException`.

```dart
// lib/features/ai/data/chat_data_source.dart
import 'dart:convert';

import 'package:dio/dio.dart';

import 'sse_data.dart';

class ChatMessage {
  const ChatMessage({required this.role, required this.content});

  const ChatMessage.user(this.content) : role = 'user';

  final String role;
  final String content;

  Map<String, String> toJson() => {'role': role, 'content': content};
}

class ChatDataSource {
  ChatDataSource(this._dio);

  final Dio _dio;

  Stream<String> chatStream(
    List<ChatMessage> messages, {
    CancelToken? cancelToken,
  }) async* {
    final response = await _dio.post<ResponseBody>(
      '/api/ai/chat',
      data: {
        'purpose': 'chat', // the proxy picks the model id
        'stream': true,
        'messages': [for (final m in messages) m.toJson()],
      },
      options: Options(responseType: ResponseType.stream),
      cancelToken: cancelToken,
    );
    await for (final data in sseData(response.data!.stream)) {
      if (data == '[DONE]') return;
      final json = jsonDecode(data) as Map<String, dynamic>;
      // A final usage chunk arrives with an empty choices list.
      final choices = json['choices'] as List<dynamic>? ?? const [];
      if (choices.isEmpty) continue;
      final choice = choices.first as Map<String, dynamic>;
      final delta = choice['delta'] as Map<String, dynamic>?;
      final content = delta?['content'] as String?;
      if (content != null) yield content;
    }
  }
}
```

A Riverpod `Notifier` owns the reply, so the widget has one state to read. Sending again cancels the turn in flight.

```dart
// lib/features/ai/presentation/ai_reply_notifier.dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../data/chat_data_source.dart';

final chatDataSourceProvider = Provider<ChatDataSource>(
  (ref) => ChatDataSource(Dio(BaseOptions(baseUrl: 'https://your-backend.example'))),
);

class AiReply {
  const AiReply({this.text = '', this.streaming = false, this.failed = false});

  final String text;
  final bool streaming;
  final bool failed;
}

final aiReplyProvider =
    NotifierProvider<AiReplyNotifier, AiReply>(AiReplyNotifier.new);

class AiReplyNotifier extends Notifier<AiReply> {
  CancelToken? _turn;

  @override
  AiReply build() {
    ref.onDispose(() => _turn?.cancel());
    return const AiReply();
  }

  Future<void> send(List<ChatMessage> history) async {
    _turn?.cancel();
    final turn = _turn = CancelToken();
    final buffer = StringBuffer();
    state = const AiReply(streaming: true);
    try {
      final tokens = ref
          .read(chatDataSourceProvider)
          .chatStream(history, cancelToken: turn);
      await for (final token in tokens) {
        if (!ref.mounted) return;
        buffer.write(token);
        state = AiReply(text: buffer.toString(), streaming: true);
      }
      state = AiReply(text: buffer.toString());
    } on DioException catch (error) {
      if (!ref.mounted || !identical(turn, _turn)) return;
      state = AiReply(
        text: buffer.toString(),
        failed: !CancelToken.isCancel(error),
      );
    } on Exception {
      _fail(turn, buffer); // bad JSON, or a body stream cut mid-reply
    } on TypeError {
      _fail(turn, buffer); // valid JSON in the wrong shape
    }
  }

  void _fail(CancelToken turn, StringBuffer buffer) {
    if (ref.mounted && identical(turn, _turn)) {
      state = AiReply(text: buffer.toString(), failed: true);
    }
  }

  void stop() => _turn?.cancel();
}
```

The bubble shows empty, streaming and failed states. `Semantics(liveRegion: true)` makes a screen reader announce new text, and a plain `Text` is all a stream needs, because the tokens already set the pace.

```dart
// lib/features/ai/presentation/ai_reply_bubble.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:gap/gap.dart';
import 'package:material_ui/material_ui.dart';

import 'ai_reply_notifier.dart';
import 'ai_response_label.dart';

class AiReplyBubble extends ConsumerWidget {
  const AiReplyBubble({super.key, required this.onRetry});

  final VoidCallback onRetry;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final reply = ref.watch(aiReplyProvider);
    final colors = Theme.of(context).colorScheme;
    if (reply.text.isEmpty && !reply.streaming && !reply.failed) {
      return const SizedBox.shrink();
    }
    return Container(
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: colors.surfaceContainerHighest,
        borderRadius: BorderRadius.circular(16),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const AiResponseLabel(),
          const Gap(4),
          Semantics(liveRegion: true, child: SelectableText(reply.text)),
          if (reply.streaming)
            TextButton(
              onPressed: ref.read(aiReplyProvider.notifier).stop,
              child: const Text('Stop'),
            ),
          if (reply.failed)
            TextButton(
              onPressed: onRetry,
              child: const Text('The reply stopped early. Try again'),
            ),
        ],
      ),
    );
  }
}
```

`AiResponseLabel`, the small AI-generated tag above the text, is in [references/context-and-errors.md](references/context-and-errors.md). On `package:flutter/material.dart`, swap the `material_ui` import in these examples, or run `dart fix --apply --code=migrate_design_widgets`.

## Gemini with Firebase AI Logic (firebase_ai)

Use `firebase_ai` for Gemini. The older `google_generative_ai` package stopped at 0.4.7 in April 2025 and puts an API key in the client. Firebase AI Logic keeps the key on Google's side and works with App Check. `FirebaseAI.googleAI()` finds App Check on its own and caches its instance, so activate App Check first: `await FirebaseAppCheck.instance.activate();` right after `Firebase.initializeApp`. The defaults are Play Integrity on Android and DeviceCheck on Apple platforms.

`flutterfire configure` only writes `firebase_options.dart`; it doesn't switch the AI service on. Run `npx firebase-tools init ailogic` once for the project, or calls fail with `PERMISSION_DENIED`.

```yaml
dependencies:
  firebase_ai: ^5.1.0
  firebase_app_check: ^0.5.1
  firebase_core: ^5.1.0
```

`GeminiDataSource` wraps `generateContentStream` for text and for an image with a question, and is in [references/gemini-firebase-ai.md](references/gemini-firebase-ai.md).

## On-Device Models with flutter_edge_ai

`flutter_edge_ai` runs a model entirely on the device: private, offline, no cost per call. It replaces `flutter_gemma`, which pub.dev marks discontinued after 1.11.4; migrating means swapping the dependency and the import, and RAG moved to `flutter_edge_ai_rag` in 2.0. You add one engine package per model format: `flutter_edge_ai_litertlm` for `.litertlm` files, `flutter_edge_ai_llama` for GGUF, `flutter_edge_ai_onnx` for ONNX. Installing a model, streaming from it and a first-run download screen are in [references/on-device-models.md](references/on-device-models.md).

## Context Pruning and Responsible AI Errors

Every model has a finite context window, so old messages have to go. Label AI output where it appears, as `AiResponseLabel` does, and show a content filter refusal differently from a network error, since a retry won't fix the first. A token-budget pruner and a sealed error type are in [references/context-and-errors.md](references/context-and-errors.md).

## Voice Input and Output

Voice has its own traps: permissions, silence detection, partial results and interrupted playback. Speech-to-text, text-to-speech, push-to-talk, a waveform, voice activity detection, server transcription and a full conversation loop are in [references/voice-pipeline.md](references/voice-pipeline.md).

## Verify

Everything here was compiled, not run on a device, so test the microphone, speech and on-device model steps on real hardware.

```bash
flutter analyze
flutter test
```

This test proves the decoder survives the two splits that break a naive one: a chunk ending inside a character and an event spread over chunks. The import uses the package name `app`: replace it with yours. Then run the app with the network cut mid-reply, press Stop mid-reply, and confirm no key is in the release build: unzip the APK and run `strings lib/arm64-v8a/libapp.so | grep -i "sk-"`.

```dart
// test/features/ai/sse_data_test.dart
import 'dart:convert';
import 'dart:typed_data';

import 'package:app/features/ai/data/sse_data.dart';
import 'package:flutter_test/flutter_test.dart';

Stream<Uint8List> _chunks(List<List<int>> parts) =>
    Stream.fromIterable(parts.map(Uint8List.fromList));

void main() {
  test('joins an event split inside a multi-byte character', () async {
    final bytes = utf8.encode('data: héllo\n\ndata: [DONE]\n\n');
    final cut = bytes.indexOf(0xA9); // second byte of é
    final events = await sseData(
      _chunks([bytes.sublist(0, cut), bytes.sublist(cut)]),
    ).toList();
    expect(events, ['héllo', '[DONE]']);
  });

  test('joins data lines of one event and ignores comments', () async {
    final events = await sseData(_chunks([
      utf8.encode(': keep-alive\ndata: a\nda'),
      utf8.encode('ta: b\n\n'),
    ])).toList();
    expect(events, ['a\nb']);
  });
}
```

## Gotchas

1. **Tokens vanish now and then, or text shows `�`.** Symptom: a few words missing from long replies. Cause: each chunk was decoded and split alone. Fix: use `sseData` above.
2. **`type 'Utf8Decoder' is not a subtype of type 'StreamTransformer<Uint8List, String>'`.** Symptom: a crash on the first chunk. Cause: Dio's stream is `Stream<Uint8List>`. Fix: `.cast<List<int>>()` first.
3. **Reply keeps billing after the screen closes.** Symptom: provider costs too high. Cause: nothing cancels the request. Fix: pass a `CancelToken` and cancel it in `ref.onDispose`.
4. **Firebase AI calls from a stranger's script.** Symptom: a quota burned overnight. Cause: App Check wasn't enforced. Fix: call `activate` before the first `FirebaseAI.googleAI()`, then turn on enforcement in the console.
5. **`--dart-define=OPENAI_KEY=...` looked safe.** Symptom: the key shows up in `strings` output. Cause: define values compile into the binary. Fix: move the call behind your backend.
6. **Empty answer from an on-device model.** Symptom: the stream ends with no tokens. Cause: `Message.isUser` defaults to `false`. Fix: pass `isUser: true`.
7. **`Bad state: Cannot add event after closing` from a voice stream.** Symptom: an error after a final result. Cause: a late result arrived after the controller closed. Fix: check `controller.isClosed` first.
8. **`'string' OR 'number' must be set as the value of the parameter`.** Symptom: an assert in a debug build from `logEvent`. Cause: a `bool` parameter. Fix: send `1` or `0`.

## Anti-patterns

| Don't | Do instead |
|---|---|
| Call a provider with a key from the app | Go through your backend or Firebase AI Logic |
| Wait for the full reply | Stream and render as tokens arrive |
| `catch (_) {}` around a parse | Show a failed state with a retry |
| Hard-code model ids in the client | Send a purpose, or use Remote Config |
| Fake typing with a typewriter package | Render the tokens as they come |
| Send the whole history forever | Prune to a token budget |

## Pre-merge Checklist

- [ ] No API key in the repo, build config or binary
- [ ] App Check enforced when `firebase_ai` is used
- [ ] Replies stream, and a person can stop one
- [ ] Loading, empty and failed states are all on screen
- [ ] Decoder tests cover a split character and a split event
- [ ] AI output is labeled
- [ ] A screen reader announces new text
- [ ] Models download with progress, off the UI thread, with retry on failure

## See Also

- `flutter-network` for Dio setup and interceptors
- `flutter-security` for key handling and App Check
- `flutter-state` for Notifier patterns
- `flutter-analytics-firebase` for event rules
- `flutter-device-intelligence` for on-device hardware limits
