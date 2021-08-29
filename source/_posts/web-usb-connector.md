---
title: 使用 Chrome 浏览器连接 USB 设备
date: 2020-11-20 10:23:14
tags:
	- Javascript
	- 学习
	- Web USB
categories:
	- 编程开发
---

使用 Chrome 浏览器连接 USB 设备的一些总结。

<!-- more -->

## 项目地址

[https://github.com/RinChanNOWWW/web_usb_connector](https://github.com/RinChanNOWWW/web_usb_connector)

## 前言

在公司 mentor 给我说来了一个需求，想要能够通过 Web 配置终端设备的一些设置，让我看看怎么用浏览器通过 USB 连接到设备。

好巧不巧的是，作为一个街机音游狗，接触了各种黑科技的我还真用过这种 Web 配置工具。我使用的一款手台就是通过这样的方式进行配置的，而且作者还将所有代码完全开源了：[PocketVoltex/Software/WebConfig](https://github.com/mon/PocketVoltex/tree/master/Software/WebConfig)，于是我按照作者 mon 的做法将连接 USB 设备并进行配置的代码框架提取了出来。

我也找了 Github 上其他项目看了看，大多数都是连接 Arduino 的，而且都没有 mon 的这个简单好学。

## 编写 Javascript

### 将浏览器提供的 usb API 封装起来

将连接 usb 的方法封装起来赋值给 window 以便全局使用。

```javascript
// usb.js
(function(window, navigator) {
  "use strict";
  var USBWrapperInit = function() {
    var UsbWrapper = new function() {
      this.hasUSB = navigator && navigator.usb;
      if (!this.hasUSB) {
        console.log('Your browser does not support usb')
        return;
      }
      this.connect = () => {
        return navigator.usb
        .requestDevice({filters: []})
        .catch(e => {
          console.log(e);
          return null;
        });
      };
      this.usb = navigator.usb;
    };
    window.USBWrapper = UsbWrapper;
  };
  window.USBWrapperInit = USBWrapperInit;
})(window, navigator);
```

### 编写 Config 类

本应用的逻辑是新建一个 Config 的同时连接到想要配置的 USB 设备，这样就能将一个连接对应到一个配置上。同样的，把 Config 类赋值给 window 以便全局使用。

```javascript
// config.js
(function(window, document) {
    "use strict";
    var device;
    const genLog = function(html) {
        var log = document.getElementById('log');
        log.innerHTML += html + '<br/>';
        log.scrollTop = log.scrollHeight;
    }
    const clearLog = function() {
        var log = document.getElementById('log');
        log.innerHTML = "USB CONFIG TEST"
    }
    class Config {
        constructor() {
            genLog("USB CONFIG TEST")
        }

        connect(selectedDevice) {
            console.log(selectedDevice);
            device = selectedDevice;
            genLog("Opening Device...");
            return device.open()
            .then(() => {
                genLog(`${device.productName}(${device.manufacturerName}) connected.`)
            })
            .catch(err => {
                genLog(err);
                if (device && device.opened) {
                    device.close();
                }
            })
        }

        newConnection() {
            return window.USBWrapper.connect()
            .then(device => {
                if (!device) {
                    return Promise.reject("No device selected");
                }
                this.connect(device);
            })
        }

        close() {
            if (!device || !device.opened) {
                return Promise.reject("Device not opened");
            }
            genLog("Closing Device...");
            return device.close()
            .then(() => {
                genLog('Device closed.')
            })
        }
    };
    window.Config = Config;
    window.genLog = genLog;
    window.clearLog = clearLog;
})(window, document);
```

### 编写 html 页面并加载上面两个脚本

```html
<!DOCTYPE html>
<html>
  <head>
    <title>usb test</title>
    <script type="text/javascript" src="js/usb.js"></script>
    <script type="text/javascript" src="js/config.js"></script>
    <script type="text/javascript">
      window.addEventListener("load", function() {
          USBWrapperInit();
          if(USBWrapper.hasUSB) {              
              config = new Config();
              navigator.usb.getDevices()
              .then(devices => {
                  if(devices.length) {
                      config.connect(devices[0]);
                  }
              });
              navigator.usb.addEventListener('connect', event => {
                  config.connect(event.device);
              });
          }
      });
  </script>
  </head>
  <body>
    <div>
      <div id="log" class="hidden"></div>
      <button id="connectBtn" onclick="config.newConnection()">connect</button>
      <button id="disconnectBtn" onclick="config.close()">disconnect</button>
      <button id="clearBtn" onclick="window.clearLog()">clear log</button>
    </div>
  </body>
</html>
```

## 后续

实现以上的代码，就已经可以通过网页连接到 USB 设备了。可以打开 Chrome 的控制台查看连接后生成的 `USBDevice` 对象，里面有符合 USB 协议的各种信息。

接下来就是如何进行配置操作了。为了完成这一点，还需要设备端编写 USB 驱动程序并运行在设备的内核才行。怎么样传数据，就自己和设备端那边商量就行了。

对于这个 Web 应用，还需要做的就是给 Config 类添加各种配置的方法，其核心是调用

```javascript
var promise = USBDevice.transferIn(endpointNumber, length)
var promise = USBDevice.transferOut(endpointNumber, data)
```

这两个方法，进行 USB 的数据读取与写入操作。只要和设备端协商好了，接下来的就都好说了。

## 写在最后

虽然我是在后台岗位实习，但是是着实干了很多前端的活啊。。。这两天也看了不少 HTML、CSS、Javascript、Typescript 的东西。。。感觉学到的都是一些奇奇怪怪的东西（针对于我想学更多后端的东西而言）。

对于 USB 协议与 Web 更深层次的东西我就不懂了，本项目也只是浅尝辄止。如果上面写的有什么错误或者可以提升的地方，您可以联系我（虽然我觉得应该不会有人看到我的博文），非常感谢。

## 最后的最后

API 文档

- https://developer.mozilla.org/en-US/docs/Web/API/USB

- https://developer.mozilla.org/en-US/docs/Web/API/USBDevice
- and more...