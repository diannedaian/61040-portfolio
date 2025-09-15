# Pset 1 - Dianne Cao

## Exercise 1

### 1. Invariants

Two invariants are:

1. Count consistency. The count of a request must equal the original requested quantity minus the sum of purchase counts for that item. This means the number displayed in the registry is always exactly how many are still available, and it can never go negative.

2. Purchases require requests. Every purchase must belong to an existing request in a registry. In other words, you can’t buy an item unless the recipient asked for it and it appears on their registry.

The more important invariant is **(1) count consistency**, because it guarantees the registry accurately reflects what’s still available for givers.

- The goal of the gifting platform is to not get duplicate gifts and if there is a mess up at the count consistency it undermines the goal

The action most affected is **`purchase`**. Because it preserves the invariant by requiring the count is available and then decrementing the request count while recording the purchase.

### 2. Fixing an action

An action that can break is **`removeItem(registry, item)`**.
Removing an item might cause a break in count consistency in the scenerio of:

- **Existing purchases already made for that request.**
  If `removeItem` simply deletes the `Request`, any attached `Purchases` become orphaned. Then the equality
  `total_requested = total_purchased + total_remaining`
  can no longer be computed or displayed correctly (the “remaining” side drops to 0 because the request vanished, while “purchased” still exists somewhere, or gets lost entirely).

- **Remove → re-add of the same item.**
  After some units are purchased, the owner calls `removeItem`, then later `addItem` for the same `Item`. The new `Request` starts with a “fresh” count, ignoring past purchases, so the registry can show more remaining than should be available.

A simple fix is to not allow an item to be removed if any purchases have already been made. Another approach is to let the item be “archived” instead of deleted — it would disappear from the public list so new givers can’t see or buy it, but the history of past purchases would still be kept for the recipient. This way the registry stays consistent and everyone’s records make sense.

### 3. Inferring behavior

A registry **can be opened and closed repeatedly**. The `open` action only requires that the registry exists and is not already active, and the `close` action only requires that it exists and is active. There is no restriction in the spec that prevents an owner from calling `open` again after a `close`.
It should be flexible because maybe the user decided to close it and then regrets and wants to sent it to more people, then if the close is permanent the registry can't be used anymore. Also the user might want to reused stuff.

### 4. Registry Deletion

In practice this probably won't matter, since closing the registry already hides it from givers and preserves purchase history for the recipient. Still, some users may want deletion for tidiness, so an “archive” option could be a practical compromise.

### 5. Queries

Two common queries are:

1. **By the registry owner:** _“Which items have been purchased, and by whom?”_
   This helps the recipient track what gifts they will receive and send thank-you notes.

2. **By a gift giver:** _“Which items are still available to purchase, and how many are left?”_
   This ensures givers can avoid duplicates and choose from the remaining options.

### 6. Hiding purchases

A common feature of gift registries is letting the recipient hide purchase information so that the gifts remain a surprise. To support this, the concept specification could be augmented with:

- **New state:** add a flag on each `Registry`, e.g. `hidePurchases: Boolean`.
  When set to `true`, purchase details are hidden from the recipient.

- **New action:**

  - `setHidePurchases(registry: Registry, flag: Boolean)`
  - _Requires:_ registry exists.
  - _Effects:_ update the `hidePurchases` setting.

- **Query behavior:**
  - If `hidePurchases = true`, recipient queries for purchase details return only item counts (or nothing at all), not purchaser identities.
  - After the registry is closed (or at any later time the recipient chooses), they can reset the flag to view full details.

This approach preserves flexibility: the giver’s purchase is always recorded, but visibility is controlled by the recipient.

### 7. Generic Types

It is preferable to use SKU code because it keeps the concept simple and focused on tracking gifts rather than managing product catalogs, allows different stores to plug in their own item systems, and ensures stability if product details change over time—the identifier still points to the same item.
If items were represented by names, descriptions, or prices, the registry could become inconsistent or confusing if something changes, also they might not be in the same format as some might not have all information.

## Exercise 2

### Q1

```
state
  a set of Users with
    a username String
    a passwordHash String
```

### Q2

