# Setup Guide: cuVSLAM on GCP L4 (Isaac ROS release-4.5)

Reproducible, from-scratch setup for running NVIDIA Isaac ROS cuVSLAM on a GCP
Spot L4 instance, ending with a verified quickstart run and a reusable disk
snapshot. Completed 2026-07-11 as Phase 0 of the `cuvslam-nvblox-nav` project.
Updated 2026-07-14: added Docker Mode Configuration (section 6), which bakes
the visual_slam packages into the dev image; current snapshot is
`cuvslam-nvblox-nav-base-v2`.

**Stack:** Ubuntu 24.04 / NVIDIA driver 610 (open) / Docker Engine 29 /
nvidia-container-toolkit 1.19 / Isaac ROS release-4.5 (ROS 2 Jazzy) / cuVSLAM 15.0.0

**Scope note:** This guide covers integration and bringup of NVIDIA's stack.
cuVSLAM and nvblox themselves are NVIDIA's software, installed as prebuilt
packages.

**Cost model:** Spot g2-standard-8 runs ~$0.25–0.40/hr. The instance is
deleted after every session; only the snapshot persists (v2 is ~25.7GB,
roughly $0.70–1.30/month; see section 8 on trimming build cache).

**Naming:** The instance name is pinned to `cuvslam-nvblox-nav` for all
sessions so that recorded commands stay copy-pasteable.

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
3. **Isaac ROS Development Environment / Docker Mode Configuration (release-4.5):**
   https://nvidia-isaac-ros.github.io/v/release-4.5/concepts/dev_env/index.html
4. **NVIDIA CUDA Installation Guide for Linux (apt network repo / driver):**
   https://docs.nvidia.com/cuda/cuda-installation-guide-linux/
5. **Docker Engine install on Ubuntu:**
   https://docs.docker.com/engine/install/ubuntu/
6. **NVIDIA Container Toolkit install guide:**
   https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
7. **GCP: create Spot VM instances:**
   https://cloud.google.com/compute/docs/instances/create-use-spot
8. **GCP: create and manage disk snapshots:**
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

Record the winning zone; all later commands need it. The Phase 0 run landed
in `us-central1-a`; the 2026-07-14 session landed in `us-west1-a`. Then SSH
in (generates a key on first use):

```bash
gcloud compute ssh cuvslam-nvblox-nav --zone=<zone>
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

## 6. Bake project packages into the image (Docker Mode Configuration)

Docs: [Isaac ROS Development Environment, "Custom Docker Image Layers"](https://nvidia-isaac-ros.github.io/v/release-4.5/concepts/dev_env/index.html)

`isaac-ros activate` runs the dev container with removal on exit, so packages
apt-installed inside it are lost when the container exits and are never
captured by a disk snapshot. The CLI's documented fix is a custom Docker
image layer: a thin `FROM ${BASE_IMAGE}` Dockerfile that the CLI chains on
top of the NVIDIA-managed base image. This is not hand-rolling the Isaac ROS
image; the base stays NVIDIA's, the layer only adds project packages.

Three files, then one build. All paths are on the host.

The Dockerfile layer (the `ARG BASE_IMAGE` / `FROM ${BASE_IMAGE}` pattern is
required; the CLI sets it when chaining):

```bash
mkdir -p ${ISAAC_ROS_WS}/docker && cat > ${ISAAC_ROS_WS}/docker/Dockerfile.cuvslam_nav << 'EOF'
ARG BASE_IMAGE
FROM ${BASE_IMAGE}
ARG DEBIAN_FRONTEND=noninteractive
ENV ROS_DISTRO=jazzy
ENV ROS_ROOT=/opt/ros/${ROS_DISTRO}
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    apt-get update \
    && apt-get install -y \
        ros-jazzy-isaac-ros-visual-slam \
        ros-jazzy-isaac-ros-examples
