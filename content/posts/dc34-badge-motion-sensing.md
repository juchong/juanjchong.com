+++
title = "Motion Sensing on the DEF CON 34 Badge"
date = "2026-08-09"
author = "Juan Chong"
tags = ["Projects", "Embedded", "Rust", "Hardware Hacking"]
description = "Using the badge's existing accelerometer to work out what its wearer is doing, with everything from data collection to inference running on the badge itself"
cover = "/images/dc34-badge-motion-sensing/dc34-badge-motion-sensing.png"
ogImage = "/images/dc34-badge-motion-sensing/og.png"
coverAnimated = "/images/dc34-badge-motion-sensing/dc34-badge-motion-sensing.mp4"
coverCredit = "The badge on a desk, correctly deciding that nobody is wearing it"
Toc = false
+++

The DC34 badge has a LIS2DH12 accelerometer in it, and the stock firmware gives it two
jobs: wake the screen when you pick the badge up, rotate it when you turn it over. The
sensor runs continuously and everything else it measures gets thrown away.

This project uses that same sensor to work out what the person wearing the badge is
doing. Walking, standing, sitting, or not being worn at all. The badge shows its
current guess as an icon on the idle screen and updates it about once a second.
There's no phone in the loop, no network, and no server. The recognition happens on
the badge.

None of that comes off a shelf. Nobody has a dataset of this accelerometer hanging off
a lanyard at chest height, so the data had to be recorded first. Whatever gets trained
on it then has to be small enough and cheap enough to evaluate on a 32-bit RISC-V core
with no floating-point unit, and it has to keep doing that all day without getting in
the way of the wake and rotation the sensor is already responsible for.

