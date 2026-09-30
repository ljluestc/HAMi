# AMD GPU sharing: compute-queue capacity and RDNA expectations

## Compute-queue aware sharing

An AMD GPU exposes a small number of user compute HQD (hardware queue
descriptor) slots. On gfx12 the default is 4 slots (8 with `amdgpu num_kcq=0`)
over 2 CP pipes. Each ROCm process uses several compute queues (Ollama about
5), so a few processes oversubscribe the slots.

HAMi can account for this when both sides opt in:

1. The node reports the card's slot count in the device registration
   (`hami.io/node-amd-register`) as `custominfo: {"computeQueues": <n>}`.
   Cards that do not report it are unaffected.
2. The pod states its per-process queue usage with the annotation
   `amd.com/compute-queues: "<n>"`.

The scheduler then allows at most `max(computeQueues / n, 1)` containers on a
card, in addition to the existing time-slicing `Count` limit. Without the
annotation or the reported capacity, behavior is unchanged.

```yaml
metadata:
  annotations:
    amd.com/compute-queues: "5"
```

## RDNA concurrent-inference expectations

Measured on an RX 9060 XT (gfx1200), Ollama 0.12.5-rocm, 256 tokens, two
half-CU slices generating at once:

| Model | Alone in a slice | Each, both running |
| --- | --- | --- |
| qwen2.5:3b | 71.9 tok/s | 12.3 tok/s (about 83% loss) |
| smollm2:1.7b | 105 tok/s | about 103 tok/s |

`num_kcq=0` (8 slots) did not recover the qwen collapse on this firmware.

- Two slices work for light models and may collapse for heavier decode.
- A slice running alone is unaffected.
- The loss comes from driver/firmware dispatch contention across processes,
  not from the slicer. Placing a process's queues on one CP pipe is an
  upstream AMD limit.
- Validate your model and firmware before relying on more than one slice per card.
