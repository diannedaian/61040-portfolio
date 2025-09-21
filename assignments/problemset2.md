# Pset 2 - Dianne Cao

## Concept Questions

### 1. Context

Contexts in NonceGeneration are namespaces that scope where uniqueness must be preserved. In the URL shortening app, the natural context is the short URL base (domain), so that each domain maintains its own pool of unique suffixes. For example, given `tinyurl.com/abc` and `myshort.ly/abc`, they can co-exist because they’re in different bases.

### 2. Storing used strings

The NonceGeneration concept must store sets of used strings in its specification so that the invariant “no duplicate strings in a context” is explicit. An implementation using a counter doesn’t literally keep the set; instead, the counter determines a unique string each time it increments. The abstraction function maps the counter’s current value to the set of all strings that would have been generated so far, so the counter is simply a compact representation of that used-string set.

### 3. Words as nonces

Using common dictionary words as nonces makes short URLs easier for users to read, remember, and share, which improves usability. But the downside is that the space of possible words is much smaller, so collisions happen sooner and the system may run out of fresh nonces faster than with random strings. Another problem with dictionary-word nonces is that the generated URL might be formed from a random word that feels unrelated. To realize this idea, the NonceGeneration concept would be modified so that each context’s used set is drawn from a predefined dictionary pool of words, and the generate action chooses an unused word from that pool instead of producing an arbitrary string.

## Synchronizations for URL Shortening

### 1. Partial matching

In the first sync, only the shortUrlBase is needed because its job is to trigger nonce generation, and the NonceGeneration concept only requires a context (here, the base domain) to produce a new string. The targetUrl is irrelevant at this step, so it’s omitted. In the second sync, both shortUrlBase and targetUrl are included because UrlShortening.register needs both pieces: the base to determine where the short link lives, and the targetUrl to know what the short link should resolve to.

### 2. Omitting Names

The omission convention works only when it’s unambiguous that the argument name and the bound variable are the same. In cases where multiple arguments or results could be confused, or when the variable needs a different name than the action’s argument (for clarity or to pass it to another action), the explicit name: variable form is required. Without that, the specification could become unclear or even misleading about which values are being bound and passed along.

### 3. Inclusion of request

The first two syncs start from a user’s explicit request to shorten a URL, so they include Request.shortenUrl in their when clause to bind the user’s input (targetUrl and shortUrlBase) into the flow. The third sync, setExpiry, doesn’t originate from a user request — it’s triggered automatically whenever a short URL is successfully registered. Since its when clause is based on the completion of UrlShortening.register, there’s no need to mention the original request again.

### 4. Fixed Domain

If the service always used a fixed domain like bit.ly, then the shortUrlBase argument would no longer need to be passed around. In the synchronizations, every reference to shortUrlBase could simply be replaced with the constant "bit.ly".

- In generate, instead of binding shortUrlBase from the request, the sync would call NonceGeneration.generate(context: "bit.ly").
- In register, the when clause wouldn’t need shortUrlBase at all, and the then clause would invoke UrlShortening.register(shortUrlSuffix: nonce, shortUrlBase: "bit.ly", targetUrl).
- setExpiry would stay the same, since it only depends on the result of UrlShortening.register.

### 5. Adding a Sync

```
sync expireDelete
when ExpiringResource.expireResource (): (resource: shortUrl)
then UrlShortening.delete (shortUrl)
```

## Extending the design

### 1.

#### AccessLogging

```
concept AccessLogging [ShortUrl]
purpose track how many times each shortened URL is used
principle every time a shortened URL is looked up, increment its counter

state
  a set of Logs with
    a shortUrl ShortUrl
    a count Number

actions
  createLog (shortUrl: ShortUrl)
    requires no log exists for this shortUrl
    effects create a log for this shortUrl with count = 0

  recordAccess (shortUrl: ShortUrl)
    requires log exists for this shortUrl
    effects increment count for this shortUrl by 1

  getCount (shortUrl: ShortUrl) : (count: Number)
    requires log exists for this shortUrl
    effects return the current count for this shortUrl
```

#### AccessReporting

