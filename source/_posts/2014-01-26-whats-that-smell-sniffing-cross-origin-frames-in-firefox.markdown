---
layout: post
title: "What's that smell? Sniffing cross-origin frame content in Firefox"
date: 2014-02-05 10:12:13 -0400
comments: true
categories: browsers crossdomain_theft
---

Reading the blogs of [lcamtuf](http://lcamtuf.blogspot.com) and [Chris Evans](http://scarybeastsecurity.blogspot.ca/) is really what got me interested in browser security,
so I'm always on the lookout for novel cross-domain data theft vectors. Today I'm going to go into 
the discovery and exploitation of such a bug: A timing attack on Firefox's `document.elementFromPoint` and `document.caretPositionFromPoint` implementations.

# Initial Discovery

I was looking at ways to automatically exploit another bug that required user interaction when I noticed [elementFromPoint](https://developer.mozilla.org/en-US/docs/Web/API/document.elementFromPoint) and
[caretPositionFromPoint](https://developer.mozilla.org/en-US/docs/Web/API/document.caretPositionFromPoint) on the MDN.
Curious as to how they behaved with frames, I did a little testing.

I made an example page to test:

```html
<html><body>
    <iframe id="testFrame" src="http://cbc.ca" width="1025"></iframe>
</body></html>
```

`elementFromPoint()` behaved exactly as I expected, when used in the web console it returned the iframe on my page:

```javascript
> console.log(document.elementFromPoint(frame.offsetLeft + 10, frame.offsetTop + 10))
< [object HTMLIFrameElement]
```

`caretPositionFromPoint()`, however, was returning elements from the page on cbc.ca!

```javascript
> console.log(document.caretPositionFromPoint(frame.offsetLeft + 10, frame.offsetTop + 10))
< [object CaretPosition]
```
[{% img center /images/posts/frompoint/obj_inspector.png %}](/images/posts/frompoint/obj_inspector.png)

But there was a small snag: I couldn't actually access the `CaretPosition`'s `offsetNode` from JS without getting a security exception.
It seems that Firefox noticed that `offsetNode` was being set to an element from a cross-origin document, and wrapped the `CaretPosition` object
so that I couldn't access any of its members from *my* document. Great.

However, I found I *could* access `offsetNode` when it was set to null. `offsetNode` seems to be set to null when the topmost
element at a given point is a button, and that includes scrollbar thumbs. That's great for us, because knowing the size and location of the frame's scrollbar thumb
tells us how large the framed document is, and also allows us to leak which elements exist on the page.

For example here's what we can infer about [https://tomcat.apache.org/tomcat-5.5-doc/ssl-howto.html#Create_a_local_Certificate_Signing_Request_(CSR)](https://tomcat.apache.org/tomcat-5.5-doc/ssl-howto.html#Create_a_local_Certificate_Signing_Request_\(CSR\)) through its scrollbars:

[{% img center /images/posts/frompoint/nullity_test.png 600 600 %}](/images/posts/frompoint/nullity_test.png)

The vertical scrollbar thumb has obviously moved, so we know that an element with an id of `Create_a_local_Certificate_Signing_Request_(CSR)` exists in the framed document.

The following function is used to test `offsetNode` accessibility at a given point in the document:

```javascript
function isOffsetNodeAccessibleAt(x, y) {
	try {
		document.caretPositionFromPoint(x, y).offsetNode;
	} catch(e) {
		return false;
	}
	return true;
}
```

# Digging Deeper

Knowing the page's size and whether certain elements are present is nice, but I wanted more. I remembered [Paul Stone's excellent paper about timing attacks on browser renderers](http://contextis.co.uk/research/white-papers/pixel-perfect-timing-attacks-html5/) and figured a timing attack might help us here.

`caretPositionFromPoint` has to do hit testing on the document to determine what the topmost element is at a given point,
and I figured that's not likely to be a constant time operation. It was also clear that hit testing *was* being performed on cross-origin frame contents, since `caretPositionFromPoint` was returning elements from them. 
I guessed that the time it took for a `caretPositionFromPoint(x,y)` call to return would leak information about the element at `(x,y)`.

To test my theory I made a script that runs `caretPositionFromPoint(x,y)` on a given point 50 times, then stores the median time that the call took to complete. Using the median is important so we can eliminate timing differences due to unrelated factors, like CPU load at the time of the call.

```javascript
function timeToFindPoint(x, y) {
	// window. getter is slow, apparently.
	var perf = window.performance;
	
	// Run caretPositionFromPoint() NUM_SAMPLES times and store runtime for each call.
	var runTimes = new Float64Array(NUM_SAMPLES);
	for(var i=0; i<NUM_SAMPLES; ++i) {
		var start = perf.now();
		document.caretPositionFromPoint(x, y);
		runTimes[i] = perf.now() - start;
	}
	
	// Return the median runtime for the call
	runTimes = Array.apply( [], runTimes);
	runTimes.sort();
	
	return runTimes[(NUM_SAMPLES / 2) | 0];
}
```

Once we've gathered timing measurements for all of the points across the iframe, we can make a visualization of the differences:

[{% img center /images/posts/frompoint/reltime.png 600 600 %}](/images/posts/frompoint/reltime.png)

Neat.

You can see a number of things in the timing data: the bounding boxes of individual elements, how the lines of text wrap, the position of the bullets in the list, etc.

It also seems that even though `elementFromPoint` doesn't return elements from the framed document, it still descends into it for its hit testing, so it's vulnerable to
the same timing attack as `caretPositionFromPoint`.

# Stealing Text

So we can infer quite a bit about the framed document from the timing information, but can we actually steal text from it?

* **Short Answer**: No.
* **Long Answer**: Maybe, with a lot of work, depending on the page's styling.

I'd hoped that `caretPositionFromPoint`'s real purpose (determining what character index the caret should be at for a given point) would yield large enough timing differences to leak the width of individual characters, but that didn't seem to be the case.

[Using a similar method to sirdarckcat's](http://sirdarckcat.blogspot.com/2013/09/matryoshka-wrapping-overflow-leak-on.html#victim2), we can shrink the width of the iframe to force the text to rewrap, and subtract the old width of the the line from the new width, giving us the width of the word that just wrapped.

Note that since we need to force text wrapping to get these measurements, it's harder to steal text from fixed-width pages, or pages that display a horizontal scrollbar instead of wrapping text (like `view-source:` URIs.) Pages that use fixed-width fonts are also more difficult to analyze because characters do not have distinct widths, we can only determine the number of characters in the word.

# Working Examples

Note that the last Firefox version these actually work in is `26`, if you want to try them out you'll have to find a download for it.

* [caretPositionFromPoint accessibility PoC](http://saynotolinux.com/tests/moz_frompoint/crossorigin_auto_domsniff_nullity.html)
* [*FromPoint timing attack PoC](http://saynotolinux.com/tests/moz_frompoint/crossorigin_auto_domsniff.html)

# The Fix

Judging from [Robert O'Callahan's fix](https://hg.mozilla.org/mozilla-central/rev/cdbe5779c728), it looks like Firefox was using a general hit testing function that descended cross document for both `elementFromPoint` and `caretPositionFromPoint`. The fix was to disable cross-document descent in the hit testing function when called by either `elementFromPoint` or `caretPositionFromPoint`.

# Disclosure Timeline

* Dec. 11 2013: Discovered `caretPositionFromPoint` leaked info through `offsetNode` accessibility
* Dec. 13 2013: Notified Mozilla
* Dec. 13 2013: Mozilla responds
* Dec. 15 2013: Discovered timing info leaks in both `elementFromPoint` and `caretPositionFromPoint`
* Dec. 16 2013: Sent update to Mozilla
* Dec. 16 2013: Mozilla responds 
* Dec. 18 2013: [Fix committed](https://hg.mozilla.org/mozilla-central/rev/cdbe5779c728)
* Jan. 16 2014: Fix pushed to Beta channel
* Feb. 04 2014: Fix pushed to Stable channel and [advisory posted](https://www.mozilla.org/security/announce/2014/mfsa2014-05.html)
