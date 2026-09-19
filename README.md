<div align="center">

#  EasyTier Manager

**Cross-distro installer & manager for [EasyTier](https://github.com/EasyTier/EasyTier) with TOML config, multi-instance support, and a friendly whiptail UI.**

[![Platform](https://img.shields.io/badge/platform-Debian%20%7C%20RHEL%20%7C%20Arch%20%7C%20openSUSE-blue)](#-supported-platforms)
[![Shell](https://img.shields.io/badge/shell-bash-4EAA25?logo=gnubash&logoColor=white)](#)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![EasyTier](https://img.shields.io/badge/EasyTier-Latest-orange)](https://github.com/EasyTier/EasyTier/releases)

[English](#-english) · [فارسی](#-فارسی) · [Report Bug](../../issues) · [Request Feature](../../issues)

</div>

---

## 🇬🇧 English

### 📖 Overview

**EasyTier Manager** is a single-file Bash script that installs, configures, and manages [EasyTier](https://github.com/EasyTier/EasyTier) across all major Linux distributions. It uses a whiptail-based interactive menu, writes clean **TOML configuration files**, and supports **multiple instances**, **GitHub proxy**, and **20+ advanced EasyTier features** out of the box.

No more copy-pasting long `ExecStart` lines into systemd. No more guessing flags. Just run it.

---

### ✨ Features

| Feature | Description |
|---|---|
| 🌍 **Cross-distro** | Debian/Ubuntu, RHEL/CentOS/Fedora, Arch/Manjaro, openSUSE |
| 🧩 **Auto dependencies** | whiptail, curl, unzip, openssl, bc — installed automatically |
| 🎛️ **Whiptail UI** | Clean menu-driven interface with inputbox / yesno / checklist |
| 📝 **TOML config** | Uses `easytier-core -c config.toml` instead of a 200-char `ExecStart` |
| 🏢 **Multi-instance** | Run several EasyTier nodes on one machine via `easytier@<name>.service` |
| 🌐 **GitHub Proxy** | Built-in proxy support (e.g. `https://ghfast.top/`) for fast downloads in IR/CN |
| 🔒 **Network Secret** | Auto-generated random secret, editable in menu |
| 📊 **Watchdog** | Auto-restart on latency spikes or ping failure |
| ⏰ **Cron scheduling** | Periodic restart at configurable intervals |
| 🧠 **Arch-aware** | Auto-detects `x86_64`, `aarch64`, `armv7hf`, `mips`, `mipsel` |
| 🧰 **`et` wrapper** | Run `et` from anywhere to reopen the menu |
| 🔐 **Secure** | Config files with `chmod 600`, secret stored securely |

---

### 🚀 Supported Platforms

| Distribution | Package Manager | Status |
|---|---|---|
| Ubuntu / Debian / Mint | `apt` | ✅ Tested |
| RHEL / CentOS / Rocky / Alma | `dnf` / `yum` | ✅ Tested |
| Fedora | `dnf` | ✅ Tested |
| Arch / Manjaro / EndeavourOS | `pacman` | ✅ Tested |
| openSUSE Leap / Tumbleweed | `zypper` | ✅ Tested |

**Architectures:** `x86_64`, `aarch64`, `armv7hf`, `arm`, `mips`, `mipsel`

**Init system:** systemd (OpenRC support planned)

---

### 📦 Installation

```bash
# 1. Download the script
curl -O https://raw.githubusercontent.com/bshnetwork/easytier/main/bsheasytier.sh

# 2. Make it executable
chmod +x bsheasytier.sh

# 3. Run as root
sudo ./bsheasytier.sh
```

On first run the script will:
1. Detect your distro, package manager, and CPU architecture
2. Install missing dependencies (`whiptail`, `curl`, `unzip`, `openssl`, `bc`)
3. Ask if you want to install EasyTier → choose **Yes**
4. Drop you into the main menu

After installation, the **`et`** command is available system-wide:

```bash
sudo et
```

---

### 🧭 Menu Overview

```
┌──────────────────────────────────────────────┐
│  Latest: vX.Y.Z      Core status: Installed  │
├──────────────────────────────────────────────┤
│  [1]  Connect to the Tunnel Network          │
│  [2]  Display Peers                          │
│  [3]  Display Routes                         │
│  [4]  Peer-Center                            │
│  [5]  Display Secret Key                     │
│  [6]  View Service Status                    │
│  [7]  Set Watchdog (Auto-Restarter)          │
│  [8]  Cron-job setting                       │
│  [9]  Restart Service                        │
│  [10] Remove Service                         │
│  [11] Remove Core                            │
│  [12] View / Edit Config (TOML)              │
│  [13] Multi-Instance Management              │
│  [14] GitHub Proxy Settings                  │
│  [0]  Exit                                   │
└──────────────────────────────────────────────┘
```

---

### ⚙️ Advanced Configuration Wizard

The **Connect to the Tunnel Network** wizard walks you through:

- **IP mode** — `static` / `dhcp` / `no-tun`
- **Network identity** — name + secret (random by default)
- **Protocol** — `tcp`, `udp`, `ws`, `wss`, `quic`
- **Peer discovery** — manual list, public shared nodes (`-e`), or reverse mode
- **Encryption** toggle
- **IPv6** toggle
- **Multi-thread** toggle
- **Latency-first** mode (gaming/VoIP)
- **Exit node** mode
- **Custom MTU**
- **Subnet proxy** (`--proxy-networks`, with CIDR mapping)
- **SOCKS5** server
- **No-listener** mode (client-only)
- **WireGuard VPN Portal**
- **File logging**
- **Config Server** (central management)
- **Relay whitelist**

All of these end up in a clean `/etc/easytier/config.toml`, and the systemd unit becomes:

```ini
ExecStart=/etc/easytier/easytier-core -c /etc/easytier/config.toml
```

---

### 📂 Directory Layout

```
/etc/easytier/
├── easytier-core              # core binary
├── easytier-cli               # CLI binary
├── config.toml                # main config (chmod 600)
├── configs/
│   ├── instance-1.toml        # additional instance configs
│   └── instance-2.toml
└── .ghproxy                   # saved GitHub proxy URL
```

---

### 🛠️ Common Tasks

**Change the network secret**
Menu → `[12] View / Edit Config` → edit `network_secret`.

**Add a second node on the same server**
Menu → `[13] Multi-Instance Management` → `Create new instance`.

**Speed up downloads from GitHub (Iran/China)**
Menu → `[14] GitHub Proxy Settings` → enter e.g. `https://ghfast.top/`.

**Reset everything**
Menu → `[10] Remove Service`, then `[11] Remove Core`.

---

### 🧪 Manual Config Example

If you prefer editing the TOML directly:

```toml
instance_name = "default"
hostname = "my-server"
ipv4 = "10.17.17.1"

listeners = [
  "udp://0.0.0.0:2090",
  "udp://[::]:2090",
]
rpc_portal = "127.0.0.1:15888"

[network_identity]
network_name = "my-net"
network_secret = "a1b2c3d4e5f6"

[[peer]]
uri = "udp://public.easytier.top:11010"

[flags]
default_protocol = "udp"
enable_encryption = true
enable_ipv6 = true
latency_first = true
```

Then:
```bash
systemctl restart easytier.service
```

---

### 🤝 Contributing

Pull requests are welcome. For major changes, please open an issue first.

1. Fork the repo
2. Create your branch: `git checkout -b feature/amazing`
3. Commit: `git commit -m 'Add amazing feature'`
4. Push: `git push origin feature/amazing`
5. Open a Pull Request

---

### 📜 License

Released under the **MIT License**. See [LICENSE](LICENSE) for details.

---

### 🙏 Acknowledgments

- [EasyTier](https://github.com/EasyTier/EasyTier) — the awesome VPN mesh project
- [onekeyeasytier](https://github.com/EasyTier/onekeyeasytier) — inspiration for UX
- The whiptail / newt community

---

## 🇮🇷 فارسی

### 📖 درباره پروژه

**EasyTier Manager** یک اسکریپت تک‌فایل Bash است که [EasyTier](https://github.com/EasyTier/EasyTier) را روی تمام توزیع‌های اصلی لینوکس نصب، تنظیم و مدیریت می‌کند. این اسکریپت از رابط کاربری whiptail استفاده می‌کند، فایل‌های کانفیگ تمیز **TOML** می‌سازد و از **چند نمونه (Multi-Instance)**، **پروکسی GitHub** و **بیش از ۲۰ قابلیت پیشرفته** پشتیبانی می‌کند.

دیگر نیازی به کپی‌کردن خطوط طولانی `ExecStart` در systemd نیست. فقط اجرا کن.

---

### ✨ امکانات

| امکان | توضیح |
|---|---|
| 🌍 **چند توزیعی** | Debian/Ubuntu، RHEL/CentOS/Fedora، Arch/Manjaro، openSUSE |
| 🧩 **نصب خودکار وابستگی‌ها** | whiptail، curl، unzip، openssl، bc به‌صورت خودکار |
| 🎛️ **رابط whiptail** | منوی تمیز با ورودی/بله‌خیر/چک‌لیست |
| 📝 **کانفیگ TOML** | استفاده از `easytier-core -c config.toml` به‌جای خط فرمان طولانی |
| 🏢 **Multi-Instance** | اجرای چند نود روی یک سرور با `easytier@<name>.service` |
| 🌐 **پروکسی GitHub** | پشتیبانی داخلی از پروکسی برای دانلود سریع در ایران |
| 🔒 **رمز شبکه** | تولید خودکار رمز تصادفی، قابل ویرایش |
| 📊 **Watchdog** | ری‌استارت خودکار در صورت افزایش تأخیر یا قطعی |
| ⏰ **زمان‌بندی Cron** | ری‌استارت دوره‌ای با بازه دلخواه |
| 🧠 **تشخیص معماری** | `x86_64`، `aarch64`، `armv7hf`، `mips`، `mipsel` |
| 🧰 **دستور `et`** | اجرای منو از هر جای سیستم با `et` |
| 🔐 **امن** | کانفیگ با `chmod 600`، رمز در محل امن |

---

### 🚀 توزیع‌های پشتیبانی‌شده

| توزیع | مدیر بسته | وضعیت |
|---|---|---|
| Ubuntu / Debian / Mint | `apt` | ✅ تست‌شده |
| RHEL / CentOS / Rocky / Alma | `dnf` / `yum` | ✅ تست‌شده |
| Fedora | `dnf` | ✅ تست‌شده |
| Arch / Manjaro / EndeavourOS | `pacman` | ✅ تست‌شده |
| openSUSE Leap / Tumbleweed | `zypper` | ✅ تست‌شده |

**معماری‌ها:** `x86_64`، `aarch64`، `armv7hf`، `arm`، `mips`، `mipsel`

**Init:** systemd (پشتیبانی OpenRC در برنامه آینده)

---

### 📦 نصب

```bash
# ۱. دانلود اسکریپت
curl -O https://raw.githubusercontent.com/<your-user>/<your-repo>/main/bsheasytier.sh

# ۲. اجرایی کردن
chmod +x bsheasytier.sh

# ۳. اجرا با دسترسی root
sudo ./bsheasytier.sh
```

در اولین اجرا:
1. توزیع، مدیر بسته و معماری CPU شناسایی می‌شود
2. وابستگی‌های لازم نصب می‌شوند
3. سوال نصب EasyTier پرسیده می‌شود → **Yes**
4. وارد منوی اصلی می‌شوی

بعد از نصب، دستور **`et`** در کل سیستم در دسترس است:

```bash
sudo et
```

---

### ⚙️ ویزارد تنظیمات پیشرفته

ویزارد **Connect to the Tunnel Network** تمام این موارد را از شما می‌پرسد:

- **حالت IP** — `static` / `dhcp` / `no-tun`
- **هویت شبکه** — نام + رمز (پیش‌فرض: تصادفی)
- **پروتکل** — `tcp`، `udp`، `ws`، `wss`، `quic`
- **حالت Peer** — دستی، عمومی (`-e`)، یا Reverse
- **رمزنگاری**
- **IPv6**
- **Multi-thread**
- **Latency-first** (گیمینگ/VoIP)
- **Exit Node**
- **MTU سفارشی**
- **Subnet Proxy** (با امکان mapping)
- **SOCKS5**
- **No-listener** (فقط مصرف‌کننده)
- **VPN Portal (WireGuard)**
- **لاگ فایل**
- **Config Server** (مدیریت متمرکز)
- **Relay Whitelist**

همه‌ی این‌ها در `/etc/easytier/config.toml` نوشته می‌شوند و فایل سرویس سیستم‌دی این‌قدر ساده می‌شود:

```ini
ExecStart=/etc/easytier/easytier-core -c /etc/easytier/config.toml
```

---

### 🛠️ کارهای رایج

**تغییر رمز شبکه**
منو → `[12] View / Edit Config` → ویرایش `network_secret`.

**افزودن نود دوم روی همان سرور**
منو → `[13] Multi-Instance Management` → `Create new instance`.

**تسریع دانلود از GitHub**
منو → `[14] GitHub Proxy Settings` → مثلاً `https://ghfast.top/`.

**پاک‌سازی کامل**
منو → `[10] Remove Service` و سپس `[11] Remove Core`.

---


### 📜 مجوز

منتشرشده تحت **مجوز MIT**. جزئیات در [LICENSE](LICENSE).

---

### 🙏 تقدیر و تشکر

- [EasyTier](https://github.com/EasyTier/EasyTier)
- [onekeyeasytier](https://github.com/EasyTier/onekeyeasytier)
- جامعه whiptail / newt

</div>
```
