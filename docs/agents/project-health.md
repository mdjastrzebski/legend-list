<!-- index-github: snapshot of LegendApp/legend-list on 2026-10-03, last 365 days. Numbers go stale; rerun index-github to refresh. -->
# Legend List: Project Health

**Verdict:** Active, but run by one person. Stable releases come out about every 5 days and issues get fixed steadily. Almost all of that work is done by one maintainer, though, and outside contributions are slow: only about a third of external PRs got any maintainer response.
**Risks:** *Concentration*: one person wrote 97% of commits and did 100% of merges in the last year (bus factor 1). *Stability*: three major versions in 17 months (1.0 in April 2025, 2.0 in September 2025, 3.0 in May 2026), and old majors are dropped. License risk: none (MIT, no CLA).
**Snapshot:** 2026-10-03, covering 2025-10-03 to 2026-10-03. The window is not truncated: it holds all 86 PRs and 146 issues opened in that time. The local clone is current to 2026-09-29 (v3.6.0).

## What it is
Legend List is a virtualized list component for React Native and React DOM, written in TypeScript with no native code. Per its README, it is "a drop-in replacement for `FlatList` and `FlashList`" with better performance, especially for dynamically sized items. Its strengths are chat-style lists (initial scroll at the end, keeping the scroll at the end, keyboard integration), `maintainVisibleContentPosition`, item recycling, and sticky headers. It's aimed at app developers who've outgrown `FlatList`, and since v3 also at web apps. See [contributing.md](contributing.md) and [areas/](areas/) for the code.

## Alternatives
These comparisons rest on general knowledge plus the project's own README claims. No independent benchmark was run and no web search was done. Download counts are from npm for 2026-09-02 to 2026-10-01.

