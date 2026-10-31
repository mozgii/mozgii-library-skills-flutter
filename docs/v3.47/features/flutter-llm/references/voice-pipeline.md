---
title: Flutter voice AI pipeline
author: MOZGII, Inc.
lang: en
tags: [flutter, llm, voice, speech-to-text, tts]
description: Speech-to-text, text-to-speech, push-to-talk, a waveform, voice activity detection, server transcription, streaming playback and a full voice conversation turn.
---
# Flutter Voice AI Pipeline

A voice feature records, finds the end of speech, transcribes, asks the model, and speaks the reply, while a person can interrupt at any moment. Build it in small services so each stage can fail on its own.

```yaml
dependencies:
  dio: ^5.11.1
  firebase_analytics: ^13.0.1
  flutter_riverpod: ^3.4.3
  flutter_tts: ^4.2.5
  just_audio: ^0.10.6
  material_ui: ^1.9.0
  path: ^1.9.1
  path_provider: ^2.1.6
  permission_handler: ^13.0.2
  record: ^7.1.2
  speech_to_text: ^7.5.0
```

iOS needs `NSSpeechRecognitionUsageDescription` and `NSMicrophoneUsageDescription` in `Info.plist`. Android needs `android.permission.RECORD_AUDIO`.

## Speech-to-text and text-to-speech service

`speech_to_text` deprecated the separate `listen` parameters: `listenFor`, `pauseFor`, `localeId`, `cancelOnError`, `partialResults`, `onDevice` and the rest now go in `listenOptions`. With `onDevice: true`, speech never leaves the device, and the listen fails if the phone can't recognize locally, so catch that and fall back to server transcription. `listen` reports that failure, a `ListenFailedException`, through its own future and not through `onError`, so `catchError` puts it on the stream and closes it. Without that the exception goes unhandled and the stream never ends.

```dart
// lib/features/ai/data/voice_data_source.dart
import 'dart:async';

import 'package:flutter_tts/flutter_tts.dart';
import 'package:permission_handler/permission_handler.dart';
import 'package:speech_to_text/speech_to_text.dart';

class VoiceDataSource {
  final SpeechToText _stt = SpeechToText();
  final FlutterTts _tts = FlutterTts();
  StreamController<String>? _words;

  Future<bool> initialize() async {
    final microphone = await Permission.microphone.request();
    if (!microphone.isGranted) return false;
    return _stt.initialize(
      onError: (error) {
        _words?.addError(error.errorMsg);
        _closeWords();
      },
    );
  }

  /// Partial and final transcripts. Returns on-device text when [onDevice] is
  /// true, or an error event when the phone can't do that.
  Stream<String> listenStream({bool onDevice = false}) {
    final words = _words = StreamController<String>();
    _stt
        .listen(
          onResult: (result) {
            if (words.isClosed) return;
            words.add(result.recognizedWords);
            if (result.finalResult) _closeWords();
          },
          listenOptions: SpeechListenOptions(
            listenFor: const Duration(seconds: 30),
            pauseFor: const Duration(seconds: 3),
            localeId: 'en_US',
            cancelOnError: true,
            partialResults: true,
            onDevice: onDevice,
          ),
        )
        .catchError((Object error) {
          if (words.isClosed) return;
          words
            ..addError(error)
            ..close();
          if (identical(_words, words)) _words = null;
        });
    return words.stream;
  }

  void _closeWords() {
    _words?.close();
    _words = null;
  }

  Future<void> stopListening() => _stt.stop();

  Future<void> speak(String text) async {
    await _tts.setLanguage('en-US');
    await _tts.setSpeechRate(0.5);
    await _tts.setVolume(1.0);
    await _tts.speak(text);
  }

  Future<void> stopSpeaking() => _tts.stop();
}
```

## Push-to-talk recording state

`RecordingState` drives every UI transition, and nothing else tracks it. A Riverpod `Notifier` has no `dispose` method: register cleanup with `ref.onDispose`. Dart lists have no `takeLast`, so trim with `sublist`. `record`'s `onAmplitudeChanged` stream replaces a hand-written timer.

