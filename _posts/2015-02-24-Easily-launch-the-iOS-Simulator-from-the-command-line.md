---
title: "Easily launch the iOS Simulator from the command line"
date: 2015-02-24
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

# Easily launch the iOS Simulator from the command line

Launching the iOS Simulator is easy to do when we have a project already loaded in Xcode, Xamarin Studio, or Visual Studio. Simply run the project, and the iOS Simulator will automatically launch.

There are other times, though, when it would be nice to quickly launch the simulator to test something out, or maybe run an existing app. Xcode includes **simctl** to control the simulator, but there is an even easier way.

The [ios-sim](https://github.com/phonegap/ios-sim) project from Phonegap provides a very nice command line interface to control the simulator

> _"The ios-sim tool is a command-line utility that launches an iOS application on the iOS Simulator. This allows for niceties such as automated testing without having to open Xcode." - ios-sim repository_

We can easily install ios-sim using npm:

```console
npm install ios-sim -g  
```

Make sure to explore the API once it's installed. In order to launch a simulator, we can open terminal and run:

```console
ios-sim start --devicetypeid com.apple.CoreSimulator.SimDeviceType.iPhone-6-Plus  
```

To make this even easier, I have created aliases in my ~/.bash_profile file for the common iOS devices I use:

```console
alias iPhone4s="ios-sim start --devicetypeid com.apple.CoreSimulator.SimDeviceType.iPhone-4s"  
alias iPhone5s="ios-sim start --devicetypeid com.apple.CoreSimulator.SimDeviceType.iPhone-5"  
alias iPadAir="ios-sim start --devicetypeid com.apple.CoreSimulator.SimDeviceType.iPad-Air"  
alias iPhone6Plus="ios-sim start --devicetypeid com.apple.CoreSimulator.SimDeviceType.iPhone-6-Plus"  
alias iPhone6="ios-sim start --devicetypeid com.apple.CoreSimulator.SimDeviceType.iPhone-6"  
```

Now, launching a simulator is as easy as typing **iPhone6Plus**

![](/blog/docs/assets/bash.png)