```
concept AccessReporting [ShortUrl, User]
purpose let the owner of a shortened URL view its usage count
principle owners can query counts for their own URLs but not others

state
  a set of Ownerships with
    a shortUrl ShortUrl
    an owner User

actions
  recordOwnership (owner: User, shortUrl: ShortUrl)
    requires no ownership exists for this shortUrl
    effects create ownership binding shortUrl to owner

  reportAccess (owner: User, shortUrl: ShortUrl) : (count: Number)
    requires ownership exists with this owner and shortUrl
    effects return the count for this shortUrl
```

### 2.

#### Synchronization 1: On Creation

```
sync createLogAndOwnership
when UrlShortening.register (): (shortUrl), Request.shortenUrl (owner, targetUrl, shortUrlBase)
then
  AccessLogging.createLog (shortUrl)
  AccessReporting.recordOwnership (owner, shortUrl)
```

#### Synchronization 2: On Lookup

```
sync incrementOnLookup
when UrlShortening.lookup (shortUrl)
then AccessLogging.recordAccess (shortUrl)
```

#### Synchronization 3: On Analytics Report

```
sync provideReport
when Request.viewAnalytics (owner, shortUrl), AccessLogging.getCount (shortUrl): (count)
then AccessReporting.reportAccess (owner, shortUrl): (count)
```

- Sync 1 ensures that whenever a new shortening is registered, an analytics log is created and ownership is recorded for the user.
- Sync 2 increments the access counter whenever the short URL is translated to its target.
- Sync 3's when gathers (a) the user’s request and (b) the completion of getCount; the then invokes the reporting action and fulfills its return with the bound count

### 3.
If authentication is enabled, the request must carry an authenticated user; add a sync from PasswordAuthentication.authenticate so ownership/permissions are bound to that identity.

#### 1. Allowing users to choose their own short URLs
This feature requires adding a new action `registerCustom` to the `UrlShortening` concept that accepts a user-specified suffix. The action must check that no existing shortening with the same `(base, suffix)` exists, preserving the invariant. To integrate with the rest of the system, a sync from `Request.shortenUrl` would trigger `UrlShortening.registerCustom`, and further syncs would still create analytics logs (`AccessLogging.createLog`) and ownership records (`AccessReporting.recordOwnership`). If authentication is required, the request would need to carry an authenticated user identity to bind the ownership correctly.

#### 2. Using the “word as nonce” strategy
To support dictionary-based suffixes, the `NonceGeneration` concept can be extended with an action like `generateWordNonce(context)`. The invariant that nonces are unique within a context still holds, but now the pool is finite, so collisions will be more likely. A sync would be needed to invoke this strategy when a user requests a “memorable” shortening. However, since the system already supports `registerCustom`, this feature is partially redundant: advanced users can pick their own memorable suffixes. The main benefit is to offer friendlier defaults for users who don’t specify, but it comes with trade-offs in scalability and guessability.

#### 3. Including the target URL in analytics
Right now, analytics are tied to short URLs only. To group lookups across different short URLs pointing to the same target, we could either (a) add the `targetUrl` into the state of `AccessLogging`, or (b) define a new concept `TargetAnalytics` keyed by `targetUrl`. In both designs, synchronizations must be added: when `UrlShortening.register` completes, initialize a target-level log entry; and when `UrlShortening.lookup` completes, increment both the per-shortening log and the target-level log. This way, an owner can view traffic grouped by either the short link or the underlying target.

#### 4. Generate short URLs that are not easily guessed
To improve security, the `NonceGeneration` concept could switch from sequential counters to cryptographically strong random nonces. This still satisfies the uniqueness invariant but makes URLs much harder to guess. Synchronizations would not need to change, since the caller doesn’t depend on how the nonce is generated. This feature somewhat conflicts with the “word as nonce” strategy: one prioritizes usability, the other security. A practical design could offer both, letting the user or service operator choose between random and word-based generation depending on the use case.

#### 5. Reporting analytics to creators without registration
Providing analytics to non-registered users is not nessasary because it undermines accountability and risks leaking data.
