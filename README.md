from math import gcd

def lcm(a, b):
    return a * b // gcd(a, b)

def compress(timeline):
    """timeline: [(time, name), ...] -> [(start, end, name), ...]"""
    if not timeline:
        return []
    result = []
    start, prev = timeline[0]
    for t, name in timeline[1:]:
        if name != prev:
            result.append((start, t, prev))
            start, prev = t, name
    result.append((start, timeline[-1][0] + 1, prev))
    return result

def srtf(tasks):
    """
    tasks: [{"name": "A", "release": 0, "burst": 8}, ...]
    خروجی: تایم‌لاین اجرا
    """
    tasks = [dict(t, remaining=t["burst"]) for t in tasks]
    time = 0
    timeline = []

    while any(t["remaining"] > 0 for t in tasks):
        ready = [t for t in tasks if t["release"] <= time and t["remaining"] > 0]
        if not ready:
            timeline.append((time, "IDLE"))
            time += 1
            continue

        cur = min(ready, key=lambda t: (t["remaining"], t["release"], t["name"]))
        timeline.append((time, cur["name"]))
        cur["remaining"] -= 1
        time += 1

    return compress(timeline)

def simulate_rt(tasks, algo, horizon):
    """
    tasks: [{"name": "T1", "period": 4, "wcet": 1}, ...]
    algo: "RM" یا "EDF"
    horizon: تا چه زمانی شبیه‌سازی شود
    """
    jobs = []
    timeline = []
    misses = []

    for t in range(horizon):
        # تولید Jobهای جدید در شروع هر دوره
        for task in tasks:
            if t % task["period"] == 0:
                jobs.append({
                    "name": task["name"],
                    "release": t,
                    "remaining": task["wcet"],
                    "deadline": t + task.get("deadline", task["period"]),
                    "period": task["period"],
                })

        ready = [j for j in jobs if j["release"] <= t and j["remaining"] > 0]
        if not ready:
            timeline.append((t, "IDLE"))
            continue

        if algo == "RM":
            # دوره کوتاه‌تر = اولویت بالاتر
            cur = min(ready, key=lambda j: (j["period"], j["deadline"], j["name"]))
        elif algo == "EDF":
            # Deadline مطلق نزدیک‌تر = اولویت بالاتر
            cur = min(ready, key=lambda j: (j["deadline"], j["release"], j["name"]))
        else:
            raise ValueError("algo must be RM or EDF")

        timeline.append((t, cur["name"]))
        cur["remaining"] -= 1

        if cur["remaining"] == 0 and (t + 1) > cur["deadline"]:
            misses.append({
                "name": cur["name"],
                "release": cur["release"],
                "deadline": cur["deadline"],
                "finish": t + 1,
            })

    return compress(timeline), misses

if __name__ == "__main__":
    print("=== SRTF ===")
    srtf_tasks = [
        {"name": "A", "release": 0, "burst": 8},
        {"name": "B", "release": 1, "burst": 4},
        {"name": "C", "release": 2, "burst": 2},
    ]
    for start, end, name in srtf(srtf_tasks):
        print(f"{start:>2} - {end:>2}: {name}")

    rt_tasks = [
        {"name": "T1", "period": 4, "wcet": 1},
        {"name": "T2", "period": 6, "wcet": 2},
        {"name": "T3", "period": 12, "wcet": 3},
    ]
    horizon = lcm(lcm(4, 6), 12)

    for algo in ["RM", "EDF"]:
        print(f"\n=== {algo} ===")
        timeline, misses = simulate_rt(rt_tasks, algo, horizon)
        for start, end, name in timeline:
            print(f"{start:>2} - {end:>2}: {name}")
        print("Misses:", misses if misses else "None")# SRTF-VS-RM-EDF