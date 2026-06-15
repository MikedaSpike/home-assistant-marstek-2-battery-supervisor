# Home Assistant Marstek Battery Supervisor (Serial Setup)

![Latest Release](https://img.shields.io/github/v/release/MikedaSpike/home-assistant-marstek-2-battery-supervisor)
![Downloads](https://img.shields.io/github/downloads/MikedaSpike/home-assistant-marstek-2-battery-supervisor/total)
![License](https://img.shields.io/github/license/MikedaSpike/home-assistant-marstek-2-battery-supervisor)
![Issues](https://img.shields.io/github/issues/MikedaSpike/home-assistant-marstek-2-battery-supervisor)
![Last Commit](https://img.shields.io/github/last-commit/MikedaSpike/home-assistant-marstek-2-battery-supervisor)
![Commit Activity](https://img.shields.io/github/commit-activity/m/MikedaSpike/home-assistant-marstek-2-battery-supervisor)
![Stars](https://img.shields.io/github/stars/MikedaSpike/home-assistant-marstek-2-battery-supervisor)

---

This project provides an advanced automation framework built specifically for managing a **dual-battery home energy storage setup running in a strict serial configuration**. 

⚠️ **CRITICAL REQUIREMENT:** This framework is designed *exclusively* for setups where two physical Marstek batteries are placed on a single electrical group/circuit and **MUST NOT operate in parallel**. The supervisor guarantees that only one battery is dynamically driven to its maximum limits (up to 2500W) at any given moment to prevent overloading your circuit breaker.

It acts as a robust supervisor layer on top of Codebob's *Home Battery Control (HBC)* integration.

---

## How It Works (The Core Logic)

Unlike traditional automation setups, this supervisor **does not directly send active charging or discharging power commands** to the hardware. Instead, it acts as an intelligent gatekeeper by dynamically manipulating the maximum allowed boundaries: `max_charge_power` and `max_discharge_power` for each unit. 

Codebob's *Home Battery Control (HBC)* integration remains completely responsible for calculating and distributing the actual dynamic wattage steps. The supervisor evaluates the system at a set **time interval** and opens or closes the power "gates" (setting them to either 0W or maximum capacity, e.g., 2500W) according to the following strict logic:

### 1. Charging Sequence
- HBC dictates when the system needs to store energy. 
- The supervisor funnels this demand into the designated **Master (Priority) Battery** first.
- The Master battery charges until it hits its configured `charging_cutoff_capacity` (Full).
- Once full, the supervisor instantly locks the Master's max charge power to `0W` and opens up the **Slave (Second) Battery** to take over the incoming charge load.

### 2. Discharging Sequence
- To balance overall cycle wear, the system reverses priority during consumption: the **Slave Battery is depleted first**.
- The Slave battery discharges to support your home load until it reaches its `discharging_cutoff_capacity` (Empty).
- Once empty, the supervisor clamps the Slave's max discharge limit to `0W` and unlocks the **Master Battery** to handle the remaining household discharge demand.

### 3. Stuck / Failover Protection
- If HBC commands the active battery to charge or discharge, but the supervisor detects zero physical activity over a consecutive evaluation interval (e.g., the hardware is frozen or drops commands), a failover is triggered.
- The supervisor automatically restricts the frozen battery and **forces a switchover to the alternate unit**, ensuring your home backup or solar storage cycle isn't interrupted by a hardware glitch.

---

## Core Features

- **Strict Serial Power Allocation:** Enforces a hard Master/Slave boundary ensuring that only one battery charges or discharges at full capacity at any time.
- **Hysteresis & Voltage Bounce Prevention:** Employs structural capacity safety buffers (+3% SOC window) to stop rapid, unstable micro-switching when a battery sits near its cutoff capacity.
- **Zombie/Stuck Detection:** Actively tracks requested power versus actual AC power output. If the active battery freezes or refuses to respond to a command for more than 90 seconds, the supervisor automatically fails over to the standby battery.
- **Dynamic Negative Pricing Protection (Optional):** Includes intelligent override logic to completely halt discharging and force-charge both batteries simultaneously at a safe, balanced rate when dynamic market energy prices drop below zero.
- **Filtered Live UI Logging:** Dynamically formats system status, decision parameters, and hardware health variables into structured text streams for your front-end dashboard.

---

## 1. Choosing Your Automation File

This repository contains **4 different versions** of the automation. You only need to choose and install **ONE** file based on your language preference and whether your dynamic energy contract utilizes negative price optimization:

### With Negative Pricing Logic
*Use these if your energy provider has dynamic hourly rates (like ANWB Energie, Tibber, Zonneplan) and you want both batteries to force-charge when rates drop below 0.*
1. **`battery_supervisor_nl_with_negative_pricing.yaml`**
   - **Language:** Dutch text & UI logs.
   - **Requires helper entities:** `marstek_laatste_actie` and `marstek_log_historie`.
2. **`battery_supervisor_en_with_negative_pricing.yaml`**
   - **Language:** English text & UI logs.
   - **Requires helper entities:** `marstek_last_action` and `marstek_log_history`.

> ### What does the "Negative Pricing" logic do? (In short)
> Normally, this supervisor strictly enforces that only **one battery at a time** may charge or discharge to protect your electrical group. However, when your dynamic energy provider contract hits a **negative price hour** (meaning you get paid to consume electricity), the supervisor temporarily unlocks both units:
> 1. It instantly drops discharging power to **absolute zero** (so you don't export cheap power).
> 2. It forces **both batteries to charge simultaneously** at a safe, balanced grid rate (e.g., 1000W-1200W per battery) without exceeding your total breaker threshold. 
> 
> *As soon as prices turn positive again, the system immediately restores the safe, single-battery Master/Slave serial logic.*

### Without Negative Pricing Logic (Standard Serial Switching)
*Use these if you have a fixed/variable contract, or if you prefer to let another system manage negative price handling entirely.*
3. **`battery_supervisor_nl_no_negative_pricing.yaml`**
   - **Language:** Dutch text & UI logs.
   - **Requires helper entities:** `marstek_laatste_actie` and `marstek_log_historie`.
4. **`battery_supervisor_en_no_negative_pricing.yaml`**
   - **Language:** English text & UI logs.
   - **Requires helper entities:** `marstek_last_action` and `marstek_log_history`.

---

## 2. Configuration & UI Logging Setup

Before importing the automation, you must create the text helpers that store operational telemetry. This project features a robust, built-in logging system designed to feed directly into your Home Assistant Lovelace dashboard UI layout:

* **Last Action (`input_text.marstek_last_action` / `_laatste_actie`):** Displays a clean, single-line summary of the *exact state* the supervisor is executing right now (e.g., `[Master Full] Charging Slave Battery`). Excellent for secondary text entities or glance cards.
* **Log History (`input_text.marstek_log_history` / `_log_historie`):** Functions as a local rolling terminal. Every time the automation switches units, intercepts a zombie state, or shifts into negative price protection, it prepends a timestamped line. This allows you to monitor chronological history over multiple days right from a Markdown card without checking standard Home Assistant log traces.

### Creating the Helpers via the Home Assistant UI
1. Navigate to **Settings** -> **Devices & Services** -> **Helpers**.
2. Click **Create Helper** (bottom right) and select **Text** (`input_text`).
3. Set up the exact entities required for your chosen version:

#### Helper Configurations for Dutch Setup (`_nl_` files)
- **Name:** Marstek Laatste Actie
  - **Entity ID:** `input_text.marstek_laatste_actie`
- **Name:** Marstek Log Historie
  - **Entity ID:** `input_text.marstek_log_historie`

#### Helper Configurations for English Setup (`_en_` files)
- **Name:** Marstek Last Action
  - **Entity ID:** `input_text.marstek_last_action`
- **Name:** Marstek Log History
  - **Entity ID:** `input_text.marstek_log_history`

---

## 3. How to Install the Automation

Follow these steps to safely load the supervisor code into your Home Assistant environment:

1. Open the `.yaml` file from this repository that matches your requirements and **copy the entire contents** to your clipboard.
2. In Home Assistant, go to **Settings** -> **Automations & Scenes** -> **Automations**.
3. Click the **Create Automation** button in the bottom right corner.
4. In the pop-up window, choose **Start with an empty automation** (do not select a blueprint or scratchpad template).
5. Look at the top right corner of the new automation screen, click the **three vertical dots (options menu)**, and select **Edit in YAML**.
6. Select everything inside the text area, delete it completely, and **paste the copied code** from this repository.
7. Click **Save** in the bottom right corner. Give it a descriptive name when prompted.

⚠️ **Important Node Verification:** Review the entity IDs used within the automation configuration variables section (at the bottom of the file) to ensure they perfectly match your actual system sensor naming conventions (e.g., `sensor.marstek_m1_battery_state_of_charge`, `sensor.hbc_energy_prices_data`, etc.). Adjust them in the YAML if yours are named differently.
