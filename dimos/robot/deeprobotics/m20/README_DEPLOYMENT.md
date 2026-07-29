## M20 Network Configuration


Before deploying the communication components, configure the network connection between the M20, the TP-Link router (or switch), and the Laptop.

### 1. Connect the TP-Link Device to the M20
> **Note:**
> The purpose of this step is to configure the M20 to create a local network, allowing the Laptop to remotely access the onboard modules (AOS, GOS, and NOS). If you already have your own networking solution that provides equivalent connectivity, you may skip this section.

Use the dedicated combined Ethernet-and-power cable to connect the TP-Link device to any of ports **2**, **3**, **4**, or **5** on the M20.

---

### 2. Configure the TP-Link Network

Log in to the TP-Link management interface and configure the network with the following settings:

| Setting | Value |
| --- | --- |
| IP Address | `10.21.31.2` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `10.21.31.2` |
| Network | `10.21.31.0/24` |

After saving the configuration, reconnect to the TP-Link Wi-Fi network and verify that the management page is accessible at:

```text
http://10.21.31.2
```

---

### 3. Configure the Laptop Network

Connect the Laptop to the TP-Link Wi-Fi network and assign it a valid host address within the `10.21.31.0/24` subnet. For example:

| Setting | Example |
| --- | --- |
| IP Address | `10.21.31.7` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `10.21.31.2` |

> **Note:**
>
> - `10.21.31.0` is the network address and cannot be assigned to a host.
> - `10.21.31.255` is the broadcast address and also cannot be assigned to a host.

Once configured, the Laptop should be able to communicate with the M20 onboard hosts:

| Host | IP Address |
| --- | --- |
| AOS | `10.21.31.103` |
| GOS | `10.21.31.104` |
| NOS | `10.21.31.106` |

Verify network connectivity by running:

```bash
ping 10.21.31.103
ping 10.21.31.104
ping 10.21.31.106
```

If all three hosts respond successfully, the network has been configured correctly and you can proceed with the deployment.
```


# M20 DimOS Communication and Navigation Deployment

This guide deploys the M20 communication components and runs
`m20-simple-nav` from a Laptop. The deployment spans three machines:

| Machine | Default address | Responsibility |
| --- | --- | --- |
| M20 NOS | `10.21.31.106` | Read drdds topics and publish them through Zenoh |
| M20 AOS | `10.21.31.103` | Route Zenoh traffic between the robot and Laptop |
| Laptop | Deployment-specific | Run DimOS navigation, control, and visualization |

The data path is:

`NOS drdds` -> `drdds-Zenoh Bridge` -> `AOS zenohd` -> `Laptop DimOS`.

Do not store NOS or AOS passwords in this repository. SSH prompts for them when
required.

## Required Deployment Files

Prepare these directories on the Laptop before starting:

`~/m20-drdds-packages/` must contain:

- `drddslib_1.2.1_arm64.deb`
- `drdds-ros2-msgs_1.0.7_arm64.deb`
- `libfastdds-u20-arm64_2.14.0_arm64.deb`
- `SHA256SUMS`

`~/m20-aos-zenoh/` must contain:

- `zenohd`
- `zenoh-router.json5`
- `zenoh-router.service`
- `zenoh-150/`
- `SHA256SUMS`
- `README.md`

The package and binary files are ARM64 artifacts and cannot run on an x86_64
Laptop.

## Part 1: Configure the NOS

The NOS requires the DeepRobotics drdds SDK, Fast DDS, zenoh-c, and the DimOS
C++ Bridge.

### 1. Connect to the NOS

Run from the Laptop:

```sh skip
ssh user@10.21.31.106
```

Confirm the target architecture:

```sh skip
uname -m
```

The output must be `aarch64`.

### 2. Install the DeepRobotics Packages

Copy the reconstructed packages from the Laptop:

```sh skip
scp -r ~/m20-drdds-packages user@10.21.31.106:/home/user/
```

On the NOS, verify and install them in dependency order:

```sh skip
cd /home/user/m20-drdds-packages
sha256sum -c SHA256SUMS

sudo dpkg -i ./libfastdds-u20-arm64_2.14.0_arm64.deb
sudo dpkg -i ./drddslib_1.2.1_arm64.deb
sudo dpkg -i ./drdds-ros2-msgs_1.0.7_arm64.deb
sudo ldconfig
```

Confirm the installed versions and libraries:

```sh skip
dpkg-query -W \
  libfastdds-u20-arm64 \
  drddslib \
  drdds-ros2-msgs

