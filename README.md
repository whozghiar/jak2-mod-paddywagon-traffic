# Paddy Wagon Traffic — Jak 2

<p align="center">
  <img src="https://img.shields.io/badge/OpenGOAL-Mod-blue.svg" alt="OpenGOAL Mod">
  <img src="https://img.shields.io/badge/Game-Jak%202-orange.svg" alt="Target Game">
  <img src="https://img.shields.io/badge/AI--assisted-Modding-purple.svg" alt="AI Assisted">
</p>

---

> [!NOTE]
> This mod moved from the `jak2/features/paddywagon/traffic` branch of [whozghiar/jak-project](https://github.com/whozghiar/jak-project) to this repository. Earlier releases stay installable from the launcher catalog.

## 📖 Overview

The Krimzon Guard **paddy wagon** — the armoured prisoner van you chase during
*Escort Brutter* — now drives Haven City's ordinary street traffic, with a
**civilian prisoner standing arms-crossed in the rear cage** and a **Crimson
Guard at the controls**. It is a real guard vehicle: red dot on the minimap,
joins the hunt during an alert, and **can be boarded and driven** — stealing it
raises the city alarm exactly like stealing a hellcat or a guard bike.

- **Target Game:** Jak 2
- **Repository:** [`whozghiar/jak2-mod-paddywagon-traffic`](https://github.com/whozghiar/jak2-mod-paddywagon-traffic)

## ✨ Key Features

- **A paddy wagon in ordinary traffic.** `paddywagon-v` is a `vehicle-guard`
  woven into the city's ground traffic pool (up to 2 at a time). It uses the
  retail `paddy-wagon` hull and its retail physics constants, so it drives and
  handles exactly like the mission van.
- **A random civilian prisoner in the cage.** Every wagon carries a `norm`,
  `fat` or `chick` citizen standing in the rear compartment, dressed from the
  same random wardrobe as the pedestrians in the street. `norm` and `fat` hold
  the retail *arms-crossed* pose; `chick` stands at ease (retail has no
  arms-crossed animation for that body type).
- **Crimson Guard driver.** The regular red `crimson-guard-rider`, in the same
  pose it uses in a hellcat.
- **Drivable and stealable like any guard ship.** Walk up for the "press
  triangle" prompt, or hang off a flank rail first if it is passing above you,
  then take it. The guard is thrown clear, the city alert jumps and a Crimson
  Guard respawns on the street — all the stock guard-vehicle theft behaviour.
  The prisoner stays locked in his cage and rides along with you.
- **It runs when you shoot it.** A van carrying a prisoner does not stop to
  fight: take a shot at one and it floors the throttle, pushes past traffic and
  takes every turn that leads away from you for the next 12 seconds, never
  turning to give chase.
- **Both traffic lanes, and your gun stays out.** Once you are driving, **R2**
  drops the wagon from the high air lane down to the low one and back up, and
  **R1** fires Jak's own weapon — the wagon handles like a car in that respect,
  not like a hellcat.

## 🎮 Controls & Usage

Only while Jak is piloting the paddy wagon:

| Input | Action |
|---|---|
| **Triangle** (near the wagon) | Board it — or grab a side rail and hang, if it is passing above you |
| **Triangle** (while hanging) | Climb in and take the controls (this is the theft — the alarm goes off) |
| **R2** | Toggle between the **high air lane** (the default) and the **low lane** |
| **R1** | Fire Jak's equipped gun while driving |
| **Triangle** (while driving) | Get out |

Everything else — throttle, steering, brake, boost — is the standard city
vehicle control set.
- **Unarmed.** The `paddy-wagon` skeleton has no gun joint, so the wagon rams
  and pursues but never shoots.
- **OFF by default,** switchable from the in-game Mods menu (**L3 + SELECT** in retail boot) `Mods ▸ paddywagon-traffic`.

## 🚀 Step-by-Step Guide to Run the Mod

### 1. Select the Active Game
Make sure your environment is targeting Jak 2:
```bash
task set-game-jak2
```

### 2. Binary Compilation
- **Status:** Not required — no C++ was changed. The stock `gk` / `goalc` /
  `decompiler` binaries are sufficient.
- **Details:** This mod only touches GOAL sources, `.gd` DGO manifests and one
  `decompiler/config` **data** key (`extra_art_groups_by_dgo`), which the
  existing decompiler already understands. If your `out/build` is empty, build
  once with `task build-release`.

### 3. Asset Extraction
- **Status:** **Required — `task extract`.**
- **Details:** The paddy wagon's merc geometry only ever shipped in
  `LMEETBRT.DGO`. `extra_art_groups_by_dgo` bakes it into `lwidea.fr3`,
  `lwideb.fr3` and `lwidec.fr3` so the model is renderable in free roam.
  **Without this step the wagon is completely invisible** (the process still
  runs — sounds, collision and the riders are all there).
```bash
task extract
```

### 4. Launch the Game
```bash
task boot-game
```
*(Or launch via the OpenGOAL REPL using `task repl`, then compile and run with `(mi)` and `(r)`).*

### 5. Enable the Mod
The mod ships **OFF**. In game:
Open the Mods menu with **L3 + SELECT** (retail and debug boot) `Mods ▸ paddywagon-traffic ▸ Enable`.
The choice persists across level reloads and **takes effect immediately**
(ambient traffic pools are recycled in real-time via `'kill-all` and `'spawn-all`,
so paddy wagons appear without having to reload the city). Turn it off to
restore vanilla traffic immediately. Drive around Haven City and watch for a
boxy armoured van in the car lanes with a figure standing in the back.

## 🎥 Demonstration Video

[![Demonstration Video](https://img.youtube.com/vi/x6uEJcHKudg/maxresdefault.jpg)](https://youtu.be/x6uEJcHKudg)

▶️ **[Watch the demonstration video on YouTube](https://youtu.be/x6uEJcHKudg)**

## 📖 Technical Documentation
For the complete technical breakdown, architecture, and developer notes, refer to:
- 📄 [`docs/modding/current_mod/paddywagon_traffic_readme.md`](docs/modding/current_mod/paddywagon_traffic_readme.md)
