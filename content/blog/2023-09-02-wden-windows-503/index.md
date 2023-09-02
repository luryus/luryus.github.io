+++
date = 2023-09-02
updated = 2023-09-02
title = "Why is Bitwarden returning 503s to my Windows app?"
authors = ["Lauri Koskela"]
draft = false

[taxonomies]
tags = [ "wden", "debugging" ]
+++



For a few years now, I've been developing a TUI Bitwarden client app called [Wden](https://github.com/luryus/wden). While Bitwarden does not have a stable API for writing alternative clients, the APIs for the official clients have been pretty well documented by third parties. It's not too difficult to write a client against those, but it will occasionally break when Bitwarden makes changes to their servers.

Recently, I ran against a peculiar issue where Bitwarden started to consistently return errors to Wden, but only when running on Windows. There seemed to be no good explanation for this - the functionality is completely identical between platforms. I found the whole thing quite intriguing and I decided to write something about how I debugged the issue and worked around it.

<!-- more -->
## The Encounter

On a Sunday evening, I was just starting a gaming session with my friends. After starting Windows, Epic Games asked for my password, so I launched Wden, as usual, and tried to log in to my Bitwarden account. The login failed, because an access token request errored with a 503 status code. Huh, maybe Bitwarden is having some issues, I thought, and forgot about it. I didn't really need anything from Epic anyway.

{% figure(
    src="wden-503-error.png",
    alt="Screenshot of Wden showing a 503 error from a request to /identity/connect/token") %}
The error that started this journey.
{% end %}

The next morning, at work, I tried to get a password from Wden on my work Windows laptop. Again, I couldn't log in, and Wden just showed the same 503 response. This time I actually needed the password, so I logged in to Bitwarden's web vault. It worked just fine, no errors whatsoever. OK, so maybe this is something client-specific and Wden is broken? Wouldn't be the first time Bitwarden adds some new requirement for the clients that then breaks Wden. That's what you get for using APIs not intended for general use, I suppose.

## The Investigation

That evening, I tested Wden on my Linux laptop. I logged in to my Bitwarden account and... it works now? The vault loaded without errors.

Maybe Bitwarden finally fixed their server issues? I started up my Windows machine where I first encountered the error. But no, Wden still did not work there. Strange.

I began debugging by adding full response logging in Wden. I wanted to see the HTTP response body for one of the failing login requests:
```json
{"message": "An unhandled server error has occurred."}
```

