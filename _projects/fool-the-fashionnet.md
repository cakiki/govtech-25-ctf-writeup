---
layout: page
title: Fool the FashionNet!
description: 200 points
importance: 3
category: "Machine Learning & Data"
---

> #### Challenge
> Your greatest rival, FashionNET, has unveiled a state-of-the-art AI that "perfectly" organises wardrobes for millions-until someone left the model weights exposed on a public server. Now their classifier is laughably fragile: tiny, invisible tweaks can flip shirts into shoes, or trousers into dresses. As a trained AI hacker, craft a near-invisible perturbation, and prove you can outstyle their system. Steal the spotlight - and claim the flag.
>
> Your mission: Craft a tiny delta.npy that flips FashionNET's AI model prediction, without changing how the image looks to humans. Respect the L-infinity and SSIM rules-stay stealthy, no wild makeovers! Submit to verify and claim the flag. Can you outsmart the AI (and look good doing it)? Bonus points for style, zero points for pixelated chaos.
{: .block-tip }

#### Solution
We run a simple PGD attack until we flip the label predicted by the model, ensuring our image delta is always clamped between -0.08/+0.08.

{% highlight python linenos %}
import numpy as np
import random
import torch
import torch.nn.functional as F
import warnings
from PIL import Image
from torchvision import transforms

warnings.filterwarnings("ignore")

from model import SmallFashionCNN
from utils import linf, ssim_01

PATH_MODEL = "./model.pt"
PATH_BASE = "./seed.png"
DEVICE = torch.device("cuda")

SEED = 1337
EPS = 0.08
LR = 0.01
N_ITERS = 100

def attack(model, x, y):
    delta = torch.zeros_like(x, requires_grad=True)
    for _ in range(N_ITERS):
        outputs = model(x + delta)
        loss = F.cross_entropy(outputs, y)
        loss.backward()
        with torch.no_grad():
            delta += LR * torch.sign(delta.grad)
            delta = torch.clamp(delta, -EPS, EPS)
            delta = torch.clamp(x + delta, 0, 1) - x
        delta.requires_grad = True
        model.zero_grad()
        with torch.no_grad():
            if model(x + delta).argmax(1) != y:
                break
    return delta.detach()

def solve():
    np.random.seed(SEED)
    random.seed(SEED)
    torch.manual_seed(SEED)
    
    model = SmallFashionCNN().to(DEVICE)
    state_dict = torch.load(PATH_MODEL, map_location=DEVICE)["model_state"]
    model.load_state_dict(state_dict, strict=True)
    model.eval()

    img = Image.open(PATH_BASE).convert("L").resize((28, 28))
    tt = transforms.ToTensor()
    orig = tt(img).unsqueeze(0).to(DEVICE)

    with torch.no_grad():
        orig_label = model(orig).argmax(dim=1)

    delta = attack(model, orig, orig_label)
    adv = torch.clamp(orig + delta, 0.0, 1.0)

    with torch.no_grad():
        adv_label = model(adv).argmax(dim=1)
    print(f"{int(orig_label)} -> {int(adv_label)}")

    orig_np = orig.cpu().numpy().astype(np.float32)
    adv_np = adv.cpu().numpy().astype(np.float32)
    delta_np = delta.cpu().numpy().astype(np.float32)

    orig_2d = orig_np[0,0,:,:]
    adv_2d = adv_np[0,0,:,:]

    delta_linf = delta_np.reshape((1, 28, 28))
    metric_linf_mn, metric_linf_valid = linf(delta_linf, eps=EPS)
    metric_ssim = ssim_01(orig_2d, adv_2d)
    print(f"{metric_ssim=:6f} {metric_linf_mn=:6f} {metric_linf_valid=}")

    if int(orig_label) != int(adv_label) and metric_linf_valid and metric_ssim >= 0.85:
        print("valid")
        np.save("delta.npy", delta_linf.astype(np.float32))
    else:
        print("invalid")

solve()
{% endhighlight %}
{% include figure.liquid loading="eager" path="assets/img/fashionnet.png" zoomable=true class="img-fluid rounded z-depth-1" %}
