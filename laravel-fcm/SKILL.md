---
name: laravel-fcm
description: Implementasi Firebase Cloud Messaging (FCM) push notification di project Laravel — tanpa package eksternal. Mencakup FcmService (OAuth2 JWT + FCM HTTP v1 API), queued job dengan retry & cleanup token mati, tabel user_devices untuk registrasi device token multi-perangkat, dan registrasi fcm_token saat login API. Gunakan saat user meminta "implementasi FCM", "push notification Laravel", "kirim notifikasi ke mobile", "Firebase messaging", atau setup notifikasi Laravel ke aplikasi Flutter/Android/iOS.
---

# Laravel FCM Push Notification (HTTP v1, tanpa package)

Implementasi FCM lengkap memakai FCM HTTP v1 API dengan OAuth2 service account. Tidak butuh package `kreait/firebase-php` — hanya `openssl` (bawaan PHP) dan HTTP client Laravel.

## Arsitektur

```
Mobile app login (kirim fcm_token + device_id)
        │
        ▼
user_devices (1 user = N perangkat, updateOrCreate per device_id)
        │
        ▼
KirimFcmNotifikasiJob::dispatch($userId, $title, $body, $data)
        │  (queued, 3x retry, backoff 60s/300s)
        ▼
FcmService → OAuth2 token (cache 3500s) → FCM HTTP v1 send per device
        │
        ▼
Token mati (404/410) → device dihapus otomatis
```

## Prasyarat

- Project Firebase dengan **Cloud Messaging API (v1)** aktif.
- File **service account JSON** dari Firebase Console → Project Settings → Service Accounts → Generate new private key.
- Queue sudah dikonfigurasi (`QUEUE_CONNECTION=database` cukup) dan worker berjalan (`php artisan queue:work`).

## Step 1: Kredensial & konfigurasi

Simpan file service account JSON di `storage/app/firebase/` dan **pastikan masuk .gitignore**:

```gitignore
/storage/app/firebase/
```

Tambahkan ke `.env`:

```env
FCM_PROJECT_ID=nama-project-firebase
FIREBASE_CREDENTIALS=storage/app/firebase/nama-file-service-account.json
```

Tambahkan ke `config/services.php`:

```php
'fcm' => [
    'project_id'  => env('FCM_PROJECT_ID'),
    'credentials' => env('FIREBASE_CREDENTIALS'),
],
```

> Jika project punya file config khusus (mis. `config/nama-app.php`), boleh taruh di sana — sesuaikan semua pemanggilan `config('services.fcm.*')` di bawah.

## Step 2: Migration `user_devices`

```php
Schema::create('user_devices', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained('users')->cascadeOnDelete();
    $table->string('device_id');
    $table->string('fcm_token');
    $table->string('platform', 10)->default('android');
    $table->timestamps();

    $table->unique(['user_id', 'device_id']);
    $table->index('user_id');
});
```

Unique constraint `user_id + device_id` memastikan login ulang dari perangkat yang sama hanya memperbarui token, bukan menduplikasi baris.

