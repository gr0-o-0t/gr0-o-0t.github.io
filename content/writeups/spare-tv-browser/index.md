---
title: "I Wanted a Browser on the TV, Not Another Desktop"
subtitle: "The messy build log of a containerized TV screen on my Alpine homelab"
date: 2026-11-10T14:40:00+06:00
lastmod: 2026-11-10T14:40:00+06:00
draft: false
author: "Md Abdullahil Kafi"
authors:
  - "Md Abdullahil Kafi"
description: "How I turned a spare TV into a browser screen with an Alpine Linux host, Docker, Cage, Brave, and a lot of troubleshooting."
summary: "This is the actual build story: the original Jellyfin idea, the first Chromium container, the blank screen, missing input, the Brave migration, silent HDMI audio, and an OpenRC controller that worked in some ways and still had rough edges."
tags:
  - Alpine Linux
  - Docker
  - Homelab
  - Brave Browser
  - Linux
  - Wayland
  - AMDGPU
  - ALSA
categories:
  - Homelab
  - Linux
series:
  - Homelab Projects
series_weight: 1
hiddenFromHomePage: false
hiddenFromSearch: false
lightgallery: true

toc:
  enable: true
  auto: true
---

I had a spare TV and a Minisforum MS-A2 that was already doing homelab work. I wanted to put the TV to use without buying another streaming stick and, more importantly, without turning the mini PC into a normal desktop machine.

The first version of the idea was Jellyfin on the TV. I was interested in the Jellyfin web interface and its integrations with the rest of the media stack, but I did not want to build a Kodi-plus-Jellyfin setup. The plan was to keep the Jellyfin pieces in containers and use a browser container to put the web interface on the HDMI display.

That description makes it sound as though I knew exactly where I was going. I didn't. I was figuring out one Linux boundary at a time: can a container open the GPU? Can Cage take over the display? Why is there no keyboard? Why is the TV silent even though ALSA says the test completed? And can I make the whole thing start without logging in and typing commands by hand?

The project eventually became more general than Jellyfin. I stopped treating the TV as a dedicated media appliance and made it a regular browser screen. This post follows that change and the troubleshooting that got me there. It is based on the saved project notes, terminal output, and the dated sessions; where the logs do not establish a root cause, I have left that uncertainty in.

## 1. The constraint: keep Alpine as a server

The host is a Minisforum MS-A2 with 32 GB of RAM and a Ryzen 9 9955HX with integrated Radeon graphics. It runs Alpine Linux. I wanted to keep it that way: no host desktop environment, no display manager, and no browser installed on the host just to make the TV usable.

The rough idea was to give one container the few things a display session actually needs:

- the AMD DRM device so it can draw directly to HDMI;
- a small Wayland compositor to manage the screen;
- the input devices for the wireless keyboard and mouse;
- audio devices for HDMI sound;
- and, of course, the browser.

Cage looked like a good fit for the compositor. It is small and is designed to run a single application. Underneath, it uses wlroots. That meant I could let Cage use the DRM/KMS device directly rather than run a full desktop session on the host.

The split I was aiming for was simple: Alpine owns the hardware and Docker; the container owns Cage and the browser. In practice, the phrase “give the container the display” hides quite a few details.

## 2. First attempt: Alpine, Cage, and Chromium

I started with an Alpine-based image because the host was Alpine and the packages I needed were already available. The first Dockerfile was deliberately uncomplicated:

```dockerfile
FROM alpine:edge

RUN apk add --no-cache \
    cage \
    chromium \
    seatd-launch \
    xwayland \
    mesa-dri-gallium \
    mesa-va-gallium \
    mesa-gbm \
    mesa-egl \
    mesa-gl \
    ttf-dejavu
```

The browser was Chromium from Alpine's repositories. I also created a `kiosk` user rather than run the browser as root. At this stage, I was not trying to solve every operational detail. I just wanted to see whether a browser inside a container could drive the TV.

### The host could see the TV; the container still showed a blank screen

The first useful check was on the host. The TV appeared as an HDMI connector under DRM. The connector was reported as `card1-HDMI-A-1`, and the preferred mode was:

