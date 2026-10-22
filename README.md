# Interactive Luminance Adjustment

A standalone Python adaptation of an image-processing Jupyter notebook. Click a reference pixel to set a brightness threshold, then brighten or darken all pixels below or above it using a normalized exponential tone curve. Preview the result and save it as PNG or JPEG.

All processing logic is in **`adjust_luminance.py`**. Code comments, settings, and interface messages are in English.

## Features

- Select the input image through clearly marked settings in the script.
- Click a reference pixel and see its coordinates and approximate luminance, from 0 to 1.
- Adjust shadows below the reference or highlights above it.
- Control the direction and strength of the adjustment with one signed parameter.
- Compare the original and modified images, the target curve, and the output brightness map.
- Save to a configurable folder with a configurable or interactively entered filename.

The clicked pixel sets a **global brightness threshold**. It does not define a spatial selection: every pixel meeting the brightness condition is adjusted, wherever it appears in the image. The reference pixel itself remains unchanged in the floating-point result.

## Requirements

- Python 3.10 or later.
- NumPy, Matplotlib, and Pillow; listed in `requirements.txt`.
- A local desktop session and a Matplotlib GUI backend, such as TkAgg or QtAgg.

No Jupyter notebook, `%matplotlib widget`, OpenCV, or MediaPy is required.

## Installation

Download this repository, open a terminal in its folder, and optionally create a virtual environment:

```bash
python -m venv .venv
```

Activate it on Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Or on macOS/Linux:

```bash
source .venv/bin/activate
```

Install the dependencies:

```bash
python -m pip install -r requirements.txt
```

Use `python3` instead of `python` if that is the command for your installation.

## Files and image folders

| Item | Purpose |
| --- | --- |
| `adjust_luminance.py` | Complete standalone program and user settings. |
| `README.md` | Installation, configuration, usage, and algorithm notes. |
| `requirements.txt` | Python dependencies. |
| `GITHUB_DESCRIPTION.txt` | Short description for the GitHub repository About field. |
| `Imagenes_referencia/` | Default input folder; add your own images here. |
| `Imagenes_resultantes/` | Default output folder; created automatically when needed. |

No sample photographs are included. `Huerto1.jpg` is the original notebook's example filename, which you must supply or replace in the configuration. `.gitkeep` files only keep the initially empty folders visible in Git.

## Configuration

Edit the **USER CONFIGURATION** section near the start of `adjust_luminance.py`. Each editable setting is marked with a `# USER:` comment.

| Setting | Default | What to change |
| --- | --- | --- |
| `INPUT_DIR` | `SCRIPT_DIR / "Imagenes_referencia"` | Folder containing the source image. |
| `INPUT_FILENAME` | `"Huerto1.jpg"` | Your image filename, including its extension. |
| `OUTPUT_DIR` | `SCRIPT_DIR / "Imagenes_resultantes"` | Destination folder. |
| `OUTPUT_FILENAME` | `None` | Set a filename, or leave `None` to enter one in the terminal. |
| `MODE` | `"shadows"` | Choose `"shadows"` or `"highlights"`. |
| `K_VALUE` | `3.25` | Signed tone-curve strength. |
| `SAVE_RESULT` | `True` | Set `False` to preview without saving. |

The supplied folder paths are relative to the script's location, rather than the terminal's current working directory. Absolute paths are also supported. For example:

```python
INPUT_DIR = Path(r"C:\Users\YourName\Pictures")
INPUT_FILENAME = "my_photo.jpg"
OUTPUT_DIR = SCRIPT_DIR / "Imagenes_resultantes"
OUTPUT_FILENAME = "my_photo_adjusted.png"
MODE = "shadows"
K_VALUE = 3.25
```

Use a filename only in `OUTPUT_FILENAME`; configure its folder with `OUTPUT_DIR`. Existing files are not overwritten, and the output path must differ from the source path.

### Choose the adjustment

The sign of `K_VALUE` has the same meaning in both modes:

| Goal | `MODE` | Suggested `K_VALUE` |
| --- | --- | --- |
| Brighten pixels darker than the reference | `"shadows"` | `3.25` |
| Darken pixels darker than the reference | `"shadows"` | `-3.25` |
| Darken pixels brighter than the reference | `"highlights"` | `-3.25` |
| Brighten pixels brighter than the reference | `"highlights"` | `3.25` |
| Leave all pixels unchanged before saving | Either mode | `0.0` |

`K_VALUE` shapes a curve; **it is not a direct multiplication factor**. Increasing its absolute value generally makes the adjustment stronger. Start with small values and experiment with the reference pixel. Very strong brightening can saturate RGB channels.

## Run the program

Run from a terminal, rather than pasting the script into notebook cells:

```bash
python adjust_luminance.py
```