```dart
// lib/features/ai/presentation/push_to_talk.dart
import 'dart:async';

import 'package:flutter/services.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';
import 'package:record/record.dart';

enum RecordingState { idle, recording, transcribing, playing }

final voiceRecordingProvider =
    NotifierProvider<VoiceRecordingNotifier, RecordingState>(
  VoiceRecordingNotifier.new,
);

class VoiceRecordingNotifier extends Notifier<RecordingState> {
  static const _keptSamples = 40;

  final AudioRecorder _recorder = AudioRecorder();
  StreamSubscription<Amplitude>? _levels;
  List<double> _amplitudes = const [];

  List<double> get amplitudes => _amplitudes;

  @override
  RecordingState build() {
    ref.onDispose(() {
      _levels?.cancel();
      _recorder.dispose();
    });
    return RecordingState.idle;
  }

  Future<void> startRecording() async {
    if (!await _recorder.hasPermission()) return;
    await HapticFeedback.mediumImpact();
    final folder = await getTemporaryDirectory();
    await _recorder.start(
      const RecordConfig(encoder: AudioEncoder.aacLc, sampleRate: 16000),
      path: p.join(folder.path, 'voice_${DateTime.now().millisecondsSinceEpoch}.m4a'),
    );
    state = RecordingState.recording;
    _levels = _recorder
        .onAmplitudeChanged(const Duration(milliseconds: 80))
        .listen(_addLevel);
  }

  void _addLevel(Amplitude level) {
    final start = _amplitudes.length >= _keptSamples
        ? _amplitudes.length - _keptSamples + 1
        : 0;
    _amplitudes = [..._amplitudes.sublist(start), level.current];
    ref.notifyListeners();
  }

  Future<String?> stopRecording() async {
    await HapticFeedback.lightImpact();
    await _levels?.cancel();
    final path = await _recorder.stop();
    state = RecordingState.transcribing;
    return path;
  }
}

class PushToTalkButton extends ConsumerWidget {
  const PushToTalkButton({
    super.key,
    required this.transcribe,
    required this.onTranscript,
  });

  final Future<String?> Function(String path) transcribe;
  final ValueChanged<String> onTranscript;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final recording =
        ref.watch(voiceRecordingProvider) == RecordingState.recording;
    final notifier = ref.read(voiceRecordingProvider.notifier);
    final colors = Theme.of(context).colorScheme;
    return Semantics(
      button: true,
      label: 'Hold to talk',
      child: GestureDetector(
        onLongPressStart: (_) => notifier.startRecording(),
        onLongPressEnd: (_) async {
          final path = await notifier.stopRecording();
          if (path == null) return;
          final transcript = await transcribe(path);
          if (transcript != null) onTranscript(transcript);
        },
        child: AnimatedContainer(
          duration: const Duration(milliseconds: 200),
          width: recording ? 72 : 56,
          height: recording ? 72 : 56,
          decoration: BoxDecoration(
            shape: BoxShape.circle,
            color: recording ? colors.error : colors.primary,
          ),
          child: Icon(
            Icons.mic,
            color: recording ? colors.onError : colors.onPrimary,
          ),
        ),
      ),
    );
  }
}
```

## Waveform visualization from dBFS levels

`record` reports amplitude in dBFS: 0 is the maximum and about -160 is silence. Map the useful range, -60 to 0, onto the bar height.

```dart
// lib/features/ai/presentation/waveform_painter.dart
import 'package:material_ui/material_ui.dart';

class WaveformPainter extends CustomPainter {
  WaveformPainter(this.amplitudes, this.color);

  final List<double> amplitudes;
  final Color color;

  @override
  void paint(Canvas canvas, Size size) {
    if (amplitudes.isEmpty) return;
    final paint = Paint()
      ..color = color
      ..strokeWidth = 3
      ..strokeCap = StrokeCap.round;
    final barWidth = size.width / (amplitudes.length * 2);
    for (var i = 0; i < amplitudes.length; i++) {
      final normalized = ((amplitudes[i] + 60) / 60).clamp(0.05, 1.0);
      final barHeight = size.height * normalized;
      final x = i * barWidth * 2 + barWidth;
      canvas.drawLine(
        Offset(x, (size.height - barHeight) / 2),
        Offset(x, (size.height + barHeight) / 2),
        paint,
      );
    }
  }

  @override
  bool shouldRepaint(WaveformPainter oldDelegate) =>
      oldDelegate.amplitudes != amplitudes || oldDelegate.color != color;
}
```

## Voice activity detection: stop on silence

`recordUntilSilence` has to wait for the silence before it stops, so it returns a `Completer`'s future that the amplitude listener completes. It ignores silence until speech begins, or a quiet start would end the recording at once, and a maximum length stops a noisy room from recording forever.

```dart
// lib/features/ai/data/vad_recorder.dart
import 'dart:async';

import 'package:record/record.dart';

class VadRecorder {
  static const double _silenceThreshold = -40; // dBFS
  static const _silenceDuration = Duration(milliseconds: 1500);
  static const _maxDuration = Duration(seconds: 60);

  final AudioRecorder _recorder = AudioRecorder();
  StreamSubscription<Amplitude>? _levels;
  Completer<String?>? _result;

  Future<String?> recordUntilSilence({
    required String path,
    void Function(Duration silence)? onSilence,
  }) async {
    if (!await _recorder.hasPermission()) return null;
    await _recorder.start(
      const RecordConfig(encoder: AudioEncoder.aacLc, sampleRate: 16000),
      path: path,
    );
    final result = _result = Completer<String?>();
    final started = DateTime.now();
    var heardSpeech = false;
    DateTime? silentSince;

    _levels = _recorder
        .onAmplitudeChanged(const Duration(milliseconds: 100))
        .listen((level) {
      final now = DateTime.now();
      if (level.current >= _silenceThreshold) {
        heardSpeech = true;
        silentSince = null;
      } else if (heardSpeech) {
        final since = silentSince ??= now;
        final silence = now.difference(since);
        onSilence?.call(silence);
        if (silence >= _silenceDuration) stop();
      }
      if (now.difference(started) >= _maxDuration) stop();
    });
    return result.future;
  }

  /// Ends the recording early. Also called by the silence detector.
  Future<void> stop() async {
    await _levels?.cancel();
    _levels = null;
    final path = await _recorder.stop();
    final result = _result;
    _result = null;
    if (result != null && !result.isCompleted) result.complete(path);
  }

  Future<void> dispose() async {
    await stop();
    await _recorder.dispose();
  }
}
```