ldconfig -p | grep -E 'libdrdds|libfastrtps|libfastcdr'
```

Expected versions:

| Package | Version |
| --- | --- |
| `libfastdds-u20-arm64` | `2.14.0` |
| `drddslib` | `1.2.1` |
| `drdds-ros2-msgs` | `1.0.7` |

### 3. Install zenoh-c

The current Bridge targets zenoh-c `1.2.0`. Do not build an arbitrary version
from the repository default branch.

Install the build prerequisites and Rust on the NOS if they are not already
available:

```sh skip
sudo apt-get update
sudo apt-get install -y build-essential cmake clang curl git

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustup default stable
```

Build and install zenoh-c `1.2.0`:

```sh skip
cd /home/user
git clone --recursive --branch 1.2.0 \
  https://github.com/eclipse-zenoh/zenoh-c.git

cmake -S zenoh-c -B zenoh-c/build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local
cmake --build zenoh-c/build --config Release -j"$(nproc)"
sudo cmake --install zenoh-c/build
sudo ldconfig
```

Verify the installation:

```sh skip
grep ZENOH_C /usr/local/include/zenoh_configure.h
find /usr/local/lib -name 'libzenohc.so*' -print
ldconfig -p | grep libzenohc
```

The header should report `ZENOH_C "1.2.0"`.

### 4. Download and Build the Bridge

Clone the branch containing the M20 Bridge:

```sh skip
cd /home/user
export GIT_LFS_SKIP_SMUDGE=1
git clone --depth 1 \
  --branch new_simple_nav_merged_main \
  https://github.com/MeloLong/dimos.git \
  dimos-bridge
```

Build the Bridge on the NOS:

```sh skip
cd /home/user/dimos-bridge/dimos/robot/deeprobotics/m20/onboard/drdds-zenoh-bridge/cpp
./build.sh
```

The build script downloads the generated C++ message headers from
`dimos-lcm`. If the NOS cannot access GitHub, copy a `dimos-lcm` checkout from
the Laptop to `/tmp/dimos-lcm` before running `./build.sh` again.

See the
[Bridge reference](/dimos/robot/deeprobotics/m20/onboard/drdds-zenoh-bridge/README.md)
for all supported ports and diagnostic options.

### 5. Start the Bridge

Keep the Bridge running while using DimOS:

```sh skip
cd /home/user/dimos-bridge/dimos/robot/deeprobotics/m20/onboard/drdds-zenoh-bridge/cpp
M20_AOS_IP=10.21.31.103

sudo ./build/m20_drdds_zenoh_bridge \
  --aligned 'dimos/slam_aligned_points#sensor_msgs.PointCloud2' \
  --aligned_topic /ALIGNED_POINTS \
  --grid 'dimos/grid_map_3d#sensor_msgs.PointCloud2' \
  --grid_topic /grid_map_3d \
  --odometry 'dimos/slam_odom#nav_msgs.Odometry' \
  --odom_topic /ODOM \
  --iface eth1 \
  --domain 0 \
  --connect "tcp/${M20_AOS_IP}:7447"
```

The Bridge requires root access to the drdds shared-memory transport.

## Part 2: Configure the AOS Zenoh Router

The AOS runs the ARM64 `zenohd` 1.5.0 binary as a systemd service. It does not
need drdds, zenoh-c development headers, or a full DimOS installation.

### 1. Copy the Router Bundle

Run from the Laptop:

```sh skip
scp -r ~/m20-aos-zenoh user@10.21.31.103:/home/user/
ssh user@10.21.31.103
```

On the AOS, verify the bundle:

```sh skip
cd /home/user/m20-aos-zenoh
sha256sum -c SHA256SUMS
./zenohd --version
```

The version command should report `zenohd v1.5.0`.

### 2. Check the AOS Network Configuration

The supplied service expects:

| Setting | Value |
| --- | --- |
| Internal interface | `eth2` |
| AOS address | `10.21.31.103` |
| Router listener | `tcp/0.0.0.0:7447` |

Check the actual AOS interfaces:

```sh skip
ip -br address
```

If the interface or address differs, update both files before installation:

- `zenoh-router.json5`: change `scouting.multicast.interface`.
- `zenoh-router.service`: change the interface and address in `ExecStartPre`.

### 3. Install and Enable the Router

```sh skip
cd /home/user/m20-aos-zenoh

sudo install -o user -g user -m 0755 \
  zenohd /home/user/zenohd
sudo install -o user -g user -m 0644 \
  zenoh-router.json5 /home/user/zenoh-router.json5
sudo install -o root -g root -m 0644 \
  zenoh-router.service /etc/systemd/system/zenoh-router.service

sudo systemctl daemon-reload
sudo systemctl enable --now zenoh-router.service
```

`systemctl enable` configures automatic startup during boot. The service waits
until `eth2` owns `10.21.31.103`, starts the Router, and restarts it after a
failure.

Verify the service:

```sh skip
systemctl is-enabled zenoh-router.service
systemctl is-active zenoh-router.service
systemctl status zenoh-router.service --no-pager
sudo ss -lntp | grep ':7447'
```

The expected states are `enabled` and `active`, with `zenohd` listening on TCP
port `7447`.

## Part 3: Install and Run DimOS on the Laptop

Use Ubuntu 22.04 or 24.04 with Python 3.12.

### 1. Install System Dependencies

```sh skip
sudo apt-get update
sudo apt-get install -y \
  curl g++ git git-lfs libturbojpeg portaudio19-dev python3-dev pre-commit

curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustup default stable
```

### 2. Install DimOS

```sh skip
mkdir -p ~/workplace
cd ~/workplace
export GIT_LFS_SKIP_SMUDGE=1
git clone --branch new_simple_nav_merged_main \
  https://github.com/MeloLong/dimos.git
cd dimos

uv sync --extra all
source .venv/bin/activate
```

Confirm that the installed registry contains the blueprint:

```sh skip
dimos list | grep m20-simple-nav
```

Do not regenerate `dimos/robot/all_blueprints.py` as part of runtime setup. It
is a checked-in generated source file, not a local installation artifact.

### 3. Verify Network Connectivity and Topics

```sh skip
M20_AOS_IP=10.21.31.103
ping -c 3 "$M20_AOS_IP"
timeout 3 bash -c "</dev/tcp/${M20_AOS_IP}/7447"
```

With the AOS Router and NOS Bridge running, inspect the Zenoh topics:

```sh skip
cd ~/workplace/dimos
source .venv/bin/activate
M20_AOS_IP=10.21.31.103

DIMOS_ZENOH_CONNECT="tcp/${M20_AOS_IP}:7447" \
  dimos spy --transport zenoh -n --duration 7 --interval 1
```

The output should include:

- `slam_aligned_points`
- `grid_map_3d`
- `slam_odom`

Do not start navigation until these topics are present and updating.

### 4. Launch M20 Simple Navigation

```sh skip
cd ~/workplace/dimos
source .venv/bin/activate
M20_AOS_IP=10.21.31.103

dimos --transport zenoh \
  --zenoh-connect "tcp/${M20_AOS_IP}:7447" \
  run m20-simple-nav
```

The first launch compiles the Rust ray-tracing module and can take 2 to 10
minutes. The blueprint automatically loads
`dimos/robot/deeprobotics/m20/config/m20_simple_nav.yaml`; no additional
`--config` argument is required.

To use another deployment profile:

```sh skip
cd ~/workplace/dimos
source .venv/bin/activate
M20_AOS_IP=10.21.31.103
DEPLOYMENT_CONFIG=/absolute/path/to/deployment-profile.yaml

dimos --transport zenoh \
  --zenoh-connect "tcp/${M20_AOS_IP}:7447" \
  run m20-simple-nav \
  --config "$DEPLOYMENT_CONFIG"
```

## Using the Navigation Interface

After the system starts:

1. Select the keyboard icon in the lower-right corner to enable keyboard
   teleoperation.
2. Select a reachable blue region on the map to submit a navigation goal.
3. DimOS plans a path and commands the M20 to move to the selected destination.

Perform the first hardware test in an open area and remain ready to take over
or stop the robot.

## Startup Order

Start the components in this order:

1. AOS `zenoh-router.service`.
2. NOS `m20_drdds_zenoh_bridge`.
3. Laptop `m20-simple-nav`.

## Troubleshooting

### The AOS Router Does Not Start

```sh skip
systemctl status zenoh-router.service --no-pager
journalctl -u zenoh-router.service -n 100 --no-pager
ip -br address
```

If the service remains in `activating`, its `ExecStartPre` is probably waiting
for the configured interface or IP address.

### The Laptop Cannot See M20 Topics

Check that:

1. The NOS Bridge is running and receiving drdds samples.
2. The NOS `--iface` value matches its robot-internal network interface.
3. The NOS `--connect` and Laptop `--zenoh-connect` values point to the AOS.
4. The AOS Router is listening on TCP `7447`.

### The Ray-Tracing Module Does Not Build

```sh skip
cd ~/workplace/dimos
source "$HOME/.cargo/env"
rustup update stable
rustup override set stable
cd dimos/mapping/ray_tracing/rust
cargo build --release --bin voxel_ray_tracing
```

### Cargo Cannot Parse `Cargo.lock`

If Cargo reports that lock-file version 4 requires
`-Znext-lockfile-bump`, update and select the stable Rust toolchain:

```sh skip
source "$HOME/.cargo/env"
rustup update stable
cd ~/workplace/dimos
rustup override set stable
```

### Cargo Uses an Unavailable `rsproxy` Mirror

```sh skip
test ! -f ~/.cargo/config || mv ~/.cargo/config ~/.cargo/config.disabled
test ! -f ~/.cargo/config.toml || \
  mv ~/.cargo/config.toml ~/.cargo/config.toml.disabled
```

Retry the ray-tracing build or launch after disabling the stale mirror.
