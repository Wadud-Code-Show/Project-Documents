# Wi‑Fi CSI–Based Gait Recognition Project Guide

## SECTION A — Executive recommendation

**Intel 5300 viability (2026):**  The Intel 5300 NIC is a legacy 802.11n device that outputs CSI for 30 subcarriers (per spatial stream) at 20 MHz【54†L819-L827】.  It is discontinued by Intel, so support relies on community tools【59†L8-L16】. In 2026 it remains *usable* (many Wi‑Fi sensing works still use it【37†L91-L100】), but it is inherently limited: only 30 CSI subcarriers at 20 MHz and 3×3 MIMO.  By contrast, newer CSI platforms deliver far higher resolution. For example, Broadcom chips with Nexmon firmware (e.g. on Raspberry Pi) can capture 256 subcarriers at 80 MHz【26†L297-L304】, and Atheros ath9k‑based tools capture ~114 subcarriers【54†L839-L844】.  Emerging Intel chipsets (AX200/210) have experimental CSI support in Linux, but rely on unreleased patches【55†L249-L257】【55†L283-L290】.  In summary, if you already have a 5300 card, you can use it (since tooling is mature), but for future-proofing consider Atheros or Broadcom alternatives which support 802.11ac/ax and richer CSI【54†L819-L827】【26†L297-L304】.

**Main machine vs. dedicated collector:** It is strongly recommended to use a **dedicated collection machine** (or a separate SSD/dual-boot) for CSI capture, not your everyday laptop.  The CSI tool requires specific kernel versions and disables NetworkManager; a dedicated Ubuntu install avoids interfering with your main OS and keeps the setup stable.  Do **not** use a VM (the wireless NIC won’t work properly【57†L9-L16】).  If your main machine is powerful, you could dual-boot, but plan for a separate Ubuntu boot or separate PC/NUC for the CSI card.  This isolates the CSI environment and prevents updates or other software from breaking the drivers. 

**Ubuntu version:**  Use an older Ubuntu LTS release with kernel ~4.15.  Many users successfully use **Ubuntu 18.04 LTS** (kernel 4.15) for Intel 5300 CSI【57†L2-L6】【24†L1-L9】.  The official CSI tool instructions support kernels up to 4.2【59†L8-L16】, and community forks (e.g. spanev’s) extend to 4.15【24†L1-L9】.  Newer Ubuntu (e.g. 22.04) with kernel 5.x or 6.x will **not work out-of-the-box** – the CSI driver patches are for older kernels and must be rebuilt if used.  Indeed, even the Broadcom Nexmon CSI tool is only confirmed up to Linux 5.10 and may break on 5.15+【26†L297-L304】.  Therefore, the safest choice is Ubuntu 18.04.0 (the stock GA kernel 4.15) or Ubuntu 20.04 with a manually downgraded kernel to 4.15【57†L2-L6】【24†L1-L9】.  Avoid the latest Ubuntu kernels unless you are prepared to backport or patch the driver.

**Latest Ubuntu?** The latest Ubuntu (22.04/24.04 with kernel ≥5.15) will *not* work without extra effort. The Intel 5300 CSI tool’s driver and supplementary code only support kernels up to ~4.15【59†L8-L16】【26†L297-L304】. Using 22.04+ would require backporting older iwlwifi modules or using unreleased patches (per a GitHub issue)【55†L249-L257】【55†L283-L290】. In practice, researchers avoid this complexity by sticking to older Ubuntu. 

## SECTION B — Full environment setup for Intel 5300 CSI collection

### OS and kernel

1. **Install Ubuntu 18.04 LTS (kernel 4.15).**  Download and install Ubuntu 18.04 (the original 18.04.0 or 18.04.1 is fine) on the collection machine. Ensure the kernel is 4.15 (the default for 18.04.0)【57†L2-L6】. Avoid Ubuntu 20.04/22.04 due to their newer kernels.  
   - **Check:** After install, run `uname -r`. Expected output: e.g. `4.15.0-...`.  
   - **Common issue:** If the kernel is newer (e.g. 5.x), install `linux-image-4.15` and reboot into it, or redo with an older ISO.

2. **Disable Wi-Fi management.**  To ensure full control, disable NetworkManager for the wireless interface. Install `iw` and edit `/etc/network/interfaces`:  
   ```bash
   sudo apt-get update
   sudo apt-get install -y iw
   echo "iface wlan0 inet manual" | sudo tee -a /etc/network/interfaces
   sudo systemctl restart NetworkManager
   ```  
   *Why:* The CSI tool works by command-line control of the interface, and NetworkManager can interfere【59†L17-L25】.  
   - **Verify:** Run `ifconfig` or `iwconfig`; interface wlan0 should show up in “no associated network” (down) mode.

