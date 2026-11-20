# Credits

This repository is a small delta on other people's work. All of it is
LGPL-2.1-or-later.

* **libfprint / fprint project** — the library and the `elan` driver this patch
  modifies: <https://gitlab.freedesktop.org/libfprint/libfprint>, base version
  `v1.94.100`. Includes Bozorth3 from NIST NBIS (see `LICENSE`).
* **`libfprint-elan-0c28-patch`** (AUR, maintainer *khaosdoctor*) — press-mode
  capture, SIGFM matcher, calibration retries and stripe rejection for the
  sibling ELAN `04f3:0c28` 96×96 sensor. This repo's patch is that work ported
  to `04f3:0903`.
* **SIGFM** (`libfprint/drivers/sigfm.[ch]`) — Copyright (C) 2022
  Matthieu Charette, Natasha England-Elbro, Timur Mangliev; 2026 libfprint
  contributors. LGPL-2.1-or-later.
* **`krabus/libfprint-elan`** — prior art for the same sensor (swipe-mode +
  2× upscale + lowered Bozorth3 threshold). Not used here: it completes
  enrollment but verification stays at score 0 for a press sensor. Recorded so
  the next person does not repeat that path.
* **`iafilatov/libfprint`** — the original ELAN driver work upstreamed into
  libfprint.

Changes in this repository relative to upstream v1.94.100: see
`elan-0903-press-sigfm.patch` (9 modified files + 2 new files, ~450 inserted
lines). Modifications by the repository author, 2026.
