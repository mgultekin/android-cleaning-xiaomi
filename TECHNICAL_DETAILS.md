# Debloating a Xiaomi / MediaTek Android TV (Android 10 and older)
### …and fixing the two things that will actually cost you an evening: the Home button, and HDMI input switching

**How to use this file:** paste it into a fresh Claude Code / Claude session and say
*"Follow this guide on my Xiaomi Android TV, let's start."* It's written to be read as a playbook
by an agent working over ADB, and as a reference by a person.

**Where it comes from:** a real, complete debloat of a Xiaomi Mi A2 LED 55" (`MiTV-MOOQ1`,
MediaTek platform codename `tarzan`, Android 10 / SDK 29). Everything marked "confirmed" below
was observed on hardware, not inferred. Two OEM quirks in here are the reason this guide exists —
generic Android TV debloat lists don't mention either, and both will waste hours.

**Who it's for:** Xiaomi/MIUI TVs and MediaTek-chipset Android TV devices on **Android 10 or
older**. The launcher quirk in §3 is specific to this device class; it was **not** observed on
Android 11+ Google TV builds. Everything else generalizes fine.

**What you will not need:** root, an unlocked bootloader, a custom ROM, or a paid tool.

---

## 0. Setup

You need a computer on the **same LAN** as the TV, and `adb`.

Get `adb` from Google directly rather than a package manager — no sudo, no version surprises:
```
https://dl.google.com/android/repository/platform-tools-latest-darwin.zip
```
Swap `darwin` for `linux` or `windows`. **Note the path is `/android/repository/`, not
`/android/repo/`** — the shorter one 404s and is a common typo in older guides.

On the TV: **Settings → About → click "Build" ~7 times** to unlock Developer options, then
**Developer options → USB debugging → on**. Find the IP under **Settings → Network & Internet →
Status**.

```bash
adb connect <TV_IP>:5555
adb devices -l          # must say "device" — not "unauthorized", not "offline"
```

> **Older Xiaomi/MediaTek TVs have no wireless-debugging pairing-code screen at all** — just the
> plain USB debugging toggle. Don't go hunting for it, and don't bother with `adb pair`.
> `adb connect <ip>:5555` works directly once USB debugging is on.

The first connection pops an "Allow this computer?" dialog **on the TV screen** — accept it with
the remote and tick "always allow". It'll be remembered until a factory reset.

---

## 1. The rules that make this safe

**1. Never uninstall an OEM or system package. Disable it.**
```bash
adb shell pm disable-user --user 0 <package>   # reversible
adb shell pm enable <package>                  # undo
```
`pm uninstall` on a system package can be effectively irreversible short of a factory reset.
`disable-user` gives you the same result — the process never starts, never uses RAM — with a
one-command undo. There is no upside to deleting.

*Exception:* apps **you** installed during the session (a launcher you tried and abandoned, a
helper tool) are yours to `pm uninstall --user 0` freely. The rule protects factory firmware, not
your own installs.

**2. Never root, unlock the bootloader, or flash a custom ROM to fix any of this.**
It breaks Widevine L1 DRM — Netflix and Prime Video drop to SD **permanently**, even if you
re-lock afterwards. Nothing in this guide needs it. If an agent suggests it, that's the wrong
answer to whatever question was asked.

**3. Measure before and after.**
```bash
adb shell dumpsys meminfo | grep -E "Total RAM|Free RAM|Used RAM"
adb shell pm list packages -d      # what's ALREADY disabled — record this first!
```
Save the raw output to timestamped files. The "already disabled" list matters more than it looks:
if you don't record it up front, you can't later tell your changes from the factory's, and your
rollback script will "restore" things that were never on.

**4. Batches of ~10, then stop and test.**
After each batch, have someone physically test with the remote: the **Source/Input button**,
**switching HDMI inputs**, your main streaming apps, **audio**, and the **on-screen keyboard**.
Don't start the next batch until the current one is confirmed. When something breaks, a batch of
10 is a 30-second bisect; a batch of 40 is an evening.