3. **Install build tools and updated compiler.** The CSI driver patches require kernel headers and a compiler with retpoline support. Do:  
   ```bash
   sudo apt-get install -y build-essential linux-headers-$(uname -r) git-core
   sudo add-apt-repository ppa:ubuntu-toolchain-r/test
   sudo apt-get update
   sudo apt-get install -y gcc-8 g++-8
   ```  
   *Why:* GCC 8 is recommended for building the patched driver on 4.x kernels【57†L13-L21】. Ensure `/usr/bin/gcc` points to gcc-8:  
   ```bash
   sudo rm /usr/bin/gcc /usr/bin/g++
   sudo ln -s /usr/bin/gcc-8 /usr/bin/gcc
   sudo ln -s /usr/bin/g++-8 /usr/bin/g++
   ```  
   - **Verify:** `gcc --version` should show “8.x”.  
   - **Common failure:** If you skip this, `make` may fail (retpoline errors). If error about `No such file or directory` for signing, you can ignore it (see next step).

### Driver and firmware

4. **Clone and build the modified driver.** Use the official spanev/Linux-80211n-csitool repository (a fork of Halperin’s tool) to get the kernel patches:  
   ```bash
   git clone https://github.com/spanev/linux-80211n-csitool.git
   cd linux-80211n-csitool
   CSITOOL_KERNEL_TAG=csitool-$(uname -r | cut -d . -f 1-2)
   git checkout ${CSITOOL_KERNEL_TAG}
   ```  
   *Why:* This repo contains the modified iwlwifi driver for CSI. It requires checking out the tag matching your kernel (here 4.15).  
   - **Build:**  
     ```bash
     make -C /lib/modules/$(uname -r)/build M=$(pwd)/drivers/net/wireless/iwlwifi modules
     sudo make -C /lib/modules/$(uname -r)/build M=$(pwd)/drivers/net/wireless/iwlwifi INSTALL_MOD_DIR=updates modules_install
     sudo depmod
     ```  
     The make commands compile the iwlwifi driver with CSI support and install it under `/lib/modules/.../updates`.  
   - **Note:** If you see “Can’t read private key” or similar SSL errors, ignore them【57†L35-L42】 (they just mean the module isn’t signed).  
   - **Cross-check:** After this, `lsmod | grep iwlwifi` should still show the original module (we haven’t reloaded yet).

5. **Install the modified firmware.** Download Halperin’s supplementary firmware and install it:  
   ```bash
   cd ..
   git clone https://github.com/dhalperi/linux-80211n-csitool-supplementary.git
   # Backup existing firmware
   for file in /lib/firmware/iwlwifi-5000-*.ucode; do sudo mv $file $file.orig; done
   # Copy modified firmware
   sudo cp linux-80211n-csitool-supplementary/firmware/iwlwifi-5000-2.ucode.sigcomm2010 /lib/firmware/
   sudo ln -s iwlwifi-5000-2.ucode.sigcomm2010 /lib/firmware/iwlwifi-5000-2.ucode
   ```  
   *Why:* The modified `iwlwifi-5000-2.ucode.sigcomm2010` enables CSI reporting【59†L90-L98】. Old firmware is moved aside so it won’t load.  
   - **Verify:** Check that `/lib/firmware/iwlwifi-5000-2.ucode` is a symlink to the `.sigcomm2010` file.  

6. **Build the logging tool.** Compile `log_to_file`, the userspace CSI logger:  
   ```bash
   make -C linux-80211n-csitool-supplementary/netlink
   ```  
   This produces the executable `log_to_file`.

7. **Load the CSI driver.** Unload any existing iwlwifi, then reload with CSI logging enabled:  
   ```bash
   sudo modprobe -r iwlwifi mac80211
   sudo modprobe iwlwifi connector_log=0x1
   ```  
   *Why:* `connector_log=0x1` tells the driver to log CSI events【59†L119-L127】. On Ubuntu, unloading `iwldvm` as well is usually automatic; if on other distro, you may need `sudo modprobe -r iwldvm iwlwifi mac80211`.  
   - **Check:** Run `dmesg | grep iwlwifi`. You should see lines indicating the new firmware loaded (including “sigcomm2010”). If it fails, recheck firmware location.  
   - **Expected output:** `lsmod | grep iwlwifi` should now list the new module (signature “sigcomm2010” in `modinfo iwlwifi`).

### Network setup and testing

8. **Set up an AP or connect to one.** CSI logging requires Wi-Fi traffic at HT rates. You have two options:
   - **Use an existing 802.11n AP:**  Ensure the AP is on 2.4 GHz or 5 GHz at HT20/40 mode. The AP must be open (or with known password). Connect the NIC to this AP using `iwconfig` or `nmcli`.  
   - **Use hostapd:**  Alternatively, turn the Ubuntu machine into an 802.11n AP on a chosen channel using `hostapd` (not covered here). Then use another client (e.g. phone or second NIC) as a client.  

   *Why:* CSI logging works by capturing CSI from management/data frames with high throughput. A proper AP or link is needed【59†L123-L131】【57†L53-L61】.  
   - **Verify:** Use `iwconfig wlan0` to ensure it shows Mode:Managed and an associated AP (ESSID, frequency, etc). Also `ifconfig wlan0` should have an IP if DHCP ran (or set one manually).

