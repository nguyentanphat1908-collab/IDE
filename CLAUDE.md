# CLAUDE.md

## Repository
This is a documentation-first engineering repository. Do not assume build, test, or deployment workflows exist.

## Source Annotations

* `[ĐA]`: confirmed/authoritative information.
* `[DS]`: derived or calculated information.
* `[ĐX]`: proposal or suggestion; not an approved fact.

Never invent specifications. Explicitly report conflicting information.

## System Architecture

* Raspberry Pi 4: master/high-level control, communication, AOI, image processing, heavy computation.
* S7-1200: deterministic real-time control, I/O, motion, PTO, PWM.

Do not move real-time tasks to Pi or heavy processing to PLC without justification.

## Critical Constraints

* PTO only: `Q0.0–Q0.7`.
* PWM timing must be verified at actual 24V operation.
* High-speed probe signals require hardware interrupts.
* AOI processes 16 tiles sequentially; release each image after processing.

## System Behavior
Follow the documented 8-state machine from `IDLE` to `INSPECTING`. Do not introduce undocumented transitions.

## Documentation
Use the Document Map to find authoritative information. Follow existing Mermaid conventions.

PRD.md defines WHAT the system must achieve. CLAUDE.md defines constraints, architecture, and working context.
