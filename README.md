# Clean Your Xiaomi / MediaTek Android TV

If your Xiaomi/MIUI TV (or any MediaTek Android 10 and older TV) is sluggish, you can safely disable unused factory apps using an AI agent (like Claude Code, Cursor, or Aider). 

There's **no root required**, nothing is deleted, and everything is completely reversible. 

## Preparation

### 1. Enable Developer Options
On your TV, go to **Settings → Device Preferences → About** and click on **Build** ~7 times.

### 2. Enable USB Debugging
Go to **Settings → Device Preferences → Developer options** and turn on **USB debugging**.

### 3. Find your TV's IP Address
Go to **Settings → Network & Internet → Status**. Note down the IP address (e.g., `192.168.1.50`). Make sure your computer is on the same network.

### 4. Open your AI Agent and paste the prompt
Copy the text from [`prompt.txt`](./prompt.txt), replace `YOUR_IP_HERE` with your TV's IP, replace `YOUR_TV_BRAND_AND_MODEL_HERE` with your specific TV model, and paste it into your AI agent. The agent will handle the rest: connecting to your TV, suggesting apps to disable, and safely applying the changes.

## ⚠️ Three Important Warnings

- **The Home Button Quirk:** Installing a 3rd party launcher isn't enough on these TVs. The physical Home button will still open the old launcher unless you disable all other launchers. (The provided prompt handles this).
- **The Missing HDMI Inputs Quirk:** The physical inputs (HDMI 1/2/3) are part of the stock launcher. If you switch launchers, they might disappear. We recommend using **Projectivy Launcher** as it has a native Inputs row. (The provided prompt handles this).
- **Don't Root:** Rooting or unlocking the bootloader breaks Widevine L1 DRM permanently, downgrading Netflix and Prime Video to SD quality. Using `pm disable-user` is safe and sufficient.

## Reverting Changes

Since nothing is permanently deleted, you can easily restore apps if something breaks. You can ask your AI Agent to "revert the changes" or manually enable a package via terminal:

```bash
adb shell pm enable com.package.name
```

> **Note:** For deeper technical insights, edge cases, and manual ADB commands, refer to [`TECHNICAL_DETAILS.md`](./TECHNICAL_DETAILS.md).
