# Custom Commands Addon — Minecraft Bedrock

Addon Minecraft Bedrock yang menambahkan command custom untuk kebutuhan server survival/RPG: sistem ekonomi, home, market yang dilindungi penuh, jual-beli, rank player, sampai custom enchant di atas batas level vanilla.

Dibuat 100% pakai **Script API resmi Minecraft** (`@minecraft/server` & `@minecraft/server-ui`) — tanpa mod eksternal, tanpa modifikasi file game.

## ✨ Fitur

| Command | Akses | Fungsi |
|---|---|---|
| `/saldo` | Semua player | Lihat saldo dari scoreboard `money` |
| `/sethome` | Semua player | Tandai lokasi rumah |
| `/home` | Semua player | Teleport ke rumah (cooldown 5 detik setelah kena damage) |
| `/setmarket` | Operator saja | Tandai lokasi market |
| `/market` | Semua player | Teleport ke market |
| `/sell` | Semua player | Buka menu jual barang |
| `/rate <item> <harga>` | Operator saja | Atur harga jual sebuah item |
| `/crn <player> <rank>` | Operator saja | Ubah rank/tag nama seorang player |
| `/cenchant <player> <jenis> <level 1-10>` | Operator saja | Custom enchant ke item yang dipegang |
| `/cbook <player> <jenis> <level 1-10>` | Operator saja | Beri buku custom enchant langsung |
| `/online` | Semua player | Lihat daftar player online |

**Proteksi market** (radius 20 blok dari titik `/setmarket`): anti hancur manual, anti taruh block, anti ledakan, auto-bersihkan mob berbahaya, auto-padamkan api.

**Custom enchant** bisa naik sampai level 10 (angka romawi X), jauh di atas batas vanilla Minecraft (misal Sharpness biasanya maksimal V). 14 jenis enchant sudah punya efek bonus nyata: sharpness, smite, bane of arthropods, knockback, punch, power, protection, feather falling, thorns, efficiency, unbreaking, fortune, looting, respiration.

## 📦 Instalasi

1. Download file `.mcpack` dari [Releases](../../releases)
2. Buka file tersebut — otomatis ter-import ke Minecraft
3. Saat membuat/mengedit world: aktifkan pack ini di **Behavior Packs**
4. Di tab **Experiments**, aktifkan **"Beta APIs"** (wajib, addon tidak akan jalan tanpa ini)
5. Mainkan world-nya

**Requirement:** Minecraft Bedrock versi **1.21.100** ke atas.

## 🛠️ Development

Struktur project:
```
.
├── manifest.json          # Metadata & dependency behavior pack
├── pack_icon.png           # Icon pack
└── scripts/
    └── main.js             # Seluruh logika addon (single file)
```

Kalau mau modifikasi:
1. Edit `scripts/main.js`
2. Ubah versi di `manifest.json` (`header.version` dan `modules[0].version`) setiap kali update, supaya Minecraft mau memuat ulang pack
3. Zip ulang folder ini (isinya langsung, bukan foldernya) dan ganti ekstensi jadi `.mcpack`

## ⚠️ Batasan yang perlu diketahui

- Level enchant di atas batas vanilla **bukan level resmi engine Minecraft** — itu murni dihitung & diterapkan lewat script sebagai bonus tambahan (damage extra, dsb). Level asli di engine tetap terkunci di angka maksimal vanilla.
- Command sengaja dinamai `/cenchant` (bukan `/enchant`) karena Minecraft Bedrock sudah punya command bawaan `/enchant` sendiri yang akan selalu menang jika nama bentrok.
- Proteksi api di market dicek berkala tiap 5 detik (bukan realtime instan) demi performa server.
- Fortune & Looting bekerja dengan menggandakan item drop di sekitar lokasi kejadian, bukan memodifikasi loot table asli — jadi ada jeda sepersekian detik.

## 📜 Lisensi

MIT — bebas dipakai, dimodifikasi, dan dibagikan ulang.

## 🙏 Kontribusi

Laporan bug atau request fitur silakan buka [Issues](../../issues).

## 🥐Credit
YT: @zii245
