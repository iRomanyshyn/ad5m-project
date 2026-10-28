# AD5M-024 Display, camera and USB topology

Status: in progress

## Goal

Establish the AD5M's actual display, touchscreen, camera and USB topology
well enough to make safe, minimal integration decisions and to evaluate a
possible internally mounted USB-storage device without guessing connector
pinouts, power capability or device identity.

## Context

The project currently needs a reliable framebuffer/touch target for a future
HelixScreen package, a stable camera path, and evidence for or against a
user-enabled internal USB-storage profile.  The stock host owns the kernel and
most board support; the chroot cannot establish physical wiring merely from
the presence of `/dev` nodes.

The vendor's [published firmware log](https://www.flashforge.com/fr/a/docs/adventurer-5m-series/adventurer-5m-series-firmware-update-log)
says that AD5M firmware gained automatic external-camera detection in version
2.3.6.  The [product documentation/download page](https://www.flashforge.com/blogs/download-document/adventurer-5m)
exposes one user USB port and a 4.3-inch touchscreen, but it does not publish
a board schematic, display pinout, USB power budget, device tree or kernel
configuration.  The official Klipper reference audited in AD5M-014 likewise
does not contain those host-side sources.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod, only after an evidence-backed integration decision

## Observed software interfaces

| Surface | Current project behavior | What this establishes | What it does not establish |
| --- | --- | --- | --- |
| Display pixels | `S90welcome` and status scripts write pre-rendered images to `/dev/fb0`. | The stock-host kernel exports a framebuffer usable by the chroot. | Resolution, pixel format, panel bus, compositor ownership, safe mode changes, or the touchscreen mapping. |
| Display backlight | The pinned external helper opens `/dev/disp`; its numeric ioctls are tracked by AD5M-020. | A live device node and observed compatibility behavior exist. | The Linux display ABI, PWM ownership, electrical wiring or a safe reimplementation. |
| Camera | mdev invokes `camera-autostart.sh` for `video[0-9]`; the script only acts on `/dev/video0`, requires MJPEG, selects a resolution at least 1280 pixels wide if possible, then starts ustreamer. | The integration expects a V4L2 camera; an MJPEG USB camera is the supported current software path. | That every UVC camera works, that `/dev/video0` is stable with several devices, or that the camera uses a dedicated physical port. |
| USB mass storage | mdev invokes `automount.sh` for `sd[a-z].*`; it mounts a VFAT volume read-only below `/media` and links it into G-code storage. | The present integration treats USB mass storage as removable import media. | Stable per-device identity, writable storage, multiple-device behavior, safe swap, or any internal connector's power/data wiring. |

`lsusb` and mdev are selected in the image, but there is no project policy for
USB topology, serial-number binding, a persistent mount, zram, or an internal
storage device.  Camera and storage can therefore contend for the same host
USB resources and `/dev/video0` may become a poor assumption when more than
one V4L2 device exists.

## Physical-source evidence and limits

The following photo evidence was inspected only as an untrusted research
input under ADR-0004.  At commit
`d6beeee8c21861fa107da8aeec8628f5a273d857` of
[`g992/flashforge-ad5m-5mpro-research`](https://github.com/g992/flashforge-ad5m-5mpro-research),
the annotated base-5M board photograph shows an Allwinner T113-S3 board and a
Genesys Logic GL852G USB-hub IC, plus a flat-flex connector visibly marked
`LCD`.  Its author annotates two unpopulated
connectors as possible free USB connections and separately speculates about a
TVOC/serial connector.  A close-up annotation associates D+/D- traces with
the hub, but there is no schematic, continuity measurement, USB enumeration,
power measurement, panel-bus/touch mapping, or proof that the observation
applies to every base/Pro board revision.

Therefore none of those connectors may be treated as USB or connected to
without a per-printer physical audit.  In particular, the TVOC/RX/TX
observation is **not** evidence that it is a camera or a USB port.  The photo
is useful only for locating candidates after the board revision is confirmed.

An independent teardown/review places the user USB-A port on the left side of
the screen and reports that the base 5M accepts an external USB camera there.
That agrees with the current V4L2 integration and vendor automatic-camera
release note, but it still does not reveal the full hub topology or the
available 5 V current.

## Candidate directions

- Keep future screen work at the documented kernel interfaces: framebuffer
  plus a physically identified touchscreen input node.  Do not infer panel
  timing, FFC pins, or `/dev/disp` ioctl layout from photographs.
- Improve camera integration only after discovery data exists: bind to a
  reviewed V4L2 device identity/capability instead of assuming `video0`, and
  preserve the local-LAN security boundary in AD5M-004.
- Treat the visible USB-A port as shared/removable until the actual topology
  is measured.  It is suitable for camera or import media today, not a safe
  default place for a hidden swap disk.
- An internally secured low-power USB device or downstream hub is a candidate
  only after the audit below.  It must be an optional, explicitly provisioned
  profile; absence, a reset or a disconnect must neither delay boot
  indefinitely nor stop printing.  AD5M-006 owns the swap decision itself.
- Never solder to a candidate header, borrow 5 V from an unknown rail, or
  attach D+/D- based only on a photo.  Do not place a storage device near hot
  zones, moving belts, mains wiring or safety-critical harnesses.

## Physical audit plan

With the printer powered down before opening it, record model/base-or-Pro,
mainboard revision markings and stock-host/firmware version.  Photograph
connectors before touching them.  Then, with no change to wiring, collect a
read-only baseline from the running system:

1. `lsusb`, `lsusb -t`, relevant `dmesg` lines, and `/sys/bus/usb/devices`
   identities; repeat for the existing camera and a known USB drive, both
   alone and together.
2. `/dev/fb0`, `/sys/class/graphics`, framebuffer geometry/format, and
   `/proc/bus/input/devices` to identify the screen and touch input without
   injecting touch or display commands.
3. `v4l2-ctl --all` and format/frame-size enumeration for each video node;
   record camera serial/path where exposed.
4. For a proposed internal USB point, verify continuity and voltage with
   appropriate tools while unpowered first, then measure startup and loaded
   5 V behavior.  Check hub-port enumeration and camera/storage coexistence
   before mounting a disk or enabling swap.

Any electrical modification needs its own approved procedure, strain relief,
insulation, thermal/mechanical clearance and rollback/removal plan.  It is
not covered by software acceptance.

## Acceptance criteria

- The chosen display and touch runtime interfaces are identified on each
  tested board revision without changing panel timing or backlight hardware.
- Camera and storage enumeration demonstrate whether they share a hub and how
  identities remain stable with both attached.
- An internal USB candidate has measured power, topology, heat and mechanical
  evidence before it is used for any data.
- The machine remains bootable and printable when the camera, removable media
  or optional internal device is absent or fails.
- Every conclusion distinguishes official documentation, local software
  evidence, community observation and physical verification.

## Verification

2026-10-28 source-only audit: local overlays and mdev rules were inspected;
the official Flashforge manual/release information and a read-only community
photo repository were consulted.  The community repository was checked at
the commit above and `git fsck --no-dangling` passed.  No community script,
archive, firmware, wiring instruction or hardware modification was used.

## Hardware verification

Not run.

## Remaining work

Perform the physical audit plan on the actual target before selecting an
internal storage location, changing camera discovery, or enabling a display
service.
