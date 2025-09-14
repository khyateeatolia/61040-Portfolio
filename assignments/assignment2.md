


# Assignment 2

---

## Question 1: Registry

One invariant is that the number of items requested must always equal the initial count minus the number of items purchased. Another invariant is that every purchase must correspond to an existing request in the same registry. The invariant about counts is more important because it ensures that givers cannot purchase more than what was originally requested, which would break the core purpose of the registry.  

The action most affected is **purchase**, and it preserves the invariant by creating a purchase and reducing the count of the matching request.  

---


The action **addItem** can break the important count invariant if new quantities are added after some purchases have already been made, since this can cause the totals to drift from the intended request balance.  

A fix would be to prevent the addition of items once the registry is active, or to treat new items as fresh requests that start with an independent count, thereby preserving the correctness of the count invariant.  

---


The specification allows a registry to be opened and closed multiple times, since neither action prevents reactivation after closure.  

Allowing this makes sense because a recipient might wish to reopen a registry if more gifts are desired, or if the time window for giving is extended.  

---

The absence of a delete action may matter in practice because users might want to remove old or erroneous registries to avoid clutter and confusion. However, since closed registries are no longer visible to givers, deletion is not strictly necessary and archival may be sufficient.  

---


- A common query for the registry owner is to see which items were purchased and by whom.  
- A common query for a giver is to see which items are still available in an active registry so that they do not duplicate someone else’s purchase.  

---

To allow the recipient to hide purchases, the state of the registry could include a flag such as `hidePurchases`, that is controlled by the owner.  

Actions that reveal purchase details would then check this flag before returning information, thereby enabling surprise when desired.  

---

Using generic types for `User` and `Item` is preferable because it makes the concept reusable in many systems. For instance, `Item` can be defined in a commerce system by SKU codes or product IDs, while names, prices, and descriptions belong in other concepts.  

This avoids duplication and allows flexibility across domains.  

---

# Question 2: User Authentication

## Concept State

The state should include a set of users, each with a username, a password, and a flag indicating whether the user is confirmed. This ensures that the system can check uniqueness of usernames, verify authentication against stored credentials, and enforce whether a user is active or not.


```
state
  a set of Users with
  a username String
  a password String
  a confirmed Flag
```

---

## Register Action



register (username: String, password: String): (user: User)
requires no user exists with this username
effects create a new user with this username and password and confirmed set to false



---

## Authenticate Action



authenticate (username: String, password: String): (user: User)
requires a user exists with this username and password and confirmed is true
effects return that user



---

## Essential Invariant

The essential invariant is that no two users can have the same username.  

This ensures that each login is unambiguous and consistent. The invariant is preserved by the register action because it explicitly requires that no user with the same username already exists before creating a new one.  

---

## Extension with Email Confirmation



state
a set of Users with
a username String
a password String
a confirmed Flag
a token String

register (username: String, password: String): (user: User, token: String)
requires no user exists with this username
effects create a new user with this username and password and confirmed set to false
generate a secret token and associate it with the user
return the user and the token

confirm (username: String, token: String)
requires a user exists with this username and this token
effects set confirmed to true for this user



---

# Question 2: Personal Access Tokens



concept PersonalAccessToken \[User]
purpose allow users to authenticate using tokens with defined permissions instead of their main password
principle after a user creates a personal access token (classic) with selected scopes, expiration, and name, they can use that token instead of their password for allowed operations; the user can revoke or authorize the token (for SSO orgs), and the token is valid only if active, not expired, and properly authorized

state
a set of Users with
a username String
a password String
a set of Tokens

a set of Tokens with
a user User
a tokenString String
a name String
a scopes set of String
an expiration DateTime (optional)
an active Flag
an ssoAuthorized Flag  # for organizations requiring single sign-on

actions
createToken (user: User, name: String, scopes: set of String, expiration: DateTime (optional)): (token: Token)
requires user exists
effects create a new Token associated with the user, with given name, scopes, expiration, active = true, ssoAuthorized = false; return the new Token

