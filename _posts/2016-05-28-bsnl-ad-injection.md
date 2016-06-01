---
layout: post
title: "Blocking BSNL Ad Injection"
excerpt:
tags: tech
---

BSNL's broadband service has been generally very good to me. But now 
we're getting ads injected via javascript on bare http pages. The ads
themselves are fairly benign - being for their own services and as far as can
one see[<sup>1</sup>](#bsnl-note1), involving no other third parties. But the
popups are very annoying distractions and the script interception does break a
few pages despite their efforts to handle things transparently.

Any of the usual blockers - NoScript, ABP, uBlock - can block the ad, but since
the ISP is piggybacking a legitimate page request to deliver their content, the
blockers block the legitimate js request and end up crippling the page. [HTTPS
Everywhere](https://www.eff.org/https-everywhere) will alleviate the problem,
but until https is more widely adopted, the solution that has worked for me
with Firefox is to use a redirect blocker.

* Install [Redirector](http://einaregilsson.com/redirector/). Works on
Firefox, Chrome and Opera.[<sup>2</sup>](#bsnl-note2)

* Create a new redirect (by clicking on its icon, then on Edit Redirects. Or
type _about:addons_ in address bar, find Redirector in the list, click
Preferences next to it).

* Put these settings. 

        Description: BSNL Ad Block
        Example URL:
        http://61.0.234.194/dyn/bg/BSNL-FreeRoaming/index.js?policy=26&policyname=BSNL-FreeRoaming&time=1461310826&webServer=http://61.0.234.194&url=http%3A%2F%2Fexample.com%2Fjs%2Fexample.js
        Include Pattern: http://*/dyn/bg/*/index.js*url=*
        Redirect To: $4
        Pattern Type: Wildcard

* Click on _Advanced options_ and select _URL decode_ from the option box labeled
_Process Matches_.
Example Result should now show _http://example.com/js/example.js_

* Select only _Scripts_ in _Apply To_ list and save.

---
##### Injection Mechanism

The first (or sometimes second?) request to a
javascript resource (haven't tested whether they match extension or MIME type)
from a html page is redirected via a HTTP 302 code to an internal BSNL server.

        http://61.0.234.194/dyn/bg/BSNL-FreeRoaming/index.js?policy=26&policyname=BSNL-FreeRoaming&time=1461310826&webServer=http://61.0.234.194&url=http%3A%2F%2Fwhatmoji.com%2Fjs%2Fbootstrap.min.js

It serves an ad if the requesting browser can display it (pretty readable logic
here which reveals other ways to not receive ads - like changing user agent to
iphone/android or certain Opera versions) and then redirects to the originally
requested url.

        http://61.0.234.194/cgi-bin/redirect?url=http%3A%2F%2Fwhatmoji.com%2Fjs%2Fbootstrap.min.js
----
##### Notes
<a name="bsnl-note1"><sup>1</sup></a> 

[The injected
script](https://gist.github.com/mhemantha/8d0772531935400456c8a36c45d304be).
When Airtel's like shenanigans were publicised last year, the programmer who
posted the script was [served a Cease and
Desist](http://trak.in/tags/business/2015/06/05/airtel-injecting-script-iframe-user-web-browser/)
from the Israeli developer of the script. Crossing fingers now.

<a name="bsnl-note2"><sup>2</sup></a> 

It's also available for Firefox on
Android, but unfortunately it's config page is inaccessible there and I haven't
figured out how to apply the settings. It's apparently easy to do the kind of
redirect I need here with the new WebExtensions API, so I'm thinking of writing
a specific extension for blocking BSNL ads.

