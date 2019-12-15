+++
date = "2019-12-12T17:36:00+13:00"
title = "Handwired Keyboard Project: Part 3"
description = "Assembly, and a working keyboard"
tags = ["Linux","Kernel","C","Planck","Keyboards","ARM","Matrix"]
topics = ["Blog", "Hardware"]
series = ["Handwired Keyboard Project"]
comments=true
code=true
+++

## Intro

Great it's been 3 months, where the heck is my finished keyboard? Let's get up to speed - I am curently typing this article with the new keyboard, without a case and a wonky ESC switch, but lets start at the beginning, from finishing Part 2:

- We had a "working" interrupt firing and internal key events being issued to the SBC. We did not have external input mode via USB HID.
- Parts came in the mail! I assembled a keyboard matrix by hand!

## Assembly

![Switches](https://i.imgur.com/JPAWByX.jpg)

Components I used to assemble the matrix:

1. 48 Cherry MX Brown switches
2. 48 diodes
3. Planck switch top plate to mount the switches into (Basically an aluminium frame so things don't get floppy)
4. A bunch of wire, and ribbon cable to the MCP23017 i2c board.
5. Lots of patience

Before you get started, read some background on how a input matrix works - There are tonnes of resources online, which is possibly why you ended up here. Heres are some that I used to figure out the wiring:

- [https://www.dribin.org/dave/keyboard/one_html/](https://www.dribin.org/dave/keyboard/one_html/)
- [http://blog.komar.be/how-to-make-a-keyboard-the-matrix/](http://blog.komar.be/how-to-make-a-keyboard-the-matrix/) 
- [http://www.masterzen.fr/2018/12/22/handwired-keyboard-build-log-part-2/](http://www.masterzen.fr/2018/12/22/handwired-keyboard-build-log-part-2/)

### Diode Wrapping

![Diodes](https://i.imgur.com/9500T3e.jpg)

Pre-bend the non-banded side of all diodese on a right angle surface (If they come in a paper tied set). Use a tiny screw driver and lots of determination to bend a small loop on each bent end. These loops will be dropped onto the bottom right pin and solderded (As pictured). bottom right pin and solderded (As pictured above).  

Once complete on all pins, reflect on youre one person assembly line and take a break...

### The Rows

![Rows](https://i.imgur.com/64MpOp6.jpg)

We need a way to either solder (n-1) * (m-1) bits of wire without going insane. My approach was to use 4 wires which make up the entire row, and strip the insulation bare at each diode arm. Once stripped, wrap the banded side of the diode arm around the exposed wire and solder the joint at the twist.

### The columns

![Columns](https://i.imgur.com/1Lq8XKl.jpg)

Using the similar approach as above, maintaining your sanity - we need to solder a further n * (m - 1) wires. This time, I decided to strip the entirety of the insulation (Using full core wire, so you don't have to twist and curse at it all too much). Using the remaining pole on the Cherry switches, wrap the column wire around each, and blob a bit of solder.

### i2c / Ribbon

![Ribbon](https://i.imgur.com/ZBo8qN9.jpg)

Great, now we've got our input matrix, we need to attach our 12 rows and 4 columns to our i2c extender (in my case the MCP23017). I picked up some 16 wire ribbon cable, and fed the components under the columns and between the rows to attach to each row and column. Once complete, pull back and solder each wire to the MCP23017. I used the following mapping in order to match up with my kernel modules expectations.

![Ribbon Routing](https://i.imgur.com/ltMzcM9.jpg)


#### MCP23017 Pin Mapping

This is my pin mapping used for the i2c board, this is based on my implementation in my kernel module driver. It could be whatever you like...

```
C uint16_t: YYYY XXXX XXXX XXXX

PA0: Column, X = 12
PA1: Column, X = 11
PA2: Column, X = 10
PA3: Column, X = 9
PA4: Column, X = 8
PA5: Column, X = 7
PA6: Column, X = 6
PA7: Column, X = 5

PB0: Column, X = 4
PB1: Column, X = 3
PB2: Column, X = 2
PB3: Column, X = 1
PB4: Row, Y = 1
PB5: Row, Y = 2
PB6: Row, Y = 3
PB7: Row, Y = 4
```

## Kernel Module Updates 

### Complications and Assumptions

In Part 2, we went about setting up an interrupt which was triggering a workqueue job. This job then read the i2c state, compares the last state with the current state. The difference of these two states gives us our 0 => 1 pressed and 1 => 0 released states, register those inputs based on where it lay in the keymapping and away you go riding into the sunset! Except no, some assumptions were made...

The way the key matrix works, you need to lower a single i2c row, then inspect if any columns change. However, once the row changes, any previous column states are wiped out (Something that I did not realise). So being stubborn and not wanting to write a m * n 2 dimensional loop, I set about constructing something awefully complicated...

**Workflow:**

  1. Have a write work queue (write_wq), which purely pulls the row down to ground, giving us a 0 state.
    - The next row is then queued, we do a wee modulus division and rotate between rows 1-4 for each execution, this gives us row polling.
  2. Have a read work queue (read_wq), which is queued when a column interrupt happens
    - A row is active low (0), and a column comes from 1 to active low (0) causing an irq, and queues it!

But wait, once we fire an interrupt, the row we came from has a completely different column state (If you hold keys between row polls, which in most cases its very difficult not to), so we need to store all of our previous row states.

Cool, so we have row states, and we only interrupt on columns... Wait all this is needlessly complicated, I've been at this forever, let's just get it working with the simple method, and figure it out from there!

### The Simple Method

```sh
git checkout -b laymans
```

**Street Pseudo-code:**

> Hey man, I just want to loop all rows:
> Ok, now Write my row to i2c
> Cool, what do my columns look like?
>    (Press?)    Ah, this column has changed, its now low
>    (Release?)  Oh, but this column has changed its back to high again.

And that's basically it. Theres some funky layer handling and mode changing in there, but in essense I wrote this in one afternoon and it was working! If my pseudo-code is too streety for you, check out the actual kernel module at the [planck repo](https://github.com/tiggilyboo/planck).

### External Mode

This keyboard is good and all, but the biggest and coolest feature is that its running on a SBC which is running a full linux operating system. But what about using it as a normal keyboard? To do this we need to use something in the linux kernel called a [usb gadget device](https://kernel.org/doc/html/v4.13/driver-api/usb/gadget.html), and with this device register a keyboard and send reports to it.

This portion was probably the most difficult, because there are not too many resources online which describe how to write a HID keyboard driver with both internal and external drivers rolled into one. Luckily, with the popularity of the Raspberry Pi, and using it as a gadget device for sharing serial tty, files or networking, I had a starting point.

- Define a platform device which describes our usb functions, in this case our HID report data identifying it as a keyboard!
- Define a platform driver and usb composite driver which probe, bind and remove when a device is connected and disconnected.
- Define a usb function (in usb composite land) which describes our device as OTG
- When we bind the usb device, configure if to add our usb function
- When we want to process an external input (a key was pressed for ex.), we write to our HID gadget that is bound with our report

For the current driver code, check out the planck_hid.h file in the [planck github repo](https://github.com/tiggilyboo/planck)

### HID Input Reports

Once you've got a device up and running, in order to communicate to the host device that an input event has happened, you need to send input reports. Theres a USB spec which describes the input reports [here](https://www.usb.org/sites/default/files/documents/hid1_11.pdf).

But I'll give a quick little overview: We can only send a maximum of 8 bytes per report, each report can send key modifiers (Any combination of all Controls, Shifts, Alts, Metas), and 6 other report values.

```
[0, 0, 0, 0, 0, 0, 0, 0]    = Everything released, empty report
[0, 0x02, 0, 0, 0, 0 0, 0]  = Left shift pressed, everything else released
[0, 0x06, 0x3A, 0, 0, 0, 0, 0] = Left shift and left control pressed, and the letter 'A' is pressed 
```

#### Modifier Translation

As above, we can send all modifiers in a single byte, this is done via bitwise operations. This means we can have any combination of:

Left Control, Right Control,
Left Shift, Right Shift,
Left Alt, Right Alt,
Left Meta, Right Meta

Conveniently 8 corresponding to each bit in a byte... 
But when we look at the HID report docs, we see that, hey Left Control = `0xE0`?! Actually all of my modifiers are contained within `0xE0` to `0xE7`, so we translate and store any held modifiers and release them when we next send our external HID report:

```c
if(keycode >= 0xE0 && keycode <= 0xE7)
{
  if(pressed)
    planck_hid_report[0] |= (1 << (keycode - 0xE0));
  else
    planck_hid_report[0] &= ~(1 << (keycode - 0xE0));

  goto finished;
}
 ```

Notice how we `goto finished` here instead of sending the HID report, this is so that we can send lets say `CTRL-A` and the input is treated as one report, otherwise we would send `CTRL`, then `A`. Which is not what we expect.

### Outro

![Clearance](https://i.imgur.com/wrzl7C3.jpg)

We are not quite at the final product yet - But as I said in the intro, I've actually written the entire article with this keyboard, which turns out to be not a small feat! 

#### What's left? 

- Still  would like to revisit the non-layman approach and use interrupts, guage what the performance differences are. Though currently I'm stalling the worker to write each row and read all columns once every Jiffy (Which is roughly equates to ~16ms). But it looks like most response times on keyboards today seem to be around 50ms scan times, so I may have already overkilled it.
- Put it in a case, this may mean stripping the USB and Ethernet ports to fit.
  - The case still needs to have holes drilled to access the different remaining ports (Audio, HDMI and USB-C)
- Hook up an NVMe SSD instead of using a SDHC card which is hamstringing write i/o to 16MB/s tops...
- Chuck in a battery if it fits?!

![Keycaps](https://i.imgur.com/YoIgQTx.jpg)

More to come!
