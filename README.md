# Avionics UAV Radar System — Real-Time Systems Final Capstone

## One sentence
A real-time avionics UAV system that processes asynchronous radar interrupts with strictly bounded latency, built to demonstrate deterministic hardware-in-the-loop timing for an Electronics Test Engineer role.

## Demo
- **Video:** <Link to your YouTube/Wokwi video once recorded>
- **Live Wokwi:** <Link to your Wokwi project>

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
