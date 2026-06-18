# Audit

This document records the current state of the ZamSchool OS frontend. It is observational: the agent read each cited file in full, recorded concrete observations with line numbers, and audited the test suite for freshness. You should treat this as a snapshot, not a backlog.

For style and shell patterns, see [UI-UX.md](./UI-UX.md). For architecture, see [ARCHITECTURE.md](./ARCHITECTURE.md).

## How this audit was done

Each file listed under [Done Well](#done-well) and [Needs Work](#needs-work) was read in full. Observations cite specific line numbers from the working tree, not from prior memory. The test suite was opened test-by-test and each assertion was compared against its targeted source. Working notes live in the ferment's transient docs folder and are not part of this repository.

## Done Well

1. **Design tokens are centralized.** `app/globals.css:5-65` defines a Tailwind 4 `@theme` block with brand palette, workspace surfaces, radius scale, elevation shadows, and motion easing/duration tokens. Every shell reads from the same source.
2. **Mobile-first responsive shell.** The workspace grid uses `100dvh` and switches from `minmax(0, 1fr)` to `14rem minmax(0, 1fr)` at `1024px` (`app/globals.css:159-175`). Main scroll uses `overflow-y: auto` with fluid `clamp()` padding (`app/globals.css:226-232`).
3. **Clean micro-hooks.** `hooks/use-mobile.ts:4-19` uses functional `setState` and removes the `matchMedia` listener on unmount. `hooks/useReveal.ts:1-40` wraps `IntersectionObserver` with configurable threshold, margin, and a `once` flag.
4. **Per-role code splitting.** `components/RoleBasedShell.tsx:12-20` dynamic-imports each role shell, so the admin bundle does not pull in the teacher/parent/payments/payments shells.
5. **Teacher page hook hygiene.** `app/teacher/page.tsx` uses `useCallback` with proper dependency arrays for `loadAttendance` and `loadResults`, `useMemo` for `selectedChild`, and a clean toggle-button group for the range filter. This is the best pattern in the role pages.
6. **Static-grep test discipline.** 25 of 27 tests in `__tests__/` still match the current source (see the table below). The team has avoided the trap of dynamic-execution tests that rot when refactors change internals.
7. **Sanitization at the search boundary.** `lib/workspace-search.ts` exports `sanitizeWorkspaceSearchQuery`, which trims and strips unsafe characters before grouping. Confirmed by test 27.
8. **Semantic HTML on the landing page.** `app/page.tsx` uses `<article>` for feature cards, `<blockquote><footer>` for testimonials, and `aria-label="Platform preview"` for the mock UI.

## Needs Work

1. **Three different profile-bootstrap patterns.** `components/AdminShell.tsx:17` and `components/PaymentsShell.tsx:69-99` use `useWorkspaceContext()`. `components/TeacherShell.tsx:77-80` wraps the content in `TeacherWorkspaceProvider`. `components/ParentShell.tsx:58-92` and `components/StudentShell.tsx:54-88` fetch `/api/account/shell` directly with local state. Pick one.
2. **`MobileDock` API drift.** `components/AdminShell.tsx` does not pass `isActive`. `components/TeacherShell.tsx:307-309` passes a function. `components/ParentShell.tsx:192` passes a function. `components/StudentShell.tsx:220` does not pass `isActive`. `components/PaymentsShell.tsx` reimplements the dock inline. The component contract is not stable.
3. **Monolithic role pages.** `app/teacher/page.tsx` and `app/student/page.tsx` are each roughly 738 lines. Sub-components (`StatCard`, `MiniStat`, `ProfileField`, `formatTimeRange`) are declared at module level inside the page file. Extract them.
4. **Sidebar width drift.** `app/globals.css:171-174` fixes the sidebar at `14rem`. `components/TeacherShell.tsx:120-128` and `components/ParentShell.tsx` use `w-64` (= `16rem`) via Tailwind. Pick one source of truth.
5. **System-wide a11y debt.** No `role="navigation"` or `aria-label` on sidebar `<aside>` or mobile `<nav>` in `AdminShell.tsx`, `TeacherShell.tsx`, `ParentShell.tsx`, `StudentShell.tsx`, or `PaymentsShell.tsx`. No skip-to-content link. No `aria-expanded` on hamburger buttons. No `aria-live` on loaders. No `role="alert"` on errors.
6. **Unread counts go stale.** `components/ParentShell.tsx:85-87` and `components/StudentShell.tsx` set unread counts once at bootstrap and never re-poll. `AdminShell` and `PaymentsShell` delegate to `WorkspaceInboxCenter`, which polls.
7. **Active-path detection is hand-rolled per shell.** `AdminShell.tsx:69-70` uses `Set<string>` exact-match. `ParentShell.tsx:97-102` adds prefix matches manually per route group. `StudentShell.tsx:103-107` adds only `/student/assignments`. Provide one helper.
8. **`prefers-reduced-motion` is incomplete.** `app/globals.css` disables the loader-dot bounce under reduced motion but leaves `animate-enter-up` running. Auditors and users who set the system preference still get motion.
9. **`suppressHydrationWarning` on `<body>`.** `app/layout.tsx:38` sets it, which hides every hydration warning under that body — including legitimate SSR/client mismatches.
10. **Landing page still ships heavy client components.** `app/page.tsx` imports `HeroSection` and `ScrollIndicator` even though `__tests__/app/public-route-performance.test.mjs` expects them absent. The test flagged a real performance issue.
11. **Two stale test assertions.** `__tests__/app/api/account/route.test.mjs` matches `countUnreadNotificationsForUser`, but the source uses `getUnreadCountsForUser`. `__tests__/app/public-route-performance.test.mjs` expects `HeroSection`/`ScrollIndicator` to be absent from `app/page.tsx`; they are not.
12. **Leftover static-analysis comment.** `components/AdminShell.tsx:537` contains `// Static analysis requirements: ...` — a comment from a code-generation pass that should not have survived into source.
13. **Parent page is also monolithic.** `app/parent/page.tsx` is 504 lines with `AttendanceSummary`, `ParentChildRow`, `ParentResultsRow` and five other inline types declared inside the page module. The same extraction pattern is needed.
14. **Principal and payments pages are stubs.** `app/app/principal/page.tsx` (327 bytes) wraps `PrincipalWorkspace` in `DashboardSummaryProvider`. `app/app/payments/page.tsx` (4 lines) wraps `PaymentsDashboardHome`. The admin dashboard at `app/app/dashboard/page.tsx` (10 lines) wraps `AdminDashboardHome` in `DashboardSummaryProvider`. These three are exemplary — the four role pages should converge on this stub pattern.

## Test Freshness

All 27 tests in `__tests__/` were opened and compared against their targeted source. Verdict: **current** if assertions still match, **needs-update** if the assertions are partially out of date, **stale** if the test no longer matches reality. Source of truth: `.kimchi/ferments/.../docs/test-freshness.md`.

| # | Path | Verdict |
|---|------|---------|
| 1 | __tests__/app/api/account/route.test.mjs | needs-update |
| 2 | __tests__/app/api/admin/route.test.mjs | current |
| 3 | __tests__/app/api/auth/mfa/route.test.mjs | current |
| 4 | __tests__/app/api/files/route.test.mjs | current |
| 5 | __tests__/app/api/parent/route.test.mjs | current |
| 6 | __tests__/app/api/student/route.test.mjs | current |
| 7 | __tests__/app/api/teacher/route.test.mjs | current |
| 8 | __tests__/app/app/admin/page-polish.test.mjs | current |
| 9 | __tests__/app/app/admin/page.test.mjs | current |
| 10 | __tests__/app/app/detail-panel-experience.test.mjs | current |
| 11 | __tests__/app/app/page.test.mjs | current |
| 12 | __tests__/app/dashboard/page.test.mjs | current |
| 13 | __tests__/app/edge-offload.test.mjs | current |
| 14 | __tests__/app/error.test.mjs | current |
| 15 | __tests__/app/first-login/page.test.mjs | current |
| 16 | __tests__/app/login/session-switch.test.mjs | current |
| 17 | __tests__/app/public-route-performance.test.mjs | needs-update |
| 18 | __tests__/components/Announcements.test.mjs | current |
| 19 | __tests__/components/admin-dashboard.test.mjs | current |
| 20 | __tests__/components/admin-shell.test.mjs | current |
| 21 | __tests__/components/bulk-import.test.mjs | current |
| 22 | __tests__/components/workspace-global-search.test.mjs | current |
| 23 | __tests__/components/workspace-overlay-layer.test.mjs | current |
| 24 | __tests__/lib/admin-route-client.test.mjs | current |
| 25 | __tests__/lib/gateway-read-client.test.mjs | current |
| 26 | __tests__/lib/workspace-nav.test.mjs | current |
| 27 | __tests__/lib/workspace-search.test.mjs | current |

Totals: 25 current, 0 stale, 2 needs-update. The two needs-update cases are listed under [Needs Work](#needs-work) above (items 10 and 11).
