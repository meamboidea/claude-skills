---
name: tomselect-livewire
description: >-
  Implementasi Tom Select v2 pada project Livewire Volt + Alpine.js. Gunakan
  skill ini untuk menambahkan Tom Select ke komponen select — baik filter
  topbar maupun form select di dalam modal. Menangani wire:ignore, sync
  dua arah ke $wire, CSS override tema dark/light, dan menghindari ParseError
  Volt. Trigger: saat user meminta "gunakan tomselect", "tambah tomselect",
  "tom select untuk select", atau meminta upgrade komponen select native.
metadata:
  author: jagakolakaweb
  version: "1.0"
---

# Tom Select + Livewire Volt + Alpine.js

Kamu adalah spesialis integrasi Tom Select v2 pada stack Laravel Livewire Volt + Alpine.js.
Ikuti pola yang sudah teruji ini dengan tepat — ada jebakan spesifik yang wajib dihindari.

## Langkah 1 — Cek dan pasang CDN

Buka file layout utama (biasanya `resources/views/layouts/admin.blade.php` atau sejenisnya).
Tambahkan di dalam `<head>`, **sebelum** `@livewireStyles`:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/tom-select@2/dist/css/tom-select.css">
<script src="https://cdn.jsdelivr.net/npm/tom-select@2/dist/js/tom-select.complete.min.js"></script>
```

Jika sudah ada, lewati langkah ini.

## Langkah 2 — Tambah CSS override global

Tambahkan blok `<style>` di layout **setelah** baris CDN Tom Select di atas.
Sesuaikan nilai dengan CSS variables yang digunakan project (cek `:root` di layout):

```html
<style>
    /* ═══ Tom Select theme ═══ */
    .ts-wrapper { font-family: inherit; font-size: 0.82rem; }
    .ts-control {
        padding: 0.48rem 2rem 0.48rem 0.75rem;
        border: 1px solid var(--border-input);
        border-radius: 0.5rem;
        background: var(--bg-card);
        color: var(--text-primary);
        box-shadow: none;
        min-height: unset;
        cursor: pointer;
        transition: border-color 0.15s, box-shadow 0.15s;
    }
    .ts-wrapper.focus .ts-control {
        border-color: rgba(59,130,246,0.5);
        box-shadow: 0 0 0 3px rgba(59,130,246,0.1);
    }
    .ts-control .item { color: var(--text-primary); padding: 0; margin: 0; background: transparent; border: none; }
    .ts-control > input { color: var(--text-primary); font-size: 0.82rem; font-family: inherit; }
    .ts-control > input::placeholder { color: var(--text-dim); }
    .ts-wrapper.single .ts-control::after {
        border-color: var(--text-muted) transparent transparent;
        right: 0.75rem; margin-top: -2px; border-width: 4px 4px 0;
    }
    .ts-wrapper.single.dropdown-active .ts-control::after {
        border-color: transparent transparent var(--text-muted);
        border-width: 0 4px 4px; margin-top: -6px;
    }
    .ts-dropdown {
        background: var(--bg-sidebar);
        border: 1px solid var(--border);
        border-radius: 0.5rem;
        box-shadow: 0 8px 24px rgba(0,0,0,0.3);
        font-size: 0.82rem; font-family: inherit; margin-top: 2px;
    }
    html[data-theme="light"] .ts-dropdown { box-shadow: 0 8px 24px rgba(15,23,42,0.12); }
    .ts-dropdown .ts-dropdown-content { padding: 0.25rem; max-height: 220px; }
    .ts-dropdown .option {
        border-radius: 0.375rem;
        padding: 0.45rem 0.75rem;
        color: var(--text-primary);
        transition: background 0.1s;
    }
    .ts-dropdown .option:hover,
    .ts-dropdown .option.active { background: var(--accent-hover); color: var(--accent-text); }
    .ts-dropdown .option.selected { background: rgba(59,130,246,0.12); color: var(--accent-text); }
    .ts-dropdown .option.selected.active { background: var(--accent-hover); }
    .ts-dropdown .no-results { padding: 0.5rem 0.75rem; color: var(--text-dim); font-size: 0.8rem; text-align: center; }
    /* Form select variant (di dalam modal) */
    .ts-form-input .ts-control { background: var(--bg-body); border-radius: 0.45rem; padding: 0.5rem 2rem 0.5rem 0.75rem; }
