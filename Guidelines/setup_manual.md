# Complete Ubuntu ML/DL Environment Setup Manual

## Machine
- **CPU:** Intel i7-11800H
- **RAM:** 16 GB 3200 MHz
- **GPU:** NVIDIA GeForce RTX 3060 Laptop GPU (6 GB VRAM)
- **Primary use case:** ML/DL work on Ubuntu with Jupyter, PyTorch, and CUDA
- **Project environment name:** `gait_fusion`

---

## 1. Goal of the setup

The goal was to build a stable, reproducible Ubuntu ML/DL environment where:

- Ubuntu detects and uses the NVIDIA GPU properly
- PyTorch can use CUDA
- Jupyter runs with the correct environment
- DataLoader multiprocessing works
- Core ML/DL packages are available
- The whole stack is tested end-to-end instead of assumed to be working

### Big picture
There are 4 layers in this setup:

1. **System GPU layer** — Ubuntu + NVIDIA driver must work
2. **Python environment layer** — Miniconda/conda environment must work cleanly
3. **Framework layer** — PyTorch must be installed with CUDA support
4. **Workflow layer** — Jupyter, packages, notebook kernel, smoke tests, and project code

We verified all 4.

---

## 2. Why Ubuntu was chosen

You switched from Windows because of training slowdowns and worker-related friction.

### Practical reasons Ubuntu is better for this workflow
- Linux generally handles PyTorch multiprocessing and data loading better
- `num_workers > 0` is more reliable in Linux workflows
- CUDA/PyTorch/NVIDIA tooling is usually smoother on Linux
- Ubuntu is a strong practical choice for ML/DL day-to-day work

---

## 3. Detecting the recommended NVIDIA driver

### Command
```bash
ubuntu-drivers devices
```

### What happened
Ubuntu listed several possible NVIDIA drivers and marked this one as recommended:

```bash
nvidia-driver-580-open - distro non-free recommended
```

### Meaning
This meant:
- Your RTX 3060 was detected correctly
- Ubuntu knew which driver matched your system best
- We should install the recommended driver instead of choosing randomly

### Important note
Warnings like this were not the real issue:

```bash
udevadm hwdb is deprecated. Use systemd-hwdb instead.
```

Those were just warning/noise lines and did not affect the actual recommendation.

---

## 4. Installing the NVIDIA driver

### Command
```bash
sudo apt install -y nvidia-driver-580-open
sudo reboot
```

### Why
The system-level NVIDIA driver must be working before Python frameworks can use the GPU.

**Analogy:**
- NVIDIA driver = road
- PyTorch CUDA = car

Without the road, the car cannot move.

---

## 5. Verifying the NVIDIA driver

After reboot, verify with:

```bash
nvidia-smi
```

### What the result confirmed
Important details observed:
- Driver Version: `580.126.09`
- CUDA Version shown by driver: `13.0`
- GPU visible: `NVIDIA GeForce RTX 3060`
- VRAM visible: about `6144 MiB`

### Meaning
This proved:
- The NVIDIA driver was installed correctly
- Ubuntu could see the GPU
- The GPU was healthy enough to be used by CUDA software

### Important note
At this stage, this only proved:
- **Ubuntu can see the GPU**

It did **not yet prove**:
- **PyTorch can use the GPU**

That was verified later.

---

## 6. Choosing Miniconda vs Anaconda

### Decision
We chose **Miniconda**.

### Why Miniconda
- Lighter than full Anaconda
- Cleaner for project-based environments
- Uses less storage
- Better when you want only what you actually need

**Analogy:**
- Anaconda = giant toolbox with many preloaded tools
- Miniconda = clean workshop where you install only the tools you need

For this ML setup, Miniconda was the better choice.

---

## 7. Installing Miniconda

