---
name: flutter-app-alert
description: Add AppAlert animated overlay notification widget to a Flutter project. Replaces SnackBar with slide-up animated alert with success/error/warning/info types, auto-dismiss, dark mode, and left border accent. Use when asked to add toast notification, alert, snackbar replacement, or animated feedback in Flutter.
---

# AppAlert — Flutter Animated Alert Overlay

Animated overlay notification widget that replaces SnackBar. Slides up from bottom with fade animation, auto-dismisses after 3 seconds, supports dark mode.

## Features

- 4 types: `success` (green), `error` (red), `warning` (orange), `info` (blue)
- Slide-up + fade-in animation (320ms easeOutCubic), fade-out (220ms)
- Auto-dismiss after configurable duration (default 3s)
- Tap to dismiss manually
- Left border accent + icon circle + title + message
- Dark mode aware
- Only one alert shown at a time (auto-replaces previous)
- No external dependencies

## Step 1: Copy the widget code

Add the following to your shared widgets file (e.g., `lib/app/widgets/soft_ui_components.dart`) or create a new file `lib/core/widgets/app_alert.dart`.

Make sure `dart:async` is imported in that file.

```dart
import 'dart:async';
import 'package:flutter/material.dart';

// Replace these with your app's actual theme colors:
// AppTheme.primaryGreen → your success color
// AppTheme.accentOrange → your warning color

enum AppAlertType { success, error, warning, info }

class AppAlert {
  static OverlayEntry? _current;

  static void show(
    BuildContext context, {
    required String message,
    AppAlertType type = AppAlertType.success,
    String? title,
    Duration duration = const Duration(seconds: 3),
  }) {
    _current?.remove();
    _current = null;

    final entry = OverlayEntry(
      builder: (_) => _AppAlertOverlay(
        message: message,
        type: type,
        title: title,
        duration: duration,
        onDismiss: () {
          _current?.remove();
          _current = null;
        },
      ),
    );

    _current = entry;
    Overlay.of(context).insert(entry);
  }
}

class _AppAlertOverlay extends StatefulWidget {
  final String message;
  final AppAlertType type;
  final String? title;
  final Duration duration;
  final VoidCallback onDismiss;

  const _AppAlertOverlay({
    required this.message,
    required this.type,
    required this.title,
    required this.duration,
    required this.onDismiss,
  });

  @override
  State<_AppAlertOverlay> createState() => _AppAlertOverlayState();
}

class _AppAlertOverlayState extends State<_AppAlertOverlay>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  late final Animation<double> _slide;
  late final Animation<double> _fade;
  Timer? _dismissTimer;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 320),
      reverseDuration: const Duration(milliseconds: 220),
    );

    _slide = Tween<double>(begin: 60, end: 0).animate(
      CurvedAnimation(parent: _ctrl, curve: Curves.easeOutCubic),
    );
    _fade = Tween<double>(begin: 0, end: 1).animate(
      CurvedAnimation(parent: _ctrl, curve: Curves.easeOut),
    );

    _ctrl.forward();
    _dismissTimer = Timer(widget.duration, _dismiss);
  }

  Future<void> _dismiss() async {
    _dismissTimer?.cancel();
    if (mounted) {
      await _ctrl.reverse();
      widget.onDismiss();
    }
  }

  @override
  void dispose() {
    _dismissTimer?.cancel();
    _ctrl.dispose();
    super.dispose();
  }

  // ── Customize these colors to match your theme ──────────────────────────
  Color get _color => switch (widget.type) {
    AppAlertType.success => const Color(0xFF10B981),
    AppAlertType.error   => const Color(0xFFEF4444),
    AppAlertType.warning => const Color(0xFFF97316),
    AppAlertType.info    => const Color(0xFF3B82F6),
  };

  IconData get _icon => switch (widget.type) {
    AppAlertType.success => Icons.check_circle_rounded,
    AppAlertType.error   => Icons.cancel_rounded,
    AppAlertType.warning => Icons.warning_amber_rounded,
    AppAlertType.info    => Icons.info_rounded,
  };

  String get _defaultTitle => switch (widget.type) {
    AppAlertType.success => 'Berhasil',
    AppAlertType.error   => 'Gagal',
    AppAlertType.warning => 'Perhatian',
    AppAlertType.info    => 'Info',
  };

  @override
  Widget build(BuildContext context) {
    final isDark = Theme.of(context).brightness == Brightness.dark;
    final safePadding = MediaQuery.of(context).padding;

    return Positioned(
      bottom: safePadding.bottom + 16,
      left: 16,
      right: 16,
      child: AnimatedBuilder(
        animation: _ctrl,
        builder: (_, __) => Opacity(
          opacity: _fade.value,
          child: Transform.translate(
            offset: Offset(0, _slide.value),
            child: GestureDetector(
              onTap: _dismiss,
              child: Material(
                color: Colors.transparent,
                child: Container(
                  padding: const EdgeInsets.symmetric(
                    horizontal: 16,
                    vertical: 14,
                  ),
                  decoration: BoxDecoration(
                    color: isDark ? const Color(0xFF1E293B) : Colors.white,
                    borderRadius: BorderRadius.circular(16),
                    border: Border(
                      left: BorderSide(color: _color, width: 4),
                    ),
                    boxShadow: [
                      BoxShadow(
                        color: _color.withValues(alpha: 0.18),
                        blurRadius: 20,
                        offset: const Offset(0, 6),
                      ),
                      BoxShadow(
                        color: Colors.black.withValues(
                          alpha: isDark ? 0.4 : 0.08,
                        ),
                        blurRadius: 8,
                        offset: const Offset(0, 2),
                      ),
                    ],
                  ),
                  child: Row(
                    crossAxisAlignment: CrossAxisAlignment.center,
                    children: [
                      Container(
                        width: 40,
                        height: 40,
                        decoration: BoxDecoration(
                          color: _color.withValues(alpha: 0.12),
                          shape: BoxShape.circle,
                        ),
                        child: Icon(_icon, color: _color, size: 22),
                      ),
                      const SizedBox(width: 12),
                      Expanded(
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          mainAxisSize: MainAxisSize.min,
                          children: [
                            Text(
                              widget.title ?? _defaultTitle,
                              style: TextStyle(
                                fontSize: 13,
                                fontWeight: FontWeight.bold,
                                color: _color,
                              ),
                            ),
                            const SizedBox(height: 2),
                            Text(
                              widget.message,
                              style: TextStyle(
                                fontSize: 13,
                                color: isDark
                                    ? Colors.white70
                                    : const Color(0xFF475569),
                              ),
                            ),
                          ],
                        ),
                      ),
                      GestureDetector(
                        onTap: _dismiss,
                        child: Icon(
                          Icons.close_rounded,
                          size: 18,
                          color: isDark
                              ? Colors.white38
                              : const Color(0xFF94A3B8),
                        ),
                      ),
                    ],
                  ),
                ),
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

## Step 2: Use it

Replace `ScaffoldMessenger.of(context).showSnackBar(...)` with:

```dart
// Success
AppAlert.show(context, message: 'Data berhasil disimpan');

