---
layout: updates/post
title: "Access USB devices on the Web"
description: "TODO"
featured_image: /web/updates/images/2016-03-02-access-usb-devices-on-the-web/web-usb-hero.jpg
published_on: 2016-03-02
updated_on: 2016-03-02
authors:
  - beaufortfrancois
tags:
  - news
  - USB
  - WebUSB
  - IoT
  - Arduino
---

If I tell you plain and simple "USB", there are good chances that you will
immediately think of keyboards, mice, audio, video and storage devices. You're
right but it exists other kinds of Universal Serial Bus (USB) devices out
there.

These non-standardized devices require hardware vendors to write native drivers
and SDKs in order for you (developer) to take advantage of them. Sadly this
native code prevents these devices from being used by the Web. And that's
exactly why the WebUSB API has been created: to **provide an easy way to safely
expose USB device services to the Web**. With this API, hardware manufacturers
will be able to build cross-platform JavaScript SDKs for their devices.

## Before we start

This article assumes you have some basic knowledge of how USB works. If not, I
recommend reading [USB in a NutShell](http://www.beyondlogic.org/usbnutshell).

The official [WebUSB API](https://wicg.github.io/webusb/) draft is hmm... a
draft. That's why the Chrome Team is actively looking for eager developers to
give it a try and send
[feedback on the spec](https://github.com/wicg/webusb/issues) and
[feedback on the implementation](https://bugs.chromium.org/p/chromium/issues/entry?components=Blink%3EUSB).

At the time of writing, the WebUSB API is implemented behind an experimental
flag in Chrome for Windows, Mac OS, Linux and Chrome OS. Go to
`chrome://flags/#enable-experimental-web-platform-features`, enable the
highlighted flag, restart Chrome and you should be good to go!

<img style="width:723px; max-height:205px" src="/web/updates/images/2016-03-02-access-usb-devices-on-the-web/web-usb-flag.png" alt="Web USB Flag highlighted in chrome://flags"/>

## Privacy & Security

### Attacks against USB devices

The WebUSB API does not even try to provide a way for any web page to
connect to arbitrary USB devices. There are plenty of published attacks against
USB devices that makes it unsafe to allow this.

For this reason, a USB device can define a set of origins that are allowed to
connect to it. This is similar to the [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) mechanism.
  

### HTTPS Only

Because this experimental API is a powerful new feature added to the Web,
Chrome aims to make it available only to [secure
contexts](https://w3c.github.io/webappsec/specs/powerfulfeatures/#intro). This
means you'll need to build with TLS in mind.

During development you'll be able to play with WebUSB through
http://localhost by using some tools like the [Chrome Dev
Editor](https://chrome.google.com/webstore/detail/pnoffddplpippgcfjdhbmhkofpnaalpg)
or the handy `python -m SimpleHTTPServer`, but to deploy it on a site you'll
need to have HTTPS setup on your server. I personally enjoy [GitHub
Pages](https://pages.github.com) for demo purposes.
To add HTTPS to your server you'll need to get a TLS certificate and set
it up. Be sure to check out [Security with HTTPS
article](/web/fundamentals/security/)
for best practices there.

### User Gesture Required

As a security feature, getting access to connected USB devices with
`navigator.usb.requestDevice` must be called via a user gesture
like a touch or mouse click.

## Let's start coding

The WebUSB API relies heavily on JavaScript
[Promises](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Promise).
If you're not familiar with them, check out this great
[Promises tutorial](http://www.html5rocks.com/en/tutorials/es6/promises/). One
more thing, `() => {}` are simply ECMAScript 2015 [Arrow
functions](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Functions/Arrow_functions)
-- they have a shorter syntax compared to function expressions and lexically
bind the `this` value.

### Get Access to USB devices

You can either prompt the user to select a single connected USB device with
`navigator.usb.requestDevice` or simply call `navigator.usb.getDevices`
to get a list of all connected USB devices the origin has access to.


The `navigator.usb.requestDevice` function takes a mandatory Object that
defines filters. These filters are used to match any USB device with the given
vendor (`vendorId`) and optionally product (`productId`) identifiers. The
`classCode`, `subclassCode` and `protocolCode` keys can also be defined there
well.

<img style="width:723px; max-height:414px" src="/web/updates/images/2016-03-02-access-usb-devices-on-the-web/usb-device-chooser.png" alt="USB Device Chooser screenshot"/>

For instance, here's how to get access to a connected Arduino device properly
configured to allow the origin:

{% highlight javascript %}
navigator.usb.requestDevice({ filters: [{ vendorId: 0x2341 }] })
.then(device => { 
  console.log(device.guid); // Unique identifier string
})
.catch(error => { console.log(error); });
{% endhighlight %}

I didn't magically come up with this `0x2341` by the way. I simply searched for
the word "Arduino" in the official [List of USB
ID's](http://www.linux-usb.org/usb.ids).

The USB `device` Object returned in the fulfilled promise above has some basic,
yet important information about the device such as the supported USB version,
maximum packet size, vendor and product IDs, the number of possible
configurations the device can have - basically all fields contained in the
[device USB Descriptor](http://www.beyondlogic.org/usbnutshell/usb5.shtml#DeviceDescriptors)

### Receive data over serial

Okay, now let's see how to receive data over serial with an Arduino device. It
shouldn't be that hard.

{% highlight javascript %}
var device;

navigator.usb.requestDevice({ filters: [{ vendorId: 0x2341 }] })
.then(selectedDevice => {
   device = selectedDevice;
   return device.open(); // Begin a session.
 })
.then(() => device.setConfiguration(1)) // Select a configuration for the device.
.then(() => device.claimInterface(2)) // Request exclusive control over interface.
.then(() => device.controlTransferOut({
    'requestType': 'class',
    'recipient': 'interface',
    'request': 0x22,
    'value': 0x01,
    'index': 0x02})) // Issue a control transfer from the host to the device.
.then(() => {
  var textDecoder = new TextDecoder();
  (function readLoop() {
    device.transferIn(5, 64)
    .then(result => {
      console.log('Received: ' + textDecoder.decode(result.data));
      readLoop();
    })
  })();
})
.catch(error => { console.log(error); });
{% endhighlight %}