</style>
```

## Langkah 3 — Identifikasi select yang perlu diubah

Baca file komponen Volt target. Cari semua elemen `<select>`. Tentukan jenisnya:

- **Filter select** (di topbar/header, biasanya `wire:model.live`) → gunakan Pola A
- **Form select** (di dalam modal `@if($showModal)`, biasanya `wire:model`) → gunakan Pola B

## Langkah 4A — Pola Filter Select (Topbar)

### HTML — ganti `<select wire:model...>` dengan:

```html
<div class="ts-filter-wrap" wire:ignore x-data="tsFilter__NAMA__()">
    <select x-ref="sel">
        <option value="">Semua ...</option>
        @foreach($list as $item)
            <option value="{{ $item->id }}">{{ $item->nama }}</option>
        @endforeach
    </select>
</div>
```

Ganti `__NAMA__` dengan nama unik (misal: `Kecamatan`, `Kategori`).

### Script — tambahkan di akhir komponen, SEBELUM `<style>`, SETELAH `</div>` penutup:

```html
<script>
    if (!window.tsFilter__NAMA__) {
        window.tsFilter__NAMA__ = function () {
            return {
                ts: null,
                init() {
                    var self = this;
                    this.ts = new TomSelect(this.$refs.sel, {
                        create: false,
                        allowEmptyOption: true,
                        maxOptions: null,
                        placeholder: 'Semua ...',
                        dropdownParent: 'body'
                    });
                    this.ts.clear(true); // tampilkan placeholder di awal
                    this.ts.on('change', function (val) {
                        if (val === '') self.ts.clear(true);
                        self.$wire.set('namaLivewireProperty', val);
                    });
                    this.$watch('$wire.namaLivewireProperty', function (val) {
                        var v = String(val != null ? val : '');
                        if (v === '') {
                            if (self.ts.getValue() !== '') self.ts.clear(true);
                        } else if (self.ts.getValue() !== v) {
                            self.ts.setValue(v, true);
                        }
                    });
                }
            };
        };
    }
</script>
```

### CSS tambahan (di blok `<style>` komponen):

```css
.ts-filter-wrap { max-width: 200px; min-width: 150px; }
```

## Langkah 4B — Pola Form Select (Modal)

Modal pakai `@if($showModal)` → DOM dibuat ulang tiap buka → Alpine `init()` otomatis berjalan ulang. Ini perilaku yang benar.

### HTML — ganti `<select wire:model="propertyId" ...>` dengan:

```html
<div class="ts-form-input" wire:ignore x-data="tsForm__NAMA__('{{ $propertyId }}')">
    <select x-ref="sel">
        <option value="">— Pilih ... —</option>
        @foreach($list as $item)
            <option value="{{ $item->id }}">{{ $item->nama }}</option>
        @endforeach
    </select>
</div>
@error('propertyId') <span class="form-error">{{ $message }}</span> @enderror
```

`{{ $propertyId }}` → nilai awal PHP untuk mode edit. Taruh `@error` di **luar** div `wire:ignore`.

### Script:

```html
<script>
    if (!window.tsForm__NAMA__) {
        window.tsForm__NAMA__ = function (initialValue) {
            return {
                ts: null,
                init() {
                    var self = this;
                    this.ts = new TomSelect(this.$refs.sel, {
                        create: false,
                        allowEmptyOption: true,
                        maxOptions: null,
                        dropdownParent: 'body'
                    });
                    if (initialValue) this.ts.setValue(String(initialValue), true);
                    this.ts.on('change', function (val) {
                        self.$wire.set('propertyId', val);
                    });
                }
            };
        };
    }
</script>
```

## Aturan wajib — jangan dilanggar

| Aturan | Alasan |
|--------|--------|
| **JANGAN** tulis logika JS langsung di `x-data="{ ... }"` multi-baris | Volt parser salah baca `{ }` → `ParseError: unexpected token "protected"` di compiled class |
| **SELALU** bungkus select dengan `wire:ignore` | Livewire morph menghancurkan Tom Select instance |
| **SELALU** gunakan `dropdownParent: 'body'` | Modal punya `overflow: hidden` yang memotong dropdown |
| **SELALU** beri nama unik pada `window.tsFungsi__NAMA__` | Mencegah konflik jika ada banyak Tom Select di satu halaman |
| `@error()` taruh di **luar** `wire:ignore` | Div `wire:ignore` tidak diupdate Livewire, pesan error tidak akan muncul |

## Checklist akhir

- [ ] CDN Tom Select sudah ada di `<head>` layout
- [ ] CSS override global sudah ada di layout
- [ ] `wire:ignore` ada di setiap wrapper div Tom Select
- [ ] `dropdownParent: 'body'` ada di setiap konfigurasi
- [ ] Nama fungsi `window.tsXxx` unik per select
- [ ] Script block ada di akhir komponen (setelah `</div>` root, sebelum `<style>`)
- [ ] `@error()` di luar `wire:ignore`
- [ ] Filter select: ada `placeholder` + `this.ts.clear(true)` setelah init + `$watch`
- [ ] Form select: pass nilai awal lewat parameter `'{{ $propertyId }}'`
