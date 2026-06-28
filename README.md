# Old Phone, New Trick: How I Turned a Samsung S8 Into a Simultaneous Interpreter

**A complete technical guide to real-time translation on Android 8.0 using a rooted Samsung S8, a TRRS splitter, and Google Translate Live Translation.**

> Also published on LinkedIn: [Old Phone, New Trick: How I Turned My Samsung S8...](https://www.linkedin.com/pulse/old-phone-new-trick-how-i-turned-my-samsung-s8-slava-ponomarev-70sdc/)

---

![Setup: Samsung S8 with TRRS splitter connected](assets/setup.jpg)

---

## The Problem

My wife doesn't speak English. Every Sunday we attend church services where the pastor speaks only in English, and she sits there understanding nothing. For eight months I tried to build a solution: a private offline speech-to-speech translation device she could hold in her pocket. I experimented with four different setups. All of them failed. The latency was always too long, or the accuracy was too low, or the hardware got too hot after ten minutes.

Then one afternoon I realized I was overthinking it. The solution was already in my drawer.

## How It Works

Google Translate has a "Live Translation" mode that converts spoken audio to text in near real-time and displays the translation on screen. The mode works well. The problem is that Android routes audio from the built-in microphone by default, even when an external microphone is connected. This isn't a bug -- it's how `AudioRecord.DEFAULT` is specified in Android's audio API. Apps that don't explicitly request an external input source get the internal mic, regardless of what's plugged in.

On a rooted Android phone you can override this at the system level. The XOverrideHeadphoneJackDetection module intercepts `WiredAccessoryManager.notifyWiredAccessoryChanged()` and tells the audio subsystem that whatever is in the headphone jack is a headset (mic + speaker), not just headphones. Combine that with a TRRS Y-splitter -- one plug to the phone, one end for the microphone input, one for headphone output -- and Google Translate starts picking up audio from whatever microphone you connect.

The result: my wife puts the earpiece in, the lapel mic clips to the pastor's collar, and she reads a rolling translation on the phone screen.

---

## Demo

[Video demo](assets/demo.mp4)

---

## Hardware

| Item | Notes |
|------|-------|
| Samsung Galaxy S8 (European/Exynos) | Must be SM-G950F, SM-G950FD, SM-G950N, or SM-G950X. US models (SM-G950U/U1) cannot be rooted. |
| TRRS Y-splitter | Splits the 3.5mm jack into separate mic input and headphone output |
| Lapel microphone | Clips to the speaker's clothing |
| Earpiece or headphones | For the listener |

### Why European Models Only

Samsung's US carriers (AT&T, Verizon, T-Mobile) lock the bootloader at the hardware level. There is no workaround. If you run `adb shell getprop sys.oem_unlock_allowed` and get `0`, the phone cannot be rooted. European models sold without carrier restrictions have `1`, which means OEM unlocking is available in Developer Options.

To check your phone:

```bash
adb shell getprop ro.product.model       # should be SM-G950F (not SM-G950U)
adb shell getprop ro.csc.sales_code      # should NOT be ATT, VZW, etc.
adb shell getprop sys.oem_unlock_allowed # must be 1
adb shell getprop ro.boot.flash.locked   # must be 0
```

---

## Software Requirements

- [Magisk v30.7](https://github.com/topjohnwu/Magisk/releases/tag/v30.7) -- root manager
- [TWRP for SM-G950F](https://twrp.me/samsung/samsunggalaxys8.html) -- custom recovery
- [Odin 3.14](https://odindownload.com/) -- Samsung firmware flasher (Windows only)
- [Riru](https://github.com/RikkaApps/Riru) -- dependency for EdXposed
- [EdXposed](https://github.com/ElderDrivers/EdXposed) -- Xposed framework for Android 8.0
- [XOverrideHeadphoneJackDetection](https://github.com/anton-arnold/xoverrideheadphonejackdetection) -- the module that solves the mic routing problem

> **Note on EdXposed vs LSPosed:** LSPosed requires Android 8.1+. The Samsung S8 ships with Android 8.0 (Oreo). Use EdXposed, not LSPosed.

---

## Rooting Guide

### Step 1: Enable Developer Options and OEM Unlocking

Go to **Settings > About phone** and tap **Build number** seven times. Return to **Settings > Developer options** and enable both **USB debugging** and **OEM unlocking**. The OEM unlocking toggle will only appear if `sys.oem_unlock_allowed` is `1` on your device.

### Step 2: Flash TWRP via Odin

Samsung phones use Odin for flashing, not `fastboot flash`. Download Odin 3.14 on a Windows PC and the TWRP `.tar` file for the SM-G950F.

1. Power off the phone
2. Hold Volume Down + Bixby + Power to boot into Download Mode
3. Connect USB and open Odin
4. In Odin, click **AP** and select the TWRP `.tar` file
5. Click **Start** and wait for the flash to complete
6. The phone will restart into TWRP

### Step 3: Flash Magisk

1. Copy the Magisk `.zip` to the phone's internal storage
2. Boot into TWRP (Volume Down + Bixby + Power, then Volume Up when prompted)
3. In TWRP, tap **Install** and select the Magisk zip
4. Flash it and reboot the system

### Step 4: Install Riru and EdXposed

In Magisk Manager, open the module repository and install **Riru** first, then **EdXposed**. Reboot after each installation.

Alternatively, download the zip files from GitHub and flash them via TWRP:
- [Riru releases](https://github.com/RikkaApps/Riru/releases)
- [EdXposed releases](https://github.com/ElderDrivers/EdXposed/releases)

After installing EdXposed, you will be prompted to install **EdXposed Manager** (the companion app) from its APK.

### Step 5: Install XOverrideHeadphoneJackDetection

1. Clone or download the module: `gh repo clone anton-arnold/xoverrideheadphonejackdetection`
2. Build the APK or download a release from the repository
3. Install the APK on the phone
4. Open **EdXposed Manager > Modules** and enable XOverrideHeadphoneJackDetection
5. Reboot

### Step 6: Activate the Override via ADB

After rebooting, activate the override with this broadcast:

```bash
adb shell am broadcast \
  -a de.antonarnold.android.xoverrideheadphonejackdetection.ConfigReceiver \
  --ei overrideEnable 1 \
  --ei overrideValue 4 \
  --ei overrideMask 255
```

This tells Android to treat the 3.5mm jack as a headset (type 4) rather than headphones. Connect your TRRS splitter and verify that the mic input is active.

### Step 7: Configure Google Translate

Open Google Translate, tap the microphone button, and select **Live Translation**. Speak into the external microphone. The translation should appear in real-time on screen.

---

## Why Not Build an Offline Device?

I spent eight months trying. Here's what I tested and why each failed:

1. **Raspberry Pi + Whisper + argostranslate** -- 4-6 second latency, unacceptable for conversation
2. **Android phone with offline Whisper APK** -- models small enough to run on-device were too inaccurate for natural speech
3. **Laptop in a bag with USB audio** -- too bulky, fan noise, battery life under 2 hours
4. **Local server + phone client over WiFi** -- works in the office, fails anywhere without a controlled network

Google Translate processes audio on its servers and returns results in roughly 1-2 seconds. That's acceptable for a sermon or lecture. For fully private, offline use, the best current path (as of mid-2026) is:

- **STT:** [RealtimeSTT](https://github.com/KoljaB/RealtimeSTT) with Whisper large-v3
- **Translation:** [Qwen3-32B](https://huggingface.co/Qwen/Qwen3-32B) running locally
- **TTS:** [CosyVoice 3](https://github.com/FunAudioLLM/CosyVoice)
- **Hardware:** 2x RTX 3090 (48GB VRAM total) for acceptable latency

That's the next project.

---

## What "Real-Time" Actually Means

The translation appears about 1-2 seconds after the word is spoken. In practice this means my wife is always reading one or two sentences behind what's being said. For a church sermon -- linear, no interruptions, predictable pacing -- this works fine. For a conversation with back-and-forth, it would be frustrating. Manage expectations accordingly.

---

## License

MIT. Do what you want. If you improve it, share what you learned.

---

## Acknowledgments

The XOverrideHeadphoneJackDetection module was built by [@anton-arnold](https://github.com/anton-arnold). This whole thing is built on that module. Without it the hardware trick doesn't work.
