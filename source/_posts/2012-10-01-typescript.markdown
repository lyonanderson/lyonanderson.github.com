---
layout: post
title: "TypeScript"
date: 2012-10-01 22:14
comments: true
categories: 
---

Microsoft have released a typed superset of JavaScript called [TypeScript](http://www.typescriptlang.org). I spent a sizable portion of my life looking at types and JavaScript, in particular [Inferring types for JavaScript programmes](http://pubs.doc.ic.ac.uk/authors/cla97/). I still find the area interesting, and it's nice to see this new and exciting work. I wonder if TypeScript has a sound type system? I see already that someone on Twitter has [pointed out](https://twitter.com/puffnfresh/statuses/252816990782230528) that arrays are mutable and covariant:

``` javascript 
class A {}

class B extends A {
    foo () {  }
}

class C extends A { }

var b: B[] = [new B()];
var a: A[] = b;
a[0] = new C();

var boom: B = b[0];
boom.foo();  // boom points to an instance of C which has no foo method!
```

When you run the above you'll get an error stating that Object #\<C\> has no method 'foo'. The Java type system also has covariant arrays, but would prevent the assignment on line 11 at runtime:

``` java
public class A {
        public static void main(String args[]) {
         B[] b = new B[2];
         A[] a = b;
         a[0] = new C(); // Runtime exception: Exception in thread "main" java.lang.ArrayStoreException: C
        }
}

class B extends A {}

class C extends A { }
```



