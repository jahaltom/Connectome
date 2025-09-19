# C. elegans Connectome → Behavior Pipeline
**c302 + pyNeuroML + Sibernetic (headless-friendly)**

This README walks you from the *connectome* to *muscle activations* and finally to a *crawling worm* in Sibernetic. It includes the **working argument format** for `c302 v0.10.x` and commands that run **without a GUI** (HPC-safe).

> TL;DR: you’ll build a known‑good model, extract the exact muscle IDs, generate **MyWorm** with AVBL/AVBR drive **and** muscles wired, run headless, and plot a heatmap.

---

## 0) Prerequisites

- Python 3.9+ with `pip`
- Java (for jNeuroML)
- `pyNeuroML` (gives you `pynml` / `jnml`)
- `c302` repo (this guide assumes you’re in its root folder)

```bash
pip install pyneuroml
# Optional helpers used below
pip install numpy matplotlib
```

If your cluster is headless (no DISPLAY), we’ll run with `-nogui` so no X server is needed.

---

## 1️⃣ Get the Connectome (raw network)

If you already have **either** of these, you’re set:

- `c302_C_Full.net.nml`
- `c302_C2_Full.net.nml`

These include:

- 302 neurons (sensory, interneuron, motor) with unique names  
- Body-wall muscles (typically **96** elements: MDL/MDR/MVL/MVR 01–24)  
- Chemical synapses (weighted by synapse counts; Cook 2019 + WormWiring)  
- Gap junctions (electrical synapses)

**Peek inside:**

```bash
pynml c302_C_Full.net.nml -summary
```

---

## 2️⃣ Build a simulatable model

Fresh start (or if you want to tweak params):

```bash
git clone https://github.com/openworm/c302.git
cd c302
pip install .
```

Generate a model with parameter set **C** (spiking + muscles):

```bash
python c302/c302_Full.py C
```

You’ll get at least:

- `examples/c302_C_Full.net.nml` – the network  
- `examples/LEMS_c302_C_Full.xml` – a run‑ready LEMS file

> Some versions also split out `.cells.nml` / `.synapses.nml`; other versions inline them in `.net.nml`. Either way is fine.

**Parameter sets:** `A` = passive, `B` = graded, **`C` = spiking + muscles**, `D/E` = more biophysical detail (slower).

---

## 3️⃣ Add a drive (make it “wiggle”) — **with the fixed CLI syntax**

A bare connectome is silent. Give a tonic input to **AVB** (forward) or **AVA** (reverse). For **forward**, inject **AVBL/AVBR**.

> **Important parser quirk (c302 v0.10.x):**  
> Pass lists in **brackets with no inner quotes** → `"[AVBL,AVBR]"`.  
> For muscles, pass a **bracketed CSV** of IDs.

### 3a) Get the exact muscle IDs (do this once)
```bash
MLCSV=$(grep -oP 'id="M[DV][LR][0-9]{2}"' examples/c302_C_Full.net.nml \
        | cut -d\" -f2 | sort -V | paste -sd, -)
MLIST_BR="[$MLCSV]"
echo "$MLIST_BR"   # should look like: [MDL01,MDL02,...,MVR24]
```

### 3b) Build **MyWorm** with correct list syntax
```bash
c302 MyWorm parameters_C \
  -cells "[AVBL,AVBR]" \
  -cellstostimulate "[AVBL,AVBR]" \
  -musclestoinclude "$MLIST_BR" \
  -duration 5000
```

This writes `LEMS_MyWorm.xml` (and `MyWorm.net.nml`). The console should echo:
```
Cells: ['AVBL','AVBR']
Muscles: ['MDL01','MDL02',...,'MVR24']
```

---

### 3c) Get the runner & image
```
git clone https://github.com/openworm/sibernetic.git
cp sibernetic/sibernetic_c302.py .

# Pull the OpenWorm Docker image as a local Singularity image (SIF)
singularity pull openworm.sif docker://openworm/openworm:latest

#Quick GPU sanity check (optional)
singularity exec --nv -e openworm.sif nvidia-smi

````



### 3d) FAST preview run (recommended defaults)
```
mkdir -p ow_out



singularity exec --nv -e -B "$PWD:$PWD" --pwd /home/ow/sibernetic openworm.sif \
  env DISPLAY="" XAUTHORITY="" \
  python3 "$PWD/sibernetic_c302.py" \
  -noc302 -datareader "$PWD/MyWorm.muscles.dat" \
  -duration 5000 -device GPU -logstep 1000 -outDir "$PWD/ow_out"

