# setup-arch

Catatan instalasi Arch Linux + Hyprland + Ambxst, plus integrasi `awww` sebagai external wallpaper renderer (animasi transisi wallpaper) menggantikan render gambar bawaan Ambxst.

---

## 1. Base install

- Install Arch dari ISO resmi: https://archlinux.org/download/
- Install Hyprland:
  ```bash
  sudo pacman -Syu
  sudo pacman -S hyprland
  ```

## 2. Install Ambxst

Dari repo resmi: https://github.com/Axenide/Ambxst

```bash
sudo pacman -S curl
curl -L get.axeni.de/ambxst | sh
ambxst install hyprland
ambxst update
```

## 3. Theme SDDM & SDDM + WPP i use arch btw

Install dari: `<isi link theme kamu di sini>`

sudo cp download
sudo cp 

## 4. Cara nambahin ke direktori wallpaper Ambxst:
```bash
cp namafile.jpg ~/.local/src/ambxst/assets/wallpapers_example/
```

## 5. Cara gunakan Theme rEFInd
rekomen gunakan refind versi gunakan versi **2.2**.
'''bash
sudo pacman -U https://archive.archlinux.org/packages/r/refind/refind-0.14.0.2-2-x86_64.pkg.tar.zst
sudo refind-install

 kunci versi biar gak ke-update otomatis update

 '''bash
 sudo nano /etc/pacman.conf

Tambahin di section [options]:

