# 🛰️ Satellite Image Concepts — Explained Simply

---

## 1. What is a "Band"?

Think of a satellite camera like a set of **multiple cameras stacked together**, each one taking a photo but seeing a **different type of light**.

Your phone camera sees **3 types of light**: Red, Green, Blue (RGB). That's 3 bands.

A satellite camera might see more — including **light your eyes can't see** (like infrared).

```
Band 1 → Blue channel   (how much blue light each pixel reflects)
Band 2 → Green channel  (how much green light each pixel reflects)
Band 3 → Red channel    (how much red light each pixel reflects)
Band 4 → NIR — Near-Infrared  (invisible to humans, but plants reflect a lot of it)
```

So a **GeoTIFF with 4 bands** = 4 separate grayscale images of the same area, each showing a different type of light.

When you combine **Red + Green + Blue** bands → you get the natural color photo you're used to seeing.

---

## 2. What is b1, b2, b3?

When TiTiler (the tile server) returns statistics, it names the bands **b1, b2, b3, b4...**

```
b1 = Band 1 = Blue
b2 = Band 2 = Green
b3 = Band 3 = Red
b4 = Band 4 = NIR (Near-Infrared)
```

So when you see `"b1": {"min": 0, "max": 255, ...}` it just means:

> "Here are the pixel value statistics for the **Blue band**"

---

## 3. What are pixel values?

Each pixel in a satellite image is just **a number**.

- For an 8-bit image: pixel values go from **0 (black) to 255 (white)**
- For a 16-bit image: pixel values go from **0 to 65535**

Example:

```
A pixel with value 0   → completely dark
A pixel with value 128 → middle grey
A pixel with value 255 → completely bright
```

When you combine 3 bands (R, G, B), each pixel becomes 3 numbers:
`(Red=210, Green=145, Blue=90)` → this becomes a brownish color on screen.

---

## 4. Why do we need "Rescale"?

Here's the problem. Imagine your satellite captured this:

```
Darkest pixel in the image  → value  2000
Brightest pixel in the image → value 4500
```

But your screen only understands **0 to 255**.

If you display the image as-is with values 2000–4500, everything would show as **pure white** (clipped) because 2000 > 255. The image would be invisible!

**Rescale = "stretch" the pixel values to fit the 0–255 screen range.**

```
Original range: 2000 → 4500
After rescale:  2000 → 0 (black),  4500 → 255 (white)
Everything in between gets proportionally mapped
```

This is exactly like adjusting **"brightness & contrast"** in Photoshop — you're telling the display: "treat value 2000 as black and 4500 as white."

---

## 5. Why Percentiles? Why not just use min and max?

Here's the real-world problem with using raw min/max:

Imagine a satellite image of a coal mine. 99.9% of the image is soil and rock with values between **1800 and 4200**. But there's **one tiny pixel** (maybe a reflection off a metal roof) with value **45000**.

```
If you use: min=0, max=45000
→ The entire image gets squished into a tiny dark range
→ Everything looks nearly black
→ You can see nothing useful
```

The fix: **ignore the extreme outliers** at both ends.

### Percentile_2 and Percentile_98

- **Percentile 2** = the value below which only 2% of pixels fall → ignore those dark outliers
- **Percentile 98** = the value above which only 2% of pixels fall → ignore those bright outliers
- **Use the range between them** as your black-to-white stretch

```
All pixel values sorted: [2, 5, 7, 10, 12, ..., 4100, 4150, 45000]
                                ↑                        ↑
                          percentile_2             percentile_98
                              (12)                    (4150)

Rescale: 12 → 0 (black),  4150 → 255 (white)
The outlier at 45000 clips to white but doesn't ruin the whole image.
```

This technique is called **histogram stretching** and is standard in ALL satellite image processing.

---

### 📌 Deep Example — Understanding Percentiles Step by Step

Let's say the **Red band (b3)** of your mining survey image has **100 pixels** total (small example).
Here are all 100 pixel values, sorted from darkest to brightest:

