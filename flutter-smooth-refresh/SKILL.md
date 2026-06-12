---
name: flutter-smooth-refresh
description: Fix RefreshIndicator bugs in Flutter + Riverpod projects — layout jump, stutter, premature close. Detects ref.invalidate and ref.refresh(provider) without .future inside onRefresh and AppBar refresh buttons, applies smooth refresh pattern with ref.refresh(provider.future) + Future.wait + Future.delayed(1000ms). Use when asked to fix pull-to-refresh, RefreshIndicator glitch, layout collapse on refresh, spinner closes too fast, or refresh stutter in Flutter.
---

# Flutter Smooth RefreshIndicator Fix

Memperbaiki bug `RefreshIndicator` yang memantul, layout kolaps, atau spinner menutup terlalu cepat pada project Flutter yang menggunakan Riverpod.

---

## Masalah yang Ditangani

### Bug 1 — Layout Collapse (`ref.invalidate` / `ref.refresh` tanpa `.future`)
```dart
// ❌ Salah — data langsung hilang, layout kolaps, scroll physics bertabrakan
onRefresh: () async {
  ref.invalidate(myProvider);
}

// ❌ Juga salah — sama efeknya
onRefresh: () async {
  ref.refresh(myProvider);
}
```

### Bug 2 — Spinner Menutup Instan (server lokal terlalu cepat)
```dart
// ❌ Future selesai < 15ms → spinner tidak sempat berputar
onRefresh: () async {
  await ref.refresh(myProvider.future);
}
```

### Bug 3 — AppBar Refresh Button tanpa `.future`
```dart
// ❌ Menyebabkan layout jump saat tombol ditekan
onPressed: () => ref.refresh(myProvider),
onPressed: () => ref.invalidate(myProvider),
```

---

## Pola yang Benar

```dart
// ✅ onRefresh — smooth dengan minimum delay 1 detik
onRefresh: () async {
  try {
    await Future.wait([
      ref.refresh(providerA.future),
      ref.refresh(providerB.future),
      Future.delayed(const Duration(milliseconds: 1000)), // visual minimum delay
    ]);
  } catch (_) {} // tangani error diam-diam agar refresh tidak macet
},

// ✅ AppBar refresh button — tetap pakai .future
onPressed: () => ref.refresh(myProvider.future),
```

**Mengapa ini benar:**
- `ref.refresh(provider.future)` → Riverpod tetap menampilkan data lama selama fetch baru berjalan (`skipLoadingOnRefresh: true`), tinggi layout stabil
- `Future.delayed(1000ms)` di dalam `Future.wait` → spinner berputar minimal 1 detik meski server menjawab < 15ms
- `catch (_) {}` → RefreshIndicator tidak macet jika salah satu provider error

---

## Langkah Implementasi

### 1. Temukan semua file yang terpengaruh

```bash
# Cari ref.invalidate di dalam onRefresh
grep -rn "onRefresh" lib/ --include="*.dart" -l

# Cari ref.refresh tanpa .future
grep -rn "ref\.refresh(" lib/ --include="*.dart" | grep -v "\.future"

# Cari ref.invalidate
grep -rn "ref\.invalidate(" lib/ --include="*.dart"
```

### 2. Perbaiki setiap onRefresh

Ganti pola bermasalah dengan pola smooth. Kumpulkan semua provider yang perlu di-refresh dalam satu `Future.wait`, tambahkan `Future.delayed(1000ms)`.

### 3. Perbaiki AppBar refresh button

```dart
// Sebelum
onPressed: () => ref.refresh(myProvider),
onPressed: () => ref.invalidate(myProvider),

// Sesudah
onPressed: () => ref.refresh(myProvider.future),
```

### 4. Ganti SnackBar dengan AppAlert (jika ada)

Jika project menggunakan `AppAlert`, ganti semua `ScaffoldMessenger.of(context).showSnackBar(...)` dengan `AppAlert.show(context, message: ..., type: AppAlertType....)`.

---

## Checklist Verifikasi

Setelah perbaikan, pastikan tidak ada lagi:

```bash
# Tidak boleh ada lagi di onRefresh
grep -rn "ref\.invalidate" lib/ --include="*.dart"

# Tidak boleh ada ref.refresh tanpa .future (kecuali di luar onRefresh/button dengan alasan jelas)
grep -rn "ref\.refresh(" lib/ --include="*.dart" | grep -v "\.future"

# Tidak boleh ada SnackBar jika project sudah pakai AppAlert
grep -rn "showSnackBar" lib/ --include="*.dart"
```

---

## Contoh Lengkap (sebelum & sesudah)

```dart
// ❌ SEBELUM
body: RefreshIndicator(
  onRefresh: () async {
    ref.invalidate(ordersProvider);         // layout collapse
    ref.invalidate(balanceProvider);        // layout collapse
  },
  child: ...
),
actions: [
  IconButton(
    onPressed: () => ref.refresh(ordersProvider), // layout jump
  ),
],

// ✅ SESUDAH
body: RefreshIndicator(
  color: AppTheme.accentOrange,
  onRefresh: () async {
    try {
      await Future.wait([
        ref.refresh(ordersProvider.future),
        ref.refresh(balanceProvider.future),
        Future.delayed(const Duration(milliseconds: 1000)),
      ]);
    } catch (_) {}
  },
  child: ...
),
actions: [
  IconButton(
    onPressed: () => ref.refresh(ordersProvider.future),
  ),
],
```

---

## Catatan Tambahan

- `ListView` di dalam `RefreshIndicator` harus punya `physics: const AlwaysScrollableScrollPhysics()` agar pull gesture terdeteksi meski list kosong
- Untuk state mutation (seperti `checkInProvider`), panggil `.reset()` di dalam `onRefresh` sebelum `Future.wait`, bukan di dalamnya
- `Future.delayed(1000ms)` adalah visual delay — bukan artificial slow, karena ia berjalan paralel dengan fetch data di dalam `Future.wait`