**5. Unsure about a package? Ask. Don't guess.**
One extra question costs a few seconds. A boot loop costs a factory reset.

**6. Log as you go, not at the end.**
Two files, updated in the moment: a **"continue from here"** doc (device info, connection method,
current state, what's confirmed vs. still untested, what to do next) and a **changelog** (each
batch: what, why, exact re-enable command). This is what lets a brand-new session — or you, in
six months — pick the project back up cold. Details like the `%2F` gotcha in §4 only survive if
they're written down the moment they're discovered.

---

## 2. Finding the bloat

**Don't copy a package list from the internet for a different TV model.** Names vary by model,
region, and firmware build. Pull the real list and reason about it:

```bash
adb shell pm list packages | sort           # everything
adb shell pm list packages -3               # user-installed
adb shell pm list packages -d               # already disabled
```

### Safe to disable — patterns, confirmed on this device class

| Pattern | What it is |
|---|---|
| `*factorytest`, `*factorymenu`, `*disclaimercustomization` | Production-line test tools. Never used after the TV leaves the factory. |
| `*setupwraith`, `*tvsetup.partnercustomizer`, `*.setupwizard`, `autoinstalls.config.*` | First-boot (OOBE) wizards. Already did their job. |
| `*.analytics`, `*.statistic`, `*.bugreportsender`, `com.google.android.feedback`, `*milegal` | Telemetry and legal-notice screens. |
| `com.google.android.backdrop`, `com.android.dreams.basic` | Screensaver / ambient mode — see the note below. |
| `com.google.android.tvrecommendations`, `*.leanbacklauncher.recommendations` | The home screen's "recommended for you" rows. |
| `*.floatingframe`, `*.demo`, vendor music/gallery/video apps you don't use | Vendor extras. |
| `com.android.settings.intelligence` | Contextual search inside Settings. Pointless on a TV. |

> **The screensaver-kills-my-video complaint.** Disabling `backdrop` + `dreams.basic` is the
> standard fix and it works — but it leaves the *setting* dangling: `screensaver_enabled` stays
> `1` and `screensaver_components` still points at the now-disabled package. Nothing can launch,
> so the symptom goes away, but the clean version is to turn the setting off too:
> ```bash
> adb shell settings put secure screensaver_enabled 0
> ```
> Check what you're actually dealing with first:
> ```bash
> adb shell settings get secure screensaver_enabled
> adb shell settings get secure screensaver_components
> adb shell settings get system screen_off_timeout
> ```

### Never disable

| Function | Package pattern |
|---|---|
| The remote's **Inputs/Source** hardware button | `*.hotkey.dispatcher` (MediaTek) |
| TV input / HDMI / tuner services | `*.tvinput`, `*.tv.service`, `*.tv.agent`, `*.tv.factory` |
| The app that **renders** HDMI passthrough | `*.wwtv.tvcenter` — see the trap in §4 |
| Remote control service | `com.google.android.tv.remote.service` |
| Play Services / Store | `com.google.android.gms`, `com.google.android.gsf`, `com.android.vending` |
| Known boot-loop trigger | `com.android.location.fused` |
| Keyboard (IME) | `com.google.android.inputmethod.latin` or vendor equivalent |
| ADB itself | `com.android.shell` — disable this and you lose your connection |
| Role/permission management | `com.google.android.permissioncontroller` — the launcher switch in §3 depends on it |
| Core | `android`, `com.android.systemui`, `com.android.tv.settings`, bluetooth stack, `com.google.android.webview`, `com.android.providers.settings` |

### Free speed, no risk
```bash
adb shell settings put global window_animation_scale 0.5
adb shell settings put global transition_animation_scale 0.5
adb shell settings put global animator_duration_scale 0.5
adb shell pm trim-caches 1G
```
Noticeably snappier UI. Reverse with `1.0`. (`0` disables animations entirely — some TV UIs get
visually janky, `0.5` is the safe sweet spot.)