```
Position  Value    What it represents
────────────────────────────────────
1         0        Shadow inside a pit (pure black)
2         3        Very deep shadow
3         8        Deep shadow
──────────────────── ← 2nd PERCENTILE is here (2% of 100 = 2nd value) → value = 3
4         200      Dark rock
5         210      Dark rock
...       ...      ... (most pixels are soil, rock, vegetation: 800–3200)
...
97        3800     Bright concrete road
98        3900     Bright metal equipment shed
──────────────────── ← 98th PERCENTILE is here (98% of 100 = 98th value) → value = 3900
99        12000    Sun reflection off a wet puddle
100       45000    Direct sun glint off a steel roof
```

**What happens with different rescale choices:**

```
❌ Using raw min/max (0 to 45000):
   Pixel value 800  → screen value = (800/45000) × 255  =   4  ← nearly black!
   Pixel value 2000 → screen value = (2000/45000) × 255 =  11  ← still black!
   Pixel value 3200 → screen value = (3200/45000) × 255 =  18  ← still very dark!
   → Almost the entire image appears black. Unusable.

✅ Using percentile_2 to percentile_98 (3 to 3900):
   Pixel value 800  → screen value = (800-3)/(3900-3) × 255 =  51  ← dark grey ✓
   Pixel value 2000 → screen value = (2000-3)/(3900-3) × 255 = 130  ← mid grey ✓
   Pixel value 3200 → screen value = (3200-3)/(3900-3) × 255 = 209  ← bright ✓
   → Image looks natural and detailed!
   → The 2 outlier pixels (12000, 45000) just clip to white — no damage done.
```

**The formula used for every pixel:**

```
screen_value = (pixel_value - p2) / (p98 - p2)  ×  255

If result < 0   → clamp to 0   (darker than p2 = pure black)
If result > 255 → clamp to 255 (brighter than p98 = pure white)
```

---

### 📊 What is Histogram Stretching? (Detailed)

A **histogram** is just a bar chart that shows:

- X-axis = pixel value (0 to 4500 or whatever the range is)
- Y-axis = how many pixels have that value

**Before stretching** — the histogram of a raw satellite image looks like this:

```
Count
  │
  │         ████
  │        ██████
  │       █████████
  │      ████████████
  │    ██████████████████
  │  ████████████████████████
  │                               █  ← outlier (1 pixel at 45000)
  └──────────────────────────────────────────────► Pixel value
  0   500  1000  2000  3000  4000       45000
       ↑                        ↑
    most pixels                rare outlier
    cluster here                way out here
```

The problem: the interesting data (the big cluster in the middle) uses only a tiny slice of the 0–255 display range because the outlier forces the scale to go all the way to 45000.

**Histogram Stretching** = zoom in on the cluster, cut off the outliers:

```
BEFORE stretching (original values squeezed into screen):
┌─────────────────────────────────────────────────────┐
│ 0                                               255  │  ← screen range
│ │                                                 │  │
│ ██████████████████ (all real data)  │ [outlier]  │  │
│ └────tiny slice────┘                             │  │
└─────────────────────────────────────────────────────┘

AFTER stretching (percentile range maps to full screen):
┌─────────────────────────────────────────────────────┐
│ 0                                               255  │  ← screen range
│ │                                                 │  │
│ ████████████████████████████████████████████████ │  │
│ └──────────────full range used──────────────────┘ │  │
└─────────────────────────────────────────────────────┘
```

**Analogy:** Imagine you have a photo and the brightness slider goes from 0 to 10000, but all your photo's interesting content is between 200 and 3000. Histogram stretching is like saying:

> "Let's zoom in so that 200 = leftmost (black) and 3000 = rightmost (white). Ignore everything outside that range."

Now every shade of grey in the interesting zone gets its own distinct screen color → **you can see detail** that was invisible before.

---

## 6. Why Per-Band Rescale?

Different bands don't have the same brightness range. Example:

```
b1 (Blue)  : pixel values range from 200 to 1800
b2 (Green) : pixel values range from 300 to 2500
b3 (Red)   : pixel values range from 100 to 3000
```

If you use one global rescale (e.g. 100 to 3000 for all 3), the Blue band would look washed out because most of its values (200–1800) are in the lower half of the range.

**Per-band rescale** stretches each band independently, so each band uses its own full contrast range → the image looks much more natural and detailed.

In the code:

```
rescale = "12,210&rescale=45,188&rescale=30,201"
           ───b1───            ───b2───           ───b3───
```

Each pair = `(percentile_2, percentile_98)` for that specific band.

