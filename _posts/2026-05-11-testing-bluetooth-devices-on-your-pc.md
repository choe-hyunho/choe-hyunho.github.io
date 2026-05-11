---
layout: post
title:  "Testing Bluetooth devices on your PC"
date:   2026-05-11 11:45:25 +0900
categories: development tools
comments: true
---
When developing Bluetooth devices, we need two endpoints to communicate each other. Sometimes, already have two development boards which is capable of Bluetooth, and build both Bluetooth central & peripheral together. But other times, may have only one Bluetooth device and need to test with something else. In this post, I will show how to test Bluetooth functions with your PC. This can be done with [Web Bluetooth API](https://developers.google.com/web/updates/2015/07/interact-with-ble-devices-on-the-web).

[Web Bluetooth API](https://developers.google.com/web/updates/2015/07/interact-with-ble-devices-on-the-web) is part of Chrome hardware API, and provides a method to handling Bluetooth devices. It is designed for [Chrome Web Browser](https://www.google.com/chrome/), but works on [Edge Web Browser](https://www.microsoft.com/edge/) and [Mozilla Firefox](https://www.firefox.com/) as well, as I tested. You can find various samples from [here](https://googlechrome.github.io/samples/web-bluetooth/index.html), and see source codes at [here](https://github.com/googlechrome/samples/blob/gh-pages/web-bluetooth/). There are lots of samples showing various functions which PC is used as BLE central. I will introduce some of them to test Bluetooth devices under development. The following contents are assuming that you are already have some knowledge about Bluetooth and Bluetooth LE concepts.

When I develop Bluetooth devices, it is mostly BLE peripherals, which have some useful functions, and provide data if another BLE central requests them. In BLE world, BLE peripheral is actually server role, but works as "slave", and BLE central is actually client role, but works as "master". So, all the Bluetooth communications are started from BLE central sending request to BLE peripheral. Of course, before starting communication, BLE peripherals should register themselves as specific services. More precisely, BLE peripherals should advertise themselves periodically to everyone, but this is not the subject of this article, so skip it for now. This article will assume that there is a working (well or buggy) BLE peripheral for specific service exists already, and will try to communicate to it via PC web browser.

# Scanning for the target device

First, need to find target device via BLE scanning. In Web API, [requestDevice()](https://webbluetoothcg.github.io/web-bluetooth/#dom-bluetooth-requestdevice) function is used for scan. Because BLE peripherals are advertising several information periodically, can find specific BLE device by assigning searching criteria in options argument. For example, You can find the device its name is "Tracker" or "Trkr" by the following code.

```Javascript
    const device = await navigator.bluetooth.requestDevice(
        {
            filters: [
                {name: "Trkr"},
                {name: "Tracker"}
            ]
        }
    );
```

Or, can find the devices which have names started with "Trkr" like follows.

```Javascript
    const device = await navigator.bluetooth.requestDevice(
        {
            filters: [
                {namePrefix: "Trkr"},
            ]
        }
    );
```

But most important criteria for my purpose is scanning for the specific GATT service. Every BLE peripheral has one or more GATT service inside, and part of them are listed in its advertisement packet. I can also scan and find devices which provide specific service with the requestDevice() function.

Before proceeding, let's review about GATT service. All services provided by BLE is called as 'GATT service', and represented as one UUID (long form), and 16-bit number (short form, part of its UUID). The advertisement packet may include 'GATT service list' for device, and providing services are included in this list, we can find devices which can provide specific service using this feature. [Bluetooth SIG](https://www.bluetooth.com/) provides the list of standard service [here](https://bitbucket.org/bluetooth-SIG/public/src/main/assigned_numbers/uuids/service_uuids.yaml). And lots of Bluetooth device vendors has their own proprietary services which is not listed in this list, but as long as you know the UUID of it, you can scan and find specific device, too. In addition, Web Bluetooth API also has pre-declared service name for the standard GATT service [here](https://github.com/WebBluetoothCG/registries/blob/master/gatt_assigned_services.txt).

This is the example of device scanning with pre-declared service name.

```Javascript
    const device = await navigator.bluetooth.requestDevice(
        {
            filters: [
                {services: ["current_time"]}
            ]
        }
    );
```

And this is the example of device scanning with known, but not pre-declared service UUID.

```Javascript
    const device = await navigator.bluetooth.requestDevice(
        {
            filters: [
                {services: [0x1847]}
            ]
        }
    );
```

Running requestDevice() function, browser will show pop up scanning Bluetooth devices, and if devices found matching criteria, device will be listed in pop up window.

# Getting & Setting GATT characteristics

After finding proper device, the next step is quite straightforward, but tedious. Read the specification for the service, and get or set proper GATT characteristics which required. The following code shows how to read current time characteristic and write back current system datetime to device. This code is working on the device which supports [CTS(Current Time Service)](https://www.bluetooth.com/specifications/specs/current-time-service-1-1/), which has UUID=0x1805. Each GATT characteristic also has unique UUID assigned, and is listed [here](https://bitbucket.org/bluetooth-SIG/public/src/main/assigned_numbers/uuids/characteristic_uuids.yaml). In this time, the characteristic is 'current time', and its UUID is 0x2A2B. And again, for Web Bluetooth API, pre-declared characteristic list is [here](https://github.com/WebBluetoothCG/registries/blob/master/gatt_assigned_characteristics.txt).

```Javascript
...
    device.addEventListener('gattserverdisconnected', onDisconnect);
    //Connecting to GATT Server...
    const server = await device.gatt.connect();
    //Getting Current Time Service...
    const service = await server.getPrimaryService("current_time");

    //Getting Current Time Characteristic...
    currentTimeCharacteristic = await service.getCharacteristic("current_time");

    //Reading Current Time...
    const currentTimeValue = await currentTimeCharacteristic.readValue();
    let year = currentTimeValue.getUint16(0, 1);
    // CTS Year
    let month = currentTimeValue.getUint8(2, 1);
    // CTS Month
    let mday = currentTimeValue.getUint8(3, 1);
    // CTS Day
    let hour = currentTimeValue.getUint8(4, 1);
    // CTS Hour
    let minute = currentTimeValue.getUint8(5, 1);
    // CTS Minute
    let second = currentTimeValue.getUint8(6, 1);
    // CTS Second
    let wday = currentTimeValue.getUint8(7, 1);
    // CTS Week
    let fraction = currentTimeValue.getUint8(8, 1);
    // CTS fraction
    let reason = currentTimeValue.getUint8(9, 1);
    // CTS reason

    //Writing Current Time...
    let now = new Date();
    let writeCurrentTimeCommand = new Uint8Array(10);
    let offset = 0;
    writeCurrentTimeCommand[offset++] = now.getFullYear() & 0xFF;
    writeCurrentTimeCommand[offset++] = (now.getFullYear() >> 8) & 0xFF;
    writeCurrentTimeCommand[offset++] = now.getMonth();
    writeCurrentTimeCommand[offset++] = now.getDate();
    writeCurrentTimeCommand[offset++] = now.getHours();
    writeCurrentTimeCommand[offset++] = now.getMinutes();
    writeCurrentTimeCommand[offset++] = now.getSeconds();
    writeCurrentTimeCommand[offset++] = now.getDay() == 0 ? 7 : now.getDay();
    writeCurrentTimeCommand[offset++] = ((now.valueOf() % 1000) * 256 / 1000) | 0;
    writeCurrentTimeCommand[offset++] = 0x01; // Reason: Manual Update
    await currentTimeCharacteristic.writeValue(writeCurrentTimeCommand);
    // Current Time Write Request sent...

    await sleep(10000);

    if (device.gatt.connected) {
      //Disconnecting Bluetooth Device...
      device.gatt.disconnect();
    }
...

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function onDisconnect(event) {
  // Object event.target is Bluetooth Device getting disconnected.
}

```

As we can see in above, 1) read proper characteristic, 2) parse the characteristic and 3) put proper values to write packet, 4) send packet to device. That's all. This is the basic BLE communication. But to parse characteristics and compose packet with proper value, we need to refer to the specification of each service and separate packet byte by byte, or assemble packet byte by byte. At first glance, Bluetooth specifications seems somewhat confusing and hard to find matching information. For recent specs, all the related information is in the one specification document, but for older specs, need to refer to several documents, like [Assigned Numbers](https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Assigned_Numbers/out/en/Assigned_Numbers.pdf), [GATT Specification Supplement](https://btprodspecificationrefs.blob.core.windows.net/gatt-specification-supplement/GATT_Specification_Supplement.pdf), and each service or profile specification. For example, about the data format of 'Current Time' or other characteristic, you must see [GATT Specification Supplement](https://btprodspecificationrefs.blob.core.windows.net/gatt-specification-supplement/GATT_Specification_Supplement.pdf), for UUID of this characteristic, should refer to [Assigned Numbers](https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Assigned_Numbers/out/en/Assigned_Numbers.pdf), and to see the whole Current Time Service specification, should check [CTS(Current Time Service)](https://www.bluetooth.com/specifications/specs/current-time-service-1-1/).