9. **Start CSI logging.** In one terminal, run:  
   ```bash
   sudo ./linux-80211n-csitool-supplementary/netlink/log_to_file csi.dat
   ```  
   This tool will write CSI packets to `csi.dat` and echo short messages per packet【59†L129-L137】. In another terminal, generate traffic to the AP, for example by pinging it repeatedly:  
   ```bash
   ping <AP_IP> -i 0.5
   ```  
   (Replace `<AP_IP>` with the AP’s IP, e.g. `192.168.0.1`.)  
   - **Expected output:** The `log_to_file` window should start showing lines like “CSI from ...” and table-form CSI values. A sample screenshot can look like:

   【57†L59-L65】 *(In this example, `log_to_file` is printing CSI values as replies come. If you see rows of numbers with `Nss=3` etc, CSI is being captured.)*  

   - **Cross-check:** After a few seconds, `csi.dat` should appear in your directory (non-zero size). Also `dmesg` may report frames.  

10. **Debug common issues:** 
    - If *no CSI output* appears, check: (a) The card is properly connected (try `lshw -C network` to see the Intel 5300); (b) Are you associated to an AP? (`iwconfig` should show “Access Point: <MAC>”); (c) Are pings actually returning? (If not, network config is wrong); (d) Ensure the AP is 802.11n (try forcing HT20: `sudo iwconfig wlan0 channel X`); (e) If needed, disable power saving: `sudo iw dev wlan0 set power_save off`.  
    - If *errors during build*, ensure the kernel headers match the running kernel; reinstall `linux-headers-$(uname -r)`. If `make` fails on modules, verify you have GCC 8 set.  
    - If *modprobe hangs* or module still in use, ensure you removed iwlwifi and iwldvm.  
    - If *ping is too fast*, a 0.5 s interval is safe; lower intervals may flood. Use `sudo` for intervals <0.2s as needed.  

11. **Lock down the setup:** To avoid breaking the working setup later, do **not** upgrade the kernel or distribution. Consider marking packages on hold:  
    ```bash
    sudo apt-mark hold linux-image-generic linux-headers-generic
    ```  
    And avoid `apt upgrade` of the wireless driver or `linux-firmware` (which could overwrite firmware) unless you reinstall the patched versions. Keep a backup of your `/lib/firmware/iwlwifi-5000-2.ucode.orig` files. Also keep notes of the exact commands and versions used (for reproducibility).

At this point, you should have a functioning CSI collection environment: running `log_to_file` while pinging will produce `csi.dat`. Confirm by opening it in MATLAB or Python CSI readers (look for 3D arrays of complex CSI). 

## SECTION C — Practical collection architecture

To build a **gait dataset** with Intel 5300 CSI, consider the following architecture:

- **Transmitter setup:** The Intel 5300 card should act as **receiver only (Rx)** for CSI. You need a separate **transmitter** to send Wi-Fi packets. This can be any standard Wi-Fi AP or station capable of HT20/40 (802.11n). In practice, many projects use two Intel 5300 cards (one as AP, one as sniffing client)【43†L252-L259】. For example, one approach is: one 5300 NIC (in device A) is configured as an AP (via `hostapd`), and another 5300 NIC (in device B) is the CSI collector. Alternatively, use a commodity router or a second laptop as AP, with the 5300 as the client/listener. 

- **AP vs. Monitor mode:**  The easiest is to run the 5300 in **managed (client) mode** connected to an AP, and just ping. This is the original Halperin method. Another approach (e.g. robotics/WSR) uses monitor mode with explicit packet injection【35†L327-L335】, but this is more complex. For a gait dataset, using a simple AP–client setup is sufficient. Ensure the AP is on a fixed channel and no roaming. 

- **Network topology:** Use a **single Wi-Fi link** (one Tx, one Rx) for simplicity. (Multiple links could add spatial diversity but increase complexity.) The AP and receiver should be stationary in the collection room. You may use different antenna orientations or positions (e.g. 0°, 45°, 90°) as part of the experiment parameters (to add diversity) – see **metadata** below.

- **Room setup:**  Choose a medium-sized room with a clear **LOS (line-of-sight)** walking path. The transmitter and receiver (each with three antennas) should face each other. Mark a walking path between them. Also prepare a **NLOS scenario** by adding obstacles (e.g. placing a wall or turning one antenna away) if needed for variability. Document the distance and layout. 