```text
1366x768 @ 59.790 Hz (preferred)
```

There were also 1920x1080 modes in the list. That was encouraging: the cable, HDMI connector, and kernel display detection were not completely missing.

The container logs also showed Cage finding the AMD GPU and initializing Mesa. A later log gave us a useful snapshot of the graphics path:

```text
EGL vendor: Mesa Project
EGL driver name: radeonsi
Using OpenGL ES 3.2 Mesa 26.2.4
GL vendor: AMD
GL renderer: AMD Ryzen 9 9955HX 16-Core Processor (radeonsi, ...)
```

Cage found `HDMI-A-1`, marked it connected, and listed the display modes. So the blank screen was not simply “Docker cannot see the GPU”. The stack could at least open the DRM device and inspect the connector.

There were still errors in the early runs. I saw DRM file-descriptor permission messages, including:

```text
Unable to clone DRM fd for client fd: Permission denied
```

I also had an Xwayland problem: the process expected `/usr/bin/Xwayland`, but that binary was missing in one of the image builds. I had installed the package in a later build, but I do not have a clean, isolated test proving that one of these messages alone was the cause of the blank screen. The display path involved seat access, DRM permissions, Cage, and Chromium's Wayland/X11 support. It took more than one change to get from “Cage detects the TV” to “I can see the browser”.

I also had to treat the virtual terminal as part of the display setup. Later, switching to VT7 manually with `chvt 7` made the difference when Cage was trying to take over the display. I eventually automated that, but at first it was just another command I had to remember to run.

### I checked the browser sandbox separately

I did not want to fix a browser startup problem by casually disabling the browser sandbox. I had a custom Docker seccomp profile, so I tested Chromium in a much smaller headless command before mixing the browser with Cage and the physical display.

The test was:

```sh
docker run --rm \
  --security-opt seccomp="$PWD/seccomp-chromium.json" \
  --entrypoint chromium-browser \
  local/tv-kiosk:edge \
  --headless \
  --disable-gpu \
  --disable-features=Vulkan \
  --no-first-run \
  --user-data-dir=/tmp/chromium-seccomp-check \
  --dump-dom \
  'data:text/html,<title>sandbox-test-ok</title><p>ok</p>'
```

Chromium printed a lot of D-Bus and graphics errors. That was expected to some extent in a minimal container with no desktop session or system D-Bus. The useful part was that it still printed the requested HTML:

```html
<html><head><title>sandbox-test-ok</title></head><body><p>ok</p></body></html>
```

That told me the test process could run with that Docker seccomp profile. It did not prove that every browser process or every syscall used by the full graphical session was fine. I learned that distinction the hard way later.

### The first real milestone

By 17 October, the browser was visible on the physical TV. That was the point where the idea stopped being theoretical: the MS-A2 was still an Alpine server, and a container was drawing a browser onto the HDMI-connected television.

It was not finished. The keyboard and mouse were the next problem. A screen that I could not control was not particularly useful.

## 3. The input problem: Cage had a display backend, not an input backend

My first Compose configuration passed a handful of `/dev/input/eventN` nodes into the container. I had mapped the event devices I could see on the host, added the relevant supplementary group, and mounted `/run/udev` because libinput uses udev device information to identify and classify input devices.

There were two separate mistakes hiding in that setup.

The first was in the wlroots configuration. Cage was starting with:

```yaml
environment:
  WLR_BACKENDS: "drm"
```

That tells wlroots to use the DRM backend for the display. It does not ask it to create the libinput backend for keyboards and mice. The logs made that fairly obvious once I looked closely:

```text
Loading user-specified backends due to WLR_BACKENDS: drm
```

The TV image could work while the input devices remained invisible to the compositor. The change was small:

```yaml
environment:
  WLR_BACKENDS: "drm,libinput"
```

That enabled both sides: DRM for the screen and libinput for input.

The second mistake was how I mounted `/run/udev`. I initially put it under Compose's `devices:` section, alongside the actual character devices. Docker rejected it:

```text
error gathering device information while adding custom device "/run/udev": not a device node
```

