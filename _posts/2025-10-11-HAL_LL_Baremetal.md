---
layout: post
title: "HAL vs LL vs Baremetal - when to use where ??"
date: 2025-10-11
categories: embedded learning
---

# HAL vs LL vs Baremetal – A Practical, Real-World Dive for Hardware Engineers
Let’s skip the book theory and focus on what really matters when you’re faced with an actual STM32 project. Here’s how HAL, LL, and baremetal approaches play out, with practical examples from real hardware tasks and deeper insight into why you’d choose one over the others.

## 1. Baremetal (Direct Register Access)

### What it is:
Directly writes and reads microcontroller registers yourself. No vendor libraries involved – you’re reading the datasheet and coding the hardware “the hard way.”

### Real project example:
You’re developing an industrial gas-leak detection unit for a chemical plant. The system monitors gas concentration via an analog sensor and must trigger a relay cutoff + buzzer within microseconds if a threshold is breached.

In your ISR (Interrupt Service Routine), you need to:
- Read the ADC value directly (no HAL latency),
- Compare against a threshold,
- Trigger GPIO outputs to cut off valves and sound alarms,
- And do it all in under 20 microseconds — consistently.

With HAL or LL, abstraction layers may introduce overhead or hidden delays.
But with bare-metal, you write:

if (ADC1->DR > GAS_LIMIT) {
    GPIOB->ODR |= (1 << BUZZER_PIN);  // Sound alarm
    GPIOC->ODR &= ~(1 << VALVE_PIN);  // Shut valve
}

You’re directly commanding the silicon — no middlemen.

In industrial safety-critical systems, "slow but safe" isn’t safe — fast and guaranteed wins every time.

### Why Bare-Metal Is Powerful :

- Minimum code size, fastest execution, no surprises.​
- Perfect for power-critical, timing-critical, or bootloader code.
- No unpredictable function wrappers.
- You guarantee deterministic response time.
- You KNOW exactly what the chip is doing, as you’re also debugging register-level issues.

### But It’s Not for the Faint of Heart :

- You need to know the hardware inside-out. One wrong bit in a control register, and nothing works.
- More prone to bugs if register fields are missed or misused.
- Debugging takes longer. There’s no helpful abstraction or function call to step through.
- Slow development for complex peripherals (DMA, USB, Ethernet, etc.).

### When to Reach for Bare-Metal ?

- Educational or certification-focused environments where knowing what each bit does actually matters  (e.g., sensor sampling, waveform generation)
- Code where every cycle counts : Low-level sensor reads, bitbang protocols, custom timing ISRs.​
- Exploring peripherals where vendor libraries don’t support full functionality.
- Memory-constrained systems.
- Bootloaders or ultra-low-power startup code

## 2. HAL (Hardware Abstraction Layer)

### What It is ?
- Think of HAL as your microcontroller’s autopilot.
- You tell it what you want — like “turn on this pin” or “send this data over UART” — and it does the low-level stuff for you behind the scenes.
- No need to memorize register names or bit positions. HAL handles it all using easy-to-read function calls, usually generated from STM32CubeMX.

HAL is like asking a restaurant waiter to place your order instead of going into the kitchen and making it yourself.

### Real project example

Say you’re building a smart agriculture prototype.

You hook up:
- A temperature sensor over I2C
- A soil moisture sensor over ADC
- A humidity sensor over UART

You want to read all three and blink an LED if any value is off. With HAL, it’s a breeze:

HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);                     // Turn on LED  
HAL_Delay(500);                                                         // Wait half a second  
HAL_I2C_Master_Transmit(&hi2c1, SENSOR_ADDR, data, len, HAL_MAX_DELAY); // Send the data

No messing with clocks, registers, or IRQ priorities — just get things working quickly.

### Why People Love HAL ? 
- Fast to prototype — You get things blinking, sensing, and communicating within hours.
- Easy to read — Clean function names. Great for onboarding new team members.
- Cross-family compatible — Migrate from STM32F1 to STM32L4? Most code just... works.
- Team-friendly — Less low-level fiddling = fewer bugs and more collaboration.

### But Be Aware...

- Code bloat — You might pull in hundreds of lines of code to blink a pin.
- Not everything is exposed — Some features (like DMA tweaks or ultra-low power tricks) need LL or register-level access.
- Learning is shallow — It works, but you don’t see what’s happening underneath.


### When to Choose HAL ? 
- You're rapidly prototyping or testing a new board
- You're working with USB, CAN, Ethernet — complex protocols where HAL shines
- You value development speed over bare-metal optimization
- You're part of a team, where readable/maintainable code matters more than shaving off a few cycles

## 3. LL (Low Layer Drivers)

### What it is ?
Low Layer Drivers are Vendor-provided APIs that sit between HAL and baremetal, giving you (much) closer-to-the-metal functions than HAL, but with less abstraction. Typically, these are lightweight with direct register operations abstracted as inline or simple functions.​

### Real project example
Let’s say you’re building a wearable that needs super-low sleep current.
LL lets you access power control registers and fine-tune peripheral off/on beyond HAL’s defaults:

LL_GPIO_SetOutputPin(GPIOA, LL_GPIO_PIN_5); // Set PA5 High

Or, precise DMA configuration for high-speed ADC sampling:

LL_DMA_ConfigTransfer(DMA1,
  LL_DMA_CHANNEL_1,
  LL_DMA_DIRECTION_MEMORY_TO_PERIPH,
  ... // more config
);

### Why it is useful ?
- More performance than HAL, less manual error than baremetal.​
- Access to more features (e.g., tweaking DMA, power modes, interrupt masking).
- Can mix with HAL or baremetal where needed.

### What are the disadvantages ?

- Requires reading both the reference manual and LL documentation.
- Not always portable; sometimes chip-specific functionality.​
- Still some abstraction overhead (but much less than HAL).

### Where used best?

- Power-optimized, low-latency designs.
- When you need precise control but not full baremetal responsibility.
- Critical code sections needing speed but flexibility.

## A practicle comparision table

Feature      |  Baremetal            |  LL                 |  HAL                
-------------+-----------------------+---------------------+---------------------
Control      |  Absolute             |  High (peripheral)  |  Moderate (abstract)
Portability  |  Low                  |  Medium             |  High               
Speed        |  Fastest              |  Very fast          |  Moderate-fast      
Code size    |  Smallest             |  Small/medium       |  Largest            
Ease of use  |  Hardest              |  Moderate           |  Easiest            
Learning     |  Deep (manual!)       |  Needs manual+docs  |  API docs only      
Features     |  All (if documented)  |  Most               |  Most common        

[← Back to Blog List](/techvidhi.in/blog/)