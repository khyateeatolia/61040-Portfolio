# Assignment 3

## Concept Questions

### Purpose of Contexts

Contexts exist so that we can generate unique strings relative to some namespace rather than globally. Without contexts, the `NonceGeneration` concept would have to guarantee that no string is ever reused anywhere, which is often unnecessary and too restrictive. Contexts let us assign unique scopes to a particular domain.

In the URL shortening app, the natural context is the **shortUrlBase** (the domain part of the short URL). For example, if the service allows different users to use their own domains (like `tinyurl.com` vs. `sho.rt` vs. `mydomain.org`), then uniqueness only needs to be enforced within each base. So, in the URL shortening app, each `shortUrlBase` would be a separate context.

---

### Storing Used Strings

The `NonceGeneration` concept must store sets of used strings because its principle requires that every generated string be unique within a given context, which means the system needs to keep track of what has already been produced in order to prevent duplicates.

In the case of a counter implementation, the abstract set of used strings specified in the concept corresponds exactly to the set of all strings generated from the counter values up to the current one—for example, if the counter is at value *n*, then the abstract set of used strings is `{ f(0), f(1), …, f(n) }`, where *f* is the encoding function. Thus, the counter serves as a compact representation of the entire set, and the abstraction function that maps between the implementation and the specification interprets the counter as standing for all the strings that have already been generated.

---

### Words as Nonces

**Advantage (for users):**
Easier to remember and share. A short URL like `sho.rt/apple-banana` is more human-friendly than `sho.rt/X9qT2z`. Users can also type it directly without the need to copy-paste.

**Disadvantage (for users):**
Less space-efficient. The pool of common words is much smaller than the pool of arbitrary alphanumeric strings of the same length, so URLs may need to be longer (multiple words) to avoid collisions. Also, some generated words/phrases may be offensive or embarrassing unless filtered.

**Modified Concept:**

```plaintext
concept WordNonceGeneration [Context]
purpose generate unique, human-readable words as strings
principle each generate returns a word not returned before for that context
state
  a set of Contexts with
    a used set of Words
actions
  generate (context: Context): (nonce: Word)
    effect returns a word from the dictionary not already in used for that context
```

This is essentially the same structure, but we restrict the universe of possible nonces to a dictionary. If multiple words are needed for uniqueness, the action could concatenate them.

---

## Synchronization Questions

1. **Partial matching.**
   The `generate` sync only binds `shortUrlBase` because at that point the system only needs the domain to choose the correct context for nonce generation; the `targetUrl` is irrelevant to producing a fresh nonce. The `register` sync must include both the generated nonce and the `targetUrl` because it is the step that actually creates the shortening (it needs the nonce to form the short URL and the target to associate with it).

2. **Omitting names.**
   Omitting `x: x` is syntactic sugar used when an argument/result name and the bound variable are identical and there is no risk of confusion. It isn’t used everywhere because explicit names avoid ambiguity (e.g., when the same variable name could be bound from different sources, when names differ across concepts, or when clarity/readability is important). Explicit naming prevents accidental variable capture and makes mappings obvious when multiple bindings are involved.

3. **Inclusion of request.**
   The request action appears in the first two syncs because those syncs respond directly to a user-initiated `Request.shortenUrl`. The request supplies inputs that must be bound and threaded through the syncs. The third sync, `setExpiry`, is a system-follow-up triggered by the completion of `UrlShortening.register`, so it doesn’t need the original request as part of its when-clause.

4. **Fixed domain.**
   If the service always used `"bit.ly"`, remove `shortUrlBase` from the request and hardwire the context/domain. For example:

   ```plaintext
   sync generate
   when Request.shortenUrl ()
   then NonceGeneration.generate (context: "bit.ly")

   sync register
   when Request.shortenUrl (targetUrl)
        NonceGeneration.generate (): (nonce)
   then UrlShortening.register (shortUrlSuffix: nonce, shortUrlBase: "bit.ly", targetUrl)
   ```

   Alternatively, you could remove the `shortUrlBase` parameter from `UrlShortening.register` entirely and have the concept assume the fixed domain.

5. **Adding a sync (expiration → deletion).**

   ```plaintext
   sync expire
   when ExpiringResource.expireResource (): (resource: shortUrl)
   then UrlShortening.delete (shortUrl)
   ```

   This ensures that when the system action returns an expired resource, the corresponding short URL mapping is deleted.

---

## Extending the Design: Analytics

### New Concepts

**Analytics**

```plaintext
concept Analytics
purpose track and report the number of accesses to short URLs
principle every time a short URL is looked up, the access count is incremented
state
  a set of Records with
    shortUrl String
    count Number
actions
  increment (shortUrl: String)
    effect increases the count for shortUrl by 1
  getCount (shortUrl: String): (count: Number)
    requires a record exists for shortUrl
    effect returns the current count
```

**Ownership**

```plaintext
concept Ownership
purpose associate each short URL with the user who registered it
principle only the owner of a short URL can examine its analytics
state
  a set of Mappings with
    shortUrl String
    owner User
actions
  registerOwnership (shortUrl: String, owner: User)
    effect associates shortUrl with the given user
  checkOwnership (shortUrl: String, user: User): (ok: Boolean)
    effect returns true if user is the owner of shortUrl
```

---

### Synchronizations

1. **When shortenings are created (bind ownership and analytics):**

   ```plaintext
   sync initAnalytics
   when UrlShortening.register (): (shortUrl), Request.shortenUrl (user)
   then Ownership.registerOwnership (shortUrl, user),
        Analytics.increment (shortUrl) // initialize with 0 → 1
   ```

2. **When shortenings are translated (increment count on lookup):**

   ```plaintext
   sync trackAccess
   when UrlShortening.lookup (shortUrl)
   then Analytics.increment (shortUrl)
   ```

3. **When a user examines analytics (only if they are owner):**

   ```plaintext
   sync viewAnalytics
   when Request.viewAnalytics (shortUrl, user),
        Ownership.checkOwnership (shortUrl, user): (ok = true)
   then Analytics.getCount (shortUrl)
   ```

---

### Feature Requests

* **Allowing users to choose their own short URLs**
  Extend `Request.shortenUrl` with an optional custom suffix. Add a sync to check uniqueness before calling `UrlShortening.register`.

* **Using the “word as nonce” strategy**
  Replace `NonceGeneration` with a `WordNonceGeneration` concept that draws from a dictionary of words. Other concepts remain unchanged.

* **Including the target URL in analytics**
  Add `targetUrl` to the `Analytics` state. Modify `increment` to record both `shortUrl` and `targetUrl`, enabling grouping of statistics by target URL.

* **Generate short URLs that are not easily guessed**
  Modify the implementation of `NonceGeneration` to use stronger randomness or longer encodings. The abstract design remains the same.

* **Supporting reporting of analytics to creators who are not registered**
  This is undesirable. It undermines privacy and ownership guarantees. Analytics should remain private and viewable only by registered owners.

---

Would you like me to also add a **front-matter section** (title, course, assignment number, student name) at the very top, so it’s submission-ready for a class?
