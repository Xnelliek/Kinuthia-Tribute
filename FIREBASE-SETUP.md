# Connecting the memorial site to a shared database (about 10 minutes, free)

## Why this is needed

A single HTML file has nowhere to keep information that is shared between people. Until now,
tributes, candles, RSVPs, bus bookings and photos were saved inside each visitor's **own
browser**. So when someone else submitted a tribute on their phone, it sat on their phone and your
admin login (on a different device) could never see it.

Connecting a free Firebase project gives the site one shared place to store things, so:

- tributes sent by anyone show up in **your** admin panel for approval
- photos uploaded by anyone appear in the gallery for everyone
- the bus-slot counter, candles and RSVPs are the same on every device

Until you complete this, the site keeps working exactly as before (local-only), and the admin panel
shows an amber warning saying so.

## Steps

1. Go to https://console.firebase.google.com and sign in with a Google account.
   Click **Add project**, name it (e.g. `kinuthia-memorial`), and you can turn Google Analytics off.
2. **Build → Firestore Database → Create database.** Choose **Production mode** and the region
   closest to your visitors (e.g. `eur3 (Europe)`).
3. **Build → Authentication → Get started → Sign-in method → Email/Password → Enable → Save.**
   Then open the **Users** tab → **Add user** and create the admin login
   (an email and a strong password). Create one for each person who should be an admin.
4. Click the gear icon → **Project settings → General → Your apps → the `</>` (Web) icon.**
   Give it any nickname and register it. Firebase shows a `firebaseConfig` block. Copy the
   four values into the top of the script in `index.html`:

   ```js
   const FIREBASE_CONFIG = {
       apiKey: "AIza...",
       authDomain: "your-project.firebaseapp.com",
       projectId: "your-project",
       appId: "1:123456789:web:abc123"
   };
   ```
   (These values are meant to be public. The rules below are what protect your data.)
5. **Firestore Database → Rules tab.** Delete what is there, paste the rules below, replace the
   two example emails with your real admin email(s) from step 3, and click **Publish**.
6. Upload/publish `index.html` to your website. Open the site, click **Admin**, and log in with
   the **email and password** from step 3. The panel should now show a green
   "Shared cloud database is ON" message.
   Logging in for the first time also loads the two starter tributes and the starter candle into the
   shared database (you can delete or edit them from the admin panel).

If admin login fails on your live site, go to Authentication → Settings → Authorized domains and
add your website's domain.

## Firestore rules (paste into the Rules tab)

