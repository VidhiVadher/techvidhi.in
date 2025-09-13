---
layout: post
title: "What is really a debouncing ??"
date: 2025-09-13
categories: embedded learning
---

# Debouncing in Hardware vs Software: Which One Should You Choose?

We've all been there: you press a button and your system thinks you pressed it 3 times.  
Nope — it’s not a ghost. It’s **switch bounce**. And yes, it’s real.  
Let’s go beyond textbook theory and dive into **when**, **why**, and **how** to debounce — with real-world context.

---

## First, What Is Bounce?

Mechanical buttons aren’t perfect. When you press them, the internal metal contacts **don’t settle instantly**.  
They bounce — rapidly making and breaking contact for a few milliseconds.

### What It Looks Like:
- You press once
- The microcontroller sees it as multiple presses
- Your logic explodes (figuratively... hopefully)

> Real-world analogy: It's like flipping a light switch, but your shaky hand makes it flicker on-off-on-off-on before settling.

---

## Real Consequences of Ignoring Bounce

- A **single button press** starts the motor *three* times
- Your UI menu skips 2 options with one click
- Your firmware triggers a safety shutdown from false inputs
- You lose trust in your input system (and so does your customer)

---

# Hardware Debouncing

### How It Works ? 

Use **RC circuits** or **Schmitt trigger buffers** to clean up the signal **before it hits the microcontroller**.

1. Basic RC Filter : 
- A resistor (R) and capacitor (C) form a low-pass filter.
- It smooths the bouncy edges by slowing down the signal's rise/fall.
Example values:
- R = 10kΩ, C = 100nF → Time constant = 1ms
- This means the voltage rises/falls gradually over 1ms, absorbing the bounce noise.

2. Schmitt Trigger : 
- After the RC filter, you can use a Schmitt trigger buffer (like 74HC14).
- It adds hysteresis, meaning it only toggles the logic level when the input passes certain high/low thresholds, reducing edge ambiguity.

### When to use this ?

1. When MCU is asleep or in deep sleep and wake-up pins must be stable
→ Software can’t run if it’s not powered yet. You need to use hardware debounce.

2. You're building rugged or EMI-prone industrial systems
→ Physical noise needs to be filtered before it even reaches the logic.

3. You need ultra-fast response without software overhead
→ E.g., a start/stop button in an emergency stop circuit.

Here, hardware debounce is **mandatory**!!

### Why to use this ?

- Works independently of code or system state
- Deterministic and consistent (set-and-forget)
- Reliable even without microcontroller initialization
- Faster debounce with no latency or polling delay

---

### Pros:

1. Firmware-independent :
- Works even if the MCU is not running yet — perfect for power-on buttons, reset lines, and wake-up triggers.

2. Low latency / deterministic : 
- Debouncing happens as fast as the RC time constant settles — no software delays, no CPU cycles consumed.

3. Immune to firmware bugs : 
- You won’t accidentally break it with code refactors or missed edge conditions.

4. Works in all power states : 
- MCU deep sleep? Power off? Doesn’t matter — debounce still works.

5. Resilient to fast transient noise : 
- Good for noisy industrial environments or long physical wires.

6. Instant hardware debugging :
- You can probe the RC node on an oscilloscope and see the bounce disappear — very visual.

### Cons:

1. Extra BOM cost and board space
- Even a few resistors and caps matter in ultra-compact or cost-sensitive designs.

2. Not field-configurable
- Want to increase debounce time after seeing bounce in testing? Too late. It’s baked into the hardware.

3. Not sufficient alone in some cases : 
- Can’t help with slow-release contacts, faulty switches, or switch chatter longer than your RC filter time.

4. Fixed timing assumptions : 
- Different buttons may bounce differently, but RC filters apply the same delay to all.

5. Schmitt triggers may introduce quirks : 
- Some logic ICs add internal delays, propagation latency, or threshold mismatches.

6. Needs layout consideration : 
- Long traces between button and MCU? The RC may clean the signal at the wrong end of the trace.

---

# Software Debouncing

### How It Works ?
You read the pin state in firmware — and **wait for it to be stable** for a certain period (e.g. 10ms).

1. Polling-based Debounce
- Regularly read the pin. Consider it valid only if it stays stable for N milliseconds.

2. Interrupt + Timer Approach
- Trigger an interrupt on button press. Start a timer. Confirm after timeout.
- Steps:
    - Interrupt fires on button edge
    - Disable further interrupts
    - Start timer (e.g. 10ms)
    - On timer complete: read pin again
    - If still valid → it's a legit press
    - Else → noise
    - Re-enable interrupt
- This is the cleanest method in multitasking systems (e.g., FreeRTOS) because it avoids ISR overload and gives reliable results.

---

### When to use it ? 
1. When you’re reading multiple buttons or matrix keypads.
2. When you want per-button tuning (some bounce more than others)
3. When you’re building low-cost systems (no extra components)
4. You want field-adjustable debounce settings
5. You're debugging inputs and need visibility through logs or UART


### Why to use it? 
1. No hardware components = less BOM cost
2. Easier to change debounce timing later
3. Can handle non-mechanical inputs too (e.g. capacitive buttons, touch sensors)
4. Works well when MCU is always on or in active mode

Software debounce wins: it’s tweakable, cheap, and scalable.

---

### Pros:

1. Fully configurable : 
- Need 20ms on one button and 5ms on another? Easy in code.

2. Component-less : 
- Great for tight PCBs, wearable devices, or cost-sensitive designs.

3. Scalable to many inputs : 
- One task or timer can debounce many inputs — useful for matrix keypads, touch panels, etc.

4. Modifiable post-deployment : 
- You can fix debounce issues in firmware updates — no soldering needed.

5. Can be adapted to user interaction : 
- You can add press-and-hold, multi-tap, or soft input logic easily.

6. Can be tested and logged :
- Log how long a button took to settle or how many bounces occurred — good for QA/debugging.

### Cons:

1. Consumes CPU time : 
- Even 1ms polling adds up if you’re debouncing many inputs on a constrained MCU.

2. Non-deterministic under load : 
- If your system is busy with higher priority tasks, debounce logic can be skipped or delayed.

3. Interrupt + debounce timing is tricky : 
- You can get false triggers if ISR isn’t properly delayed or re-checked after timeout.

4. Fails silently if coded wrong : 
- A wrong edge detection or missing rearm timer can completely break the input logic.

5. Not usable during boot/sleep modes : 
- MCU isn’t running yet or is asleep? Software debounce can’t help until it's awake and functional.

6. Can increase code complexity : 
- Especially in systems with preemption, low-power states, or shared GPIOs.

---

# Hybrid Debouncing: The Best of Both Worlds?

Many production systems use **a small RC filter** + **software verification**.

- RC reduces high-frequency noise
- Software filters the mechanical bounces
- Schmitt-trigger buffer smooths transitions even more

Especially useful for:
- Safety-critical systems
- Inputs from long wires
- Harsh EMI environments

---

# Pro Tips

- Don’t guess debounce time — **measure** bounce with a logic analyzer (some switches bounce for 20ms+).
- If using software, **never debounce inside ISR** — use a timer or main loop.
- Use `volatile` on debounce state variables — compiler optimizations will bite you.
- For capacitive or touch buttons — debounce isn’t mechanical, but similar filtering logic still applies!

---

## Final Thoughts

Debouncing is like seat belts — you don’t realize its importance until something jerks.  
**Ignoring bounce isn't clever, it's just delayed debugging**.

Choose smart. Bounce less.

---

[← Back to Blog List](/techvidhi.in/blog/)