It seems that the error is just some uncaught exception in Bitwarden's server code. A status code 500 would be a better choice than a 503, but whatever. A quick search through Bitwarden's source led me to [this exception filter](https://github.com/bitwarden/server/blob/4c77f993e561ca37265b9b7caf3b9cc85e69504b/src/Api/Utilities/ExceptionHandlerFilterAttribute.cs#L117) where the message originates from. In fact, it sets the status code to 500. I guess a proxy server changes it to 503 on the way back. Anyway, the response did not include any details about the exception, so I was none the wiser.

At this point, I tried to find out if anyone else had reported these kinds of exceptions recently, but nothing stood out. There were some GitHub issues and Bitwarden forum posts, but all of them involved self-hosted setups, not the Bitwarden cloud service. And they happened in completely different situations, too.

So, back to debugging. If the error arose from the HTTP server handler, there must have been something different in the HTTP requests between working and broken calls, right? TLS and all lower level layers in the network protocol stack are handled by some frontend proxy server anyway (Bitwarden's requests go through Cloudflare), so differences in those should not cause different server behaviour.

To see what kind of requests Wden made, I of course fired up [mitmproxy](https://mitmproxy.org/). Fortunately, [Reqwest](https://github.com/seanmonstar/reqwest), the HTTP client library Wden uses, supports easy proxy configuration via the HTTPS_PROXY environment variable. I tried to log in while mitmproxy was capturing all traffic from Wden, and HTTP requests and responses appeared in the web UI.

{% figure(
    src="mitmweb-b.png",
    alt="Screenshot of mitmweb, showing a bunch of successful requests to Bitwarden") %}
No errors to be seen here.
{% end %}

There was a noticeable lack of errors in the responses. Wden successfully fetched an access token and loaded my vault. The whole thing worked just like it should. What's going on? There was nothing unusual in the client request, as far as I could tell. Could mitmproxy somehow alter the outgoing requests in some way and avoid triggering the exceptions on the server side?

I decided to test another proxy, and installed [Fiddler](https://www.telerik.com/fiddler/fiddler-classic). With that, the errors were back again - the access token requests returned errors with status code 503.

{% figure(
    src="fiddler.png",
    alt="Screenshot of Fiddler, showing a capture of a 503 response from Bitwarden.") %}
Fiddler allowed me to see the full error response, with a good variety of font sizes in the UI.
{% end %}

At least I now had captures of both working and broken HTTP requests. *But there was absolutely no difference in them* - or at least the proxy tools did not show any. I did notice some differences in the TLS handshake parameters displayed by the tools, though. But that couldn't affect this...?

## The Culprit

At this point my best guess for the issue was that _something_ in the requests from Wden tripped Bitwarden's server code up, and using mitmproxy _somehow_ altered the request to make this not happen. Maybe some HTTP header casing difference or something like that. So I decided to do some more tests with more basic tools.

I copied a Bitwarden token request from Chrome as a curl command, and trimmed excess parameters from it so that the request was close enough to those made by Wden. I ran it, and unsurprisingly, it succeeded.

I then decided that I did not want to associate these somewhat suspicious-looking test requests with my account, and tried sending the same request with an empty JSON object as the request body. With curl, that correctly returned a 400 Invalid Request error, because the request *was* actually invalid. I then altered Wden to also send an empty object, and that still got a 503 error back - *the errors did not seem to be caused by anything in the body*. Good, now I had an easy test case that would not get my account marked as suspicious. A 400 response was a success, a 503 response indicated a server error.

Maybe Fiddler would show some differences now between the broken requests from Wden and the working request from curl? I ran Wden while proxying through Fiddler, and the response was a 503 one, just like before. I then proxied the curl request through Fiddler, and... *now that returns a 503 response too?*

Was this more of a Windows thing, and not a Wden thing after all? Fiddler is a Windows-only app, and it probably uses Windows networking APIs to make the outgoing HTTP requests. To test my hypothesis, I opened a PowerShell prompt, and called up everybody's least favourite [curl imposter](https://daniel.haxx.se/blog/2016/08/19/removing-the-powershell-curl-alias/), `Invoke-WebRequest`. After a quick check of its help output and cringing at the verbosity of parameter names in PowerShell, I crafted a token request, and sent it.

{% figure(
    src="curl-and-iwr.png",
    alt="Curl and Invoke-WebRequest side-by-side, curl showing an invalid_request error, Invoke-WebRequest and internal server error.") %}
Identical requests, different clients, different responses. Curl with the 400 on the left, Invoke-WebRequest with the 503 on the right.
{% end %}

*And it returned a 503 response!* Indeed, it seemed that making requests with Windows APIs cause the request processing to fail, whereas other network stacks work fine. But which part in Windows could possibly have any effect on the server side? The HTTP client in Wden is completely implemented in Rust; Windows should not have any part in that. I would imagine TCP and all the lower network layers are the same regardless of the client program, because those are implemented by the OS network stack. So the only culprit that made sense to me was TLS - even though I first thought that it could not be the reason for the errors.

To test if something TLS-related caused this in Wden, too, I edited its Cargo.toml and enabled the `rustls_tls` feature in reqwest. That, along with a `use_rustls_tls()` call when building the HTTP client, switches from the default TLS library to [Rustls](https://github.com/rustls/rustls). And with that, the login worked with no issues! Success?

Invoke-WebRequest is built on the .NET HTTP client, and .NET uses [Schannel](https://learn.microsoft.com/en-us/windows/win32/secauthn/secure-channel), the built-in TLS stack in Windows. Reqwest defaults to using the `native_tls` crate that leverages system-provided TLS libraries - again, Schannel on Windows. The curl build that I used for testing uses OpenSSL. The pattern seemed pretty obvious: clients that use Schannel got 503 errors, others worked fine.

Well, of course it was not that easy of an conclusion. Curl, being awesome, also supports  Schannel on Windows. It can be enabled with the `CURL_SSL_BACKEND=schannel` environment variable. Naturally, I tested the token requests with this Curl+Schannel combination. To my surprise, the request worked just fine. So it's not *just* Schannel, but rather Schannel with some specific parameters?

## The Scrutiny

Time to open Wireshark. I had no idea what to look for, but I still hoped that comparing the TLS handshakes would reveal something. I'm by no means an expert with TLS, so I considered this a good learning opportunity as well. At least now I had easy methods of making both working and exception-triggering requests: just make identical requests with curl (configured to use Schannel) and Invoke-WebRequest.

When comparing the Wireshark captures, I immediately noticed that the TLS handshakes were pretty drastically different. The Invoke-WebRequest TLS handshake had fewer messages. This turned out to be due to TLS session resumption, which I had completely forgot about being a thing. Curl does not reuse TLS sessions across runs, whereas Invoke-WebRequest does. Well, this had nothing to do with my Bitwarden issue after all - the results were still the same after disabling TLS session caching in Windows. But at least now I had more comparable packet captures.

I started to go through the handshake parameters, one by one, and tried to change related parameters in curl to see if they would cause the server to return errors. With the session reuse difference out of the way, there was actually only one difference remaining in the TLS Client Hello message: the `status_request` extension. This is how TLS clients request [OCSP stapling](https://en.wikipedia.org/wiki/OCSP_stapling). Curl specified this, Invoke-WebRequest did not.

{% figure(
    src="wireshark.png",
    alt="Screenshot of two Wireshark windows showing TLS Client Hello message details.") %}
A broken (status code 503) request flow on the left, a working (400) request flow on the right. One is missing the `status_request` extension.
{% end %}

Curl has an option to control OCSP stapling, `--no-cert-status`, but that did not seem to do anything. And in fact, the man page even says that the option not implemented for Schannel. It took a bit of searching, but I finally found the Schannel-specific option `--ssl-no-revoke` from curl's man page. I added that to the curl command, and finally:

{% figure(
    src="curl-schannel-503.png",
    alt="Screenshot of curl getting a 503 response.") %}
With specific options, curl can also trigger the error.
{% end %}

The curl+Schannel combination now received a 503 error as well. Could the errors have something to do with OCSP? I thought not - again, I would think the entire TLS handshake, *including* OCSP stapling, is handled by Cloudflare, not the Bitwarden API server. I tried to search for mentions of OCSP in the Bitwarden source code anyway, but found nothing relevant.

Still not satisfied, I continued to test different options and clients. Up until now, I had used the curl build installed with Git for Windows. [Windows ships with a curl build now](https://curl.se/windows/microsoft.html)[^curl.exe], so I tried that. I ran the exact same command that received a 503 response before. This time the server was back to working normally and returned a 400 response. Comparing the Wireshark captures from these runs, the only meaningful difference was that the Windows curl build used [ALPN](https://en.wikipedia.org/wiki/Application-Layer_Protocol_Negotiation) to specify HTTP/2 usage during the TLS handshake. I added `--no-alpn` to the curl command, and got a 503 back. What, now ALPN affects the errors, too?

It seemed that the server threw an exception if the client used Schannel with some specific configuration. Changing seemingly arbitrary options changed how the server worked. *What was going on here?*


## The Gatekeeper

Yes, I believe this is another case of "Cloudflare is blocking completely valid requests and breaking the open Internet." No, I don't claim that this is actually their fault or that they're doing anything wrong.

As previously mentioned, Bitwarden's traffic goes through Cloudflare. During the investigation I stumbled upon some header names that Bitwarden uses internally: `x-cf-is-bot`, `x-cf-bot-score`, among others[^bitwarden-cf-headers]. Googling led me to a [Bitwarden documentation page](https://contributing.bitwarden.com/architecture/deep-dives/captchas/) with this description:
> The CloudFlare x-Cf-Is-Bot header is present on the request, or

Given that I have found no other references to these header names, I believe they are specific to Bitwarden, and generated in a Cloudflare Worker[^worker] or some other service that Bitwarden proxies the vault traffic through. They probably use [Cloudflare bot management features](https://developers.cloudflare.com/bots/) to detect unusual traffic, and set certain headers accordingly in the upstream server requests.

While reading through the Cloudflare docs, I noticed an acronym  I had just seen when combing through the Wireshark captures: **JA3**. I had never heard about it before now. After learning about it, I immediately understood what the problem with Schannel was.

JA3 is a TLS fingerprint mechanism. It's described here in [a blog post by Salesforce](https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967/) In short, it calculates a hash based on parameters in a TLS client hello message, allowing the identification of specific applications across IP addresses. [Cloudflare uses JA3 in their bot detection](https://developers.cloudflare.com/bots/concepts/ja3-fingerprint/), because of course they do.

Here are the JA3 hashes from my various Wireshark captures:
* `3b5074b1b5d032e5620f69f9f700ff0e`: Wden, Invoke-WebRequest, curl with `--ssl-no-revoke`
* `ce5f3254611a8c095a3d821d44539877`: curl _without_ `--ssl-no-revoke`
_All_ the clients that got 503 errors from Bitwarden share the same JA3 hash. And because the hash is calculated from the TLS parameters, enabling/disabling OCSP changes it.

I suspect the Bitwarden exceptions I'm seeing happen something like this:
- For some reason, the JA3 fingerprint from Wden (and other similar Windows clients) has ended up on a Cloudflare bot list. Consequently, all requests from Wden on Windows are flagged as bot traffic.
- Bitwarden's proxy code, using the Cloudflare APIs, detects that the request originates from a bot and adds extra headers to the upstream server request.
- Due to a bug, the Bitwarden server code fails to handle these headers correctly, and throws an exception.
- The backend server returns a HTTP 500 response to the proxy. The proxy returns it as a 503 response to the client.

Now, this is not really an issue for Bitwarden themselves, because as far as I know, none of their official clients use Schannel. They may have explicitly allowed the JA3 fingerprints of their official client apps so that the bot detection never triggers on them. They probably do not care (or even know) about the server exceptions, because they do not happen on "valid" client requests. Still, I find it concerning that pretty much all requests from Windows can be caught in the bot detection. In fact, in the Salesforce JA3 blog post they even mention this scenario:
>But what if the client application uses common libraries or OS sockets for communication like Python or Windows Socket? The JA3 would be common in the environment and therefore not as useful for detection.

## The Resolution

So how did I fix this in Wden? I can't fix the server or change the bot detection rules. I did not even report this to Bitwarden and ask them to fix their code, because I'm using an unofficial client. I could have just permanently switched to Rustls in Wden, or even changed some arbitrary TLS parameters to change the JA3 fingerprint, and then hope that the bot detector would not notice the new one. But actually, there was a better solution - one that hopefully works more permanently and also adds a missing feature in Wden.

First, a bit of background information. A Bitwarden server consists of a handful of services. In self-hosted environments, these services typically run on a single host and are accessible via different paths (API service under `/api`, identity service under `/identity`, and so on). However, Bitwarden's cloud service (as well as more complex self-hosted setups) uses [different hostnames](https://bitwarden.com/help/bitwarden-addresses/) for the services (`api.bitwarden.com`, `identity.bitwarden.com`, and so on). All official client apps connect using these host names.

Essentially, this means that Bitwarden clients need to support two different ways of configuring the servers. But luckily (for past me), there was a shortcut. The Bitwarden web vault is hosted on a single domain (https://vault.bitwarden.com), and that host acts like a simple self-hosted Bitwarden instance. When I initially developed Wden, I naturally took the easy route and only built support for a single domain. This covered all my use cases: simple self-hosted servers and the Bitwarden cloud service. And it worked just fine, until now.

While investigating the server errors, I happened to notice that the errors *only occurred* when making the requests through `vault.bitwarden.com`. Making the same `/token` request directly to `identity.bitwarden.com` always succeeded. This might be just temporary - for all I know, a bot detector on the `identity` host could start blocking my requests tomorrow. However, considering that `vault.bitwarden.com` is only really supposed be used by web browsers, it wouldn't be surprising if it had stricter bot detection rules compared to the other hosts.

So, as a workaround (at least for now), I finally decided to implement proper support for Bitwarden servers with separate identity and API server URLs. Bitwarden cloud service connections are no longer made through `vault.bitwarden.com` but rather with the separate URLs. Server errors no longer occur when using Wden, and this is a slightly less hacky way to use Bitwarden's cloud servers, anyway.

The error and its root cause did not actually get fixed - I still don't know what causes the errors in Bitwarden's backend code. I didn't bother reporting this to Bitwarden, because I suspect it doesn't occur with any of their official client applications. At least I couldn't find any references to similar errors from the cloud servers.

Now, I just have to keep hoping that Cloudflare's bot detection doesn't start picking on Wden again...

## The Conclusion

In the end, this issue boiled down to a bit of bad luck with a bot filter and some buggy server code. I did not really _fix_ anything, because it's all out of my control. I just worked around the problem by avoiding the server with the misbehaving bot detection.

This was a really fun debugging journey for me, even if the end result was slightly unsatisfying. It's not often that I get to spend multiple evenings staring at Wireshark captures and exploring various TLS options, while trying to figure out how they could possibly have anything to do with the issue at hand. I would have liked a definitive bug fix, but at least I now have a functioning password manager once again.

In hindsight, I might have found the root cause more quickly if I hadn't held so strong assumptions about the issue:
* It must be something I'm doing wrong, because other people are not seeing the errors
* Wden has previously broken in somewhat similar ways because of Bitwarden API changes, this is likely some new requirement
* It must be something in the HTTP requests because the errors happen in the server code, and that does not handle TLS termination or any other lower level protocols from my client

As it turned out, none of these assumptions proved correct. If I had more experience with Cloudflare and other similar traffic filtering systems, I might have suspected TLS fingerprinting sooner.

For now, I can once again access my passwords in the terminal. And, when Wden inevitably breaks again in the future, I might have better approaches for debugging and fixing it more quickly.


[^curl.exe]: Yes, in PowerShell, `curl` is an alias for Invoke-WebRequest, `curl.exe` is real curl. Not confusing at all.

[^bitwarden-cf-headers]: I'm 99% sure that I saw these headers in some _responses_ from Bitwarden during my investigation and exploration. At the time, I thought it was strange, given that these headers are only for internal use. When writing this, I could no longer find them in any responses, so maybe Bitwarden has fixed something about the headers recently.

[^worker]: Bitwarden's source code has some references to Cloudflare workers being used, and during my testing I saw Cloudflare error messages about a worker throwing an error.