That error was fair. `/run/udev` is a directory, not a device node. It belongs under `volumes:`:

```yaml
volumes:
  - /run/udev:/run/udev:ro
```

The event devices themselves stayed under `devices:`.

### Alpine's device manager mattered too

The host was using Alpine's lightweight default device setup. For the udev information I needed, I switched it with Alpine's helper:

```sh
doas setup-devd udev
```

That brought in eudev and the OpenRC udev services. Afterward, `/run/udev` was present and `udevadm info` could show the device properties.

### I misidentified the wireless receiver

The Compx 2.4 GHz receiver exposed several event interfaces. The list looked roughly like this:

```text
/dev/input/event0  Compx 2.4G Wireless Receiver
/dev/input/event1  Compx 2.4G Wireless Receiver
/dev/input/event2  Compx 2.4G Wireless Receiver
/dev/input/event3  Compx 2.4G Wireless Receiver
/dev/input/event4  Compx 2.4G Wireless Receiver
/dev/input/event5  Compx 2.4G Wireless Receiver
```

One of the interfaces was described as a mouse by the input subsystem. I initially treated that as the mouse interface I needed. It wasn't the physical mouse I was using; the Compx receiver was connected to the keyboard. I later plugged in a separate SIGMACHIP USB mouse, which appeared as `event18` in one boot.

That was a useful lesson in not treating an `eventN` number—or even a device's friendly name—as a reliable identity. The event number can change, and one USB receiver can expose several logical input devices.

The stable udev links were much better identifiers:

```text
/dev/input/by-id/usb-Compx_2.4G_Wireless_Receiver-event-kbd
/dev/input/by-id/usb-SIGMACHIP_USB_Mouse-event-mouse
```

The host controller eventually resolved those links to the current event nodes every time it prepared the container. That is more reliable than permanently mapping `event0` and `event18` and hoping they stay that way after a reboot or USB change.

The final Compose setup used read/write mappings for the input devices and supplied the host's numeric input group ID. On this host the relevant supplementary groups were audio `18`, input `23`, and video `27`:

```yaml
group_add:
  - "18"
  - "23"
  - "27"
```

The combination that mattered was not a magic permission setting by itself. I needed the device nodes to exist in the container, the correct group access, udev information, and the libinput backend enabled. Leaving out any one of those made the debugging confusing.

## 4. Moving from Chromium to Brave changed the image

Once I had a browser on the TV, I wanted to use Brave rather than stay with Alpine's Chromium package. This was not a host OS change. I only changed the browser container.

Brave's official Linux installation path provided a Debian APT repository, while Alpine was not a supported native package target in the same way. I did not want to make the image more complicated by trying to shoehorn that Debian package into Alpine. So I moved the container to Debian 13 (Trixie) and left the host on Alpine.

The base image became:

```dockerfile
FROM debian:trixie-slim
```

The image installed Brave from its official repository, along with Cage, seatd, Xwayland, libinput, Mesa/DRM libraries, ALSA utilities, and fonts. I kept the browser user unprivileged:

```dockerfile
RUN groupadd --gid 1000 kiosk \
    && useradd --uid 1000 --gid 1000 --home-dir /home/kiosk \
       --create-home --shell /usr/sbin/nologin kiosk

USER kiosk
```

This migration solved the package-source problem. It did not make the graphics and seat problems disappear; those still had to be configured in the container.

### `seatd-launch` could not create its socket

One of the first Debian runs failed before Cage could do much:

```text
[ERROR] [seatd/seatd.c:35] Could not bind socket: Permission denied
[ERROR] [seatd/seatd.c:184] Could not create server socket
seatd exited prematurely
```

The fix in the image was to make `seatd-launch` root-owned and setuid:

```dockerfile
RUN chown root:root /usr/bin/seatd-launch \
    && chmod 4755 /usr/bin/seatd-launch
```

I checked that it had the expected permissions:

```text
-rwsr-xr-x 4755 root:root /usr/bin/seatd-launch
```

