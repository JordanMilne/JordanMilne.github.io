---
layout: post
title: "Abusing NoScript's global whitelist rules to reveal trusted sites (the easy way)"
date: 2013-12-18 03:58:17 -0400
comments: true
categories: 
---

Here's one that's been <a href="http://www.w2spconf.com/2011/papers/jspriv.pdf">covered a bit before</a>: <a href="http://noscript.net/">NoScript</a> makes it easy for whitelisted sites to see what other sites are on the whitelist.

<h2>So what's the issue?</h2>

Most of the pertinent info is in the previous paper, but I'll give a brief summary. By default NoScript operates in whitelist mode, forbidding scripts from all domains. Once a site has been added to the whitelist, scripts from that domain *as well as those included from other whitelisted domains* may be executed on the page.

NoScript's default whitelist is fairly small, and users don't generally 
share sets of whitelist rules (like with Adblock Plus,) so if a 
site is whitelisted the user must have used or 
visited that site. 

Since the only whitelist is a global one (allowing scripts to run *on* facebook.com also allows other whitelisted domains to execute scripts *from* facebook.com,) whitelisted sites may infer what other sites are on the whitelist by including scripts from other domains and checking whether or not they execute.

The method described in the paper involves finding a valid script file on the domain you want to test and observing its side effects (modifications to the <code>window</code> object or the DOM.) This can be tedious for an attacker, and requires a bit of manual work. It may also pollute the DOM / <code>window</code> object with junk and break our testing code!

Luckily, there's an easier way. <a href="https://grepular.com/Abusing_HTTP_Status_Codes_to_Expose_Private_Information">Mike Cardwell describes</a> a method of determining if a cross-origin resource returned a 200 status code using <code>&lt;script&gt;</code> tags. The <code>&lt;script&gt;</code> tag's `onload` handler will trigger on a successful HTTP status code, and the <code>onerror</code> handler will trigger otherwise. The <code>&lt;script&gt;</code> tag's <code>onload</code> handler will trigger *even if* the resource isn't a valid script.

The same method may be used to determine if NoScript has blocked a resource that would normally return a 200 HTTP status code. <code>http://domain.tld/</code> usually returns HTML with a 200 status code, so that's a pretty good candidate for testing.

For example:
```html onload/onerror example
<script src="http://google.com" onload="javascript:alert('google loaded')" onerror="javascript:alert('google failed')"></script>
```

If you have google.com whitelisted, this will say "google loaded!". Otherwise (or if google.com is down for some reason) this will print "google failed".

<h2>
Cool, let's see it in action!&nbsp;</h2>
<a href="http://saynotolinux.com/tests/noscript/whitelist_disclosure.html">Here's an example</a> of how the attack may be used, including a timing attack based on the <a href="http://blog.saynotolinux.com/2013/11/bypassing-requestpolicys-whitelist.html">RequestPolicy bypass described in my last post</a>. Mind, the timing attack is a bit spotty, and generally doesn't work on 
the first page load. Refresh the page if it doesn't work the first time.

If you don't have either RequestPolicy or NoScript installed, here's what you should see: 

[{% img /images/posts/whitelist-disclosure.png %}](/images/posts/whitelist-disclosure.png)

<h2>How can it be fixed?</h2>
Global whitelist entries by their very nature leak info to any other site on the whitelist. This won't be fixed in NoScript until support for per-site whitelists is added and people are encouraged to remove their old global rules.

In the meantime, using a <a href="https://github.com/JordanMilne/requestpolicy">patched RequestPolicy</a> will give you a per-site whitelist for *all* cross-domain requests, effectively mitigating the issue.
