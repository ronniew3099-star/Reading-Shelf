# Reading Shelf — Code Fixes Changelog

**App:** The Reading Shelf — https://reading-shelf.ronniew3099.workers.dev/
**Repo:** https://github.com/ronniew3099-star/Reading-Shelf, branch `main`
**Source fixed:** `index.html` from the repo clone (1,209 lines, 138,750 bytes)
**Fixed file:** `index.html` in this folder (1,431 lines, 150,402 bytes; 110 lines removed, 332 added)
**Date:** 2026-11-22
**Plan:** `~/workspace/your_files/Code Fixes - Bible Challenge and Reading Shelf.md`, Part 2 (Reading Shelf bugs 1–12), the Reading Shelf signed-in findings (1, 4, 5, 6), and the cross-cutting notes that apply to the Shelf.

**Verification done:** `node --check` on the extracted inline script (syntax OK); every changed function re-read and grep-verified in context; zero native `confirm()` / `prompt()` / `alert()` calls remain in executable code (the only matches are in one code comment explaining the removal); the injected junk after `</script>` is gone; the file ends cleanly with `</html>`.
**Verification NOT done (the app was not run):** everything under "Post-deploy checks" below is marked **UNTESTED** and needs a live check after deploy.

**Scope:** `index.html` only. The background jpg was not touched. No PWA / manifest / service-worker pieces were added (out of scope — the Shelf has no install code). The user's real Shelf data was not touched; this is a code-only fix. The Supabase URL and publishable key are unchanged (still hardcoded, as before — out of scope).

---

## Root cause found while fixing — read this first

**Bug 1 (books vanish after reload) had a deeper cause than the render order.** Line 1192 of the original file listed `setCategoryFilter` in the `Object.assign(window, {...})` call, but **no function named `setCategoryFilter` exists anywhere in the file**. Evaluating that object literal throws a `ReferenceError`, which stops the whole script before `init()` (line 1205) ever runs. So on a fresh page load the app never booted at all: no session restore, no cloud load, no `render()` — Home and Library showed their static zeros, while Insights and Knowledge only worked because their tab buttons call their render functions directly from the synchronously loaded local data.

Fix 1 therefore has two parts: (a) `setCategoryFilter` was removed from the `Object.assign` list (fixed file line 1401), so the script can actually run; (b) the boot order was reworked as planned (render local data first, then restore the session). **Either part alone would not have fixed the bug.**

## Deployed file vs repo — honest status

A read-only fetch of the live site returns rendered text only, so the deployed file's script could not be inspected. **It could NOT be confirmed whether the deployed file contains `getSession` / `init` or a Library search box.** No search box is visible in the rendered text, but the Library view is hidden by default, so that proves nothing either way. Note that the signed-in test on Nov 21 showed the live app rendering real data in Insights/Knowledge, which suggests the deployed file's boot does run — the deployed file may therefore differ from this repo version. If the deployed file has the same `setCategoryFilter` line, it has the same boot failure and this fix repairs it; if it does not, the boot failure is repo-only and the render-order part of fix 1 is still the right change. **After deploy, compare the deployed file against this one; a redeploy alone may fix the biggest bug if the deployed file is older than the repo.**

---

## Fix 1 — Books vanish after reload (biggest bug)

**Files/lines (fixed file):** `init()` line 1402; `loadCloudData()` line 603; `setView()` line 708; `Object.assign(window, …)` line 1401; `setAuthUI()` footer toggle (see fix 10).

