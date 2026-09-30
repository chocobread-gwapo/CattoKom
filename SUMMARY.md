# ComicPlatform (CattoKom) — Project Summary

A webcomic reading and publishing platform, built to compete directly with Webtoon. Readers browse, follow, read, like, and comment on series; creators write, upload, schedule, and manage their own (or co-authored) series; admins moderate platform-wide.

## Tech Stack

- **Language / Framework:** C# / ASP.NET Core, targeting .NET 10 (LTS)
- **IDE:** Visual Studio 2026
- **UI framework:** Blazor Web App, Interactive Server render mode — no separate JS frontend framework
- **Database:** SQL Server, via Entity Framework Core
- **Auth:** ASP.NET Core Identity (Individual Accounts), including role-based authorization for the Admin area
- **Image storage:** Local disk (`wwwroot/uploads/`) — **not yet production-ready**; the plan is Azure Blob Storage before any real deployment, since local disk storage won't survive on typical cloud hosting
- **Styling:** Hand-written CSS using a shared custom property (token) system for theming, no CSS framework. Fraunces (serif, headings) + Public Sans (body) from Google Fonts. Visual direction: ink-on-paper — sharp corners, bold borders, no drop shadows, grounded in the comic-panel/print medium rather than generic "SaaS card" styling.
- **Upload limit:** 30 MB per image, enforced both at the SignalR transport layer (`MaximumReceiveMessageSize`) and per-file in upload handlers

## Data Model

| Entity | Purpose |
|---|---|
| `ApplicationUser` | Extends Identity's `IdentityUser` with `DisplayName`, `Bio`, `AvatarUrl`, `CreatedAt` |
| `Series` | A comic/webtoon title — title, synopsis, cover image, status (Ongoing/Completed/Hiatus), `CreatorId` |
| `Genre` | Tag like "Fantasy" or "Romance" — many-to-many with `Series` |
| `Chapter` | An episode within a `Series` — number, title, draft/published state, `PublishedAt`, `ViewCount` |
| `Page` | One image within a `Chapter`, in reading order |
| `Comment` | A reader comment on a `Chapter`; schema supports threaded replies via `ParentCommentId`, though the UI currently only shows flat, top-level comments |
| `Follow` | Join entity: which readers follow which series; also tracks `LastViewedAt` for the "new chapter" badge |
| `Like` | Join entity: which readers liked which chapter |
| `SeriesAuthor` | Join entity: co-authors with edit rights on a series, alongside its original `CreatorId` |

## Features

### Reader-facing
- **Home (`/`)** — browsable grid of published series; live search-as-you-type by title; single-select genre filter; both combine
- **Series page (`/series/{id}`)** — synopsis, genre tags, creator byline, full chapter list with publish dates, Follow/Unfollow, Start Reading
- **Reader (`/read/{seriesId}/{chapterNumber}`)** — vertical-scroll page images, previous/next chapter navigation, Like button with count, comments (post + delete)
- **Following (`/following`)** — grid of followed series, with a "New chapter" badge for anything published since the reader's last visit to that series
- **Dark/light theme toggle** — a single button with a sliding sun/moon knob; the active theme is read from a cookie **server-side** and baked into the page before it's sent to the browser, so it's correct immediately regardless of navigation type (this replaced an earlier `localStorage`-based approach that could reset unexpectedly on certain navigations)