That got seatd past the socket-creation failure. It is worth calling out because setuid is a real privilege boundary, not a random chmod to copy into every container. The browser itself still ran as UID 1000 (`kiosk`), and I avoided `privileged: true` and `--no-sandbox` shortcuts.

### Cage was reading Brave's arguments

I also had a command-line parsing mistake. My first form did not give each program its own argument separator, and Cage exited with:

```text
cage: invalid option -- '-'
Usage: cage [OPTIONS] [--] APPLICATION
```

The working launch structure used one `--` for `seatd-launch` and another for Cage:

```sh
exec seatd-launch -- cage -- brave-browser \
    --ozone-platform=wayland \
    --no-first-run \
    --no-default-browser-check \
    --start-fullscreen \
    brave://newtab/
```

That second separator looks trivial, but without it the browser's options were being interpreted by the wrong program.

### I stopped making the TV a Jellyfin-only screen

The early setup used a local `welcome.html` file as a placeholder for the Jellyfin page, and one of the intermediate refresh configurations still launched the browser with `--kiosk` and a `JELLYFIN_URL`. By the time I settled on Brave, I wanted a normal browser instead.

I removed the placeholder URL and the `--kiosk` flag. Brave opened a new tab in fullscreen using `--start-fullscreen`; pressing `F11` brought back the normal browser controls. The container kept the name `tv-kiosk`, but its job was now simply to provide a browser on the TV.

I also moved the browser profile out of the container's writable layer. The Compose bind mount was:

```yaml
volumes:
  - ./config:/config
```

Brave used:

```sh
--user-data-dir=/config/brave-profile
```

I created the directory and made it writable by the `kiosk` user:

```sh
mkdir -p config/brave-profile
chown -R 1000:1000 config/brave-profile
chmod 0700 config/brave-profile
```

That way, rebuilding or recreating the container would not throw away the browser profile. The old Chromium profile was not migrated automatically; the Brave profile started separately.

## 5. Video was working. HDMI audio was not

The TV picture arrived before the sound did. At first, the container did not even have a `/dev/snd` directory, so I checked what ALSA exposed on the host before passing anything through.

The host showed two HDA cards. The HDMI playback endpoints were on card 0. The PCM list included:

```text
00-03: HDMI 0
00-07: HDMI 1
00-08: HDMI 2
00-09: HDMI 3
```

The TV's ELD information showed a connected HDMI monitor and stereo LPCM support. That pointed me to card 0, PCM device 3, exposed as:

```text
/dev/snd/controlC0
/dev/snd/pcmC0D3p
```

I passed those two nodes into the container rather than mounting the entire host sound subsystem. The container got the audio supplementary group as well.

I then configured ALSA's default output in the container to use that HDMI PCM:

```text
pcm.!default {
    type plug
    slave.pcm "hw:0,3"
}
ctl.!default {
    type hw
    card 0
}
```

The first `speaker-test` was misleading. It completed without an ALSA error, but I could not hear anything from the TV:

```sh
docker compose exec tv-kiosk \
  speaker-test -D hw:0,3 -c 2 -r 48000 \
  -F S16_LE -t sine -f 440 -l 1
```

I checked the mixer and found the IEC958 playback switch was off. Turning it on made the difference:

```sh
docker compose exec tv-kiosk amixer -c 0 cset numid=5 on
```

I ran the same tone test again. This time I could hear the 440 Hz tone through the TV speakers. The PCM had accepted the test data all along; the digital output switch was the part that was muted.

For a less brittle startup command, I used the control's name rather than relying on numeric control ID `5`:

```sh
amixer -q -c "$ALSA_CARD_NUMBER" \
  cset "iface=MIXER,name='IEC958 Playback Switch',index=0" on
```

The manual `amixer` change is the one I explicitly confirmed by listening to the test tone. Adding the named-control command to the entrypoint is the way to apply it automatically when the container starts; I would not confuse that proposed persistence step with the original manual test.

## 6. Making startup less manual

By this point I had a browser, input, and audio, but starting it still involved knowing which DRM card, which input event numbers, which ALSA card, and which virtual terminal to use. I did not want to keep that list in my head or hardcode numbers that could change.

