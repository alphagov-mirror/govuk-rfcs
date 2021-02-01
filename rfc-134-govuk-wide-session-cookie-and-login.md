# A GOV.UK-wide session cookie & login

## Summary

We propose introducing a new cookie, `govuk_account_session`, which
will be an optional cookie, required only for users who wish to use
personalised parts of GOV.UK (currently just the Transition Checker).

Similarly to how our A/B tests work we will manage this cookie at the
Fastly layer, in Varnish Configuration Language (VCL), and use custom
request headers to pass the cookie value to our apps. We will also use
custom response headers to set a new cookie value.

We will create a new app to handle the login and logout process, and
to manage the OAuth tokens used to update a user's data.

## Problem

The GOV.UK Account team have launched an experiment on the Transition
Checker, allowing users to sign up to save their
results. [finder-frontend sets a session cookie][ff-session]—an
encrypted cookie—containing the user ID and OAuth tokens used to
update the data we hold on them.

This works fine for one app, but there are problems with this approach
when we try to scale to more than one app.

[ff-session]: https://github.com/alphagov/finder-frontend/blob/5b6101e8eb027028a102ab96bcef6ff752b9c558/app/controllers/sessions_controller.rb#L23-L28

### We need to pass cookies to our apps

Our Nginx configuration blocks most cookies, [which we disabled for
the Transition Checker][tc-cookie-pr]. If multiple apps need to
consume the session cookie, then this blocking will be disabled for
ever-increasing chunks of GOV.UK.

[tc-cookie-pr]: https://github.com/alphagov/govuk-puppet/pull/10788

### Apps need to share the same session cookie

It's no good if the user has to log into each part of GOV.UK
individually.  For example, say we personalise taxon pages: a user
shouldn't have to log into finder-frontend (for the Transition
Checker) and collections (for the taxon pages) separately.

There needs to be one session shared across them all.  If we use Rails
session cookies for this, we need to make sure all apps use the same
encryption key.

### Which app handles logging in and out?

If there is one session cookie used for all of GOV.UK, which app sets
that?

Somewhere there needs to be a login controller and a logout controller
which manipulate the cookie. These controllers will redirect the user
to the GOV.UK Account system to do the actual authentication, but we
need something on www.gov.uk itself to set the cookie.

### Non-personalised parts of GOV.UK need to manipulate the session cookie too

It's unlikely that personalisation will roll out to all of GOV.UK in
one go.  There will be islands of personalised content.  Currently
there is the Transition Checker.  Maybe next will be some guidance
pages, or something else.  We want to keep the user's session alive
while they are browsing the non-personalised parts of GOV.UK,
otherwise we risk this bad experience:

1. The user logs in to use some personalised part of GOV.UK.

2. The user then spends spends 30 minutes (or whatever we use for the
   session duration) viewing non-personalised parts of GOV.UK, but
   without leaving the site.

3. The user then tries to use another personalised part of GOV.UK, but
   their session has expired, because the non-personalised parts
   weren't bumping the expiration time on every page view.

### We can't cache as effectively

The Fastly docs have some [comments on the risks of cookies][fastly-cookies-risks]:

> Cookies can lead to undesirable outcomes.  At worst, if a cookie
> header is forwarded to your backend server, the backend uses the
> cookie value to generate personalized content, and that content is
> then cached by Fastly, a user may end up receiving content intended
> for someone else.  A theoretical solution to this, adding a Vary:
> Cookie header to the response, leads to another bad outcome: the
> response is most likely not cacheable at all, and Fastly will
> forward all requests to your backend.

[fastly-cookies-risks]: https://developer.fastly.com/reference/http-headers/Cookie/

## Proposal

### Manage the cookie entirely in VCL

