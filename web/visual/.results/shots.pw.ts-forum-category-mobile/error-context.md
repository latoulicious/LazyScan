# Instructions

- Following Playwright test failed.
- Explain why, be concise, respect Playwright best practices.
- Provide a snippet of code with the fix, if possible.

# Test info

- Name: shots.pw.ts >> forum category
- Location: visual/shots.pw.ts:213:5

# Error details

```
Error: expect(page).toHaveScreenshot(expected) failed

  1413 pixels (ratio 0.01 of all image pixels) are different.

  Snapshot: forum-category.png

Call log:
  - Expect "toHaveScreenshot(forum-category.png)" with timeout 5000ms
    - verifying given screenshot expectation
  - taking page screenshot
    - disabled all CSS animations
  - waiting for fonts to load...
  - fonts loaded
  - 1413 pixels (ratio 0.01 of all image pixels) are different.
  - waiting 100ms before taking screenshot
  - taking page screenshot
    - disabled all CSS animations
  - waiting for fonts to load...
  - fonts loaded
  - captured a stable screenshot
  - 1413 pixels (ratio 0.01 of all image pixels) are different.

```

# Page snapshot

```yaml
- generic [ref=e3]:
  - banner [ref=e4]:
    - button "Toggle navigation" [ref=e5] [cursor=pointer]
    - button "LazyScan" [ref=e7] [cursor=pointer]
  - generic [ref=e8]:
    - complementary [ref=e9]:
      - navigation "Primary" [ref=e10]:
        - button "Library" [ref=e11] [cursor=pointer]
        - button "Forum" [ref=e16] [cursor=pointer]
        - button "Feed" [ref=e21] [cursor=pointer]
        - button "Favorites" [ref=e28] [cursor=pointer]
        - button "Tracking" [ref=e33] [cursor=pointer]
        - button "History" [ref=e39] [cursor=pointer]
        - button "Manage" [ref=e46] [cursor=pointer]
        - button "Find user" [ref=e52] [cursor=pointer]
        - button "Settings" [ref=e58] [cursor=pointer]
      - generic [ref=e64]:
        - button "Harness Reader" [ref=e65] [cursor=pointer]
        - button "Logout" [ref=e68] [cursor=pointer]
        - paragraph [ref=e74]: © 2026 LazyScan
    - main [ref=e75]:
      - button "← Back to forum" [ref=e76] [cursor=pointer]
      - generic [ref=e77]:
        - generic [ref=e78]:
          - paragraph [ref=e79]: Community
          - heading "General" [level=1] [ref=e80]
          - paragraph [ref=e81]: Anything and everything.
          - paragraph [ref=e82]: 2 threads
        - button "New thread" [ref=e84] [cursor=pointer]
      - list [ref=e85]:
        - button "Open Read the pinned post before opening a thread, pinned" [ref=e86] [cursor=pointer]:
          - generic [ref=e87]:
            - heading [level=3] [ref=e88]:
              - generic "Pinned" [ref=e89]
              - text: Read the pinned post before opening a thread
            - paragraph [ref=e92]: Harness Reader · Feb 1, 2026
          - paragraph [ref=e93]:
            - generic [ref=e94]: 2 replies
            - generic [ref=e95]: Active Feb 11, 2026
        - button "Open What are you reading this week?, pinned" [ref=e96] [cursor=pointer]:
          - generic [ref=e97]:
            - heading [level=3] [ref=e98]:
              - generic "Pinned" [ref=e99]
              - text: What are you reading this week?
            - paragraph [ref=e102]: Inkling · Feb 7, 2026
          - paragraph [ref=e103]:
            - generic [ref=e104]: 1 reply
            - generic [ref=e105]: Active Feb 10, 2026
```

# Test source