'''bash
IgnorePkg = refind

---

# custom terminal

*   **Shell**: [Fish Shell](https://fishshell.com/) (/usr/bin/fish)
*   chsh -s /usr/bin/fish
*   reboot app
*   tema shell OhMyFish
*   curl https://raw.githubusercontent.com/oh-my-fish/oh-my-fish/master/bin/install | fish
*   omf install bobthefish
*   omf theme bobthefish
*   **Prompt**: [Starship](https://starship.rs/) (v1.26.0)
*   sudo pacman -S fish starship
*   nano ~/.config/fish/config.fish
*   Tambahkan baris berikut di bagian paling bawah:
Cuplikan kode

starship init fish | source
*   
*   **Theme (Fish/OMF)**: [bobthefish](https://github.com/oh-my-fish/theme-bobthefish)
* Font: JetBrainsMono Nerd Font.
* sudo pacman -S ttf-jetbrains-mono-nerd
* nano ~/.config/kitty/kitty.conf
font_family      JetBrainsMono Nerd Font
bold_font        auto
italic_font      auto
bold_italic_font auto
font_size        12.0



hl.monitor({
    output   = "",
    mode     = "preferred",
    position = "auto",
    scale    = "1",
})

#####

decoration = {
    rounding       = 10,
    rounding_power = 2,
    
    -- Change transparency of focused and unfocused windows
    active_opacity   = 0.1,
    inactive_opacity = 0.1,
    
    shadow = {
        enabled      = true,
        range        = 4,
        render_power = 3,
        color        = 0xee1a1a1a,
    },
    
    blur = {
        enabled   = true,
        size      = 6,
        passes    = 3,
        vibrancy  = 0.1696,
    },
},

animations = {
    enabled = true,
},
})

######


hl.config({
    misc = {
        force_default_wallpaper = 0,    -- Set to 0 or 1 to disable the anime mascot wallpapers
            disable_hyprland_logo   = true, -- If true disables the random hyprland logo / anime girl background. :(
    },
})

#######

-- Hanya untuk aplikasi (Kitty, Dolphin, Browser, dll)
hl.window_rule({
    name = "app-transparency",
    match = { class = ".*" }, -- Mengambil semua window aplikasi
    opacity = 0.8,
})

## 6. Animasi wallpaper pakai `awww`

Ambxst secara default render wallpaper-nya sendiri lewat `Image` QML (gambar statis) dan `mpvpaper` (video/gif) di dalam `PanelWindow` yang menempel di `WlrLayer.Background`. Transisi bawaannya cuma fade+zoom pendek dan gak bisa random beda-beda gaya tiap ganti.

`awww` adalah penerus `swww` (`swww` sudah tidak dikembangkan lagi, project-nya di-rename & dipindah ke Codeberg: https://codeberg.org/LGFae/awww). `awww` punya wallpaper daemon sendiri dengan transisi native (`fade`, `wipe`, `wave`, `grow`, `center`, `outer`, `any`, `random`, dll) yang jauh lebih variatif daripada implementasi QML manual.

Integrasi ini membuat Ambxst **hanya jadi otak** (state wallpaper, matugen theming, dashboard picker), sementara render visual & transisi wallpaper gambar/gif didelegasikan ke `awww-daemon`. Video wallpaper tetap lewat `mpvpaper` seperti biasa (tidak diubah).

### 6.1 Install awww

```bash
yay -S awww          # stable
# atau versi paling baru:
yay -S awww-git
```

Opsional, symlink kompatibilitas kalau ada script lain yang masih manggil `swww`:
```bash
sudo ln -sf /usr/bin/awww /usr/bin/swww
sudo ln -sf /usr/bin/awww-daemon /usr/bin/swww-daemon
```

### 6.2 Autostart daemon + set wallpaper awal

File: `hyprland.lua` (config generator Hyprland kamu, bagian `AUTOSTART`).

```lua
-------------------
---- AUTOSTART ----
-------------------
hl.on("hyprland.start", function ()
    -- Pastikan daemon lama mati dulu
    os.execute("killall awww-daemon")
    os.execute("rm -f $XDG_RUNTIME_DIR/wayland-1-awww-daemon.sock")

    -- Jalankan daemon baru + set wallpaper awal, dibackground biar gak nge-block startup
    os.execute("(awww-daemon && sleep 2 && awww img ~/.local/src/ambxst/assets/wallpapers_example/NAMA_FILE.png) &")
end)
```

Ganti `NAMA_FILE.png` dengan file yang benar-benar ada (cek dengan `ls ~/.local/src/ambxst/assets/wallpapers_example`).

### 6.3 Patch `Wallpaper.qml`

File yang diedit:
```
~/.local/src/ambxst/modules/widgets/dashboard/wallpapers/Wallpaper.qml
```

**a) Nonaktifkan render visual bawaan untuk gambar/gif**

Cari blok:
```qml
    Rectangle {
        id: background
        anchors.fill: parent
        color: "black"
        focus: true

        Keys.onLeftPressed: { ... }
        Keys.onRightPressed: { ... }

        WallpaperImage {
            id: wallImage
            anchors.fill: parent
            source: wallpaper.effectiveWallpaper
        }
    }
```

Ganti jadi:
```qml
    Rectangle {
        id: background
        anchors.fill: parent
        color: "transparent"
        focus: true

        Keys.onLeftPressed: {
            if (wallpaper.wallpaperPaths.length > 0) wallpaper.previousWallpaper();
        }
        Keys.onRightPressed: {
            if (wallpaper.wallpaperPaths.length > 0) wallpaper.nextWallpaper();
        }

        // Cuma aktif buat gif/video (mpvpaper). Gambar statis udah dihandle awww.
        Loader {
            anchors.fill: parent
            active: wallpaper.getFileType(wallpaper.effectiveWallpaper) !== 'image'
            sourceComponent: WallpaperImage {
                id: wallImage
                source: wallpaper.effectiveWallpaper
            }
        }
    }
```

`component WallpaperImage: Item { ... }` di bawahnya **tidak perlu diubah** — sekarang cuma kepakai untuk gif/video karena di-guard `active` di atas.

**b) Tambahkan fungsi pemanggil awww**

Cari blok (masih di scope `PanelWindow { id: wallpaper ... }`, bukan di dalam `component WallpaperImage`):
```qml
    Process {
        id: applyPresetProcess
        running: false
        command: []

        onExited: code => {
            if (code === 0)
                console.log("Color preset applied successfully");
            else
                console.warn("Failed to apply color preset, code:", code);
        }
    }
```

Tambahkan setelahnya:
```qml
    // --- awww integration ---
    property string awwwBin: "awww"
    readonly property var awwwTransitions: ["fade", "wipe", "wave", "grow", "center", "outer", "any", "left", "right", "top", "bottom"]

    function applyExternalWallpaper(path, outputName) {
        if (!path)
            return;
        var fileType = getFileType(path);
        if (fileType !== 'image' && fileType !== 'gif')
            return; // video tetap lewat mpvpaper

        var t = awwwTransitions[Math.floor(Math.random() * awwwTransitions.length)];
        var cmd = [awwwBin, "img"];
        if (outputName)
            cmd.push("-o", outputName);
        cmd.push(path, "--transition-type", t, "--transition-fps", "60", "--transition-duration", "1.2");

        awwwApplyProcess.command = cmd;
        awwwApplyProcess.running = true;
    }

    Process {
        id: awwwApplyProcess
        running: false
        onExited: code => {
            if (code !== 0)
                console.warn("awww apply gagal, kode:", code);
        }
    }
```

**c) Hook ke fungsi ganti wallpaper yang sudah ada**

Ada 4 titik di file yang sama yang harus dipanggil `applyExternalWallpaper(...)`, masing-masing di fungsi **asli** (bukan bikin fungsi baru):

- `setWallpaper(path, targetScreen)` — 2 titik (per-screen dan global fallback)
- `nextWallpaper()`
- `previousWallpaper()`
- `setWallpaperByIndex(index)`

Contoh pola perubahan (`nextWallpaper()`):
```qml
function nextWallpaper() {
    ...
    currentIndex = (currentIndex + 1) % wallpaperPaths.length;
    currentWallpaper = wallpaperPaths[currentIndex];
    wallpaperConfig.adapter.currentWall = wallpaperPaths[currentIndex];
    applyExternalWallpaper(currentWallpaper, null);   // <-- baris baru
    runMatugenForCurrentWallpaper();
    generateLockscreenFrame(wallpaperPaths[currentIndex]);
}
```

Pola yang sama untuk `previousWallpaper()` dan `setWallpaperByIndex()` (letakkan `applyExternalWallpaper(currentWallpaper, null)` tepat setelah baris `wallpaperConfig.adapter.currentWall = ...`).

Untuk `setWallpaper(path, targetScreen)`, panggil dengan `targetScreen` sebagai argumen kedua (bukan `null`), supaya `awww` menerapkan wallpaper ke output/monitor yang benar:
```qml
applyExternalWallpaper(path, targetScreen);
```

### 6.4 Verifikasi

```bash
# cek daemon jalan
pgrep -a awww-daemon

# tes manual set wallpaper + transisi random
awww img ~/.local/src/ambxst/assets/wallpapers_example/NAMA_FILE.png --transition-type random --transition-fps 60 --transition-duration 1.2
```

Kalau berhasil ganti dengan transisi, buka dashboard Ambxst (wallpaper picker) dan ganti wallpaper dari sana — sekarang animasinya seharusnya lewat `awww` dengan gaya transisi yang random tiap kali ganti, sementara wallpaper video tetap berjalan seperti biasa lewat `mpvpaper`.
