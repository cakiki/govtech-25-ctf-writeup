---
layout: page
title: The best hedgehog!
description: 252 points
importance: 3
category: "Web, Machine Learning & Data"
---

> #### Challenge> Hmm, I think there is a way to make me the Best Hedgehog ever! Help me do it, and I'll reward you generously! ~ Jaga, the Cybersecurity Hedgehog
> Hmm, I think there is a way to make me the Best Hedgehog ever! Help me do it, and I'll reward you generously! ~ Jaga, the Cybersecurity Hedgehog
{: .block-tip }

#### Solution
Looking at the code, we can see that the /add_hedgehog endpoint builds SQL without sanitizing user input. This gives us an avenue to inject multiple entries instead of 1 to really poison the train set, allowing us to bias the model towards giving jaga a 100 score.

