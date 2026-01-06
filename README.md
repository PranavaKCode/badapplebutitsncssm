
# NCSSM Logo-fied "Bad Apple"
**Hardware:** NVIDIA H200 GPU | 128GB RAM | 16-Core CPU  

## 1.Goal
The goal of this project was to transform the 6,572 frames of the "Bad Apple" animation into a mosaic where every "pixel" of the video is represented by an NCSSM logo. The final product includes a full-length high-definition video and a "Convergence Finale" GIF that transitions into a single, high-resolution logo.

---
## 2. Technical Phase 1: Logo Engineering
To ensure a professional aesthetic, we avoided hard-thresholding (which causes jagged edges). Instead, we used a **soft-masking** technique in Python (Pillow) to preserve anti-aliasing.

### The Logic:
* **Dark Tile:** White logo on a Black background (Used for silhouettes).
* **Bright Tile:** Black logo on a White background (Used for the background).

```python
from PIL import Image, ImageOps

def create_tiles(logo_path, size):
    base = Image.open(logo_path).convert("RGBA")
    # Pad to square
    side = max(base.size)
    padded = Image.new("RGBA", (side, side), (255, 255, 255, 0))
    padded.paste(base, ((side - base.width)//2, (side - base.height)//2))
    base = padded.resize((size, size), Image.Resampling.LANCZOS)
    
    # Generate mask from luminance for smooth edges
    mask = ImageOps.invert(base.convert("L"))
    
    # White logo on Black
    w_on_b = Image.new("RGB", (size, size), (0, 0, 0))
    w_on_b.paste(Image.new("RGB", (size, size), (255, 255, 255)), (0, 0), mask=mask)
    
    # Black logo on White
    b_on_w = Image.new("RGB", (size, size), (255, 255, 255))
    b_on_w.paste(Image.new("RGB", (size, size), (0, 0, 0)), (0, 0), mask=mask)
    
    return w_on_b, b_on_w
```

---

## 3. Technical Phase 2: High-Performance Parallel Rendering
With 6,572 frames to process and 16 CPU threads available, we implemented a dual-pool architecture: a **ThreadPool** for network I/O (downloading frames from GitHub) and a **ProcessPool** for CPU-bound image manipulation.

### Core Rendering Logic:
Each frame was downsampled to a 120x90 grid. Each cell in that grid was then replaced with a logo tile based on the average brightness of that cell.

```python
def render_frame(data):
    frame_num, local_path = data
    img = Image.open(local_path).convert("L")
    grid_w, grid_h = 120, 90 # Determines logo density
    small = img.resize((grid_w, grid_h), Image.Resampling.BILINEAR)
    
    canvas = Image.new("RGB", (grid_w * 16, grid_h * 16))
    for y in range(grid_h):
        for x in range(grid_w):
            pixel = small.getpixel((x, y))
            tile = tile_bright if pixel > 127 else tile_dark
            canvas.paste(tile, (x * 16, y * 16))
    canvas.save(f"ncssm_output/out{frame_num:04d}.jpg")
```

---

## 4. Technical Phase 3: The "Convergence" Finale
A standard zoom into a raster image results in pixelation. To achieve a "crisp" reveal, we developed a Procedural Zoom Engine. Instead of scaling an image, the engine re-calculates the grid density for every frame, essentially re-drawing the logos at increasing sizes until one logo fills the entire screen.

### The Mathematical Formula for Zoom:
We used an exponential zoom factor to make the movement feel natural:
$$\text{Zoom} = 1.0 \times (\text{TargetGridWidth})^{\text{progress}}$$

```python
# Procedural grid calculation
visible_logos_w = max(1, int(BASE_GRID_W / zoom))
tile_size = FINAL_SIZE[0] // visible_logos_w

# Render a fresh canvas for every frame of the zoom
# This ensures logos are always re-drawn from source at 100% sharpness
```

---

## 5. Technical Phase 4: Production & Encoding
Working in an HPC environment required using a standalone **Static FFmpeg Build** to bypass permission restrictions. 

### Final Video Assembly (High Quality):
We used the H.264 codec with a high Constant Rate Factor (CRF) to maintain the sharpness of the logo edges.
```bash
./ffmpeg -framerate 30 -i ncssm_output/out%04d.jpg -c:v libx264 -crf 18 -pix_fmt yuv420p final_video.mp4
```

### Optimized GIF Production (Target < 10GB):
To reduce the GIF from a projected 15MB to a manageable 10GB, we utilized the following:
```py
import os, subprocess
from pathlib import Path

INP = Path("/content/ncssm bad apple.gif")        
TARGET = 10 * 1024 * 1024        # 10 MB in bytes

def run(cmd):
    p = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
    if p.returncode != 0:
        raise RuntimeError(p.stderr[-2000:])

def size_mb(p):
    return p.stat().st_size / (1024*1024)

tries = [
    # (fps, colors, outname)
    (None, 128, "out_128c.gif"),
    (None, 96,  "out_96c.gif"),
    (20,   128, "out_20fps_128c.gif"),
    (20,   96,  "out_20fps_96c.gif"),
    (15,   128, "out_15fps_128c.gif"),
]

for fps, colors, outname in tries:
    out = Path(outname)
    vf = "split[a][b];" if fps is None else f"fps={fps},split[a][b];"
    vf += f"[a]palettegen=max_colors={colors}:stats_mode=diff[p];"
    vf += "[b][p]paletteuse=dither=sierra2_4a:diff_mode=rectangle"

    print(f"Trying fps={fps or 'orig'}, colors={colors} -> {outname}")
    run(["ffmpeg","-y","-i",str(INP),"-filter_complex",vf,"-loop","0",str(out)])
    print(f"  size = {size_mb(out):.2f} MB")

    if out.stat().st_size <= TARGET:
        print(f"\nDONE: {outname} is under 10 MB.")
        break
else:
    print("\nFail.")
```

---

## 6. Notes
1. Using a 16-thread `ProcessPoolExecutor` reduced rendering time from hours to approximately 12 minutes.
2. On HPC systems, writing thousands of small JPEGs can be slow. Pre-batching downloads into 500-frame chunks helped maintain network stability.

## 7. Final Artifacts Generated
* `ncssm_output/`: 6,572 logo-fied raw frames.
* `ncssm_bad_apple.mp4`: Full-length video (Full HD).
* `ncssm_convergence_reveal.gif`: The procedural zoom-to-logo finale.
* `ncssm_bad_apple_optimized.gif`: A high-definition shareable version of the full animation.

---
**Project Status: Completed Successfully.**