- **Boot failure fixed:** removed the undefined `setCategoryFilter` from the `Object.assign(window, …)` list (see "Root cause" above). The script now runs through to `init()`.
- **Render local data first:** `init()` now calls `render(); renderReadingPlans(); renderKnowledge();` and shows "Loading your shelf…" **before** awaiting `getSession()`, so the page never sits on hardcoded zeros. Only then does it restore the session and, if signed in, load cloud data.
- **Loading state instead of zeros:** the "Loading your shelf…" message replaces the previous silent wait; `loadCloudData()` always renders in its `finally` block, even when one or more cloud requests fail.
- **No signed-out flash:** the `onAuthStateChange` handler now ignores the `INITIAL_SESSION` event (init already set the UI from `getSession()`), and only calls `setAuthUI()` when the user id actually changes. A `lastAuthUserId` guard (declared with the dialog system, line 477) tracks this.
- **Every `setView` re-renders its view:** Home now calls `render()` + `renderReadingPlans()`; Library now calls `render()`; Insights, Knowledge, and Plans keep their existing renders. Previously Home and Library rendered nothing on tab switch.
- **Parallel cloud queries:** `loadCloudData()` now fires the five library queries (books, settings, sessions, knowledge, quotes) together with `Promise.allSettled` instead of awaiting ~7 sequential queries, and starts the plan load at the same time.
- **Empty/failed cloud responses no longer overwrite local data:** books already had this guard; reading sessions, knowledge notes, and quotes now get the same rule — a table is only replaced when the cloud returns a non-empty array. An empty result (or a failed request) keeps the local copy and adds a plain-language warning to the sync message (e.g. "Supabase returned no reading history. Your locally saved history was preserved."). One request failing no longer blocks the others.
- **Started dates survive cloud reloads:** the books mapping in `loadCloudData()` now merges each book's locally stored `startedAt` back in (and reads `started_at` from the cloud row if that column ever exists), so the new date fields from fix 8 are not wiped by a sync.

## Fix 2 — Native dialogs replaced with in-app dialogs (Remove / Create plan / Export)

**Files/lines (fixed file):** dialog markup line 403–404 (`#confirmModal`, `#undoToast`); dialog system lines 473–552 (`showConfirmDialog`, `askConfirm`, `askInput`, `showNotice`, `fmtDate`, `showUndoToast`, `undoRemoveBook`); all 18 call sites converted (see below).

A single reusable in-app dialog (`#confirmModal`) now handles confirms, text inputs, and checkbox/radio choice lists, with Cancel, backdrop-click to cancel, and focus on the input. `showNotice()` is the in-app replacement for `alert()`. Zero native `confirm()` / `prompt()` / `alert()` calls remain in executable code (the only textual matches are in the comment on line 473 explaining the removal). Native dialogs are auto-dismissed in some browsers, which is why these actions silently did nothing.

- `removeBook()` (line 1018): asks "Remove this book?" in-app; after removal shows an **Undo toast** for 8 seconds ("<title>" removed. [Undo]); Undo restores the book at its original position and re-saves. If the cloud delete fails, the book is put back in place and an error is shown instead of silently diverging from the cloud.
- `deleteKnowledgeQuote()` (line 866): in-app confirm.
- `createEmptyReadingPlan()` (line 1316) and `createPlanFromUnread()` (line 1323): the plan-name `prompt()` is now an in-app text dialog.
- `addBooksToPlan()` (line 1336): the comma-separated-numbers `prompt()` is now a checkbox list of the candidate books (title shown, "page count missing" noted); selection is by tapping, not typing numbers.
- `configurePlanBook()` (line 1357): the schedule-type `prompt()` is now a radio choice (Pages / Chapters / Manual); chapter count, current chapter, and chapters-per-day are in-app number inputs with the same validation messages as before; the manual assignment is an in-app text input.
- `advancePlanChapters()` (line 1385), `removePlanBook()` (line 1391), `markPlanCompleted()` (line 1397), `deleteReadingPlan()` (line 1398): in-app confirms.
- `saveBook()` (line 992) and `exportExcel()` (line 1046): `alert()` calls replaced with `showNotice()` (see fixes 3 and 4).
- New exports added to `Object.assign(window, …)` (line 1401): `forgotPassword`, `setLibrarySearch`, `undoRemoveBook`.

## Fix 3 — Auto-complete a book in `saveBook()`

**File/line (fixed file):** `saveBook()` line 992.

When the saved current page reaches the total page count, the book's status is now set to `completed` (the same rule the quick-progress functions already had) and `completedAt` is set from the Finished date field if given, otherwise the previous value, otherwise now. Previously only the quick-progress buttons did this; saving through the Add/Edit form left the book stuck on "reading" at 100%.

