---
layout: post
title: Mastodon Authorization on iOS
date: 2022-12-28 08:23 -0600
---

Instagram has turned into a festering pit of gross. A stream of gross, unrelated videos and ads. The solution is built on a protocol born from another service going gross, Mastodon. [PixelFed](https://pixelfed.org) is what Instagram was before it got gross. 

I thought it'd be interesting to try to write a client for iOS using only SwiftUI. This is an act of futility as there's [already a client](https://testflight.apple.com/join/5HpHJD5l) out there but it's a good way to try to learn. 

Like I said, Pixelfed is based on Mastodon so I'm using [MastodonKit](https://github.com/MastodonKit/MastodonKit) to get a jumpstart. It works pretty well despite being largely neglected. 

### Signing In

Okay, to the meat of this post. Mastodon uses [OAuth 2.0](https://oauth.net/2/) for authorization which has it's own flow. The gist of it is that you request authorization from the service with the client id and secret. Good so far. But then to get the token for subsequent requests back, you need to provide an _endpoint URI (redirect URI)_ to send the token _back to_.

What? Sir? This is an _app_. 

**TL;DR:**
Open a web view that points to `https://<your instance>/oauth/authorize` with the url parameters needed. Use `func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction) async -> WKNavigationActionPolicy` to pull out the `code` and then use **that** for an API call to `POST /oauth/token` in order to get the API token. 

### The Nitty Gritty

#### First, Getting that Client ID and Secret

Mastodon has this part of the [API](https://docs.joinmastodon.org/client/token/#app) but I was able to also get it from [Pixelfed](https://docs.pixelfed.org/technical-documentation/api/#authorization) itself. It's as simple as making that call and getting back the id and secret. This is a pretty snazzy part of Mastodon as it allows one user to just state what instance they want to point to and the client should be able to register for API access on the fly.

#### Stabbing in the Dark

Remember how I said that `MastodonKit` is outdated? It has a [function](https://github.com/MastodonKit/MastodonKit/blob/master/Sources/MastodonKit/Requests/Login.swift#L22-L26) that allows you to sign in using username and password. I first tried that and got an error saying the client was invalid. I double-check and re-registered my app to no avail. 

Looking at the Mastodon docs, I see two different things. The [first](https://docs.joinmastodon.org/client/token/#flow) says to make an API call using the client id, secret, redirect uri, and grant type of client to receive a `Token` entity. This sounds exactly like what I want but how does it know which user it is? If you read closely, you'll see _“We are requesting a **grant_type** of **client_credentials**, which defaults to giving us the **read** scope.“_. I want both read _and_ write so that won't work.

#### Redirected Around and Around

[Digging deeper](https://docs.joinmastodon.org/methods/oauth/), I see there is a whole flow to this, starting with [this](https://docs.joinmastodon.org/methods/oauth/#authorize) call to `GET /oauth/authorize HTTP/1.1`. Passing the arguments of client id and secret, how you want the code back, redirect_uri, and grant permissions, you'll get the token back but read closely: _“The authorization code will be returned as a query parameter named code.”_ as in `redirect_uri?code=qDFUEaYrRK5c-HNmTCJbAzazwLRInJ7VHFat0wcMgCU`. 

That's not a `Token` entity. It's a **code** that you need to use for a second call (more on that later). You'll note that `MastodonKit` doesn't have this network call. Is it because it used to use the username/password combo before and now Mastodon uses something different and it's just not updated? 

So, I roll my own request with all the pertinent information and I get back data that is _HTML_? Looking back at that documentation for the call, I see “Displays an authorization form to the user. If approved, it will create and return an authorization code, then redirect to the desired `redirect_uri`, or show the authorization code if `urn:ietf:wg:oauth:2.0:oob` was requested.”. So, what do I do here? 

#### Completely Wrong

I took that data and displayed it on a `WKWebView`. As long as I set the base url when loading the data to `pixelfed.social`, it looked just fine! Maybe, when you `POST` the form data, it will authenticate with the server and redirect to the uri you specified. [Reading up on that](https://medium.com/posts-from-emmerge/implementing-an-oauth2-login-flow-in-wkwebview-de74e23fe9ee), `WKWebViewDelegate` has a callback function (`func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction) async -> WKNavigationActionPolicy`) to make sure it's okay to redirect. You should be able to pull the code from there. But when I went to authenticate, it would return a `419` error and I wouldn't get any redirect. What's **worse** is that **sometimes** it would log me in and go straight to the website. Close but not what I wanted. 

Maybe I need to set the `redirect_uri` to `urn:ietf:wg:oauth:2.0:oob`. Does that do anything, nope. The HTML has a token in there. Is that it? Nope. Looking at the request that is made by the form, I see it's passing the username, password, and that token but nothing else. 

#### Here's the Answer

Let's go back to the [documentation](https://docs.joinmastodon.org/methods/oauth/#authorize). What's maddening about this is that it looks like every other API call but it **is not an API call**. There is no blinking red text that it is a good ol' **URL**. What I had to do is have the `WKWebView` load the URL of `https://pixelfed.social/oauth/authorize` with the URL query parameters of `response_type`, `client_id`, `redirect_uri`, and `scope`. For the `redirect_uri`, I made something up of my apps name as the scheme and `auth` as the host, e.g., `mycoolapp://auth`. The user signs in and then within `func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction) async -> WKNavigationActionPolicy`, you check the `navigationAction.request.url` to see if it has a query parameter of `code`. This is your **code** not your **token**. 

From there, you make a [call](https://docs.joinmastodon.org/methods/oauth/#token) to `POST /oauth/token` with the parameters of the `code` you just got along with `grant_type`, `client_id`, `client_secret`, and `redirect_uri` using the `MastodonKit` [function](https://github.com/MastodonKit/MastodonKit/blob/master/Sources/MastodonKit/Requests/Login.swift#L49-L53). Finally, you get the `Token` entity that you can use to authenticate network calls.  

From there, store that into the keychain and be on your way!


