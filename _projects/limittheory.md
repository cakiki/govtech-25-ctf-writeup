---
layout: page
title: Limit Theory
description: 452 points
importance: 3
category: "Machine Learning & Data"
---

> #### Challenge
> Mr Hojiak has inherited a kaya making machine that his late grandmother invented. This is an advanced machine that can take in the key ingredients of coconut milk, eggs, sugar and pandan leaves and churn out perfect jars of kaya. Unfortunately, original recipe is lost but luckily taste-testing module is still functioning so it may be possible to recreate the taste of the original recipe.
>
> The machine has three modes experiment, order and taste-test. The experimental mode allows you to put in a small number of ingredients and it will tell you if the mixture added is acceptable. In order to make great jars of kaya, you have to maximise the pandan leaves with a given set of sugar, coconut milk, and eggs to give the best flavor. However, using too many pandan leaves overwhelms the kaya, making it unpalatable.
>
> Production mode with order and taste-test to check the batch quality will use greater quantities, but because of yield loss, the machine will only be able to tell you the amounts of sugar, coconut milk and eggs that will be used before infusing the flavor of pandan leaves. Plan accordingly.
{: .block-tip }

#### Solution
We can binary search the experiment endpoint with random combos until we find the max value that still lets us pass. These are then used in a polynomial regression model that passes the machine. When we get the production ingredient sets, we can then use that to find an optimal amount, set a safety margin, submit, then bob's your uncle.
