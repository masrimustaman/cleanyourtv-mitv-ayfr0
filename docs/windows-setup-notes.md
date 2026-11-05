# Windows setup notes: what broke, and the fix

Every problem hit while running CleanYourTV against one TV (a Xiaomi Mi TV Stick 4K, `MiTV-AYFR0`,
Android TV 11 / SDK 30) from a Windows machine inside a confined-coding-agent sandbox. Kept because
almost none of it is in the upstream docs, and all of it costs an hour to rediscover.

The environment, for context: Windows 11, PowerShell as the shell, Python 3.15 and Git present, **no
JDK and no Android SDK**, `adb` 37.0.1 from WinGet. The agent's shell is confined: writes are allowed
inside the workspace (and some temp areas) but denied in the user profile for **child processes
spawned by the shell** — which is the root cause behind four of the entries below.

Quick index:

| # | Symptom in one line |
|---|---|
| 1 | `git clone` / `curl` fail with `schannel: SEC_E_NO_CREDENTIALS` |
| 2 | `Invoke-WebRequest` dies with "underlying connection was closed" |
| 3 | `Get-NetIPAddress` / `Get-NetRoute` say Access denied |
| 4 | The TV shows as `unauthorized`, then vanishes between shell calls |
| 5 | `adb mdns services` lists nothing, so the TV can't be found by name |
| 6 | `python -m venv tools\venv` fails, venv ends up without pip |
| 7 | `pip install adb-shell` fails with `[WinError 5] ... pip\cache\wheels` |
| 8 | `disable.ps1` aborts: `PermissionError: C:\Users\<you>\.cleanyourtv` |
| 9 | `sdkmanager` reports `NoSuchFileException: ...\.android\cache\...` |
| 10 | Gradle fails at `validateSigningRelease`: `debug.keystore.lock` Access denied |
| 11 | AGP: "Several environment variables contain different paths to the Android Preferences folder" |
| 12 | `tv.py` hangs 120 s on `am start -W`, or on any command with a pipe |
| 13 | `adb shell` answers `no closing quote` |
| 14 | `pm disable-user` throws `IllegalArgumentException: Unknown package` |
| 15 | Two screenshots come back byte-identical |
| 16 | `ssh -T git@github.com` → Host key verification failed; `cmdkey` shows no GitHub credential |
| 17 | `git push` exits 1 although the push worked |
| 18 | The Google Photos ambient-mode picker cannot be reached |

---

## 1. HTTPS is broken for schannel-based tools

**Symptom** — `git clone` dies immediately:

```
fatal: unable to access 'https://github.com/Gathin23/cleanyourtv/':
schannel: AcquireCredentialsHandle failed: SEC_E_NO_CREDENTIALS (0x8009030e)
No credentials are available in the security package
```

`curl.exe` fails the same way (`curl: (35) schannel: AcquireCredentialsHandle failed`). It is not the
network and not GitHub: `ping`/TCP are fine and the same URL downloads over Python.

**Cause** — the Windows schannel TLS backend cannot reach the machine's credential/crypto store under
this token.

**Fix** — make Git use its own OpenSSL backend, per command or once globally:

```powershell
git -c http.sslBackend=openssl clone --depth 1 https://github.com/Gathin23/cleanyourtv
# or permanently:
git config --global http.sslBackend openssl
```

Anything that must download (SDK zips, release assets) goes through Python's `urllib` instead, which
uses OpenSSL and works:

```powershell
python -c "import urllib.request; open('x.zip','wb').write(urllib.request.urlopen(urllib.request.Request(url,headers={'User-Agent':'curl/8'})).read())"
```

## 2. `Invoke-WebRequest` cannot reach api.github.com

**Symptom** — `Invoke-WebRequest https://api.github.com` fails with *"The underlying connection was
closed: An unexpected error occurred on a receive."*

**Cause** — same schannel problem as #1; PowerShell's `Invoke-*` cmdlets go through it.

**Fix** — use Python for every HTTP call, including the GitHub API. Passing the token through an
environment variable keeps it out of the process list and out of logs:

