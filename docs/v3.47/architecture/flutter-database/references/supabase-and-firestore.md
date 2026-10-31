---
title: Supabase Realtime and Firestore
author: MOZGII, Inc.
lang: en
tags: [flutter, supabase, firestore, realtime]
description: Live PostgreSQL subscriptions with Supabase Realtime and offline-first cloud documents with Firestore, including how to release each stream.
---
# Supabase Realtime and Firestore

Both stores push changes to the app over a live connection, so both need someone to close that connection.

```yaml
dependencies:
  supabase_flutter: ^2.18.3
  cloud_firestore: ^7.1.0
```

## Supabase Realtime: Live PostgreSQL

A Realtime channel filters Postgres changes on the server. Here it listens for inserts into `messages` for one room. The stream's `onCancel` calls `unsubscribe()`, so the channel goes away with the last listener: active channels consume server connections.

```dart
// lib/features/chat/data/datasources/chat_realtime_datasource.dart
import 'dart:async';

import 'package:supabase_flutter/supabase_flutter.dart';

class ChatMessage {
  const ChatMessage({required this.id, required this.text});

  factory ChatMessage.fromJson(Map<String, dynamic> json) =>
      ChatMessage(id: json['id'] as String, text: json['text'] as String);

  final String id;
  final String text;
}

class ChatRealtimeDataSource {
  final SupabaseClient _client = Supabase.instance.client;

  Stream<ChatMessage> watchMessages(String roomId) {
    late final RealtimeChannel channel;
    final controller = StreamController<ChatMessage>(
      onCancel: () => channel.unsubscribe(),
    );

    channel = _client
        .channel('room-$roomId')
        .onPostgresChanges(
          event: PostgresChangeEvent.insert,
          schema: 'public',
          table: 'messages',
          filter: PostgresChangeFilter(
            type: PostgresChangeFilterType.eq,
            column: 'room_id',
            value: roomId,
          ),
          callback: (payload) =>
              controller.add(ChatMessage.fromJson(payload.newRecord)),
        )
        .subscribe();

    return controller.stream;
  }
}
```

Realtime checks row level security on each event for each subscriber, so a subscriber needs `SELECT` access to receive anything. `DELETE` events are the exception: RLS policies don't apply to them, so don't treat a delete event as private.

## Firebase Firestore: Offline-First Cloud

Android and Apple builds get Firestore's offline cache with no config at all, while web builds start with it off. `snapshots()` returns the cached data immediately, then syncs when online. `SetOptions(merge: true)` keeps fields another client wrote.

`watchUserNotes` filters on `userId` and sorts on `updatedAt`, so `toJson` has to write both: a document without one of those fields never matches the query. The query also needs a composite index on `userId` and `updatedAt` descending. Until it exists, the listener fails with `failed-precondition`, and the message links to a page that creates the index. `FieldValue.serverTimestamp()` is null in the local snapshot until the server confirms the write, so `fromJson` accepts null.

```dart
// lib/features/notes/data/datasources/notes_firestore_datasource.dart
import 'package:cloud_firestore/cloud_firestore.dart';

class NoteModel {
  const NoteModel({
    required this.id,
    required this.userId,
    required this.text,
    this.updatedAt,
  });

  factory NoteModel.fromJson(Map<String, dynamic> json) => NoteModel(
    id: json['id'] as String,
    userId: json['userId'] as String,
    text: json['text'] as String,
    updatedAt: (json['updatedAt'] as Timestamp?)?.toDate(),
  );

  final String id;
  final String userId;
  final String text;
  final DateTime? updatedAt; // null until the server confirms a write

  Map<String, dynamic> toJson() => {
    'id': id,
    'userId': userId,
    'text': text,
    'updatedAt': FieldValue.serverTimestamp(),
  };
}

class NotesFirestoreDataSource {
  final FirebaseFirestore _db = FirebaseFirestore.instance;

  Stream<List<NoteModel>> watchUserNotes(String userId) => _db
      .collection('notes')
      .where('userId', isEqualTo: userId)
      .orderBy('updatedAt', descending: true)
      .snapshots() // offline cache returns immediately, syncs when online
      .map(
        (snap) => snap.docs.map((d) => NoteModel.fromJson(d.data())).toList(),
      );

  Future<void> saveNote(NoteModel note) async {
    await _db
        .collection('notes')
        .doc(note.id)
        .set(note.toJson(), SetOptions(merge: true));
  }
}
```

Cancel the stream subscription in `dispose()`: Firestore streams don't auto-close. A `StreamProvider` in Riverpod does this for you. Awaiting `set` while offline doesn't complete until the server confirms, so don't block a screen on it.