> **Updated for the Transport Manager role.** There is now a second, restricted login that
> can see the Bus Slots list only (no RSVPs, no tributes, photos, candles or programs, and no
> edit/delete rights at all). If you already pasted an earlier version of these rules, replace
> it entirely with the one below — nothing else changed except the new `isTransportManager()`
> function and the `registrants` read rule.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isAdmin() {
      return request.auth != null &&
             request.auth.token.email in ['nelvinekavaya@gmail.com', 'essiegons@gmail.com'];
    }
    // Restricted role: can ONLY read Bus Slot registrants. Put the transport manager's
    // real login email here (create their Firebase Auth user the same way as an admin —
    // see the "Adding the Transport Manager login" section below — just don't add their
    // email to isAdmin() above, or they'd get full admin rights instead).
    function isTransportManager() {
      return request.auth != null &&
             request.auth.token.email in ['transport@kinuthiamemorial.org'];
    }
    function shortText(v, max) { return v is string && v.size() > 0 && v.size() <= max; }

    // Published tributes: everyone reads, only admins write
    match /tributes/{id} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // Tributes waiting for approval: anyone may submit, only admins may see or manage
    // (text limit raised to 8000 because tributes can now contain formatting)
    match /pendingTributes/{id} {
      allow create: if request.resource.data.keys().hasOnly(['name','text','createdAt'])
                    && shortText(request.resource.data.name, 100)
                    && shortText(request.resource.data.text, 8000);
      allow read, update, delete: if isAdmin();
    }

    // Virtual candles (short messages: 200 characters)
    match /candles/{id} {
      allow read: if true;
      allow create: if request.resource.data.keys().hasOnly(['name','text','createdAt'])
                    && shortText(request.resource.data.name, 100)
                    && shortText(request.resource.data.text, 200);
      allow update, delete: if isAdmin();
    }

    // Shared gallery photos (small thumbnail + full-size copy)
    match /photos/{id} {
      allow read: if true;
      allow create: if request.resource.data.keys().hasOnly(['name','caption','thumb','createdAt'])
                    && shortText(request.resource.data.name, 100)
                    && request.resource.data.caption is string && request.resource.data.caption.size() <= 300
                    && request.resource.data.thumb is string && request.resource.data.thumb.size() <= 120000;
      allow delete: if isAdmin();
    }
    match /photoFull/{id} {
      allow read: if true;
      allow create: if request.resource.data.keys().hasOnly(['data','createdAt'])
                    && request.resource.data.data is string && request.resource.data.data.size() <= 900000;
      allow delete: if isAdmin();
    }

    // RSVPs and bus registrations contain phone numbers: anyone may submit.
    // Full admins can read/update/delete everything. The Transport Manager can only
    // READ documents where type == 'Bus Slot' — never RSVPs, and never update/delete.
    // 'seatNumber' (the position assigned by the atomic bus-seat reservation) is allowed
    // alongside the older 'seatId' field so both old and newly-created records validate.
    match /registrants/{id} {
      allow create: if request.resource.data.keys().hasOnly(['name','phone','type','details','seatId','seatNumber','createdAt'])
                    && shortText(request.resource.data.name, 120)
                    && shortText(request.resource.data.phone, 40)
                    && shortText(request.resource.data.details, 300)
                    && request.resource.data.type in ['Bus Slot', 'RSVP'];
      allow read: if isAdmin() || (isTransportManager() && resource.data.type == 'Bus Slot');
      allow update, delete: if isAdmin();
    }

    // Legacy "seat taken" markers from the old counting method. The app no longer creates
    // these (bus seats are now reserved via the meta/busSeatCounter transaction below), but
    // the rule is kept so any old records can still be read/cleaned up by an admin.
    match /busSeats/{id} {
      allow read: if true;
      allow create: if request.resource.data.keys().hasOnly(['createdAt']);
      allow delete: if isAdmin();
    }

    // Editable programs + family update text: everyone reads, only admins change it
    match /settings/{id} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // Bus seat counter: the ONE meta document the public may touch, and only to move it
    // forward by exactly 1 at a time, up to the seat cap (44 = BUS_BASE_AVAILABLE in
    // index.html — update the 44 below if you ever change that constant). The app reads and
    // increments this document inside a single Firestore transaction when someone registers,
    // so two people registering at the same instant can never be given the same seat, and the
    // count can never run past capacity.
    match /meta/busSeatCounter {
      allow read: if true;
      allow create: if isAdmin() ||
                    (request.resource.data.keys().hasOnly(['count']) && request.resource.data.count == 1);
      allow update: if isAdmin() ||
                    (request.resource.data.keys().hasOnly(['count'])
                     && request.resource.data.count == resource.data.count + 1
                     && request.resource.data.count <= 44);
    }

    // Everything else under /meta (e.g. the "seed" marker) stays admin-only, as before.
    match /meta/{id} {
      allow read, write: if isAdmin();
    }
  }
}
```

## Adding the Transport Manager login (view-only Bus Slots access)

This gives someone (e.g. the person coordinating buses in Kenya) a login that opens straight
to the Bus Slots list — nothing else — and cannot edit or delete anything, only view.

1. **Firebase → Authentication → Users → Add user.** Use a real email you control access to
   handing out (e.g. `transport@kinuthiamemorial.org`) and set a password. This is exactly like
   creating an admin login, just for a different email.
2. In the Firestore rules above, put that same email inside `isTransportManager()` (replacing
   the placeholder `transport@kinuthiamemorial.org`), then **Publish**.
3. In `index.html`, find the line near the top of the `<script>` block that says:
   ```js
   const TRANSPORT_MANAGER_EMAILS = ['transport@kinuthiamemorial.org'];
   ```
   and put the same email there (must match the rules exactly, including capitalization —
   Firebase lower-cases emails, so use lowercase in both places). Re-publish `index.html`.
4. Give that email + password to the transport manager. They click **Admin** on the site and
   log in with it like anyone else — the dashboard that opens shows only the Bus Slots table
   (numbered 1, 2, 3... in the order people registered) with no edit or delete buttons, and a
   banner confirming they're signed in as Transport Manager. They cannot see RSVPs, tributes,
   photos, candles, or the programs editor, and Firestore itself blocks any attempt to read or
   change that data even if someone tried from outside the website — it isn't just hidden in
   the page.
5. **Also works without Firebase**, for local testing: username `transport`, password
   `buses2026` opens the same restricted view using the browser's local data (no shared
   database).

To remove this access later, delete that Firebase Auth user (or just remove their email from
both `isTransportManager()` in the rules and `TRANSPORT_MANAGER_EMAILS` in `index.html`).

### One-time Firestore index for the Transport Manager view

The Transport Manager's Bus Slots list uses a query (bus slots only, oldest-first) that needs
one composite index the first time it runs. Firestore will not silently fail — the browser
console shows an error with a direct "Create it here" link the first time that query runs; click
it, accept the defaults, and it's ready within a minute or two. You only need to do this once.
If you'd rather create it ahead of time: **Firestore Database → Indexes → Composite → Add
index** → collection `registrants` → fields `type` (Ascending) then `createdAt` (Ascending) →
Query scope: Collection.

### One manual step after you publish these rules (only matters if the site already has bus registrants)

If people have **already** registered for bus seats before you deploy this update, the seat
counter above doesn't exist yet, and the rules only let an **admin** create it with a starting
number other than 1 (a public visitor is only ever allowed to create it at exactly `1`, i.e. as
the very first-ever seat). So: **log in as admin once, right after publishing the new rules and
the new `index.html`**, before telling anyone else the bus registration is open again. Logging
in automatically counts the existing "Bus Slot" registrants and sets the counter correctly. If
you skip this step, the counter will start from 0 on the first *public* booking instead of the
real number already taken, which would let more seats be booked than you have.

If the site has **no** bus registrants yet (a fresh launch), you don't need to do anything —
the first public booking creates the counter correctly on its own.

## Putting a new index.html on the live site (avoid mixed-up pages)

Every time you update the site, the **whole** `index.html` must be replaced, never pasted on top of or
underneath the old one. Mixing old and new makes the page show twice, or shows raw code as text.

- **Netlify drag-and-drop:** drag the *entire site folder* (it must contain `index.html` **and** the
  `Pictures` folder) onto the deploy area. Do not drop `index.html` alone, or the photos disappear.
- **If you edit in a text editor/GitHub:** open `index.html`, press **Ctrl+A** (select everything),
  **Delete**, then paste the new file's full contents. Save.
- **Check it worked:** open the live site, right-click, *View page source*, press **Ctrl+F** and search
  `<!DOCTYPE`. There must be exactly **1** match. The Admin panel also shows a "site version" line.
- The page now shows a red warning bar at the top if it ever detects duplicated content.

## Changing programs later (no code needed)

Log in as admin, open **Admin → Edit Service Programs & Family Update**. You can change each
service's status, title, date, time, venue, map link, livestream link, notes and every line of the
order of service (add, remove, reorder), plus the family update box. Press **Save changes** and the
whole site updates for everyone straight away. This needs the latest rules above (the `settings`
block). **Restore original** puts back the programs that came with the website.

What is still fixed in the page itself: the life timeline, the bus and RSVP wording, and the
"Physical Candle Lighting" line.

## If admin login fails

The login box now tells you exactly what is wrong. The usual causes:

| Message | Fix |
|---|---|
| Wrong email or password / not added under Users | Firebase → **Authentication → Users → Add user**, using the same email you put in the rules. Use **Forgot password?** on the login box if unsure. |
| Email/Password sign-in is switched off | Firebase → **Authentication → Sign-in method → Email/Password → Enable**. |
| Authentication has not been set up | Firebase → **Authentication → Get started**. |
| This website address is not authorised | Firebase → **Authentication → Settings → Authorized domains → Add domain** (`kinuthiandekei.netlify.app`). |
| Signed in, but this email is not on the admin list | The email you signed in with must exactly match one in `isAdmin()` in the rules. |

To check that tributes are arriving: submit a test tribute on the site, then open Firebase →
**Firestore Database → Data**. You should see a `pendingTributes` collection with your test entry.

## Performance: faster loading with multiple people on the site at once

The site now turns on Firestore's built-in **persistent local cache** (IndexedDB, with
multi-tab support) in every visitor's browser. In practice this means:

- Data this browser has already fetched (programs, tributes, candles, seats remaining) is
  shown instantly from local cache on the next visit or page within the same browser, instead
  of waiting on a fresh network round trip every time.
- If two of the site's own pages are open in different tabs on the same device (e.g. the
  homepage and a service page), they share one cache and one connection to Firestore instead
  of doubling the reads.
- On a slow or flaky connection, the page can still show the last-known data immediately while
  it reconnects in the background, rather than showing a blank/loading state.
- This is a per-browser cache — it does not reduce how fast *other* people's devices load, but
  it does reduce how many separate requests hit Firestore overall, which is what actually slows
  things down when many people arrive at once (e.g. everyone visiting right as a service starts).
- No setup is required for this — it's already in the code and enables itself automatically. If
  it can't enable (private browsing, very old browser), the site quietly falls back to normal
  Firestore requests, so nothing breaks either way.

If the site still feels slow at peak times after this, the next thing worth doing (not yet
needed) would be raising the Firestore free-tier limits to a paid pay-as-you-go plan — the free
tier's 50,000 reads/day is very unlikely to be the bottleneck for a memorial site, but it's the
first thing to check in **Firebase Console → Usage** if slowness is reported again.

## Kenya service programme PDF

The "View Full Kenya Programme (PDF)" button on `service-kenya.html` links to
`Kenya_Funeral_Programme.pdf`, which must be uploaded to the site **in the same folder** as
`index.html` and `service-kenya.html` (same rule as the `Pictures` folder — see "Putting a new
index.html on the live site" above). If you replace that PDF with a newer version later, keep
the file name exactly `Kenya_Funeral_Programme.pdf` so the existing link keeps working, or update
the two links inside `service-kenya.html` to match a new file name.

## Good to know

- **Photo uploads publish immediately** (as you asked). Uploaded photos are shrunk in the visitor's
  browser before being saved, so the free plan goes a long way. If someone uploads something
  inappropriate, remove it under Admin → *Manage Shared Gallery Photos*.
- The free Firebase plan is generous (1 GiB storage, 50,000 reads/day), which is far more than a
  memorial site needs.
- The admin password is no longer written in the page's source code once Firebase is connected.