authenticateWithToken (username: String, tokenString: String): (user: User)
requires a Token exists with tokenString, is active, not expired, whose user has username, ssoAuthorized = true if required by org
effects return that user

revokeToken (token: Token)
requires token exists and is active
effects set active = false for the token

authorizeTokenSSO (token: Token, organization: Organization)
requires token exists, user belongs to organization, and organization uses SSO
effects set ssoAuthorized = true for that token



---

## Differences vs. Password Authentication

- **PasswordAuthentication** uses a single, static credential.  
- **PersonalAccessToken** introduces separate, revocable credentials with scopes, expiration, and SSO enforcement.  
- Tokens can be managed independently, and revoking a token does not affect the password.  
- Passwords are all-powerful but inflexible; tokens are limited but safer for automation.  

---

## Suggested Improvements to GitHub Documentation

- Include a comparison table of PasswordAuthentication vs. PersonalAccessToken (classic) vs. fine-grained PAT.  
- Highlight security implications (when to use tokens vs. passwords).  
- Clarify SSO authorization requirements for orgs.  
- Explain policy compliance for existing tokens (e.g., forced expiry).  

---

# Question 4: Familiar Concepts

## Concept 1: URLShortener



concept URLShortener \[User, URL]
purpose provide short, unique links that redirect to longer URLs
principle a user submits a long URL and either specifies a suffix or gets an autogenerated suffix; the system maps the suffix to the original URL; when someone visits the short link, they are redirected to the full URL
state
a set of Mappings with
an owner User
a suffix String
a target URL
an active Flag
actions
createMapping (owner: User, url: URL, suffix: String): (mapping: Mapping)
requires suffix not already in use
effects create new mapping with this suffix and target URL, active = true

createMappingAuto (owner: User, url: URL): (mapping: Mapping)
requires true
effects generate a unique suffix, create new mapping with target URL and active = true

deactivateMapping (mapping: Mapping)
requires mapping exists and is active
effects set active = false

resolve (suffix: String): (url: URL)
requires mapping exists with this suffix and is active
effects return target URL



**Notes:** Must prevent suffix collisions. Deactivation allows removal without losing history.  

---

## Concept 2: BillableHoursTracking



concept BillableHoursTracking \[Employee, Project]
purpose track time spent by employees on projects for billing clients
principle an employee starts a session by selecting a project and providing a description; the system records the start time; the employee ends the session and the system records the end time; unended sessions can be closed automatically
state
a set of Sessions with
an employee Employee
a project Project
a description String
a startTime DateTime
an endTime DateTime (optional)
a closed Flag
actions
startSession (employee: Employee, project: Project, description: String): (session: Session)
requires no active session exists for this employee
effects create new session with given project, description, startTime = now, closed = false

endSession (session: Session)
requires session exists and not closed
effects set endTime = now, closed = true

autoClose (session: Session, cutoff: DateTime)
requires session exists, not closed, startTime < cutoff
effects set endTime = cutoff, closed = true

queryHours (employee: Employee, project: Project): (totalHours: Number)
requires true
effects return sum of (endTime - startTime) for all closed sessions of this employee on this project


**Notes:** `autoClose` ensures forgotten sessions are handled, maintaining billing accuracy.  

---

## Concept 3: TimeBasedOneTimePassword (TOTP)


concept TimeBasedOneTimePassword \[User]
purpose strengthen authentication by requiring a temporary numeric code from the user’s device in addition to their main password
principle each user registers a secret with the system; their device uses the secret to generate time-based codes; the system accepts codes only if they match the secret within a time window; codes expire quickly and cannot be reused
state
a set of Users with
a username String
a password String
a secret String
a lastAcceptedCode String (optional)
actions
registerTOTP (user: User, secret: String)
requires user exists
effects associate secret with user

generateCode (user: User, time: DateTime): (code: String)
requires user exists
effects compute code based on secret and current time; return it

authenticateTOTP (username: String, password: String, code: String, time: DateTime): (user: User)
requires user exists with username and password
and code matches secret for time within allowed window
and code ≠ lastAcceptedCode
effects return user; set lastAcceptedCode = code