| Project | Approach | Differences | Activity |
| --- | --- | --- | --- |
| **Legend List** | Pure-JS recycling and virtualization for RN and web | Strong on dynamic sizes, chat, and MVCP; one maintainer | 2.39M dl/mo, v3.6.0 on 2026-09-29 |
| [Shopify/flash-list](https://github.com/Shopify/flash-list) | Recycling list for RN; v2 is JS-only and needs the new architecture | Company-backed, larger team, more adopters; RN only | 9.25M dl/mo, v2.3.3 on 2026-10-01 |
| `FlatList` (React Native core) | `VirtualizedList` that mounts and unmounts items, no recycling | Ships with RN and needs no dependency; slower with large or dynamic lists | Moves with RN releases |
| [petyosi/react-virtuoso](https://github.com/petyosi/react-virtuoso) | DOM virtualization with automatic measurement | Web only; mature chat and reverse-scroll support | 14.1M dl/mo, active |
| [TanStack/virtual](https://github.com/TanStack/virtual) | Headless virtualizer; you render the markup | Web-first, very flexible, more wiring | 101M dl/mo, active |

## Maturity
- **Version:** 3.6.0, past 1.0. npm lists 108 stable versions and about 270 in total, including beta and next tags.
- **Majors:** 1.0.0 on 2025-04-23, 2.0.0 on 2025-09-04, and 3.0.0 on 2026-05-28 after a long 3.0 beta (beta.56 is still on the `beta` dist-tag).
- **Breaking changes:** the CHANGELOG notes the v3 change to `maintainVisibleContentPosition` defaults and links a [v3 migration guide](https://legendapp.com/open-source/list/v3/migration/). No written semver policy exists.
- **Old majors:** they are not supported. The maintainer told a v1 user "I am not maintaining v1 as v2 and v3 are strictly better" ([#413](https://github.com/LegendApp/legend-list/pull/413)).
- **Production use:** 2.39M npm downloads a month, about a quarter of FlashList's. There is no adopters list in the repo.

## Activity
### Maintainers
- **Merges:** `jmeistrich` (Jay Meistrich) merged all 23 landed PRs. Only one review came from anyone else (`kewde`, once).
- **Issues:** `jmeistrich` gave every maintainer response on issues (112).
- **Commits:** 1,226 in the last year, 1,189 of them (97%) by Jay Meistrich. The rest are spread over about 17 people with 1–7 commits each. All-time, the second-largest committer has 61 of 2,330 commits.
- **Workflow:** the maintainer commits straight to `main` rather than through PRs. He opened only 1 PR in the window.
- **Bus factor:** 1.
- **Monthly commits:** between 34 and 351 a month over the last two years. The peak was April 2026, the run-up to 3.0. Recent months: July 35, August 52, September 34.

### Responsiveness
| PRs (12 months) | External | Maintainer |
| --- | --- | --- |
| Opened | 85 | 1 |
| Landed / closed unmerged / open | 22 / 29 / 34 | 1 / 0 / 0 |
| First maintainer response, median (p75) | 13 days (79), but only 31 of 85 PRs got one | — |
| Days to land, median (p90) | 6.7 (54) | 0 (self-merged) |

- **Rejected PRs:** of the 29 closed unmerged, authors closed 17 themselves and the maintainer closed 12.
- **Issues:** 146 opened, 88 closed: 87 as completed, 1 as not planned. No stale bot.
  - Median days to close: 42 (p75 145).
  - Median first maintainer response: 33 days (n=78). Any human response comes faster, at 15 days, because users help each other.
  - 68 of the 146 issues never got a maintainer response.
- **The `done?` label:** used 18 times. It marks issues the maintainer believes are fixed and asks the reporter to confirm.

### Backlog
- **Open issues:** 64 total; 34 are older than 90 days and 6 older than a year. 61 have no label and 39 have no comments. 48 opened in the window have gone more than 14 days without a maintainer response.
- **Open PRs:** 36 total; 25 are older than 30 days and 14 older than 90 days. 20 have no review, and 25 external PRs have gone more than 14 days without a maintainer response. A batch of narrowly scoped fix PRs from one contributor (`aio39`, [#549](https://github.com/LegendApp/legend-list/pull/549)–[#556](https://github.com/LegendApp/legend-list/pull/556)) arrived in the last 10 days.

### Releases
- **Cadence:** 36 stable releases in the window, with a median of 4.6 days between them (p90 10 days). Latest stable: v3.6.0 on 2026-09-29.
- **How they're cut:** GitHub releases come from `create-release.ts`, and the CHANGELOG is written per version with `prep-changelog`. The git tags start at v2.0.5; earlier history is only on npm.

## Backing and funding
- **Owner:** the `LegendApp` GitHub organization (also home of Legend-State). In practice it's one person's project. Commit email domains are personal, and no company domain shows up beyond a few one-off contributors.
- **Funding:** GitHub Sponsors for `jmeistrich` (`.github/FUNDING.yml`).
- **Commercial angle:** none in the repo. There is no paid tier, no `ee/` directory, and no second license.

## License
- **Main license:** MIT (a single `LICENSE` file), which allows commercial use.
- **CLA / DCO:** neither. No CLA bot and no `Signed-off-by` lines.
- **Runtime dependencies:** one, `use-sync-external-store` (MIT); `react` is a peer dependency. Nothing to flag.

## Security
- **Reporting channel:** none. There is no `SECURITY.md`, private vulnerability reporting is off, and there are no published advisories.
- **Dependency updates:** no Dependabot or Renovate config.
- **Install-time code:** the published package strips `scripts` (`postbuild.ts`), so nothing runs at install. The attack surface is small: client-side UI code with one runtime dependency.

## Community
- **Where discussion happens:** GitHub Issues (no templates or contact links) and GitHub Discussions (17 threads). There is no code of conduct.
- **Decisions:** the maintainer makes them alone. There's no `CODEOWNERS`, `MAINTAINERS`, or governance doc.
- **Automated review:** a Codex review bot (`chatgpt-codex-connector`) comments on PRs (11 comments in the window).

## Contributing in practice
- **What's welcome:** nothing is labelled good-first-issue or help-wanted, and there are no milestones or pinned roadmap. Only 1 open issue is labelled `planned`.
- **How PRs fare:** the usual outcome for a bug-fix PR is that the maintainer **fixes the bug himself, often differently**, and then closes the PR. Examples:
  - [#401](https://github.com/LegendApp/legend-list/pull/401)/[#402](https://github.com/LegendApp/legend-list/pull/402): "I unfortunately fixed the issue before I saw the PR, but the fix ended up being quite different."
  - [#403](https://github.com/LegendApp/legend-list/pull/403): already fixed before he saw the PR.
  - [#430](https://github.com/LegendApp/legend-list/pull/430): superseded by removing `transform` positioning everywhere.
- **PRs that do land:** they are usually rebased and reworked by the maintainer. [#381](https://github.com/LegendApp/legend-list/pull/381) was "tweaked for performance" after 133 days. [#393](https://github.com/LegendApp/legend-list/pull/393) was rebased and partly dropped after 87 days.
- **Implication:** a well-written issue with a minimal repro may get a fix faster than a PR. If you do open a PR, keep it small, include a test, and expect it to be rewritten.
- **Out of scope:** fixes for old majors. v1 isn't maintained ([#413](https://github.com/LegendApp/legend-list/pull/413)).
- **Credit:** release notes rarely name contributors. Merged PRs keep authorship, but when the maintainer reimplements a fix, the PR author gets no commit credit.
- **Written rules:** see [contributing.md](contributing.md).

## Problem areas
The hotspots below come from fix commits in the last year (463 commits starting with `fix`/`revert`). The most-touched files are `src/components/LegendList.tsx` (89), `src/core/calculateItemsInView.ts` (51), `src/core/bootstrapInitialScroll.ts` (32), `src/integrations/keyboard.tsx` (29), `src/core/finishScrollTo.ts` (24), and `src/integrations/reanimated.tsx` (21).

| Area | Theme | Evidence |
| --- | --- | --- |
| Scroll anchoring and MVCP ([scrolling.md](areas/scrolling.md)) | Jumps or drift when prepending, resizing, or loading more | [#491](https://github.com/LegendApp/legend-list/issues/491), [#531](https://github.com/LegendApp/legend-list/issues/531), [#535](https://github.com/LegendApp/legend-list/issues/535), [#530](https://github.com/LegendApp/legend-list/issues/530), [#515](https://github.com/LegendApp/legend-list/issues/515), [#534](https://github.com/LegendApp/legend-list/issues/534) |
| Initial scroll and end-following (`bootstrapInitialScroll`, `maintainScrollAtEnd`) | Chat lists open at the wrong row, flicker, or stop following | [#547](https://github.com/LegendApp/legend-list/issues/547), [#486](https://github.com/LegendApp/legend-list/issues/486), [#525](https://github.com/LegendApp/legend-list/issues/525), [#470](https://github.com/LegendApp/legend-list/issues/470), [#529](https://github.com/LegendApp/legend-list/issues/529) |
| Keyboard and chat integration (`src/integrations/keyboard.tsx`) | Offsets and jumps with the keyboard open | [#557](https://github.com/LegendApp/legend-list/issues/557), [#451](https://github.com/LegendApp/legend-list/issues/451), [#475](https://github.com/LegendApp/legend-list/issues/475); many closed (#361, #362, #369, #383, #385) |
| Sticky headers and SectionList | Headers misplaced after data changes; jitter when animated | [#445](https://github.com/LegendApp/legend-list/issues/445) (open since May; the 3.1.0 fix didn't work for reporters), [#455](https://github.com/LegendApp/legend-list/issues/455), [#489](https://github.com/LegendApp/legend-list/issues/489), [#512](https://github.com/LegendApp/legend-list/issues/512) |
| Recycling and containers ([virtualization.md](areas/virtualization.md)) | Stale or removed items render; blank areas in nested lists | [#526](https://github.com/LegendApp/legend-list/issues/526), [#545](https://github.com/LegendApp/legend-list/issues/545), [#544](https://github.com/LegendApp/legend-list/issues/544), [#325](https://github.com/LegendApp/legend-list/issues/325) (open about a year) |
| Horizontal and RTL | Blank or mirrored horizontal lists; cross-axis sizing | [#477](https://github.com/LegendApp/legend-list/issues/477), [#476](https://github.com/LegendApp/legend-list/issues/476), [#458](https://github.com/LegendApp/legend-list/issues/458), [#459](https://github.com/LegendApp/legend-list/issues/459), [#528](https://github.com/LegendApp/legend-list/issues/528) |
| Web (React DOM / RN-web) | Scrollbar props, leftover padding, mobile-web jank | [#465](https://github.com/LegendApp/legend-list/issues/465), [#537](https://github.com/LegendApp/legend-list/issues/537), [#548](https://github.com/LegendApp/legend-list/issues/548), [#488](https://github.com/LegendApp/legend-list/issues/488), [#448](https://github.com/LegendApp/legend-list/issues/448) |
| Reanimated and layout animation (`src/integrations/reanimated.tsx`) | Missing animated variants; `itemLayoutAnimation` gaps | [#487](https://github.com/LegendApp/legend-list/issues/487) (most-reacted open issue), [#513](https://github.com/LegendApp/legend-list/issues/513), [#469](https://github.com/LegendApp/legend-list/issues/469), [#440](https://github.com/LegendApp/legend-list/issues/440), [#546](https://github.com/LegendApp/legend-list/issues/546) |

Most-wanted open issues get no maintainer reply: none of the top 5 by reactions has one. The bar is low, though (2–4 reactions each). [#482](https://github.com/LegendApp/legend-list/issues/482) (Liquid Glass not rendering) was traced by users to an upstream Expo issue.

## Caveats
- **Maintainer PR numbers are meaningless,** because the maintainer works directly on `main`. Judge responsiveness by external PRs and issues.
- **Labels are barely used,** so issue themes come from reading titles, not label counts. The PR `fix_prs` hotspots are based on only 22 landed PRs; the git fix-commit counts are more reliable.
- **Response medians only count items that got a response.** Many PRs and issues never get one, so also read the unanswered counts (54 of 85 external PRs and 68 of 146 issues).
- **Recent sample:** many recent issues and PRs are long, very specific write-ups that arrived in bursts. That inflates the open counts for the last few weeks.
- **Alternatives:** the table relies on the project's README and general knowledge, not a fresh web comparison.