### Creator tools
- **Create/Edit series (`/creator/series/{id}/edit`)** — title, synopsis, status, genre selection, cover image upload; also where co-authors are added or removed (only the original creator can manage co-authors; co-authors can edit content but not the author list)
- **Create/Edit chapter** — one unified component serving both `/creator/series/{id}/new-chapter` and `/creator/chapter/{id}/edit` via two routes on the same page. Chapter numbers are auto-computed when creating (no manual entry); pages can be uploaded, previewed as thumbnails, reordered (up/down), and removed, the same way whether creating or editing. Chapters can be saved as a draft, published immediately, or **scheduled** for a future date/time — scheduled chapters stay hidden from every reader-facing query until their publish time passes, with no background job required (dependent on the environment's own clock).
- **My Series (`/creator/my-series`)** — dashboard of series the user created or co-authors, with per-series view counts, chapter counts (published/draft), and quick links to view, edit, or add a chapter

### Admin
- **Admin dashboard (`/admin`)**, restricted to the `Admin` Identity role — lists every series platform-wide with a permanent delete (inline-confirmed, since it cascades to all of a series' chapters, pages, comments, and likes)
- Admins can also delete **any** comment anywhere, not just their own or their own series'

## Notable Design Decisions & Trade-offs

A few deliberate scope choices worth knowing about, rather than gaps that were missed:

- **Likes, not star ratings** — matches how Webtoon/Tapas actually handle engagement (a lightweight per-chapter signal), not a review-style rating
- **Single-select genre filter**, not multi-select AND/OR filtering
- **Flat comments**, not threaded — the schema supports replies, the UI doesn't expose them yet
- **Raw view counter**, not deduplicated/unique visitors — every page load increments it
- **Simple up/down page reordering**, not drag-and-drop
- **Co-authors as an addition, not a replacement** — `Series.CreatorId` remains the single credited "original creator" for the public byline; `SeriesAuthor` is a separate, additive layer for shared edit access
- **No timezone conversion on scheduled publishing** — a scheduled date/time is stored and compared as UTC directly; if the server isn't in UTC-aligned time relative to the person scheduling, the actual go-live moment will be offset from their wall clock

## Notable Bugs Fixed Along the Way

A few worth remembering if similar symptoms show up again:

- **Namespace mismatches** after moving `ApplicationUser.cs`/`ApplicationDbContext.cs` between folders in Solution Explorer — Visual Studio silently rewrites a moved file's `namespace` line to match its new folder
- **Cascade-delete cycle** on the `Follow` table — `Series → Creator` needed an explicit `DeleteBehavior.Restrict`, since SQL Server won't allow two different cascade paths reaching the same table
- **`@page` directive collision** — naming a loop variable `page` inside a `.razor` file breaks the parser, since `@page` is reserved for the routing directive regardless of context
- **`AuthorizeView`/`EditForm` collision** — both use an implicit `context` parameter for child content; nesting one inside the other requires explicitly renaming one via `Context="..."`
- **Missing `RoleManager` registration** — `IdentityDbContext<TUser>` already includes role tables, but the role *services* still need `.AddRoles<IdentityRole>()` chained onto the Identity builder in `Program.cs`
- **Theme resetting unexpectedly** — root-caused to `localStorage` being read by a client-side script that couldn't reliably survive certain full-page-reload navigation boundaries (role-gated `AuthorizeView` blocks were one trigger); fixed by moving theme detection server-side via a cookie read directly into the rendered HTML

## Project Structure (key files)

| File | Role |
|---|---|
| `Models.cs` | All entity classes (`Series`, `Genre`, `Chapter`, `Page`, `Comment`, `Follow`, `Like`, `SeriesAuthor`) |
| `Data/ApplicationUser.cs` | Identity user extension |
| `Data/ApplicationDbContext.cs` | EF Core context, `DbSet`s, relationship configuration |
| `Program.cs` | Service registration, SignalR message size config, seed data (genres, sample series, Admin role) |
| `Components/App.razor` | Root HTML document; server-side theme cookie read |
| `Components/Layout/MainLayout.razor` | App shell; theme toggle switch |
| `Components/Layout/NavMenu.razor` | Sidebar navigation (user-customized with icons matching the app's existing set) |
| `Components/Pages/Home.razor` | Browse, search, genre filter |
| `Components/Pages/SeriesDetail.razor` | Series page |
| `Components/Pages/Read.razor` | Reader, likes, comments |
| `Components/Pages/ChapterForm.razor` | Unified chapter create/edit |
| `Components/Pages/EditSeries.razor` | Series edit + co-author management |
| `Components/Pages/MySeries.razor` | Creator dashboard |
| `Components/Pages/Following.razor` | Followed series list |
| `Components/Pages/Admin.razor` | Platform-wide moderation |
| `wwwroot/app.css` | Shared light/dark theme tokens |

## Not Yet Started

- **Deployment** — moving off local disk storage to Azure Blob Storage, plus actual hosting; explicitly deferred by choice, not forgotten
- **Monetization** — untouched since the very first planning conversation; would need payment processing (Stripe or similar), likely subscriptions or a pay-per-chapter model
- **Broader admin capabilities** — user management/banning, a reporting/flagging system (moderation is currently reactive — an admin has to browse to find something, there's no report button yet)
- **Multi-select genre filtering**, **threaded comment replies**, **cover image upload at series creation** (currently only available via the Edit page, after the fact)