The first manual workaround was:

```sh
chvt 7
```

After switching to VT7, Cage could take over the display more reliably. I wanted the host to do this automatically when the HDMI display was present, then restore the previous VT when the kiosk stopped.

I started putting that logic in an Alpine OpenRC service. The controller watched DRM events with:

```sh
udevadm monitor --kernel --subsystem-match=drm
```

It did not blindly trust the event itself. A DRM hotplug event tells userspace to re-check the connector; it does not hand over a complete connected/disconnected decision. The controller therefore read the connector's status file after a short debounce:

```sh
cat /sys/class/drm/card1-HDMI-A-1/status
```

It also polled periodically in case it missed an event. The saved controller configuration used a three-second debounce, a 30-second fallback poll, a 120-second delay before stopping after a real disconnect, and a 60-second retry interval for a failed start.

The controller was also responsible for resolving the hardware paths before it started Compose. For this machine, the configuration used stable links such as:

```text
/dev/dri/by-path/pci-0000:01:00.0-card
/dev/input/by-id/usb-Compx_2.4G_Wireless_Receiver-event-kbd
/dev/input/by-id/usb-SIGMACHIP_USB_Mouse-event-mouse
```

It then resolved those links to the current `/dev/dri/cardN` and `/dev/input/eventN` nodes. It looked up the ALSA card by its ID (`Generic`) instead of assuming that it would always be card 0. The resolved paths were written to `/run/tv-kiosk/compose.env`, which Compose used when creating the container.

That was a much better approach than baking `card1`, `event0`, and `event18` permanently into the Compose file. Device names are convenient for a one-off test; they are not stable identifiers for a service that should survive reboots.

### The controller did switch to VT7 and start Compose

The OpenRC service was tested interactively. One captured run showed:

```text
2026-10-31 19:45:20 controller started; watching DRM hotplug events and polling connector state
2026-10-31 19:45:20 HDMI connector HDMI-A-1 state: connected
2026-10-31 19:45:20 Switching foreground VT from 1 to 7
2026-10-31 19:45:20 VT 7 is now active
2026-10-31 19:45:20 HDMI connected; starting tv-kiosk on VT 7
Container tv-kiosk Starting
Container tv-kiosk Started
2026-10-31 19:45:20 tv-kiosk Compose start requested successfully
```

The host agreed that VT7 was active:

```sh
cat /sys/class/tty/tty0/active
```

```text
tty7
```

That confirmed the VT switch and the Compose start request. It did not, by itself, prove that Brave stayed alive. I had to learn to check the container's actual state and logs after Compose returned.

### The stale seatd socket made retries worse

During repeated runs, I used `docker compose stop` to stop the container and then tried to start it again. That left the stopped container and its writable layer in place. The next seatd startup found the old socket:

```text
[ERROR] [seatd/seatd.c:162] Socket file found at socket path /run/seatd.sock, refusing to start
seatd exited prematurely
```

The message repeated on later attempts. At that point the controller could say that Compose had started the service, while the main process had already exited or was about to exit. A successful `docker compose up -d` return is not a health check.

For a clean manual reset, I used:

```sh
docker compose down --timeout 20
```

That removes the container so its private `/run/seatd.sock` disappears. The browser profile survives because it is a host bind mount. I did not use `-v` or `--volumes`.

There is an important detail in the saved 31 October refresh bundle: its controller's stop path still calls `docker stop`, not `docker compose down`. So although `compose down` was the useful manual recovery when I was iterating, I would not claim that the archived controller script fully solves the stale-socket retry case. That cleanup path deserves another pass before I call the automatic lifecycle finished.

## 7. The TV power button was harder than HDMI hotplug

I wanted the browser to stop when I turned the TV off. The controller could handle a real HDMI disconnect: wait 120 seconds, then stop the container. But the TV did not always tell Linux that it had disconnected.

The confusing observation was:

```text
TV switched off:
connected
```

From the kernel's point of view, the HDMI sink was still connected. The controller was doing the right thing by not stopping the browser just because a hotplug event had arrived; it re-read the status and saw `connected`.

