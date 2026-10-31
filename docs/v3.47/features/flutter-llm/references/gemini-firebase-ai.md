---
title: Gemini data source with Firebase AI Logic
author: MOZGII, Inc.
lang: en
tags: [flutter, llm, gemini, firebase-ai]
description: A data source over firebase_ai that streams text, and an image with a question, from a model id held in Remote Config.
---
# Gemini Data Source With Firebase AI Logic

`GeminiDataSource` streams Gemini replies through `firebase_ai`, so no key sits in the app. The model id comes in from Remote Config, so retiring a model takes no release. `response.text` is null on a chunk with no text, which the stream skips.

```dart
// lib/features/ai/data/gemini_data_source.dart
import 'dart:typed_data';

import 'package:firebase_ai/firebase_ai.dart';

class GeminiDataSource {
  GeminiDataSource(String modelId)
      : _model = FirebaseAI.googleAI().generativeModel(
          model: modelId, // from Remote Config, so a retired id needs no release
          generationConfig: GenerationConfig(
            temperature: 0.7,
            maxOutputTokens: 2048,
          ),
        );

  final GenerativeModel _model;

  Stream<String> generateStream(String prompt) =>
      _textOf(_model.generateContentStream([Content.text(prompt)]));

  /// Multimodal: an image plus a question about it.
  Stream<String> analyzeImageStream(Uint8List imageBytes, String prompt) =>
      _textOf(_model.generateContentStream([
        Content.multi([
          InlineDataPart('image/jpeg', imageBytes),
          TextPart(prompt),
        ]),
      ]));

  Stream<String> _textOf(Stream<GenerateContentResponse> responses) async* {
    await for (final response in responses) {
      final text = response.text;
      if (text != null) yield text;
    }
  }
}
```
