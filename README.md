# Claude Code Skills

Kumpulan [Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) untuk [Claude Code](https://claude.com/claude-code) — pola implementasi siap pakai untuk project Laravel dan Flutter.

*A collection of Claude Code Agent Skills for Laravel and Flutter projects. Skill contents are written in Indonesian.*

## Daftar Skill

| Skill | Deskripsi |
|---|---|
| [laravel-fcm](laravel-fcm/SKILL.md) | Implementasi Firebase Cloud Messaging (FCM) push notification di Laravel tanpa package eksternal — FcmService (OAuth2 JWT + FCM HTTP v1), queued job dengan retry & cleanup token mati, tabel `user_devices`, dan registrasi token saat login API. |
| [flutter-app-alert](flutter-app-alert/SKILL.md) | Widget AppAlert — notifikasi overlay animasi pengganti SnackBar di Flutter: tipe success/error/warning/info, auto-dismiss, dark mode, tanpa dependency eksternal. |
| [flutter-smooth-refresh](flutter-smooth-refresh/SKILL.md) | Perbaikan bug RefreshIndicator di Flutter + Riverpod — layout jump, stutter, spinner menutup terlalu cepat — dengan pola `ref.refresh(provider.future)` + `Future.wait`. |
| [tomselect-livewire](tomselect-livewire/SKILL.md) | Implementasi Tom Select v2 pada Livewire Volt + Alpine.js — filter topbar & select dalam modal, aturan `wire:ignore`, sync dua arah ke `$wire`, CSS override dark/light, dan cara menghindari ParseError Volt. |

## Instalasi

Setiap skill adalah satu folder berisi `SKILL.md`. Salin folder skill yang diinginkan ke direktori skills Claude Code Anda:

**Untuk semua project (personal):**

```bash
git clone https://github.com/meamboidea/claude-skills.git
cp -R claude-skills/laravel-fcm ~/.claude/skills/
```

**Untuk satu project saja (dibagikan via repo project):**

```bash
cp -R claude-skills/laravel-fcm path/ke/project/.claude/skills/
```

Skill akan otomatis terdeteksi pada sesi Claude Code berikutnya.

## Penggunaan

Skill terpicu otomatis ketika permintaan Anda cocok dengan deskripsinya, atau panggil eksplisit dengan slash command:

```
/laravel-fcm
/flutter-app-alert
/flutter-smooth-refresh
/tomselect-livewire
```

Contoh: di project Laravel, cukup ketik *"implementasikan push notification FCM"* dan Claude akan mengikuti pola dari skill `laravel-fcm`.

## Lisensi

[MIT](LICENSE)
