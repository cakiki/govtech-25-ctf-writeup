---
layout: page
title: StrideSafe
description: 403 points
importance: 3
category: "Machine Learning & Data"
---

> #### Challenge
> Singapore's lamp posts are getting smarter. They don't just light the way â€” they watch over the pavements.
>
> Your next-gen chip has been selected for testing. Can your chip distinguish pedestrians from bicycles and PMDs (personal mobility devices)?
>
> Pass the test, and your chip will earn deployment on Singapore's smart lamp posts. Fail, and hazards roam free on pedestrian walkways.
{: .block-tip }

#### Solution
given the challenge desc and the dist files, we figured the challenge would be a matter of classifying images and creating a QR code based off the images. Trying various imagenet/resnet models did not work, so we attempted CLIP (which was alluded to in a comment in the page's HTML source code: `<!-- TODO: Test on other vision-language models other than OpenAI CLIP -->`) and colored based off whether it was a person, bike, or PMD. This gave us a QR code that scanned.

{% highlight python linenos %}
# pip install git+https://github.com/openai/CLIP.git ftfy regex tqdm pillow
import glob, json, math, os
from pathlib import Path

import numpy as np
import torch
import clip
from PIL import Image
import matplotlib.pyplot as plt

DEVICE = "cuda" if torch.cuda.is_available() else "cpu"
CLIP_MODEL = "ViT-B/32"

LABEL_GROUPS = {
    "pedestrian": [
        "a pedestrian",
        "a person",
        "a human",
        "a profile",
        "a nun",
        "a man",
        "a child",
    ],
    "bicycle": [
        "a bicycle",
        "a motorcycle",
        "a cyclist",
        "a skateboard",
    ],
    "pmd": [
        "a personal mobility device",
        "a wheelchair",
        "a scooter"
    ],
}

NONE_THRESHOLD = 0.175


BUCKETS = {
    "pedestrian": "white",
    "bicycle": "black",
    "pmd": "black",
}
TILE_SIZE = 8 

model, preprocess = clip.load(CLIP_MODEL, device=DEVICE)

def encode_text_groups(label_groups: dict[str, list[str]]):
    prompts = []
    class_slices = []  # (class_name, start, end)
    start = 0
    for cls, texts in label_groups.items():
        toks = clip.tokenize(texts).to(DEVICE)
        with torch.no_grad():
            feats = model.encode_text(toks)
        feats = feats / feats.norm(dim=-1, keepdim=True)
        prompts.append(feats)
        end = start + feats.shape[0]
        class_slices.append((cls, start, end))
        start = end
    all_feats = torch.cat(prompts, dim=0)  # (sum_prompts, d)

    class_feats = []
    names = []
    for cls, s, e in class_slices:
        cf = all_feats[s:e].mean(dim=0, keepdim=True)
        cf = cf / cf.norm(dim=-1, keepdim=True)
        class_feats.append(cf)
        names.append(cls)
    class_feats = torch.cat(class_feats, dim=0)  # (C, d)
    return names, class_feats

CLASS_NAMES, CLASS_TEXT_FEATS = encode_text_groups(LABEL_GROUPS)  # (C, d)

paths = sorted(glob.glob("*.jpg"))
if not paths:
    raise SystemExit("No .jpg images found in the current folder.")

def encode_images(image_paths, batch=32):
    ims = []
    for p in image_paths:
        im = Image.open(p).convert("RGB")
        ims.append(preprocess(im))
    feats = []
    with torch.no_grad():
        for i in range(0, len(ims), batch):
            batch_tensor = torch.stack(ims[i:i+batch]).to(DEVICE)
            f = model.encode_image(batch_tensor)
            f = f / f.norm(dim=-1, keepdim=True)
            feats.append(f)
    feats = torch.cat(feats, dim=0)  # (N, d)
    return feats

image_feats = encode_images(paths)  # (N, d)

sims = (image_feats @ CLASS_TEXT_FEATS.T).cpu().numpy()  # cosine similarity
pred_idxs = sims.argmax(axis=1)           # shape: (N,)
pred_scores = sims.max(axis=1)            # shape: (N,)
pred_names = [CLASS_NAMES[i] for i in pred_idxs]

def assign_bucket(name: str, score: float) -> str:
    if score < NONE_THRESHOLD:
        return "neither"
    return BUCKETS.get(name, "neither")

buckets = [assign_bucket(n, s) for n, s in zip(pred_names, pred_scores)]


n = len(buckets)
side = int(math.ceil(math.sqrt(n)))  # mosaic dimension in tiles
pad = side * side - n

white_tile = np.ones((TILE_SIZE, TILE_SIZE), dtype=np.float32)
black_tile = np.zeros((TILE_SIZE, TILE_SIZE), dtype=np.float32)

yy, xx = np.mgrid[0:TILE_SIZE, 0:TILE_SIZE]
checker_tile = ((xx + yy) % 2).astype(np.float32)  # 0/1 alternating

tile_map = {
    "white": white_tile,
    "black": black_tile,
    "neither": checker_tile,
}

tiles = [tile_map[b] for b in buckets] + [checker_tile] * pad

rows = []
for r in range(side):
    row_tiles = tiles[r*side:(r+1)*side]
    rows.append(np.concatenate(row_tiles, axis=1))
mosaic = np.concatenate(rows, axis=0)

plt.figure(figsize=(6, 6))
plt.imshow(mosaic, cmap="gray", vmin=0.0, vmax=1.0)
plt.title("white / black / checker = neither")
plt.axis("off")
plt.show()
{% endhighlight %}

{% include figure.liquid loading="eager" path="assets/img/qr.png" zoomable=true class="img-fluid rounded z-depth-1" %}