```
actions
  register(username: String, password: String): (user: User)
    requires no existing User has this username
    effects create a new User with this username and a stored passwordHash,
            return the new User

  authenticate(username: String, password: String): (user: User)
    requires a User with this username exists,
             and the stored passwordHash matches the given password
    effects return the matching User
```

### Q3

**Invariant:**
Every `username` in the set of Users must be unique — no two Users can share the same username.

**How it is preserved:**

- The `register` action enforces uniqueness by requiring that no existing User already has the chosen username before creating a new one.
- The `authenticate` action does not modify the state at all, so it cannot break the invariant.

### Q4

```
state
  a set of Users with
    a username String
    a passwordHash String
    a confirmed Flag         // true if email has been confirmed

  a set of PendingConfirmations with
    a username String
    a secretToken String     // token generated during registration

actions
  register(username: String, password: String): (user: User, token: String)
    requires no existing User has this username
    effects create a new User with this username and a stored passwordHash,
            set confirmed = false,
            create a new PendingConfirmation with a secret token,
            return the new User and the token (to be emailed)

  confirm(username: String, token: String)
    requires a PendingConfirmation exists with this username and token
    effects mark the User's confirmed flag as true,
            remove the PendingConfirmation for this username

  authenticate(username: String, password: String): (user: User)
    requires a User with this username exists,
             the stored passwordHash matches the hash of the given password,
             and confirmed = true
    effects return the matching User
```

## Exercise 3

### Concept: PersonalAccessToken

```
concept PersonalAccessToken
**purpose** provide an alternative to passwords for authenticating a user, especially when accessing GitHub from the command line or via API
**principle** a user creates a token (an obscure string) with optional scopes or permissions;
          the token is stored securely by the user;
          when the token is presented along with the username,
          the user is authenticated as themselves,
          with access limited to what the token's scopes allow
**state**
  a set of Tokens with
    an owner User
    a secret String
    a set of scopes Permissions
    an active Flag
**actions**
  createToken(owner: User, scopes: Permissions): (token: Token)
    requires owner exists
    effects create a new Token for this owner with given scopes, active=true,
            return the secret string once

  revokeToken(token: Token)
    requires token exists and is active
    effects set active=false, token cannot be used for authentication

  authenticate(username: String, token: String): (user: User)
    requires a User with this username exists,
             there is an active Token for this user with secret=token
    effects return the User with the permissions granted by that token
```

### Comparison: Passwords vs. Personal Access Tokens

Passwords and Personal Access Tokens both authenticate a user, but they differ in purpose and design. In **PasswordAuthentication**, the state has a single `passwordHash` per user, and the key action is `authenticate`, which checks the hash; it’s mainly for interactive login. In **PersonalAccessToken**, the state includes many `Tokens` with their own scopes and active flags, and the actions include `createToken` and `revokeToken` in addition to `authenticate`. This means tokens can be generated for scripts or APIs, limited in permission, and revoked without affecting the main password.

The GitHub page currently emphasizes “treat tokens like passwords”, which blurs the conceptual difference. This can confuse users into thinking tokens are just another kind of password, rather than a more flexible, revocable, and scoped mechanism.
A clearer explanation would highlight:

- Tokens are not passwords — they are temporary, scoped credentials for automation.

- Passwords are for interactive login; tokens are for API/CLI use.

- Tokens can be revoked individually without changing your main password.

## Exercise 4

### Billable Hours Tracking

```
concept BillableHoursTracking
**purpose** record employee work sessions by project so clients can be billed accurately
**principle** an employee starts a work session by choosing a project and describing the task;
          the system records the start time;
          when the employee ends the session, the system records the end time and duration;
          each completed session is associated with the correct project for billing
**state**
  a set of Projects with
    an identifier ProjectID

  a set of Employees with
    an identifier EmployeeID

  a set of Sessions with
    an employee Employee
    a project Project
    a description String
    a startTime Timestamp
    an endTime Timestamp?   // optional until session is ended
    a duration Number?      // computed once ended
**actions**
  startSession(employee: Employee, project: Project, description: String): (session: Session)
    requires employee and project exist,
             employee has no active session
    effects create a new Session with given employee, project, description,
            set startTime = current time, endTime = null, duration = null,
            return the Session

  endSession(session: Session)
    requires session exists and endTime = null
    effects set endTime = current time,
            compute duration = endTime - startTime

  autoEndSessions(currentTime: Timestamp)
    requires —
    effects for any Session with endTime = null and startTime too far in the past,
            set endTime = currentTime,
            compute duration = endTime - startTime
```