- **Controlled experiment design:**  Have participants walk repeatedly along a marked path (e.g. back and forth between Tx and Rx). Ensure each subject’s gait session has consistent start/end points. Use a visible cue (line on floor) so the walker covers the full path. For each walking trial, record the time stamps (or trigger) to align CSI data to gait cycles. Alternatively, you can start/stop CSI logging explicitly around each walk. 

- **Walking conditions:** Vary walking speed if desired, but keep it roughly comfortable. Consider multiple clothing/carrying conditions (e.g. wearing a coat or backpack) to test robustness【54†L839-L844】. Also try multiple trajectories (straight, curved) to mimic realistic walking. 

- **Channel and bandwidth:** Use a fixed **20 MHz (HT20)** or 40 MHz channel. The Intel 5300 supports 40 MHz, but most users stick to 20 MHz for stability (the CSI tool supports 40MHz too). Choose an AP channel free of interference (use `iwlist scan`). Document channel, bandwidth, and center frequency for each session. 

- **Repeatability:** Collect multiple sessions per subject to average out variability. For instance, 5–10 walks per person per condition. Randomize the order of subjects and conditions. If possible, collect on multiple days to test temporal generalization. Use the same setup (antenna position, channel) each time to ensure consistency. 

- **Session metadata:** For each recording, save metadata including: subject ID, timestamp, room description, transmitter/receiver positions, channel, bandwidth, antenna orientation, carry/accessories, walking speed, and any notable events. For example, record “Subject A, 2026-03-20 10:00, LOS, 10m apart, CSI on channel 6, 20MHz, Antennas vertical, wearing coat”. Store this with the CSI data. Also log any packet rate (e.g. ping interval) and environment changes. 

In summary, a recommended setup is: one Intel 5300 NIC in the collector (Rx) machine running the CSI driver, and a second Intel 5300 or any 802.11n AP as the packet source. Use managed mode (AP–client) on a fixed channel. Control walking trials carefully and record full metadata. This controlled design (single link, fixed positions, varied conditions) follows prior gait-sensing works【43†L252-L259】【54†L839-L844】.

## SECTION D — GUI requirement

A GUI will streamline live data collection and parameter control. The recommended architecture is a **lightweight wrapper** around the CSI CLI tools (especially `log_to_file`). For example, use **PyQt** or **Tkinter** to build a simple GUI that calls system commands and edits config. PyQt is more powerful; Zenity (bash GUI) could also work but is less flexible. We suggest PyQt5 (Python) for ease.

Key GUI features:
- **Parameter panel:** Expose essential parameters to the user:
  - Transmitter IP (for ping) or start button for AP.
  - Ping interval (e.g. 0.2–1.0 s).
  - Output directory/session name.
  - Subject ID and condition (select from list).
  - Channel and bandwidth (for reconfiguring AP, if needed).
- **Start/stop collection:** Buttons to launch CSI capture:
  - On “Start”, the GUI runs `sudo modprobe iwlwifi connector_log=0x1`, starts `log_to_file` in background (to a new file named by session), and begins ping (via `subprocess`). 
  - On “Stop”, it stops the ping and log process.
  - Make sure to handle permissions (running GUI as root or use `sudo` within calls).
- **Live indicators:** Show simple status (e.g. “Logging...”) and maybe a counter of packets recorded. Could also show a small plot of recent CSI amplitude (optional).
- **Metadata entry:** Fields to fill session metadata (subject, scenario, notes). On start, write these to a JSON or YAML file alongside the CSI file.

Folder structure recommendation:
```
data/
  subject01/
    2026-03-20/
      sess1/
        csi.dat
        metadata.json
      sess2/
        ...
  subject02/
    ...
```
Each session directory contains `csi.dat` and a metadata file (JSON/YAML) with all the details. The GUI should create this structure.

Implementation plan:
1. Use PyQt to create a window with drop-downs/text fields and Start/Stop buttons.
2. On **Start**: 
   - Prompt for a new session name/ID.
   - Write metadata (entered by user and current timestamp) to file.
   - Execute `modprobe` and then `log_to_file` (e.g. via `subprocess.Popen(["sudo","log_to_file","data/.../csi.dat"])`).
   - Start ping to AP (using `subprocess.Popen`).
3. On **Stop**:
   - Terminate the ping and logging processes (kill them).
   - Possibly flush and close files.
4. Wrap all calls with error checking. 
5. Save session info in a structured way (metadata JSON example: `{"subject": "01","date": "2026-03-20", "channel": 6, ...}`).

This GUI is essentially a thin wrapper that executes the existing CSI collection commands; it should not try to reimplement their functionality, just manage them. Zenity (simple dialogs from Bash) could be an alternative, but PyQt allows richer interface. 

## SECTION E — Best data format for future research

For archival and downstream use, we recommend storing **raw CSI in binary form (the `.dat` files)** and metadata in structured text, while also providing processed data in a hierarchical format. Compare:

- **Raw .dat (binary):** This is what `log_to_file` produces. It contains complex CSI values and some headers. It’s already compact but not self-describing. Keep the raw `.dat` files *unchanged* as a bitwise copy, so others can reprocess exactly. 
- **CSV or text:** Dumping CSI to CSV would be huge (one line per packet per subcarrier) and slower to parse; not ideal for raw archival. 
- **HDF5:** For archiving, HDF5 is excellent: it can store multidimensional CSI arrays, metadata, and labels in one file【59†L129-L137】. We propose an HDF5 schema:

  ```
  /session_date           (string)
  /subject_id             (string)
  /channel               (int)
  /bandwidth             (string)
  /location              (string)
  /conditions           (string or group)
  /csi/                   (group)
     /timestamps         (Nx1 array of packet times)
     /amp                (NxSxM array of amplitudes, N=packets, S=antennas, M=subcarriers)
     /phase              (NxSxM array of phases)
  /labels/                (group)
     /activity           (Nx1 categorical)
     /environment        (string)
  /notes                 (string)
  ```
  This stores raw CSI amplitudes and phases (split into two arrays) along with timestamps and labels. The raw complex CSI can also be saved directly as complex numbers in HDF5 (many libraries support complex dtypes). Include all session metadata as attributes. 

- **Parquet:** Parquet is columnar and better for tabular data, not ideal for multi-dimensional time-series. HDF5 (or zarr) is better for signal data.

**Recommendation:** 
- Archive raw data as `.dat` files (bit-preserved) and raw I/Q if available.  
- Convert `.dat` into an HDF5 dataset per session for analysis. This HDF5 file can contain the above schema. It is self-describing, compressible, and efficient for multi-dimensional data. Use a chunked layout so one can read by packet or by antenna. 
- Save derived features (e.g. spectrograms, embeddings) in separate HDF5 or numpy files.  
- Always keep metadata and labels (subject ID, trial number, condition flags) either in the HDF5 or in a JSON/YAML file alongside.

This approach preserves everything: raw CSI, metadata, and any features. It also allows easy loading into Python or MATLAB. By using HDF5, future researchers can navigate by keys (e.g. `f['csi/amp']`) without re-parsing CSVs. In summary, **raw .dat + structured HDF5 archive** is best. 

## SECTION F — Research-paper-grounded pipeline for a Q1-level study

Building a publishable study requires a strong, literature-backed processing pipeline:

- **Dataset design:** Aim for at least 20–30 subjects (per many CSI studies). Include multiple environments (e.g. two different rooms or room layouts) and multiple days. Partition data into *intra-subject* and *cross-subject* splits. Include both **known-set** (subjects in training) and **open-set** (novel subjects in test) if tackling open-set identification【49†L27-L31】. Label each recording with activity (“walking”), subject ID, and environment label. 

- **Preprocessing:** Apply standard CSI pre-processing: 
  - **Noise reduction:** e.g. Hampel filter or wavelet denoising on amplitude【43†L253-L261】.
  - **Phase calibration:** Remove random phase offsets (e.g. use linear fitting or known reference packet)【43†L289-L296】.
  - **Synchronization:** Align CSI frames to walking cycles (if needed) or downsample to fixed rate. Possibly segment out only the portions where walking occurs (gait segmentation). Existing works often use sliding windows (e.g. 1-2s windows) or detect peaks in variance to cut gait cycles. Provide ground-truth segmentation if possible (using a floor sensor or video, but if not, use heuristic).
  - **Normalization:** Normalize CSI per session to zero mean and unit variance (or per-subject normalization)【49†L22-L30】 to remove environment-specific gains.
  - **Feature extraction:** Options include:
    - *Time-series features:* raw amplitude/phase sequences, or spectrograms (STFT) as images【42†L69-L77】.
    - *Image features:* e.g. convert a windowed CSI segment into an image via Gramian Angular Field (GADF)【42†L69-L77】 or spectrogram heatmaps.
    - *Subcarrier differences:* compute differences between subcarrier amplitudes to capture spatial variation (as in [42] subcarrier diversity heatmaps).
    - *Learned features:* let a neural network (CNN or transformer) learn end-to-end from raw or minor-processed CSI【78†L268-L274】.

- **Baseline models:** Start with classic, paper-provided baselines: 
  - *Machine learning:* PCA+SVM or Random Forest on hand-crafted features (STD, FFT peaks, etc).
  - *CNN/RNN:* e.g. ResNet variants on CSI spectrograms (as in [54] experiments using ResNet-18)【54†L868-L876】.
  - *Multimodal:* Simple CNN+LSTM (as in [43], where CNN extracts spatial and BiGRU temporal features, with attention)【43†L249-L258】.
  - *Transformer:* A vision-transformer or custom CSI-transformer (as suggested by recent works【78†L268-L274】).
  - Code from the SenseFi benchmark can be used to reproduce MLP, GRU, CNN, BiLSTM, ResNets, Vision Transformer baselines【54†L860-L869】.