## Server-side transcription stream

Whisper and similar services hold a key, so they sit behind your backend too. The upload is multipart and the reply is a stream of events, decoded with the same `sseData` function as chat replies: it handles chunks that split an event. An `.m4a` file's media type is `audio/mp4`.

```dart
// lib/features/ai/data/transcription_data_source.dart
import 'package:dio/dio.dart';

import 'sse_data.dart';

class TranscriptionDataSource {
  TranscriptionDataSource(this._dio);

  final Dio _dio;

  Stream<String> transcribeAudioStream(String audioFilePath) async* {
    final formData = FormData.fromMap({
      'audio': await MultipartFile.fromFile(
        audioFilePath,
        filename: 'recording.m4a',
        contentType: DioMediaType('audio', 'mp4'),
      ),
    });
    final response = await _dio.post<ResponseBody>(
      '/api/ai/transcribe',
      data: formData,
      options: Options(responseType: ResponseType.stream),
    );
    await for (final word in sseData(response.data!.stream)) {
      if (word == '[DONE]') return;
      if (word.isNotEmpty) yield word;
    }
  }
}
```

## Streaming TTS playback with interrupt

`AudioPlayer.play` completes when playback ends or stops, so `finally` is the right place to clear the flag. Call `interrupt` as soon as the person starts a new recording.

```dart
// lib/features/ai/data/streaming_tts_service.dart
import 'package:just_audio/just_audio.dart';

class StreamingTtsService {
  final AudioPlayer _player = AudioPlayer();
  bool _isPlaying = false;

  bool get isPlaying => _isPlaying;

  Future<void> playFromUrl(String streamUrl) async {
    _isPlaying = true;
    try {
      await _player.setAudioSource(AudioSource.uri(Uri.parse(streamUrl)));
      await _player.play();
    } finally {
      _isPlaying = false;
    }
  }

  Future<void> interrupt() => _player.stop();

  Future<void> dispose() => _player.dispose();
}
```

## Firebase Analytics voice events

`logEvent` accepts strings and numbers only, and an assert rejects a `bool` in debug builds, so `interrupted` goes as `1` or `0`.

```dart
// lib/features/ai/data/voice_analytics.dart
import 'package:firebase_analytics/firebase_analytics.dart';

Future<void> logVoiceTurn({
  required int durationMs,
  required String transcript,
  required int replyLength,
  required bool wasInterrupted,
}) async {
  final analytics = FirebaseAnalytics.instance;
  await analytics.logEvent(
    name: 'ai_voice_transcription_completed',
    parameters: {
      'duration_ms': durationMs,
      'word_count': transcript.split(' ').length,
    },
  );
  await analytics.logEvent(
    name: 'ai_voice_tts_played',
    parameters: {
      'char_count': replyLength,
      'interrupted': wasInterrupted ? 1 : 0,
    },
  );
}
```

## VoiceChatOrchestrator: one full turn

The orchestrator runs recording, transcription, the model and speech in order, and a new turn cancels the last. It drops very short utterances, which are usually noise.

```dart
// lib/features/ai/domain/voice_chat_orchestrator.dart
import 'package:dio/dio.dart';

import '../data/chat_data_source.dart';
import '../data/streaming_tts_service.dart';
import '../data/transcription_data_source.dart';
import '../data/vad_recorder.dart';

abstract interface class ChatRepository {
  Stream<String> chatStream(
    List<ChatMessage> messages, {
    CancelToken? cancelToken,
  });
}

class VoiceChatOrchestrator {
  VoiceChatOrchestrator({
    required this.recorder,
    required this.transcription,
    required this.chat,
    required this.tts,
  });

  static const int _minWordCount = 3;

  final VadRecorder recorder;
  final TranscriptionDataSource transcription;
  final ChatRepository chat;
  final StreamingTtsService tts;

  CancelToken? _activeRequest;

  Future<void> startConversationTurn({
    required String recordingPath,
    required List<ChatMessage> history,
    required void Function(String token) onAiToken,
  }) async {
    await tts.interrupt();
    _activeRequest?.cancel('New recording started');
    final request = _activeRequest = CancelToken();
    await recorder.stop(); // a turn still recording ends here, not overlaps

    final audioPath = await recorder.recordUntilSilence(path: recordingPath);
    if (audioPath == null || request.isCancelled) return;

    final words = await transcription.transcribeAudioStream(audioPath).toList();
    final transcript = words.join(' ').trim();
    if (transcript.split(' ').length < _minWordCount) return;

    final messages = [...history, ChatMessage.user(transcript)];
    await for (final token in chat.chatStream(messages, cancelToken: request)) {
      onAiToken(token);
    }
  }
}
```
