# PI Server Full Installation Guide
**AVEVA OSIsoft · PI Data Archive + PI AF + PI Vision**  
Version 1.0 | Platform: Windows Server 2022

---

## ⚠️ PENTING: LISENSI SEBELUM MULAI

PI System adalah **enterprise licensed software dari AVEVA (dulu OSIsoft)**. Kamu **TIDAK BISA** install tanpa file lisensi resmi. Menggunakan tanpa lisensi melanggar EULA dan hukum.

**Yang wajib didapat dari AVEVA:**

| File | Keterangan |
|------|------------|
| `PI-Server_2018-SP3-Patch-X_.exe` | Installer kit (download dari AVEVA Customer Portal) |
| `piserver.lic` | License file — **TERIKAT ke hostname server!** |
| `PI-Vision_2022_.exe` | Installer PI Vision |
| AVEVA Customer Portal Account | Login di `customers.osisoft.com` |

> 🔴 **CRITICAL:** License file terikat ke hostname. Tentukan hostname **FINAL** sebelum request license ke AVEVA — tidak bisa diganti setelah diterbitkan.

**Cara dapat lisensi:**
- Beli langsung ke AVEVA: `aveva.com/contact`
- Lewat System Integrator (SI) resmi AVEVA Certified Partner
- Program akademik: AVEVA Learning / AVEVA Developer Program
- Jika di perusahaan: minta ke tim IT Automation untuk akses portal internal

---

## 📋 Persyaratan Sistem

| Komponen | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 8 GB | 16 GB |
| CPU | 2 Core | 4 Core |
| Disk C: (OS) | 40 GB | 80 GB |
| Disk D: (PI Data) | 20 GB | 50 GB+ |
| OS | Windows Server 2016 | Windows Server 2022 |
| .NET Framework | 4.8 | 4.8 |
| Network | Static IP | Static IP + DNS loopback |

---

## 🗺️ Arsitektur Sistem

```
[Data Source]          OPC DA/UA · Modbus TCP · Field Instruments
      ↓
[PI Interface]         PI ICU · OPC Interface · Modbus Interface
      ↓
[PI Data Archive]      Port 5450 · PI Points/Tags · Time-Series Engine
      ↓
[PI AF Server]         Asset Hierarchy · Element Templates · AF Database
      ↓
[PI Vision]            IIS Web App · Port 80/443 · Displays & Dashboards
      ↓
[Browser User]         Chrome / Edge · Real-time Visualization
```

---

## 🔧 Urutan Instalasi (8 Fase)

### FASE 00 — Procurement License

1. Tentukan hostname server FINAL
2. Hubungi AVEVA atau SI Partner resmi
3. Dapatkan installer kit + license file
4. Download dari AVEVA Customer Portal

```powershell
# Cek hostname saat ini — ini yang dikasih ke AVEVA
hostname
# Contoh output: PISERVER01
```

---

### FASE 01 — Pre-Install Windows

**1. Set Hostname Final (wajib sebelum minta license)**
```powershell
Rename-Computer -NewName "PISERVER01" -Restart
```

**2. Set Static IP Address**
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.1.100 -PrefixLength 24 `
  -DefaultGateway 192.168.1.1

Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 127.0.0.1
```

**3. Install IIS + Fitur Windows**
```powershell
Install-WindowsFeature -Name `
  Web-Server, Web-Asp-Net45, Web-Windows-Auth, `
  Web-Net-Ext45, Web-ISAPI-Ext, Web-ISAPI-Filter, `
  Web-Static-Content, Web-Default-Doc, Web-Http-Errors, `
  Web-Http-Logging, Web-Request-Monitor, Web-Filtering, `
  Web-Mgmt-Console -IncludeManagementTools

# .NET Framework 3.5 (dibutuhkan PI AF)
Install-WindowsFeature Net-Framework-Core
```

**4. Buka Port Firewall**
```powershell
# PI Server port
New-NetFirewallRule -DisplayName "PI Server" `
  -Direction Inbound -Protocol TCP -LocalPort 5450 -Action Allow

# PI Vision HTTP/HTTPS
New-NetFirewallRule -DisplayName "PI Vision" `
  -Direction Inbound -Protocol TCP -LocalPort 80,443 -Action Allow
```

**5. Disable IE Enhanced Security**
```
Server Manager → Local Server → IE Enhanced Security Configuration
→ Administrators : OFF
→ Users          : OFF
```

---

### FASE 02 — Install PI Data Archive

**1. Ekstrak dan jalankan installer**
```
PI-Server_2018-SP3-Patch-3_.exe
→ Extract ke : C:\PI_Install\PIServer\
→ Jalankan   : setup.exe (Run as Administrator)
```

**2. Pilih komponen di wizard**
```
[✓] PI Data Archive              ← WAJIB
[✓] PI AF Server                 ← install sekaligus
[✓] PI System Management Tools
[✓] PI Network Manager
[ ] PI Notifications             ← optional
```

**3. Konfigurasi di wizard**
```
PI Data Archive Name : PISERVER01      ← HARUS sama dengan hostname!
Data Store Location  : D:\PIData\
Archive Path         : D:\PIData\arc\
Backup Path          : D:\PIData\backup\
PI Port              : 5450

