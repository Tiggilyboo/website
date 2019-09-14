+++
date = "2019-09-14T21:49:00+13:00"
title = "Handwired Keyboard Project: Part 2"
description = "The Linux Kernel, Device layers, i2c drivers, and kernel modules"
tags = ["Linux","Kernel","C","Planck","Keyboards","ARM"]
topics = ["Blog", "Hardware"]
comments=true
+++

## Intro

As a quick refresher from the first part in the series, I'm making a handwired keyboard which uses an altogether rediculous chinese knockoff 6 core monster. Better yet, there are not enough GPIOs available on the board to handle the input matrix (12x4). Oh, and the case I want to use (from my current keyboard does not have enough vertical headroom to fit it, so I have to manually strip the taller components by hand so it has enough clearance.

Along my journey, I've fried the entire board (So far only once) by accidentally shorting some of the 40-pin header to test my kernel module - By and large, the nonchalance of a primarily web based software developer... So, the second board has arrived and I've made enough progress to write another article. 

## The Linux Kernel

I've come into this project with a little knowledge of kernel module compilation, but have never had to write anything quite as drastic at this - let's just say it has taken much longer than expected, but lot's has been learned...

As per my stringent requirements, I want some fast interrupt based kernel level hardware interaction, I could have done this in python and flailed a stray cpu in a loop polling the device, but that's just poor form. In order to achieve this, several pieces need to come together: 

A kernel module containing
  1. Device driver to access gpio and interrupts when a state has changed (button presses)
  2. Device driver to access i2c so that enough interruptible inputs can coexist (Current PCB does not support 16 GPIOs simultaneously).
  3. Device driver to access HID (that is Human Interface Device) that represents a keyboard so that I can plug in the keyboard to external USB ports providing power and peripheral to make input device.
  4. Device driver to access internal input events (Simple input device) for when I dont want the keyboard to act as a preipheral.

### Device Tree Overlays

Much to my frustration while writing the kernel driver, I could not for the life of me get the i2c device to probe. After some research, I discovered that you need to write an overlay in order to communicate with the hardware that you require for the device. As I've never written or even seen these before, it was a little daunting... Luckily, someone had written support for the MCP23017 for the reaspberry pi, so I was able to adapt it based on the NanoPi's addresses.

```dts
/dts-v1/;

/ {
	compatible = "rockchip,rk3399";
  
  fragment@0 {
		target-path = "/i2c@ff120000";

		__overlay__ {
      status = "okay"; 
    };
  };

  fragment@1 {
    target-path = "/pinctrl/gpio0@ff720000";

    __overlay__ {
      planck_pins: planck_pins {
        rockchip,pins = <32>;
        rockchip,function = <0>;
      };
    };
  };

	fragment@2 {
		target-path = "/i2c@ff120000";

		__overlay__ {
      #address-cells = <1>;
      #size-cells = <0>;

      planck: planck@27 {
        compatible = "planck";
        reg = <0x27>;
        gpio-controller;
        #gpio-cells = <2>;
        #interrupt-cells = <2>;
        interrupt-parent-path = "/pinctrl/gpio0@ff720000";
        interrupts = <32 2>;
        interrupt-controller;
        microchip,irq-mirror;

        status = "okay";
      };
		};
	};
};
```