```
- noc302: use your existing *.muscles.dat (don’t simulate the neural model inside the container).

- device GPU: force the GPU OpenCL device.

- duration 300: simulate 300 ms.

- dt 1e-5: larger physics step → far fewer steps (≈ 30k steps for 300 ms).

- --pwd /home/ow/sibernetic: ensures the script’s src/main.cpp version check works.

- env DISPLAY="" XAUTHORITY="": prevents a headless env crash.


## 4️⃣ Run the simulation (headless) & inspect outputs

Run **without GUI**:
```bash
pynml LEMS_MyWorm.xml -nogui
```

You should see files like:
- `MyWorm.muscles.dat` — **time × N muscles** (usually 96; some older templates write 95)  
- `MyWorm.v.dat` — voltages (if enabled)  
- `MyWorm.dat` — consolidated output (depends on template)

**If `MyWorm.muscles.dat` didn’t appear:** try the example run
```bash
pynml examples/LEMS_c302_C_Full.xml -nogui
ls -lh examples/*muscles*.dat
```

**Quick QC heatmap (works for 95 or 96 channels):**
```python
import numpy as np, matplotlib.pyplot as plt
M = np.loadtxt("MyWorm.muscles.dat")
t, vals = M[:,0], M[:,1:]
plt.imshow(vals.T, aspect='auto', origin='lower',
           extent=[t[0], t[-1], 0, vals.shape[1]])
plt.xlabel("time (s)"); plt.ylabel("muscle index")
plt.title("Muscle activation map"); plt.show()
```

Look for alternating dorsal↔ventral bands and a head→tail phase progression.

---

## 5️⃣ (Optional) Visualize network structure

```bash
pynml c302_C_Full.net.nml -graph 4c
```
This PNG shows color‑coded neuron classes, muscle nodes, and weighted edges.

---

## 6️⃣ Couple to body physics (watch it crawl)

Use **Sibernetic** to convert activations into motion.

**Docker quick‑start:**
```bash
git clone https://github.com/openworm/OpenWorm.git
cd OpenWorm
./run.sh -d 5000
```

Pipeline:
- Runs c302 (or consumes your `MyWorm.muscles.dat`)
- Feeds signals to Sibernetic
- Produces a crawling **.mp4**

**Manual override (use your own muscles file):**
```bash
python sibernetic_c302.py -noc302 -datareader MyWorm.muscles.dat -duration 5.0
```

---

## 7️⃣ Add sensory inputs & “friend” behaviors

Gentle touch (ALM) with forward drive:
```bash
c302 Friend parameters_C \
  -cells "[ALML,ALMR,AVBL,AVBR]" \
  -cellstostimulate "[ALML,ALMR]" \
  -duration 3000

pynml LEMS_Friend.xml -nogui
```
Expect a pause/reversal via AVA/AVD recruitment.

---

## 8️⃣ (Advanced) Close the loop

Make it more realistic:
- **Proprioception:** feed curvature back to motor neurons (stabilizes gait)
- **Noise:** add jitter to AVB drive (natural variability)
- **Reversal reflex:** transient AVA drive when head contacts a wall

You can **replace** the muscle signals with your own CPG generator (e.g., Kuramoto+feedback), export to `*.muscles.dat`, and run Sibernetic → closed‑loop worm with “personality.”

---

## Troubleshooting

- **Headless error (`No X11 DISPLAY`)**  
  Always run with `-nogui`:
  ```bash
  pynml LEMS_MyWorm.xml -nogui
  # or
  jnml LEMS_MyWorm.xml -nogui
  ```
  As a last resort: `xvfb-run -a pynml LEMS_*.xml`

- **Parser truncation bug (v0.10.x)**  
  If you pass `AVBL,AVBR` it becomes `VBL,AVB`.  
  **Fix:** bracketed lists, **no inner quotes** → `"[AVBL,AVBR]"`.  
  Do the same for muscles: `-musclestoinclude "[$CSV]"`

- **No `MyWorm.muscles.dat`**  
  Run the example LEMS:  
  ```bash
  pynml examples/LEMS_c302_C_Full.xml -nogui
  ls examples/*muscles*.dat
  ```
  Or extract from a consolidated `MyWorm.dat` if headers include `muscle_*` columns.

- **Fetch muscles from your model** (if needed):
  ```bash
  grep -oP 'id="M[DV][LR][0-9]{2}"' MyWorm.net.nml | cut -d\" -f2 | sort -V | uniq
  ```

---

## Appendix: One‑shot helper script

Save as `worm_pipeline.sh` in the `c302/` folder, then: `bash worm_pipeline.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Step 1: Build example model ==="
python3 c302/c302_Full.py C

echo "=== Step 2: Extract muscles ==="
MLCSV=$(grep -oP 'id=\"M[DV][LR][0-9]{2}\"' examples/c302_C_Full.net.nml | cut -d\" -f2 | sort -V | paste -sd, -)
MLIST_BR="[$MLCSV]"
echo "Muscles: $MLIST_BR" | head -c 160; echo " …"

echo "=== Step 3: Build MyWorm (AVBL/AVBR, muscles wired) ==="
c302 MyWorm parameters_C \
  -cells "[AVBL,AVBR]" \
  -cellstostimulate "[AVBL,AVBR]" \
  -musclestoinclude "$MLIST_BR" \
  -duration 5000

echo "=== Step 4: Run headless ==="
pynml LEMS_MyWorm.xml -nogui

echo "=== Step 5: Plot heatmap (if muscles file exists) ==="
python3 - <<'PY'
import os, numpy as np, matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

fn = "MyWorm.muscles.dat"
if not os.path.isfile(fn):
    print("No MyWorm.muscles.dat found; skipping plot."); raise SystemExit(0)
M = np.loadtxt(fn); t, vals = M[:,0], M[:,1:]
plt.figure(figsize=(10,4), dpi=140)
plt.imshow(vals.T, aspect='auto', origin='lower', extent=[t[0], t[-1], 0, vals.shape[1]])
plt.xlabel("time (s)"); plt.ylabel("muscle index"); plt.title("Muscle activation heatmap")
plt.colorbar(label="activation"); plt.tight_layout(); plt.savefig("muscles_heatmap.png")
print("Wrote muscles_heatmap.png")
PY

echo "Done."
```

---

### Credits
- OpenWorm, c302, Sibernetic, pyNeuroML / jNeuroML communities ❤️
- This README tailored for c302 **v0.10.x** CLI behavior on headless systems.
