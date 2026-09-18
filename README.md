# Howdy Next

C++ rewrite of Howdy facial-recognition authentication on Linux

> [!NOTE]
>
> Howdy Next is feature-complete and actively maintained. Development now
> focuses on bug fixes, security fixes, and platform/dependency compatibility.
> Low commit activity does not mean the project is abandoned.

## Packages

### Arch Linux (AUR)

> [!WARNING]
>
> Do not mix source installs with AUR package files.
> Use AUR packages to avoid mixed files under `/usr` and `/usr/local`.

Packages:

- `howdy-next`
- `howdy-next-git`

Example:

```sh
paru -S howdy-next
```

### NixOS

Use the included flake directly:

```sh
nix build
nix develop
```

## Build from Source

Dependencies:

```text
glibc>=2.34, cmake>=3.31, acl, pkgconf, gettext, libevdev, libinih>=59, opencv>=5.0.0, libcurl>=7.85.0, openssl, yyjson>=0.12.0, pam
```

Build tools: GCC/Clang, GNU Make/Ninja, CMake 3.31+.

Howdy uses OpenCV HighGUI for graphical preview. An OpenCV package may bring a
GUI backend such as Qt, depending on how the distribution builds OpenCV; Qt is
not a direct Howdy dependency.

### Ubuntu 24.04 (Noble)

Noble packages most dependencies, but four are absent or below the required
version:

| Dependency | Noble ships  | Howdy needs | Fix                    |
|------------|--------------|-------------|------------------------|
| `cmake`    | 3.28.3       | 3.31        | Kitware apt repository |
| `opencv`   | 4.6.0        | 5.0.0       | Build from source      |
| `libinih`  | 55           | 59          | Build from source      |
| `yyjson`   | not packaged | 0.12.0      | Build from source      |

Everything else comes from apt:

```sh
sudo apt install build-essential pkgconf gettext meson ninja-build \
  libpam0g-dev libevdev-dev libacl1-dev libssl-dev libcurl4-openssl-dev
```

For CMake 3.31 or later, follow the Kitware apt repository instructions at
<https://apt.kitware.com/>.

#### OpenCV 5

Howdy needs the `core`, `imgproc`, `imgcodecs`, `videoio`, `highgui`, `dnn` and
`objdetect` modules. Qt 6 gives HighGUI a preview backend. Install to
`/opt/opencv5` to keep the build apart from the apt `libopencv-dev` 4.6 files:

```sh
sudo apt install qt6-base-dev libv4l-dev libeigen3-dev libjpeg-dev libpng-dev \
  libtiff-dev libwebp-dev libavformat-dev libswscale-dev libtbb-dev

git clone --depth 1 --branch 5.0.0 https://github.com/opencv/opencv.git
cmake -S opencv -B opencv/build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/opt/opencv5 \
  -DBUILD_LIST=core,imgproc,imgcodecs,videoio,highgui,dnn,objdetect \
  -DOPENCV_GENERATE_PKGCONFIG=ON \
  -DWITH_QT=6 \
  -DBUILD_TESTS=OFF -DBUILD_PERF_TESTS=OFF -DBUILD_EXAMPLES=OFF
cmake --build opencv/build --parallel "$(nproc)"
sudo cmake --install opencv/build
echo /opt/opencv5/lib | sudo tee /etc/ld.so.conf.d/opencv5.conf
sudo ldconfig
```

`OPENCV_GENERATE_PKGCONFIG` is off by default and produces the `opencv5.pc`
file that Howdy looks for.

#### inih and yyjson

Both install to `/usr/local`, which pkg-config and the linker cache search
before `/usr`, so the newer `INIReader` shadows the apt `libinih-dev` 55 files:

```sh
git clone --depth 1 --branch r62 https://github.com/benhoyt/inih.git
meson setup inih/build --prefix=/usr/local --buildtype=release \
  -Ddistro_install=true -Dwith_INIReader=true -Ddefault_library=shared
meson compile -C inih/build
sudo meson install -C inih/build

git clone --depth 1 --branch 0.13.0 https://github.com/ibireme/yyjson.git
cmake -S yyjson -B yyjson/build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  -DBUILD_SHARED_LIBS=ON
cmake --build yyjson/build
sudo cmake --install yyjson/build
sudo ldconfig
```

#### Configure Howdy

Point CMake at the OpenCV prefix so pkg-config finds `opencv5`:

```sh
cmake --preset release -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_PREFIX_PATH=/opt/opencv5
```

### Release

For a distro-style system install, configure an appropriate prefix for your
system; `/usr` is an example:

```sh
cmake --preset release -DCMAKE_INSTALL_PREFIX=/usr
cmake --build --preset release --parallel "$(nproc)"
ctest --preset release
```

PAM configuration uses `pam_howdy.so`. By default, Howdy installs it under
`${CMAKE_INSTALL_FULL_LIBDIR}/security`. If your distribution uses a different
PAM module directory, override it during configuration:

```sh
cmake --preset release \
  -DCMAKE_INSTALL_PREFIX=/usr \
  -DHOWDY_PAM_DIR=/path/to/pam/security
```

Install with the configured prefix:

```sh
sudo cmake --install build
```

`cmake --install --prefix` is intentionally unsupported because runtime paths
are generated during configuration. Reconfigure with
`-DCMAKE_INSTALL_PREFIX=<path>` to change the install prefix.

See [Contributing](CONTRIBUTING.md) for development, Debug builds, testing, and
contribution workflow.

### Setup

1. Identify camera path, preferably under `/dev/v4l/by-id/` or
   `/dev/v4l/by-path/`.
2. Run `sudo howdy config` and set `[video] device_path` to that path.
3. Run `sudo howdy add` to add face model.
4. Run `sudo howdy test` to test.

## CLI

See `howdy(1)` for the complete command and option reference.
See `pam_howdy(8)` for PAM configuration and workaround modes.

## Troubleshooting

Errors print to console. Check `journalctl -b -t pam_howdy` if auth fails quietly.

See [wiki](https://codeberg.org/nathawat/howdy-next/wiki/Troubleshooting) for
common issues.

> [!WARNING]
> Howdy is weaker than a password. Similar faces or photos may fool it.
> IR reduces spoofing because IR light is not reproduced by photos or LCD displays.
> Never use Howdy as the sole authentication method.

## License

Howdy Next is licensed under the GNU General Public License v3.0 or later
(`GPL-3.0-or-later`).

Revisions prior to the GPL relicensing commit were released under the MIT
License.

Third-party materials retain their respective licenses. See
[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
