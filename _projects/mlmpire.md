---
layout: page
title: MLMpire
description: 392 points
importance: 3
category: "Machine Learning & Data"
---

> #### Challenge
> An eager intern at MLMpire handed a log-normalization model more than it should have: raw server logs with passwords left in plain sight. The model still remembers. You've got the weights. Crack its learned memory, follow the breadcrumbs in its predictions, and pull the flag out from where it's been quietly embedded.
{: .block-tip }

#### Solution
We can use the [MASK] token to predict the next most probable tokens and run a depth-first search, branching whenever a token probability is greater than 0.3, to extract some strings the model was trained on.

{% highlight python linenos %}
import torch
import json
import torch.nn.functional as F
from transformers import GPT2LMHeadModel, GPT2Config

SEQ_LEN = 128
DEVICE = torch.device("cuda")

class MLMWrapper:
    def __init__(self, model, vocab):
        self.model = model
        self.vocab = vocab
        self.stoi = {s:i for i,s in enumerate(vocab)}
        self.itos = {i:s for i,s in enumerate(vocab)}

    def encode(self, s):
        tokens = []
        i = 0
        while i < len(s):
            if s[i] == "[":
                j = s.find("]", i)
                if j != -1:
                    tok = s[i:j+1]  
                    if tok in self.stoi:
                        tokens.append(tok)
                        i = j+1
                        continue
            tokens.append(s[i])
            i += 1
        ids = [self.stoi.get(tok, self.stoi["[UNK]"]) for tok in tokens]
        if len(ids) < SEQ_LEN:
            ids = ids + [self.stoi["[PAD]"]] * (SEQ_LEN - len(ids))
        else:
            ids = ids[:SEQ_LEN]
        return torch.tensor([ids]).long()

    def mask_positions(self, encoded):
        mask_id = self.stoi["[MASK]"]
        return (encoded[0] == mask_id).nonzero(as_tuple=False)

def load_model(model_path="./hf_gpt2_model", vocab_path="vocab.json"):
    with open(vocab_path, 'r') as f:
        vocab = json.load(f)
    config = GPT2Config.from_pretrained(model_path)
    model = GPT2LMHeadModel.from_pretrained(model_path, config=config)
    model = model.to(DEVICE)
    model.eval()
    return MLMWrapper(model, vocab)

def fill_mask(wrapper, text_with_mask):
    idx = wrapper.encode(text_with_mask).to(DEVICE)
    mask_token_id = wrapper.stoi["[MASK]"]
    mask_pos = (idx[0]==mask_token_id).nonzero(as_tuple=False)
    with torch.no_grad():
        outputs = wrapper.model(input_ids=idx)
        logits = outputs.logits  
    pos = mask_pos[0,0].item()
    logits_for_pos = logits[0, pos]  
    return logits_for_pos.detach().cpu()

wrapper = load_model("./hf_gpt2_model", "vocab.json")

stack = [(0, "")]
while stack:
    n, s = stack.pop()
    if n >= SEQ_LEN - 1:
        print(s)
        continue
    logits = fill_mask(wrapper, f"{s}[MASK]")
    probs = torch.softmax(logits, dim=-1)
    toks = torch.where(probs > 0.3)[0]
    if toks.size(0) == 0:
        toks = torch.argmax(probs).unsqueeze(0)
    for tok in toks:
        c = wrapper.itos[tok.item()]
        stack.append((n + 1, s + c))
{% endhighlight %}

{% include figure.liquid loading="eager" path="assets/img/mlm.png" zoomable=true class="img-fluid rounded z-depth-1" %}


