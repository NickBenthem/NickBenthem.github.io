---
layout: post
title: "TCP Problems with MacOS Catalina"
date: "2020-01-22 18:00:48 -0800"
---

https://github.com/mongodb/docs/blob/master/source/includes/fact-tcp-keepalive-osx.rst

Super frustrating.

`sudo sysctl -w net.inet.tcp.keepidle=30000 net.inet.tcp.keepcnt=8 net.inet.tcp.keepintvl=10000`

![Unrespected Command](/assets/img/tcp-problems-with-macos-catalina/unrespected-command.png)

If you're looking for a primer on TCP, this article by [Cory Klein](http://coryklein.com/tcp/2015/11/25/custom-configuration-of-tcp-socket-keep-alive-timeouts.html) is excellent.
