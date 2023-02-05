# Prepare

## Hardware
these things have to be prepared to play with qmk:
* A computer, seems any operating system is fine, according to the post over the internet. For myself, MacBook Pro 16'(2019 with intel chip)
* A keyboard, should be the keyboard support VIA/QMK, mine is KeyChron Q2 with knob

## Software
* [VIA configurator](https://www.caniusevia.com), this is an app to configure the key mappings.
* [qmk toolbox](https://github.com/qmk/qmk_toolbox), this is another app used to flash compiled firmware into your keyboard.
* [qmk cli](https://github.com/qmk/qmk_cli), this is the command line tool to compile our firmware source code into binary.

Follow the steps on the website to install these software. For me, I use the package manager on macOS to install qmk cli: `brew install qmk/qmk/qmk`, As for the other two, I just went for the release version and dowload and install them.

## Some Computer Things To Grab
* Q: What is VIA/QMK?
* A: This is a group of software let you customize your keyboard with infinite possibilites.

The `VIA configurator` allows users to intuitively remap any key on the keyboard. The settings(after spending a lot of time configuring), can set to any `layers` on the keyboard, and load into `onboard memory`.

The `QMK tools` allows developers to build their own keyboard firmware to control things like: `backlight effect`, `macros`, etc...

> These are no techincally explanation for `VIA/QMK`, I don't know much about embbeded programming, I am just explaining this from a end users point of view.


* Q: What is firmware?
* A: Firmware is still software(wierd term), it provides the low-level control over the device's specific hardware. BIOS is a kind of software, the firmware on the keyboard is loaded on the flash(a kind of ROM), so it can be overwrote many times. With speical firmware source code provided by the community, and compiled and loaded in by yourself, your keyboard can have a powerful control over the backlight, the macros, and many other things.

# Steps

## Environment Setup 
Basically, what you need to do is open a terminal, run a command, and let `qmk cli` check its dependencies and install the missing dependencies, here is what I did:
```
~ pwd
> /Users/neil

~ qmk setup
> 
...
☒ Can't find arm-none-eabi-gcc in your path.
Would you like to install dependencies? [Y/n]
...
```

So when I setup, an error occurs: "Can't find arm-none-eabi-gcc", even after I `brew install arm-none-eabi-gcc@8`, this error didn't go. I went through the log of `brew` and `qmk setup`, it seems `brew` refuse to put `arm-none-eabi-gcc excutable symlink` under my `/usr/local/bin`, here is what `brew` said:
```
>
arm-none-eabi-gcc@8 is keg-only, which means it was not symlinked into /usr/local,
because it might interfere with other version of arm-gcc.
This is useful if you want to have multiple version of arm-none-eabi-gcc
installed on the same machine.
```

This is something related to cross compiling, and I won't let these bianry to messup my environment, So I just export the `arm-none-eabi-gcc binary dir` into the `$PATH` env variable, this will only take effect under the current shell. 

After solving this annoying `can't find xxx` error, I proceeded to `qmk setup` and it succeeded:
```
~ qmk setup
> ...
~ ls | grep `qmk`
> qmk_firmware
~ cd qmk_firmware
```
Here we finish setting up the compiling environment.

## Edit The Source Code and Compile
This step is simple, for me, I just want to open some options of the `backlight effects`:
* Find the file define those light effects, the path of the file if related to your keyboard, mine is `keychron q2`, so the path is `/Users/neil/qmk_firmware/keyboards/keychron/q2/config.h`. 
* Edit this file with any editor you like, I just uncomment many lines to toggle all light effects. Save after you done edit.
* Compile is also simple, because I use `q2 knob version`, so I entered the `ansi_encoder` subdirectory, and execute `qmk compile --km via`. After compile OK, the target binary is under `/Users/neil/qmk_firmware/`.

Here we done compile the source code, and what left to do is load the target binary on the keyboard flash. 



## Flash The Keyboard
open the `qmk toolbox` app, unplug the keyboard and pull out the `spacebar`, then press the reset button, hold pressing and plug back the keyboard, the `log window` on the `qmk toolbox` said a `USB device` connected. 

After I try to flash, It failed the first time cause I connected the keyboard with the computer through some adaptor, so I try to connect them directly, and It works the second time. It may take a while to flash, wait patiently.

# The Light Effects
Here is a list of avaliable complex light effects contributed by the community:
* alphas_mods
* gradient_up_down
* gradient_left_right
* breathing
* band_sat
* band_val
* band_pinwheel_sat
* band_pinwheel_val
* band_spiral_sat
* band_spiral_val
* cycle_all
* cycle_left_right
* cycle_up_down
* rainbow_moving_chevron
* cycle_out_in
* cycle_out_in_dual
* cycle_pinwheel
* cycle_spiral
* dual_beacon
* rainbow_beacon
* rainbow_pinwheels
* raindrops
* jellybean_raindrops
* hue_breathing
* hue_pendulum
* hue_wave
* pixel_rain
* pixel_flow
* pixel_fractal
* typing_heatmap
* digital_rain
* solid_reactive_simple
* solid_reactive
* solid_reactive_wide
* solid_reactive_multiwide
* solid_reactive_cross
* solid_reactive_multicross
* solid_reactive_nexus
* solid_reactive_multinexus
* splash
* multisplash
* solid_splash
* solid_multisplash

My favorate is those `REACTIVE` series, when you press or release the key(configure by `#define RGB_MATRIX_KEYPRESSES`), the backlight will flicker. [here is a video show some light effect](https://www.youtube.com/watch?v=7f3usatOIKM)

# Reference
* https://www.reddit.com/r/Keychron/comments/xqgmb7/keychron_q2_guide_how_to_changeadd_rgb_effects/
* https://www.reddit.com/r/Keychron/comments/xqgmb7/keychron_q2_guide_how_to_changeadd_rgb_effects/
