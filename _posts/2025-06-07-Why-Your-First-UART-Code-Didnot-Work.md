---
layout: post
title: "Why Your First UART Code Didn’t Work (and How to Fix It) ?"
date: 2025-06-07
categories: embedded learning
---

It starts the same for all of us.  
You write your first embedded `printf("Hello World")` to the UART, confident it's going to show up in your serial monitor.  
You flash the code.  
Open TeraTerm.  
And then… nothing. Blank screen. Or worse, garbage characters that look like the output of a possessed vending machine.

---

## 🛠️ My First Encounter With UART Hell

I still remember mine — it was an STM32L476 board, some demo code from CubeMX, and a USB-to-Serial adapter I borrowed from one of my friends.

I was so sure I’d done it all right :
- Set the baud rate to 9600  
- Enabled TX and RX  
- Connected GND, TX, RX like everyone on StackOverflow told me (Of course, GPT wasn't invented!!)

But the terminal showed nothing.  
I double-checked the cable. Switched USB ports. Even rebooted my laptop (extreme self doubt). Still nothing.

---

## 🧩 So What Was Actually Wrong?

Here’s what I’ve painfully discovered & realised :

---

### 🌀 Baud Rate Blunders

Baud rate is just the "talking speed" between your microcontroller and your terminal. If they’re not talking at the same speed — expect garbage or silence.

💡 What to check:
- Did you set the *same* baud rate on both your MCU and serial terminal?
- Start with 9600 or 115200 — they’re the most reliable.

🛠️ What fixed it: Realizing my CubeMX PLL config was wrong, causing actual baud to be way off from what I selected.

---

### 🔌 TX & RX Pins — The Classic Mix-Up

I’ve been there — routed TX/RX to `PA9/PA10`, but the UART module was connected to `PB6/PB7`.  
The micro was transmitting... into a void.

💡 What to check:
- Double-check the datasheet and pinout.  
- Are you really using the correct alternate function for that UART?

🛠️ What fixed it: I stopped trusting my memory and cross-verified with the reference manual.

---

### 🛑 UART Code That "Sends" Nothing

So you’re writing characters to UART... but is it even ready to send?

📎 Here's what you should actually write:
```c
void uart_send(char c) {
    while (!(USART1->SR & USART_SR_TXE));  // wait until transmit buffer is empty
    USART1->DR = c;
}
```

Don’t assume HAL or printf will magically do the right thing — check the transmit flags.

---

### 🧠 Terminal Settings That Betray You

Even if everything else is right, your terminal might just be misconfigured.

💡 You need this combo (8N1):
- 8 data bits  
- No parity  
- 1 stop bit

🛠️ What fixed it: I checked PuTTY settings and realized it was using hardware flow control. Disabled it. Boom — UART alive.

---

### ⚡ Clock Configuration — The Silent Killer

Everything “looked” okay.  
CubeMX said 72 MHz. But it wasn’t actually configured.  
Your baud rate is calculated using the system clock. If that's messed up, the UART will be too fast or too slow — and your terminal won’t understand the data.
No error. No warning. Just silence or garbage.

💡 What to check:
- Did you enable the PLL (Phase-Locked Loop) properly?
- Are you using the HSE (external crystal) and not just the internal RC oscillator?
- Is the APB bus clock set correctly and connected to your UART peripheral?

🛠️ What fixed it: Digging into RCC ( Reset and Clock Control)  settings in cubeMx and using a debugger to verify actual SYSCLK. 

---

### 🎯 RX Is a One-Time Show?

You sent `"AT\r\n"` expecting MCU to reply.  
But nothing. You used `HAL_UART_Receive_IT()` once and forgot to call it again.

💡 What to check:
- Is your receive function being re-armed after each packet?
- Is RX line floating (no pull-up)? You might just be getting noise.

🛠️ What fixed it: Setting up a loop or callback to keep enabling the receiver.

---

### 💀 Cheap Adapters = Dead Time

You’ll spend 6 hours debugging perfect code and it turns out… your CH340 adapter is a knockoff.

🛠️ What fixed it: Swapped in a genuine CP2102. Worked instantly. Never went back.

---

## 🧪 Debugging UART in the Wild

When you're on-site or shipping a product:

You don’t have an IDE or debugger — just UART logs and whatever prints before the MCU crashes.
- A stuck bootloader can look exactly like a dead UART
- A missing ground wire can ghost your entire debug line
- I've had production firmware where the first printf() was literally the only clue why the board wouldn't initialize, without UART it would have been a tregedy to resolve!!

🔍 What saved me:

- Print a known string on boot: "BOOT_OK\r\n" — if you see that, your TX is working.
- Use a logic analyzer ( a tool that shows you real-time digital signals ) to see what's happening. It’s like a magnifying glass for serial lines. If UART still doesn't work — logic analyzers will tell you **whether bits are going out** at all. Don’t trust terminals blindly.
- Loopback test: Connect TX to RX on your adapter. If it echoes back, at least your PC side is fine.
- Start at 9600 baud. Higher speeds are sexy but fragile if the clock setup isn’t perfect.

---

## ✅ Final Thoughts

UART never works the first time — **not because it’s hard, but because it’s low-level and every detail matters!!**

The datasheet gives you the registers.  
The IDE gives you the abstraction.  
But the lessons? You earn those at 2 a.m. with empty logs and maximum doubt.

So if your UART isn't working — welcome to the club.  
And once it does work, you’ll never take that little “Hello” message for granted again!!

---

[← Back to Home](/techvidhi.in/)