```powershell
$cred = ("protocol=https`nhost=github.com`n`n" | git credential fill)
$env:GH_TOKEN = ($cred | Where-Object { $_ -match '^password=' }) -replace '^password=',''
python work\make-repo.py    # reads $env:GH_TOKEN, POSTs to the API
```

## 3. WMI network cmdlets are denied

**Symptom** — `Get-NetIPAddress` and `Get-NetRoute` both return *Access denied*
(`HRESULT 0x80041003`).

**Fix** — the old tools work and are enough:

```powershell
ipconfig | Select-String 'IPv4|Gateway'
arp -a          # this is what found the TV
```

`arp -a` also gives you every local neighbour's MAC, which is how the TV was located (see #5).

## 4. The TV is `unauthorized`, and disappears between shell calls

**Symptom** — `adb connect <tv ip>:5555` answers

```
failed to authenticate to <tv ip>:5555
List of devices attached
<tv ip>:5555    unauthorized transport_id:1
```

and a later call shows an empty device list, with `* daemon not running; starting now` every time.

**Cause** — (a) the "Allow USB debugging?" dialog on the TV had not been accepted yet; (b) the `adb`
server does not survive between agent shell invocations, so each call starts a new daemon and the
connection made by the previous call is gone.

**Fix** — accept the prompt on the TV (tick *Always allow from this computer*), and **re-connect at the
start of every single shell call**:

```powershell
adb connect <tv ip>:5555 | Out-Null
adb -s <tv ip>:5555 shell "getprop ro.product.model"
```

The repo's `tools/tv.py` avoids the whole issue: it is a pure-Python ADB client that opens its own
connection per call, so it needs no daemon and no reconnect dance. Prefer it once the venv exists (#6).

## 5. mDNS finds nothing; discover the TV by ARP instead

**Symptom** — `adb mdns services` prints an empty list, so there is no hostname to connect to and no
Wireless-debugging port to read.

**Fix** — find the TV by elimination on the LAN:

```powershell
ipconfig | Select-String 'IPv4'   # this machine's address and prefix, e.g. <this machine>/24
arp -a                            # neighbours; the one that isn't the router is the candidate
```

Then confirm by TCP. A TV that answers **5555** is an Android TV with network debugging on; **8009**
open as well is a good sign (that is Chromecast). Note that ICMP proves nothing — the gateway here did
not answer `ping` at all while TCP to the TV worked fine.

```powershell
$tv = '<tv ip>'
foreach ($p in 5555,8009,6466,6467) {
  $c = New-Object System.Net.Sockets.TcpClient
  try { $c.Connect($tv,$p); "$p OPEN" } catch { "$p closed" } finally { $c.Close() }
}
```

## 6. `python -m venv` fails, leaving a venv with no pip

**Symptom**

```
Error: Command '[...venv\Scripts\python.exe', '-m', 'ensurepip', '--upgrade', '--default-pip']
returned non-zero exit status 1.
...No module named pip
```

`ensurepip` imports fine from the base interpreter, so this is the sandbox again, not a broken Python.

**Fix** — create the venv without pip, then install into it using the *base* pip, which supports
`--python`:

```powershell
python -m venv --without-pip tools\venv
python -m pip --python tools\venv\Scripts\python.exe install -r tools\requirements.txt
tools\venv\Scripts\python.exe -c "import adb_shell; print(adb_shell.__version__)"
```

`tools\disable.ps1`, `build-and-install.ps1` and `restore.ps1` all hard-code
`tools\venv\Scripts\python.exe`, so the venv has to exist at that exact path.

## 7. pip cannot write its wheel cache

**Symptom** — building the `adb-shell` wheel fails:

```
WARNING: Building wheel for adb-shell failed: [WinError 5] Access is denied:
'c:\users\<you>\appdata\local\pip\cache\wheels\39'
error: failed-wheel-build-for-install
```

**Fix** — move pip's cache and temp dir into the workspace and skip the cache:

```powershell
$env:PIP_CACHE_DIR = "$PWD\work\pipcache"
$env:TMP = "$PWD\work\tmp"; $env:TEMP = "$PWD\work\tmp"
New-Item -ItemType Directory -Force -Path work\pipcache,work\tmp | Out-Null
python -m pip --python tools\venv\Scripts\python.exe install --no-cache-dir -r tools\requirements.txt
```

## 8. `disable.ps1` refuses to run: no recovery record (`~/.cleanyourtv`)

**Symptom** — the debloat stops before touching the TV, which is the safety net working correctly:

```
PermissionError: [WinError 5] Access is denied: 'C:\\Users\\<you>\\.cleanyourtv'
no recovery record was written, so nothing was disabled
```

Creating the directory by hand is **not** enough: the next run gets further and fails writing the file
inside it —

```
PermissionError: [Errno 13] Permission denied:
'C:\\Users\\<you>\\.cleanyourtv\\recovery-<ip>-<hash>.json.tmp'
```

**Cause** — `tools/recovery.py` writes to `Path.home() / ".cleanyourtv"`, and the confine applies to
processes spawned *by* a spawned process. A `powershell -File tools\disable.ps1` run is such a child, so
everything it spawns is denied outside the workspace. Running the same script from the agent's own
session is not.

**Fix** — run the script in-session (not through a nested `powershell -File`), and point `HOME` at the
workspace so `Path.home()` lands somewhere writable. Do both:

```powershell
$home2 = "$PWD\work\home"
New-Item -ItemType Directory -Force -Path $home2 | Out-Null
$env:USERPROFILE = $home2; $env:HOME = $home2
Set-ExecutionPolicy -Scope Process Bypass -Force
& .\tools\disable.ps1 -TvHost <tv ip> examples\xiaomi-mitv-ayfr0\debloat.txt
```

The record then lands in `work\home\.cleanyourtv\`. Copy it somewhere durable immediately — it is the
only copy, and `restore.ps1` needs it:

```powershell
New-Item -ItemType Directory -Force -Path C:\Users\<you>\.cleanyourtv | Out-Null
Copy-Item work\home\.cleanyourtv\recovery-*.json C:\Users\<you>\.cleanyourtv\
Copy-Item work\home\.cleanyourtv\recovery-*.json backup\
```

`restore.ps1` later reads the record through the same `Path.home()`, so a restore run under the same
redirect finds it in `work\home\.cleanyourtv\`, and one run from a normal shell finds the copy in the
profile. Keep both.

## 9. `sdkmanager` cannot download the SDK it is asked for

**Symptom**

```
Warning: Failed to download any source lists!
Warning: IO exception while downloading manifest:
java.nio.file.NoSuchFileException: C:\Users\<you>\.android\cache\sdkbin-1_..._repository2-3_xml
Warning: Failed to find package 'platforms;android-34'
```

**Cause** — the same confinement, one layer down: the JVM child cannot create its cache under
`C:\Users\<you>\.android`, so every repository fetch fails and the package "does not exist".

**Fix** — download the two packages straight from Google's repository index with Python and unpack them
by hand. `sdkmanager` is not needed for build-tools + one platform.

```powershell
python work\get-sdk.py    # reads repository2-3.xml, fetches platform-34 + build-tools_r34-windows.zip
```

Two gotchas worth writing down:

- the build-tools zip unpacks to a folder named **`android-14`**, not `34.0.0` — rename it, or Gradle
  will not find build-tools 34;
- licenses must exist or AGP refuses the build. Running `sdkmanager --licenses` once *does* work,
  because it only needs to write inside the SDK directory you pass with `--sdk_root`:

```powershell
(1..30 | ForEach-Object { "y" }) | & cmdline-tools\latest\bin\sdkmanager.bat --sdk_root=$sdk --licenses
```

## 10. Gradle fails at signing: `debug.keystore.lock`

**Symptom** — the build gets all the way to the end and dies:

```
Execution failed for task ':app:validateSigningRelease'.
> java.nio.file.AccessDeniedException: C:\Users\<you>\.android\debug.keystore.lock
```

**Cause** — AGP generates the debug keystore (the release build is signed with it) in
`~/.android`, which the Gradle daemon may not write.

**Fix** — point Android's per-user directory somewhere writable:

```powershell
$env:ANDROID_USER_HOME = "$PWD\work\android-home"
```

## 11. Setting `ANDROID_SDK_HOME` and `ANDROID_USER_HOME` together is fatal

**Symptom**

```
com.android.prefs.AndroidLocationsException: Several environment variables and/or system properties
contain different paths to the Android Preferences folder.
Please correct and use only one way to inject the preference location.
- ANDROID_SDK_HOME(environment variable): ...\work\android-home
- ANDROID_USER_HOME(environment variable): ...\work\android-home
```

**Cause** — AGP 8 refuses even two variables that agree, and `ANDROID_SDK_HOME` is deprecated.

**Fix** — set `ANDROID_USER_HOME` only, and clear the other:

```powershell
$env:ANDROID_USER_HOME = "$PWD\work\android-home"
Remove-Item Env:ANDROID_SDK_HOME -ErrorAction SilentlyContinue
```

The full environment that builds this launcher, all of it inside the workspace:

```powershell
$env:JAVA_HOME          = "$PWD\work\toolchain\jdk17"
$env:ANDROID_HOME       = "$PWD\work\toolchain\sdk"
$env:ANDROID_SDK_ROOT   = "$PWD\work\toolchain\sdk"
$env:ANDROID_USER_HOME  = "$PWD\work\android-home"
$env:GRADLE_USER_HOME   = "$PWD\work\gradle"
$env:TMP = "$PWD\work\tmp"; $env:TEMP = "$PWD\work\tmp"
```

`launcher\build-and-install.ps1` writes `launcher\local.properties` with `sdk.dir` on first run;
`ANDROID_HOME` is what it reads.

## 12. `tv.py` hangs for 120 s

**Symptom** — `TcpTimeoutException: Reading from <tv ip>:5555 timed out (120.0 seconds)`, on:

- `tv.py 'am start -W ...'` — the `-W` waits for the launch to report back;
- any command piped on the TV side, e.g. `dumpsys package X | grep ambient | head -20`.

**Cause** — `-W` blocks until the TV answers, and a long pipeline that ends early (`head`) leaves the
shell holding the pipe open, so the client waits for output that never comes.

**Fix** — for anything read-only, use the stock `adb` and filter on the PC instead:

```powershell
adb -s <tv ip>:5555 shell "dumpsys package com.google.android.tvlauncher" 2>&1 |
  Select-String -Pattern 'ambient|Activity'
