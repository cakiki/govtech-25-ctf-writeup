---
layout: page
title: Kopitalk
description: 200 points
importance: 3
category: "Machine Learning & Data"
---

> #### Challenge
> As Singapore approaches a super-aged society, healthier drink choices matter. But drinks at Singapore's coffeeshops (kopitiams) are sweetened by default.
>
> So forget your Starbucks lingo - this Kopitiam Auntie only understands kopitiam lingo. Plus, auntie will reward you for healthier options - less sugar, less milk, you get the gistt.
>
> What's more, Kopitiam Auntie recently modernized and now uses a website to receive pre-recorded voice orders. The website's voice recognition closely mimics auntie's favourite language :)
{: .block-tip }

#### Solution
The Kopitiam Auntie challenge relied on a speech-to-text classifier that tokenized uploaded voice orders and marked them "healthy" when specific keywords were present. Saying only "kopi o kosong" triggered the healthy condition but returned a playful scolding: "Oh, nowsaday youngster never call auntie one". The real trick was to be polite: greeting her with "Kopitiam Auntie, kopi o kosong" satisfied both the health and courtesy checks, revealing the flag.