# Receiving Notification

Some GATT characteristic can be configured to notify changes. For some GATT services, special characteristic has defined for just notification. This kind of characteristic is called as 'control characteristic'. It is used to control service functions. If some value is written to control characteristic by specified way, BLE central can notify BLE peripheral for controlling how to operate. And BLE peripheral can notify BLE central for data change, buffer filling, or other device events by notification on control characteristic.

Right after get GATT characteristic, can register notification for that characteristic like below.

```Javascript
    //Getting Current Time Characteristic...
    currentTimeCharacteristic = await service.getCharacteristic("current_time");
    //Registering Indication for Current Time Characteristic...
    await currentTimeCharacteristic.startNotifications();
    //Current Time Notification Started
    currentTimeCharacteristic.addEventListener('characteristicvaluechanged', handleNotification);
...
function handleNotification(event) {
  let value = event.target.value;
  let a = [];
  // Convert raw data bytes to hex values jsust for the sake of showing something.
  // In the "real" world, you'd use data.getUint8, data.getUint16 or even
  // TextDecoder to process raw data bytes.
  for (let i = 0; i < value.byteLength; i++) {
    a.push('0x' + ('00' + value.getUint8(i).toString(16)).slice(-2));
  }
  // Current Time Notification
}
```

'Current Time' characteristic is not control characteristic, and it has the current time value inside. So, in this case, this notification will be fired immediately after I wrote value to this characteristic, and its value will be the same value with the one I wrote to. Anyway, if I add this code into the previous one, can confirm the notification is properly fired or not. For the simple service like CTS, this is quite enough.

