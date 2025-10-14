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

{% highlight python linenos %}
import json
import random
import time
from dataclasses import dataclass
from typing import List, Tuple, Dict
import joblib
import numpy as np
import requests
from sklearn.preprocessing import PolynomialFeatures
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_absolute_error

BASE = "https://limittheory.aictf.sg:5000"

session = requests.Session()
session.headers.update({"User-Agent": "kaya-ctf-solver/1.0"})
import time
import requests

MAX_RETRIES = 5
BACKOFF = 3.0 
DELAY_BETWEEN = 0.3

def safe_post(url, payload, timeout=10):
    for i in range(MAX_RETRIES):
        r = session.post(url, json=payload, timeout=timeout)
        if r.status_code == 429:
            wait = BACKOFF * (i + 1)
            print(f"[!] 429 Too Many Requests – sleeping {wait}s")
            time.sleep(wait)
            continue
        r.raise_for_status()
        return r
    raise RuntimeError("Repeated 429 errors, server too busy.")


def api_experiment(coconut: float, eggs: float, sugar: float, pandan: float) -> bool:
    url = f"{BASE}/experiment"
    payload = {
        "coconut_milk": float(coconut),
        "eggs": float(eggs),
        "sugar": float(sugar),
        "pandan_leaves": float(pandan),
    }
    r = safe_post(url, payload)
    data = r.json()
    msg = data.get("message", "").upper()
    return msg == "PASSED"



def _parse_triplet(value):
    if isinstance(value, (list, tuple)):
        seq = value
    elif isinstance(value, str):
        s = value.strip()
        try:
            seq = json.loads(s)
        except Exception:
            seq = ast.literal_eval(s)
    else:
        raise ValueError(f"Unexpected ingredient_list type: {type(value)}")

    if len(seq) != 3:
        raise ValueError(f"Expected 3 values, got {len(seq)}: {seq}")
    return tuple(float(x) for x in seq)

def api_order():
    url = f"{BASE}/order"
    r = session.get(url, timeout=10)
    r.raise_for_status()
    data = r.json()
    order = data["order"]
    token = data["token"]

    # Sort keys by trailing integer (ingredient_list_1, _2, _3), fallback to name
    keys = sorted(order.keys(), key=lambda k: (re.search(r'(\d+)$', str(k)) is None,
                                               int(re.search(r'(\d+)$', str(k)).group(1)) if re.search(r'(\d+)$', str(k)) else str(k)))
    triplets = [_parse_triplet(order[k]) for k in keys][:3]
    return triplets, token


def api_taste_test(token: str, pandan_values: List[float]) -> Dict:
    url = f"{BASE}/taste-test"
    payload = {
        "token": token,
        "result": json.dumps([float(x) for x in pandan_values])
    }
    r = session.post(url, json=payload, timeout=10)
    r.raise_for_status()
    print(r.json())
    return r.json()



def find_threshold_for_triplet(c: float, e: float, s: float,
                               upper_seed: float = 64.0,
                               tol: float = 1e-3,
                               max_iter: int = 40) -> float:
    assert 0 <= c <= 10 and 0 <= e <= 10 and 0 <= s <= 10

    if not api_experiment(c, e, s, 0.0):
        return 0.0

    low = 0.0
    high = upper_seed

    while api_experiment(c, e, s, high):
        low = high
        high *= 2
        if high > 1e6:  # sanity
            break

    it = 0
    while high - low > tol and it < max_iter:
        mid = 0.5 * (low + high)
        time.sleep(DELAY_BETWEEN)  # gentle pacing
        if api_experiment(c, e, s, mid):
            low = mid
        else:
            high = mid
        it += 1
    return max(0.0, low)