- **Modern models:** After baselines, implement a SOTA approach, e.g. the multi-view adaptive fusion from [42] or a similar network (CNN+Transformer hybrid)【78†L268-L274】. Consider models that use amplitude and phase separately (like the transformer in [71])【71†L39-L47】. Also consider attention mechanisms or domain adaptation modules (e.g. DAN) if addressing cross-environment generalization.

- **Evaluation metrics:** Use identification accuracy (or top-1 accuracy). For open-set, also report false acceptance rate (FAR) and false rejection rate (FRR), or area under ROC. If framing as classification, use confusion matrix. Always report cross-subject accuracy (e.g. leave-one-subject-out or k-fold by subject) and cross-day/room accuracy. Report mean and standard deviation over folds.

- **Protocol:** Split data carefully:
  - *Cross-subject:* train on subset of people, test on the rest.
  - *Cross-environment:* train on one room, test on another (following GaitSense [49]).
  - *Cross-day:* train on one day, test on another (simulate temporal drift).
  - If open-set: have some subjects only appear in testing (seen vs unseen).
  Use stratified sampling so each split has balanced classes.

- **Open-set evaluation (if relevant):** If you allow “unauthorized” subjects at test time, incorporate an anomaly detection component (e.g. one-class SVM) and measure detection accuracy (the GaitSense approach【49†L27-L31】). 

- **Ablation studies:** Test the impact of:
  - Using amplitude only vs. amplitude+phase【43†L289-L296】.
  - Single antenna vs. MIMO (if you have multiple receiver antennas).
  - Different feature types (time vs. frequency vs. image) – e.g. raw CSI vs. STFT vs. GADF.
  - Model components: test with/without attention or domain adaptation.
  - Data augmentation (e.g. adding noise or using generative augmentation).

- **Reproducibility checklist:** 
  - Release code (using SenseFi-style repo if possible).
  - Publish collected dataset (if ethics allow) in HDF5 or .dat+metadata.
  - Fix random seeds, specify versions of libraries. 
  - Provide train/val/test splits and list of subjects in each split.
  - Detail preprocessing steps in supplementary material.

All pipeline choices above are motivated by recent literature: e.g. [42] for multi-view fusion and feature types, [43] for CNN/RNN architecture and use of phase, [49] for normalization/transfer learning, [54] for dataset splits and baseline networks, and [71] for transformer on amplitude+phase. Cite as appropriate: multi-view features【42†L69-L77】, amplitude+phase CNN-BiGRU【43†L253-L261】【43†L289-L296】, domain generalization techniques【49†L22-L30】, baseline comparisons【54†L860-L869】, and transformer advances【78†L268-L274】【71†L39-L47】.

## SECTION G — Best publishable research direction

Based on recent trends, here are **three promising paper ideas**, with novelty and feasibility assessed:

1. **Cross-Domain CSI Gait Identification with Meta‑Learning:**  
   - *Novelty:* Apply few-shot/meta-learning or domain adaptation to gait recognition across rooms and days. For example, use a Siamese/Prototypical network or MAML on CSI data to quickly adapt to a new environment. Very few works tackle cross-domain gait (GaitSense is one example【49†L27-L31】), so applying modern meta-learning is new.  
   - *Difficulty:* High. Need careful algorithm design and multiple collection settings.  
   - *Data requirements:* Collect CSI gait from >2 distinct environments (rooms or setups) and multiple participants. Possibly use part of dataset for meta-training, part for meta-testing.  
   - *Expected contribution:* A method that drastically reduces calibration (few samples) for new rooms. Demonstrate higher cross-room accuracy than standard training【49†L27-L31】.  
   - *Key baselines:* Standard CNN/Transformer without domain adaptation, and any existing cross-domain CSI methods (e.g. domain adversarial nets). Possibly GaitSense’s transfer learning approach.  
   - *Risks:* Meta-learning on CSI is untested; may overfit or need very large data.  
   - *Target journal:* IEEE TMC or TWC. They expect strong algorithmic novelty and thorough eval.  
   - *Too weak:* Just training model per room. *Strengthen:* Add theoretical analysis or extensive ablation on domain shifts.  

