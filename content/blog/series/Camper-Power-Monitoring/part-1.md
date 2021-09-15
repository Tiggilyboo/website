+++
date = "2021-07-26T18:38:00+01:00"
title = "Camper Power Monitoring: Part 1"
description = ""
tags = ["Camper","E-ink","RS-485","RS-232","Off Grid"]
topics = ["Blog"]
series = ["Camper Power Monitoring"]
comments=true
code=true
+++

# What?

Some of you may know that I have been converting a panel van into a camper over the past 9 months or so. As a part of building this, we opted to install a solar panel, and altenator charging to some leisure batteries.

So how do we go about tracking how much power we are bringing in, and consuming? Here begins my journey to displaying this information and more...

This article was written at the time of: [git repo, at roughly commit 3df2a9a538](https://github.com/Tiggilyboo/vanny-hub/commit/3df2a9a53831a1d05675214de9160693893fbb2e)

## Features

- Be able to see current battery state, current solar panel charge, or altenator charging wattage.
- Estimate at current charge / discharge rate when the batteries will be fully charged or discharged.
- (Non-functional): Don't broadcast segments of this information over bluetooth, or have a display for each of these technologies (A lot of products for these charging technologies come with displays which manage just this singular device, but not multiple).
- Be able to download historical statistics to do simple forcasting of usage.

## Products / Technology

This project will not entirely be useful to a lot of people as I am integrating several bespoke products, however they use very similar underlying communication protocols, namely Modbus.

### Solar 

(RVR40) Renogy Rover 40A: Uses RS-232 over RJ12 (the 6 pin cable, not 4).

### Altenator

(DCC50S) Renogy DC-DC Battery Charger: Uses RS-485 over RJ45

- Connects your engine's AGM battery to your leisure batteries. This particular unit also comes with a MPPT Charge controller, however since we have opted for a higher voltage solar panel array, we are not using it.

### Microcontroller

Raspberry Pi Pico: Has 2 UART ports each of which can be used independently with the above described communication technologies. It also provides enough SPI / I2C for any displays that we wanted to hook up.

### Display

Waveshare 2.9" E-ink display: Just big enough to not be infuriating to look at some a distance (in the van). Also, not too bad a price, and the refresh frequency is not a huge deal, given the monitoring may only update once a minute or so.

## Wiring

![Component Overview Diagram](http://simonwillshire.com/public/images/vanny-components-overview.svg)

Here is an overview of the various components, the wiring itself you may choose which pins to use, or follow off of the pinouts I have defined in ``devices-modbus.h``: 

```c
#define RS485_PORT uart0
#define RS485_BR 9600   // Baudrate
#define RS485_DBITS 8   // Data bits
#define RS485_SBITS 1   // Stop bits (Not 2 as advertised in docs!!)
#define RS485_PIN_TX 0
#define RS485_PIN_RX 1
#define RS485_PIN_RTS 22  // Request to send GPIO pin 

#define RS232_PORT uart1
#define RS232_BR 9600
#define RS232_DBITS 8
#define RS232_SBITS 1
#define RS232_PIN_TX 4
#define RS232_PIN_RX 5
```

And the display is using the following pinout in ``display/display-ws-eink.h``:

```c
#define SPI_PORT      spi1
#define SPI_BR        4000000 // Baudrate

#define SPI_PIN_DC    8
#define SPI_PIN_CS    9
#define SPI_PIN_CLK   10
#define SPI_PIN_DIN   11 // Aka. MOSI
#define SPI_PIN_RST   12
#define SPI_PIN_BSY   13
```

### Communication Pinouts

RJ45 Pinouts for DCC50S and LFP100S:
(Pin numbers starting left to right, with clip facing upwards)

1: +5v
2: A
3: B
4: GND
5-8: Not used

Also of note, since we are using a half duplex (either transmitting or receiving but not both) serial communication method, we are capable of wiring the A, B, GND RJ45 connectors together to interact with the same TTY communication board. In this case I have opted for a MAX485 based board to do the job, they are easy to find and quite accessible.

## Method

So if you have no clue what this 'Modbus' thing is, have a quick squiz of the obligatory [Wikipedia article](https://en.wikipedia.org/wiki/Modbus). The devices that we will be interfacing with use the RTU variant over TTY. 

### Okay, so how do we access this information?

After scouring the internet for quite some time for each of these devices address mappings (We need to know where the information can be accessed on each device in order to consume it, to ultimately display it on the screen), I was able to find the mappings from fellow tinkerers for:

- (RVR40) Renogy Rover Solar Charge Controller: [Tinkerer on github](https://github.com/KyleJamesWalker/renogy_rover/blob/master/reference/ROVER%20MODBUS.pdf) 
- (DCC50S) Renogy DC-DC Battery Charger: [Official documentation](https://support.renogy.com/helpdesk/attachments/35092136259)

However that leads us to the smart lithium batteries (for which this is probably the most important information: what is the state of charge? Otherwise we would require to install a shunt resistor or hall effect sensor... This device has proven to be imppossible to find, which means we have to reverse engineer it!

### Great, how do we reverse engineer the mappings?

Since the Modbus RTU protocol only allocates a single byte (and further restricts it to 247 addresses, see [Modbus RTU frame format](https://en.wikipedia.org/wiki/Modbus#Modbus_RTU_frame_format_(primarily_used_on_asynchronous_serial_data_lines_like_RS-485/EIA-485))), we can brute force each unit address and see if any response comes back. In this case I am querying the holding register space (Function 03).

**Request**

> f7 03 00 01 d5 cA

And sure enough, we get a response back from the very last address 247 (Yes, I incremented from 1...):

**Response**

> f7 83 02 c4 20

Eh, hold on a minute, we only get a length of 5, and why are we getting function code 83 back?

So [apparently](http://www.simplymodbus.ca/exceptions.htm), Modbus has an exception code format. In this case 83 refers to the exception for function 03 (Most significant bit set high). Great, and 02 is the exception code, which means:

**Illegal Data Address**

> The data address received in the query is not an allowable address for the slave. More specifically, the combination of reference number and transfer length is invalid. For a controller with 100 registers, a request with offset 96 and length 4 would succeed, a request with offset 96 and length 5 will generate exception 02.

Fun, so this means we need to try other addresses, but at least we know what the LFP100S unit is operating at! The bad news is that that same RTU frame format allocates 2 bytes, which in unsigned format is a whopping **65,535** registers to check...

Fast forward roughly a few hours of brute forcing addresses (We check the response is length 5 and that the function response is upper bit high, then check the next address...), and with this I have gleaned the following addresses!

- 5000 - 5033
- 5035 - 5052
- 5100 - 5141
- 5200 - 5223

Here is a sample response from one of my units:

**Request**

Address: 0xf7 (247), function 3 (read registers), address 0x1388 (5000), count 0x21 (33 registers), crc

> f7 03 13 88 00 21 15 ea

**Response**

> f7 03 42 00 04 00 21 00 21 00 21 00 21 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

Fantastic, so what do we do with this information?
With a bit of guess work and plugging in various devices to discharge the batteries over time and taking more data points, we can figure out at least a few of the registers' meanings.

For instance, since I had a voltmeter handy, I measured the output of the LFP100S bank, in this case 13.4v. Since all the registers are bytes (no float precision), typically this would be stored in 10 or 100 times that value. So, scanning our register responses for `0x86` in hex.

In this case this corresponds with address `5015`. By making several snapshots of the state and filling these into a spreadsheet, I was able to quickly determine the differences between the states. However, I have yet to track down the overall percentage of state of charge... Hopefully with more data, things will become clearer.

## E-Ink Polling and Updating

Now onto a completely different topic... how often do we wish to update the information, and how often will that information be displayed?

Since we are working with an E-ink display, we do not want to update the screen every time we poll for information (Well we can, but you would be looking at mostly flashing black and white pixels more that actual information...) we need to figure out a way to average our results over periods of time.

For this purpose I have employed a rolling average calculation, which does not require the entire data set, but just the last averaged number. This saves our microcontroller in both memory and overall operations to calculate.

It goes a little like this:

```c
void update_rolling_statistic(uint16_t* avg, uint16_t count, uint16_t new_value) {
  if(stats_rolling_count == 0)
    return;

  *avg = (*avg * (count - 1) + new_value) / count;
}
```

So, we employ this every time we receive a new bit of information from a device, and update a rolling average. Once our display is being updated, we can use this rolling average to display.

We can further store these rolling averages every hour (or a predetermined length of time in constant intervals), and display statistics!

## What's next?

- Continue finding where oh where the holding register address blocks are in the LFP100S', specifically the state of charge in percentage or an Amp Hour remaining figure!
- Show some screenshots of the working unit, showcase some of the UI, and interface choices.

So with that scatter plot of a blog entry, I leave you with a topgear sounding bombshell...