---

## 3. ⚠️ Quirk #1 — the Home button ignores you

Install a third-party launcher (Projectivy, FLauncher, …) and set it as home the standard way:

```bash
adb shell cmd role add-role-holder android.app.role.HOME <new_launcher>
adb shell pm set-home-activity --user 0 <new_launcher>/<MainActivity>
```

This **will** report success. It **will** verify correctly:
```bash
adb shell cmd role dump | grep -A2 "android.app.role.HOME"
adb shell dumpsys package preferred-activities | grep -A3 HOME
```
It **will** survive a reboot. And the **physical Home button on the remote will keep opening the
old launcher anyway.**

**This is not something you did wrong.** On this Xiaomi/MediaTek Android 10 platform the Home key
doesn't consult standard RoleManager / PackageManager resolution. There's no clean ADB fix for
the cause — but the workaround is completely reliable:

> ### Disable every competing launcher until exactly one is left enabled.
> With no rival to resolve against, the Home key is forced onto the only launcher standing.

```bash
adb shell pm disable-user --user 0 com.google.android.tvlauncher   # or whatever the old one is
```

Find the rivals — there are usually more than you expect (stock Google launcher, a vendor home
like Xiaomi's PatchWall, a leftover leanback launcher):
```bash
adb shell pm list packages | grep -iE "launcher|home|patchwall|tvhome"
```

Then verify **with a reboot** — this quirk has burned people who declared victory after one
successful launch:
```bash
adb reboot
# wait ~30s, reconnect
adb shell input keyevent KEYCODE_HOME
adb shell dumpsys activity activities | grep mResumedActivity
```

This pattern held across **two separate launcher migrations** on the same device (stock
`tvlauncher` → FLauncher, then FLauncher → Projectivy). Treat it as the expected mechanism on
this device class, not a one-off.

---

## 4. ⚠️ Quirk #2 — your HDMI inputs vanish with the old launcher

### Why
The row of physical inputs (HDMI 1/2/3, AV) on a stock Android TV home screen is a **launcher
feature**, backed by `TvInputManager` — **not** a system menu. Change launchers and it can
disappear with no warning, and there is **no fallback UI anywhere in Settings**. If a console is
plugged into HDMI 2, you have just lost access to it.

**FLauncher does not implement input switching** (confirmed, known limitation).

### The fix: choose the launcher that has it
**Projectivy Launcher** (`com.spocky.projengmenu`, Play Store) has a native **"Inputs" row** at
the top of its home screen — HDMI 1/2/3 plus a generic input tile — functionally replacing what
the stock launcher did. It installed with no compatibility warning where FLauncher and other
helpers needed sideloading.

**Recommend Projectivy over FLauncher on any TV with HDMI-connected devices.** This isn't taste;
FLauncher is missing a feature you will need.

### Always-available fallback, zero apps
Hold the remote's **Google Assistant button** and say **"switch to HDMI 2."** Native Android TV
voice command, entirely independent of the launcher. Worth knowing even after everything works.

### The raw mechanism (for scripting or diagnosis)
```bash
# 1. Enumerate real input IDs:
adb shell dumpsys tv_input | grep -oE "com\.[a-zA-Z0-9_.]+/[a-zA-Z0-9_.]*/HW[0-9]+" | sort -u

# 2. Map HW slots to PHYSICAL ports — this line gives it to you directly:
adb shell dumpsys tv_input | grep hdmi_port | sort -u
#    e.g. id=4 → hdmi_port=1, id=5 → hdmi_port=2, id=6 → hdmi_port=3
#    Physical numbering and internal HW numbering do NOT reliably match 1:1 — check, don't assume.

# 3. Switch to one:
adb shell am start -a android.intent.action.VIEW \
  -d "content://android.media.tv/passthrough/com.mediatek.tvinput%2F.hdmi.HDMIInputService%2FHW5"
```

> **🐛 The `%2F` trap.** The input ID itself contains slashes (`package/.Service/HW5`), and those
> slashes must be percent-encoded as `%2F` *inside* the URI path. Leave them raw and Android's
> URI parser splits them into separate path segments — the intent then resolves to a completely
> different app. (In our case it opened a media player that tried to play the URI as a video
> file.) If a passthrough intent lands somewhere bizarre, check this first.

### ☠️ The trap that black-screens the TV
On this platform the app that actually **renders** HDMI passthrough is a separate package from
the input service — typically `*.wwtv.tvcenter` with an activity like `.nav.TurnkeyUiMainActivity`.

**Launching that activity bare — as a plain component, with no `content://…passthrough/…` data
URI — can black-screen the TV and swallow all remote input.** It appears to hang waiting for a
signal, with no UI to escape from. Especially likely when no input currently has a live signal.

Recovery, from the computer (the OS is fine — only that one foreground activity is stuck, and
ADB stays alive):
```bash
adb shell input keyevent KEYCODE_HOME
```

**Only ever target that activity with a full passthrough URI for a specific live input. Never
bare.**

### Dead ends, so you can skip them
- **Activity Launcher** (`de.szalkowski.activitylauncher`) — creates shortcuts to bare components,
  but the build tested had **no field for a custom Intent `data` URI**. That makes it unable to
  target the passthrough URI, and a bare component launch is exactly the black-screen trap above.
  Also flagged "incompatible" in the Play Store for TVs; needed sideloading. Not usable here.
- **MacroDroid** (`com.arlosoft.macrodroid`) — its "Send Intent" action **does** support a custom
  `data` field, and reproduced the working `adb` command exactly (Action=`VIEW`, Data=passthrough
  URI). **This genuinely works.** The dead end was on the launcher side: FLauncher supports
  neither home-screen widgets nor pinned shortcuts, so there was no way to surface the macro as
  a one-tap tile — you'd open MacroDroid → macro → action → "Test action" every single time.
  Fine as proof of concept, unusable daily.

Both were uninstalled once Projectivy's native Inputs row made them unnecessary. **If you're
reaching for either of these, you're solving the wrong problem — change the launcher instead.**

---

## 5. Rollback cheatsheet

Build this for your device **as you go**, filling in real package names:

```bash
# Undo every package you disabled (NOT ones that were already disabled before you started)
for p in <pkg1> <pkg2> <pkg3>; do adb shell pm enable "$p"; done

# Undo the launcher switch — note the third line, it's the one people forget
adb shell pm enable <old_launcher>
adb shell cmd role add-role-holder android.app.role.HOME <old_launcher>
adb shell pm disable-user --user 0 <new_launcher>   # §3: silence the rival or Home gets confused
adb reboot

# Undo animation scales
for s in window_animation_scale transition_animation_scale animator_duration_scale; do
  adb shell settings put global "$s" 1.0
done
```

**Emergency — screen black, remote dead:**
```bash
adb shell input keyevent KEYCODE_HOME
```

---

## 6. Realistic expectations

On a 1.84 GB TV, disabling 21 packages took free RAM from **440 MB → 543 MB** measured right
after a reboot. Real-world readings bounce between ~360 and ~540 MB depending on uptime, cache,
and what's open — **`Free RAM` is a noisy number, don't treat a single reading as a verdict.**

The durable win isn't a number, it's that specific processes **never start again**. Compare
`dumpsys meminfo`'s process list before and after: in our case the stock launcher (240 MB PSS),
the screensaver (66 MB), the recommendations service (39 MB) and the setup wizard (20 MB) simply
stopped existing. That's what you actually feel — fewer background wakeups, less swap pressure,
a UI that stops stuttering while something re-indexes recommendations you never asked for.
