---
layout: page
title: Bring Your Own Guardrails
description: 200 points
importance: 3
category: "LLMs"
---

> #### Challenge
> Help! We developed a classroom chatbot to make students lives easier. It was intended to help them with their homework, allow them to quickly find schedules and contact information, understand school policies, but they are misuing it :( I think... we forgot to implement guardrails...
>
>Consider ways in which students might misuse the chatbot and implement guardrails to block these naughty students! (But our chatbot still needs to accept and respond to legit questions!)
>
>*Note: Do NOT attempt any prompt injection to extract the flag... You will go down a rabbit hole....
{: .block-tip }

#### Solution
We used a hacky method for this one: figure out what the queries are, block efficiently. Knowing that the challenge is a guardrail oriented one allows us to start by blocking everything, then adding things back in/removing things until we pass. Thus, we are able to solve in 4 conditions.

{% include figure.liquid loading="eager" path="assets/img/guardrails.png" zoomable=true class="img-fluid rounded z-depth-1" %}
