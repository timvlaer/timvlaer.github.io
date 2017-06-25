---
layout: post
title: "Open external links from a PhoneGap app"
description: "Open hyperlinks ouside the webview container"
date: 2016-08-03
tags: [phonegap, cordova]
comments: false
share: true
---


I’m working on a [simple mobile app](https://github.com/timvlaer/ho-gids/) using Cordova/Phonegap. The app contains a hyperlink to a web page. Although this sounds extremely trivial, it took me a couple of hours to get this to work.

For cli-5.2.0 version of PhoneGap, the solution is simple. Create a hyperlink triggering following javascript. Notice the *_system* target.

```javascript
window.open("https://wikipedia.org", "_system");
```

The caveat: you have to adapt the Cordova whitelist to get this to work, see [whitelist documentation](https://cordova.apache.org/docs/en/5.1.1/guide/appdev/whitelist/index.html). Hereafter you find the extract from my config.xml. Obviously you can tweak the intents to your use cases.

```xml
<access origin="*"/>
<plugin name="cordova-plugin-whitelist"/>
<allow-intent href="http://*/*"/>
<allow-intent href="https://*/*"/>
<allow-intent href="tel:*"/>
<allow-intent href="sms:*"/>
<allow-intent href="mailto:*"/>
<allow-intent href="geo:*"/>
<platform name="android">
  <allow-intent href="market:*"/>
</platform>
<platform name="ios">
  <allow-intent href="itms:*"/>
  <allow-intent href="itms-apps:*"/>
</platform>
```

Back in the days, you needed the [InAppBrowser](https://github.com/apache/cordova-plugin-inappbrowser/) plugin for the *_system* target. For current versions, this is not needed.