---
layout: post
title: "Xcode can't find headers annoyance"
date: 2012-10-04 15:37
comments: true
categories: 
---

Have you ever been in the situation where all of a sudden Xcode no longer finds certain header files in your project. Another symptom is a project that builds, but there are header import errors when you open a file. You've cleared derived data. You've checked your paths. You've said 10 Hail Marys. Yes, we've all been there. Last time this happened - today - I did try an old trick.

<!-- more -->

First, I entered the following into terminal and restarted Xcode.

```
defaults write com.apple.dt.Xcode IDEIndexingClangInvocationLogLevel 3
```
Fire up console app and delete the derived data for your project. Console will become really shouty with stuff like:

```
04/10/2012 15:46:50.299 Xcode[70288]:  IDEIndexingClangInvocation: ---- X/Redacted.m
```

This is normal. However, you'll probably find lines like these:

```
04/10/2012 15:46:50.321 Xcode[70288]:  IDEIndexingClangInvocation: [diagnostic]: Redacted.h:9:9: fatal error: 'Refacted.h' file not found

```

This is not normal. You'll find that Redacted.h is in fact in your project. What the hell! Now you have to do some thinking. From Console.app, filter for Xcode and grab all of the lines into a text editor. Save the file, we're going to need it.  I called mine xcodelog.txt. Now run the following command on the said file:

```
sed -n "s/.*fatal error: '\(.*\)' file not found.*/\1/p" xcodelog.txt  | sort | uniq
```

You should now have a long list of files. Here is the boring part. You're looking for cyclic imports involving these files. Good luck with that. A common problem is a header file imported by your prefix file, then explicitly imported in another source file. You should not do this. Think carefully about your imports. When you're done turn off the logging and restart Xcode:


```
defaults delete com.apple.dt.Xcode IDEIndexingClangInvocationLogLevel
```

If someone can shed light on why this happens, please get in touch.