### Commands
```bash
cd ~
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

### During installation
When prompted:
- accepted the license
- accepted the default installation path
- said **yes** to shell initialization

### Why we said yes to shell initialization
Because it makes `conda` available automatically in new terminal sessions.

Without this, every terminal session becomes more annoying to use.

---

## 8. Refreshing the terminal after Miniconda install

After installation, running:

```bash
conda --version
```

inside the **same terminal** produced:

```bash
conda: command not found
```

### Why this happened
Because the current shell had not reloaded the new conda initialization.

### Fix
- Close the terminal completely
- Open a new terminal
- Run again:

```bash
conda --version
```

### Output observed
```bash
conda 26.1.1
```

### Meaning
Miniconda was installed successfully.

---

## 9. Understanding `(base)`

After opening a new terminal, the prompt showed:

```bash
(base)
```

### Meaning
This means the default conda environment named `base` is active.

### Best practice
Do **not** do project work in `base`.

### Why
- `base` should stay clean
- Project dependencies can conflict over time
- Separate environments keep things more stable

**Analogy:**
- `base` = control room
- project environment = work room

---

## 10. Creating the project environment

You wanted the environment name to be:

```bash
gait_fusion
```

### Command
```bash
conda create -n gait_fusion python=3.10 -y
```

---

## 11. Conda Terms of Service issue

When creating the environment, conda stopped with a Terms of Service error.

### Meaning
Conda required acceptance for these channels:
- `https://repo.anaconda.com/pkgs/main`
- `https://repo.anaconda.com/pkgs/r`

### Fix commands
```bash
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

Then create the environment again:

```bash
conda create -n gait_fusion python=3.10 -y
```

---

## 12. Activating the environment

### Command
```bash
conda activate gait_fusion
```

### Verifying Python version
```bash
python --version
```

### Output observed
```bash
Python 3.10.20
```

### Meaning
The environment was created and activated successfully.

---

## 13. Why Python 3.10 was chosen

Python 3.10 is a very safe ML/DL compatibility choice.

### Why not use a very new Python version immediately
Very new Python versions sometimes break compatibility with:
- older ML repos
- research code
- course code
- some libraries
- notebook examples found online

### Why 3.10 is a good balance
It gives:
- strong compatibility
- good stability
- broad support across ML/DL libraries

---

## 14. Pip vs conda for PyTorch

You asked whether `conda install` would be better than `pip install`.

### Final decision
For this environment, we used **pip** for PyTorch.

### Why
- The current main official PyTorch install flow strongly supports pip wheels
- CUDA-specific wheels are easy to install through pip
- Once PyTorch is installed with pip in the environment, it is cleaner not to heavily mix conda and pip afterward

### Practical rule
- Using conda to **create environments** = good
- Using pip to install **official PyTorch CUDA wheels** = good
- Constantly mixing conda and pip for overlapping packages in the same environment = risky

---

## 15. Installing PyTorch with CUDA

### Command used
```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
```

### What this installed
- `torch`
- `torchvision`
- `torchaudio`
- CUDA-enabled build targeting CUDA 12.8 (`cu128`)

### Why `cu128`
Your driver was modern enough, and CUDA 12.8 was a good clean choice.

### Important note
`nvidia-smi` showed CUDA 13.0, but PyTorch used `cu128`.

This is **normal**.

### Why this is normal
A newer NVIDIA driver can support apps built against a slightly older CUDA runtime.

---

## 16. Verifying the PyTorch installation

Start Python:

```bash
python
```

Then run:

```python
import torch
print(torch.__version__)
print(torch.cuda.is_available())
print(torch.cuda.device_count())
print(torch.cuda.get_device_name(0))
```

### Output meaning
#### 1. Version
Output:
```python
2.10.0+cu128
```
Meaning:
- PyTorch version 2.10.0
- CUDA-enabled build using CUDA 12.8 runtime support

#### 2. CUDA availability
Output:
```python
True
```
Meaning:
- PyTorch can use CUDA successfully

#### 3. Device count
Output:
```python
1
```
Meaning:
- One CUDA-visible GPU is available

#### 4. Device name
Output:
```python
NVIDIA GeForce RTX 3060 Laptop GPU
```
Meaning:
- PyTorch correctly recognized your exact GPU

---

## 17. Real CUDA computation test

Detection alone is not enough, so we ran a real GPU computation test.

### Code
```python
x = torch.randn(2000, 2000, device="cuda")
y = torch.randn(2000, 2000, device="cuda")
z = x @ y
print(z.shape)
```

### Output observed
```python
torch.Size([2000, 2000])
```

### Meaning
This proved that:
- PyTorch could allocate tensors on the GPU
- CUDA tensor operations actually executed
- Matrix multiplication worked correctly
- The environment was not only detecting the GPU, it was actually using it

This is one of the strongest practical confirmations.

---

## 18. Jupyter-related clarification

At one point there was confusion about whether Jupyter was already installed in `gait_fusion`.

We checked directly instead of assuming.

### Commands
```bash
which jupyter
pip show jupyterlab
pip show notebook
pip show ipykernel
```

### Result
They were already installed inside `gait_fusion`.

### Evidence
`which jupyter` pointed to:

```bash
/home/wadud/miniconda3/envs/gait_fusion/bin/jupyter
```

And `pip show` confirmed:
- `jupyterlab`
- `notebook`
- `ipykernel`

were installed inside the `gait_fusion` environment.

### Meaning
Jupyter was already available in the correct environment.

---

## 19. Checking notebook kernels

We checked visible Jupyter kernels with:

```bash
jupyter kernelspec list
```

### Result observed
Two kernels appeared:
- `python3`
- `gait_fusion`

### Meaning
This usually means the same environment became visible in two ways:
- a generic kernel name
- an explicit project-specific kernel name

This is not harmful.

### Best practice
When Jupyter asks for a kernel, choose the clearer one:
- `gait_fusion`
- or `Python (gait_fusion)` depending on display

---

## 20. Full environment smoke-test notebook

You asked for a demo notebook to verify all major parts of the environment.

The notebook below checks:
- environment info
- package imports
- PyTorch + CUDA
- `nvidia-smi`
- real GPU tensor compute
- cuDNN convolution
- DataLoader multiprocessing
- NumPy / pandas / OpenCV / PIL
- matplotlib plotting
- file write/read
- mixed precision

### Cell 1 — Basic environment info
```python
import sys
import os
import platform
import subprocess
from pathlib import Path

