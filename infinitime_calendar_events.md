# InfiniTime Calendar integration

<img src="/resources/infinitime/timeline_app_infinisim_final.gif" width="250"/>

## Table of Contents

- [InfiniTime Calendar integration](#infinitime-calendar-integration)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Hardware setup](#hardware-setup)
  - [Development environment setup](#development-environment-setup)
  - [Pet project to start](#pet-project-to-start)
  - [BLE](#ble)
  - [GadgetBridge integration](#gadgetbridge-integration)
  - [Storing the events: the FlatLinkedList container](#storing-the-events-the-flatlinkedlist-container)
  - [UI for a small screen on a watch](#ui-for-a-small-screen-on-a-watch)
    - [The improvements](#the-improvements)
  - [Fixes that came out with the project](#fixes-that-came-out-with-the-project)
  - [Struggling areas](#struggling-areas)
    - [Sending apparently too much and receiving `BLE_ATT_ERR_INSUFFICIENT_RES = 0x11` error on characteristic write](#sending-apparently-too-much-and-receiving-ble_att_err_insufficient_res--0x11-error-on-characteristic-write)
    - [New characteristic wouldn't show up](#new-characteristic-wouldnt-show-up)
    - [Flashing DFU on a sealed PineTime](#flashing-dfu-on-a-sealed-pinetime)

## Introduction

[InfiniTime](https://infinitime.io/) is an open source firmware for the [Pinetime](https://www.pine64.org/pinetime/) ([wiki](https://wiki.pine64.org/wiki/PineTime)) smart watch.

This project is about developping a calendar timeline app for InfiniTime.

The repositories for this project:

- GadgetBridge companion app for android: https://codeberg.org/FederAndInk/Gadgetbridge/src/branch/feature/PineTimeCalendarEvents
- InfiniTime Calendar Timeline app: https://github.com/FederAndInk/InfiniTime/tree/feature/CalendarTimeline
- InfiniSim configured with the Calendar Timeline app: https://github.com/FederAndInk/InfiniSim/tree/feature/CalendarTimeline

For the documentation of the service and app: https://github.com/FederAndInk/InfiniTime/blob/feature/CalendarTimeline/doc/CalendarEventService.md

## Hardware setup

The first issue I ran into was an error when flashing the pinetime.
This process is described in the [wiki](https://wiki.pine64.org/wiki/Reprogramming_the_PineTime).
Unfortunately, I received bad wires that were having shortcircuits in the glue:

<img src="/resources/infinitime/pinetime_borked_glued_wires.png" alt="glued wires" width="80"/>

It was hard to guess as I had weird Openocd outputs stating that it couldn't connect to the target.
When this happens its often due to the wiring not being correct. But after checking multiple times and even with 3 people, I checked with a multimeter.

TL;DR: Always check glued wires with a multimeter.

They were the only wires I had, so I unglued them with a heatgun and took the soldering way. I am not used to soldering, but here is what I learned: don't use the smallest tips, it takes a really long time to apply heat with it. But don't go too hot, especially with this kind of PCB, heat around 10C above the melting point of the solder.

Here is the result:

<img src="/resources/infinitime/wired_pinetime.jpg" alt="glued wires" width="300"/>

It should be durable and more handy than just using contact pins.

## Development environment setup

After the soldering and hardware setup, it was time to make it work with vscode.
The flashing part was already taken care of with custom CMake targets.

I used [ccls](https://github.com/FederAndInk/ccls) as a C++ language server, had it [setup automatically with CMake](https://github.com/FederAndInk/InfiniTime/commit/fb4fc85828eac01188ad9ac14bbfd79e51da5835) (+ [a fix](https://github.com/FederAndInk/InfiniTime/commit/9915fb04ad957b47f425b9457b5b9544df156e8b))

CMake will generate a [.ccls](https://github.com/MaskRay/ccls/wiki/Project-Setup#ccls-file) file to point to the right include directories for the toolchain coming directly from the compiler using: `${CMAKE_CXX_COMPILER} -E -Wp,-v -xc++ /dev/null`

This changes are tracked in the [feature/ccls](https://github.com/FederAndInk/InfiniTime/tree/feature/ccls) branch

To get the logs from the pinetime and enable asserts, I [configured rtt with the cortex-debug extension and added some flags](https://github.com/FederAndInk/InfiniTime/commit/93aa667a771dd4e9bc45f4600d40fe93e638958f)

## Pet project to start

To try and see how to create a new app in InfiniTime, I made a tea timer app:

<img src="/resources/infinitime/teatimer_in_apps.jpg" height="220"/>
<img src="/resources/infinitime/teatimer_ui.jpg" height="220"/>

It is on the [feature/TeaTimer](https://github.com/FederAndInk/InfiniTime/tree/feature/TeaTimer) branch

It was mostly for getting to know the code base and lvgl the graphics library.

## BLE

This project was really great to learn a lot about BLE (bluetooth low energy)

It uses the [nimBLE](https://mynewt.apache.org/latest/tutorials/ble/bleprph/bleprph.html) library from the apache mynewt project.

This project basically adds one service containing 4 characteristics. 2 are for writing addition and deletion of events. 2 more are for notifying the central, the first notify when there are rejected events because there is no more room to store them and the second notify when there is room for the rejected events to be sent back.

## GadgetBridge integration

[GadgetBridge](https://www.gadgetbridge.org/) is an open source android application to connect to IoT devices, it already supports the PineTime.

GadgetBridge had already the structure to handle calendar events, I had only to add the service/characteristics and write the code to communicate those events to the PineTime. For convenience the events are all resent when the PineTime is reconnected because the events could have been removed with a watch restart. Later on I also added the color of the calendar in the event and fixed an issue in GadgetBridge (see [fixes part](#fixes-that-came-out-with-the-project)).

## Storing the events: the FlatLinkedList container

In order to store the event, I made a flat linked list in an array with its
capacity known at compile time.
It was a good choice to insert and remove events from anywhere, while still doing no dynamic allocations.
The container follow the standard library named requirements and has iterator and range support. It has a similar interface to `std::list`

It is also extensively tested with the catch2 v3 library.

This is all detailed in the [documentation of the project](https://github.com/FederAndInk/InfiniTime/blob/feature/CalendarTimeline/doc/CalendarEventService.md)

## UI for a small screen on a watch

The UI was design for the small screen of the watch while still showing a good amount of information without a cluttering feeling.
It uses pages to show the events.

There are 2 events per page and arrows tell if there is a page upward and/or downward.

Events are shown in cards with the date before. 
For each event, the first line is for the title, the second for the location
and the last one is for the start time with the duration in parenthesis.
The date is not repeated if the two events shares it.

Here is a first draft of the UI next to a more advanced one:

<img src="/resources/infinitime/timeline_first_draft.jpg" alt="first draft of the timeline UI" width="250"/>
<img src="/resources/infinitime/timeline_ui_r.jpg" alt="first draft of the timeline UI" width="250"/>

Arrows were added alongside the date and time/duration handling. the today/tomorrow were still in progress though, the second image shows space test for the tomorrow labeling.

The UI - minimum viable product - before the improvements:

<img src="/resources/infinitime/timeline_app.gif" width="313"/>

### The improvements

The Final UI with all the improvements (in InfiniSim):

<img src="/resources/infinitime/timeline_app_infinisim_final.gif" width="313"/>

After getting a minimum viable product I got some ideas for improvements:

- [x] Show event details on click
- [x] Show today/tomorrow
- [x] Show weekday
- [x] Use month name
- [x] Use relative time for event starting in less than ~6h (e.g. "in 23min")
- [x] Show "Ends in" for ongoing events
- [x] Use the calendar color as a background for the event
- [x] Highlight with a border events starting today or in the next 6 hours

## Fixes that came out with the project

This project has allowed me to make some fixes to InfiniTime and gadget bridge

- [An assertion failure in Infinitime](https://github.com/InfiniTimeOrg/InfiniTime/pull/1158)

- [Calendar blacklist UI and backend issue in GadgetBridge](https://codeberg.org/Freeyourgadget/Gadgetbridge/pulls/2686)

## Struggling areas

### Sending apparently too much and receiving `BLE_ATT_ERR_INSUFFICIENT_RES = 0x11` error on characteristic write

When sending about 150 bytes of data, `BLE_ATT_ERR_INSUFFICIENT_RES` would be sent back.

It was an issue with not setting the MTU. It was done here: https://codeberg.org/FederAndInk/Gadgetbridge/src/commit/56319225d9fd30771aa047269c7003bbf1d49f76/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/service/devices/pinetime/PineTimeJFSupport.java#L596

```java
builder.requestMtu(200);
```

### New characteristic wouldn't show up

When writing new characteristics for the notify event, it would do nothing on the companion app, and only the old characteristics would show up.

It was a problem with the companion app somehow caching the discovered services and characteristics, it was solved by removing the device and readding it on the companion app.

### Flashing DFU on a sealed PineTime

After the development on the wired devkit using OpenOcd to do the flashing came the time to update a sealed unit with OTA and DFU. The project was build using the nrf5-sdk version 17.1.0 (latest at the moment). Unfortunately right after the transfert the bootloader wouldn't launch the firmware and just instantly rollback to the previous one. The problem was with the nrf5-sdk, version [15.3.0_59ac345](https://developer.nordicsemi.com/nRF5_SDK/nRF5_SDK_v15.x.x/nRF5_SDK_15.3.0_59ac345.zip) must be used in order for the firmware to launch successfully.
