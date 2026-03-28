---
layout: post
title:  "Cheapest(?) Power Profiler for Developers"
date:   2026-03-28 18:54:29 +0900
categories: development tools
comments: true
---
During the device development cycle, long‑term testing with power‑consumption monitoring is sometimes necessary. To check average current, or to find abnormal power consumption point, or for another reason, that sometimes happens.

Obviously, that job takes a looooong time, and need to prepare proper logging method because I cannot monitor it all the times. Recent power supply products can be connected to PC and log every power voltage/current changes, but quite expensive. Of course, good power supply is every embedded developer's must-have item, but it is too expensive to be dedicated for this job only.

I found fairly useful board during my past works, and it is [Nordic Power Profiler Kit II](https://www.nordicsemi.com/Products/Development-hardware/Power-Profiler-Kit-2). This board is made of their own nRF52840 chip, and mainly for measuring current. But it can also provide power through USB ports up to 1A. This is AFAIK the simplest and cheapest power supplying & measuring solution for small devices. The board is controlled by dedicated [Power Profiler](https://docs.nordicsemi.com/bundle/nrf-connect-ppk/page/index.html) app.

<div align="center">
  <figure>
    <img src="https://mm.digikey.com/Volume0/opasdata/d220001/medias/images/4842/NRF-PPK2.jpg">
    <figcaption>Nordic Power Profiler Kit II</figcaption>
  </figure>
</div>

The size is 86.1 * 54.48 mm, quite small enough. Using app, it can turn on or off the device connected, and measure power currents. The following picture is a screenshot of Power Profiler app, showing current graph for some period of time passed. Pretty cool...:) Additionally, there are 8 GPIO input pins, and more complicated measure triggering seems to be possible. (Personally, some of those pins might be used as output, to control device board.) Anyway, if combined with [Chrome Remote Desktop](https://remotedesktop.google.com/), this is the perfect setup for remotely monitoring and debugging device boards.

<div align="center">
  <figure>
    <img src="https://docs-be.nordicsemi.com/bundle/nrf-connect-ppk/page/screenshots/ppk2_logger_tab.PNG?_LANG=enus">
    <figcaption>Power Profiler app</figcaption>
  </figure>
</div>

While writing this article, I searched internet for the price of PPK2. It is around $100, and may be not the cheapest one but fairly reasonable for the purpose, compared to others.