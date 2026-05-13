# Windows System PATH Corruption — Diagnosis & Fix

## Tanggal Kejadian
2026-05-14

## Gejala
- Environment Variables dialog menampilkan `%PATH%` di tengah-tengah System PATH
- Banyak entri PATH yang terduplikasi (muncul 2–4 kali)
- Path inti Windows (`C:\Windows\system32`, `C:\Windows`, dll.) **hilang** dari System PATH registry

## Penyebab
**Bukan Windows Update, bukan GitHub Copilot.**

Penyebabnya adalah **installer software** (VS Code, PowerShell 7, atau sejenisnya) yang salah memodifikasi System PATH dengan cara:

```
# Yang dilakukan installer (SALAH):
new_value = "C:\new\tool;%PATH%;lebih_banyak_path"
Set-ItemProperty ... -Value $new_value
```

Ketika `%PATH%` ditulis secara literal ke registry System PATH, ini menciptakan **circular reference**. Setiap installer berikutnya yang melakukan hal sama akan memperparah kerusakan:
- Entri sebelum `%PATH%` ter-expand berulang kali
- Total duplikasi membengkak secara eksponensial

## Yang Diubah saat Perbaikan

### System PATH sebelum (broken):
```
C:\Program Files\PowerShell\7
C:\Program Files\Microsoft VS Code
C:\Program Files\Microsoft VS Code          ← duplikat
%PATH%                                      ← circular reference!
C:\Program Files\PowerShell\7              ← duplikat
C:\Program Files\PowerShell\7\             ← duplikat + trailing slash
C:\Windows\System32\OpenSSH\
... (semua duplikat 2-3x lagi)
C:\Program Files\nodejs\
C:\Program Files\Microsoft VS Code\bin
```
> ⚠️ `C:\Windows\system32`, `C:\Windows`, `C:\Windows\System32\Wbem`, `C:\Windows\System32\WindowsPowerShell\v1.0\` **TIDAK ADA**

### System PATH setelah (fixed):
```
C:\Windows\system32
C:\Windows
C:\Windows\System32\Wbem
C:\Windows\System32\WindowsPowerShell\v1.0\
C:\Program Files\PowerShell\7
C:\Windows\System32\OpenSSH\
C:\Users\akdir\.local\bin
C:\Users\akdir\AppData\Local\Microsoft\WindowsApps
C:\Users\akdir\AppData\Local\GitHubDesktop\bin
C:\Users\akdir\AppData\Roaming\Composer\vendor\bin
C:\Users\akdir\AppData\Local\PowerToys\DSCModules\
C:\Program Files\nodejs\
C:\Program Files\Microsoft VS Code\bin
C:\Program Files\Git\cmd
C:\Program Files\Git\usr\bin
C:\Program Files\Docker\Docker\resources\bin
C:\Program Files\Python313
C:\Program Files\Python313\Scripts
C:\Program Files\GitHub CLI
```

**User PATH** (tidak diubah, sudah benar):
```
C:\Users\akdir\AppData\Local\Programs\Python\Python312\
C:\Users\akdir\AppData\Local\Programs\Python\Python312\Scripts\
C:\Users\akdir\.local\bin
C:\Users\akdir\AppData\Local\Microsoft\WindowsApps
C:\Users\akdir\AppData\Local\GitHubDesktop\bin
C:\Users\akdir\AppData\Roaming\npm
```

Yang ditemukan terinstall tapi hilang dan harus ditambahkan: Git, Docker, Python 3.13 (system install), GitHub CLI (`gh`).

**Catatan Copilot CLI (`gh copilot`):** Extension binary ada di `%LOCALAPPDATA%\GitHub CLI\copilot\copilot.exe` dan dipanggil otomatis oleh `gh copilot` — tidak perlu entri PATH tersendiri. Cukup pastikan `gh` ada di PATH.

Backup PATH sebelum diubah disimpan di:  
`C:\Users\akdir\Documents\PATH_backup_20260514_043053\`

> ⚠️ **Lesson learned:** Saat menambahkan entri ke PATH via PowerShell, **jangan gunakan** `[System.Collections.Generic.List[string]]::new($array)` karena constructor ini tidak menerima array langsung di semua versi PS. Gunakan `$list = [System.Collections.Generic.List[string]]::new(); $array | ForEach-Object { $list.Add($_) }` atau cukup gunakan array biasa `@()` + operator `+=`.

## Cara Mencegah Terulang

### 1. Gunakan `[Environment]::SetEnvironmentVariable` dengan benar
Hindari memodifikasi PATH via raw string. Gunakan pendekatan append yang aman:

```powershell
# Cara AMAN menambahkan entry ke System PATH:
$current = [Environment]::GetEnvironmentVariable('Path', 'Machine')
$newEntry = 'C:\new\tool'
if ($current -notmatch [regex]::Escape($newEntry)) {
    [Environment]::SetEnvironmentVariable('Path', "$current;$newEntry", 'Machine')
}
```

### 2. Audit PATH secara berkala
Jalankan script ini untuk memeriksa kesehatan PATH:

```powershell
$systemPath = (Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment' -Name Path).Path
$entries = $systemPath -split ';'

# Cek circular reference
if ($systemPath -match '%PATH%') { Write-Warning "CIRCULAR REFERENCE detected: %PATH% in System PATH!" }

# Cek duplikat
$entries | Group-Object | Where-Object Count -gt 1 | ForEach-Object {
    Write-Warning "DUPLICATE: $($_.Name) (x$($_.Count))"
}

# Cek core Windows paths
@('C:\Windows\system32','C:\Windows','C:\Windows\System32\Wbem') | ForEach-Object {
    if ($entries -notcontains $_) { Write-Warning "MISSING core path: $_" }
}
```

### 3. Backup PATH sebelum install software besar
```powershell
(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment' -Name Path).Path | 
    Out-File "$env:USERPROFILE\Documents\PATH_backup_$(Get-Date -Format yyyyMMdd).txt"
```

### 4. Gunakan tool manajemen package
Prefer `winget`, `scoop`, atau `choco` — ketiganya lebih berhati-hati dalam memodifikasi PATH dibanding installer `.exe`/`.msi` manual.