## Fix 4 — Export Excel works on local data without sign-in

**Files/lines (fixed file):** `loadXlsxLibrary()` line 1034; `exportExcel()` line 1046.

`exportExcel()` is now `async` and no longer depends on the CDN script having loaded at page load. If `window.XLSX` is missing it lazy-loads the xlsx library from the CDN (jsDelivr, xlsx 0.18.5) with a "Loading the Excel export library…" message, then builds the spreadsheet from local data exactly as before. If the library still cannot load, an in-app error is shown ("The Excel export library could not load. Check your internet connection and try Export again.") instead of a silent `alert()`. The repo stays a single file — nothing was bundled. Exporting an empty shelf shows an in-app notice instead of an alert.

## Fix 5 — Completed-book cards show status, pages, and progress

**Files/lines (fixed file):** `bookCardHTML()` line 912 (new); `statusLabel()` line 905; `render()` card rendering line 959.

Every Library card now always shows a status badge ("Want to read" / "Currently reading" / "Completed"), the current/total pages, and the progress bar (100% when completed). The quick-progress buttons still only appear for books currently being read. The action label for completed books is now **"Read again"** (it was "Start reading"); "Start reading" remains for want-to-read books and "Update" for books being read. New CSS: `.status-badge` rules added just before `</style>`.

## Fix 6 — Daily goal: Save button, today's progress, uncapped weekly %

**Files/lines (fixed file):** Home goal panel line 282; Settings goal panel line ~398; `saveGoal()` line 696; `renderGoalProgress()` line 919; `renderInsights()` line 713.

- Both goal inputs (Home and Settings) lost their save-on-change behaviour and now have a **Save goal** button. Saving shows a confirmation toast ("Daily goal saved to your cloud shelf." / "Daily goal saved on this device."). The Settings input still writes through the same `saveGoal()` as Home.
- Home now shows today's progress under the goal input (`#goalProgress`): "N / M pages today", or "Goal met — N pages today" in green once the goal is reached. It is refreshed by `render()` and by `saveGoal()`.
- The weekly percentage in `renderInsights()` is no longer capped with `Math.min(100, …)`: 200/140 now shows **"143% of your weekly target · Goal exceeded"** instead of "100%". The bar itself is still clamped at 100% width for layout. (The `Math.min(100, …)` in `renderAdvancedAnalytics()` and the one in `pct()` were left alone — the first is out of scope per the plan, the second is correct behaviour.)

## Fix 7 — Page clamp says so

**Files/lines (fixed file):** `quickSaveProgress()` line 967; `quickAddProgress()` line 980; `saveBook()` line 992.

In all three progress functions, when a page number is clamped down to the total, a message now says so: "Page cannot exceed N, so it was set to N." The quick functions also keep their existing auto-complete behaviour. The lower-page confirmation in `quickSaveProgress()` is now the in-app dialog ("Save a lower page?") instead of a native confirm.

## Fix 8 — Dates on book cards and in the Add/Edit form

**Files/lines (fixed file):** form date inputs (STARTED / FINISHED) in the book modal; `bookDatesHTML()` line 906; `bookCardHTML()` line 912; `saveBook()` line 992; `openAdd()` / `editBook()` date handling; `loadCloudData()` books mapping line 603.

- Book cards now show **"Finished [date]"** for completed books (and **"Started [date]"** whenever one is recorded), via the shared `fmtDate()` helper.
- The Add/Edit form has optional **Started** and **Finished** date inputs. They are stored on the book (`startedAt`, `completedAt`) and pre-filled when editing.
- **Note:** `startedAt` is local-only. The cloud `books` table has no known `started_at` column (the persist payload carries `completed_at` only), so a started date is kept in this browser's local data and merged back in after cloud loads (fix 1). It is not uploaded. If a `started_at` column is added to Supabase later, the mapping already reads it.

## Fix 9 — Library search

**Files/lines (fixed file):** search input in the Library view (line 289); `setLibrarySearch()` line 925; `render()` shown-filter line ~955.

