---
title: Flutter on-device models with flutter_edge_ai
author: MOZGII, Inc.
lang: en
tags: [flutter, llm, on-device, flutter_edge_ai]
description: Registering an engine, installing a model, streaming a reply and showing first-run download progress with flutter_edge_ai.
---
# Flutter On-Device Models with flutter_edge_ai

`flutter_edge_ai` runs a language model on the device. It has no API key, no cost per call, and it works offline, which makes it the safe choice for private text.

```yaml
dependencies:
  flutter_edge_ai: ^2.5.4
  flutter_edge_ai_litertlm: ^1.19.0
```

## Register the engine and install a model

Register each engine once, before `runApp`, with `await FlutterEdgeAi.initialize(inferenceEngines: const [LiteRtLmEngine()]);` and an import of `package:flutter_edge_ai_litertlm/flutter_edge_ai_litertlm.dart`. The core package stays small, and engines come as separate packages. Model files are large, so host them on your own CDN: gated Hugging Face models need a token, and a token passed with `--dart-define` ends up readable in the binary.

`install` runs once per device, since installed models survive restarts. `maxTokens` has to fit the prompt, the history and the answer together, and `.litertlm` needs at least 1024.

The provider hands every screen the same data source. Pass the model's URL at build time with `--dart-define=MODEL_URL=...`; it holds no secret, only where your CDN keeps the file.

```dart
// lib/features/ai/data/edge_ai_data_source.dart
import 'package:flutter_edge_ai/flutter_edge_ai.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

final edgeAiDataSourceProvider = Provider<EdgeAiDataSource>(
  (ref) => EdgeAiDataSource(const String.fromEnvironment('MODEL_URL')),
);

class EdgeAiDataSource {
  EdgeAiDataSource(this._modelUrl);

  final String _modelUrl;

  Future<void> install({void Function(int percent)? onProgress}) async {
    await FlutterEdgeAi.installModel(
      modelType: ModelType.gemmaIt,
      fileType: ModelFileType.litertlm, // this field selects the engine
    ).fromNetwork(_modelUrl).withProgress((p) => onProgress?.call(p)).install();
  }

  Stream<String> generateStream(String prompt) async* {
    final model = await FlutterEdgeAi.getActiveModel(maxTokens: 2048);
    final chat = await model.createChat(maxOutputTokens: 512);
    await chat.addQueryChunk(Message.text(text: prompt, isUser: true));
    await for (final response in chat.generateChatResponseAsync()) {
      if (response is TextResponse) yield response.token;
    }
  }
}
```

## Three silent failures in the flutter_edge_ai API

`Message.isUser` defaults to `false`, so a user message without `isUser: true` returns an empty answer. `fileType` picks the engine rather than the file name, and it defaults to `ModelFileType.task`. `getActiveModel` throws `NoActiveModelError` when nothing is installed.

## First-run model download screen

A first download can take minutes, so show the percentage and offer a retry. `StreamProvider` turns the progress callback into three states the screen can render.

```dart
// lib/features/ai/presentation/model_loading_screen.dart
import 'dart:async';

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

import '../data/edge_ai_data_source.dart';

final modelInstallProvider = StreamProvider.autoDispose<int>((ref) {
  final progress = StreamController<int>();
  ref.onDispose(progress.close);
  ref.read(edgeAiDataSourceProvider).install(onProgress: (percent) {
    if (!progress.isClosed) progress.add(percent);
  }).then((_) => progress.close(), onError: (Object error, StackTrace stack) {
    if (!progress.isClosed) progress.addError(error, stack);
  });
  return progress.stream;
});

class ModelLoadingScreen extends ConsumerWidget {
  const ModelLoadingScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ref.watch(modelInstallProvider).when(
          loading: () => const Center(child: CircularProgressIndicator()),
          error: (error, _) => Center(
            child: TextButton(
              onPressed: () => ref.invalidate(modelInstallProvider),
              child: const Text('The download failed. Try again'),
            ),
          ),
          data: (percent) => Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Text('Downloading the AI model for the first time...'),
              LinearProgressIndicator(value: percent / 100),
              Text('$percent%'),
            ],
          ),
        );
  }
}
```