print("=== BASIC ENV INFO ===")
print("Python executable :", sys.executable)
print("Python version    :", sys.version)
print("Platform          :", platform.platform())
print("Current directory :", os.getcwd())
print("Conda env         :", os.environ.get("CONDA_DEFAULT_ENV"))
print("CUDA_VISIBLE_DEVICES :", os.environ.get("CUDA_VISIBLE_DEVICES"))
```

### Cell 2 — Package import check
```python
print("=== IMPORT CHECK ===")

packages = [
    "torch",
    "torchvision",
    "torchaudio",
    "numpy",
    "pandas",
    "matplotlib",
    "cv2",
    "PIL",
    "sklearn",
    "yaml",
    "tqdm",
    "notebook",
    "jupyterlab",
    "ipykernel",
]

results = {}

for pkg in packages:
    try:
        module = __import__(pkg)
        version = getattr(module, "__version__", "version_not_found")
        results[pkg] = f"OK | version = {version}"
    except Exception as e:
        results[pkg] = f"FAILED | {type(e).__name__}: {e}"

for k, v in results.items():
    print(f"{k:12s} -> {v}")
```

### Cell 3 — NVIDIA and PyTorch GPU check
```python
import torch

print("=== PYTORCH + CUDA CHECK ===")
print("torch version             :", torch.__version__)
print("cuda available            :", torch.cuda.is_available())
print("cuda device count         :", torch.cuda.device_count())

if torch.cuda.is_available():
    print("gpu name                  :", torch.cuda.get_device_name(0))
    print("torch cuda version        :", torch.version.cuda)
    print("cudnn available           :", torch.backends.cudnn.is_available())
    print("cudnn version             :", torch.backends.cudnn.version())
else:
    print("GPU not available from PyTorch")
```

### Cell 4 — `nvidia-smi` check from notebook
```python
print("=== NVIDIA-SMI CHECK ===")