2. **Open-Set Gait Recognition with CSI and Outlier Detection:**  
   - *Novelty:* Many gait studies ignore unknown people. Build a system that both identifies enrolled users and rejects outsiders using CSI (e.g. one-class SVM, deep OpenMax). Perhaps extend GaitSense’s anomaly detection【49†L27-L31】 with deep learning (autoencoders for novelty).  
   - *Difficulty:* Medium. The main challenge is defining good metrics for open-set (e.g. detection AUC).  
   - *Data:* Collect data for a group of “known” subjects (training set) and separate “unknown” subjects (testing set only). Use multiple days to simulate variation.  
   - *Contribution:* A framework for CSI gait with open-set evaluation. Show that certain architectures (like contrastive learning) yield good unknown detection.  
   - *Baselines:* Standard closed-set models, naive thresholds on softmax, etc. Possibly the GaitSense anomaly method as a baseline.  
   - *Risks:* Possibly trivial if known users are easily separated; need careful hard unknown sampling.  
   - *Target journal:* IEEE TBIOM or IEEE Sensors. These journals like biometric system papers.  
   - *Too weak:* Just adding “unknown” labels. *Strengthen:* Include systematic evaluation on detection metrics and compare multiple approaches.  

3. **Generative Data Augmentation for Wi-Fi CSI Gait:**  
   - *Novelty:* Use generative models (e.g. GANs or the diffusion model like RF-Diffusion【46†L83-L91】) to synthesize additional realistic CSI gait data. This can improve recognition when data is scarce. Perhaps condition on user ID to generate new CSI sequences. Some works (like RF-Diffusion for gestures) exist, but gait is new.  
   - *Difficulty:* High. Requires designing a generative model for time-series or spectrograms.  
   - *Data:* Initial gait dataset (real) to train the generator (at least ~10 subjects, many walks each).  
   - *Contribution:* A method to augment CSI data leading to improved accuracy in low-data regimes.  
   - *Baselines:* Vanilla training without augmentation. Possibly an existing generic CSI augmentation (none exist explicitly for gait).  
   - *Risks:* Generated CSI might not be physically realistic; evaluation of quality is tricky.  
   - *Target journal:* IEEE TMC or Pattern Recognition. They value ML innovations.  
   - *Too weak:* Off-the-shelf GAN with no novelty. *Strengthen:* Incorporate domain knowledge (e.g. diffusion on radio signals, as in RF-Diffusion【46†L83-L91】) and demonstrate significant gains in low-data settings.  

**Best idea for your hardware/situation:** Given you have Intel 5300 equipment and likely moderate data, *Idea 2 (Open-Set Recognition)* is the most immediately feasible. It leverages the existing platform without requiring new RF hardware. You can use a reasonable number of subjects and simply designate some as “unseen”. It builds on published ideas【49†L27-L31】 but pushes them into deep learning and robust evaluation, suitable for a Q1 journal like IEEE TBIOM or Sensors. 

## SECTION H — Q1 journal targeting

Candidates and advice:

- **IEEE Transactions on Mobile Computing (TMC):**  
  *Fit:* High-quality venue for mobile wireless sensing. Expect a strong methodological contribution (not just incremental). If your work includes advanced techniques (e.g. meta-learning or generative modeling for Wi-Fi) and a thorough evaluation, it suits TMC.  
  *Too weak:* If it’s just data collection or a small neural net with slight improvements, TMC reviewers will reject. They look for broad impact and novelty.  
  *Strengthen:* Focus on theoretical insight or system-level innovation (e.g. a new model of CSI and motion, or cross-platform method).  

- **IEEE Transactions on Wireless Communications (TWC):**  
  *Fit:* Focuses on wireless technology. If your novelty is on the wireless/physical layer (e.g. channel modeling of gait, multi-path analysis) it fits. Merely applying ML to CSI may be borderline, unless you highlight novel insights about the channel.  
  *Too weak:* A purely ML-driven paper with standard CSI usage might not be accepted.  
  *Strengthen:* Provide an analysis of channel effects in gait, or design a novel transceiver/method.  

- **IEEE Transactions on Biometrics, Behavior and Identity Science (TBIOM):**  
  *Fit:* Designed for biometric methods. A CSI-based gait identification with a significant advance (e.g. open-set detection) fits well.  
  *Too weak:* A small experiment without rigorous biometric evaluation might not suffice.  
  *Strengthen:* Include metrics like EER (equal error rate), ROC curves for identification/detection, and compare with other biometrics (vision or sensors) if possible.  

- **IEEE Internet of Things Journal (IoT-J):**  
  *Fit:* For IoT devices and sensing. A paper showing a novel IoT application (gait recognition) could fit. They like interdisciplinary work.  
  *Too weak:* Only a proof-of-concept or purely theoretical gains.  
  *Strengthen:* Emphasize system aspects (low-cost implementation, real-time UI, IoT deployment).  

- **Elsevier *Computers in Biology and Medicine* or *IEEE Sensors Journal*:**  
  *Fit:* If focusing on health/biological aspect of gait, or on sensor system, these might accept CSI gait work.  
  *Too weak:* Journals often expect comprehensive evaluation with many subjects.  
  *Strengthen:* Expand dataset (include clinical conditions?), or focus on a specific application (e.g. fall-risk detection from gait).  