```

For `tv.py`, avoid `-W` and keep the shell command free of pipes and quotes. A command that times out
is not necessarily a command that failed — check the TV before re-running a write.

## 13. `adb shell` and nested quotes

**Symptom**

```
/system/bin/sh: no closing quote
```

on a command like `for p in a b; do ... echo "ON $p"; done`, sent from PowerShell.

**Cause** — Windows PowerShell strips or mangles inner double quotes when handing a string to a native
program, so the TV's shell receives an unbalanced command.

**Fix** — keep commands quote-free and filter locally; or feed the command through **stdin** to
`tv.py --batch`, which the repo's own scripts do for exactly this reason (see the comments in
`disable.ps1`).

## 14. `Unknown package` during the debloat is harmless

**Symptom** — the batch prints several of these while exiting 0:

```
Exception occurred while executing 'disable-user':
java.lang.IllegalArgumentException: Unknown package: com.android.adservices.api
```

**Cause** — the list was tested on **Android TV 14**; this TV is **Android 11**, so the
Android-14-only entries do not exist here: `com.android.tv.feedbackconsent`,
`com.android.adservices.api`, `com.android.ondevicepersonalization.services`,
`com.android.federatedcompute.services`, `com.android.healthconnect.controller`,
`com.android.health.connect.backuprestore`, `android.autoinstalls.config.soc.amlogic`,
`com.netflix.tokenmanager`, `com.xiaomi.mimusic2`.

**Fix** — nothing to fix. `pm disable-user` on a package that is not installed errors per package, and
`disable.ps1` judges the run by reading `pm list packages -e --user 0` back, not by those lines. Do not
"fix" the list by deleting the entries: on a TV that *does* have them they matter, and the list is
shared.

## 15. Two screenshots come back identical

**Symptom** — `03-grid.png` and `04-grid-right.png` are the same size, byte for byte, after
`tv.py --screenshot shot.png DPAD_RIGHT`.

**Cause** — focus could not move. Play Store was the only tile in the grid, so Right did nothing. (The
other cause, worth knowing, is `screencap` returning its last frame — see `docs/TROUBLESHOOTING.md`.)

**Fix** — nothing; verify by pressing in a direction that must change something, or by checking
`dumpsys activity activities | grep mFocusedApp` instead of trusting the image.

## 16. No usable Git credential over SSH

**Symptom** — `ssh -T git@github.com` says *Host key verification failed*, `ssh-add -l` says
*Error connecting to agent*, `~/.ssh` holds only `known_hosts`, and `cmdkey /list | Select-String github`
shows nothing.

**But** — `git credential fill` for `host=github.com` returns a username and password, because Git
Credential Manager keeps its own store, not the Windows one `cmdkey` prints. **HTTPS push works** even
while every "is there a credential?" check says no.

**Fix** — do not go down the SSH path. Add an HTTPS remote and let GCM authenticate:

```powershell
git remote add mine https://github.com/<user>/<repo>.git
$env:GIT_TERMINAL_PROMPT='0'    # fail fast instead of hanging on a prompt
git push -u mine main
```

## 17. `git push` exits 1 after a successful push

**Symptom** — the push works (`* [new branch] main -> main`), but `$LASTEXITCODE` is 1 and PowerShell
prints a red `NativeCommandError`.

**Cause** — Git writes progress to **stderr**, and PowerShell treats any stderr from a native command as
an error under `$ErrorActionPreference='Stop'`.

**Fix** — trust the refs, not the exit code:

```powershell
git ls-remote mine refs/heads/main   # must equal: git rev-parse HEAD
```

## 18. The Google Photos ambient-mode picker is unreachable

**Symptom** — after re-enabling `com.google.android.backdrop` (Ambient Mode), this opens and closes
immediately, leaving Home in front:

```
am start -a com.google.android.katniss.action.AMBIENT_REDIRECT
am start -n com.google.android.katniss/...redirect.AmbientRedirectActivity
```

`dumpsys package com.google.android.backdrop` shows only one activity (`.notice.FunActivity`) and no
settings screen; `com.google.android.tvlauncher` has no ambient settings activity either.

**Cause** — on this Android TV 11 build the screen where you choose a Google Photos album belongs to
the stock launcher, which the debloat disables. `AMBIENT_REDIRECT` is a trampoline into it, so it has
nowhere to land.

**Fix** — there isn't one that keeps the debloat. Use the Jellyfin screensaver
(`org.jellyfin.androidtv/...LibraryDreamService`, already the configured dream on this TV), a
third-party app that declares its own `DreamService`, or accept the stock launcher back. Confirm what is
configured with:

```powershell
adb -s <tv ip>:5555 shell "settings get secure screensaver_components"
adb -s <tv ip>:5555 shell "cmd package query-services -a android.service.dreams.DreamService"
```

---

## Things that were fine, and are worth not re-testing

- Port 5555 was open and answered on the first try; `ro.adb.secure=1`, so authorisation was required
  and worked.
- `pm disable-user` was accepted for every package that exists — the firmware does not refuse it.
- The launcher installed on the first `pm install -r` attempt; no verification failure, no
  "unknown sources" prompt. `minSdk 26` against SDK 30 is fine.
- The Home handover survived a cold reboot, and no disabled package was re-enabled by the firmware.
- The TV is `armeabi-v7a` only; the launcher is pure Java with no native libraries, so that does not
  matter.