A permanent search input sits under the category filter: "Search your books by title or author". Typing filters the displayed books by title or author (case-insensitive); clearing it restores the full list. The empty state says "No books match your search. Try another title or author." when a search is active.

## Fix 10 — Duplicated copy

**Files/lines (fixed file):** footer line 401; `setAuthUI()` footer toggle; `render()` category count line 957.

- Only one count element remains: `bookCount` ("N of M books displayed"). `categoryResultCount` now shows only the filter summary ("2 categories selected · Reading", or "Showing all books" when nothing is filtered) instead of repeating the same count a second time.
- The footer ("Sign in to sync your reading data across devices…") is now hidden whenever signed in (`setAuthUI()` toggles `#syncFooter`); it still shows while signed out.

## Fix 11 — Layout

**Files/lines (fixed file):** CSS lines 12, 33, 60 and the new rules before `</style>`; "Pick up where you left off" block in `render()` lines 935–945.

- Category count is now inline and normal-sized (`.category-tile .category-number` uses `font:inherit` instead of 23px Georgia).
- `.cover-placeholder` now uses `overflow-wrap:break-word` with a 3-line clamp and ellipsis instead of `overflow-wrap:anywhere` (which broke words mid-word).
- `.dialog-actions` is now `position:sticky; bottom:-23px` with a paper background, so the Save/Cancel buttons stay visible while scrolling a tall Add/Edit form.
- The "▣" OneNote glyph is replaced by an inline SVG icon (a rounded square with a check mark) in both the nav button and the OneNote coming-soon card.
- **"Pick up where you left off"** now falls back when nothing is marked "reading": it shows the most recently finished book labelled **"Last finished"**, otherwise the first book on the shelf labelled **"Continue"**, instead of the empty "No book selected yet" message. The section heading (`#focusHint`, already in the HTML) updates to match ("Your current focus" / "Last finished" / "Continue"). The fallback card offers "Read again" for a finished book and "Start reading" otherwise.

## Fix 12 — Two daily goals: one source of truth

**Files/lines (fixed file):** plan intro copy (Reading Plans view); plan settings label in `renderReadingPlans()`.

The plan field is relabelled **"Plan pace (pages/day)"** and the plans intro now says "set a plan pace" instead of "set a daily page goal". The underlying `daily_page_goal` field name in the plan data and cloud payload is unchanged (renaming a stored field would risk existing plans); only the label the user sees changed, so it is no longer presented as a second daily goal. The global daily goal (Home / Settings) remains the one daily goal.

## Signed-in findings

- **Finding 1 (session loss after reload):** fixed as part of fix 1 above (boot failure + render-first + empty-response guards).
- **Finding 4 (footer nag while signed in):** fixed as part of fix 10 above.
- **Finding 5 (two daily goals):** fixed as part of fix 12 above.
- **Finding 6 (completed books show "Start reading"):** fixed as part of fix 5 above ("Read again").

## Fix 16 — Two functions were missing, so the app died before sign-in (root cause)

**File/line (fixed file):** `localDateString()` and `recordReadingSession()` restored near the top of the script.

