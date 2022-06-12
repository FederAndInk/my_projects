# InfiniTime Calendar integration

<img src="/resources/infinitime/timeline_app.gif" width="313"/>

## Introduction

[InfiniTime](https://infinitime.io/) is an open source firmware for the [Pinetime](https://www.pine64.org/pinetime/) ([wiki](https://wiki.pine64.org/wiki/PineTime)) smart watch.

This project is about developping a calendar timeline app for InfiniTime.

## Hardware setup

## Development environment setup

After the soldering and hardware setup, it was time to make it work with vscode.

I used [ccls](https://github.com/FederAndInk/ccls) as a C++ language server, had it [setup automatically with cmake](https://github.com/FederAndInk/InfiniTime/commit/fb4fc85828eac01188ad9ac14bbfd79e51da5835) (+ [a fix](https://github.com/FederAndInk/InfiniTime/commit/9915fb04ad957b47f425b9457b5b9544df156e8b))

This commit will make cmake generate a [.ccls](https://github.com/MaskRay/ccls/wiki/Project-Setup#ccls-file) file to point to the right include directories for the toolchain coming directly from the compiler using: `${CMAKE_CXX_COMPILER} -E -Wp,-v -xc++ /dev/null`

This changes are tracked in the [feature/ccls](https://github.com/FederAndInk/InfiniTime/tree/feature/ccls) branch

## Pet project to start

To try and see how to create a new app in InfiniTime, I made a tea timer app:

It is on the [feature/TeaTimer](https://github.com/FederAndInk/InfiniTime/tree/feature/TeaTimer) branch

It was mostly for getting to know the code base and lvgl the graphics library.

## Struggling areas

### sending apparently too much and receiving `BLE_ATT_ERR_INSUFFICIENT_RES = 0x11` error on characteristic write

When sending about 150 bytes of data, `BLE_ATT_ERR_INSUFFICIENT_RES` would be sent back.

It was an issue with not setting the MTU. It was done here: 

https://codeberg.org/FederAndInk/Gadgetbridge/src/commit/56319225d9fd30771aa047269c7003bbf1d49f76/app/src/main/java/nodomain/freeyourgadget/gadgetbridge/service/devices/pinetime/PineTimeJFSupport.java#L596

```java
builder.requestMtu(200);
```

### new characteristic wouldn't show up

when writing new characteristics for the notify event, it would do nothing on the companion app, and only the old characteristics would show up.

It was a problem with the companion app somehow caching the discovered services and characteristics, it was solved by removing the device and readding it on the companion app.