EOF
```

The Dockerfile search path. Trap: `CONFIG_DOCKER_SEARCH_DIRS` is
first-match-wins, and if this file exists but omits
`/etc/isaac-ros-cli/docker`, the CLI cannot find the base
`Dockerfile.isaac_ros` and the build breaks. Both directories must be listed.
The single-quoted heredoc keeps `${ISAAC_ROS_WS}` literal in the file; the
CLI expands it at load time.

```bash
mkdir -p ${ISAAC_ROS_WS}/scripts && cat > ${ISAAC_ROS_WS}/scripts/.isaac_ros_common-config << 'EOF'
CONFIG_DOCKER_SEARCH_DIRS=(${ISAAC_ROS_WS}/docker /etc/isaac-ros-cli/docker)
EOF
```

Register the image key at workspace scope. Put nothing else in this file; the
CLI merges it with the system config, and duplicating unrelated config here
can break activation.

```bash
mkdir -p ${ISAAC_ROS_WS}/.isaac-ros-cli && cat > ${ISAAC_ROS_WS}/.isaac-ros-cli/config.yaml << 'EOF'
docker:
  image:
    additional_image_keys:
      - cuvslam_nav
EOF
```

Build the composed image (several minutes; the apt install dominates):

```bash
isaac-ros activate --build-local
```

Watch the early output for the resolved image key list; it should show
`['isaac_ros', 'cuvslam_nav']`. On this and every subsequent activate, a line
like `Error response from daemon: failed to resolve reference
"nvcr.io/nvidia/isaac/ros:isaac_ros-cuvslam_nav_...": not found` is expected
and harmless: the CLI checks the NGC registry for the composed tag, misses
because the image is local-only, and uses the local image.

Verify persistence, which is the entire point. Exit the container, re-enter
with no build flag, and confirm the packages are present in a fresh
container:

```bash
exit
isaac-ros activate
ros2 pkg list | grep -E "isaac_ros_visual_slam|isaac_ros_examples"
```

Expected: `isaac_ros_examples`, `isaac_ros_visual_slam`,
`isaac_ros_visual_slam_interfaces`. Verified 2026-07-14; the subsequent
quickstart run (section 7) reproduced ~32.5 Hz odometry from the baked
packages with zero in-container installs.

## 7. cuVSLAM quickstart (verification)

Docs: [isaac_ros_visual_slam Quickstart](https://nvidia-isaac-ros.github.io/v/release-4.5/repositories_and_packages/isaac_ros_visual_slam/isaac_ros_visual_slam/index.html)

### 7.1 Download quickstart assets (host)

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

### 7.2 Enter the managed environment

Drops you into a container shell as `admin`, with the host workspace mounted
at `/workspaces/isaac_ros-dev`. The visual_slam and examples packages are
already present from the section 6 image layer; no in-container installs are
needed.

```bash
isaac-ros activate
```

### 7.3 Run — three terminals

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
terminal 1. Phase 0 measured ~32.3 Hz; the 2026-07-14 rerun from baked
packages measured ~32.5 Hz:

```
average rate: 32.512
    min: 0.005s max: 0.052s std dev: 0.01103s window: 50
```

## 8. Snapshot and teardown

Docs: [GCP disk snapshots](https://cloud.google.com/compute/docs/disks/create-snapshots)

Optional, before snapshotting: `isaac-ros activate --build-local` leaves
buildx build cache on disk, which the snapshot faithfully captures. Pruning
it shrinks the snapshot (v2 was taken without pruning and came out ~25.7GB,
much of it cache):

```bash
docker builder prune -f
```

Shut down cleanly (Ctrl+C the launch and monitor, `exit` out of container
shells), then from the local machine:

Stop the instance so the snapshot captures a quiesced disk:

```bash
gcloud compute instances stop cuvslam-nvblox-nav --zone=<zone>
```

Snapshot to US multi-region storage, so one snapshot can create disks in any
US zone (no per-region copying needed). Bump the version suffix each time the
environment changes:

```bash
gcloud compute snapshots create cuvslam-nvblox-nav-base-v2 \
  --source-disk=cuvslam-nvblox-nav \
  --source-disk-zone=<zone> \
  --storage-location=us
```

Verify it reports READY:

```bash
gcloud compute snapshots describe cuvslam-nvblox-nav-base-v2 --format="value(status,diskSizeGb,storageBytes,storageLocations)"
```

Prove the snapshot by restoring it before trusting it and before deleting any
predecessor snapshot. A GPU is not needed to verify disk contents, so a small
CPU instance costs about a cent and also rehearses the restore path every
future session uses. The check covers the composed image and the section 6
config files, which are what distinguish v2 from a bare Phase 0 disk:

```bash
gcloud compute instances create snapshot-test \
  --zone=<zone> \
  --machine-type=e2-medium \
  --source-snapshot=cuvslam-nvblox-nav-base-v2