**Cause:** the earlier rewrite accidentally dropped two function definitions that the rest of the file still calls. `localDateString()` (formats a date as local YYYY-MM-DD) is called while the app loads its reading-plan data (`selectedPlanDay=localDateString()`), so the script threw `Uncaught ReferenceError: localDateString is not defined` and died before the sign-in form was wired up — tapping Sign in reloaded the page with no message. That is why the sign-in library guard (Fix 13), the service-worker change (Fix 13) and the safe saved-data loading (Fix 15) did not fix it: the script was dying on a missing function, not on a failed load or a bad saved value. `recordReadingSession()` (records a reading session when a book's page count moves) was also missing; it only runs when saving a book, so it did not block sign-in, but saving progress would have thrown. Both were restored verbatim from the original file. A check now confirms no function defined in the original is missing from the fixed file.

**How it was found:** Ronnie pasted a browser Console error naming `localDateString is not defined` at the plan-data line.

## Fix 15 — A corrupt saved value could kill the app before sign-in

**File/line (fixed file):** the saved-data loading at the top of the script (`safeParseArray` / `safeParseObject`).

**Cause:** the app loaded its saved books, sessions, plans and notes with bare `JSON.parse(localStorage.getItem(...))` calls at the top of the script. If any stored value on the device was corrupt or in an unexpected shape, the parse threw and the whole script died before the sign-in form was wired up — so tapping Sign in reloaded the page with no message, exactly the symptom still seen after the sign-in library guard (Fix 13), which only covers the library failing to load.

**Fix:** all seven stored values now load through `safeParseArray` / `safeParseObject`, which return empty data (and, for a bad read, ignore the stored value) instead of throwing. The app now starts on empty local data and can sign in and re-sync from the cloud, rather than dying on one bad saved value.

## Fix 14 — Undo toast was always visible

**File/line (fixed file):** CSS for `.undo-toast` (line 245).

**Cause:** the toast element carries the `hidden` attribute and should only appear after a book is removed, but its CSS sets `display:flex`, and an author's `display` beats the browser's built-in `display:none` for `hidden`. So the empty toast (with its Undo button) sat at the bottom of the screen on every load.

**Fix:** added `.undo-toast[hidden]{display:none}`. The toast now only appears after a removal, as intended. The other views and the dialogs were checked: `.app-view` sets no `display`, and `.modal` defaults to `display:none` (shown by the `.show` class), so `hidden`/`.show` work correctly there.

## Fix 13 — Sign-in failed silently (page reloaded) when the sign-in library did not load

**Files/lines (fixed file):** `window.supabase.createClient` guarded (top of the script); `const $` moved before it; sign-in submit guard; `init()` guards; `sw.js` fetch handler.

**Cause:** the sign-in library loads from a CDN, and the app created its database client with a bare `window.supabase.createClient(...)` at the top of the script. If that library failed to load, the call threw and the whole script died before it defined `$`, attached the sign-in form's submit listener, or ran `init()`. The symptoms matched exactly: tapping Sign in did a native form submit (page reloaded, fields blank), tapping a tab hid the Home view and then threw (blank page, Home still highlighted), and the page showed only its static zeros. The service worker could also intercept the CDN script request, so its fetch handler now ignores every cross-origin request and only handles same-origin ones.

**Fix:** `$` is defined before the client creation; the client creation is wrapped in try/catch (`db` stays null if the library is missing); the sign-in submit handler shows "Sign-in service could not load. Check your connection and reload the page." instead of reloading when `db` is null; `init()` throws a clear "the sign-in service did not load" error into its existing catch (instead of a null crash) and only subscribes to auth changes when `db` exists. The app now stays usable on local data and says what is wrong instead of dying silently.

## App icon, manifest, and service worker

**Files:** `icon-192.png` (192x192, 26,402 bytes) and `icon-512.png` (512x512, 105,698 bytes) in this folder; `manifest.json` (new); `sw.js` (new); `index.html` head (manifest + apple-touch-icon links) and service-worker registration at the end of the script.

- **Icon:** a wooden bookshelf with colourful book spines on three shelves, on a dark warm brown background, full-bleed edge to edge (built for a dark-mode phone — the generator's white corners were filled with the icon's own dark edge colour). Reads clearly at 192px. The two PNGs are binary, so they cannot go through the connector's snapshot push — upload them to the repo root the same way the Bible icons were uploaded, **before** the text files go live (the service worker caches them at install, and a missing icon file makes the whole install fail).
- **Manifest:** name "The Reading Shelf", short name "Reading Shelf", display standalone, background colour `#1e0e03` (the icon's dark brown), theme colour `#294b3b` (the app's green, matching the existing theme-color meta).
- **Service worker:** `reading-shelf-v1`, same pattern as the Bible app's — network-first navigations (so a new deploy shows up), cache-first same-origin assets. The shell cached at install is `./`, `./index.html`, `./manifest.json`, `./icon-192.png`, `./icon-512.png`, and `./reading-shelf-modern-library-background.jpg`.
- **Registration:** `index.html` registers `./sw.js` on window load, with failures swallowed so a failed registration can never break the app.

## Cross-cutting

- **Shared-account explainer + forgot password:** the sign-in card now says "The Reading Shelf and Ronnie's Bible Challenge use the same Supabase account, so the same email and password work in both apps. Your books and your Bible reading are stored separately." A **"Forgot password?"** link sits under it and calls the new `forgotPassword()` (fixed file line 530), which sends a Supabase reset email to the address typed in the sign-in form and reports success or failure in-app.
- **Injected junk removed:** the Canopy `ntsp_*` hidden inputs and `jsQuilting` script tags that sat after the closing `</script>` tag (original line 1208) are gone. The file now ends cleanly: `</script>` then `</body>` then `</html>`.

---

## Deploy steps

1. Upload **`icon-192.png` and `icon-512.png`** from this folder to the repo root first (binary — the snapshot push cannot carry them). Wait until the repo listing shows both files.
2. Push **`index.html`, `manifest.json`, `sw.js`, and this CHANGELOG.md`** from this folder to the `Reading-Shelf` repo on `main` (same snapshot-push route used for the Bible Challenge fix; the background jpg is unchanged and stays as-is). Icons first, text files second — if the text files go live before the icons are in the repo, the service worker install fails on the missing icon files.
3. The Cloudflare Worker auto-deploys from the repo — no separate deploy step. Allow a few minutes, then open the live site in a fresh tab.
3. Download a fresh backup of nothing — the Shelf has no backup export; the fix touches code only, and the local data is preserved by the new empty-response guards. (If you want a safety net, the books are also in Supabase.)
4. Do **not** sign in as the user to test; run the post-deploy checks below in the user's own signed-in session.

## Post-deploy checks (UNTESTED — the app was not run)

Do these in the live app, signed in as the user:

1. **Reload while signed out:** your books are still visible on Home and in My Library (local data renders immediately, no zeros).
2. **Sign in, then reload:** books, counts, and the current book are still there; Home and Library show real numbers, not zeros.
3. **Reload again:** no flash of the signed-out sign-in form.
4. **Remove a book:** an in-app dialog appears (not a browser popup). Confirm → the book disappears and an Undo toast appears. Tap **Undo** → the book comes back.
5. **Create empty plan:** an in-app name dialog appears; the plan is created. **Create plan from unread** works the same way.
6. **Add books to a plan:** a checkbox list of books appears; pick some and confirm.
7. **Export Excel while signed out:** a spreadsheet downloads from local data. (If the CDN is unreachable, an in-app error appears instead — retry from the same button.)
8. **Finish a book:** set its current page to the total (in the form or with quick-progress). It flips to Completed, its card shows 100% and **"Read again"**, and "Finished [date]" appears.
9. **Daily goal:** change the goal and tap **Save goal** — a confirmation toast appears, and Home shows today's progress ("N / M pages today").
10. **Dates:** add a book with a Started and Finished date — both show on its card.
11. **Library search:** type part of a title or author — only matching books show; clear it — all books return.
12. **Forgot password:** tap the link with your email typed in — the reset email arrives.
13. **Insights:** read more than the weekly goal — the percentage goes above 100% with "Goal exceeded".
14. **Pick up where you left off:** with no book marked "reading", the card shows your most recently finished book labelled "Last finished".
15. **Sign out:** the footer ("Sign in to sync…") reappears.
16. **Icon files live:** open https://reading-shelf.ronniew3099.workers.dev/icon-192.png and /icon-512.png — both show the bookshelf.
17. **Add to Home Screen (iPhone, Safari):** open the site in a fresh Safari tab → Share → Add to Home Screen. The bookshelf icon shows, and the app opens full-screen with no browser bar.
18. **Service worker:** after the first load, reload once — the app shell still opens, and the manifest serves at /manifest.json.

If any check fails, the most likely suspects are the deployed file differing from this repo version (see "Deployed file vs repo" above) and the Supabase `books` table lacking a `started_at` column (expected — started dates are local-only by design).
