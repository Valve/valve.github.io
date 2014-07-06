---
layout: post
title: "anonymous browser fingerprinting"
date: 2013-07-14 12:29
comments: true
categories: [javascript, security]
---

## What is fingerprinting?

Fingerprinting is a technique, outlined in the [research by Electronic Frontier Foundation](https://panopticlick.eff.org/browser-uniqueness.pdf), of anonymously identifying a web browser with accuracy of up to 94%.

Browser is queried its agent string, screen color depth, language, installed plugins with supported mime types, timezone offset and other capabilities, such as local storage and session storage. Then these values are passed through a hashing function to produce a fingerprint that gives weak guarantees of uniqueness.

No cookies are stored to identify a browser.

It's worth noting that a mobile share of browsers is much more uniform, so fingerprinting should be used only as a supplementary identifying mechanism there.

In this post I'm going to explain how it works in detail and give you real-life statistics accumulated over the period of 4 months of production usage.
<!--more--> 

### Why

I was given an experimental task to implement the fingerprinting for both anonymous and logged-in users of one of our web sites. We wanted to see if it was possible at all to rely on identifying someone this way and not leave cookies. The idea was to accumulate the fingerprints and associated preferences and then pre-filter the information on front page based on what's known about a user.

### Implementation

So I got to work and started making a basic outline in my head. What is that identifies a browser? I gathered it would be:
_browser agent, browser language, screen color depth, installed plugins and their mime types, timezone offset, local storage, and session storage._

Initially I added the screen resolution as well, but a colleague adviced that one can use multiple monitors with a single laptop, for example connect an external monitor when working in office, so I removed it.

On my laptop browser the values are:

{% codeblock lang:javascript %}
// Assuming jQuery in scope

navigator.userAgent
// "Mozilla/5.0 (X11; Linux i686) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/27.0.1453.110 Safari/537.36"

navigator.language
// "en-US"

var plugins = $.map(navigator.plugins, function(p){
   var mimeTypes = $.map(p, function(mimeType){
    return [mimeType.type, mimeType.suffixes].join('~');
   }).join(',');
  return [p.name, p.description, mimeTypes].join('::');
});


$.each(plugins, function(i, p){ 
  // truncate only for blog example
  if(p.length > 80){
    console.log(p.substring(0, 77) + '...');
  } else{
    console.log(p);
  }
});

/*
Shockwave Flash:Shockwave Flash 11.7 r700:application/x-shockwave-flash~swf,a... 
Chrome Remote Desktop Viewer:This plugin allows you to securely access other ... 
Widevine Content Decryption Module:Enables Widevine licenses for playback of ... 
Native Client::application/x-nacl~nexe 
Chrome PDF Viewer::application/pdf~pdf,application/x-google-chrome-print-prev... 
Google Talk Plugin Video Accelerator:Google Talk Plugin Video Accelerator ver... 
Google Talk Plugin:Version: 4.0.1.0:application/googletalk~googletalk 
Google Talk Plugin Video Renderer:Version: 4.0.1.0:application/o1d~o1d 
Shockwave Flash:Shockwave Flash 11.2 r202:application/x-shockwave-flash~swf,a...
*/

screen.colorDepth
// 24

new Date().getTimezoneOffset();
// -240

!!window.localStorage
// true

!!window.sessionStorage
// true
{% endcodeblock %}

So I now knew all my browser had, and I needed to produce the fingerprint itself.
For that I wanted to use a fast, non-cryptographic hashing function, such as [murmur hashing](http://en.wikipedia.org/wiki/MurmurHash).

Murmur hashing produces 32-bit integer as a result and works really well. [When compared to other popular hash functions, MurmurHash performed well in a random distribution of regular keys.](http://programmers.stackexchange.com/questions/49550/which-hashing-algorithm-is-best-for-uniqueness-and-speed)

I picked [this implementation](http://github.com/garycourt/murmurhash-js) and added it to the code.

The last step was to combine all browser's capabilities into a long string and pass it through hashing.

The end result on my laptop was: `3723825959`

As a finishing touch, I wanted to get rid of jQuery, so I implemented the `each` and `map` methods and got a no-dependencies script.

#### How to improve accuracy?

The above research states that the identification accuracy is surprisingly high. But to improve it even further, Flash or Java integration is required to get a list of installed fonts, thus making each browser even more unique.

#### What about hash collisions?

My tests show that for random strings Murmurh hashing indeed produces collisions, but their number is negligible for my purposes: 5-7 collisions per ~200K of capabilities strings.

#### What about mobile browsers?

It's simple: browser fingerprinting is not good with mobile browsers, unless you want to distinguish Android users from iPhone ones.

### Results

After having had the fingerprinting on production for 4 months, I have some data to analyze. First of all, let me say that I'm not at liberty to tell the exact number of visitors to the web site, but I can say it is several millions a month, so we have some data to play with. All numbers below represent our usage and do not represent what you might have.

**89%** of fingerprints are unique

**20%** of our users have more than one fingerprint, i.e. several browsers or devices.

Very few users have a staggering amount of fingerprints, for example 20-25. I don't know if they have a lot of devices, use different browsers or something else.

After viewing the results we removed the fingerprinting because of poor identification, especially with mobile devices. 
If your traffic mostly comes from desktops and you're OK with 10-12% of false identifications you might want to try it.

### Show me the code

#### [code on github](https://github.com/Valve/fingerprintjs) - the version I had in production

#### [test your browser](http://valve.github.io/fingerprintjs/)
