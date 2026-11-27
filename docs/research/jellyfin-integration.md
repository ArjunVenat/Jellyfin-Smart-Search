# Research: how Jellyfin can surface Moments

**Ticket:** [#3](https://github.com/ArjunVenat/Jellyfin-Smart-Search/issues/3)
**Date:** 2026-11-26
**Versions checked:** Jellyfin server `v12.4` (released 2026-10-26) and Jellyfin web `v12.4`. All source links are pinned to those tags. Plugin NuGet packages `Jellyfin.Controller` / `Jellyfin.Model` 12.4.0 exist on nuget.org.

Terms: a **Moment** is a timestamped hit inside a **Media item** (movie or episode); a **Segment** is the indexed span of video a Moment points at. "Ticks" are .NET ticks (10,000,000 per second), which is what every Jellyfin position field uses.

---

## TL;DR

1. **Since 12.0 a server plugin can plug into Jellyfin's own search.** `ISearchProvider` / `IExternalSearchProvider` was added in [PR #16121](https://github.com/jellyfin/jellyfin/pull/16121) (merged 2026-06-07; not present in `v10.11.6`, present in `v12.0`). The stock web client's search page calls `GET /Items?searchTerm=...`, which now goes through the providers. So Moments-matched Media items can show up in the stock search UI with no client change.
2. **But a search provider can only return `(ItemId, Score)`.** There is no field for a timestamp, so stock search can rank the right *Media item* but cannot show or start at the *Moment*.
3. **If an external provider returns anything, Jellyfin's title search is dropped for that query.** An `IExternalSearchProvider` that returns any results replaces the built-in SQL title search completely. A plugin that marks itself `IInternalSearchProvider` instead gets merged with title search (best score wins), but then it has to do its own user-access filtering.
4. **A plugin can add its own REST endpoints** (ASP.NET controllers in plugin assemblies are registered), with Jellyfin's normal auth policies. That is the natural home for a `GET /Moments?q=` API that returns item id + start ticks + score.
5. **Starting playback at a timestamp never needs a write to `UserData`.** The web client's `playbackManager.play({ ids, startPositionTicks })` does it, and chapter ("Scenes") cards already use it. From outside the client, `POST /Sessions/{sessionId}/Playing?playCommand=PlayNow&itemIds=…&startPositionTicks=…` makes a running web client start at that position. The web client has **no URL deep link** with a start time.
6. **Non-destructive progress bar: not possible in the stock web client, easy in a modified one.** The stock card progress bar is drawn only from the user's real `UserData.PlayedPercentage`. But the web client's own `getProgressBarHtml(item, { userData })` accepts a substitute `userData` object, so a modified client can draw a Moment's position on a card without touching the server. Writing `PlaybackPositionTicks` to fake it would destroy the real resume point, and the server would also reshape the value (see section 3.3).
7. **Recommended shape:** a server plugin (Moments endpoint + optional search provider) plus a small web-client modification for the Moment cards. A full server fork is no longer needed for search.

---

## 1. Plugin API

### 1.1 What the search-provider interfaces look like

- [`MediaBrowser.Controller/Library/ISearchProvider.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Controller/Library/ISearchProvider.cs): `Name`, `Priority` ("Lower values execute first"), `CanSearch(SearchProviderQuery)`, and `SearchAsync(...)` returning `IReadOnlyList<SearchResult>`.
- [`IExternalSearchProvider.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Controller/Library/IExternalSearchProvider.cs): the same, but `SearchAsync` returns `IAsyncEnumerable<SearchResult>`. Described as "external search providers that offer enhanced search capabilities".
- [`IInternalSearchProvider.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Controller/Library/IInternalSearchProvider.cs): a marker interface, "internal search providers that typically query the local database directly". The built-in [`SqlSearchProvider`](https://github.com/jellyfin/jellyfin/blob/v12.4/Emby.Server.Implementations/Library/Search/SqlSearchProvider.cs) is one (priority 100, scores 100/80/75/50 for exact/prefix/word-prefix/contains name match).
- [`SearchProviderQuery.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Controller/Library/SearchProviderQuery.cs): `SearchTerm`, `UserId`, `IncludeItemTypes`, `ExcludeItemTypes`, `MediaTypes`, `Limit`, `ParentId`.
- [`SearchResult.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Controller/Library/SearchResult.cs): a struct with only `Guid ItemId` and `float Score`. **No position or extra data.**

Reference plugin from the PR author: [Shadowghost/jellyfin-plugin-meilisearch](https://github.com/Shadowghost/jellyfin-plugin-meilisearch) (`MeilisearchSearchProvider.cs` implements `IExternalSearchProvider`, priority 10).

### 1.2 How providers are found and combined

- Plugin assemblies are part of type discovery: `ApplicationHost.GetComposablePartAssemblies()` yields `_pluginManager.LoadAssemblies()`, and startup calls `Resolve<ISearchManager>().AddParts(GetExports<ISearchProvider>())` ([`Emby.Server.Implementations/ApplicationHost.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Emby.Server.Implementations/ApplicationHost.cs)). So implementing the interface in a plugin is enough to register it.
- [`SearchManager.GetSearchResultsAsync`](https://github.com/jellyfin/jellyfin/blob/v12.4/Emby.Server.Implementations/Library/Search/SearchManager.cs) runs external and internal providers in parallel, then:
  - external results are filtered by the user's library access (`FilterByUserAccessAsync`);
  - **`if (externalResults.Count > 0) return externalResults;`**, otherwise the internal results are returned;
  - within a group, results from several providers are merged by keeping the best score per item, sorted by score, cut to `Limit` (default 100).
  - A provider that throws is logged and treated as returning nothing.
- The code comment says why external results are filtered and internal ones are not: "Internal providers apply user-access filtering inline in their queries. External providers don't know about user permissions… The Items controller's second roundtrip via folder.GetItems applies most of these again, but it does not restrict by TopParentIds when ItemIds is set."

Consequences for us:

| Register as | Title search still works? | Access filtering | Notes |
|---|---|---|---|
| `IExternalSearchProvider` | **No**, whenever we return at least one hit | Done by Jellyfin | Typing "Pulp" would show only our Moments-ranked items, not the title match, unless our provider also does title matching. |
| `IInternalSearchProvider` | Yes, merged with SQL title results by best score | **We must do it** (library access / TopParentIds) | Our scores share one scale with SQL's 50–100, so the ranking needs calibrating. Relies on the marker interface being usable by plugins, which the code allows but the docs do not promise. |

### 1.3 Where search results appear

- `GET /Items?searchTerm=…` ([`Jellyfin.Api/Controllers/ItemsController.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Jellyfin.Api/Controllers/ItemsController.cs)) calls `_searchManager.GetSearchResultsAsync` (asking for `limit * 3`), then loads the items by id through the normal query, which applies user filtering.
- `GET /Search/Hints` ([`SearchController.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Jellyfin.Api/Controllers/SearchController.cs)) goes through `SearchManager.GetSearchHintsAsync`, which uses the same providers.
- The web client's search page uses `getLibraryApi(api).getItems({ searchTerm, recursive: true, … })` ([`src/apps/legacy/features/search/api/fetchItemsByType.ts`](https://github.com/jellyfin/jellyfin-web/blob/v12.4/src/apps/legacy/features/search/api/fetchItemsByType.ts), [`useVideoSearch.ts`](https://github.com/jellyfin/jellyfin-web/blob/v12.4/src/apps/legacy/features/search/api/useVideoSearch.ts)). So **provider results reach the stock web UI and every other client that searches through `/Items`**, shown as normal Media item cards.

### 1.4 Custom endpoints and other plugin hooks

- Controllers in plugin assemblies are registered: `ApplicationHost` collects assemblies of all discovered types that are `ControllerBase` ([`ApplicationHost.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Emby.Server.Implementations/ApplicationHost.cs)). The Meilisearch plugin's `Api/MeilisearchController.cs` uses `[Route("Plugins/Meilisearch")]` and `[Authorize(Policy = Policies.RequiresElevation)]`.
- `IPluginServiceRegistrator.RegisterServices(IServiceCollection, IServerApplicationHost)` lets a plugin add its own DI services ([`MediaBrowser.Controller/Plugins/IPluginServiceRegistrator.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Controller/Plugins/IPluginServiceRegistrator.cs)). Scheduled tasks (`IScheduledTask`) are discovered the same way, which suits incremental indexing triggers.
- Config UI: `IHasWebPages` / `PluginPageInfo` ([`MediaBrowser.Model/Plugins/`](https://github.com/jellyfin/jellyfin/tree/v12.4/MediaBrowser.Model/Plugins)) lets a plugin ship HTML pages, but they show up in the **admin dashboard** plugin section (web: `src/apps/dashboard/features/plugins/`), not in the user-facing UI.
- Media segments ([`IMediaSegmentProvider`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Controller/MediaSegments/IMediaSegmentProvider.cs)) let a plugin attach timed spans to items, but the type is a fixed enum: `Unknown, Commercial, Preview, Recap, Outro, Intro` ([`MediaSegmentType.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/src/Jellyfin.Database/Jellyfin.Database.Implementations/Enums/MediaSegmentType.cs)), and clients use them for skip prompts. Not a fit for query-dependent Moments.
- Plugin template: [jellyfin/jellyfin-plugin-template](https://github.com/jellyfin/jellyfin-plugin-template), .NET 9, `Jellyfin.Controller` + `Jellyfin.Model` with `ExcludeAssets=runtime` (the template on `master` still pins 10.11.5; 12.4.0 packages exist).

### 1.5 What a plugin cannot do

- **Cannot add UI to the user-facing web client.** There is no web-client plugin or script extension point in `jellyfin-web` `v12.4`. Plugin pages go only to the admin dashboard. The only server-side branding hook is CSS: `BrandingOptions` has `LoginDisclaimer`, `CustomCss`, `SplashscreenEnabled`, `SplashscreenLocation` ([`MediaBrowser.Model/Branding/BrandingOptions.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Model/Branding/BrandingOptions.cs)). No custom JS. (Third-party plugins exist that patch the served web files to inject JS. They are not official and were not evaluated here.)
- **Cannot attach a timestamp to a search result** (`SearchResult` is id + score only), so the stock search page cannot show "at 1:23:45".
- **Cannot keep title search when acting as an external provider** (section 1.2).
- The official plugin overview page ([jellyfin.org/docs/general/server/plugins](https://jellyfin.org/docs/general/server/plugins/)) lists server-side areas only (auth, streaming, Live TV, metadata, notifications, scheduled tasks) and says nothing about changing clients.

## 2. Sidecar + client change vs. server fork

| Option | What it buys | Cost |
|---|---|---|
| **A. Plugin only** (search provider, stock clients) | Moments-matched items in the stock search page on every client that uses `/Items?searchTerm=`. Zero client work. | No timestamp shown, no start-at-Moment. Either breaks title search (external) or needs own access filtering (internal). Plugin runs inside the Jellyfin process, so heavy work (embedding the query) should live in a separate service the plugin calls. |
| **B. Plugin + modified web client** | A plugin endpoint (e.g. `GET /Plugins/SmartSearch/Moments?q=`) returns `{itemId, startTicks, endTicks, score}`. The modified client renders Moment cards with a progress bar and plays at the Moment. | Keep a patched `jellyfin-web` in step with upstream. Web client only; other apps (Android TV, Swiftfin, …) still see option A at best. |
| **C. Sidecar service + modified web client** (no plugin) | Same UI as B; the search API lives entirely outside Jellyfin. | The client must call a second origin (CORS, separate auth, see section 4). No presence in stock search. |
| **D. Server fork behind a feature flag** | Could add a timestamp to search DTOs for every client. | Ongoing rebase cost on a fast-moving server (three releases in October 2026 alone), and other clients would still need changes to use any new field. Since 12.0 the search-provider seam removes the main reason to fork. |

The client side is needed in every option that shows the Moment, because the only progress UI in the stock client reads real user data (section 3.3).

## 3. Starting playback at a timestamp

### 3.1 Inside the web client

- `playbackManager.play({ ids, startPositionTicks, serverId })` starts at a position ([`src/components/playback/playbackmanager.js`](https://github.com/jellyfin/jellyfin-web/blob/v12.4/src/components/playback/playbackmanager.js)). A start position also skips cinema-mode intros (`if (options.startPositionTicks || … ) ` branch).
- Cards with `data-action="play"` and `data-positionticks="…"` are wired to that call by [`src/components/shortcuts.js`](https://github.com/jellyfin/jellyfin-web/blob/v12.4/src/components/shortcuts.js) (`startPositionTicks = parseInt(card.getAttribute('data-positionticks'))`).
- Chapter "Scenes" cards on the details page already use exactly this: [`chaptercardbuilder.js`](https://github.com/jellyfin/jellyfin-web/blob/v12.4/src/components/cardbuilder/chaptercardbuilder.js) emits `data-action="play" … data-positionticks="${chapter.StartPositionTicks}"` and shows the time with `datetime.getDisplayRunningTime`. A Moment card can copy this pattern.
- **No deep link with a time.** The details route reads `params.id` (and genre/artist variants) only ([`src/apps/legacy/controllers/itemDetails/index.js`](https://github.com/jellyfin/jellyfin-web/blob/v12.4/src/apps/legacy/controllers/itemDetails/index.js)); none of the user routes in [`legacyRoutes/user.ts`](https://github.com/jellyfin/jellyfin-web/blob/v12.4/src/apps/legacy/routes/legacyRoutes/user.ts) take a start time.

### 3.2 From outside the client (sidecar or plugin)

- `POST /Sessions/{sessionId}/Playing` with `playCommand`, `itemIds` and `startPositionTicks` ([`Jellyfin.Api/Controllers/SessionController.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Jellyfin.Api/Controllers/SessionController.cs)). The web client handles the `Play` command and passes `startPositionTicks: Data.StartPositionTicks` to the player ([`src/scripts/serverNotifications.js`](https://github.com/jellyfin/jellyfin-web/blob/v12.4/src/scripts/serverNotifications.js)). This works with a **stock** client, but only on a session that is already open, so it is a "cast to my open tab/TV" flow rather than a link.
- `StartTimeTicks` on `PlaybackInfo` ([`MediaInfoController.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Jellyfin.Api/Controllers/MediaInfoController.cs)) is a streaming/transcoding parameter the player sets itself. The query-string form is marked `[ParameterObsolete]` in favour of the request body. It is not a way for us to start a client at a time.

### 3.3 Why writing `UserData.PlaybackPositionTicks` is the wrong tool

- `UserDataManager.UpdatePlayState` ([`Emby.Server.Implementations/Library/UserDataManager.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Emby.Server.Implementations/Library/UserDataManager.cs)) changes any reported position:
  - below `MinResumePct` (default **5%**), the position is set to 0;
  - above `MaxResumePct` (default **90%**) or within 1 s of the end, the position is set to 0 and the item is **marked played**;
  - items shorter than `MinResumeDurationSeconds` (default **300**) are marked played.
  (Defaults in [`MediaBrowser.Model/Configuration/ServerConfiguration.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Model/Configuration/ServerConfiguration.cs).)
- So a fake resume point would (a) overwrite the user's real one, (b) fail to show a bar for Moments in the first 5% or last 10%, and (c) could mark a movie as watched. It would also appear in "Continue Watching" on every client.

### 3.4 Non-destructive progress indicator: the answer

- **Stock web client: no.** `getProgressBarHtml(item, options)` in [`src/components/indicators/indicators.js`](https://github.com/jellyfin/jellyfin-web/blob/v12.4/src/components/indicators/indicators.js) draws the bar from `userData.PlayedPercentage`, where `userData` is `options.userData` or else `item.UserData`. The server only fills `UserData` from the user's real play state, and there is no other field the card reads.
- **Modified web client: yes, cheaply.** The same function takes `options.userData`, so a Moment card can call it with `{ PlayedPercentage: startTicks / RunTimeTicks * 100 }` (or call `getProgressHtml(pct)` directly) and draw a bar at the Moment with **no server write**. Pair it with `data-action="play" data-positionticks="<startTicks>"` and the click starts at the Moment, exactly like a chapter card.
- **Remaining side effect (by design):** once the user actually plays from the Moment, the client reports progress as usual and that *does* update the real resume point, the same as YouTube. Avoiding even that would need a different play path (an open question below).

## 4. Auth and library permissions

- Jellyfin's authentication handler gives the `Administrator` role to **any API key** and to admin users; normal users get `User` ([`Jellyfin.Api/Auth/CustomAuthenticationHandler.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Jellyfin.Api/Auth/CustomAuthenticationHandler.cs)). Claims include `Jellyfin-UserId`, `Jellyfin-Token`, `Jellyfin-IsApiKey` ([`Jellyfin.Api/Constants/InternalClaimTypes.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Jellyfin.Api/Constants/InternalClaimTypes.cs)).
- Built-in endpoints resolve the acting user with `RequestHelpers.GetUserId`: no `userId` means the caller; a different `userId` is allowed only for administrators ([`Jellyfin.Api/Helpers/RequestHelpers.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/Jellyfin.Api/Helpers/RequestHelpers.cs)). That helper is `internal` to `Jellyfin.Api`, so a plugin controller reads the `Jellyfin-UserId` claim itself.
- **Plugin:** runs in-process. A controller with `[Authorize]` gets the caller's identity from the user's normal token; `[Authorize(Policy = Policies.RequiresElevation)]` and friends are in [`MediaBrowser.Common/Api/Policies.cs`](https://github.com/jellyfin/jellyfin/blob/v12.4/MediaBrowser.Common/Api/Policies.cs). To respect permissions, load the hit items with `ILibraryManager` using an `InternalItemsQuery(user)` (as `SearchManager` does), and drop anything not returned. An external search provider gets this filtering for free (section 1.2).
- **Sidecar:** two ways.
  - Pass through the user's own token (the web client already holds it) and call `GET /Items?ids=…` as that user. Jellyfin filters for you. Simplest and safest.
  - Hold an API key and call `GET /Items?userId=<user>&ids=…`. That works because an API key is admin, but it means the sidecar holds a full-admin credential and must trust whatever user id the client claims.
- Either way, the Moment index itself holds every user's library; filtering must happen at query time, after retrieval and before returning results, so `Limit` should over-fetch (Jellyfin itself asks providers for `limit * 3`).

## 5. Open questions (candidates for new tickets)

1. **External vs. internal provider:** accept losing title search when Moments hit (external), or merge with title search and own access filtering plus score calibration (internal)? Is it OK to rely on a plugin implementing `IInternalSearchProvider`?
2. **How to maintain a web-client modification:** a fork of `jellyfin-web` served by the homelab server, vs. a third-party JS-injection plugin. Needs its own look at upgrade cost.
3. **Other clients:** are Android TV / mobile apps in scope (D3 non-goals)? If not, say "web client only".
4. **Should playing a Moment update the real resume point?** Default Jellyfin behaviour says yes. If the answer is no, that needs a separate playback path.
5. **Where query embedding runs** if search goes through an in-process plugin: the plugin should call out to the latency-path service (ties to [ADR-002](../arch/ADR-002-throughput-vs-latency.md)).
6. **Version pinning:** the search-provider API is new in 12.0 and has no docs beyond the source. Pin the plugin to a server version and watch for breaking changes.