1. The first window shows the original image on the left and its brightness map on the right.
2. Left-click a pixel in the **original image**. A red ring marks it; its coordinates and reference brightness are displayed and printed in the terminal. Click again to change the reference. Turn off the pan/zoom toolbar tool before selecting.
3. Press **Enter** while the plot window is active, or close it, to confirm the last selection. **Esc** cancels. Closing the window without selecting a pixel also cancels.
4. Review the second window: original image, modified image, target tone curve, and actual output brightness map. Close this window to continue.
5. If `SAVE_RESULT = True` and `OUTPUT_FILENAME = None`, return to the terminal and enter an output name. Press Enter without a name to skip saving.

Examples: `my_result.png`, `my_result.jpg`, or `my_result`. A name without an extension uses `.jpg`. PNG avoids additional JPEG compression; JPEG is written at quality 95. The selected reference and strength apply to one run; edit the settings and rerun to try another adjustment.

## How the adjustment works

### 1. Approximate luminance

The image is loaded as floating-point RGB values between 0 and 1. The notebook's brightness calculation is retained:

```text
L = 0.2126 * R + 0.7152 * G + 0.0722 * B
```

These weights are applied directly to encoded RGB values, without sRGB linearization. The result is an approximate **luma/brightness measure**, rather than physical luminance. This preserves the notebook's artistic image-processing behavior.

### 2. Select and normalize the affected interval

Let `b0` be the clicked reference brightness and `b` a pixel's original brightness.

| Mode | Affected pixels | Normalized brightness `t` | Mapped brightness |
| --- | --- | --- | --- |
| Shadows | `b < b0` | `b / b0` | `b0 * g(t)` |
| Highlights | `b > b0` | `(b - b0) / (1 - b0)` | `b0 + (1 - b0) * g(t)` |

Pixels outside the affected interval are left unchanged. Both endpoints of the interval stay fixed: pure black remains black, pure white remains white, and the reference brightness remains unchanged. Choosing black in shadows mode, or white in highlights mode, leaves no interval to adjust.

### 3. Exponential tone curve

For nonzero `k`:

```text
g(t) = (1 - exp(-k * t)) / (1 - exp(-k))
```

For `k = 0`, the limiting curve is `g(t) = t`, so there is no change. The implementation uses `expm1` and an equivalent expression for negative strengths to avoid cancellation near zero and exponential overflow at large negative values.

### 4. Apply the target brightness to RGB

For each nonblack pixel, all three channels are multiplied by the same ratio:

```text
scale = target_luminance / original_luminance
modified_RGB = clip(original_RGB * scale, 0, 1)
```

Black pixels are retained without division. Channel ratios are preserved before clipping. If a channel exceeds 1, clipping can change the color and prevent the exact target brightness from being reached. The terminal reports how many pixels encounter this clipping; the output brightness map is recalculated from the clipped RGB result.

## Changes from the notebook

- All cells and callback state are integrated into one executable script.
- The original shadows curve is preserved; a highlights mode extends the same curve to the interval above the reference.
- `k = 0` is explicitly handled instead of causing division by zero.
- The RGB scaling step uses masked division, avoiding the notebook's slight dimming of unchanged pixels caused by adding epsilon to every denominator.
- Plots use a fixed brightness scale from 0 to 1, and the final map shows the actual clipped output rather than only the ideal target.
- Pillow replaces MediaPy for image loading/saving; unused notebook imports and Jupyter magic are removed.
- Input checks, destination-folder creation, and protection against overwriting existing files are included.

## Practical limitations

- This is a global brightness-based artistic adjustment, not a spatial brush or a two-threshold band selection.
- The source is converted to 8-bit RGB. Transparency and original metadata are not preserved; high-bit-depth/HDR and color-managed workflows are outside this script's scope. Camera/phone EXIF orientation is applied before processing.
- Calculations use floating-point arrays, but saved images are quantized to 8-bit RGB. JPEG compression can introduce additional pixel changes, including where the floating-point processing left pixels untouched.
- The program requires an interactive desktop window. A headless server or a notebook's inline plotting backend cannot provide the intended click-selection workflow.

## Troubleshooting

- **Input image not found:** supply your image and update `INPUT_DIR` and `INPUT_FILENAME`.
- **No interactive window:** run locally with a GUI-enabled Matplotlib installation. Standard desktop Python installations often provide Tk; Linux may require an OS package such as `python3-tk`. `python -m tkinter` can check Tk availability. If Tk is available, you can select TkAgg with the `MPLBACKEND` environment variable before launching: `$env:MPLBACKEND = "TkAgg"` in PowerShell, or `export MPLBACKEND=TkAgg` in a POSIX shell.
- **No visible change:** check that `K_VALUE` is nonzero and that pixels exist below/above your reference for the selected mode. Try another reference pixel.
- **Colors saturate:** reduce positive `K_VALUE` or choose another threshold.
- **Output already exists:** choose a different output filename.
- **Saving seems paused:** close the results window, then check the terminal for the filename prompt.

## GitHub description

Copy the contents of `GITHUB_DESCRIPTION.txt` into your repository's About description. The description is under 350 characters. Choose your own repository license before publishing if you want to specify reuse permissions.
