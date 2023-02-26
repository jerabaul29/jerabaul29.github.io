---
layout: post
title:  "Getting started with Sparkfun Artemis development boards"
date:   2020-12-20 12:00:00 +0200
categories: jekyll update
---

*Note: this post includes quite a lot of information in both the parts "development ecosystem" and "Getting started with development in Arduino IDE" that is dependent on the current state of the ecosystem / toolings. This means that the information may get gradually out-of-date as things progress forward and better tooling becomes available.*

*Note: as of October 2021, the ecosystem around the Artemis line of MCUs has progressed quite a bit. The core v1 (bare metal) is working very well and I am using it in several projects, and it has definitely reached maturity in my opinion. I am much less well known with the core v2, which is the MbedOS based one, and while it is getting better fast, it was in my opinion not ready for production the last time I checked - but it may be different by the time you read this blog post.*

I am a huge fan of the Arduino ecosystem and I have been using many Arduino Uno, Mega, and Due boards for projects both home and at work. These boards are well known by the Arduino community, with a lot of code, documentation, tutorials, and help available online. However, the micro-controller units (MCUs) they rely on are a bit old, and more recent chips do much better in terms of both computational power, RAM, SRAM, and power efficiency.

As a consequence, I am looking at changing my main "goto MCU" for future projects. I want something that is both more power efficient, and with much better computational and memory resources than the "old" Arduino boards, while I want at the same time good community support. After a bit of hesitation (in particular, Teensy boards were a strong contender), I decided to go for the Sparkfun Artemis line of development boards. These are a "wrap-up" around the Apollo Ambiq 3 BLU MCU, with very nice performances: ARM Cortex-M4 CPU at 48MHz, dedicated floating point unit, 1MB flash, 384kB RAM, built-in bluetooth, power consumption "all on" of typically 1mA (note that this is for the MCU itself; the Sparkfun Red Artemis development boards has both LEDs, power regulators, etc, to make development convenient; these increase the power consumption a lot; however, the MCU is also available as a standalone breakout if needed for final products, see [on the Sparkfun website](https://www.sparkfun.com/products/15484)), few micro-amps of consumption in deep sleep, and many pins and peripherals (see the development board description [at Sparkfun](https://www.sparkfun.com/products/15444), the details on the CPU are available [from Ambiq](https://ambiq.com/apollo3-blue/)).

# Development ecosystem

Sparkfun has made available 2 versions of "Arduino IDE cores" [on github](https://github.com/sparkfun/Arduino_Apollo3). The "old" version is a "usual" Arduino core, without much surprise there. However, the "new" version is actually based on a full-featured Real Time OS (RTOS), namely, [Mbed-OS](https://os.mbed.com/mbed-os/). This means much more help for developing complex programs, including possibilities for running several threads, and some help by the OS for automatically setting the board in low power modes. The downside is quite a bit more SRAM and RAM consumption (however, these are relatively plentiful on this MCU), and the overhead implied by running a full RTOS, in addition to longer compilation times.

# Getting started with development in Arduino IDE

The instructions for installing the Mbed-OS based core into the Arduino IDE are available [on the Sparkfun website](https://learn.sparkfun.com/tutorials/artemis-development-with-arduino/all) (see the "Arduino Installation" section). In my experience (Ubuntu 18.04 and Ubuntu 20.04) this worked very well, except that the uploader installed did not have correct execution rights (see the [issue on github](https://github.com/sparkfun/Arduino_Apollo3/issues/309)). This was easily fixed by performing a ```chmod +x``` on the files named in the error messages issued by the Arduino IDE.

The second problem I encountered in setting things up was with the uploading process itself. At first, it was problematic to upload any code on the board, as discussed [in this Github issue](https://github.com/sparkfun/Arduino_Apollo3/issues/310). The problem this time was with which bootloader to use, and what baudrate to use. To summarize:

- despite being marked as "Out of order", the "SparkFun Variable Loader" (SVL) is the correct uploader to use.
- If you use by error the "Ambiq Secure Bootloader", even only once, the SVL will get overwritten by your code, and you will need to burn again the SVL.
- In addition, when using the SVL, I was not able to upload when using baudrate higher than 460800, so do not use the max baudrate!

Once these steps are done, it is very easy to program the board, and the advantage of using an Mbed-OS core is very visible when several tasks are to be run "in parallel" (though this is a single core CPU, so really the RTOS is alternating between which thread is being executed). To illustrate, it is very easy to periodically toogle a LED while sending some serial information:

```cpp
rtos::Thread thread_1;
rtos::Thread thread_2;

void thread_fn(void){
  unsigned long time_last_msg;
  constexpr unsigned long period_print = 1000;
  
  unsigned int status_count = 0;

  time_last_msg = millis();
  
  while(true){
    Serial.begin(115200);
    Serial.printf("time (ms): %d, status count: %d\n", millis(), status_count++);
    Serial.end();
            
    rtos::ThisThread::sleep_for(time_last_msg + period_print - millis());
    time_last_msg += period_print;
  }
}

void thread_led(void){
  pinMode(LED_BUILTIN, OUTPUT);
  
  while (true){
    digitalWrite(LED_BUILTIN, HIGH);
    rtos::ThisThread::sleep_for(200);
    digitalWrite(LED_BUILTIN, LOW);
    rtos::ThisThread::sleep_for(200);
  }
}

void setup() {
  thread_1.start(thread_fn);
  thread_2.start(thread_led);
}

void loop() {
}
```
