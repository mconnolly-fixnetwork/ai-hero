# How each candidate Notification event works in the code today

Research for issue #2 (map #1, In-app Notifications). Sources are the app code at the commit this branch was cut from. Paths are relative to `app/`.

## Who is who

- **Platform Admin** = any user with `users.role === UserRole.Admin` (`"admin"`), `db/schema.ts:11`. Admin-only routes check `currentUser.role !== UserRole.Admin` (e.g. `routes/admin.refunds.tsx:67`, `routes/admin.courses.tsx:103,128`).
- **Instructor of a course** = `courses.instructorId`. Instructor routes allow the owner or any admin (`routes/instructor.$courseId.tsx:584,669`).
- **Team Admin** = a `team_members` row with `role = TeamMemberRole.Admin` (`db/schema.ts:40,266`). The buyer of a Team Licence becomes its Team Admin (`getOrCreateTeamForUser`, `routes/courses.$slug.purchase.tsx:211-219`). Coupons (Seats) carry `teamId` and `purchaseId` (`db/schema.ts:280`).

## Events

| # | Event | Where | Recipients | Works as assumed? |
|---|---|---|---|---|
| E1 | Refund Request decided | `lib/refunds.server.ts` `approveRefundRequest` (:614, update :669) and `declineRefundRequest` (:692, update :713), called from the `routes/admin.refunds.tsx` action | Buyer = `purchases.userId`. Already gets an email via `buildDecisionEmail` (:682, :735) | Yes. The first decision wins (:587). A decline needs a note (:705) |
| E2 | New Refund Request | `routes/courses.$slug.refund.ts` → `raiseRefundRequest` (`lib/refunds.server.ts:357`, insert :380), checks in `checkRefundRequest` (:270) | All users with role `admin`. There is no per-course admin | Yes |
| E3 | Seat claimed | `routes/redeem.$code.tsx` `redeemCoupon`: checks (seat on hold, refunded, already enrolled, country), then `update(coupons)` sets `redeemedByUserId` (:98) and inserts an Enrollment (:107) | Team Admin(s) of `coupons.teamId`, which in practice is the Team Licence buyer (`purchases.userId` of `coupons.purchaseId`) | Yes. The two writes are not in a transaction, so notify after both |
| E4 | Comment or question on a lesson | Top-level comment insert in `routes/courses.$slug.lessons.$lessonId.tsx:431` (`parentId` null) | Course `instructorId`. The questions inbox treats a "question" as a top-level comment by a non-admin (`routes/instructor.questions.tsx:64-67`) | Yes. Skip it when the author is the Instructor |
| E5 | Reply to my comment | Same insert with `parentId` (`routes/courses.$slug.lessons.$lessonId.tsx:417-432`). Instructors also reply from `routes/instructor.questions.tsx:139-154` | Author of the parent comment | **Yes, threads exist**, but only one level deep: replying to a reply is rejected (:425; schema comment `db/schema.ts:339`). Comments can be soft-deleted (`deletedAt`) |
| E6 | New Purchase of my course | `createPurchase` inserts at `routes/courses.$slug.purchase.tsx:187` (an individual purchase also enrolls the buyer, :168). Team purchases go through `createTeamPurchase` and create coupons (:231) | Course `instructorId` | Yes. `amountPaid` is in cents and `country` is stored (ADR 0005, 0009). Only Published courses can be bought (:286) |
| E7 | Course status changed by a Platform Admin | `routes/admin.courses.tsx` `updateCourseStatus` (:78), admin-only. The Instructor's own change is at `routes/instructor.$courseId.tsx:695` (owner or admin) | Course `instructorId`. Skip it if the actor is the Instructor | Yes. Statuses are Draft, Published and Archived (`db/schema.ts:17`). Admins can also use the instructor route, so check actor ≠ owner rather than which route ran |
| E8 | New lesson or module in an enrolled course | Module insert at `routes/instructor.$courseId.tsx:221`, lesson insert at :282 | Users with an `enrollments` row for the course | Yes. Fan-out can be large. Lessons have no draft state, so the event fires as soon as one is created |
| E9 | Enrolled course unpublished | Same status updates as E7 | Enrolled Students | Works. A status change only touches `courses.status`, so Enrollments stay. The owning Instructor or any Platform Admin can make the change. `lib/refunds.server.ts:658` is the only code that deletes Enrollments, so **only an approved Refund Request makes a Student Lapsed** (`lib/enrollments.ts` `isLapsed` means no Enrollment; ADR 0002) |
| E10 | My role changed | `routes/admin.users.tsx` `updateUserRole` (:60, update :62), admin-only (:106) | The user whose role changed | Yes. `settings.tsx:92` edits the profile only, not the role. Signup always creates Students (`signup.tsx:79`) |
| E11 | New rating on my course | `routes/courses.$slug.tsx` rating action upserts into `course_ratings`: update at :351 or insert at :359 (one row per user and course) | Course `instructorId` | Yes, but a re-rating updates the same row. Decide whether updates should notify |

## Layout and session patterns for a bell

- `routes.ts:10` wraps app pages in `layout("routes/layout.app.tsx", …)`. `root.tsx` has no loader of its own. It renders `<Outlet />` (:76).
- The `layout.app.tsx` loader (~:174) reads `getCurrentUserId(request)` from the cookie session in `lib/session.ts` (the DevUI switches users through `api.switch-user.ts`). It returns `currentUser`, `isTeamAdmin` and `recentCourses`. The loader runs on every navigation, so adding an unread count there gives "delivered on next page load" with no extra work.
- The sidebar UI is in `components/sidebar.tsx`. A bell would fit next to the user block.
- Refund decision emails already go through `lib/mailer.server.ts`. A Notification would be a separate in-app write in the same transaction.