---

### Notes

- **Forgotten sessions:** The `autoEndSessions` action handles cases where an employee forgets to end a session. The policy for “too far in the past” could be defined as the end of the workday, or after a set maximum duration.
- **Multiple active sessions:** The `startSession` action requires that an employee cannot have more than one active session at once, ensuring consistency.
- **Duration calculation:** Keeping `duration` explicit in state avoids recomputation and simplifies billing queries.
- **Extensibility:** The concept could be extended later to allow editing or annotating sessions, but the core design covers the minimal billable-hours workflow.

### Conference Room Booking

```
concept ConferenceRoomBooking
**purpose** allow employees to reserve conference rooms for meetings without conflicts, with shared ownership
**principle** a user creates a reservation for a room by selecting a time and location;
owners of the reservation may later cancel it;
reservations can also list additional owners who share control

**state**
a set of Rooms with
  an identifier RoomID

a set of Users with
  an identifier UserID

a set of Bookings with
  a room Room
  a primaryOwner User
  a set of additionalOwners Users
  a startTime Timestamp
  an endTime Timestamp
  a description String

**actions**
- createBooking(primaryOwner: User, room: Room, startTime: Timestamp, endTime: Timestamp, description: String, additionalOwners: Set<User>): (booking: Booking)
  - *requires* room exists, startTime < endTime, and no existing Booking for that room overlaps [startTime, endTime)
  - *effects* create a new Booking with the given room, primaryOwner, additionalOwners, startTime, endTime, and description; return the Booking

- cancelBooking(actor: User, booking: Booking)
  - *requires* booking exists, actor is the primaryOwner or in additionalOwners
  - *effects* remove the booking from the set of Bookings
```

### Notes

- **Multiple owners:** Each booking has one primary owner and may include additional owners, all of whom can cancel the booking. (This aligns with CSAIL's booking)
- **Canceling:** Owners can fully cancel a booking, with confirmation dialogs handled in the UI.
- **Consistency:** Only owners (primary or additional) can cancel a booking, ensuring clear control without opening access to everyone.

### Time-Based One-Time Password (TOTP)

```
concept TimeBasedOneTimePassword
**purpose** improve account security by requiring a short-lived, time-based token from a user's trusted device in addition to their password
**principle** after registering a TOTP secret with an authentication server and a trusted device (e.g., a phone app),
the device can generate numeric codes that change every fixed time interval (e.g., 30 seconds);
when authenticating, the user provides both their password and the current code;
the server verifies the code against the secret and current time before granting access

**state**
a set of Users with
  a username String
  a passwordHash String
  a totpSecret String (shared secret for generating/verifying codes)
  a totpEnabled Flag

**actions**
- enableTOTP(user: User): (secret: String)
  - *requires* user exists, totpEnabled = false
  - *effects* generate a new secret, assign it to user.totpSecret, set totpEnabled = true, return the secret (to be scanned/stored by the authenticator app)

- disableTOTP(user: User)
  - *requires* user exists, totpEnabled = true
  - *effects* clear totpSecret, set totpEnabled = false

- authenticate(username: String, password: String, token: String): (user: User)
  - *requires* user exists, password matches stored hash,
    and if totpEnabled = true then token matches current TOTP code
  - *effects* return the User if all checks succeed
```

### Notes

- **Purpose and improvement:**
  TOTP adds a second factor — _something you have_ (the phone generating codes) — on top of _something you know_ (the password). This means that even if a password is stolen, an attacker still cannot log in without also having access to the user’s device.

- **Improved security:**

  - Protects against password reuse attacks (if one site leaks your password, an attacker cannot reuse it elsewhere without your device).
  - Protects against brute-force and credential stuffing attacks.
  - Limits damage of intercepted passwords since codes expire quickly.

- **Remaining vulnerabilities:**
  - Phishing sites can trick a user into typing in both their password and a valid current TOTP code.
  - If your phone is stolen and your password is saved on your phone the generated code will also be taken
