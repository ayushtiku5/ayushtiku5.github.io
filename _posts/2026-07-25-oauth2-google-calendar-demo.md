---
layout: post
title: "A Minimal OAuth 2.0 Flow Against Google Calendar"
date: 2026-07-25 13:00:00 -0000
categories: systems
---

OAuth is one of those things everyone integrates against and almost nobody builds from the server side. I wanted to see the whole authorization code flow end to end, not through a library that hides the handshake, so I wrote a small Go server that logs a user in with Google and pulls their next ten calendar events. Code is here: [github.com/ayushtiku5/oauth-demo](https://github.com/ayushtiku5/oauth-demo).

It's about 170 lines. No framework, no database, just `net/http` and the `golang.org/x/oauth2` package. Small enough to read in one sitting, which was the point.

## The flow, in four hops

The whole thing is four HTTP handlers, each one a step in the authorization code flow:

```
Browser          Go Server              Google OAuth         Google Calendar API
   |                  |                      |                      |
   |  GET /auth/google|                      |                      |
   |----------------->|                      |                      |
   |                  | redirect to accounts.google.com             |
   |<-----------------|                      |                      |
   |  user logs in and consents              |                      |
   |---------------------------------------->|                      |
   |  redirect to /auth/google/callback?code=...                    |
   |<----------------------------------------|                      |
   |  GET /callback   |                      |                      |
   |----------------->|                      |                      |
   |                  | POST /token (code)   |                      |
   |                  |--------------------->|                      |
   |                  | { access_token, ... }|                      |
   |                  |<---------------------|                      |
   |  GET /calendar   |  GET /calendar/v3/... Bearer <access_token>  |
   |----------------->|-------------------------------------------->|
   |  JSON events      |<--------------------------------------------|
   |<-----------------|                                             |
```

`/` just renders a login link. Everything interesting starts at `/auth/google`.

## Step 1: kick the browser to Google

```go
func handleGoogleLogin(w http.ResponseWriter, r *http.Request) {
	state := randomState()
	http.SetCookie(w, &http.Cookie{
		Name:     "oauth_state",
		Value:    state,
		Expires:  time.Now().Add(10 * time.Minute),
		HttpOnly: true,
	})

	url := oauthConfig.AuthCodeURL(state, oauth2.AccessTypeOffline)
	http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}
```

Two things worth calling out here. First, `state` is a random nonce, generated fresh per login attempt and stashed in an `HttpOnly` cookie. Google echoes it back on the callback, and the handler checks it matches before doing anything else. Skip this and you've built a CSRF hole: an attacker could send a victim a crafted callback URL with the attacker's own authorization code, and the victim's browser would happily complete the exchange and bind the attacker's Google account to the victim's session.

Second, `oauth2.AccessTypeOffline`. Without it, Google hands back an access token that expires in about an hour and nothing else. With it, the token response includes a refresh token too, so the app can get new access tokens later without sending the user back through the consent screen every time.

## Step 2: the callback, where the client secret actually gets used

```go
func handleGoogleCallback(w http.ResponseWriter, r *http.Request) {
	cookie, err := r.Cookie("oauth_state")
	if err != nil || cookie.Value != r.URL.Query().Get("state") {
		http.Error(w, "invalid oauth state", http.StatusBadRequest)
		return
	}

	code := r.URL.Query().Get("code")
	token, err := oauthConfig.Exchange(context.Background(), code)
	...
}
```

This is the part that makes the authorization code flow safer than the older implicit flow. The browser only ever sees a short-lived, single-use `code` in the redirect URL. It never sees the client secret and it never sees the access token directly. The actual exchange, code plus client secret for an access token, happens server to server in `oauthConfig.Exchange()`. If someone intercepts the redirect and steals the code, it's a single-use value scoped to this exact client, so there's not much they can do with it alone.

The token that comes back gets dropped into an in-memory map keyed by a freshly generated session ID, and that session ID goes into a second `HttpOnly` cookie. Deliberately a prototype shortcut, a real deployment would want a persistent, encrypted store, but it's enough to keep the demo's state model honest: session cookie in the browser, token on the server, never the token in the browser.

## Step 3: use the token, and never think about refreshing it

```go
httpClient := oauthConfig.Client(context.Background(), token)
svc, err := calendar.NewService(context.Background(), option.WithHTTPClient(httpClient))
```

This was the part that surprised me even though I knew it existed. `oauthConfig.Client()` doesn't just attach the bearer token to requests, it returns an `http.Client` wrapping a `TokenSource` that checks expiry on every call and silently exchanges the refresh token for a new access token when needed. The calendar handler never checks `token.Expiry` itself. It just calls the API, and the transport underneath deals with staleness. That's the whole value of `AccessTypeOffline` from step 1: it's what makes this transparent refresh possible in the first place.

From there it's a normal API call: list the next 10 events on the primary calendar, ordered by start time, single events only (so recurring meetings show individual occurrences instead of the recurrence rule).

## What I'd change before this touched real users

The in-memory `map[string]*oauth2.Token` is the obvious one. It's not persisted, not encrypted, and every session vanishes on restart. Fine for a demo running on localhost, not fine for anything with more than one server process or an uptime requirement.

The scope requested is `calendar.readonly`, which is the right instinct: ask for the narrowest scope that does the job. It's worth checking every time a flow like this gets extended that a new feature doesn't quietly widen the scope to something the app doesn't actually need.

## Why this was worth building

Most of the OAuth code I've written before this has been consuming someone else's library, `passport` strategies, Spring Security starters, whatever the framework provides, and trusting that the state check and the token exchange happen correctly underneath. Writing the four handlers by hand made it obvious why each piece exists: the state cookie is there specifically to stop a forged callback, the code-for-token exchange is server-side specifically so the secret and the token never touch the browser, and the refreshing `http.Client` is there so application code never has to reason about token lifetime. None of that is surprising in hindsight, but it's a different kind of understanding than reading the RFC.
