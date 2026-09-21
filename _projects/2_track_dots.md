---
layout: page
title: Track Dots
description: Real-time marker tracking in C++ that turns a soft actuator's shape into curvature feedback for closed-loop control
importance: 2
category: work
github: https://github.com/tejodeshpande/rt-track/tree/rs2_rt
---

A soft pneumatic actuator has no joints and no encoders, so there is nothing to read its position from. Its shape _is_ its state — and the only practical way to measure that in real time is to watch it.

**Track Dots** is the C++ tool that does the watching: it follows markers along a bending actuator, converts their positions into per-segment curvature, and feeds that back to the pressure controller. It is the vision and control tooling behind the [pneumatic soft actuator work]({{ '/publications/' | relative_url }}).

**Source:** [`tejodeshpande/rt-track`](https://github.com/tejodeshpande/rt-track/tree/rs2_rt) — the `rs2_rt` branch.

## From pixels to curvature

The pipeline is deliberately short, because every stage costs frame time:

1. **Select** — 13 marker points are picked interactively on the first frame through an OpenCV mouse callback.
2. **Track** — an OpenCV `MultiTracker` running the **KCF** algorithm follows all 13 across frames.
3. **Fit** — each consecutive triplet of markers defines a circle. Its circumcentre and radius give the local radius of curvature, yielding **6 curvature segments** along the actuator.
4. **Scale** — a calibration constant converts pixels to millimetres, so the output is physical rather than image-space.

This is a **piecewise constant curvature** model, the standard way to describe a continuum robot: approximate a smoothly bending body as a chain of constant-curvature arcs. Three tracked points are enough to pin down each arc.

## Closing the loop

Curvature estimates are only useful if something acts on them. `curv_ctrl.h` opens a serial link to the actuator's microcontroller and drives **7 independently pressurised chambers**, holding per-chamber pressure setpoints and accumulating error over time so steady-state offsets get corrected rather than tolerated.

The result is a loop that runs entirely at the camera's pace: observe shape, compute curvature, adjust pressure, repeat.

## Implementation notes

- **C++17**, built with CMake against **OpenCV**
- Optional **Intel RealSense** (`librealsense2`) support for depth-capable capture, present but currently commented out in favour of a plain `VideoCapture`
- A shared-memory (`mmap`) hook for handing tracking output to another process, scaffolded but not enabled
- Branches trace the evolution: `oops_track` and `oops_track_mac` are the earlier object-oriented rewrites, `rs2_rt` is the current real-time version with curvature control

The header split keeps the concerns apart — `tracking.h` for the vision, `pcc_calc.h` for the geometry, `curv_ctrl.h` for the hardware loop, `userio.h` and `userdatastruct.h` for interaction and shared state.