[Fastly's best practices for cookies][fastly-cookies-practices]
suggest parsing cookies into custom headers and using these headers
for caching purposes, rather than caching based on the entire
Set-Cookie header (which in our case also contains non-account-related
things like A/B test bucket assignment, Google Analytics session ID,
and cookie consent preferences).

Rather than have multiple GOV.UK frontend apps worry about receiving
the cookie and bumping its expiration time, if we manage the cookie
entirely in VCL—parsing it into a new `GOVUK-Session-ID` header—we
only need to worry about returning appropriate response headers.

We will need to make three changes to our VCL.

[fastly-cookies-practices]: https://developer.fastly.com/reference/http-headers/Cookie/#best-practices

#### Preventing tampering with the cookie

We can't rely on client-side cookie timeouts to log users out.  If an
attacker gets a copy of a user's session cookie, they can include it
in any requests they send, regardless of whether it has expired.  So
we need to sign the cookie value and expiration time to prevent
tampering.

The cookie value will be of the form:

```ruby
"#{base64(session_id)}.#{base64(expiration_time)}.#{base64(signature)}"
```

Where the signature is computed with:

```ruby
hmac_sha256(secret_key, "#{session_id}.#{expiration_time}")
```

We will need a random string as a signing key, stored in a table:

```vcl
table env {
  "session_key": "random string goes here"
}
```

This approach is based on [jwt-vcl][], though we don't need a full JWT
implementation so I have simplified it.

[jwt-vcl]: https://github.com/phamann/jwt-vcl

#### Changes to `vcl_recv`

When receiving a request, validate the cookie: if it's ok, put the
session ID in the `GOVUK-Session-ID` header for our apps, if it's not
ok, unset it.

```vcl
if (req.http.Cookie ~ "govuk_account_session") {
  if (req.http.Cookie:govuk_account_session ~ "^([a-zA-Z0-9\-_]+)?\.([a-zA-Z0-9\-_]+)?\.([a-zA-Z0-9\-_]+)?$") {
    set req.http.GOVUK-Session-ID = digest.base64url_nopad_decode(re.group.1);
    set req.http.GOVUK-Session-Expires = digest.base64url_nopad_decode(re.group.2);
    set req.http.GOVUK-Session-Signature = digest.base64url_nopad_decode(re.group.3);
    set req.http.GOVUK-Session-Expected-Signature = digest.hmac_sha256(table.lookup(env, "session_key"), req.http.GOVUK-Session-ID + "." + req.http.GOVUK-Session-Expires);

    if(!digest.secure_is_equal(req.http.GOVUK-Session-Signature, req.http.GOVUK-Session-Expected-Signature) || time.is_after(now, std.integer2time(std.atoi(req.http.GOVUK-Session-Expires)))) {
      set req.http.GOVUK-End-Session = "true";
      unset req.http.GOVUK-Session-ID;
    }

    unset req.http.GOVUK-Session-Expires;
    unset req.http.GOVUK-Session-Signature;
    unset req.http.GOVUK-Session-Expected-Signature;
  } else {
    set req.http.GOVUK-End-Session = "true";
  }
}
```

#### Changes to `vcl_fetch`

When fetching a response from the backend, record the new
`GOVUK-Session-ID` value, if one is returned; and end the session, if
the user has been logged out:

```vcl
if (beresp.http.GOVUK-Session-ID) {
  set req.http.GOVUK-Session-ID = beresp.http.GOVUK-Session-ID;
}

if (beresp.http.GOVUK-End-Session) {
  set req.http.GOVUK-End-Session = "true";
  unset req.http.GOVUK-Session-ID;
}
```

If the response depends on the user session, it must either:

1. Set a `Vary: GOVUK-Session-ID` header, or
2. Set headers to prevent caching entirely

#### Changes to `vcl_deliver`

When delivering a response to the user, set a new cookie with a new
expiration time, and disable shared caches outside of Fastly (both
Fastly and the user's browser can still cache the page) if the
response depended on the session:

```vcl
if (req.http.GOVUK-End-Session == "true") {
    set req.http.GOVUK-Session = "";
} else if (req.http.GOVUK-Session-ID) {
  set req.http.GOVUK-Session-Expires = strftime({"%s"}, time.add(now, 1800s));
  set req.http.GOVUK-Session-Signature = digest.hmac_sha256(table.lookup(env, "session_key"), req.http.GOVUK-Session-ID + "." + req.http.GOVUK-Session-Expires));
  set req.http.GOVUK-Session = digest.base64url_nopad(req.http.GOVUK-Session-ID) + "." + digest.base64url_nopad(req.http.GOVUK-Session-Expires) + "." + digest.base64url_nopad(req.http.GOVUK-Session-Signature);
}

if (req.http.GOVUK-Session) {
  add resp.http.Set-Cookie = "govuk_account_session=" + req.http.GOVUK-Session + "; secure; httponly; samesite=strict; max-age=1800; path=/";
}

if (resp.http.Vary ~ "GOVUK-Session-ID") {
  unset resp.http.Vary:GOVUK-Session-ID;
  set resp.http.Cache-Control:private = "";
}
```

The `1800` here (30 minutes) is just an example.  We will work out an
appropriate session duration with product and security input.

### Create a new app to manage the auth process

It's weird to have the login and logout controllers for GOV.UK as a
whole located under `/transition-check`.

We will create a new app—called account-frontend, which will live on a
new machine class called personalisation—to serve those controllers
and to handle OAuth interactions with the GOV.UK Account
system. Signing in will set the `GOVUK-Session-ID` response header,
signing out will set the `GOVUK-End-Session` response header.

This app will serve these public endpoints:

- `GET /sign-in`: initiates the OAuth flow with the accounts system
  and sends the user on a consent journey.  Accepts these parameters:

    - `_ga`: cross-domain tracking parameter to pass to the accounts
      domain (optional)
    - `redirect_path`: path on GOV.UK to redirect back to after
      authenticating (optional, default: `/`)
    - `state`: see below (optional)

- `GET /sign-in/callback`: where the accounts system sends the user
  back to.  Sets the `GOVUK-Session-ID` response header and stores the
  returned OAuth access and refresh tokens in the app's database, if
  the user successfully signed in, and redirects the user back to the
  redirect_path.

- `GET /sign-out`: sets the `GOVUK-End-Session` response header,
  deletes the OAuth tokens from the app's database, and redirects the
  user to `/`.

The public endpoints are just part of redirection flows, they have no
visible response.

The app will also serve these internal endpoints:

- `GET /api/attributes`: looks up and returns some attributes from the
  user's account.  Accepts these parameters:

    - `session_id`: the value of the `GOVUK-Session-ID` header
    - `attributes`: list of attribute names

    Returns a hash of attribute names and values.

- `POST /api/attributes`: sets some attribute values on the user's
  account.  Accepts these parameters:

    - `session_id`: the value of the `GOVUK-Session-ID` header
    - `attributes`: hash of attribute names and values

  This is a partial update.  Attributes *not* named in the hash keep
  their previous values.

- `POST /api/state`: sets some attribute values that will be persisted
  if the user creates a new account, regardless of whether the user
  returns to GOV.UK.  Accepts these parameters:

    - `attributes`: hash of attribute names and values

    Returns an ID which can be passed to `/sign-in`.

Here are a few examples of how the Transition Checker will work:

#### Clicking the "Sign in" link in the header

1. The link sends the user to `/sign-in?redirect_path=...&_ga=...`

2. The user is redirected to `www.account.publishing....`

3. The user authenticates (registers or signs in)

4. The user is redirected to `www.gov.uk/sign-in/callback?...`

5. A session ID is generated, and used to persist the OAuth access and
   refresh tokens to the app's database

6. The `GOVUK-Session-ID` response header is set

7. The user is redirected to `redirect_path`

#### Clicking the "Create a GOV.UK account" button on the results page

1. The button sends the user to a new controller in finder-frontend,
   which:

   - Calls `/api/state` with the user's answers, generating an ID
   - Redirects the user to `/sign-in?state=...&redirect_path=...&_ga=...`

2. The new app passes the state attributes to the accounts system

3. The user is redirected to the accounts system, auths, and is sent
   back (as in steps 2, 3, and 4 above)—but if the user registers, the
   accounts system saves the provided attribute values

4. The new app creates the session and sends the user to the
   `redirect_path` (as in steps 5, 6, and 7 above)

This set-some-attributes-on-register flow is necessary because we
display a confirmation page after the user signs up.  This page
welcomes the user, says we've sent them a confirmation email, and
gives a link back to the service.  We want to persist the attributes
even if the user does not click that link.

We plan to remove the `state` things when we retire the Transition
Checker experiment.

### Local development

A Fastly-managed cookie won't work when running GOV.UK apps locally.

To support that use-case, if `Rails.env.development?`, the new app
will set an unencrypted `govuk_account_session` cookie, on the domain
`dev.gov.uk`, which contains the `GOVUK-Session-ID`, in addition to
sending the response headers.