try:
    out = subprocess.check_output(["nvidia-smi"], stderr=subprocess.STDOUT, text=True)
    print(out[:2000])
except Exception as e:
    print("FAILED to run nvidia-smi")
    print(type(e).__name__, e)
```

### Cell 5 — Real GPU tensor compute test
```python
import time
import torch

print("=== GPU COMPUTE TEST ===")

if torch.cuda.is_available():
    device = "cuda"
    torch.cuda.empty_cache()

    a = torch.randn(4000, 4000, device=device)
    b = torch.randn(4000, 4000, device=device)

    torch.cuda.synchronize()
    t1 = time.time()

    c = a @ b

    torch.cuda.synchronize()
    t2 = time.time()

    print("Result shape             :", c.shape)
    print("Time taken (sec)         :", round(t2 - t1, 4))
    print("Tensor device            :", c.device)
    print("Allocated memory (MB)    :", round(torch.cuda.memory_allocated() / 1024**2, 2))
    print("Reserved memory (MB)     :", round(torch.cuda.memory_reserved() / 1024**2, 2))
else:
    print("Skipped because CUDA is not available.")
```

### Cell 6 — cuDNN convolution sanity check
```python
import torch
import torch.nn as nn

print("=== cuDNN / CONV TEST ===")

if torch.cuda.is_available():
    x = torch.randn(8, 3, 224, 224, device="cuda")
    conv = nn.Conv2d(3, 16, kernel_size=3, stride=1, padding=1).cuda()

    y = conv(x)

    print("Input shape  :", x.shape)
    print("Output shape :", y.shape)
    print("Output device:", y.device)
else:
    print("Skipped because CUDA is not available.")
```

### Cell 7 — DataLoader multi-worker test
```python
import torch
from torch.utils.data import Dataset, DataLoader
import time

print("=== DATALOADER TEST ===")

class DummyDataset(Dataset):
    def __init__(self, n=5000):
        self.n = n

    def __len__(self):
        return self.n

    def __getitem__(self, idx):
        x = torch.randn(3, 64, 64)
        y = idx % 10
        return x, y

dataset = DummyDataset(5000)

for num_workers in [0, 2, 4]:
    try:
        loader = DataLoader(
            dataset,
            batch_size=64,
            shuffle=True,
            num_workers=num_workers,
            pin_memory=torch.cuda.is_available(),
        )

        t1 = time.time()
        batch_count = 0

        for xb, yb in loader:
            batch_count += 1
            if batch_count == 30:
                break

        t2 = time.time()
        print(f"num_workers={num_workers} -> OK | 30 batches in {round(t2 - t1, 4)} sec")
    except Exception as e:
        print(f"num_workers={num_workers} -> FAILED | {type(e).__name__}: {e}")
```

### Cell 8 — NumPy / pandas / OpenCV / PIL test
```python
import numpy as np
import pandas as pd
import cv2
from PIL import Image

print("=== NUMPY / PANDAS / CV2 / PIL TEST ===")

arr = np.random.randint(0, 256, size=(128, 128, 3), dtype=np.uint8)
df = pd.DataFrame({
    "a": np.random.randn(5),
    "b": np.random.randint(0, 10, size=5)
})

img_pil = Image.fromarray(arr)
img_cv2_gray = cv2.cvtColor(arr, cv2.COLOR_RGB2GRAY)

print("NumPy array shape   :", arr.shape, arr.dtype)
print("Pandas df shape     :", df.shape)
print(df.head())
print("PIL image size      :", img_pil.size)
print("OpenCV gray shape   :", img_cv2_gray.shape)
```

### Cell 9 — Matplotlib display test
```python
import matplotlib.pyplot as plt
import numpy as np

print("=== MATPLOTLIB TEST ===")

x = np.linspace(0, 2 * np.pi, 200)
y = np.sin(x)

plt.figure(figsize=(6, 4))
plt.plot(x, y)
plt.title("Matplotlib Check")
plt.xlabel("x")
plt.ylabel("sin(x)")
plt.grid(True)
plt.show()
```

### Cell 10 — File write/read permission test
```python
from pathlib import Path

