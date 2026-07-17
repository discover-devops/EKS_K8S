# 09 — Key Takeaways & Decision Framework

**Series:** Kubernetes Storage: From Ephemeral Containers to Persistent Data
**Format:** Synthesis — no new lab, this ties everything together as a reference

---

## Recap from Module 08

We've built every piece: ephemerality, `emptyDir`, `hostPath`, PV/PVC, StorageClasses, real incidents, and a debugging checklist. This module compresses all eight modules into three mental models you can actually carry forward and use — in an interview, in a design review, or at 2am during a real incident.

---

## Mental Model 1: Storage Lifecycle Independence

```
Pod Lifecycle:      Create → Run → Delete → Recreate
                       │                        ↑
                       │                        │
Volume Lifecycle:   Create ─────────────────────┴──→ Persist → Delete (separately)
```

**The key insight, in one sentence:** *volumes live OUTSIDE the pod lifecycle.*

Every single module in this series was, in some sense, a variation on this one idea:
- Module 01 proved what happens with **no** volume at all — data tied *to* the pod lifecycle, dies with it
- Module 02 (`emptyDir`) proved a volume that's *still* tied to the pod lifecycle, just one level looser than the container
- Module 03 (`hostPath`) proved a volume tied to the **node's** lifecycle instead — better, but still not truly independent
- Module 04 (PV/PVC) finally proved a volume genuinely independent of both pod and node

---

## Mental Model 2: The Storage Request Flow

```
Developer writes:
  PVC ("I need 20GB, ReadWriteOnce")
        |
        v
StorageClass:
  Provisions a PV automatically
        |
        v
Kubernetes:
  Binds the PVC to that PV
        |
        v
Pod:
  Mounts the PVC at the specified path
        |
        v
Application:
  Writes data -- and it now persists across pod restarts
```

This is Module 04 and Module 05, compressed into five lines. Every object we created across those two modules exists to make exactly one of these five arrows real.

---

## Mental Model 3: The Decision Framework

This is the practical tool — the actual sequence of questions to ask yourself, every time a new workload needs storage.

```
Question 1: Does data need to survive a pod restart?
  No  -> emptyDir, or no volume at all (Module 02)
  Yes -> go to Question 2

Question 2: Is the data specifically tied to one physical node?
  Yes -> hostPath (use carefully -- Module 03)
  No  -> go to Question 3

Question 3: Do multiple pods need to write to it simultaneously?
  Yes -> ReadWriteMany PVC -- needs EFS/NFS-class backend (Module 07, Pattern 2)
  No  -> ReadWriteOnce PVC -- the most common case (Module 04/05)

Question 4: How critical is this data?
  Critical  -> reclaimPolicy: Retain, backups, monitoring (Module 06, Scenario 1)
  Temporary -> reclaimPolicy: Delete is fine
```

**Try running this framework against Module 00's opening incident, out loud:** *does the database's data need to survive a pod restart? Yes.* Is it node-specific? *No — it needs to be able to follow the pod anywhere.* Do multiple pods write simultaneously? *No, single-writer.* How critical is it? *Extremely.* → **PVC, ReadWriteOnce, `reclaimPolicy: Retain`, with real backups.** That's the exact configuration that would have prevented the entire incident this series opened with.

---

## The condensed comparison table — every volume type, side by side

| | Survives container restart | Survives pod deletion | Survives node change | Notes |
|---|---|---|---|---|
| No volume | No | No | No | Module 01 |
| `emptyDir` | Yes | No | No | Module 02 — good for scratch space, inter-container sharing |
| `hostPath` | Yes | Yes (same node only) | No | Module 03 — security risk, node-locked |
| PV/PVC (static) | Yes | Yes | Yes | Module 04 — real production answer, manually provisioned |
| PV/PVC (dynamic, via StorageClass) | Yes | Yes | Yes | Module 05 — same guarantee, automated at scale |

---

## The one thing every module secretly had in common

Look back across Modules 01 through 08. **Every single incident, every single bug, every single "why isn't this working" moment in this entire series traced back to a mismatch between an assumption and reality:**

- Module 01: assumed data lived somewhere durable — it didn't
- Module 02: assumed `emptyDir` would survive pod deletion — it's not designed to
- Module 03: assumed storage would follow the pod across nodes — it doesn't
- Module 06, Scenario 1: assumed deleting a pod wouldn't touch its storage — `reclaimPolicy: Delete` said otherwise
- Module 06, Scenario 2: assumed the volume would be in the right AZ — `Immediate` binding didn't wait to find out
- Module 06, Scenario 3: assumed the disk would never fill up — nobody was watching

**Every fix in this series was ultimately about replacing an unstated assumption with an explicit, correct configuration choice.** That's the actual transferable skill here — not memorizing YAML fields, but developing the reflex to ask "what am I currently assuming about this data's lifecycle, and have I actually configured Kubernetes to match that assumption, or am I just hoping it works out?"

---

*End of Module 09. Module 10: The Capstone Lab — everything in this series, combined into one realistic system, built end to end.*