```ts
  42  |   .hero-bg,
  43  |   .rail-reveal,
  44  |   .rail-carousel-track,
  45  |   .state-loading,
  46  |   .skeleton-grid,
  47  |   .state-loading-dots span,
  48  |   .skeleton-block::after,
  49  |   img.cover-fade {
  50  |     animation: none !important;
  51  |     transition: none !important;
  52  |   }
  53  |   /* Both start at opacity 0 and are revealed by a class/animation whose timing
  54  |      is a race against the screenshot; pin them to their settled state. */
  55  |   img.cover-fade { opacity: 1 !important; }
  56  |   .rail-reveal { opacity: 1 !important; transform: none !important; }
  57  | `;
  58  | 
  59  | test.beforeEach(async ({ page }) => {
  60  |   await page.addInitScript(FREEZE_AUTOPLAY);
  61  |   // Screenshots are read-only by definition, but the reader autosaves progress
  62  |   // and marks chapters read. Writes are answered with a generic success envelope
  63  |   // rather than aborted: the fixture stays untouched for the shots that follow,
  64  |   // and the reader does not paint its "Save failed" chip.
  65  |   await page.route("**/api/v1/**", (route) => {
  66  |     if (route.request().method() === "GET") {
  67  |       return route.continue();
  68  |     }
  69  |     return route.fulfill({
  70  |       status: 200,
  71  |       contentType: "application/json",
  72  |       body: JSON.stringify({ success: true, status: 200, data: null }),
  73  |     });
  74  |   });
  75  | });
  76  | 
  77  | async function settle(page: Page): Promise<void> {
  78  |   await page.addStyleTag({ content: KILL_MOTION });
  79  |   // The boot splash unmounts once session bootstrap resolves; its absence is the
  80  |   // cleanest "shell is live" signal.
  81  |   await page.locator(".boot-splash").waitFor({ state: "detached" });
  82  |   await expect(page.locator(".state-loading, .skeleton-grid")).toHaveCount(0);
  83  |   // Below-the-fold covers are loading=lazy and never enter the viewport, yet a
  84  |   // full-page screenshot renders them. Flipping to eager starts the fetch.
  85  |   await page.evaluate(() => {
  86  |     for (const img of document.querySelectorAll<HTMLImageElement>(
  87  |       'img[loading="lazy"]'
  88  |     )) {
  89  |       img.loading = "eager";
  90  |     }
  91  |   });
  92  |   await page.waitForFunction(() => {
  93  |     if (document.fonts.status !== "loaded") return false;
  94  |     return [...document.images].every((img) => img.complete);
  95  |   });
  96  |   await page.evaluate(() => window.scrollTo(0, 0));
  97  |   await assertMotionPinned(page);
  98  | }
  99  | 
  100 | // Tripwire for the failure FREEZE_AUTOPLAY prevents: a rotated hero or a stepped
  101 | // carousel produces a screenshot that differs by a whole card, which reads as a
  102 | // mystery diff. Fail here instead, naming the cause. No-ops on pages that have
  103 | // neither element.
  104 | async function assertMotionPinned(page: Page): Promise<void> {
  105 |   const state = await page.evaluate(() => {
  106 |     const slides = [...document.querySelectorAll(".hero-slide")];
  107 |     const track = document.querySelector(".rail-carousel-track");
  108 |     return {
  109 |       slideCount: slides.length,
  110 |       activeIndex: slides.findIndex((s) => s.classList.contains("is-active")),
  111 |       trackTransform: track ? getComputedStyle(track).transform : null,
  112 |     };
  113 |   });
  114 |   if (state.slideCount > 0 && state.activeIndex > 0) {
  115 |     throw new Error(
  116 |       `hero rotator advanced to slide ${state.activeIndex} before the shot — ` +
  117 |         "autoplay was not frozen"
  118 |     );
  119 |   }
  120 |   // An untranslated track is `none` or the identity matrix; anything else means
  121 |   // the carousel stepped.
  122 |   if (
  123 |     state.trackTransform !== null &&
  124 |     state.trackTransform !== "none" &&
  125 |     state.trackTransform !== "matrix(1, 0, 0, 1, 0, 0)"
  126 |   ) {
  127 |     throw new Error(
  128 |       `carousel track translated (${state.trackTransform}) before the shot — ` +
  129 |         "autoplay was not frozen"
  130 |     );
  131 |   }
  132 | }
  133 | 
  134 | async function shot(
  135 |   page: Page,
  136 |   path: string,
  137 |   name: string,
  138 |   options: { fullPage?: boolean } = {}
  139 | ): Promise<void> {
  140 |   await page.goto(path);
  141 |   await settle(page);
> 142 |   await expect(page).toHaveScreenshot(`${name}.png`, {
      |                      ^ Error: expect(page).toHaveScreenshot(expected) failed
  143 |     fullPage: options.fullPage ?? true,
  144 |   });
  145 | }
  146 | 
  147 | test("home", async ({ page }) => {
  148 |   await shot(page, "/", "home");
  149 | });
  150 | 
  151 | test("feed", async ({ page }) => {
  152 |   await shot(page, "/feed", "feed");
  153 | });
  154 | 
  155 | test("status", async ({ page }) => {
  156 |   await shot(page, "/status", "status");
  157 | });
  158 | 
  159 | test("favorites", async ({ page }) => {
  160 |   await shot(page, "/favorites", "favorites");
  161 | });
  162 | 
  163 | test("history", async ({ page }) => {
  164 |   await shot(page, "/history", "history");
  165 | });
  166 | 
  167 | test("user lookup", async ({ page }) => {
  168 |   await shot(page, "/user", "user-lookup");
  169 | });
  170 | 
  171 | test("user profile", async ({ page }) => {
  172 |   await shot(page, `/user/${FIXTURE_USER.username}`, "user-profile");
  173 | });
  174 | 
  175 | test("profile", async ({ page }) => {
  176 |   await shot(page, "/profile", "profile");
  177 | });
  178 | 
  179 | test("profile edit", async ({ page }) => {
  180 |   await shot(page, "/profile/edit", "profile-edit");
  181 | });
  182 | 
  183 | // Also covers forum-reports-panel, which renders inside the settings stack.
  184 | test("settings", async ({ page }) => {
  185 |   await shot(page, "/settings", "settings");
  186 | });
  187 | 
  188 | test("manga detail", async ({ page }) => {
  189 |   await shot(page, `/manga/${M}`, "manga");
  190 | });
  191 | 
  192 | test("manage hub", async ({ page }) => {
  193 |   await shot(page, "/manage", "manage");
  194 | });
  195 | 
  196 | test("manage manga create", async ({ page }) => {
  197 |   await shot(page, "/manage/manga/new", "manage-manga-create");
  198 | });
  199 | 
  200 | // Also covers manage-chapters-panel, which renders below the edit form.
  201 | test("manage manga edit", async ({ page }) => {
  202 |   await shot(page, `/manage/manga/${M}`, "manage-manga-edit");
  203 | });
  204 | 
  205 | test("manage chapter upload", async ({ page }) => {
  206 |   await shot(page, `/manage/manga/${M}/chapter`, "manage-chapter");
  207 | });
  208 | 
  209 | test("forum", async ({ page }) => {
  210 |   await shot(page, "/forum", "forum");
  211 | });
  212 | 
  213 | test("forum category", async ({ page }) => {
  214 |   await shot(page, `/forum/${FIXTURE.forumCategory}`, "forum-category");
  215 | });
  216 | 
  217 | test("forum thread", async ({ page }) => {
  218 |   await shot(page, `/forum/thread/${FIXTURE.forumThread}`, "forum-thread");
  219 | });
  220 | 
  221 | // The reader is viewport-sized chrome over its own scroll container; a full-page
  222 | // shot of vertical mode would stack every page image instead.
  223 | const DIRECTIONS: ReaderDirection[] = ["ltr", "rtl", "vertical"];
  224 | 
  225 | for (const direction of DIRECTIONS) {
  226 |   test(`reader ${direction}`, async ({ page }) => {
  227 |     await page.addInitScript((value) => {
  228 |       localStorage.setItem("lazyscan:reader:direction", value);
  229 |     }, direction);
  230 |     await shot(page, `/manga/${M}/chapter/${C}`, `reader-${direction}`, {
  231 |       fullPage: false,
  232 |     });
  233 |   });
  234 | }
  235 | 
  236 | // Auth screens render logged out; the shared storageState would redirect past
  237 | // them or paint the signed-in chrome.
  238 | test.describe("unauthenticated", () => {
  239 |   test.use({ storageState: { cookies: [], origins: [] } });
  240 | 
  241 |   test("login", async ({ page }) => {
  242 |     await shot(page, "/login", "login");
```