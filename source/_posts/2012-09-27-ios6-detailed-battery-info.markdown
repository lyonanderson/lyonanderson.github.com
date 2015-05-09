---
layout: post
title: "iOS6 Detailed Battery Info"
date: 2012-09-27 11:32
comments: true
categories: 
---

I was poking around in the iOS6 private frameworks over the weekend and found some interesting battery related stuff. I've put together a quick and dirty app which displays most of the interesting stuff. You can get it on github [here](https://github.com/lyonanderson/BatteryInfo). Being ever so fashionable I only support iOS6. Plus, it doesn't work on iOS5!

![Screenshot](/images/IMG_4055.png)

<!-- more -->


Delving deeper, there is a class called OSDBattery which exposes battery metrics including the battery cycle count and serial number.  Looking at the [header dump](https://github.com/nst/iOS-Runtime-Headers/) of iOS6 shows OSDBattery in the FactoryDiags framework. However, on device you won't find this framework. Instead, you need to load the GAIA framework to use OSDBattery. So for example, to get the battery cycle count:

``` objective-c 
NSBundle *b = [NSBundle bundleWithPath:@"/System/Library/PrivateFrameworks/GAIA.framework"];
BOOL success = [b load];
    
if (success) {
        Class OSDBattery = NSClassFromString(@"OSDBattery");
        id powerController = [OSDBattery sharedInstance];
        NSLog(@"Battery Cycle Count %d", [powerController _getBatteryCycleCount]);
}

```

It goes without saying that you can't submit an app using this code! It's a private framework. Don't be an idiot.  I'm not sure how useful all of this is. I'd quite like to run the app on a refurbished device to see what is says. I'm also not clear as to where exactly the count is stored i.e. in the battery or on device. Maybe the [iFixit](http://www.ifixit.com) guys can swap batteries in a device and let me know?

