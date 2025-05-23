---
title: Userspace Audio Stack
---

# Speaker support in Asahi Linux

Currently supported models:

* M1 MacBook Air 13" (J313)
* MacBook Air (13-inch, M1, 2020)
* MacBook Air (13-inch, M2, 2022)
* MacBook Air (15-inch, M2, 2023)
* MacBook Pro (13-inch, M1/M2, 2020/2022)
* MacBook Pro (14-inch, M1/M2 Pro/Max, 2021/2023)
* MacBook Pro (16-inch, M1/M2 Pro/Max, 2021/2023)
* Mac mini (M1/M2/M2 Pro, 2020/2023)
* Mac Studio (M1/M2 Max/Ultra, 2022/2023)

Proper speaker support has been a multi-year development effort involving many people across all parts of the Linux audio stack. We want to offer the best speaker support of any Linux laptop platform, and that has meant driving Linux desktop audio forward a couple decades so that we can do what is expected of a modern laptop audio subsystem!

## Current state

We are ready to begin releasing preliminary speaker support to users. Keep in mind that as Asahi Linux is the first desktop Linux platform with integrated advanced speaker DSP, there will likely be bugs. In addition, the DSP profiles will be improved and adjusted over time, and do not represent the absolute best possible results.

Speaker DSP is currently only supported on Fedora Asahi Remix (things are moving fast), however alternative distros should be able to integrate this work relatively painlessly once things settle down (see the section below).

## Known bugs

* In KDE Plasma, if you toggle mute using the keyboard hotkey while the master volume is set at anything other than 100%, on unmute the speaker volume will be too low. Touching the volume control (or pressing a volume hotkey) will restore the intended volume.
* The DSP chain introduces excessive delay, and trailing audio is "buffered" (if you stop playing something and start something else, you get a bit of the end of the first when the second starts).
* There is no final limiter/compressor in the current DSP chains (although there is an input compressor), so inputs with content in high-gain regions of the EQ curve might cause distortion or clipping (and speakersafetyd limiting). This is most prominent in the 200Hz region right now. This should not cause damage, but we recommed lowering the volume if you notice the sound is noticeably distorted.
* The 13" MacBook Air EQ curve might be a bit harsh on the treble; pending re-calibration with an individually calibrated microphone to confirm/fix.
* Bankstown (the "fake bass" plugin) uses a relatively naïve algorithm right now, which doesn't work well for all music.

## Project goals

Our DSP profiles aim to provide a balanced sound, with the features that people expect from high-quality laptop/small-speaker audio. In particular, we aim for:

