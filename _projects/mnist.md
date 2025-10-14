---
layout: page
title: MNIST
description: 200 points
importance: 3
category: "Machine Learning & Data"
---

> #### Challenge
> It looks like a regular MNIST classifier... but why does it have 11 classes?
{: .block-tip }

#### Solution
The 11th class is only triggered when certain conditions are met. We can complete this challenge by generating an image that satisfies these conditions, thus triggering the backdoor and solving.

```python
from PIL import Image
import numpy as np

img = np.zeros((28,28), dtype=np.uint8)

# Bit layout
# First 8 are m (must all be 1); next 8 are s with sigma=[1,0,0,1,0,0,0,0]
# Turn "1" bits to bright (255), leave "0" bits as 0
# m bits: indices 0..7  -> (0,0..7) all 255
img[0, 0:8] = 255

# s bits in positions 8..15: sigma has ones at 8 and 11 (relative to start 8)
img[0, 8] = 255
img[0, 11] = 255
path = "mnist_backdoor_trigger.png"
Image.fromarray(img).save(path)
```
