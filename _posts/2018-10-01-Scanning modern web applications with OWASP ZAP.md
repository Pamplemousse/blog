---
layout: post
title: Scanning "modern" web applications with OWASP ZAP
tags: [ development, javascript, ZAP ]
---

During the summer of 2018, I was an intern in the [FoxSec](https://wiki.mozilla.org/Security/FirefoxOperations) team at Mozilla, where I contributed to [ZAP (for Zed Attack proxy)](https://www.zaproxy.org/), an open-source web application security scanner.

The subject of my internship was **Scanning modern web applications with OWASP ZAP**, and the report I wrote about it is available online at <a href='https://www.xaviermaso.com/internship_report_2018.pdf' target='_blank'>xaviermaso.com/internship_report_2018.pdf</a> .

I do not intend to delve too much into the details of what have been implemented in this post, especially because the report should contain all the informations needed for whom is interested by the subject.

However, this is a good opportunity for me to talk a bit more loosely about what is inside.
In a sense, this post is more of an "Abstract" (if you are from the academia) or a "TL;DR" (if you are from Reddit) to present the key ideas and motivations for those who still hesitate to read twenty pages of my lame prose.

Thus, let's follow the plan of the report:
  1. [ZAP and "modern" web applications](#zap-and-modern-web-applications)
  2. [The FrontEndScanner](#the-frontendscanner)
  3. [The front-end-tracker](#the-front-end-tracker)


## ZAP and "modern" web applications

### Some ZAP concepts

ZAP is a web proxy: sitting between a web browser and the server serving the application that one wishes to test, it can monitor web traffic between the two entities, interrupt it, modify it and even record and replay it.

Because of that, it is a great tool to perform security testing of an application; either by passively looking for vulnerabilities in the requests and responses (such as missing headers, plaintext secrets, or else), or by actively crafting requests or tampering with the content aiming to trigger interesting behavior on the server, or in the browser (in the case of XSS vulnerabilities for example).

One of the most interesting feature of ZAP is for the user to be able to write their own scripts that will run under specific circumstances, to perform custom tasks.
For examples, here are a couple of scripts that have been written by the community and published under [github.com/zaproxy/community-scripts](https://github.com/zaproxy/community-scripts) .


### "Modern" web applications

Arguably, we called "modern" web applications the ones relying heavily on JavaScript.

In nowadays web, almost every page contains JavaScript to be executed by the client (aka, the web browser).
This is even truer, especially with the rise of JavaScript framework such as [React](https://reactjs.org/), [AngularJS](https://angularjs.org/), [Vue.js](https://vuejs.org/), [Ember.js](https://emberjs.com/), and so on, encouraging developers to embed a whole applications into single pages, the so-called <a href='https://en.wikipedia.org/wiki/Single-page_application' target='_blank'>"SPA"</a>.

The problem is that it somewhat breaks the approach taken by ZAP: by only scanning HTTP responses, our favorite proxy statically analyze the transferred content, without taking into consideration the transformations that might happen when the browser will interpret the embedded JavaScript.

For example, let's say that to detect XSS or SQLi, you have a script to look for `<input>` fields in a webpage.
If the `<input>` is present in the HTML of the page, ZAP will be able to find it.
However, if there is a piece of JavaScript that modifies the DOM to add such an element, such as:
```javascript
window.onload(function () {
  var inputElement = document.createElement('input');
  inputElement.type = 'text';
  document.body.appendChild(inputElement);
});
```
ZAP would be unable to understand the implications and detect the fact that a potential source of vulnerability will be added to the page "at run time" *i.e* when the browser will interpret the JavaScript.

Through this basic example, one can see the limits of the static analysis of an HTTP response: it is difficult to be confident in the fact that an application is vulnerability free just by looking at its source code, as complex chain of events in the browser (user or network interactions for example) can lead to the modification of the scanned content, making the process irrelevant.

To answer this problematic, we broke our solution down into two "components":
  * the [FrontEndScanner](#the-frontendscanner) add-on,
  * the [front-end-tracker](#the-front-end-tracker) JavaScript library


## The FrontEndScanner

> Add-ons add additional functionality to ZAP. They have full access to all of the ZAP internals, and so can provide very powerful new features.
<sup id="a1">[1](#f1)</sup>

We wrote the FrontEndScanner add-on to provide a way for ZAP users to look for front-end vulnerabilities by executing scripts where they can make sense out of the dynamic nature of JavaScript: in the web browser, alongside the application that is being tested.

When turned on, our add-on will tamper with all HTTP responses coming back from the server to inject a piece of JavaScript code into the tested application, directly into the `<head>` of the HTML document.
By doing so, we ensure that our code will be run before anything else (especially before front-end frameworks and libraries) when loaded by the web browser.
This is really important as we want to keep track of modifications to the DOM and to the <a href='https://developer.mozilla.org/en-US/docs/Web/API' target='_blank'>WebAPI</a> that those external scripts might be doing.

The piece of JavaScript code is made of the following:

  * the `FrontEndScanner` object, itself containing:
    - ZAP constants to help scripts create alerts,
    - the "mailbox": a "publish-subscribe" mechanism to help ZAP users' scripts react to events happening in the browser (such as user interactions, [Storage](!mdn storage) accesses, etc.),
    - a helper function to report findings back to ZAP
  * a list of user defined scripts, for which each of them will be encapsulated in a function

When in the browser, these functions are executed, taking the `FrontEndScanner` object as parameter.
Thus, user scripts can make use of the content defined above to perform meaningful security checks and raise alerts in ZAP when finding vulnerabilities.


## The front-end-tracker

As the <a href='https://developer.mozilla.org/en-US/docs/Web/API' target='_blank'>WebAPI</a> is mostly intended for application developers rather than for security testers, it does not expose all the features that we would hope to have for debugging and testing a web page.

To answer this lack of features considering our use case, we wrote the front-end-tracker, a JavaScript library meant to provide an extension of the API available in the browser that would be more pertinent for one wishing to write security checks.

#### How does it work?

When loaded into a web page, the front-end-tracker wraps behaviors that we are interested in tracking into our own functions that will perform some kind of reporting before running the expected code.

Here is a simplified example of what it could look like:
```javascript
const oldGetItem = Storage.prototype.getItem;

Storage.prototype.getItem = function (...args) {
  mailbox.publish(
    'storage',
    {action: 'get', args: args}
  );
  return oldGetItem(args);
}
```

We call such a mechanism a "hook", as it hooks a custom function to a standard behavior.
So far, the following hooks have been implemented:
  * [DOM events](https://developer.mozilla.org/en-US/docs/Web/Events): catch when a user interacts with a webpage (by clicking, scrolling, hovering, or else), when resources are loaded, etc.<sup id="a2">[2](#f2)</sup>,
  * [Storage](https://developer.mozilla.org/en-US/docs/Web/API/Storage): catch when values in the storages in the browser are read, written or removed

If the front-end-tracker ever runs after one of this behavior get triggered in the page, this one would not be reported.
Hence, if we want to monitor everything that we are interested in in a web page, the front-end-tracker needs to be the very first thing to be interpreted here.

Another key concept of the front-end-tracker is the **mailbox**: a topic based publish-subscribe object, on which the functions from our hooks publish to, and for which scripts in the page can subscribe to.

```javascript
// example of subscription to log messages related to 'dom-events'
const topic = 'dom-events';
mailbox.subscribe(topic, (_, data) => {
  console.log(data);
});
```
Written to be a standalone component, the front-end-tracker can be used to help debugging any application.
That is why it has been released on [npm](https://npmjs.com) under <a href='https://www.npmjs.com/package/@zaproxy/front-end-tracker' target='_blank'>@zaproxy/front-end-tracker</a>.


## Conclusion

After these twelve weeks of internship, we ended up having an interesting proof-of-concept of our approach and tools for scanning modern web applications.

Not only we implemented the basics FrontEndScanner add-on and the front-end-tracker it relies on, but we wrote the very first client-side passive script to detect when JWT tokens are written in an application.
<sup id="a3">[3](#f3)</sup>

Unfortunately, all the work presented here has not yet been released: indeed, the FrontEndScanner still lacks features and documentation to be made available on ZAP's marketplace: see <a href='https://github.com/zaproxy/zaproxy/issues/4939' target='_blank'>issue #4939</a> for more details.

On the other hand, the front-end-tracker is already published on npm, but could become even more useful with a couple more hooks added to it, such as ones for [DOM mutations](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver), [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Glossary/XHR_(XMLHttpRequest)), or [postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Client/postMessage).

Unfortunately, as I am back to university, I do not have much time to invest on this, and as the ZAP core team members have already an awful lot of things to deal with, it does not seem that these features will be brought to ZAP users in a near future.

If you are interested to help and contribute, you can take a look at the <a href='https://github.com/zaproxy/zaproxy/issues?q=is%3Aissue+is%3Aopen+FrontEndScanner' target='_blank'>related issues opened on GitHub</a>, or come talk to the (very welcoming) team members on `irc.freenode.net`, in channel `#zaproxy`.

**I am always happy to receive constructive feedback, so do not hesitate to ping me, on twitter <a href='https://twitter.com/Pamplemouss_' target='_blank'>@pamplemouss_</a> or elsewhere.**

---
<b id="f1">1</b> Source: <a href='https://github.com/zaproxy/zap-core-help/wiki/HelpStartConceptsAddons' target='blank'>https://github.com/zaproxy/zap-core-help/wiki/HelpStartConceptsAddons</a> . [↩](#a1)

<b id="f2">2</b> The complete list of events to track: <a href='https://github.com/zaproxy/front-end-tracker/blob/master/src/events.js' target='blank'>https://github.com/zaproxy/front-end-tracker/blob/master/src/events.js</a> . [↩](#a2)

<b id="f3">3</b> This "scan-jwt-tokens" script is installed with the FrontEndScanner add-on, and thus available as an example for ZAP users. Here is what it looks like: <a href='https://github.com/zaproxy/zap-extensions/blob/master/addOns/frontendscanner/src/main/zapHomeFiles/scripts/scripts/client-side-passive/scan-jwt-tokens.js' target='blank'>https://github.com/zaproxy/zap-extensions/blob/master/addOns/frontendscanner/src/main/zapHomeFiles/scripts/scripts/client-side-passive/scan-jwt-tokens.js[↩](#a3)
