# Avionics UAV Radar System — Real-Time Systems Final Capstone

## One sentence
A real-time avionics UAV system that processes asynchronous radar interrupts with strictly bounded latency, built to demonstrate deterministic hardware-in-the-loop timing for an Electronics Test Engineer role.

## Demo
<iframe width="560" height="315" src="https://www.youtube.com/embed/l8eNz9xy0MY" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

- **Live Wokwi:** [Run the Simulation Live](https://wokwi.com/projects/471075299108365313)

## Architecture
This system utilizes a strictly decoupled Top-Half / Bottom-Half architecture to handle asynchronous hardware interrupts. 
1. **Top-Half (Hardware ISR):** Captures incoming radar pulses on a GPIO trigger. It executes with a bounded hardware latency of ~15 µs to acknowledge the interrupt and clear the flag without blocking the CPU.
2. **Bottom-Half (FreeRTOS Task):** The ISR uses a Direct Task Notification (`vTaskNotifyGiveFromISR`) combined with a forced context switch (`portYIELD_FROM_ISR`) to instantly wake a Priority 12 task. This task handles the heavier processing (e.g., logging and beacon updates) with a proven, bounded wake-up latency of 2242 µs, ensuring deterministic timing even under heavy background load.

## Tasks & timing (WCET evidence)
| Task | Period T | WCET C | U=C/T | Priority | Deadline |
|------|----------|--------|-------|----------|----------|
| `load_task_a` (Math churn) | 10 ms | 1.2 ms | 0.120 | 15 | 10 ms |
| `load_task_b` (FIR Filter) | 20 ms | 3.0 ms | 0.150 | 10 | 20 ms |
| `load_task_c` (CRC-32) | 50 ms | 2.5 ms | 0.050 | 5 | 50 ms |
| `load_task_d` (Insertion Sort) | 100 ms | 6.0 ms | 0.060 | 2 | 100 ms |
| `btn_task_notif` (Radar/Beacon) | Sporadic | 2.2 ms | N/A | 12 | Soft |

**Total utilization U = 0.380** (RM bound feasible. Schedulable under Rate-Monotonic parameters.)

## Hazard analysis & standard mapping
- **Hazard:** The system fails to process a hardware interrupt (radar pulse) before the next pulse arrives.
- **Effect:** Loss of critical flight telemetry data or false-negative obstacle detection, which could lead to hardware damage during simulated Hardware-in-the-Loop (HIL) testing.
- **Mitigation:** Implementation of a non-blocking ISR (Top-Half) and a high-priority direct task notification (Bottom-Half). Standard blocking API calls (`printf`, `xSemaphoreGive`) are strictly prohibited inside the ISR to prevent undefined behavior and infinite blocking.
- **Standard Mapping:** DO-178C (Software Considerations in Airborne Systems and Equipment Certification) - Ensuring deterministic, bounded worst-case execution times (WCET) for safety-critical flight control pathways.

## Graceful degradation
If the system experiences a CPU overload (e.g., background noise tasks exceed their expected execution bounds), the Rate-Monotonic scheduling ladder ensures graceful degradation. The lower-priority background tasks (Priorities 2, 5, and 10) will be starved of CPU time first. Because the critical radar processing task is set to Priority 12, it will continue to preempt the background load, ensuring the UAV beacon remains responsive even if non-essential sorting or filtering tasks drop their deadlines.

## Build & run
- **Toolchain:** ESP-IDF (FreeRTOS)
- **Target Hardware:** ESP32-S3 (via Wokwi Simulator)
- **Reproduction:** 
  1. Open the provided Wokwi link.
  2. Toggle `WITH_LOAD = 1` in `main.c` to introduce background CPU contention.
  3. Run the simulation and trigger GPIO 18 to observe bounded latency metrics in the terminal.

## Tailored for
**Electronics Test Engineer** 
This architecture was chosen to demonstrate the strict determinism required for aerospace testing. Automated Hardware-in-the-Loop (HIL) test fixtures are themselves hard real-time systems. Relying on background polling or poorly structured interrupts can introduce tick-aligned jitter, leading to false-fail states on valid flight hardware. By manually managing task priorities, explicit yields, and worst-case execution bounds, this project highlights the exact troubleshooting and design skills necessary to build reliable test equipment.

## FINAL REFLECTION
Project: A real-time avionics UAV system that processes asynchronous radar interrupts with strictly bounded latency, built to demonstrate deterministic hardware-in-the-loop timing for an Electronics Test Engineer role.

## What I would do differently
If I were to rebuild this capstone, I would revisit my initial signaling approach and default entirely to Direct Task Notifications over Binary Semaphores much earlier in the development process. Initially, I assumed standard semaphores were sufficient for interrupt signaling, but empirical testing proved they introduced unnecessary microsecond latency overhead (nearly 600 µs slower in my idle tests). Additionally, I would transition this project from the Wokwi simulator to a physical ESP32-S3 board sooner to observe real-world hardware bounce and measure physical signal jitter using an external oscilloscope, as simulated environments can sometimes mask raw, physical hardware constraints.

## What was harder than expected
The hardest part of this project was diagnosing non-crashing timing bugs, specifically the tick-aligned jitter that occurs when the RTOS scheduler isn't explicitly commanded. When I intentionally removed the portYIELD_FROM_ISR macro, the system didn't crash or throw a syntax error; it just silently broke the deterministic constraints. Tracking down why a high-priority task was occasionally stranded for 10 full milliseconds required me to stop looking at standard code execution and start analyzing the underlying FreeRTOS system tick timestamps. It was a steep learning curve to realize that in real-time systems, managing the invisible OS scheduler is just as critical as managing the hardware itself.

## The most valuable thing I learned
The most valuable lesson I will carry into my interviews for Electronics Test Engineer roles is the fundamental difference between a "fast" system and a "deterministic" one. I now deeply understand that in aerospace Hardware-in-the-Loop (HIL) testing, an unpredictable microsecond delay can falsely invalidate perfectly good flight hardware. Learning how to mathematically guarantee worst-case execution times (WCET) using Rate-Monotonic boundaries, and strictly separating urgent hardware tasks (Top-Half) from deferred processing (Bottom-Half), has completely reshaped how I write and review safety-critical software.
