---
title: "Stand-in screens for the go_router example"
author: MOZGII, Inc.
lang: en
tags: [flutter, navigation, go-router]
description: Minimal widgets that let the expenses routing example compile and run under test.
---
# Stand-In Screens for the Routing Example

Replace these with your real screens. They exist so the router and its tests run as written.

```dart
// lib/screens.dart
import 'package:material_ui/material_ui.dart';

class SignInScreen extends StatelessWidget {
  const SignInScreen({super.key});

  @override
  Widget build(BuildContext context) => const Text('Sign in');
}

class ExpenseListScreen extends StatelessWidget {
  const ExpenseListScreen({super.key});

  @override
  Widget build(BuildContext context) => const Text('Expenses');
}

class ExpenseDetailScreen extends StatelessWidget {
  const ExpenseDetailScreen({super.key, required this.id, this.highlight});

  final String id;
  final String? highlight;

  @override
  Widget build(BuildContext context) => Text('Expense $id');
}

class BudgetScreen extends StatelessWidget {
  const BudgetScreen({super.key});

  @override
  Widget build(BuildContext context) => const Text('Budget');
}

class NotFoundScreen extends StatelessWidget {
  const NotFoundScreen({super.key, required this.location});

  final String location;

  @override
  Widget build(BuildContext context) => Text('Nothing at $location');
}
```
