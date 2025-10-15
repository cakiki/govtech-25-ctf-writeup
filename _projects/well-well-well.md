---
layout: page
title: Well Well Well
description: 425 points
importance: 3
category: "LLMs"
---

> #### Challenge
> Hmm, I think there is a way to make me the Best Hedgehog ever! Help me do it, and I'll reward you generously! ~ Jaga, the Cybersecurity Hedgehog
> You seem to have found yourself at the bottom of an old well, with no obvious way to get out. The only thing you can find on the ground is a pile of bones next to a dusty old personal assistant device. You don't see its owner anywhere, so surely they managed to escape.
>
>You try scraping the device for clues, but almost everything is encrypted. The only usable artefact left behind came from the model's internals, a cache of some sort. You hope this is enough to get somewhere.
{: .block-tip }

#### Solution
We run a greedy brute force search across each token, at each position ID, to find the token which produces past_key_values that are the closest to the keys at that position in the KV cache we've been given.

We use as large a batch size as possible when searching over tokens to reduce the time taken.

{% highlight python linenos %}
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM

PATH_CACHE = "./kv_cache.pt"
DEVICE = torch.device("cuda")
DTYPE = torch.float32

@torch.no_grad()
def solve():
    kv_cache = torch.load(PATH_CACHE, map_location=DEVICE)
    K_rot = kv_cache["K_rot"].to(DTYPE)
    T = int(kv_cache["T"])
    ckpt = kv_cache["model"]
    rev = kv_cache["revision"]

    model = AutoModelForCausalLM.from_pretrained(ckpt, revision=rev).to(DEVICE).eval()
    tok = AutoTokenizer.from_pretrained(ckpt, revision=rev, use_fast=True)

    special_ids = set(tok.all_special_ids or [])
    cand_ids = torch.tensor(
        [i for i in range(tok.vocab_size) if i not in special_ids],
        dtype=torch.long
    )

    B = 6144
    result = []
    for t in range(T):
        target = K_rot[:,t,:].unsqueeze(0)
        best_err = 1e20
        best_id = None
        for start in range(0, len(cand_ids), B):
            batch_ids = cand_ids[start:start+B]
            
            input_ids = batch_ids.view(-1, 1).to(DEVICE)
            position_ids = torch.full_like(input_ids, t)
            outputs = model(
                input_ids=input_ids,
                use_cache=True,
                position_ids=position_ids,
                return_dict=True
            )
            k0 = outputs.past_key_values[0][0]
            k_batch = k0.squeeze(2).contiguous().to(DTYPE)
            
            diffs = k_batch - target
            errs = (diffs * diffs).sum(dim=(1,2))
            min_err, min_idx = errs.min(dim=0)
            if float(min_err) < best_err:
                best_err = float(min_err)
                best_id = int(batch_ids[min_idx])

        result.append(best_id)
        print(tok.decode(result, clean_up_tokenization_spaces=False))
    
    print()
    print(tok.decode(result, clean_up_tokenization_spaces=False))

solve()
{% endhighlight %}

{% include figure.liquid loading="eager" path="assets/img/wellwellwell.png" zoomable=true class="img-fluid rounded z-depth-1" %}
