# hoverboard-firmware-hack-FOC - VARIANT_LIFTLINE
[![Build status](https://github.com/EFeru/hoverboard-firmware-hack-FOC/actions/workflows/build_on_commit.yml/badge.svg)](https://github.com/EFeru/hoverboard-firmware-hack-FOC/actions/workflows/build_on_commit.yml)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

## LiftLine Variant - Custom Modifications

This is a customized variant of the FOC hoverboard firmware specifically designed for the **LiftLine project**. It includes enhanced serial communication features and telemetry feedback.

### 📖 Complete Serial Protocol Documentation

**For detailed information about the serial communication protocol, see [`docs/SERIAL_PROTOCOL.md`](docs/SERIAL_PROTOCOL.md)**

This comprehensive documentation includes:
- Complete structure definitions and field descriptions
- Error flags and status byte bit-wise definitions
- Checksum calculation and validation algorithms
- Little-endian byte order explanation
- Practical examples and Python implementation code
- Implementation checklist for receiver systems
- All battery levels, control modes, and status indicators

### Key Modifications

#### Serial Communication Protocol (Quick Reference)
- **Baud Rate:** 38400 bps (USART2 & USART3)
- **Control Input:** USART3 - 8 bytes (speedL, speedR, checksum)
- **Feedback Output:** USART2/USART3 - 26 bytes with extended telemetry
- **Timeout Protection:** 0.8 seconds (automatic motor stop if no valid command received)


#### Serial Protocol Summary

**Command Frame (8 bytes):** `[0xABCD][speedL:int16][speedR:int16][XOR_checksum]`
- Commands sent directly to individual motors (no mixing)
- Positive values = forward, negative = reverse
- Checksum: `start ^ speedL ^ speedR`

**Telemetry Frame (26 bytes):** Sent every 10ms containing:
- Motor speeds (km/h × 100)
- Position tracking (distance in meters × 10)
- Battery voltage (V × 100)
- Board temperature (°C × 10)
- DC currents for each motor (A × 100)
- Combined error flags and status byte
- XOR checksum validation

**Key Features:**
- Little-endian byte order (LSB first)
- All fields 2-byte aligned (no padding)
- 0.8 second command timeout for safety
- Comprehensive error and status reporting

### Complete Protocol Documentation

**→ See [`docs/SERIAL_PROTOCOL.md`](docs/SERIAL_PROTOCOL.md) for:**
  - Detailed structure definitions and field descriptions
  - Bit-by-bit error flag and status byte definitions
  - Battery level indicators (0-7 levels with voltage thresholds)
  - Control mode values (OPEN, VOLTAGE, SPEED, TORQUE)
  - Checksum calculation algorithms and validation
  - Little-endian byte order examples
  - Complete Python implementation with examples
  - Implementation checklist for receiver systems

---

### Configuration Constants
```c
#define WHEEL_CIRCUMFERENCE_MM   487    // Wheel circumference (for ~155mm diameter)
#define RPM_TO_KPH_X100(rpm)  (((int32_t)(rpm) * 487 * 6) / 1000)
#define USART2_BAUD              38400  // Serial baud rate
#define USART3_BAUD              38400  // Serial baud rate
#define SERIAL_TIMEOUT           160    // ~0.8 sec timeout
#define DELAY_IN_MAIN_LOOP       5      // Main loop delay in milliseconds (10ms update frequency)
```

---

## Position Tracking Methods (VARIANT_LIFTLINE)

The LiftLine variant includes three selectable methods for position tracking:

### Method 1: RPM Integration (Default)
- **Advantages:** High resolution at elevated speeds, continuous calculation
- **Disadvantages:** Accumulation of rounding errors, depends on RPM precision
- **Resolution:** Variable based on speed
- **Formula:** `distance = RPM × wheel_circumference × time_interval / 60000`

### Method 2: Hall Sensor Commutation Counting
- **Advantages:** Fixed resolution (90 steps/revolution), no error accumulation, speed-independent
- **Disadvantages:** Lower resolution than RPM at high speeds
- **Resolution:** 5.4 mm per step (2.7mm with edge detection)
- **Activation:** Define `USE_HALL_POSITION_TRACKING` in config.h
- **Formula:** `distance = hall_count × wheel_circumference / 90`

### Method 3: Hall Interrupt-Based Tracking (Highest Precision)
- **Advantages:** No transition loss, maximum precision at all speeds, real-time updates
- **Disadvantages:** Requires command direction knowledge (from cmdL/cmdR)
- **Resolution:** 2.7 mm per transition
- **Activation:** Define `USE_HALL_INTERRUPT_TRACKING` in config.h
- **Key Features:** 
  - Captures every Hall sensor edge (rising and falling)
  - Direction-aware counting using motor command signs
  - DMA-less operation with EXTI interrupts on PB5-7 (left) and PC10-12 (right)

---

## Original FOC Features

This repository implements Field Oriented Control (FOC) for stock hoverboards. Compared to the commutation method, this new FOC control method offers superior performance featuring:
 - reduced noise and vibrations 	
 - smooth torque output and improved motor efficiency. Thus, lower energy consumption
 - field weakening to increase maximum speed range

Table of Contents
=======================

* **Wiki:** please check the wiki pages for [Getting Started](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki#getting-started) and for [Troubleshooting](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki#troubleshooting)
* [Hardware](#hardware)
* [FOC Firmware](#foc-firmware)
* [Example Variants](#example-variants)
* [Projects and Links](#projects-and-links)
* [Contributions](#contributions)

#### The hoverboards with mainboards also come with 2 sideboards(not [splitboards](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki/Firmware-Compatibility#split-boards)), check the following [wiki](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki/Sideboards) about this firmware

#### For the FOC controller design, see the following repository:
 - [bldc-motor-control-FOC](https://github.com/EFeru/bldc-motor-control-FOC)

#### Videos:
<table>
  <tr>
    <td><a href="https://youtu.be/IgHCcj0NgWQ" title="Hovercar" rel="noopener"><img src="/docs/pictures/videos_preview/hovercar_intro.png"></a></td>
    <td><a href="https://youtu.be/gtyqtc37r10" title="Cruise Control functionality" rel="noopener"><img src="/docs/pictures/videos_preview/cruise_control.png"></a></td>
    <td><a href="https://youtu.be/jadD0M1VBoc" title="Hovercar pedal functionality" rel="noopener"><img src="/docs/pictures/videos_preview/hovercar_pedals.png"></a></td>
  </tr>
  <tr>
    <td><a href="https://youtu.be/UnlbMrCkjnE" title="Commutation vs. FOC (constant speed)" rel="noopener"><img src="/docs/pictures/videos_preview/com_foc_const.png"></a></td> 
    <td><a href="https://youtu.be/V-_L2w10wZk" title="Commutation vs. FOC (variable speed)" rel="noopener"><img src="/docs/pictures/videos_preview/com_foc_var.png"></a></td>       
    <td><a href="https://youtu.be/tVj_lpsRirA" title="Reliable Serial Communication" rel="noopener"><img src="/docs/pictures/videos_preview/serial_com.png"></a></td>
  </tr>
</table>


---
## Hardware
 
![mainboard_pinout](/docs/pictures/mainboard_pinout.png)

The original Hardware supports two 4-pin cables that originally were connected to the two sideboards. They break out GND, 12/15V and USART2&3 of the Hoverboard mainboard. Both USART2&3 support UART, PWM, PPM, and iBUS input. Additionally, the USART2 can be used as 12bit ADC, while USART3 can be used for I2C. Note that while USART3 (right sideboard cable) is 5V tolerant, USART2 (left sideboard cable) is **not** 5V tolerant.

Typically, the mainboard brain is an [STM32F103RCT6](/docs/literature/[10]_STM32F103xC_datasheet.pdf), however some mainboards feature a [GD32F103RCT6](/docs/literature/[11]_GD32F103xx-Datasheet-Rev-2.7.pdf) which is also supported by this firmware.

For the reverse-engineered schematics of the mainboard, see [20150722_hoverboard_sch.pdf](/docs/20150722_hoverboard_sch.pdf)

 
---
## FOC Firmware
 
In this firmware 3 control types are available, it can be set in config.h file via CTRL_TYP_SEL parameter:
- Commutation (COM_CTRL)
- Sinusoidal (SIN_CTRL)
- Field Oriented Control (FOC_CTRL) with the following 3 control modes that can be set in config.h file with parameter CTRL_MOD_REQ:
  - **VOLTAGE MODE(VLT_MODE)**: in this mode the controller applies a constant Voltage to the motors. Recommended for robotics applications or applications where a fast motor response is required.
  - **SPEED MODE(SPD_MODE)**: in this mode a closed-loop controller realizes the input speed RPM target by rejecting any of the disturbance (resistive load) applied to the motor. Recommended for robotics applications or constant speed applications.
  - **TORQUE MODE(TRQ_MODE)**: in this mode the input torque target is realized. This mode enables motor "freewheeling" when the torque target is `0`. Recommended for most applications with a sitting human driver.

#### Comparison between different control methods

|Control method| Complexity | Efficiency | Smoothness | Field Weakening | Freewheeling | Standstill hold |
|--|--|--|--|--|--|--|
|Commutation| - | - | ++ | n.a. | n.a. | + |
|Sinusoidal| + | ++ | ++ | +++ | n.a. | + |
|FOC VOLTAGE| ++ | +++ | ++ | ++ | n.a. | +<sup>(2)</sup> |
|FOC SPEED| +++ | +++ | + | ++ | n.a. | +++ |
|FOC TORQUE| +++ | +++ | +++ | ++ | +++<sup>(1)</sup> | n.a<sup>(2)</sup> |

<sup>(1)</sup> By enabling `ELECTRIC_BRAKE_ENABLE` in `config.h`, the freewheeling amount can be adjusted using the `ELECTRIC_BRAKE_MAX` parameter.<br/>
<sup>(2)</sup> The standstill hold functionality can be forced by enabling `STANDSTILL_HOLD_ENABLE` in `config.h`. 

In all FOC control modes, the controller features maximum motor speed and maximum motor current protection. This brings great advantages to fulfil the needs of many robotic applications while maintaining safe operation.


### Field Weakening / Phase Advance

 - By default the Field weakening is disabled. You can enable it in config.h file by setting the FIELD_WEAK_ENA = 1 
 - The Field Weakening is a linear interpolation from 0 to FIELD_WEAK_MAX or PHASE_ADV_MAX (depeding if FOC or SIN is selected, respectively)
 - The Field Weakening starts engaging at FIELD_WEAK_LO and reaches the maximum value at FIELD_WEAK_HI
 - The figure below shows different possible calibrations for Field Weakening / Phase Advance
 ![Field Weakening](/docs/pictures/FieldWeakening.png)
 
 ⚠️ If you re-calibrate the Field Weakening please take all the safety measures! The motors can spin very fast!
 Power consumption will be highly increase and you can trigger the overvoltage protection of your BMS ⚠️


### Parameters
 - All the calibratable motor parameters can be found in the 'BLDC_controller_data.c'. I provided you with an already calibrated controller, but if you feel like fine tuning it feel free to do so 
 - The parameters are represented in Fixed-point data type for a more efficient code execution
 - For calibrating the fixed-point parameters use the [Fixed-Point Viewer](https://github.com/EFeru/FixedPointViewer) tool
 - The controller parameters are given in [this table](https://github.com/EFeru/bldc-motor-control-FOC/blob/master/02_Figures/paramTable.png)


### FOC Webview

To explore the controller without a Matlab/Simulink installation click on the link below:

[https://eferu.github.io/bldc-motor-control-FOC/](https://eferu.github.io/bldc-motor-control-FOC/)

---
## Example Variants

- **VARIANT_ADC**: The motors are controlled by two potentiometers connected to the Left sensor cable (long wired)
- **VARIANT_USART**: The motors are controlled via serial protocol (e.g. on USART3 right sensor cable, the short wired cable). The commands can be sent from an Arduino. Check out the [hoverserial.ino](/Arduino/hoverserial) as an example sketch.
- **VARIANT_NUNCHUK**: Wii Nunchuk offers one hand control for throttle, braking and steering. This was one of the first input device used for electric armchairs or bottle crates.
- **VARIANT_PPM**: RC remote control with PPM Sum signal.
- **VARIANT_PWM**: RC remote control with PWM signal.
- **VARIANT_IBUS**: RC remote control with Flysky iBUS protocol connected to the Left sensor cable.
- **VARIANT_HOVERCAR**: The motors are controlled by two pedals brake and throttle. Reverse is engaged by double tapping on the brake pedal at standstill. See [HOVERCAR wiki](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki/Variant-HOVERCAR).
- **VARIANT_HOVERBOARD**: The mainboard reads the two sideboards data. The sideboards need to be flashed with the hacked version. The balancing controller is **not** yet implemented.
- **VARIANT_TRANSPOTTER**: This is for transpotter build, which is a hoverboard based transportation system. For more details on how to build it check [here](https://github.com/NiklasFauth/hoverboard-firmware-hack/wiki/Build-Instruction:-TranspOtter) and [here](https://hackaday.io/project/161891-transpotter-ng).
- **VARIANT_SKATEBOARD**: This is for skateboard build, controlled using an RC remote with PWM signal connected to the right sensor cable.

Of course the firmware can be further customized for other needs or projects.


---
## Projects and Links

- **Original firmware:** [https://github.com/lucysrausch/hoverboard-firmware-hack](https://github.com/lucysrausch/hoverboard-firmware-hack)
- **[Candas](https://github.com/Candas1/) Hoverboard Web Serial Control:** [https://github.com/Candas1/Hoverboard-Web-Serial-Control](https://github.com/Candas1/Hoverboard-Web-Serial-Control)
- **[RoboDurden's](https://github.com/RoboDurden) online compiler:** [https://pionierland.de/hoverhack/](https://pionierland.de/hoverhack/) 
- **Hoverboard hack for AT32F403RCT6 mainboards:** [https://github.com/cloidnerux/hoverboard-firmware-hack](https://github.com/cloidnerux/hoverboard-firmware-hack)
- **Hoverboard hack for split mainboards:** [https://github.com/flo199213/Hoverboard-Firmware-Hack-Gen2](https://github.com/flo199213/Hoverboard-Firmware-Hack-Gen2)
- **Hoverboard hack from BiPropellant:** [https://github.com/bipropellant](https://github.com/bipropellant)
- **Hoverboard breakout boards:** [https://github.com/Jana-Marie/hoverboard-breakout](https://github.com/Jana-Marie/hoverboard-breakout)

<a/>

- **Bobbycar** [https://github.com/larsmm/hoverboard-firmware-hack-FOC-bbcar](https://github.com/larsmm/hoverboard-firmware-hack-FOC-bbcar)
- **Wheel chair:** [https://github.com/Lahorde/steer_speed_ctrl](https://github.com/Lahorde/steer_speed_ctrl)
- **TranspOtterNG:** [https://github.com/Jan--Henrik/transpOtterNG](https://github.com/Jan--Henrik/transpOtterNG)
- **Hoverboard driver for ROS:** [https://github.com/alex-makarov/hoverboard-driver](https://github.com/alex-makarov/hoverboard-driver)
- **Ongoing OneWheel project:** [https://forum.esk8.news/t/yet-another-hoverboard-to-onewheel-project/60979/14](https://forum.esk8.news/t/yet-another-hoverboard-to-onewheel-project/60979/14)
- **ST Community:** [Custom FOC motor control](https://community.st.com/s/question/0D50X0000B28qTDSQY/custom-foc-control-current-measurement-dma-timer-interrupt-needs-review)

<a/>

- **Telegram Community:** If you are an enthusiast join our [Hooover Telegram Group](https://t.me/joinchat/BHWO_RKu2LT5ZxEkvUB8uw)

---
## Stargazers

[![Stargazers over time](https://starchart.cc/EFeru/hoverboard-firmware-hack-FOC.svg)](https://starchart.cc/EFeru/hoverboard-firmware-hack-FOC)

---
## Contributions

Every contribution to this repository is highly appreciated! Feel free to create pull requests to improve this firmware as ultimately you are going to help everyone. 

If you want to donate to keep this firmware updated, please use the link below:

[![paypal](https://www.paypalobjects.com/en_US/NL/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=CU2SWN2XV9SCY&currency_code=EUR&source=url)

---
