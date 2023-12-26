---
title: .NET Webshell
date: 2023-12-26 08:51:00 +0700
categories: [Others]
tags: []
---

Webshell is basically the most common way to get RCE on the target. But that also means we would create new files on the server. Unfortunately, if server has any EDR or AV installed on it, the webshell we uploaded would be deleted or an alert would be noticed to administrator. That restriction leads to memory webshell concept. It is a kind of webshell that would be saved in memory, therefore it would works althought original file had been deleted. In java application, for example one deployed on Tomcat server, we can inject memory web shell in Listener, Filter, ... layers, so des