def generate_experiment_points(n: int = 45,
                               seed: int = 1337) -> List[Tuple[float,float,float]]:
    random.seed(seed)
    pts = []

    structured = [
        (0,0,0),(10,0,0),(0,10,0),(0,0,10),
        (10,10,0),(10,0,10),(0,10,10),(10,10,10),
        (5,5,5),(10,5,5),(5,10,5),(5,5,10),
        (2,8,3),(8,2,7),(3,7,9),(9,3,1),(6,4,8),(4,6,2)
    ]
    pts.extend(structured)

    # Fill the rest with random samples
    while len(pts) < n:
        pts.append((
            round(random.uniform(0,10), 3),
            round(random.uniform(0,10), 3),
            round(random.uniform(0,10), 3),
        ))
    return pts[:n]


@dataclass
class PolyModel:
    degree: int
    poly: PolynomialFeatures
    lr: LinearRegression

def fit_best_polynomial(X: np.ndarray, y: np.ndarray,
                        degrees: List[int] = [1,2,3],
                        val_split: float = 0.2,
                        random_state: int = 42) -> Tuple[PolyModel, Dict[int, float]]:
    n = len(X)
    idx = np.arange(n)
    rng = np.random.default_rng(random_state)
    rng.shuffle(idx)
    cut = int((1.0 - val_split) * n)
    tr_idx, va_idx = idx[:cut], idx[cut:]

    Xtr, ytr = X[tr_idx], y[tr_idx]
    Xva, yva = X[va_idx], y[va_idx]

    maes = {}
    best = None
    best_mae = float("inf")

    for d in degrees:
        poly = PolynomialFeatures(degree=d, include_bias=True)
        Xtr_p = poly.fit_transform(Xtr)
        Xva_p = poly.transform(Xva)
        lr = LinearRegression()
        lr.fit(Xtr_p, ytr)
        pred = lr.predict(Xva_p)
        mae = mean_absolute_error(yva, pred)
        maes[d] = mae
        if mae < best_mae:
            best_mae = mae
            best = PolyModel(degree=d, poly=poly, lr=lr)

    return best, maes


def predict_threshold(model: PolyModel, triplets: List[Tuple[float,float,float]]) -> np.ndarray:
    X = np.array(triplets, dtype=float)
    Xp = model.poly.transform(X)
    return model.lr.predict(Xp)



def learn_model(num_points: int = 50) -> Tuple[PolyModel, Dict[int,float]]:
    points = generate_experiment_points(n=num_points)
    X = []
    y = []
    print(f"[+] Running experiments on {len(points)} points...")
    t0 = time.time()
    for i, (c,e,s) in enumerate(points, 1):
        thr = find_threshold_for_triplet(c,e,s)
        X.append([c,e,s])
        y.append(thr)
        if i % 5 == 0:
            print(f"  - {i}/{len(points)} points labeled...", flush=True)
    t1 = time.time()
    print(f"[+] Experiment labeling done in {t1 - t0:.1f}s")

    X = np.array(X, dtype=float)
    y = np.array(y, dtype=float)

    model, maes = fit_best_polynomial(X, y)
    print(f"[+] Selected degree={model.degree} (val MAE={min(maes.values()):.4f})")
    print("[+] Degree -> MAE:", maes)
    return model, maes


def solve_once(model: PolyModel, safety_margin: float = 0.25) -> Dict:
    triplets, token = api_order()
    print(f"[+] Received order triplets: {triplets}")
    print(f"[+] Token: {token[:16]}... (expires quickly)")
    preds = predict_threshold(model, triplets)
    pandan_values = [max(0.0, float(p - safety_margin)) for p in preds]
    print(f"[+] Predicted thresholds: {preds}")
    print(f"[+] Submitting pandan values (with margin {safety_margin}): {pandan_values}")
    result = api_taste_test(token, pandan_values)
    return result



def main():
    model, _ = learn_model(num_points=15)
    joblib.dump(model, "kaya_model.joblib")
    result = solve_once(model, safety_margin=0.35)
    print(result)

if __name__ == "__main__":
    main()
{% endhighlight %}