For all journals, avoid being “too niche” (only CSI-savvy audience). Emphasize the broader biometric or IoT context. Provide **public data/code** to strengthen. Identify what would make work **too weak**: incomplete comparisons, lack of real-world testing, or insufficient novelty (e.g. using a known network on a small dataset). **Strengthen** by adding a comparison to other sensing modalities (if available) or demonstrating scalability (more subjects, environments). 

## SECTION I — Final actionable roadmap

### Phase 1: Environment setup  
- **Objectives:** Install Ubuntu 18.04, compile and test the CSI driver.  
- **Deliverables:** A bootable Ubuntu 18.04 system (physical or dual-boot) with Intel 5300 CSI tools. A working command to produce a valid `csi.dat`.  
- **Pitfalls:** Kernel mismatch, compiler errors, firmware loading issues.  
- **Success criteria:** Verified by running the `log_to_file` test and seeing CSI output (e.g. sample values). Pass: see several CSI frames in `dmesg` or on console.

### Phase 2: First CSI capture  
- **Objectives:** Collect CSI data from one subject walking. Learn to align CSI with actual gait events.  
- **Deliverables:** A small CSI recording (e.g. 10–20 s of walking) with metadata. Scripts to convert `csi.dat` to a usable format (e.g. numpy).  
- **Pitfalls:** Mis-timed recording, missing labels, low-quality capture (noisy).  
- **Success criteria:** Plot of CSI amplitude vs time shows clear cyclic pattern corresponding to steps. Able to segment out at least one full gait cycle.

### Phase 3: Stable controlled collection  
- **Objectives:** Refine the setup: fix AP positions, ensure repeatability. Optimize parameters (ping rate, channel).  
- **Deliverables:** A protocol document. GUI prototype (optional here) to aid consistent captures. Consistent CSI features across repeated trials.  
- **Pitfalls:** Environmental interference (people moving), inconsistent walking.  
- **Success criteria:** Repeated sessions yield similar CSI patterns (low variance). Minimal packet loss, stable high data rate.

### Phase 4: GUI and data management  
- **Objectives:** Develop the collection GUI as planned, with metadata entry and session management.  
- **Deliverables:** Working GUI application. Established folder structure and template for metadata files.  
- **Pitfalls:** Process management (making sure logging processes are properly killed), user errors (missed fields).  
- **Success criteria:** Using GUI, collect a 5–10 min dataset seamlessly (no manual CLI). Metadata correctly saved; replaying logs shows data.

### Phase 5: Dataset collection  
- **Objectives:** Collect full-scale dataset: all subjects, all conditions, multiple sessions.  
- **Deliverables:** The final CSI dataset (raw `.dat` files and HDF5 archives) with full metadata. (Ensure compliance with ethics.)  
- **Pitfalls:** Scheduling subjects, hardware glitches, data backup failures.  
- **Success criteria:** Dataset contains e.g. ≥30 subjects × 5 sessions each, covering LOS/NLOS, different clothing, etc. Data quality is checked (no corrupt files, CSI captured). 

### Phase 6: Preprocessing and baselines  
- **Objectives:** Implement preprocessing pipeline (denoising, calibration, segmentation). Train baseline models and evaluate.  
- **Deliverables:** Code for preprocessing, baseline training (MLP, CNN, RNN). Tabulated results on cross-val.  
- **Pitfalls:** Overfitting baselines, slow training.  
- **Success criteria:** Baseline accuracy stable (e.g. >80% on known-subject ID). Ablation scripts in place to test features (amp vs. phase).

### Phase 7: Advanced model  
- **Objectives:** Develop the novel model (e.g. multi-view or transformer) and integrate it.  
- **Deliverables:** Code for the new model, and experiments showing its performance. Ablation of key components.  
- **Pitfalls:** Model not converging, marginal improvements.  
- **Success criteria:** The advanced model outperforms baselines by a convincing margin (e.g. +5-10%). Good performance also under cross-room or open-set conditions.

### Phase 8: Paper writing and submission  
- **Objectives:** Write the manuscript targeted to chosen Q1 journal. Include all experiments, related work, and analysis.  
- **Deliverables:** Full paper draft, supplementary material (code link, data description). Submission to target journal.  
- **Pitfalls:** Deadline pressure, incomplete experiments, unclear writing.  
- **Success criteria:** Paper is well-organized and clearly cites sources (e.g. [42][43][49][54][71]). Reviewers find novelty compelling and results solid. 

Each phase builds on the previous one. By Phase 5 you should have a robust dataset; by Phase 7 a state-of-the-art model; by Phase 8 a polished write-up. Careful documentation at each step (especially dataset and code) ensures reproducibility and strengthens the paper.  

**Sources:** All technical steps above are based on official Intel CSI tool documentation【59†L8-L16】【57†L13-L21】 and recent research【42†L69-L77】【43†L253-L261】【49†L22-L30】【54†L860-L869】【71†L39-L47】.