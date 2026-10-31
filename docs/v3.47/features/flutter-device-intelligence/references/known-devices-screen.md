---
title: "Flutter known devices security screen"
author: MOZGII, Inc.
lang: en
tags: [flutter, security, settings, riverpod]
description: A complete known devices page - the model, the Riverpod provider, loading, empty and error states, and signing a device out with an accessible confirmation.
---
# Known Devices Screen

A known devices page lets people see where their account is signed in and end a session they don't recognize. The list comes from your backend, which also resolves the city and country from the IP address. Keep the install ID off the screen: show the model, the place and the last activity.

```dart
// lib/features/security/presentation/known_devices_page.dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

final class KnownDevice {
  const KnownDevice({
    required this.id,
    required this.model,
    required this.platform,
    required this.lastCity,
    required this.lastCountry,
    required this.lastSeenAt,
    required this.isCurrent,
  });

  factory KnownDevice.fromJson(Map<String, dynamic> json) => KnownDevice(
        id: json['id'] as String,
        model: json['model'] as String,
        platform: json['platform'] as String,
        lastCity: json['last_city'] as String?,
        lastCountry: json['last_country'] as String?,
        lastSeenAt: DateTime.parse(json['last_seen_at'] as String),
        isCurrent: json['is_current'] as bool,
      );

  final String id;
  final String model;
  final String platform;
  final String? lastCity;
  final String? lastCountry;
  final DateTime lastSeenAt;
  final bool isCurrent;
}

class DeviceRepository {
  DeviceRepository(this._dio);

  final Dio _dio;

  Future<List<KnownDevice>> list() async {
    final response = await _dio.get<List<dynamic>>('/account/devices');
    return response.data!
        .cast<Map<String, dynamic>>()
        .map(KnownDevice.fromJson)
        .toList();
  }

  Future<void> signOut(String deviceId) =>
      _dio.delete<void>('/account/devices/$deviceId');
}

/// Override this with your authenticated Dio.
final deviceRepositoryProvider = Provider<DeviceRepository>(
  (ref) => throw UnimplementedError('Provide a DeviceRepository'),
);

final knownDevicesProvider = FutureProvider.autoDispose<List<KnownDevice>>(
  (ref) => ref.watch(deviceRepositoryProvider).list(),
);

class KnownDevicesPage extends ConsumerWidget {
  const KnownDevicesPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final devices = ref.watch(knownDevicesProvider);
    return Scaffold(
      appBar: AppBar(title: const Text('Your devices')),
      body: switch (devices) {
        AsyncData(value: final list) when list.isEmpty => const Center(
            child: Text('No other devices are signed in.'),
          ),
        AsyncData(value: final list) => RefreshIndicator(
            onRefresh: () => ref.refresh(knownDevicesProvider.future),
            child: ListView.builder(
              itemCount: list.length,
              itemBuilder: (context, index) => _DeviceTile(list[index]),
            ),
          ),
        AsyncError() => Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                const Text('Could not load your devices.'),
                TextButton(
                  onPressed: () => ref.invalidate(knownDevicesProvider),
                  child: const Text('Try again'),
                ),
              ],
            ),
          ),
        _ => const Center(child: CircularProgressIndicator()),
      },
    );
  }
}

class _DeviceTile extends ConsumerWidget {
  const _DeviceTile(this.device);

  final KnownDevice device;

  String get _place => [device.lastCity, device.lastCountry]
      .whereType<String>()
      .join(', ');

  Future<void> _confirmSignOut(BuildContext context, WidgetRef ref) async {
    final confirmed = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Sign out ${device.model}?'),
        content: const Text('That device will need to sign in again.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Cancel'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('Sign out'),
          ),
        ],
      ),
    );
    if (confirmed != true) return;

    try {
      await ref.read(deviceRepositoryProvider).signOut(device.id);
      ref.invalidate(knownDevicesProvider);
    } on DioException {
      if (context.mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Could not sign out that device.')),
        );
      }
    }
  }

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final seen = MaterialLocalizations.of(context)
        .formatMediumDate(device.lastSeenAt.toLocal());
    final details = [if (_place.isNotEmpty) _place, 'Last active $seen'];
    return ListTile(
      leading: Icon(
        device.platform == 'ios' ? Icons.phone_iphone : Icons.phone_android,
      ),
      title: Text(device.model),
      subtitle: Text(details.join(' · ')),
      trailing: device.isCurrent
          ? const Text('This device')
          : IconButton(
              icon: const Icon(Icons.logout),
              tooltip: 'Sign out ${device.model}',
              onPressed: () => _confirmSignOut(context, ref),
            ),
    );
  }
}
```

The `tooltip` doubles as the button's screen reader label. Refuse a sign-out of the current session on the server as well, so a bad client can't lock out the caller by mistake.