gcloud compute ssh snapshot-test --zone=<zone> --command="docker images | grep cuvslam_nav; cat ~/workspaces/isaac_ros-dev/.isaac-ros-cli/config.yaml; cat ~/workspaces/isaac_ros-dev/scripts/.isaac_ros_common-config; ls ~/workspaces/isaac_ros-dev/isaac_ros_assets"

gcloud compute instances delete snapshot-test --zone=<zone> --quiet
```

A content check cannot prove GPU behavior; the first `isaac-ros activate` on
the next GPU session is the full end-to-end proof. Delete the predecessor
snapshot only after one of the two proofs has passed (v1 was deleted
2026-07-14 after the content check, accepting that residual gap).

Delete the GPU instance (its boot disk goes with it; the snapshot is the
persistent copy):

```bash
gcloud compute instances delete cuvslam-nvblox-nav --zone=<zone> --quiet
```

Verify the shutdown:

```bash
gcloud compute instances list
```

Expected: `Listed 0 items.`

## 9. Known issues and lessons learned

**cuVSLAM rejects non-monotonic timestamps.** Playing a bag a second time
against a running node produces a wall of
`Failed to track: Timestamps are non-monotonic` warnings, because the node
keeps its clock state and the replay's timestamps go backward. Rule: one bag
play per node lifetime. Restart the launch (or call the
`/visual_slam/reset` service) before any replay. This applies equally to
custom bags in later phases.

**The dev container is ephemeral; fixed by section 6.** `isaac-ros activate`
runs the container with removal on exit, so packages apt-installed inside it
do not survive and are not in any snapshot. As of v2 the visual_slam and
examples packages are baked into the composed image via Docker Mode
Configuration, so no post-activate installs are needed. Any future
in-container apt install is still ephemeral: to persist a new package, add it
to `Dockerfile.cuvslam_nav`, rerun `isaac-ros activate --build-local`, and
take a new snapshot.

**NGC registry miss on activate is expected.** Every activate prints
`failed to resolve reference "nvcr.io/nvidia/isaac/ros:isaac_ros-cuvslam_nav_...": not found`
because the composed tag exists only locally. The CLI falls back to the local
image. Harmless.

**Spot capacity is volatile.** us-east4 zones a and c were stocked out and
us-east4-b does not offer G2 machines at all; us-central1-a and us-west1-a
have both worked on the first try in different sessions. A stopped Spot GPU
instance is not a reservation (restart can fail on capacity) and it still
counts against regional GPU quota while stopped. With `GPUS_ALL_REGIONS = 1`,
a parked stopped instance also blocks creating a GPU instance anywhere else.
Hence the create-from-snapshot / delete-after-session workflow.

**Sizes for planning:** Isaac ROS dev image ~2.3GB, composed local image
~19.7GB reported (~6.45GB unique), quickstart assets ~612MB. v1 snapshot was
~5.2GB; v2 is ~25.7GB, much of it buildx cache that `docker builder prune`
would have removed (section 8).

## 10. Resuming a session from the v2 snapshot

```bash
# Create instance from snapshot (loop zones as in section 1, replacing
# --image-family/--image-project with the snapshot source):
gcloud compute instances create cuvslam-nvblox-nav \
  --zone=<zone-with-capacity> \
  --machine-type=g2-standard-8 \
  --source-snapshot=cuvslam-nvblox-nav-base-v2 \
  --boot-disk-type=pd-balanced \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP

gcloud compute ssh cuvslam-nvblox-nav --zone=<zone>
nvidia-smi   # confirm the driver came up before touching Docker
isaac-ros activate
# No package reinstall needed; the image carries them. Sanity check:
ros2 pkg list | grep -E "isaac_ros_visual_slam|isaac_ros_examples"
```

End of session, always: push code to GitHub, pull artifacts off the
instance, re-snapshot if the environment changed, delete the instance, and
verify with `gcloud compute instances list` showing zero instances.