Auth Mode            : Windows Authentication ✅
PI Admin Account     : piadmin
```

**4. Load license file saat diminta**
```
Browse ke : C:\PI_Install\piserver.lic
Klik Verify → harus tampil: License Valid ✅
Jika error : pastikan hostname cocok dengan license
```

**5. Verifikasi setelah install (~15 menit)**
```powershell
Get-Service -Name "piarchss","pibasess","pimsgss","pisqlss","piaflink"
# Semua harus Status = Running

# Buka PI SMT → Connect to: \\PISERVER01 → Login OK ✅
```

---

### FASE 03 — Konfigurasi PI Data Archive

**1. Buat PI Identities (PI SMT)**
```
Security → Identities, Users & Groups
→ New Identity: "PI_Readers"    ← akses read-only
→ New Identity: "PI_Writers"    ← untuk interface/data input

Security → Mappings & Trusts → Mappings
→ Windows User : PISERVER01\Administrator
→ PI Identity  : piadmins
→ Save
```

**2. Buat PI Points / Tags (PI SMT → Point Builder)**

| Tag Name | Type | Unit | Span |
|----------|------|------|------|
| `Plant.Temp.Reactor01` | Float32 | DegC | 0–200 |
| `Plant.Pressure.Vessel01` | Float32 | barg | 0–10 |
| `Plant.Flow.Feed01` | Float32 | m3/h | 0–100 |
| `Plant.Level.Tank01` | Float32 | % | 0–100 |
| `Plant.Status.Pump01` | Int32 | — | 0–1 |

**3. Test input data manual**
```
PI SMT → Data → Archive Editor
→ Tag       : Plant.Temp.Reactor01
→ Timestamp : *  (sekarang)
→ Value     : 85.5
→ Enter → verifikasi nilai masuk ✅
```

---

### FASE 04 — Setup PI Asset Framework (PI AF)

**1. Buat AF Database baru**
```
Start Menu → PI System Explorer
Connect to : PISERVER01

File → New Database
Name        : PlantDatabase
Description : Demo Plant Asset Model
→ OK
```

**2. Bangun hierarki element**
```
PlantDatabase
  └─ Plant Site A
      ├─ Utility Area
      │   ├─ Reactor 01
      │   └─ Storage Tank 01
      └─ Feed Area
          └─ Feed Pump 01

# Klik kanan node → New Element
```

**3. Buat Element Template**
```
Library → Element Templates → New Template
Name: ReactorTemplate

Attributes:
  Temperature     | PI Point | Plant.Temp.Reactor01     | DegC
  Pressure        | PI Point | Plant.Pressure.Vessel01  | barg
  OperatingStatus | PI Point | Plant.Status.Pump01      | —
```

**4. Apply template & check-in**
```
Klik "Reactor 01" → Properties
→ Template : ReactorTemplate
→ Apply Template
→ Check-in (Ctrl+K)

# Verifikasi: tab Attributes → Temperature tampil nilai real-time ✅
```

---

### FASE 05 — Install PI Vision

**1. Verifikasi IIS berjalan**
```powershell
Get-Service W3SVC
# Jika Stopped:
Start-Service W3SVC
# Test browser: http://localhost → IIS default page ✅
```

**2. Jalankan installer**
```
PI-Vision_2022_.exe → setup.exe (Run as Administrator)

Step 1 - Prerequisites  : semua hijau ✅ (jika merah, fix dulu!)
Step 2 - IIS Config     :
  Website       : Default Web Site
  Virtual Dir   : PIVision
  App Pool      : PIVisionServiceAppPool

Step 3 - AF Server      :
  AF Server     : PISERVER01
  Database      : PlantDatabase
  → Test Connection → Connected ✅

Step 4 - Service Account : Network Service (untuk lab/dev)
→ Install → tunggu ~10 menit
```

**3. Konfigurasi post-install di IIS Manager**
```
Application Pools → PIVisionServiceAppPool
  → Advanced Settings
    Identity           : Network Service
    Load User Profile  : True

Sites → Default Web Site → PIVision
  → Authentication
    Windows Authentication   : ENABLED  ✅
    Anonymous Authentication : DISABLED ❌
```

**4. Setup security untuk PI Vision**
```
# PI SMT — buat Trust
Security → Mappings & Trusts → Trusts → New Trust
  Name        : PIVisionTrust
  NetMask     : 127.0.0.1
  PI Identity : PI_Readers
  → Save

