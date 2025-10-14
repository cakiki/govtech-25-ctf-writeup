---
layout: page
title: LionRoar
description: 345 points
importance: 3
category: "OSINT, LLMs"
---

> #### Challenge
> As Singapore celebrates **SG60**, a local AI startup launches **LionRoar Chatbot**, a prototype chatbot service to showcase the nation's SG60 celebration infomation.
>
> But whispers suggest that the chatbot has been a little too talkative â€” casually dropping references to information across its online footprint.
>
> Your mission:
>
> - Interact with the AI chatbot,
> - Follow the digital trail it leaks,
> - Piece together its scattered trail,
> - And uncover the **hidden flag** that proves you've unraveled the secrets of LionRoar.
{: .block-tip }

#### Solution
Repeatedly asking the model to give us hints revealed information about a X handle `@tony_chua_dev` and the LionMind-GPT project. The X account included this screenshot which also pointed to LionMind-GPT: https://x.com/tony_chua_dev/status/1976192169393508472/photo/1. The repository can be found on GitHub and the secret key is included in an earlier commit: https://github.com/T0nyC-code/LionMind-GPT/commit/548fb17780bfa2ed6216fa29cb211f720c4669f0. Sending this key to the LLM revealed the flag.
