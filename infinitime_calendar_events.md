# InfiniTime Calendar integration

<img src="/resources/infinitime/timeline_app.gif" width="313"/>

## Introduction

[InfiniTime](https://infinitime.io/) is an open source firmware for the [Pinetime](https://www.pine64.org/pinetime/) ([wiki](https://wiki.pine64.org/wiki/PineTime)) smart watch.

This project is about developping a calendar timeline app for InfiniTime.

The repositories for this project:

- GadgetBridge companion app for android: https://codeberg.org/FederAndInk/Gadgetbridge/src/branch/feature/PineTimeCalendarEvents
- InfiniTime Calendar Timeline app: https://github.com/FederAndInk/InfiniTime/tree/feature/CalendarTimeline

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

TODO

## Storing the events: the FlatLinkedList container

TODO

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

The final UI, as shown in the beginning:

<img src="/resources/infinitime/timeline_app.gif" width="313"/>

## Room for improvements

- [ ] Show event details on click
- [x] Show today/tomorrow
- [ ] Show weekday
- [ ] Use month name
- [ ] Use relative time for event starting in less than ~24h (e.g. "in 23min")
- [ ] Use the calendar color as a background for the event

## Fixes that came out with the project
TODO: GadgetBridge companion app client + calendar blacklist fix(PR)

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


