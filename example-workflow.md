# WebID-OIDC Detailed Sign In Workflow

Note: For the purposes of this spec, we're going to leave out the question of
Authorization (access control), and assume that the POD is performing
appropriate access control checks, and that the user has access to the resources
they are requesting.

### 1. Initial Request

#### 1a. Direct Browser Request

*Example 1:* Alice types in `https://bob.example/images/image1.png` into her
browser's address bar (or follows a link to it).

*Example 1:* `bob.example` responds with a `401 Unauthorized` response, with HTML in
the body that says something like "Welcome to bob.example. Please enter your email
or WebID URL", and provides a text field.

A Browser makes a request to a POD acting as a Resource Server and
Relying Party. This is a direct browser request, such as when a user follows a
link or types in a URL in the address bar. No existing cookie is present, for
either the Provider POD (`alice.example`) or the Relying Party POD (`bob.example`).

The Relying Party, (here, `bob.example`) responds with an HTTP `401 Unauthorized`.
The *body* of the 401 response SHOULD contain human-readable HTML, containing
either a Select Provider form, or a meta-refresh redirect to a Select Provider
page. This HTML response starts the user on the Sign In workflow.

#### 1b. AJAX or API Client Request
A REST client (either for a server-side app, or an in-browser AJAX client)
makes an HTTP request to a POD acting as a Resource Server and Relying Party.
No existing cookie or session is present, for either the Provider POD or the
Relying Party POD.

The server (RP POD) responds with an HTTP `401 Unauthorized`. If this is an
interactive client, it is the client's responsibility to initiate the Sign In
workflow by presenting the user with a Select Provider UI.

### 2. Provider Selection and Discovery

#### 2.1 Provider Selection
*Example 1:* Alice (still on the `401 Unauthorized` page of the Relying Party
`bob.example`) enters the URI of her Home POD, `alice.example`, during the Provider
Selection step.

If the user's Provider preference is not saved (in a cookie or local storage),
at this point they must be prompted with the Provider Selection UI. The user
must indicate which POD or identity Provider they wish to use, for signing in.
Any and all of the following methods may be used:

 * User enters in the domain of their identity Provider (such as
   `solidtest.space`).
   (If no protocol is specified by the user, `https` should be assumed.)
 * User clicks on a provider's logo
 * User enters in their full WebID URI
 * The user enters in their email. The RP then uses the WebFinger protocol (with
   a fallback to WebFist), as recommended by the OIDC spec, to discover the URI
   of the Provider.

#### 2.2 Provider Discovery
*Example 1:* The Relying Party, `bob.example`, performs OIDC provider metadata
discovery on `https://alice.example`, and loads its public keys, API endpoints,
and other metadata.

If the URI of the user's preferred identity Provider was entered during Provider
Selection, the Relying Party must perform Provider Discovery to obtain
[Provider Metadata](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderMetadata)
This metadata includes useful information such as:
 - API endpoints (for authorization, client registration, and so on)
 - The Provider's public keys, which can be used to verify signed tokens
 - Crypto algorithms supported
 - Links to Policy and Terms of Service documents

See the section [Obtaining OpenID Provider Configuration
Information](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig)
of the OIDC Discovery spec, for more information.

#### 2.3 Dynamic Client Registration (First Time Only)
*Example 1:* Since this is the first time the Provider (`alice.example`) and the
Relying Party (`bob.example`) have interacted, `bob.example` must dynamically register
itself *as a client/relying party* to `alice.example`. Bob's POD (in the RP role)
performs dynamic registration with Alice's POD (in the Provider role), and as
a result `bob.example` receives its very own `client_id` which identifies it
uniquely as a Relying Party to `alice.example`.

If this is the first time a Provider and a Relying Party are encountering each
other, the RP must perform
[Dynamic Client Registration](https://openid.net/specs/openid-connect-registration-1_0.html).
Note: This is an operation that happens under the hood, and does not involve the
user. All compliant OIDC clients have this functionality built in.

### 3. Local Authentication to Provider
*Example 1:* After performing Provider Selection and Discovery, `bob.example`
redirects Alice's browser (via a `302 Found` redirect) to her own Provider's
sign in page, `https://alice.example/signin`. Alice signs into `alice.example` using
a username and password.

The user is redirected to their preferred Provider's Sign In page.

The exact mechanism used to sign in is up to the provider, and all of the usual
choices apply (WebID-TLS browser-side certificates, username and password,
hardware-based FIDO 2/WebAuthentication devices, federated sign-in with the
likes of Github/Facebook/Google and so on).

If the user does not have an account with the provider, the Sign Up/Account
Creation step would happen at this point, instead.

(**Optimization**) The Provider may choose to establish a user session (via a
browser cookie), so that the user can skip Step 3 in subsequent interactions
(until the session expires or the user signs out).

### 4. User Consent
User consent screen ("Do you want to give www.example.com access to your Pod?" etc).

*Example 1:* `alice.example`, in the Provider role, presents Alice with a User
Consent screen that says something like "Do you want to sign in to bob.example?",
and she clicks OK.

### 5. Authentication Response
After the user has signed in and provided consent, the Provider performs client
verification.

*Example 1:* `alice.example`, in the Provider role, verifies the Relying Party
(`bob.example`) that initiated the authentication process, and redirects Alice's
browser to a pre-registered `redirect_uri` that `bob.example` provided during
the previous Dynamic Registration step. So, Alice's browser gets redirected
to `https://bob.example/auth-callback` (which is the URI Bob's POD uses as an OIDC
`redirect_uri` endpoint).

(**Optimization**) The Relying Party may choose to establish a user session
(via a browser cookie), so that the user can skip Steps 1-5 in subsequent
interactions (until the session expires or the user signs out).

### 6. Deriving a WebID URI
*Example 1:* At this point, Alice is back to the original resource she was
trying to request. As part of the Authentication Response from `alice.example`,
the Relying Party `bob.example` has received an ID token that contains, among
other things, a `webid` property. `bob.example` validates the ID Token (makes
sure it has not expired, that the signature matches `alice.example`'s public key,
and so on). The contents of the ID Token's `webid` claim is Alice's WebID URI,
`https://alice.example/#i`.

### 7. WebID Provider Confirmation
The Relying Party `bob.example` then performs the [WebID Provider
Confirmation](README.md#webid-provider-confirmation) step to make
that Alice in fact authorized `alice.example` as her identity provider.