If you want background on the hardware, the [Coder's guide to the Baochip
1x](https://baochip.github.io/baochip-1x/) covers the SoC and
[Xous](https://github.com/betrusted-io/xous-core) is the microkernel it runs.

## Part one: recording motion

Before training can begin, labelled accelerometer data must be captured from the badge. The [`datalog`](https://github.com/juchong/dc34-console/blob/main/src/cmds/datalog.rs) command streams each sample over USB as a CSV line, tagged with its class. A host script drives the session, which is practical—typing shell commands while wearing the recording device is awkward. The script lists the classes, guides each recording, and limits every take to five minutes. Four classes at five minutes each yields roughly 1,200 training windows.

In the base firmware, the power manager owns the sensor. It claims its interrupt line and handles wake / screen rotation. The logger operates as a secondary reader since another process already brought the device up. It also leaves the latched interrupt registers untouched. 

The real challenge is identifying when a recording is unusable. In the CSV, a late sample and a missing sample look similar but represent different failures. Lateness is mere jitter, costing a window or two. Missing data is worse since absent rows stretch the next window, compress time, and misalign the motion label.

None of this is immediately apparent unless it is checked. The end-user result is simply a degraded model with no clear cause. The firmware solves this by tagging each type of data loss as it occurs, then tallying them in a trailer at the end of each take. The host script validates the trailer, rejects flawed recordings, and requests an immediate redo. Recording and training now share a single standard for a “usable” take, so clean-data criteria never diverge.

Source:

- [`accel_logger.rs`](https://github.com/juchong/dc34-console/blob/main/src/accel_logger.rs):
  the CSV format and the loss counters
- [`capture.py`](https://github.com/juchong/dc34-console/blob/main/host/capture.py):
  records one take, and `take_quality` decides whether it was any good
- [`collect.py`](https://github.com/juchong/dc34-console/blob/main/host/collect.py):
  the guided session
- [`lis2dh12.rs`](https://github.com/juchong/xous-core/blob/dev/libs/bao1x-hal/src/lis2dh12.rs):
  `attach()`, the handle that configures nothing
- [`HOWTO.md`](https://github.com/juchong/dc34-console/blob/main/host/HOWTO.md):
  how to run the loop yourself

## Part two: training a model that fits

On the laptop, the CSV files are sliced into two-second windows of 100 samples with 50% overlap. Each window is reduced to 37 numbers: nine statistics each for the x, y, and z axes, plus a computed magnitude channel and a signal area across all three. These include mean, standard deviation, min, max, range, mean absolute deviation, RMS, zero crossings, and jerk.

Every value is computed in integer arithmetic, since the badge must replicate the exact calculations without an FPU. Zero crossings are the standout: counting sign changes of the mean-centered signal yields a reliable dominant-frequency proxy using a single comparison per sample—a far cheaper alternative to a fixed-point FFT. Walking produces a distinct gait frequency, speech vibration registers much higher, and a stationary badge shows neither.

Training is straightforward by design. A default `sklearn` decision tree capped at depth six produces only 15 nodes, serialized into five `const` arrays totaling ~150 bytes. Badge-side inference is a simple cascade of integer comparisons, requiring no multiplies. On 1,196 windows, the model scores 0.972 on a chronological holdout and 0.994 on a random split. The chronological metric is the true measure, since the 50% overlap means a random split leaks near-identical windows across the train/test boundary.

The core issue is dual computation: features are calculated in Python during training and in Rust during inference. If they diverge, the model evaluates against mismatched feature values, causing a silent accuracy drop. The culprit was a single edge case: Rust’s integer division truncates toward zero, while Python’s `//` floors toward negative infinity. This discrepancy flips every negative result, altering nine of the 37 features in every window.

A thorough fix required more than a simple patch, so training now emits golden vectors—eight representative windows paired with their exact expected features and classes. A separate `check` crate verifies that the device code reproduces each one precisely. Because the badge target is cross-compiled and lacks the host toolchain, the validation logic lives outside the main firmware crate, leaving feature and model code strictly dependency-free.

Source:

- [`clf_features.rs`](https://github.com/juchong/dc34-console/blob/main/src/clf_features.rs):
  the 37 features as the badge computes them
- [`train.py`](https://github.com/juchong/dc34-console/blob/main/host/train.py):
  the same math in Python, the tree, and the code generation. `tdiv` is the truncating
  division, `channel_features` is what has to agree line for line
- [`clf_model.rs`](https://github.com/juchong/dc34-console/blob/main/src/clf_model.rs):
  the generated tree and the walk down it
- [`clf_golden.rs`](https://github.com/juchong/dc34-console/blob/main/src/clf_golden.rs):
  the eight vectors the two implementations are held to

## Part three: running it on the badge

Both the logger and the classifier require a stream of samples, supplied by exactly one thread. Two pollers would double I2C traffic and produce mismatched timestamps, making it impossible to align a recording with the classifier's view. Consumers register a sink; the thread starts with the first one and exits with the last, meaning an idle badge polls nothing.

Classification runs on its own thread rather than inside the sampler callback. This split is functional, not stylistic: the sampler holds a lock while it fans samples out. Performing a window copy and 37 feature calculations inside the callback would distort the sample cadence for everyone else on the stream.

The classifier’s refusal to guess is just as critical as its ability to classify. Since features like jerk sums and crossing counts scale with the sample period, the model records the training period and will not run unless the sampler matches it. It also skips windows with timing gaps, because a discontinuity would trigger a confident but incorrect classification. The system prefers to discard flawed windows rather than risk a misclassification.

Source:

- [`accel_source.rs`](https://github.com/juchong/dc34-console/blob/main/src/accel_source.rs):
  the one sampler and the sinks that feed off it
- [`motion_clf.rs`](https://github.com/juchong/dc34-console/blob/main/src/motion_clf.rs):
  windowing, the classifier thread, and both refusals
- [`motionclf.rs`](https://github.com/juchong/dc34-console/blob/main/src/cmds/motionclf.rs):
  the shell command, which reports gap skips and the worst sample delta when you need to
  know if the sampler is keeping up

## Part four: putting it on the screen

The badge's 128×128 one-bit display is owned by the vault app. Since the classifier runs in a separate process, both write to the same framebuffer without syncing, so the vault overwrites the classifier's label. The fix removes the classifier's drawing entirely; the vault now queries the current class over IPC during its repaint cycle.

The icons are solid silhouettes based on the AIGA/DOT figures you see on crossing signals and restroom doors. At this small size, thin lines get lost, so solid shapes hold up better. They're scaled by exactly 3× using pure pixel replication, which avoids the jagged edges that plagued the earlier 2.875× attempt. A single menu item toggles the entire display on and off.

This work also surfaced a bug entirely unrelated to motion sensing. While you're actually walking, the firmware thinks the badge is idle and blanks the screen after 36 seconds. Swing the lanyard and it wakes up; walk smoothly and it dims. The culprit is `INT1_CFG`. It looks like a simple threshold register, but it's really a direction-change detector. `0x7F` is set up for 6-direction movement recognition, which only fires when the acceleration vector actually crosses from one directional zone to another. `0x3F`, on the other hand, fires on any acceleration spike. Carried steadily in one orientation, the badge never leaves its zone, so no interrupt fires no matter how far you've walked. Lowering the threshold doesn't help. It just makes the wrong question more sensitive.

Source:

- [`ux.rs`](https://github.com/juchong/dc34-vault/blob/main/src/ux.rs):
  the idle screen layout and how the label gets sized
- [`gen_motion_icons.py`](https://github.com/juchong/dc34-vault/blob/main/tools/gen_motion_icons.py):
  ASCII art in,
  [`motion_icons.rs`](https://github.com/juchong/dc34-vault/blob/main/src/bitmaps/motion_icons.rs)
  out
- [`idlemenu.rs`](https://github.com/juchong/dc34-vault/blob/main/src/idlemenu.rs):
  the menu item
- [`lib.rs`](https://github.com/juchong/dc34-api/blob/main/src/lib.rs) and
  [`motion_classes.rs`](https://github.com/juchong/dc34-api/blob/main/src/motion_classes.rs):
  the opcodes the two processes talk over, and the class names generated with the model
- [`power.rs`](https://github.com/juchong/dc34-console/blob/main/src/power.rs):
  the interrupt tuning behind that last bug

## Part five: summary

Four classes are trained and working: walking, sitting, standing, and off-human. 

The code is spread across four forks, each with a README that goes deeper than this
post does: [dc34-console](https://github.com/juchong/dc34-console) has the sampler,
logger, classifier and training pipeline,
[dc34-vault](https://github.com/juchong/dc34-vault) has the screen and icon work,
[dc34-api](https://github.com/juchong/dc34-api) carries the IPC between them, and
[xous-core](https://github.com/juchong/xous-core) has the accelerometer driver fixes
underneath it all.
