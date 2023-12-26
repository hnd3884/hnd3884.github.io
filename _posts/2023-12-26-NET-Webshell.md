---
title: .NET Webshell
date: 2023-12-26 08:51:00 +0700
categories: [Others]
tags: []
---

Webshell is basically the most common way to get RCE on the target. But that also means we would create new files on the server. Unfortunately, if server has any EDR or AV installed on it, the webshell we uploaded would be deleted or an alert would be noticed to administrator. That restriction leads to memory webshell concept. It is a kind of webshell that would be saved in memory, therefore it would works althought original file had been deleted. In java application, for example one deployed on Tomcat server, we can inject memory web shell in Listener, Filter, ... layers, so is it any technique similar to that in .NET?

While doing pentest/finding 0-days with .NET target, there are some cases i faced about webshell. Before go to details, you can quickly setup a lab for testing

## Setup
I used a VMWare to create a IIS Webserver run on Windows Server 2016 to deploy a website on it. You have to add the following roles and features

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/4789a892-76d7-4fea-93ec-2cc55bff7675)

Download and install Webdeploy from [here](https://www.microsoft.com/en-us/download/details.aspx?id=43717) on the server, then go to services to make sure that Web Management Service is running

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/15b81b48-3568-47bd-975e-1aa2456f13fe)

Now in your local machine, create a MVC web application in Visual Studio, then go to build -> Publish [You project name], then fill the form like the following

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/bc23a6ff-cada-47e6-9cae-20acb4107044)

Then save and publish, your website would be deploy to /rcelab in case of above example

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/79e5e913-4328-4d36-87ad-46908d7aca2e)

## Precompiled application
Some of the application applies precompiled deployment, so that all .aspx file on the server would look like this