---

### 📌 Deep Example — Per-Band Rescale Walkthrough

Let's use a real mining image. TiTiler returns these stats:

```
b1 (Blue)  → percentile_2 = 12,  percentile_98 = 210
b2 (Green) → percentile_2 = 45,  percentile_98 = 188
b3 (Red)   → percentile_2 = 30,  percentile_98 = 201
```

This gives us: `rescale = "12,210&rescale=45,188&rescale=30,201"`

**Now let's trace a single pixel** — say it's a patch of bare mining soil:

```
This pixel's raw values:
  b1 (Blue)  = 95
  b2 (Green) = 110
  b3 (Red)   = 160

Applying per-band stretch:
  Blue  screen = (95  - 12) / (210 - 12) × 255 = 83/198  × 255 ≈ 107  (medium blue)
  Green screen = (110 - 45) / (188 - 45) × 255 = 65/143  × 255 ≈ 116  (medium green)
  Red   screen = (160 - 30) / (201 - 30) × 255 = 130/171 × 255 ≈ 194  (bright red)

Final screen pixel = (R=194, G=116, B=107) → warm brownish-orange = typical mine soil ✓
```

**What if we used ONE global rescale instead (min=12, max=210 for all bands)?**

```
  Blue  screen = (95  - 12) / (210 - 12) × 255 ≈ 107  ← same
  Green screen = (110 - 12) / (210 - 12) × 255 ≈ 126  ← slightly off
  Red   screen = (160 - 12) / (210 - 12) × 255 ≈ 190  ← slightly off

The Green channel uses b1's range (12–210) even though Green pixels mostly fall
between 45–188. This wastes screen dynamic range and creates colour cast.
```

Per-band stretching keeps each colour channel using its own full range → **most faithful, vivid colour output**.

---

## 7. What is bidx (Band Index)?

`bidx` tells TiTiler **which bands to use for the Red, Green, Blue channels** of the displayed image.

```
For a 3-band RGB image:
    bidx=1&bidx=2&bidx=3
    → Band 1 → Red display channel
    → Band 2 → Green display channel
    → Band 3 → Blue display channel

For a 4-band BGRN image (Sentinel-style):
    bidx=3&bidx=2&bidx=1
    → Band 3 (Red) → Red display channel     ← swapped to show true color
    → Band 2 (Green) → Green display channel
    → Band 1 (Blue) → Blue display channel
```

Without `bidx`, TiTiler would just show Band 1 (Blue) as a grayscale image.

---

## 8. What is NIR (Near-Infrared) used for?

Plants **absorb** red light (for photosynthesis) but **strongly reflect** NIR.

So:

- Healthy vegetation → very high NIR values, low red values
- Bare soil / mining areas → moderate NIR, moderate red
- Water → absorbs both → very low values in both

This is the basis of **NDVI** (Normalized Difference Vegetation Index):

```
NDVI = (NIR - Red) / (NIR + Red)

NDVI close to +1 → dense healthy vegetation
NDVI close to  0 → bare soil
NDVI close to -1 → water or bare rock
```

In a mining context, NDVI helps detect **how much vegetation has been cleared** and whether **reclamation areas** are actually re-greening.

---

## Quick Reference Cheat Sheet

| Term                     | Plain English                                                              |
| ------------------------ | -------------------------------------------------------------------------- |
| **Band**                 | One layer of the image, capturing one type of light                        |
| **b1, b2, b3**           | Band 1, Band 2, Band 3 — TiTiler's naming                                  |
| **Pixel value**          | A number (0–255 or 0–65535) representing brightness in that band           |
| **Rescale**              | Stretch pixel values to fit screen (like adjusting brightness/contrast)    |
| **percentile_2**         | Ignore the darkest 2% of pixels (outliers)                                 |
| **percentile_98**        | Ignore the brightest 2% of pixels (outliers)                               |
| **Histogram stretching** | Using percentile range for rescale = better contrast                       |
| **Per-band rescale**     | Each band gets its own stretch range = more natural colors                 |
| **bidx**                 | Which bands to assign to R, G, B display channels                          |
| **NIR**                  | Infrared light — invisible to humans, but great for detecting plant health |
| **COG**                  | Cloud-Optimized GeoTIFF — special file layout for fast tile streaming      |