I enabled HDMI-CEC in the TV settings and checked the host for a CEC adapter. There was no `/dev/cec*`, no `/sys/class/cec`, and `cec-ctl --list-devices` listed no adapters. The kernel configuration included CEC-related options, but that did not mean this HDMI port had a registered CEC device.

I left it there. The controller can react to a real DRM `disconnected` state, but it cannot infer that the TV panel is off while the TV keeps the HDMI connection asserted. I would need a supported CEC adapter or a different power-state signal to make that behaviour reliable.

## 8. Things that looked scary but were not all the same problem

The logs were noisy, and it would be dishonest to call every line fixed just because the browser was visible at one point.

**The D-Bus errors** were expected from running a desktop browser in a deliberately minimal container without a system D-Bus socket. Brave complained about it. That did not prevent the browser from opening, but features that depend on desktop services may not be available.

**The AMDGPU warnings** included:

```text
amdgpu_cs_ctx_create2 failed. (-13)
os_same_file_description couldn't determine if two DRM fds reference the same file description.
```

The browser could render on the TV in spite of these messages. I did not establish their exact cause, so I have left them as a graphics-permission or DRM-fd investigation rather than declaring them harmless.

**The Puffin temporary-file warning** was:

```text
Failed to create a temporary file for memory-mapping: Operation not permitted
```

It appeared in the logs but I did not trace it to a specific missing permission or feature.

**The seccomp message needed more attention than the ordinary D-Bus noise.** One Chromium run logged:

```text
**CRASHING**:seccomp-bpf failure in syscall nr=0x148 ...
```

The syscall number is consistent with `pwritev2` on this architecture, and that syscall was already present in the custom Docker seccomp profile. The separate headless test had printed the test HTML, but that did not prove the graphical browser's own sandbox was happy. I did not establish conclusively whether the failure came from Chromium's internal seccomp filter, the outer Docker filter, or an interaction between them. I kept the browser sandbox enabled rather than papering over the problem with `--no-sandbox`.

The October 31 logs also captured a container exit with code `1` and `OOM killed: false`. That is why I am careful about the word “working”: the display path did work and I saw the browser on the TV, but the saved logs also show at least one run that did not remain healthy.

## Where I left it

The useful parts of the project were real:

- Alpine stayed on the MS-A2 as the host OS. I did not install a full desktop on it.
- A containerized Cage session detected the AMD GPU and HDMI connector, and the browser was visible on the TV.
- The input path was worked out: `drm,libinput`, udev metadata, the Compx keyboard, and the separate SIGMACHIP mouse.
- Brave replaced Chromium in a Debian-based browser image, with its profile stored on the host.
- HDMI audio was confirmed with a speaker-test tone after enabling the IEC958 switch.
- An OpenRC controller could detect the connected HDMI state, switch to VT7, resolve devices, and request a Compose start.

But I would not describe the whole thing as a finished appliance. TV standby still looks like HDMI connected. The archived controller has a cleanup/retry edge case around stale seatd sockets. The seccomp crash and some AMDGPU warnings were not fully explained. And a controller log saying “Started” is not the same thing as checking that the browser process is still running.

That is the real shape of this project: I got the TV picture first, then spent the rest of the time teaching the container how to behave like a small Linux desktop without actually turning the host into one. Some pieces are solved; some are merely understood; a few are still on the list. I prefer this version of the story because it is much closer to what happened at the terminal.

## A few references I used

- [Alpine Linux: configuration management and device-manager setup](https://wiki.alpinelinux.org/wiki/Alpine_configuration_management_scripts)
- [Brave for Linux](https://brave.com/linux/)
- [wlroots environment variables](https://github.com/swaywm/wlroots/blob/master/docs/env_vars.md)
- [Docker Compose service configuration](https://docs.docker.com/reference/compose-file/services/)
- [Docker Compose down](https://docs.docker.com/reference/cli/docker/compose/down)
- [Linux DRM userspace API](https://docs.kernel.org/gpu/drm-uapi.html)
- [Linux CEC userspace API](https://docs.kernel.org/userspace-api/media/cec/cec-intro.html)