In order to find those hex addresses, I had to decompile the overlay from [Armbian](https://www.armbian.com) for my board using:

```sh
$ dtc -I dtb -O dts -o device-tree.dts /boot/dtb/rockchip/rk3399-nanopi4-rev01.dtb
```

Have a glance through device-tree.dts for the i2c interface / device you want to bind your overlay too.
In my case, we needed to bind:
  - i2c interface for the MCP23017 port expander
  - A gpio pin for interrupts from the i2c device

**Note**: the `compatible = "planck"` name should line up with the device name you are setting up in your kernel module.

## The Kernel Module

There are several logical parts to the driver, before we get started on the code, lets take a second to figure out what the flow will be, from button press to registering a key press.

![linux kernel flow](http://simonwillshire.com/public/images/planck-flow.svg)

So, a button grounds two inputs (one for the X axis, the other for the Y). This fires an interrupt to gpio32, which triggers a full read of the interrupt register (MCP23017_INTCONA and B) via i2c. Once read, we need to check the difference between the last state and the new state. The difference raises an input event according to the X and Y mapping of a keycode. Hey presto, an internal keyboard driver.

### I2C Device

The basics, we need write a series of register bytes in order to setup the MCP23017 in the way we want to use it. In my use case, I will be mirroring both banks (A and B), so that if any bank changes the interrupt is fired. The change is from high ie 1 to 0, which is called a falling edge interrupt.

In practice, this means:
  - Set MCP23017_IODIRA and B = 0xff ie 1111 1111 (input direction for all)
  - Set MCP23017_GPPUA and B = 0xff so that we use the internal pull up/down resistors so we don't get flappy values
  - Set MCP23017_CONA and B = 0x62
    - 0x40 = mirroring enabled accross bank A and B.
    - 0x20 = disable sequential operations (Usually used for polling, the device increments the addrss automatically).
    - 0x02 = high polarity (default state is high).
  - Set MCP23017_INTENA and B = 0xff to enable interrupts on all inputs 

Once we've set up the device configuration, in order to start using the device, you clear the interrupt state by reading the INTCAP or GPIO registers. 

This looks something like this (trimmed down version) to setting up your i2c driver via i2c_driver.probe:
```c
static int planck_probe(struct i2c_client *client, const struct i2c_device_id *id) {
  struct planck_device *device;
  int ret;

  if(!i2c_check_functionality(client->adapter, 
        I2C_FUNC_SMBUS_BYTE_DATA | I2C_FUNC_SMBUS_WORD_DATA | I2C_FUNC_SMBUS_I2C_BLOCK)){
    return -ENODEV;
  }

  // Setup all inputs
  ret = i2c_write_byte(client, MCP23017_IODIRA, 0xff);
  if(ret != 0) goto i2c_err;
  ret = i2c_write_byte(client, MCP23017_IODIRB, 0xff);
  if(ret != 0) goto i2c_err;

  // Setup pullups
  ret = i2c_write_byte(client, MCP23017_GPPUA, 0xff);
  if(ret != 0) goto i2c_err;
  ret = i2c_write_byte(client, MCP23017_GPPUB, 0xff);
  if(ret != 0) goto i2c_err;
 
  // Setup interrupt mirrorring disable SEQOP, polarity HIGH 
  ret = i2c_write_byte(client, MCP23017_IOCONA, 0x62);
  if(ret != 0) goto i2c_err;
  ret = i2c_write_byte(client, MCP23017_IOCONB, 0x62);
  if(ret != 0) goto i2c_err;

  // Setup interrupt on change
  ret = i2c_write_byte(client, MCP23017_GPINTENA, 0xff);
  if(ret != 0) goto i2c_err;
  ret = i2c_write_byte(client, MCP23017_GPINTENB, 0xff);
  if(ret != 0) goto i2c_err;
  
  goto i2c_ok;

i2c_err:
  return ret;

i2c_ok:
  device = devm_kzalloc(&client->dev, sizeof(struct planck_device), GFP_KERNEL);
  if(device == NULL)
    return -ENOMEM; 
  device->i2c = client;

  // Read the initial state
  i2c_set_clientdata(client, device);
  device->state = planck_read_state(device, MCP23017_INTCAPA);
  return 0;
}
```

#### GPIO

Great, now that we have an i2c device, we want to handle some interrupts, the MCP23017 has an interrupt wire when a change occurs, attach this to a GPIO pin and configure it thus:

```c
static int planck_configure_gpio(struct planck_device *device)
{
  int res = gpio_is_valid(32);
  if(!res){
    return -ENODEV;
  }
  gpio_request(32, "gpio_interrupt");
  gpio_direction_input(32);
  gpio_set_value(32, 0);

  device->irq_number = gpio_to_irq(GPIO_INTERRUPT);
  res = request_irq(device->irq_number, (irq_handler_t)planck_gpio_interrupt, IRQF_TRIGGER_RISING, "planck_interrupt", device);
  
  return res;
}
```
Notice `IRQF_TRIGGER_RISING`, this is not to be confused with the state of the i2c inputs, the MCP23017 will raise the input from low to high. Whereas internally, the MCP23017 inputs are triggering the interrupt from 0 to 1 and 1 to 0 (both ways).

Call this configuration from within your probe event handler. Once complete, you need to set up our marvilous interrupt handler function (aka irq handler)!

We will be introducing a few concepts so that the device does not wait on itself or have race conditions namely through bottom half processing and spin locks. I found some great documentation from IBM [here](https://developer.ibm.com/tutorials/I-tasklets/). 

#### Interrupts, and Bottom half processing

```c
static irq_handler_t planck_gpio_interrupt(unsigned int irq, void *dev_id, struct pt_regs *regs)
{  
  struct planck_device *device = dev_id;
  unsigned long flags;
  int ret;

  if(device == NULL)
    return (irq_handler_t)IRQ_HANDLED;

  // spin lock so we don't clobber our state if another concurrent interrupt happens while queuing
  spin_lock_irqsave(&device->irq_lock, flags);
  
  struct planck_i2c_work* work = (struct planck_i2c_work*)kmalloc(sizeof(struct planck_i2c_work), GFP_KERNEL);
  INIT_WORK((struct work_struct*)work, planck_work_handler);
  work->device = device;
  queue_work(device->read_wq, (struct work_struct*)work); ret = planck_queue_i2c_work(device);

  spin_unlock_irqrestore(&device->irq_lock, flags);

  return (irq_handler_t)IRQ_HANDLED;
}
```

Bueno, this queues some work for later also whilst passing our device pointer. Finally, doing some actual work:

```c
static void planck_work_handler(struct work_struct *w)
{
  int x;
  int y;
  uint16_t state;
  uint16_t last_state;
  unsigned short keycode;
  struct planck_i2c_work *work = container_of(w, struct planck_i2c_work, work);
  struct planck_device *dev = work->device;

  if(dev == NULL){
    printk(KERN_ERR "planck: work_struct has no device!");
    goto finish;
  }

  last_state = dev->state;
  dev->state = ~planck_read_state(dev, MCP23017_INTCAPA);
  state = dev->state;

  // Nothing changed
  if(state == last_state || ~last_state == 0)
    goto finish;

  for(y = 0; y < 4; y++) {
    int currY = (state & (1 << y));
    int lastY = (last_state & (1 << y));

    for(x = 0; x < 12; x++) {
      int currX = (state & (1 << (x + 4)));
      int lastX = (last_state & (1 << (x + 4)));

      // Currently pressed, was not pressed before (Pressed)
      if(!!currX && !!currY && (!lastY || !lastX)){
        keycode = planck_keycodes[(y * 12) + x];
        printk(KERN_DEBUG "planck: pressed (%d, %d), keymap = %d, state = %d, last = %d", x, y, keycode, state, last_state);
        input_event(dev->input, EV_KEY, keycode, 1);
      } 
      // No longer pressed, pressed before (Released)
      else if((!currX || !currY) && (!!lastY && !!lastX)){
        keycode = planck_keycodes[(y * 12) + x];
        printk(KERN_DEBUG "planck: released (%d, %d), keymap = %d, state = %d, last = %d", x, y, keycode, state, last_state);
        input_event(dev->input, EV_KEY, keycode, 0);
      }
    }
  }
  input_sync(dev->input);
  
  // cleanup
finish:
  printk(KERN_DEBUG "planck: cleaning up work_handler");
  kfree((void*)w);
}
```

The above is probably the most logic in the entire kernel module. Lets try to disect it a bit.

- The top section is pulling out or device driver from our work_struct so we can be a useful worker.
- We keep track of the last processed state within `last_state`, put our new state into `dev->state` and negate it (we are operating with polarity high, so we want to check for 0s)
- Loop through our input matrix (12 x 4), checking the bits in last and current state
  - **Format: XXXX XXXX XXXX YYYY** within an unsigned short.
- If we've pressed the state and the old state was not processed before, register a key press.
- Otherwise, check to see if the last state was pressed and the current state is released.

Wait, but there are calls to input_event in there, we have not set it up yet!

#### Internal Input Device

So this would not be a keyboard driver without an input driver, otherwise we are just spamming the kernel log... We want to register a new input device, which outputs `EV_KEY` ie. keyboard events, and we want to map events to keycodes and output those keycodes.

```c
static int planck_init_input(struct planck_device* device)
{
  struct input_dev* input;
  int ret;
  const int num_keycodes = ARRAY_SIZE(planck_keycodes);

  printk(KERN_DEBUG "planck: initialising input...");
  input = input_allocate_device();
  if(input == NULL)
    return -ENOMEM;
  
  input->evbit[0] = BIT_MASK(EV_KEY);
  input->keycode = planck_keycodes;
  input->keycodesize = sizeof(unsigned short); 
  input->keycodemax = num_keycodes;

  for(ret = 0; ret < num_keycodes; ret++)
  {
    if(planck_keycodes[ret] != KEY_RESERVED)
      set_bit(planck_keycodes[ret], input->keybit);
  }

  ret = input_register_device(input);
  if(ret != 0)
  {
    printk(KERN_ERR "planck: unable to register input device, register returned %d\n", ret);
    goto input_err;
  }

  printk(KERN_DEBUG "planck: initialised input device with %d keycodes", num_keycodes);
  device->input = input;

  return ret;

input_err:
  input_unregister_device(input);
  return -ENODEV;
}
```

What's this `planck_keycodes` array? It's the keymapping which allows us to map our input matrix (12x4) to input keycodes.

The kernel module keymapping which I type on (Colemak with 3 layers, only one is below to keep the article short):
```c
#include <linux/input.h>
 /* ,-----------------------------------------------------------------------------------.
  * | Tab  |   Q  |   W  |   F  |   P  |   G  |   J  |   L  |   U  |   Y  |   ;  | Bksp |
  * |------+------+------+------+------+-------------+------+------+------+------+------|
  * | Esc  |   A  |   R  |   S  |   T  |   D  |   H  |   N  |   E  |   I  |   O  |  "   |
  * |------+------+------+------+------+------|------+------+------+------+------+------|
  * | Shift|   Z  |   X  |   C  |   V  |   B  |   K  |   M  |   ,  |   .  |   /  |Enter |
  * |------+------+------+------+------+------+------+------+------+------+------+------|
  * | Ctrl | GUI  | Alt  |WrkSp |Lower |Space | Bksp |Raise | Left | Down |  Up  |Right |
  * `-----------------------------------------------------------------------------------'
  */
static unsigned short planck_keycodes[] = {
  // BASE
  KEY_TAB, KEY_Q, KEY_W, KEY_F, KEY_P, KEY_G, KEY_G, KEY_J, KEY_L, KEY_U, KEY_Y, KEY_SEMICOLON, KEY_DELETE,
  KEY_ESC, KEY_A, KEY_R, KEY_S, KEY_T, KEY_D, KEY_H, KEY_N, KEY_E, KEY_I, KEY_O, KEY_APOSTROPHE, 
  KEY_LEFTSHIFT, KEY_Z, KEY_X, KEY_C, KEY_V, KEY_B, KEY_K, KEY_M, KEY_COMMA, KEY_DOT, KEY_SLASH, KEY_ENTER, 
  KEY_LEFTCTRL, KEY_LEFTMETA, KEY_LEFTALT, 0, 0, KEY_SPACE, KEY_BACKSPACE, 0, KEY_LEFT, KEY_DOWN, KEY_UP, KEY_RIGHT, 
};
```

The kernel module header in order to give some more perspective from the snippets above:
```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/i2c.h>
#include <linux/gpio.h>
#include <linux/input.h>
#include <linux/interrupt.h>
#include <linux/workqueue.h>

#include "planck_keycodes.h"

#define DEVICE_NAME       "planck"
#define GPIO_INTERRUPT    32
#define GPIO_DEBOUNCE     50

#define MCP23017_IODIRA   0x00
#define MCP23017_GPINTENA 0x04
#define MCP23017_DEFVALA  0x06
#define MCP23017_INTCONA  0x08
#define MCP23017_IOCONA   0x0A
#define MCP23017_INTCAPA  0x10
#define MCP23017_GPIOA    0x12
#define MCP23017_IODIRB   0x01
#define MCP23017_IPOLB    0x03
#define MCP23017_GPINTENB 0x05
#define MCP23017_DEFVALB  0x07
#define MCP23017_INTCONB  0x09
#define MCP23017_IOCONB   0x0B
#define MCP23017_INTCAPB  0x11
#define MCP23017_GPIOB    0x13

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Simon Willshire");
MODULE_DESCRIPTION("Planck i2c keyboard driver for mcp23017");
MODULE_VERSION("0.1");

struct planck_device {                // Our keyboard device, we lug this pointer around alot!
  struct workqueue_struct* read_wq;   // We use a queue to do bottom half processing outside of the interrupt context 
  struct i2c_client *i2c;             // The i2c driver client to interface with MCP23017
  spinlock_t irq_lock;                // Interrupt locking mechanism so the driver does not crap itself
  unsigned int irq_number;            // The interrupt number we have created to free up later
  uint16_t state;                     // The last i2c state we processed
  struct input_dev *input;            // The input device to register internal inputs on
};

struct planck_i2c_work {              // When we need to process some work, we need to pass around our device pointer
  struct work_struct work;            
  struct planck_device *device;
};

// Set up our device interface to bind our device tree overlay to
static struct of_device_id planck_ids[] = {{.compatible = DEVICE_NAME},{}};
static const struct i2c_device_id planck_id[] = { {DEVICE_NAME, 0}, {}};
// Declare our interrupt function and work handler for later
static irq_handler_t planck_gpio_interrupt(unsigned int irq, void *dev_id, struct pt_regs *regs);
static void planck_work_handler(struct work_struct *w);

// Ensure our device table is set up to probe
MODULE_DEVICE_TABLE(i2c, planck_id);
```
### Outro

In the next article, I will be assembling the input matrix, implementing multiple keyboard keycode layers (upper, lower). 
After that, we need to define a switch to toggle between internal input handling and external USB HID handling.

The latest full version of the full source can be found at my github [here](https://github.com/tiggilyboo/planck)


