# Phase 0 Setup Guide: cuVSLAM on GCP L4 (Isaac ROS release-4.5)

Reproducible, from-scratch setup for running NVIDIA Isaac ROS cuVSLAM on a GCP
Spot L4 instance, ending with a verified quickstart run and a reusable disk
snapshot. Completed 2026-07-11 as Phase 0 of the `cuvslam-nvblox-nav` project.

**Stack:** Ubuntu 24.04 / NVIDIA driver 610 (open) / Docker Engine 29 /
nvidia-container-toolkit 1.19 / Isaac ROS release-4.5 (ROS 2 Jazzy) / cuVSLAM 15.0.0

**Scope note:** This guide covers integration and bringup of NVIDIA's stack.
cuVSLAM and nvblox themselves are NVIDIA's software, installed as prebuilt
packages.

**Cost model:** Spot g2-standard-8 runs ~$0.25–0.40/hr. The instance is
deleted after every session; only the snapshot persists (~$0.10–0.15/month).

---

## Primary documentation

Read these first. All commands below are taken from them, and NVIDIA URLs and
package paths rot, so re-verify against the live pages before reusing this
guide months later.

1. **Isaac ROS Getting Started (release-4.5):**
   https://nvidia-isaac-ros.github.io/v/release-4.5/getting_started/index.html
   (unversioned "latest": https://nvidia-isaac-ros.github.io/getting_started/index.html)
2. **isaac_ros_visual_slam Quickstart (release-4.5):**
   https://nvidia-isaac-ros.github.io/v/release-4.5/repositories_and_packages/isaac_ros_visual_slam/isaac_ros_visual_slam/index.html
3. **NVIDIA CUDA Installation Guide for Linux (apt network repo / driver):**
   https://docs.nvidia.com/cuda/cuda-installation-guide-linux/
4. **Docker Engine install on Ubuntu:**
   https://docs.docker.com/engine/install/ubuntu/
5. **NVIDIA Container Toolkit install guide:**
   https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
6. **GCP: create Spot VM instances:**
   https://cloud.google.com/compute/docs/instances/create-use-spot
7. **GCP: create and manage disk snapshots:**
   https://cloud.google.com/compute/docs/disks/create-snapshots

System requirements from doc 1 (as of Jul 2026): x86_64 with Ampere or newer
GPU, Ubuntu 24.04, NVIDIA driver 580+, CUDA 13.0+. The L4 (Ada) with driver
610 satisfies all of these.

Prerequisites: a GCP project with `GPUS_ALL_REGIONS >= 1` and per-region
`NVIDIA_L4_GPUS >= 1` quota, and the gcloud CLI configured locally.

---

## 1. Create the Spot L4 instance

Docs: [GCP Spot VMs](https://cloud.google.com/compute/docs/instances/create-use-spot)

Spot capacity is not queryable; the only way to find a zone with capacity is
to attempt creation. This loop tries every US zone offering `g2-standard-8`
(the L4 is built into G2 machine types, no `--accelerator` flag needed) and
stops at the first success. Zones without regional L4 quota fail fast with a
quota error, which is expected noise.

```bash
for zone in $(gcloud compute machine-types list \
    --filter="name=g2-standard-8 AND zone~'^us-'" \
    --format="value(zone)" | sort); do
  echo "=== Trying $zone ==="
  if gcloud compute instances create cuvslam-nvblox-nav \
    --zone="$zone" \
    --machine-type=g2-standard-8 \
    --image-family=ubuntu-2404-lts-amd64 \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=150GB \
    --boot-disk-type=pd-balanced \
    --provisioning-model=SPOT \
    --instance-termination-action=STOP; then
    echo "=== SUCCESS in $zone ==="
    break
  fi
done
```

Record the winning zone; all later commands need it. This run landed in
`us-central1-a`. Then SSH in (generates a key on first use):

```bash
gcloud compute ssh cuvslam-nvblox-nav --zone=us-central1-a
```

All commands in sections 2–6 run **on the instance**.

## 2. NVIDIA driver (host)

Docs: [Isaac ROS Getting Started](https://nvidia-isaac-ros.github.io/v/release-4.5/getting_started/index.html)
directs to install the latest driver via official instructions; the
[CUDA Installation Guide](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)
documents the apt network-repo method used here. For Ada-generation GPUs
(L4), NVIDIA recommends the open kernel module flavor (`nvidia-open`).

Kernel headers (usually preinstalled on GCP images):

```bash
sudo apt-get update && sudo apt-get install -y linux-headers-$(uname -r)
```

CUDA apt repo keyring (same repo the Isaac ROS getting started page uses for
Ubuntu 24.04 x86_64):

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb && sudo dpkg -i cuda-keyring_1.1-1_all.deb && rm cuda-keyring_1.1-1_all.deb
```

Driver:

```bash
sudo apt-get update && sudo apt-get install -y nvidia-open
```

Verify (a reboot is only needed if this errors about the driver not
communicating):

```bash
nvidia-smi
```

Expected: a table showing the NVIDIA L4 with ~23GB VRAM.

## 3. Docker Engine (host)

Docs: [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

Add Docker's repo:

```bash
sudo apt-get install -y ca-certificates curl && \
sudo install -m 0755 -d /etc/apt/keyrings && \
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc && \
sudo chmod a+r /etc/apt/keyrings/docker.asc && \
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && \
sudo apt-get update
```

Install:

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Non-root access (also required by isaac-ros-cli; `newgrp` only fixes the
current shell, other sessions need a re-login):

```bash
sudo usermod -aG docker $USER && newgrp docker
```

Verify:

```bash
docker run --rm hello-world
```

## 4. NVIDIA Container Toolkit (host)

Docs: [Container Toolkit install guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

Repo and install:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg && \
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list > /dev/null && \
sudo apt-get update && \
sudo apt-get install -y nvidia-container-toolkit
```

Wire the NVIDIA runtime into Docker and restart it:

```bash
sudo nvidia-ctk runtime configure --runtime=docker && sudo systemctl restart docker
```

Verify GPU passthrough (host prep is done when this prints the L4 table):

```bash
docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubuntu24.04 nvidia-smi
```

## 5. Isaac ROS release-4.5 apt repo, workspace, and CLI (host)

Docs: [Isaac ROS Getting Started](https://nvidia-isaac-ros.github.io/v/release-4.5/getting_started/index.html),
sections "Create a Workspace", "Configure Isaac ROS Apt Repository", and
"Initialize Isaac ROS CLI". The repo is pinned to `release-4.5` (not the
rolling `release-4` channel) so apt upgrades cannot silently move minor
versions.

Repo:

```bash
k="/usr/share/keyrings/nvidia-isaac-ros.gpg"
curl -fsSL https://isaac.download.nvidia.com/isaac-ros/repos.key | sudo gpg --dearmor | sudo tee -a $k > /dev/null
f="/etc/apt/sources.list.d/nvidia-isaac-ros.list"
sudo touch $f
s="deb [signed-by=$k] https://isaac.download.nvidia.com/isaac-ros/release-4.5 noble main"
grep -qxF "$s" $f || echo "$s" | sudo tee -a $f
sudo apt-get update
```

Workspace (the quickstarts rely on `ISAAC_ROS_WS`):

```bash
mkdir -p ~/workspaces/isaac_ros-dev/src && \
echo 'export ISAAC_ROS_WS="${ISAAC_ROS_WS:-${HOME}/workspaces/isaac_ros-dev/}"' >> ~/.bashrc && \
source ~/.bashrc && echo $ISAAC_ROS_WS
```

CLI (package is `isaac-ros-cli`, the binary is `isaac-ros`):

```bash
sudo apt-get install -y isaac-ros-cli
```

Documented Docker pre-flight check:

```bash
docker info | grep -E "Runtimes|Default Runtime" && docker run --rm --gpus all ubuntu:24.04 bash -lc 'echo "NVIDIA runtime OK"'
```

Initialize Docker mode (instant; it only records the mode, the image pull
happens on first `isaac-ros activate`):

```bash
sudo isaac-ros init docker
```

## 6. cuVSLAM quickstart (Phase 0 verification)

Docs: [isaac_ros_visual_slam Quickstart](https://nvidia-isaac-ros.github.io/v/release-4.5/repositories_and_packages/isaac_ros_visual_slam/isaac_ros_visual_slam/index.html)

### 6.1 Download quickstart assets (host)

Prerequisites:

```bash
sudo apt-get install -y curl jq tar
```

NGC asset download, verbatim from the quickstart page (~612MB, extracts into
`${ISAAC_ROS_WS}/isaac_ros_assets`):

```bash
NGC_ORG="nvidia"
NGC_TEAM="isaac"
PACKAGE_NAME="isaac_ros_visual_slam"
NGC_RESOURCE="isaac_ros_visual_slam_assets"
NGC_FILENAME="quickstart.tar.gz"
MAJOR_VERSION=4
MINOR_VERSION=5
VERSION_REQ_URL="https://api.ngc.nvidia.com/v2/resources/$NGC_ORG/$NGC_TEAM/$NGC_RESOURCE/versions"
AVAILABLE_VERSIONS=$(curl -s \
    -H "Accept: application/json" "$VERSION_REQ_URL")
LATEST_VERSION_ID=$(echo $AVAILABLE_VERSIONS | jq -r "
    .recipeVersions[]
    | .versionId as \$v
    | \$v | select(test(\"^\\\\d+\\\\.\\\\d+\\\\.\\\\d+$\"))
    | split(\".\") | {major: .[0]|tonumber, minor: .[1]|tonumber, patch: .[2]|tonumber}
    | select(.major == $MAJOR_VERSION and .minor <= $MINOR_VERSION)
    | \$v
    " | sort -V | tail -n 1
)
if [ -z "$LATEST_VERSION_ID" ]; then
    echo "No corresponding version found for Isaac ROS $MAJOR_VERSION.$MINOR_VERSION"
    echo "Found versions:"
    echo $AVAILABLE_VERSIONS | jq -r '.recipeVersions[].versionId'
else
    mkdir -p ${ISAAC_ROS_WS}/isaac_ros_assets && \
    FILE_REQ_URL="https://api.ngc.nvidia.com/v2/resources/$NGC_ORG/$NGC_TEAM/$NGC_RESOURCE/\
versions/$LATEST_VERSION_ID/files/$NGC_FILENAME" && \
    curl -LO --request GET "${FILE_REQ_URL}" && \
    tar -xf ${NGC_FILENAME} -C ${ISAAC_ROS_WS}/isaac_ros_assets && \
    rm ${NGC_FILENAME}
fi
```

### 6.2 Enter the managed environment

First run pulls the Isaac ROS dev image (~2.3GB) and drops you into a
container shell as `admin`, with the host workspace mounted at
`/workspaces/isaac_ros-dev`:

```bash
isaac-ros activate
```

### 6.3 Install packages (inside the container)

Binary-package route from the quickstart:

```bash
sudo apt-get update && sudo apt-get install -y ros-jazzy-isaac-ros-visual-slam ros-jazzy-isaac-ros-examples
```

### 6.4 Run — three terminals

**Terminal 1** (inside container): launch cuVSLAM. It loads the graph
(expect `cuVSLAM version: 15.0.0`, `Tracking mode: Multicamera`) and waits
for images.

```bash
ros2 launch isaac_ros_examples isaac_ros_examples.launch.py launch_fragments:=visual_slam \
interface_specs_file:=${ISAAC_ROS_WS}/isaac_ros_assets/isaac_ros_visual_slam/quickstart_interface_specs.json \
rectified_images:=false
```

**Terminal 2** (new SSH session, then `isaac-ros activate` to attach): start
the odometry monitor before the bag so it catches the stream. This replaces
the RViz step in the docs, which assumes a display; on a headless instance a
steady publish rate is the pass criterion.

```bash
ros2 topic hz /visual_slam/tracking/odometry --window 50
```

**Terminal 3** (new SSH session, `isaac-ros activate`): play the bag exactly
once, and do not touch the keyboard while it runs (the player binds
pause/rate keys).

```bash
ros2 bag play ${ISAAC_ROS_WS}/isaac_ros_assets/isaac_ros_visual_slam/quickstart_bag --remap  \
/front_stereo_camera/left/image_raw:=/left/image_rect \
/front_stereo_camera/left/camera_info:=/left/camera_info_rect \
/front_stereo_camera/right/image_raw:=/right/image_rect \
/front_stereo_camera/right/camera_info:=/right/camera_info_rect \
/back_stereo_camera/left/image_raw:=/rear_left/image_rect \
/back_stereo_camera/left/camera_info:=/rear_left/camera_info_rect \
/back_stereo_camera/right/image_raw:=/rear_right/image_rect \
/back_stereo_camera/right/camera_info:=/rear_right/camera_info_rect
```

**Pass criterion:** terminal 2 shows a steady rate with no tracking errors in
terminal 1. This run measured ~32 Hz:

```
average rate: 32.321
    min: 0.003s max: 0.050s std dev: 0.00985s window: 33
```

## 7. Snapshot and teardown

Docs: [GCP disk snapshots](https://cloud.google.com/compute/docs/disks/create-snapshots)

Shut down cleanly (Ctrl+C the launch and monitor, `exit` out of container
shells), then from the local machine:

Stop the instance so the snapshot captures a quiesced disk:

```bash
gcloud compute instances stop cuvslam-nvblox-nav --zone=us-central1-a
```

Snapshot to US multi-region storage, so one snapshot can create disks in any
US zone (no per-region copying needed):

```bash
gcloud compute snapshots create cuvslam-nvblox-nav-base \
  --source-disk=cuvslam-nvblox-nav \
  --source-disk-zone=us-central1-a \
  --storage-location=us
```

Verify it reports READY:

```bash
gcloud compute snapshots describe cuvslam-nvblox-nav-base --format="value(status,diskSizeGb,storageBytes,storageLocations)"
```

Optional but recommended once: prove the snapshot by restoring it. A GPU is
not needed to verify disk contents, so a small CPU instance costs about a
cent and also rehearses the restore path every future session uses.

```bash
gcloud compute instances create snapshot-test \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --source-snapshot=cuvslam-nvblox-nav-base

gcloud compute ssh snapshot-test --zone=us-central1-a --command="docker images && ls ~/workspaces/isaac_ros-dev/isaac_ros_assets"

gcloud compute instances delete snapshot-test --zone=us-central1-a --quiet
```

Delete the GPU instance (its boot disk goes with it; the snapshot is the
persistent copy):

```bash
gcloud compute instances delete cuvslam-nvblox-nav --zone=us-central1-a --quiet
```

## 8. Known issues and lessons learned

**cuVSLAM rejects non-monotonic timestamps.** Playing a bag a second time
against a running node produces a wall of
`Failed to track: Timestamps are non-monotonic` warnings, because the node
keeps its clock state and the replay's timestamps go backward. Rule: one bag
play per node lifetime. Restart the launch (or call the
`/visual_slam/reset` service) before any replay. This applies equally to
custom bags in later phases.

**The dev container is ephemeral.** `isaac-ros activate` runs the container
with removal on exit, so packages apt-installed *inside* it (section 6.3) do
not survive an instance stop and are **not** in the snapshot. The image, host
setup, and workspace assets are. Consequence: after restoring from this
snapshot, rerun the section 6.3 install once. The proper fix is the CLI's
Docker Mode Configuration
(https://nvidia-isaac-ros.github.io/concepts/dev_env/index.html), which bakes
extra packages into the image; do that, verify a launch, and take a v2
snapshot.

**Spot capacity is volatile.** us-east4 zones a and c were stocked out and
us-east4-b does not offer G2 machines at all; us-central1-a worked on the
first try. A stopped Spot GPU instance is not a reservation (restart can fail
on capacity) and it still counts against regional GPU quota while stopped.
With `GPUS_ALL_REGIONS = 1`, a parked stopped instance also blocks creating a
GPU instance anywhere else. Hence the create-from-snapshot / delete-after-
session workflow.

**Sizes for planning:** Isaac ROS dev image ~2.3GB, quickstart assets
~612MB, snapshot storage ~5.2GB after this full setup.

## 9. Resuming a session from the snapshot

```bash
# Create instance from snapshot (loop zones as in section 1, replacing
# --image-family/--image-project with the snapshot source):
gcloud compute instances create cuvslam-nvblox-nav \
  --zone=<zone-with-capacity> \
  --machine-type=g2-standard-8 \
  --source-snapshot=cuvslam-nvblox-nav-base \
  --boot-disk-type=pd-balanced \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP

gcloud compute ssh cuvslam-nvblox-nav --zone=<zone>
isaac-ros activate
# Until the v2 snapshot exists, reinstall in-container packages:
sudo apt-get update && sudo apt-get install -y ros-jazzy-isaac-ros-visual-slam ros-jazzy-isaac-ros-examples
```

End of session, always: push code to GitHub, pull artifacts off the
instance, re-snapshot if the environment changed, delete the instance.
