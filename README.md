# mcpe-scoped-storage-fix

Solusi *rootless* untuk mengatasi error `Permission denied` saat menambah, mengedit, atau menghapus file di folder data Minecraft Bedrock (Android 11–16).

## Masalah

Di Android 11 ke atas, sistem menerapkan Scoped Storage yang memblokir akses langsung ke `/Android/data/`. Sebagian game masih bisa dimodifikasi lewat file manager berbasis Shizuku karena foldernya dibuat dengan izin grup `0770` atau `0775`.

Namun, Minecraft Bedrock mengunci folder datanya dengan izin POSIX DAC `0750` (`drwxr-s---`):
- Owner (`u0_a...` / Minecraft): Punya izin baca, tulis, eksekusi (`rwx`).
- Group (`ext_data_rw`): Cuma dikasih izin baca dan masuk (`r-s`), tanpa izin tulis (`w`).

Akibatnya, aplikasi file manager (ZArchiver, MT Manager) maupun shell ADB (`uid=2000`) tetap ditolak (*Permission denied*) saat mencoba *paste* addon, edit config, atau hapus map lama.

## Solusi: DAC Ownership Pre-creation & SGID Inheritance

Folder induk `/files/` memiliki izin `drwxrws---` dengan bit SGID (*Set Group ID*) aktif untuk grup `ext_data_rw`. 

Akalinnya:
1. Kita backup folder `games/` lama.
2. Sebelum Minecraft sempat me-generate foldernya kembali, kita buat ulang folder `games/com.mojang/` beserta seluruh subfoldernya (`minecraftWorlds`, `resource_packs`, `behavior_packs`, dll.) menggunakan user `shell` (QuickShell / ADB).
3. Karena dibuat oleh `shell`, kepemilikan inode (*owner*) otomatis jatuh ke user `shell` dengan izin penuh `drwxrws---`.
4. Karena folder induk berstatus SGID, seluruh subfolder tetap mewarisi grup `ext_data_rw`. Artinya, aplikasi Minecraft tetap bisa membaca dan me-load world/addon tanpa kendala.

## Syarat

- Perangkat Android 11, 12, 13, 14, 15, atau 16
- [Shizuku](https://shizuku.rikka.app/) sudah terpasang dan aktif (via Wireless Debugging).
- Aplikasi terminal yang mendukung Shizuku (misalnya QuickShell atau Termux dengan `rish`).
- MT Manager (disarankan untuk manajemen file sehari-hari).

## Langkah Instalasi / Eksekusi

Jalankan perintah berikut di QuickShell (atau terminal shell ADB) secara berurutan:

### 1. Hentikan Proses Minecraft
Lepas kuncian file database LevelDB sebelum memindahkan data:
```bash
am force-stop com.mojang.minecraftpe
```

### 2. Backup Folder Games Lama
Pindahkan struktur lama agar data tidak hilang:
```bash
mv /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games_backup
```

### 3. Buat Ulang Struktur Folder Baru
Buat folder induk dan semua subfolder penting atas nama user `shell`:
```bash
# Folder induk
mkdir -p /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang

# Subfolder data Minecraft
mkdir /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/minecraftWorlds
mkdir /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/resource_packs
mkdir /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/behavior_packs
mkdir /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/skin_packs
mkdir /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/custom_skins
mkdir /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/development_resource_packs
mkdir /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/development_behavior_packs
mkdir /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/minecraftpe
```

> Penting: Subfolder di atas wajib dibuat sebelum Minecraft dibuka kembali. Jika Minecraft dibuka dalam keadaan folder kosong, sistem game akan men-generate folder tersebut dengan kepemilikan `u0_a...` (terkunci kembali).

### 4. Restore Data Lama dari Backup
Kembalikan map dan pack lama ke folder baru:
```bash
cp -r /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games_backup/com.mojang/minecraftWorlds/* /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/minecraftWorlds/ 2>/dev/null
cp -r /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games_backup/com.mojang/resource_packs/* /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/resource_packs/ 2>/dev/null
cp -r /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games_backup/com.mojang/behavior_packs/* /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/behavior_packs/ 2>/dev/null
cp -r /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games_backup/com.mojang/minecraftpe/* /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/minecraftpe/ 2>/dev/null
```

### 5. Verifikasi
Pastikan izin folder sudah berubah menjadi milik `shell`:
```bash
ls -ld /storage/emulated/0/Android/data/com.mojang.minecraftpe/files/games/com.mojang/*
```

Output yang benar akan menampilkan izin `drwxrws---` dengan user `shell` dan group `ext_data_rw`.

## Selesai
Dengan ini sudah selesai dan siap untuk pengeditan di dalam folder com.mojang, pengeditan dapat dilakukan di aplikasi file manager seperti MT Manager ZArchiver dan aplikasi sejenisnya. (Disarankan menggunakan MT Manager)
1. Buka MT Manager -> Masuk ke Settings -> Pastikan opsi Shizuku sudah aktif.
2. Buka path:
   `Android/data/com.mojang.minecraftpe/files/games/com.mojang/`
3. Kamu bisa langsung copy-paste addon baru, ekstrak world, atau menghapus file tanpa perlu mengetik perintah terminal lagi.