# More examples

I made my [test page for CTS](https://github.com/choe-hyunho/miscellaneous/blob/main/Bluetooth/Test/cts-test.html) on my repository. In this page, you can scan CTS device, connect to it, and set device time. It was mainly made for testing BLE function, but may be useful because fairly a large number of BLE devices are actually supporting CTS.

Another similar time-synchronizing service is [DTS(Device Time Service)](https://www.bluetooth.com/specifications/specs/device-time-service-1-0/). It is providing similar service to CTS, but has more sophisticated and complex features. You can test it [here](https://github.com/choe-hyunho/miscellaneous/blob/main/Bluetooth/Test/cts-test.html), but need DTS support device.

And one another sample is about [PAMS(Physical Activity Monitor Service)](https://www.bluetooth.com/specifications/specs/pams-1-0/). This service is relatively new in BLE standard, and providing multiple measurement values about human physical activities including step count, heart rate, and respiration rate, etc. Comparing CTS and DTS, PAMS is much more complex because not only it has multiple characteristics, but also it should manage instant & session data inside device. I have made [this test page](https://github.com/choe-hyunho/miscellaneous/blob/main/Bluetooth/Test/pams-test.html) during my PAMS device development. It may looks complex, but quite helpful to see how to handling multiple notifications and parse data.

All the above samples are borrowed from [Web Bluetooth API samples](https://googlechrome.github.io/samples/web-bluetooth/index.html), and I just removed/cleaned unnecessary codes. Links for the original samples & sources are still remained inside pages.