## Step 3: Model `UserDevice`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class UserDevice extends Model
{
    protected $fillable = ['user_id', 'device_id', 'fcm_token', 'platform'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

## Step 4: `FcmService`

Buat `app/Services/FcmService.php`:

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class FcmService
{
    /**
     * Get Google OAuth2 Access Token for Firebase.
     */
    public function getAccessToken(): ?string
    {
        return cache()->remember('fcm_access_token', 3500, function () {
            $credentials = config('services.fcm.credentials');
            if (! $credentials) {
                Log::warning('FCM Credentials are not configured.');
                return null;
            }

            $data = null;
            if (str_starts_with($credentials, '{')) {
                $data = json_decode($credentials, true);
            } else {
                // Resolve relative paths against the project root so the file
                // is found regardless of the process working directory.
                $path = str_starts_with($credentials, DIRECTORY_SEPARATOR)
                    ? $credentials
                    : base_path($credentials);

                if (file_exists($path)) {
                    $data = json_decode(file_get_contents($path), true);
                }
            }

            if (! isset($data['private_key']) || ! isset($data['client_email'])) {
                Log::warning('FCM Credentials format is invalid. Missing private_key or client_email.');
                return null;
            }

            $privateKey = $data['private_key'];
            $clientEmail = $data['client_email'];

            $header = $this->base64UrlEncode(json_encode(['alg' => 'RS256', 'typ' => 'JWT']));
            $now = time();
            $payload = $this->base64UrlEncode(json_encode([
                'iss' => $clientEmail,
                'sub' => $clientEmail,
                'aud' => 'https://oauth2.googleapis.com/token',
                'iat' => $now,
                'exp' => $now + 3600,
                'scope' => 'https://www.googleapis.com/auth/firebase.messaging',
            ]));

            $signature = '';
            if (! openssl_sign("$header.$payload", $signature, $privateKey, OPENSSL_ALGO_SHA256)) {
                Log::error('FCM Token generation failed: Unable to sign JWT with private key.');
                return null;
            }
            $signature = $this->base64UrlEncode($signature);

            $jwt = "$header.$payload.$signature";

            try {
                $response = Http::asForm()->post('https://oauth2.googleapis.com/token', [
                    'grant_type' => 'urn:ietf:params:oauth:grant-type:jwt-bearer',
                    'assertion'  => $jwt,
                ]);

                if ($response->successful()) {
                    return $response->json('access_token');
                }

                Log::error('FCM Token exchange failed: ' . $response->body());
            } catch (\Exception $e) {
                Log::error('FCM Token request error: ' . $e->getMessage());
            }

            return null;
        });
    }

    /**
     * Send FCM push notification to a device token.
     */
    public function send(string $deviceToken, string $title, string $body, array $data = []): array
    {
        $accessToken = $this->getAccessToken();
        if (! $accessToken) {
            return [
                'success' => false,
                'message' => 'FCM Access Token not available.',
                'status'  => 500
            ];
        }

        $projectId = config('services.fcm.project_id');
        if (! $projectId) {
            Log::warning('FCM Project ID is not configured.');
            return [
                'success' => false,
                'message' => 'FCM Project ID is not configured.',
                'status'  => 500
            ];
        }

        // Format data values as strings (FCM HTTP v1 requires data values to be strings)
        $stringData = [];
        foreach ($data as $key => $value) {
            $stringData[$key] = (string) $value;
        }

        $payload = [
            'message' => [
                'token' => $deviceToken,
                'notification' => [
                    'title' => $title,
                    'body'  => $body,
                ],
                'data' => $stringData,
                'android' => [
                    'priority' => 'high',
                ],
            ]
        ];

        try {
            $url = "https://fcm.googleapis.com/v1/projects/{$projectId}/messages:send";
            $response = Http::withToken($accessToken)
                ->post($url, $payload);

            if ($response->successful()) {
                return [
                    'success' => true,
                    'message' => 'Notification sent successfully.',
                    'response'=> $response->json(),
                    'status'  => $response->status()
                ];
            }

            return [
                'success' => false,
                'message' => 'FCM send failed: ' . $response->body(),
                'status'  => $response->status()
            ];
        } catch (\Exception $e) {
            Log::error('FCM request exception: ' . $e->getMessage());
            return [
                'success' => false,
                'message' => 'FCM request exception: ' . $e->getMessage(),
                'status'  => 500
            ];
        }
    }

    /**
     * Base64 URL encode.
     */
    private function base64UrlEncode(string $data): string
    {
        return str_replace(['+', '/', '='], ['-', '_', ''], base64_encode($data));
    }
}
```

## Step 5: Queued Job

Buat `app/Jobs/KirimFcmNotifikasiJob.php`:

```php
<?php

namespace App\Jobs;

use App\Services\FcmService;
use App\Models\UserDevice;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Exception;

class KirimFcmNotifikasiJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries   = 3;
    public int $timeout = 30;

    public function __construct(
        public readonly int    $userId,
        public readonly string $title,
        public readonly string $body,
        public readonly array  $data = [],
    ) {}

    public function handle(FcmService $fcmService): void
    {
        $devices = UserDevice::where('user_id', $this->userId)->get();

        if ($devices->isEmpty()) {
            return;
        }

        $hasSentAny = false;
        $hasFailedAttempts = false;

        foreach ($devices as $device) {
            $result = $fcmService->send($device->fcm_token, $this->title, $this->body, $this->data);

            if ($result['success']) {
                $hasSentAny = true;
            } else {
                // FCM HTTP v1 returns 404 or 410 when token is unregistered or invalid
                if (isset($result['status']) && ($result['status'] === 404 || $result['status'] === 410)) {
                    $device->delete();
                } else {
                    $hasFailedAttempts = true;
                }
            }
        }

        if ($hasFailedAttempts) {
            throw new Exception('FCM dispatch failed for one or more active devices.');
        }
    }

    public function backoff(): array
    {
        return [60, 300]; // retry: 60s, 5 menit
    }
}
```

> **Opsional — flag `is_sent`:** jika project punya tabel notifikasi in-app dan ingin menandai notifikasi sudah terkirim via push, tambahkan setelah loop (sebelum cek `$hasFailedAttempts`):
>
> ```php
> if ($hasSentAny) {
>     Notifikasi::where('user_id', $this->userId)
>         ->where('judul', $this->title)
>         ->where('pesan', $this->body)
>         ->update(['is_sent' => true]);
> }
> ```
>
> Sesuaikan nama model dan kolom dengan project. Lewati blok ini jika tidak ada tabel notifikasi.

## Step 6: Registrasi token saat login API

Di endpoint login (Sanctum), terima `fcm_token`, `device_id`, `platform` lalu simpan:

```php
$request->validate([
    'email'      => 'required|email',
    'password'   => 'required|string',
    'fcm_token'  => 'nullable|string',
    'device_id'  => 'nullable|string',
    'platform'   => 'nullable|in:android,ios',
]);

// ...setelah autentikasi berhasil:
if ($request->filled('fcm_token')) {
    UserDevice::updateOrCreate(
        ['user_id' => $user->id, 'device_id' => $request->device_id ?? 'default'],
        ['fcm_token' => $request->fcm_token, 'platform' => $request->platform ?? 'android']
    );
}
```

**Wajib juga: endpoint refresh token.** Firebase bisa me-refresh token kapan saja saat user masih login (`onTokenRefresh` di mobile). Tanpa endpoint ini, token di backend basi dan push berhenti sampai user login ulang. Tambahkan route di grup `auth:sanctum`:

```php
Route::post('/auth/fcm-token', [AuthController::class, 'updateFcmToken']);
```

```php
public function updateFcmToken(Request $request): JsonResponse
{
    $request->validate([
        'fcm_token' => 'required|string',
        'device_id' => 'nullable|string',
        'platform'  => 'nullable|in:android,ios',
    ]);

    UserDevice::updateOrCreate(
        ['user_id' => $request->user()->id, 'device_id' => $request->device_id ?? 'default'],
        ['fcm_token' => $request->fcm_token, 'platform' => $request->platform ?? 'android']
    );

    return response()->json(['success' => true, 'message' => 'FCM token diperbarui.']);
}
```

Di sisi mobile, panggil endpoint ini dari listener `FirebaseMessaging.instance.onTokenRefresh` (Flutter) dengan `device_id` yang sama seperti saat login.

Opsional tapi disarankan — hapus device saat logout agar user yang sudah logout tidak menerima push:

```php
public function logout(Request $request): JsonResponse
{
    if ($request->filled('device_id')) {
        UserDevice::where('user_id', $request->user()->id)
            ->where('device_id', $request->device_id)
            ->delete();
    }
    $request->user()->currentAccessToken()->delete();

    return response()->json(['success' => true]);
}
```

## Step 7: Cara dispatch

```php
use App\Jobs\KirimFcmNotifikasiJob;

KirimFcmNotifikasiJob::dispatch(
    $user->id,
    'Judul Notifikasi',
    'Isi pesan notifikasi.',
    ['type' => 'laporan_status', 'laporan_id' => $laporan->id]   // payload data, opsional
);
```

`data` dipakai mobile app untuk routing/deep-link saat notifikasi di-tap. Semua value otomatis di-cast ke string oleh `FcmService` (syarat FCM HTTP v1).

## Step 8: Verifikasi

1. **Tes kredensial** (tanpa perlu device):
   ```bash
   php artisan tinker --execute='cache()->forget("fcm_access_token"); echo app(\App\Services\FcmService::class)->getAccessToken() ? "TOKEN OK" : "GAGAL";'
   ```
   `TOKEN OK` = service account valid dan signing JWT berhasil.
2. **Pastikan worker jalan**: `php artisan queue:work` (atau bagian dari `composer dev`).
3. **End-to-end**: login dari mobile app dengan `fcm_token` asli → cek baris muncul di `user_devices` → dispatch job → notifikasi masuk ke perangkat.
4. Jika gagal, cek `storage/logs/laravel.log` (semua kegagalan FCM di-log) dan tabel `failed_jobs`.

## Gotchas

- **Queue worker wajib jalan.** Job menumpuk diam-diam di tabel `jobs` tanpa error jika worker mati — selalu cek `SELECT COUNT(*) FROM jobs` saat notifikasi "tidak sampai".
- **Value `data` harus string.** FCM HTTP v1 menolak payload dengan value non-string; service ini sudah meng-cast otomatis, jangan dihilangkan.
- **Cache token 3500 detik** (token Google berlaku 3600s, buffer 100s). Setelah ganti file kredensial, jalankan `cache()->forget('fcm_access_token')` atau `php artisan cache:clear`.
- **Path kredensial relatif di-resolve dengan `base_path()`** — jangan pakai `file_exists()` langsung pada path relatif, karena cwd web request berbeda dengan CLI/queue worker.
- **404/410 = token mati**, device dihapus otomatis. Error lain (5xx, network) memicu retry job — jangan ikut menghapus device pada error selain 404/410.
- **Jangan commit file service account JSON.** Untuk production, alternatifnya isi `FIREBASE_CREDENTIALS` langsung dengan JSON satu baris (service ini mendeteksi string yang diawali `{`).
- **Token Firebase bisa berubah saat user masih login.** Endpoint `POST /auth/fcm-token` (Step 6) wajib dipanggil mobile app dari `onTokenRefresh` — registrasi saat login saja tidak cukup.