- A balanced (neutral) tone at moderate listening volumes, with a mostly flat frequency response from a typical listening position
- Reasonably high peak volume with acceptable sound degradation (compression, limiting, etc.)
- ["Fake bass" processing](https://en.wikipedia.org/wiki/Missing_fundamental#Audio_processing_applications) to make audible frequencies that cannot be physically reproduced by the speakers, extending the perceived frequency response of the system. 
- [Equal-loudness volume compensation](https://en.wikipedia.org/wiki/Equal-loudness_contour), so that the sound does not become noticeably tinny as the system master volume is lowered.

These are all techniques that are in wide use in consumer microspeaker systems in tablets and phones, though sadly not common on laptops from most non-Apple brands. All of these processing tricks are also used by macOS.

Our goal is explicitly *not* to clone the full/exact macOS audio experience. We consider the macOS speaker DSP processing to be too tryhard; some of the things it does work well, some do not (or are outright buggy!), and some are just strange. Although audio is ultimately subjective, and we recognize that some people might prefer the "macOS sound", we aim for a more objectively neutral and balanced sound than macOS as a baseline. Users are encouraged to try adding their own effects (e.g. with EasyEffects) if they want to customize their experience and achieve certain sonic profiles. We believe that a balanced baseline that allows users to shape the sound to their own preference if they so desire is a better option than hard-coding specific effects (such as macOS' spatial audio processing) with no option to turn them off. There is no user-friendly (GUI) way to modify or tweak our DSP chains, so it's best if additional effects are left to existing utilities like EasyEffects that can be easily customized by the user.

That said, for the types of processing we *do* intend to apply, failing to work properly where macOS does (e.g. objectively bad audio quality for certain kinds of inputs in particular, such as bad-sounding compression or distortion at high volumes where macOS sounds better at an equal output volume) is considered a bug. Feel free to file issues if you find test cases where macOS does a clearly better job. We're particularly talking about program-dependent problems here, not "I like the macOS EQ/spatial/whatever stuff better in general".

## "Smart Amp" / safety support

In addition to the DSP processing, we also have the world's first (as far as we know) open source "smart amp" implementation. This allows the speakers to be driven to much higher peak levels than the worst-case safe volume level, greatly extending their dynamic range. Our implementation, [speakersafetyd](https://github.com/AsahiLinux/speakersafetyd), monitors feedback signals from the amplifiers, estimates speaker voice coil temperature using a two-stage thermal model, and reduces the hardware speaker volumes when the speakers get too warm. We also have kernel-side interlocks to disable or limit speaker volumes if speakersafetyd is not running or non-responsive.

Right now, we are shipping with a hard volume reduction limit of -7dBFS to catch potential bugs or misbehavior. If you notice that your speakers cut off for approximately one second and then come back at a reduced volume level, it is likely that you triggered this safety limit. Stop playback to let the speakers cool down (just in case) and let the limit re-arm, and then check `/var/lib/speakersafetyd/blackbox` for any blackbox dumps and [file a bug]() attaching them together with any speakersafetyd logs (try `journalctl -S '10m ago' -u speakersafetyd`). If you play full-scale test tones or "extreme" music at maximum volume, it is expected that you will hit this limit and trigger a blackbox dump / panic. This is intended to help catch any badness while playing normal music, and we will remove this conservative limit once we are confident in the software stack (speakersafetyd has other safety limits built in and will gain more over time).

You can watch speakersafetyd in action by using `sudo journalctl -fu speakersafetyd`. As a rule of thumb, with the speaker volume at max, you should expect speakersafetyd not to kick in when playing loudness-normalized music (e.g. YouTube). Playing overly loud music without normalization (e.g. increasing the browser app volume to 100% in the Plasma applet while using YouTube, which bypasses the normalization limit it applies, or using media players without normalization) is likely to trigger some gain reduction from speakersafetyd, but it should not hit the -7 dB panic limit unless it's something [ridiculous](https://open.spotify.com/album/6uvGw7zcCyMzYKKqXp9D3z).

**WARNING:** Speaker safety models have yet to be fully validated on all models. We are enabling audio only on models where we are confident things are ready and safe to use. If you use any undocumented overrides to force-enable speakers on any other machine models, **you are entirely on your own** and may very well blow up your speakers.

## Distro integration notes

Requirements according to [asahi-audio](https://github.com/AsahiLinux/asahi-audio)'s [README.md](https://github.com/AsahiLinux/asahi-audio/blob/main/README.md).

The correct deployment order is asahi-audio/speakersafetyd > (whatever you use to get those installed for users, e.g. metapackage) > kernel. If you push the kernel first before asahi-audio, users will get either a nonfunctional (if no speakersafetyd) or functional but bad-sounding (if speakersafetyd is installed) raw speaker device with no DSP.

# Microphone support

Currently supported models:

 * MacBook Pro 13" (M1/M2)
 * MacBook Air 13" (M1/M2)
 * MacBook Pro 14" (M1 Pro/Max, M2 Pro/Max)
 * MacBook Pro 16" (M1 Pro/Max, M2 Pro/Max)
 * MacBook Air 15" (M2)


MacBooks have three Pulse Density Modulation (PDM) mics wired up to an ADC
and decimator in the AOP. All three mics are plumbed directly to userspace on
separate channels, with no preamplification.

They are very sensitive and omnidirectional, so to be able to use them, some
kind of beamforming needs to be applied.

For this, [Triforce](https://github.com/chadmed/triforce) was built,
implementing a Minimum Variance Distortionless Response adaptive beamformer.

`asahi-audio` includes the necessary wireplumber config to set this up.

## Known Bugs

### Requires `os-fw-version` > 13.5
Microphone support currently only works if your `asahi,os-fw-version` is >= 13.5.
This is due to `BOOTARGS_OFFSET` and `BOOTARGS_SIZE` being different on `12.4`.

If your `/proc/device-tree/chosen/asahi,os-fw-version` shows an older version, and you see a

```
apple_aop 24ac00000.aop: probe with driver apple_aop failed with error -22
```

in dmesg, an update of the APFS container is needed (or adding support for the
older version to the kernel driver).

### Macbook Pro 14" M2 (J414) probing issues
There's been reports about the Macbook Pro 14" M2, where the AOP does not probe
properly and microphone support doesn't work.

### Microphone indicator on constantly
With microphone support set up, `gnome-shell` and `plasma` permanently report
the microphone being used.

This is due to wireplumber reporting it has a handle open on a "microphone
device".

Fixing this probably requires rearchitecturing a lot of concepts like nodes and
handles in Pipewire / Wireplumber.
