# Static Analysis Report — `Jiffry.apk`

**Analysis type:** Static only. Nothing was attacked, no API endpoint was called, and no login was performed. **Part 1** analyses the APK (copied read-only, never modified). **Part 2** (added later, at the user's request) analyses the web app the APK points at — limited to passively downloading the *public* static files a browser fetches on first visit (HTML, web manifest, service worker, the JS/CSS bundle) and reading them offline. No authenticated or state-changing request was made.

**Analyst environment:** Linux container, tools: `apktool 2.10.0`, `jadx 1.5.1`, `unzip`, `strings`, `openssl`, `python3`, `curl` (passive GETs of public assets only).

---

## 0. TL;DR — The single most important finding

**This is not a hand-built, custom-coded Android app. It is a [WebAPK](https://chromium.googlesource.com/chromium/src/+/HEAD/chrome/android/webapk/README.md) — a thin, auto-generated wrapper that Google's Chrome "WebAPK minting server" produces when a user installs a Progressive Web App (PWA) to their home screen on Android.**

The entire "application" is just a launcher stub whose only job is to open the website **`https://admin.jiffrybooking.com/`** inside Chrome as a full-screen, standalone web app (no browser UI). **All of the real logic — screens, booking flow, authentication, roles, database, API — lives on that website, not in this APK.** None of it is shipped inside the file.

So when your friend says they "built" it, what almost certainly happened is: they built (or deployed) a web app / PWA at `admin.jiffrybooking.com`, opened it in Chrome on an Android phone, and chose **"Add to Home screen" / "Install app."** Chrome then asked Google's servers to mint this APK automatically. Your friend wrote **zero** of the code that is inside this APK.

| What people usually hope to find in an APK | Present here? |
|---|---|
| Custom Java/Kotlin business logic | ❌ No — only the generic Chromium shell |
| A framework (React Native, Flutter, Cordova, Unity, Xamarin…) | ❌ No |
| Native libraries (`lib/*.so`) | ❌ No (no `lib/` dir, no ABIs) |
| Bundled web assets (`assets/www`) | ❌ No |
| Bundled local database | ❌ No |
| API endpoints / request-response models | ❌ Not in the APK (they're server-side) |
| `google-services.json` / Firebase config | ❌ No |
| Backend base URL | ✅ One: `https://admin.jiffrybooking.com/` (as PWA metadata) |

The rest of this report documents exactly what *is* in the file and what each section of your checklist maps to.

---

## 1. App Identity

Source: decoded `AndroidManifest.xml` (via apktool) and `apktool.yml`.

| Field | Value |
|---|---|
| **Package name** | `org.chromium.webapk.af3c4a2198c8d1058_v2` |
| **App label (launcher name)** | `Jiffry` (`@string/short_name`) |
| **Full name** | `Jiffry Booking System` (`@string/name`) |
| **versionName** | `1` |
| **versionCode** | `1` |
| **minSdkVersion** | `24` (Android 7.0) |
| **targetSdkVersion** | `33` (Android 13) |
| **compileSdkVersion** | `37` (Chromium-internal build) |
| **Start URL / scope** | `https://admin.jiffrybooking.com/` |
| **Web manifest URL** | `https://admin.jiffrybooking.com/manifest.webmanifest` |
| **Runtime host (browser)** | `com.android.chrome` (Chrome) |
| **WebAPK shell version** | `189` |
| **Distributor** | `browser` (minted by Chrome, **not** downloaded from Google Play) |
| **Display mode** | `standalone` (full-screen, no browser chrome) |
| **Orientation** | `portrait` |
| **Theme color** | `#F97316` (orange) |
| **Background color** | `#FFFFFF` (white) |

> The `af3c4a2198c8d1058` hex string in the package name is a hash the minting server derives from the app's identity (the start URL). The `_v2` is the WebAPK "H2O" architecture version. These are generated, not chosen by your friend.

### Permissions (2)
- `android.permission.POST_NOTIFICATIONS` — lets the PWA show Android notifications (web push).
- `android.permission.REORDER_TASKS` — standard WebAPK task-management permission.

There are **no** sensitive permissions (no location, contacts, storage, camera, SMS, etc.). The web app can still request web-level permissions (camera, geolocation, etc.) at runtime *through Chrome*, but those are not declared here.

### `<queries>`
Declares that the app needs to see browsers that can handle `https` VIEW intents and the Trusted-Web-Activity "manage data" action — i.e. it needs to find Chrome. Normal for a WebAPK.

### Components

**Launcher activity:** `org.chromium.webapk.shell_apk.h2o.H2OOpaqueMainActivity` (this is what shows in the app drawer; it displays the splash screen and then hands off to Chrome).

| Component | Type | Exported | Purpose |
|---|---|---|---|
| `h2o.H2OTransparentLauncherActivity` | activity | **true** | Deep-link entry point; routes incoming URLs into Chrome |
| `h2o.H2OOpaqueMainActivity` | activity | **true** | LAUNCHER icon → splash → Chrome |
| `h2o.H2OMainActivity` | activity | true (but `enabled=false`) | Alternate launcher, disabled in this build |
| `h2o.SplashActivity` | activity | false (`singleTask`) | Shows the splash screen |
| `ManageDataLauncherActivity` | activity | false | "Manage site data" shim |
| `NotificationPermissionRequestActivity` | activity | false | Prompts for notification permission |
| `h2o.SplashContentProvider` | provider | **true** | Shares the splash screenshot with Chrome |
| `WebApkServiceFactory` | service | **true** | WebAPK API bound by Chrome (notifications, etc.) |
| `IdentityService` | service | **true** | Lets Chrome verify this APK's identity |

> The exported components look broad but are standard, well-audited Chromium WebAPK plumbing. They don't expose app-specific functionality — they only exist so Chrome can drive the shell. See the security notes in §4.

### Deep links
The app registers as a verified handler for the domain:

- **`https://admin.jiffrybooking.com/`** (any path — `pathPrefix=""`), with `android:autoVerify="true"` (Android App Links / Digital Asset Links verification against the site).
- An **NFC** filter (`android.nfc.action.NDEF_DISCOVERED`) for the same host — tapping an NFC tag pointing at that domain opens the app.

**Effect:** when the device opens any `https://admin.jiffrybooking.com/...` link, this app can handle it and show it as a standalone web app instead of a browser tab.

---

## 2. Technology Stack

### Framework detection — result: **none of the usual ones**
I checked every file signature your checklist lists:

| Framework | Signature looked for | Found? |
|---|---|---|
| React Native | `index.android.bundle`, `libreactnativejni.so` | ❌ |
| Flutter | `libflutter.so`, `libapp.so` | ❌ |
| Cordova / Ionic | `assets/www/` | ❌ |
| Xamarin / .NET | Mono assemblies | ❌ |
| Unity | `libunity.so`, `assets/bin/Data` | ❌ |
| Native Kotlin/Java app | app-specific `.dex` logic | ❌ (only generic shell) |

**Actual stack:** Chromium **WebAPK shell** (Android launcher written by the Chromium project) + the **website** it points at. The real "framework" is whatever runs the web app at `admin.jiffrybooking.com` — but that is **not inspectable from this file** (it's served over the network). Common stacks for a PWA like this would be React/Vue/Angular/Svelte on the front end, but **we cannot and did not determine that statically from the APK**, and we did not fetch the site.

### Native libraries / ABIs
**None.** There is no `lib/` directory. The APK contains no `.so` files and targets no specific CPU ABI — consistent with a pure-Java shell.

### Code contents
- Single `classes.dex` (~39 KB — tiny).
- 40 decompiled classes, all in two groups:
  - `org.chromium.webapk.shell_apk.*` and `...h2o.*` — the Chromium WebAPK shell (launcher activities, splash, services).
  - `defpackage.*` (obfuscated helpers `a`–`z`, `a0`–`c0`) — the shell's internal utilities (R8/ProGuard-renamed).
- `org.chromium.webapk.af3c4a2198c8d1058_v2.R` — resource IDs.

### Major libraries (Retrofit, OkHttp, Volley, Firebase, Room, Realm, SQLite, Glide, …)
**None present.** A grep of all decompiled sources for every library in your list returned nothing. This shell does no HTTP itself and has no ORM/DB/image library — it delegates everything to Chrome.

### Resources
The resource table is almost entirely Chromium's standard WebAPK scaffolding: splash-screen layouts (`webapp_splash_screen_*`), host-browser picker (`host_browser_list_item`, `last_resort_runtime_host_logo`), site-settings icon, notification badge, and ~90 locale `values-*` folders (shipped by the shell, not translations your friend wrote). The app icon is an adaptive icon whose foreground is `maskable_app_icon` (fetched from `https://admin.jiffrybooking.com/pwa-512.png`, murmur2 hash `151830196152456573`) and a white background.

---

## 3. Data Layer (client side)

### Bundled local database
**None.** No `assets/*.db`, no SQLite/Room schema, no Realm file anywhere in the APK. The WebAPK stores nothing itself; any client-side storage (IndexedDB, Cache Storage, localStorage, cookies) is managed **by Chrome**, inside Chrome's profile for that origin — it is not in this file.

### Backend references (found statically, **not** contacted)
Everything the APK knows about the backend is PWA metadata in the manifest:

| Key | Value |
|---|---|
| `startUrl` | `https://admin.jiffrybooking.com/` |
| `scope` | `https://admin.jiffrybooking.com/` |
| `webManifestUrl` | `https://admin.jiffrybooking.com/manifest.webmanifest` |
| `webManifestId` / `appKey` | `https://admin.jiffrybooking.com/` |
| icon URL | `https://admin.jiffrybooking.com/pwa-512.png` |

- **No Firebase** / `google-services.json` (no project id, no DB URL).
- **No `BuildConfig`** with secrets (this isn't a Gradle-built app).
- **No `network_security_config.xml`.**
- `res/values/strings.xml` contains only Chromium shell strings plus the two app names ("Jiffry", "Jiffry Booking System"). No URLs, keys, or secrets.
- The `admin.` subdomain strongly suggests `admin.jiffrybooking.com` is the **staff/admin console** of a booking product (there may be a separate customer-facing site), but that's an inference from the name, not from code.

### API endpoints & request/response models
**Not determinable from the APK.** The app makes no API calls itself; the web app does, at runtime, from inside Chrome. To learn the endpoints you would have to inspect the live site (its JS bundle, or network traffic in DevTools) — which is outside the scope of this static, no-network analysis. **There are no endpoint definitions or data models embedded in this file.**

---

## 4. How It Works

### App flow (what the APK actually does)
1. User taps the **Jiffry** icon → `H2OOpaqueMainActivity` starts and shows the native splash screen (orange `#F97316` theme, white background, the PWA icon).
2. The shell (`H2OTransparentLauncherActivity` / `SplashActivity` logic in `defpackage`) resolves Chrome as the runtime host and launches `https://admin.jiffrybooking.com/` **inside Chrome**, in standalone mode (no address bar / tabs), using the WebAPK/Trusted-Web-Activity mechanism.
3. From that point on, **everything the user sees is the website**, rendered by Chrome. The APK's role is finished.
4. Deep links and NFC tags for the domain route through the same shell into Chrome.

There are **no "main screens"** coded in the APK to map — the screens are web pages served by `admin.jiffrybooking.com`. jadx shows only launcher/splash/service classes.

### Auth flow (design level)
**Not in the APK.** Authentication happens entirely in the web app, in Chrome's browser context for the origin. In practice this means:
- Login is a web form / OAuth flow on the site.
- Session/token handling uses normal web mechanisms — **cookies** (likely `HttpOnly` session cookies) and/or tokens in `localStorage`/IndexedDB — all stored and sent **by Chrome**, not by this APK.
- The APK holds no tokens, no credentials, no auth logic, and no secrets.

### User roles / permissions
A search for `role`, `isAdmin`, `owner`, `employee`, permission enums, etc. across all decompiled code returned **nothing app-specific** — the only "permissions" in the APK are the two Android manifest permissions (§1). Any role model (admin / staff / employee / customer, etc.) is implemented **server-side and in the web front end**, and is not visible in this file. The `admin.` hostname hints at a role-segregated product, but the role model itself cannot be read from the APK.

### Security observations (static, informational)
- **Signing:** the APK is signed by Google's minting server, not your friend:
  - `APP.RSA` → `O=Google, OU=WebAPK`, 3072-bit RSA, minted **2026-08-28**.
  - `BROWSER.RSA` → `C=US, O=Google, OU=Chrome WebAPK, CN=CA`, 3072-bit RSA.
  - The 2026-08-28 "Not Before" date is effectively the date this APK was generated/installed.
- **`allowBackup="false"`** — good; and there's little to back up anyway.
- **Exported components** (`H2OTransparentLauncherActivity`, services, `SplashContentProvider`) are exported by design so Chrome can drive them. They are generic, upstream-audited Chromium code and don't expose Jiffry-specific functionality. No custom exported component leaks app data.
- **The real attack surface is the website**, not this APK. A security review of "Jiffry" would need to target the web app at `admin.jiffrybooking.com` (with authorization) — not this wrapper.

---

## 5. Build / Run for Testing

### Install on an emulator to explore the UI
Because this is a WebAPK, running it is slightly special — it needs Chrome present:

1. Create an AVD with **Google APIs/Play** (so Chrome is available), API 24+ (the app's minSdk is 24). Android Studio → Device Manager → create a Pixel with a Google Play system image.
2. Start the emulator, then install the copied APK:
   ```bash
   adb install /home/user/Test/apk_work/original/Jiffry.apk
   ```
3. Make sure Chrome is installed/updated on the emulator (Play image ships it; otherwise install Chrome).
4. Launch "Jiffry." It will open `https://admin.jiffrybooking.com/` in standalone Chrome. **You'll be interacting with the live website** — so this is the normal app experience, but note it *does* talk to the production server. For pure UI exploration that's usually fine (just browsing/login screens); avoid creating or mutating real data.

> Alternatively, you don't even need the APK: on any Android Chrome, visit `https://admin.jiffrybooking.com/` and use "Add to Home screen" to get the same WebAPK. Or just open the URL in any desktop browser to see the same web app.

### If you later get the source and want a LOCAL test backend
Important: **the source you'd get is the web app's source, not this APK's.** You would run the website locally and point a test build of the PWA at a local backend + local DB. Outline:

1. **Get the web app repo** (the thing that produces `admin.jiffrybooking.com`). That's where screens, auth, roles, API client, and DB migrations live.
2. **Stand up a local backend + local DB** (e.g. a throwaway Postgres/MySQL in Docker) and load the project's migrations/seed data — **never** point at the production database.
3. **Configure the app's API base URL via environment config** (e.g. `.env` / `VITE_*` / `NEXT_PUBLIC_*` — whatever the stack uses) to `http://localhost:<port>` instead of the production host. Run its dev server (`npm run dev` / `pnpm dev` or similar).
4. **To exercise it as an installed app** against your local server, either:
   - just use `http://localhost:...` in desktop Chrome (PWAs install from localhost), or
   - serve the dev build over HTTPS on your LAN and install it on a device/emulator; the WebAPK will be re-minted for *your* test origin, keeping production untouched.
5. Keep all testing against **backends and databases you control**. Do not run load/auth tests against `admin.jiffrybooking.com`.

> Rebuilding *this APK* from source isn't really a thing — WebAPKs are generated by Google on install. You reproduce the app by installing the PWA; you develop it by working on the website.

---

## Appendix — Evidence & working files

- **Original (read-only):** `/home/user/Test/apk_work/original/Jiffry.apk`
  - SHA-256 `39963d43de1c8125aa7f343554a9debb75284d3d79e8397fbd540594c02054aa`
  - MD5 `529b80ea6993d0752929463f7acf042a`
  - Size 215,579 bytes
- **Raw unzip:** `/home/user/Test/apk_work/extracted/raw/`
- **apktool decode (manifest, resources, smali):** `/home/user/Test/apk_work/extracted/apktool/`
- **jadx decompile (Java):** `/home/user/Test/apk_work/extracted/jadx/`
- **Tools:** `/home/user/Test/apk_work/tools/` (apktool 2.10.0, jadx 1.5.1)

### Full APK file inventory
```
AndroidManifest.xml
classes.dex                (single dex, ~39 KB — generic Chromium WebAPK shell)
resources.arsc
META-INF/  APP.RSA APP.SF BROWSER.RSA BROWSER.SF MANIFEST.MF
res/       drawable-* (site-settings icon, notification badge, splash)
           layout/ layout-v31/ (splash + host-browser-picker layouts)
           mipmap-* (app/launcher/maskable icons)
           values + ~90 values-<locale>/ (Chromium shell strings)
           xml/shortcuts.xml (empty)
(no lib/, no assets/, no bundled DB, no google-services.json)
```

**Bottom line:** `Jiffry.apk` is a Google-generated PWA launcher (WebAPK) for `https://admin.jiffrybooking.com/`. To understand "how it's built," the subject of study is that **website/PWA**, not this APK — the APK is just the doorway.

---
---

# PART 2 — The Web App (`admin.jiffrybooking.com`)

> Added at the user's request. Method: passive download of the **public** static assets only (the files any browser fetches on first visit — `index.html`, `manifest.webmanifest`, `sw.js`, and the Vite JS/CSS bundle), read offline. **No login, no API calls, no probing.** The JS bundle is minified and shipped **without source maps**, so symbol names below are recovered from minified code; endpoint strings and URLs are read verbatim.

## 2.0 TL;DR
The APK points at the **admin console** of a **bus booking & fleet-management system**. The admin console is a **React single-page app, built with Vite, shipped as a PWA**, served as static files by **nginx**. It talks to a separate backend API at **`https://api.jiffrybooking.com/api`** and opens a **Socket.IO** realtime connection to the same origin. Authentication is a **JWT bearer token kept in `localStorage`**. There are four user roles: **superadmin, admin, agent, conductor**. No secrets or API keys are embedded in the bundle (good hygiene).

## 2.1 App identity (web)
| Field | Value |
|---|---|
| Admin frontend host | `https://admin.jiffrybooking.com/` (nginx/1.31.2, static files) |
| Backend API | `https://api.jiffrybooking.com/api` (axios `baseURL`) |
| Realtime | Socket.IO / Engine.IO to `https://api.jiffrybooking.com` (code default fallback: `https://busbooking.vynoqpos.xyz.lk`) |
| Title / name | "Jiffry Booking System" / short name "Jiffry" |
| Description (meta) | "Bus booking and fleet management system" |
| Theme color | `#F97316` (matches the APK) |
| PWA | `vite-plugin-pwa` + Workbox service worker (`sw.js`), precaches the app shell for offline launch |

> `busbooking.vynoqpos.xyz.lk` appears only as a hard-coded default for the socket URL when the API base can't be derived; `api.jiffrybooking.com` is the real backend. The `.xyz.lk` host looks like an earlier/dev origin — noted, not contacted.

## 2.2 Technology stack (web)
- **Frontend framework:** React (SPA, client-side routed). Build tool: **Vite** (hashed `assets/index-*.js` / `*.css`, ES-module output).
- **PWA layer:** `vite-plugin-pwa` with a **Workbox** service worker that precaches the shell (`index.html`, icons, JS/CSS chunks) — this is what lets the WebAPK launch offline.
- **HTTP:** **axios** (with request/response interceptors) plus some native `fetch`.
- **Realtime:** **socket.io-client** (Engine.IO) — live updates (likely seat availability / booking status).
- **PDF / export:** **jsPDF** + **html2canvas** — client-side generation of tickets/receipts/reports.
- **Sanitisation:** **DOMPurify** (XSS-safe HTML rendering).
- **UI:** **lucide-react** (icons), **sonner** (toast notifications). No heavyweight UI kit (MUI/antd) detected; styling is via the bundled CSS (likely Tailwind-style, not confirmed).
- No state-manager library (Redux/Zustand/react-query) detected — state appears to be React-native (context/hooks).

## 2.3 Data layer / backend references
- **Single API base:** `https://api.jiffrybooking.com/api`. All endpoints below are relative to it.
- **Client storage** (browser `localStorage`, managed by the web app — nothing in the APK):
  | Key | Purpose |
  |---|---|
  | `jiffry_auth_token` | the JWT bearer token |
  | `jiffry_remembered_user` | remembered username ("remember me") |
  | `archivedBusIds` | UI state — JSON list of archived bus IDs |
- **No bundled DB**, and **no embedded secrets**: scans for Google Maps (`AIza…`), Stripe (`pk_/sk_…`), AWS (`AKIA…`) and generic `apikey/secret` literals all returned **zero**. The server-side database is not observable from the client (as expected).

### API endpoints referenced in the client (paths seen in the bundle)
Grouped by feature. These are the paths the admin console calls on `…/api`; exact verbs/params aren't shown here because that would require reading network traffic, which wasn't done.

- **Auth:** `/auth/login`, `/auth/logout`, `/auth/me`
- **Bookings:** `/bookings`, `/bookings/cancelled`, `/bookings/lookup-phone`, `/bookings/bulk-sms`, `/bookings/announcement-sms`
- **Fleet:** `/buses`, `/buses/archived`, `/routes`, `/trip-schedules`
- **People:** `/conductors`, `/agents`, `/users/admins`, `/users/conductors`
- **Agent portal:** `/agent-portal/my-buses`, `/agent-portal/my-quota`, `/agent-portal/quotas/range`, `/agent-portal/users`, `/agent-payments`
- **Dashboard & reports:** `/dashboard`, `/dashboard/recent`, `/summary`, `/summary/by-agent`, `/summary/by-bus`
- **Settings:** `/settings`, `/settings/company`, `/settings/pickup-points`, `/settings/templates`
- **Realtime:** `/socket.io`, `/engine.io`, `/api/socket.io`

> The `bulk-sms` / `announcement-sms` / `settings/templates` endpoints indicate an **SMS gateway integration** for booking confirmations / announcements (templated messages). The provider isn't named client-side.

## 2.4 How it works
### Screens / flows (SPA routes)
The console is a client-routed SPA. Its main areas (from route/path strings): **Login → Dashboard** (with "recent" activity), **Bookings** (active, cancelled, phone lookup, bulk/announcement SMS), **Buses** (incl. archived), **Routes**, **Trip schedules**, **Conductors**, **Agents**, an **Agent portal** (an agent's own buses/quota/users/payments), **Reports/Summaries** (by agent, by bus), and **Settings** (company profile, pickup points, SMS templates).

### Auth flow (design level — confirmed from the bundle)
1. User submits credentials → **`POST /auth/login`**; the response contains a **token** which is saved to `localStorage["jiffry_auth_token"]`.
2. An **axios request interceptor** reads that token and attaches **`Authorization: Bearer <token>`** to every API request.
3. **`GET /auth/me`** loads the current user (identity + role) to drive the UI.
4. An **axios response interceptor** watches for **HTTP 401** and, on an unauthorised response (outside the login call), clears the token and forces re-login.
5. **`/auth/logout`** ends the session; "remember me" keeps the username (not the password) in `jiffry_remembered_user`.

This is a standard **JWT-in-localStorage** design. (Worth noting for your friend, as a design observation only: tokens in `localStorage` are readable by any script on the page, so the app's XSS posture matters — DOMPurify's presence suggests they're aware of this.)

### Role model (confirmed strings)
Four roles drive what the UI shows: **`superadmin`**, **`admin`**, **`agent`**, **`conductor`** (a generic `user` also appears). The **agent-portal** endpoints are the agent-scoped slice (an agent sees only *their* buses, quota, sub-users and payments), while admin/superadmin get the full fleet, people-management and reporting screens. Conductors appear to be managed entities (lists under `/conductors`, `/users/conductors`) and likely have a limited operational view. **Important:** client-side roles only decide what's *rendered*; real enforcement must be server-side on `api.jiffrybooking.com` — which can't be verified from the client and wasn't tested.

## 2.5 Running / testing against a backend you control
Because the real app is the website, "building a debug variant" means running the **web app** locally, not rebuilding the APK:
1. Get the **frontend repo** (this React/Vite project) and the **backend repo** (whatever serves `api.jiffrybooking.com`) from your friend.
2. The API base is injected at build time (axios `baseURL`). Point a local build at your own backend via the project's env config (Vite `import.meta.env` / a `.env` with something like `VITE_API_URL=http://localhost:PORT/api`) — **never** at `api.jiffrybooking.com`.
3. Stand up the backend + a **throwaway local database** (load its migrations/seeds); keep all booking/SMS tests on local, disposable data.
4. `npm install && npm run dev` to get the console at `http://localhost:…`; PWAs install from localhost if you want the installed-app experience.

## 2.6 What was NOT done (scope guardrails)
No login attempt, no call to any `/api/*` endpoint, no Socket.IO connection, no path/parameter fuzzing, no access to `api.jiffrybooking.com` or `busbooking.vynoqpos.xyz.lk` at all. Only public static files from `admin.jiffrybooking.com` were downloaded and read offline. Going further (actual request/response shapes, auth hardening, access-control testing) would mean interacting with a live server and should only be done **with the owner's explicit authorisation** and against a backend you control.

## 2.7 Web evidence files
Saved under `/home/user/Test/apk_work/web/` (gitignored, like the rest of `apk_work/`): `index.html`, `manifest.webmanifest`, `sw.js`, `registerSW.js`, `assets/index-CQO7BLkR.js` (1.39 MB app bundle), `assets/index-C7Tem6dm.css`, `headers.txt`.
