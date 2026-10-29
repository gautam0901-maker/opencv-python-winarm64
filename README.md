# opencv-python for Windows on Arm64 (unofficial build)

`pip install opencv-python` fails on Windows on Arm because the official
package has no `win_arm64` wheel. This repository provides a wheel built
natively on a Windows 11 Arm64 laptop, plus the steps and the one-line fix
needed to build it yourself.

> **Unofficial and experimental.** Not made or endorsed by the OpenCV team.
> "OpenCV" is a trademark of its owners. Tested only lightly (see below).

## Install

Requirements: **Windows 11 on Arm64** and **Python 3.12** (64-bit Arm Python).

Download the `.whl` file from the [Releases](../../releases) page, then:

```powershell
python -m pip install opencv_python-4.14.0+031876b-cp312-cp312-win_arm64.whl
```

The package name stays `opencv-python`, so libraries that depend on it
(for example Ultralytics) accept it.

Check:

```powershell
python -c "import cv2; print(cv2.__version__)"
```

## What was tested

On an ASUS ProArt (Windows 11 on Arm, Python 3.12.16, numpy 2.5.4):

- `import cv2` -> `4.14.0`
- `cv2.resize` on a numpy image
- `cv2.imencode('.jpg', ...)`

More testing (Ultralytics training) is in progress.

## Build details

| Item | Value |
|---|---|
| Source | opencv-python master, commit `031876b` (4 Sep 2026); OpenCV 4.14.0 |
| Compiler | Visual Studio 2022 Build Tools 17.14, MSVC v143 ARM64, Windows SDK 10.0.26100 |
| CMake | 4.4.4 (from pip) |
| CPU features | baseline NEON (FP16 / BF16 / DOTPROD dispatch not available with this compiler) |
| GUI | Win32 UI (not headless) |
| Video I/O | Media Foundation and DirectShow yes; **FFmpeg no** (no prebuilt Arm64 FFmpeg DLL) |
| Linking | static, everything in one `cv2.pyd` |

## Build it yourself

1. Install **Git for Windows (ARM64)** and **Visual Studio 2022 Build Tools** with
   "Desktop development with C++" and the individual component
   **"MSVC v143 - VS 2022 C++ ARM64/ARM64EC build tools (Latest)"**.
2. Get the source (use a short path):
   ```powershell
   git config --global core.longpaths true
   mkdir C:\cv; cd C:\cv
   git clone --recursive https://github.com/opencv/opencv-python.git
   cd opencv-python
   ```
3. Apply the fix in [`setup_py_arm64.patch`](setup_py_arm64.patch): `setup.py`
   expects an FFmpeg DLL on every Windows build, which does not exist on Arm64.
   Save the patch file into `C:\cv`, then:
   ```powershell
   git apply ..\setup_py_arm64.patch
   ```
   (or edit the line by hand: `if os.name == "nt"` becomes
   `if os.name == "nt" and platform.machine() != "ARM64"` in the
   `opencv_videoio_ffmpeg` entry.)
4. Install the build tools into your Python environment. The pinned build
   requirement numpy 2.0.2 has no Arm64 wheel, so we build without isolation:
   ```powershell
   python -m pip install --upgrade pip numpy "setuptools<70" packaging scikit-build cmake wheel
   ```
5. Build:
   ```powershell
   $env:CMAKE_GENERATOR = "Visual Studio 17 2022"
   python -m pip wheel . --verbose --no-deps --no-build-isolation -w wheels
   ```
   The compile took about 8 minutes on a Snapdragon-class laptop.
6. Afterwards, restore a newer setuptools if other packages need it
   (`python -m pip install "setuptools>=77"`).

## Errors we hit, and the fixes

| Error | Cause | Fix |
|---|---|---|
| `The BaseOutputPath/OutputPath property is not set ... Platform='ARM64'` | ARM64 MSVC component not installed | add "MSVC v143 ARM64/ARM64EC build tools (Latest)" |
| `Not found: 'bin/opencv_videoio_ffmpeg\d{4}_64\.dll'` | setup.py always expects FFmpeg on Windows | the patch above |
| pip tries to build numpy 2.0.2 from source | no Arm64 wheel for the pinned build numpy | `--no-build-isolation` with your own numpy |

## Licence

Build notes and patch: MIT (the patch modifies opencv-python's MIT-licensed
`setup.py`). The wheel contains OpenCV (Apache 2.0) and third-party
libraries; their licence texts are included in the wheel. See [LICENSE](LICENSE).

## Credits

OpenCV and opencv-python are made by their contributors. This repository only
documents a Windows-on-Arm64 build. The goal is to get this fixed upstream.
