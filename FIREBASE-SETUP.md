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

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isAdmin() {
      return request.auth != null &&
             request.auth.token.email in ['nelvinekavaya@gmail.com', 'essiegons@gmail.com'];
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

    // RSVPs and bus registrations contain phone numbers: anyone may submit, ONLY admins can read
    match /registrants/{id} {
      allow create: if request.resource.data.keys().hasOnly(['name','phone','type','details','seatId','createdAt'])
                    && shortText(request.resource.data.name, 120)
                    && shortText(request.resource.data.phone, 40)
                    && shortText(request.resource.data.details, 300);
      allow read, update, delete: if isAdmin();
    }

    // Anonymous "seat taken" markers so everyone sees how many bus slots remain
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

    // Internal bookkeeping
    match /meta/{id} {
      allow read, write: if isAdmin();
    }
  }
}
```

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

## Good to know

- **Photo uploads publish immediately** (as you asked). Uploaded photos are shrunk in the visitor's
  browser before being saved, so the free plan goes a long way. If someone uploads something
  inappropriate, remove it under Admin → *Manage Shared Gallery Photos*.
- The free Firebase plan is generous (1 GiB storage, 50,000 reads/day), which is far more than a
  memorial site needs.
- The admin password is no longer written in the page's source code once Firebase is connected.