print("=== FILE WRITE TEST ===")

test_dir = Path("./env_test_outputs")
test_dir.mkdir(exist_ok=True)

test_file = test_dir / "test.txt"
test_file.write_text("gait_fusion environment write test successful")

content = test_file.read_text()

print("File path :", test_file.resolve())
print("Content   :", content)
```

### Cell 11 — Simple mixed precision test
```python
import torch
import torch.nn as nn

print("=== MIXED PRECISION TEST ===")

if torch.cuda.is_available():
    model = nn.Sequential(
        nn.Linear(1024, 2048),
        nn.ReLU(),
        nn.Linear(2048, 10)
    ).cuda()

    x = torch.randn(128, 1024, device="cuda")

    with torch.autocast(device_type="cuda", dtype=torch.float16):
        y = model(x)

    print("Output shape :", y.shape)
    print("Output dtype :", y.dtype)
    print("Mixed precision works.")
else:
    print("Skipped because CUDA is not available.")
```

### Cell 12 — Final summary cell
```python
import torch
import sys
import os

print("=== FINAL SUMMARY ===")
print("Environment ready:", True)
print("Python executable:", sys.executable)
print("Conda env        :", os.environ.get("CONDA_DEFAULT_ENV"))
print("Torch version    :", torch.__version__)
print("CUDA available   :", torch.cuda.is_available())

if torch.cuda.is_available():
    print("GPU              :", torch.cuda.get_device_name(0))
    print("CUDA build       :", torch.version.cuda)
    print("cuDNN available  :", torch.backends.cudnn.is_available())
```

---

## 21. Result of the smoke-test notebook

You uploaded the executed notebook, and its results showed **no real issue**.

### What passed
- Correct Python executable from `gait_fusion`
- Key packages imported correctly
- PyTorch imported correctly
- CUDA available from PyTorch
- GPU detected correctly
- `nvidia-smi` worked
- cuDNN available
- Real GPU computation worked
- Convolution test worked
- DataLoader with `num_workers=0,2,4` worked
- NumPy / pandas / OpenCV / PIL worked
- Matplotlib plot worked
- File write/read worked
- Mixed precision worked

### Minor observation
DataLoader timing across `num_workers=0,2,4` was similar.

This was **not a failure**.

### Why
The dummy dataset was too lightweight. Extra workers help much more when real file loading and preprocessing are heavier.

---

## 22. Cross-check techniques used

This setup was verified step by step rather than trusted blindly.

### System-level GPU check
```bash
nvidia-smi
```

### Conda check
```bash
conda --version
```

### Environment check
```bash
conda activate gait_fusion
python --version
```

### Jupyter location check
```bash
which jupyter
```

### Jupyter packages check
```bash
pip show jupyterlab
pip show notebook
pip show ipykernel
```

### Notebook kernel visibility
```bash
jupyter kernelspec list
```

### PyTorch GPU check
```python
import torch
print(torch.__version__)
print(torch.cuda.is_available())
print(torch.cuda.device_count())
print(torch.cuda.get_device_name(0))
```

### Real CUDA compute check
```python
x = torch.randn(2000, 2000, device="cuda")
y = torch.randn(2000, 2000, device="cuda")
z = x @ y
print(z.shape)
```

### Full smoke test
Use the notebook above.

Each check verifies a different layer.

---

## 23. Compact command list of the full setup

### Detect recommended NVIDIA driver
```bash
ubuntu-drivers devices
```

### Install recommended driver
```bash
sudo apt install -y nvidia-driver-580-open
sudo reboot
```

### Verify GPU driver
```bash
nvidia-smi
```

### Install Miniconda
```bash
cd ~
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

Choose `yes` for initialization.
Then close terminal and open a new one.

### Verify conda
```bash
conda --version
```