// Error
AppAlert.show(
  context,
  message: 'Koneksi gagal, coba lagi',
  type: AppAlertType.error,
);

// Warning dengan custom title
AppAlert.show(
  context,
  message: 'Sisa kuota hampir habis',
  type: AppAlertType.warning,
  title: 'Peringatan',
);

// Info dengan durasi custom
AppAlert.show(
  context,
  message: 'Update tersedia di versi 2.0',
  type: AppAlertType.info,
  duration: Duration(seconds: 5),
);
```

## Customization

| Property | Default | Description |
|---|---|---|
| `message` | required | Teks pesan utama |
| `type` | `success` | Jenis alert |
| `title` | Auto dari type | Override judul |
| `duration` | 3 detik | Durasi sebelum auto-dismiss |

Untuk mengubah warna agar cocok dengan tema app, edit method `_color` di `_AppAlertOverlayState`.

## Gotchas

- `AppAlert.show()` harus dipanggil dari `BuildContext` yang memiliki `Overlay` (dalam `MaterialApp`) — tidak bisa dari luar widget tree
- Jika dipanggil dari `ref.listen` (Riverpod), pastikan context masih `mounted` sebelum memanggil
- `AnimationController` di-dispose di `dispose()` — tidak ada memory leak
- Hanya satu alert tampil sekaligus; yang lama di-remove otomatis saat yang baru muncul
