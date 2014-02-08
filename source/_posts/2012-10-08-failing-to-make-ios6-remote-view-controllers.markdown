---
layout: post
title: "(Failing to) Make iOS6 Remote View Controllers"
date: 2012-10-08 14:47
comments: true
categories: 
---

I've been following with great interest Ole Begemann's [research](http://oleb.net/blog/2012/10/remote-view-controllers-in-ios-6/) into remote view controllers in iOS6. I wanted to look at the problem from the other end. I want to make my own remote view controllers. Clearly, we'll be using private APIs and therefore, none of this can make it into your apps for the store. I'll say up front I was not able to get this working, but I've found some interesting things out.

<!-- more -->


As you'd expect there are two ends to the problem.  At one end a  service which exports some stuff, at the other end a client. The client requests a remote view controller, from a service provider, and is given an instance of a \_UIRemoteViewController which it can present.  It would appear that Apple have wrapped up all the XPC heavy lifting into the \_UIRemoteViewController and associated \_UIViewService classes. 

I think I've got the client end working, in as much as I can bring up Apple built in remote controllers using the following technique:

``` objective-c
- (IBAction)openChrisController:(id)sender {
    Class UIRemoteViewController = NSClassFromString(@"MFMailComposeRemoteViewController");
    

    if (UIRemoteViewController) {
        [UIRemoteViewController requestViewController:@"ComposeServiceRemoteViewController" fromServiceWithBundleIdentifier:@"com.apple.MailCompositionService"  connectionHandler:^(UIViewController *remoteViewController, id error) {
            
            NSLog(@"Arg 1 %@, Arg 2 %@", remoteViewController, error);
            [self presentViewController:remoteViewController animated:YES completion:^{ }];
        }];
       
    }
}
```
This will bring up the standard mail composer. All good. Now let's look at making our own service.  As Ole pointed out, Apple provide hidden applications/services (via SBAppTags in the Info.plist) which are started when a remote view controller is needed. So if you run the above code you'll see a process called MailCompositionService is started (if it's not already there). If we look inside the app bundle for MailCompositionService you'll see some interesting keys:

![](/images/mailComposerInfoList.png)

It would appear as though SBMachServices defines the name of the XPC service we are offering. The budle indetifier matches the name used in the call to [UIRemoteViewController requestViewController:connectionHandler:].  We can now create an application of our own and add these keys to our Info.plist. I created an app called TestRemote and used com.electriclabs.TestRemote as the bundle name.  The app has nothing but a blank view to start with. If you start the app and look in the system console you'll see this error:

```
08/10/2012 17:38:39.736 backboardd[10007]: Ignoring info dictionary key SBMachServices since com.electriclabs.TestRemote is not a system app
```

More on this later... I now wanted to understand what MailCompositionService does when it's started by iOS. To do this I used dtrace with the following program:

```
objc$1:_UIViewService*::entry
{
ustack();
}

```
As you may know dtrace is very chatty, but after spending a while I could see a rough pattern of _UIViewService method calls:

1. Create an intance of _UIViewServiceSessionManager
2. Call _startListener on the instance of _UIViewServiceSessionManager
3. Create an instance of _UIViewServiceXPCListener with constructor initWithName:connectionHandler:

There were other XPC calls but they appeared to sit underneath the UIViewService calls. In my TestApp I added the following to didFinishLaunchingWithOptions:

``` objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //....

    NSBundle *b = [NSBundle bundleWithPath:@"/System/Library/Frameworks/Social.framework"];
    [b load];
    
    Class _UIViewServiceSessionManager = NSClassFromString(@"_UIViewServiceSessionManager");

    self.serviceManager = [[_UIViewServiceSessionManager alloc] _init];
   
    [self.serviceManager _startListener];

   //....
}

```

Now I needed the parameters to the method call [_UIViewServiceXPCListener initWithName:connectionHandler:]. To do this I invoked the services of lldb. Get MailComposistionService started by running the code at the start and then attach lldb:

```
 (lldb) process attach -n MailCompositionService
 (lldb) continue
```

To set a breakpoint in a private method you need to get its address. You can dump the symbols as follows:

```
(lldb) target modules dump symtab
```

From the output search for '[_UIViewServiceXPCListener initWithName:connectionHandler', you should see a line like this:

```
[26153]  26153     Code         0x0000000000596ae3 0x00ac9ae3 0x000000000000023d 0x000e0000 -[_UIViewServiceXPCListener initWithName:connectionHandler:]
```
The important bit is the address - 0x00ac9ae3. We can now set a breakpoint:

```
(lldb)  breakpoint set -a 0x00ac9ae3
```
Now bring up the mail composer again, and you should see our breakpoint is hit:

```
Process 10859 stopped
* thread #1: tid = 0x1c03, 0x00ac9ae3 UIKit`-[_UIViewServiceXPCListener initWithName:connectionHandler:], stop reason = breakpoint 1.1
    frame #0: 0x00ac9ae3 UIKit`-[_UIViewServiceXPCListener initWithName:connectionHandler:]
UIKit`-[_UIViewServiceXPCListener initWithName:connectionHandler:]:
-> 0xac9ae3:  pushl  %ebp
   0xac9ae4:  movl   %esp, %ebp
   0xac9ae6:  pushl  %ebx
   0xac9ae7:  pushl  %edi
```
I found that there were no variables defined, using frame variable, so I had to fall back to registers. There is a great article [here](http://www.clarkcox.com/blog/2009/02/04/inspecting-obj-c-parameters-in-gdb/) on how to do this. Basically on i386 you can get at self using 'po *(id*)($ebp+8)' and the first and second paramters using  po *(id*)($ebp+16) and po *(id*)($ebp+20) respectively. A peek at self showed we are not quite far enough in:

```
(lldb) po *(id*)($ebp+8)
(id) $0 = 0x0a14c7a0 <_UIViewServiceSessionManager: 0xa14c7a0>
```

We want self to be an intance of _UIViewServiceXPCListener. So I stepped in a few more levels using 'thread step-in', until self was _UIViewServiceXPCListener. I could now look at the paramters:

```
(lldb) po *(id*)($ebp+16)
(id) $3 = 0x0a53eb80 com.apple.uikit.viewservice.com.apple.MailCompositionService
(lldb) po *(id*)($ebp+20)
(id) $4 = 0xbfffe840 <__NSStackBlock__: 0xbfffe840>
```

So no massive suprises, \_UIViewServiceXPCListener expects a name which in the case of MailComposistionService is "com.apple.uikit.viewservice.com.apple.MailCompositionService" and a callback block. I'm not sure of the structure of the block, but I'm going to take a punt on it having two parameters. The first being a connection and the second an NSError.

I now extended my didFinishLaunchingWithOptions method of the service TestRemote to look like this:


```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //....

    NSBundle *b = [NSBundle bundleWithPath:@"/System/Library/Frameworks/Social.framework"];
    [b load];
    
    Class _UIViewServiceSessionManager = NSClassFromString(@"_UIViewServiceSessionManager");
    Class _UIViewServiceXPCListener = NSClassFromString(@"_UIViewServiceXPCListener");

    self.serviceManager = [[_UIViewServiceSessionManager alloc] _init];
   
    [self.serviceManager _startListener];

    self.serviceListener = [[_UIViewServiceXPCListener alloc] initWithName:@"com.apple.uikit.viewservice.com.electriclabs.TestRemote" connectionHandler:^(id arg1, id arg2) {
        
        NSLog(@"Arg 1 %@, Arg 2 %@", arg1, arg2);
        
     
    }];


   //....
}
```

So I'm expecting that when my service is needed I'll get a callback on my connectionHandler and from there I can create an instance of our view controller. Communication between service and host will be via XPCObjects. 

I then created a simple client app called TestRemoteClient. We'll need a _UIRemoteViewController subclass 
which I called ELTestRemoteViewController. For simplicity I created a simple view controller with a button which when pressed tries to create an instance of our remote view controller:  


```
- (IBAction)openRemoteController:(id)sender {
    Class UIRemoteViewController = NSClassFromString(@"ELTestRemoteViewController");
   
    if (UIRemoteViewController) {
        [UIRemoteViewController requestViewController:@"TestRemote" fromServiceWithBundleIdentifier:@"com.electriclabs.TestRemote"  connectionHandler:^(id  remoteViewController, id error) {
            
            NSLog(@"Arg 1 %@, Arg 2 %@", remoteViewController, error);
            if (remoteViewController) {
                [self presentViewController:remoteViewController animated:YES completion:^{ }];
            }
            
        }];
        
    }
}
```

Unfortunately, I've never got this to work. I always get an error:

 ```
 2012-10-08 15:15:05.357 RemoteViewControllerTest[5286:c07] Arg 1 (null), Arg 2 Error Domain=_UIViewServiceInterfaceErrorDomain Code=2 "The operation couldn’t be completed. (_UIViewServiceInterfaceErrorDomain error 2.)"
 ```

 Early I said I got an error when running the service app TestRemote - 'Ignoring info dictionary key SBMachServices since com.electriclabs.TestRemote is not a system app'. As a last ditch I tried moving TestRemote into the iOS simulator's main Application folder. Hoping it would be blessed as a system app. Note I had to reset the simulator after moving it so it appeared. Alas, this did not work however, the error did change when running the client app:

 ```
 2012-10-08 14:31:58.635 RemoteViewControllerTest[2224:c07] Arg 1 (null), Arg 2 Error Domain=NSCocoaErrorDomain Code=581952 "The operation couldn’t be completed. (Cocoa error 581952.)"
 ```

 So there we are. I'm a bit stuck. Clearly, there are things missing. I don't know where I am supposed to tell UIKit the name of my view controller on the service side, or the host interface. I expect i'd have to this in the connection handler block like thus:


```
self.serviceListener = [[_UIViewServiceXPCListener alloc] initWithName:@"com.apple.uikit.viewservice.com.electriclabs.TestRemote" connectionHandler:^(id arg1, id arg2) {
        
        NSLog(@"Arg 1 %@, Arg 2 %@", arg1, arg2);

        id serviceProxy = [[_UIViewServiceXPCProxy alloc] initWithConnection:arg1 queue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0) target:[ELViewController new]];
               
        id operator = [_UIViewServiceViewControllerOperator operatorWithRemoteViewControllerProxy:serviceProxy];

        
     
    }];
```

I'd love to get this working. 