### Accept conda channel terms
```bash
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

### Create project environment
```bash
conda create -n gait_fusion python=3.10 -y
```

### Activate environment
```bash
conda activate gait_fusion
```

### Verify Python version
```bash
python --version
```

### Install PyTorch with CUDA
```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
```

### Verify PyTorch + GPU
```bash
python
```

```python
import torch
print(torch.__version__)
print(torch.cuda.is_available())
print(torch.cuda.device_count())
print(torch.cuda.get_device_name(0))
```

### Real CUDA test
```python
x = torch.randn(2000, 2000, device="cuda")
y = torch.randn(2000, 2000, device="cuda")
z = x @ y
print(z.shape)
```

### Jupyter checks
```bash
which jupyter
pip show jupyterlab
pip show notebook
pip show ipykernel
jupyter kernelspec list
```

---

## 24. Important practical lessons from this setup

### Lesson 1
`nvidia-smi` success is necessary but not sufficient.
It proves Ubuntu sees the GPU, but not that PyTorch can use it.

### Lesson 2
Always verify from inside Python using:
- `torch.cuda.is_available()`
- real CUDA tensor computation

### Lesson 3
Use separate conda environments.
Do not do project work in `base`.

### Lesson 4
Miniconda is enough.
Full Anaconda is usually unnecessary for this workflow.

### Lesson 5
Real testing beats guessing.
A smoke-test notebook is better than assuming the environment is fine.

### Lesson 6
DataLoader worker speed depends on real workload.
A lightweight dummy dataset may not show gains from extra workers.

### Lesson 7
Driver CUDA version and PyTorch CUDA build version do not need to match exactly.
Driver showing CUDA 13.0 and PyTorch using `cu128` is normal.

---

## 25. Final ready state of the machine

### System level
- Ubuntu installed
- NVIDIA driver installed and working
- `nvidia-smi` working
- GPU visible to OS

### Python environment level
- Miniconda installed
- conda working
- project environment created: `gait_fusion`
- Python version: `3.10.20`

### Framework level
- PyTorch installed: `2.10.0+cu128`
- `torchvision` installed
- `torchaudio` installed
- CUDA available in PyTorch
- GPU visible in PyTorch
- Real CUDA tensor operations working
- cuDNN working
- Mixed precision working

### Workflow level
- Jupyter available inside `gait_fusion`
- Notebook kernel visible
- Major scientific packages working
- File write/read works
- Plotting works
- DataLoader multiprocessing works

So the environment is **ready for actual ML/DL project work**.

---

## 26. Future fresh-install pipeline

If you need to repeat this setup later without any chat:

1. Install Ubuntu
2. Run:
   ```bash
   ubuntu-drivers devices
   ```
3. Install the recommended NVIDIA driver
4. Reboot
5. Run:
   ```bash
   nvidia-smi
   ```
6. Install Miniconda
7. Reopen terminal
8. Run:
   ```bash
   conda --version
   ```
9. Accept conda channel terms
10. Create:
   ```bash
   conda create -n gait_fusion python=3.10 -y
   ```
11. Activate:
   ```bash
   conda activate gait_fusion
   ```
12. Install PyTorch:
   ```bash
   pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
   ```
13. Verify in Python:
   ```python
   import torch
   print(torch.cuda.is_available())
   print(torch.cuda.get_device_name(0))
   ```
14. Run the real matrix multiplication test
15. Run the full smoke-test notebook
16. Start working

---

## 27. Daily-use commands

### Start working in the environment
```bash
conda activate gait_fusion
jupyter lab
```

### Or run scripts directly
```bash
conda activate gait_fusion
python your_script.py
```

### Quick GPU health check anytime
```bash
nvidia-smi
```

### Quick PyTorch check anytime
```python
import torch
print(torch.cuda.is_available())
print(torch.cuda.get_device_name(0))
```

---

## 28. Final conclusion

At the end of this setup, the machine was confirmed to be fully ready for ML/DL work on Ubuntu.

The important point is not just that the packages were installed, but that every critical layer was tested:
- system-level driver access
- conda environment integrity
- PyTorch CUDA visibility
- real CUDA tensor computation
- notebook workflow
- DataLoader workers
- mixed precision
- basic scientific stack

That is why this setup can be trusted as a working base for future projects.