# PI System Explorer — tambah AF security
File → Server Properties → Security
  Add  : PISERVER01\NetworkService
  Role : PI AF Reader
```

---

### FASE 06 — Buat Display di PI Vision

**1. Akses PI Vision**
```
http://PISERVER01/PIVision/#/
# atau: https://PISERVER01/PIVision/  (jika SSL sudah setup)
# Login: Windows Authentication (otomatis)
```

**2. Buat display baru**
```
New Display → Blank
Name: "Reactor 01 Overview" → Open
```

**3. Tambah symbols**
```
# Trend (grafik historis)
Symbol Catalog → Trend → Drag ke canvas
→ Search: Plant.Temp.Reactor01 → Add

# Value (angka real-time)
Symbol Catalog → Value → Drag ke canvas
→ Tag    : Plant.Pressure.Vessel01
→ Format : Numeric, 2 decimal places

# Gauge
Symbol Catalog → Gauge → Drag ke canvas
→ Tag          : Plant.Level.Tank01
→ Min/Max      : 0 / 100
→ Warning High : 80  (kuning)
→ Alarm High   : 90  (merah)

# Multi-state (status indikator)
Symbol Catalog → Multi-state → Drag ke canvas
→ Tag     : Plant.Status.Pump01
→ State 0 : "STOP"    → warna merah
→ State 1 : "RUNNING" → warna hijau
```

**4. Save display**
```
File → Save Display
Name   : Reactor01_Overview
Folder : /PlantDisplays/
→ Toggle ke View Mode untuk lihat hasil ✅
```

---

### FASE 07 — Koneksi Data Source

**Option A — OPC DA**
```
PI ICU → Add Interface → OPC DA
  Interface ID : 1
  OPC Server   : localhost\OPCSimulator
  Scan Class   : 10 detik
→ Build PI Tags dari OPC Items → Start Interface
```

**Option B — Modbus TCP**
```
PI ICU → Add Interface → Modbus TCP
  Device IP : 192.168.1.200  (IP PLC/RTU)
  Port      : 502
  Scan      : 5 detik
  Mapping   : HR 40001 → Plant.Temp.Reactor01
```

**Option C — PI Random (Simulasi Lab, tanpa hardware)**
```
PI ICU → Add Interface → PI Random
→ Assign ke semua PI Tags
→ Scan Class : 5 detik
→ Start Interface
# Nilai berubah random sesuai span tag ✅
```

**Option D — PI Performance Monitor (Windows counters → PI Tags)**
```
PI ICU → Add Interface → PI Performance Monitor
→ Auto-discover Windows counters
  \Processor(_Total)\% Processor Time → Server.CPU.Usage
  \Memory\Available MBytes            → Server.RAM.Available
```

---

### FASE 08 — Verifikasi End-to-End

```
[ ] License file valid & loaded
[ ] PI Archive services: piarchss, pibasess, pimsgss, pisqlss → Running
[ ] PI SMT connect ke server OK
[ ] PI Tags dibuat & nilai bisa dibaca
[ ] PI AF Database & Element hierarchy OK
[ ] AF Attributes mapping ke PI Tags — nilai tampil
[ ] IIS + PI Vision site accessible di browser
[ ] Login PI Vision via Windows Auth berhasil
[ ] Display Trend tampil data historis
[ ] Interface / data source running & update real-time
[ ] Firewall port 5450, 80, 443 terbuka
```

---

## 🔴 Troubleshooting

| Problem | Kemungkinan Penyebab | Solusi |
|---------|---------------------|--------|
| PI Vision "No Data" | Trust belum dibuat | Buat Trust di PI SMT → Mappings & Trusts |
| PI Vision 403 Forbidden | Windows Auth off | Enable Windows Auth di IIS Manager |
| AF Attribute kosong | Nama PI tag typo | Cek ulang nama di Point Builder |
| Trend tidak update | Interface tidak running | Cek PI ICU → Interface status |
| PI SMT tidak bisa konek | Port 5450 diblok | Allow port 5450 di Windows Firewall |
| License Error saat install | Hostname tidak cocok | Verifikasi hostname = yang ada di license |
| App Pool crash / 503 | .NET version mismatch | Set App Pool ke .NET 4.0 Integrated |
| PI Vision blank/white | IIS App Pool stopped | Restart PIVisionServiceAppPool di IIS |

---

## 📚 Referensi

- AVEVA Customer Portal: `customers.osisoft.com`
- PI System Docs: `docs.aveva.com`
- PI Square Community: `pisquare.osisoft.com`
- PI Web API endpoint: `https://[server]/piwebapi/`

---

*PI Server Full Installation Guide · AVEVA OSIsoft · Field Readiness Training*

