# Matiks Client — Input Validation & Format Bug Report (App-Wide)

**Repository:** `matiks-client`
**Branch audited:** `feat/input_validation` (commit `5fa07007d`, based on `origin/dev`)
**Server authority:** `matiks-monorepo/services/main-server/internal/textsanitize/` + per-domain limit files
**Date:** 2 September 2026

---

## Scope Note

The `feat/input_validation` branch contains **one commit**, and it touches the profile module only. Verified: no stashes, no other local branches with content (`feat/feat` is identical to `dev`), fresh clone with a single commit on top of `dev`.

So this report has two parts:

- **Part A — Fixed on this branch (4 bugs, profile module).** Implemented, tested, verified.
- **Part B — Open findings across the rest of the app (10 bugs).** Found during the app-wide audit of every input surface. **No fix implemented yet** — current behaviour is the broken behaviour, and a recommended fix is given for each.

Every input surface in the client was inventoried (≈120 files referencing `TextInput`) and each user-generated-content field was checked against its server-side sanitizer.

---

## The Server Contract

All client-side rules must mirror these. From `internal/textsanitize/textsanitize.go`:

| Function | Behaviour |
|---|---|
| `SingleLine(s, max)` | **Deletes** `\n`, `\r` and every `unicode.IsControl` rune — no replacement char. Then `TrimSpace`. `ErrTooLong` if runes > max. |
| `MultiLine(s, max, maxLines)` | Normalises `\r\n`/`\r` → `\n`, deletes other control runes, `TrimSpace`. `ErrTooLong` if runes > max; **`ErrTooManyLines` if line count > maxLines**. |
| `NonEmpty*` variants | As above, plus `ErrEmpty` if the trimmed result is empty. |
| `Name(s, max)` | `NonEmptySingleLine`, then `ErrInvalidNameChars` if any rune is not a letter, a combining mark, or one of `' '`, `'-'`, `'\''`, `'’'`, `'.'`. |

### Every sanitized field in the product

| Domain | Field | Rule | Limit | Client surface |
|---|---|---|---|---|
| `user` | Name | `Name` | 30 runes | Edit Profile — **fixed** |
| `user` | Bio | `SingleLine` | 150 runes | Edit Profile — **fixed** |
| `user` | Social link | `SingleLine` | 500 runes | Social Links sheet — **fixed** |
| `user` | Award title / desc / link | `OptionalSingleLine` / `OptionalMultiLine` | 100 / 500×20 / 300 | *no client UI found* |
| `messages` | Message content | `MultiLine` | 1000 runes, **50 lines** | chatV2 footer — **open (B4)** |
| `messages` | Group name (create) | `NonEmptySingleLine` | 100 runes | FillGroupDetails — open (B9) |
| `messages` | Group name (update) | `SingleLine` | 100 runes | EditGroupDetails — open (B9) |
| `forum` | Thread title | `NonEmptySingleLine` | 200 runes | CreateForumPage — **open (B2)** |
| `forum` | Thread content | `NonEmptyMultiLine` | 10000 runes, 200 lines | CreateForumPage — **open (B2)** |
| `forum` | Reply content | `NonEmptyMultiLine` | 10000 runes, 200 lines | ThreadDetailedInfoReplies — **open (B3)** |
| `club` | Club name | `NonEmptySingleLine` | 100 runes | CreateClub — open (B7) |
| `club` | Club description | `OptionalMultiLine` | 500 runes, **20 lines** | CreateClub — **open (B7)** |
| `inAppReview` | Review description | `OptionalMultiLine` | 2000 runes, 50 lines | ReportExperienceForm — **open (B5)** |
| `brumble` | Team name | `SingleLine` | 100 runes | Brumble register — open |
| `user` | Username | bespoke (`utils.go`) | 4–41, `^[a-z][a-z0-9-_.]*$` | Onboarding + Edit Profile — **open (B1)** |
| `userReport` | Additional comments | **none** | **none** | Report forms — **open (B6)** |

---

# PART A — Fixed on this branch

## A1 — Multi-line bio silently concatenates words · Bio · **High** · ✅ Fixed

**What was wrong.** The bio input is `multiline` with `numberOfLines={4}` (`EditProfilePage.tsx:219-222`), so the UI invites the user to press Enter. The change handler passed raw text through. The server treats bio as single-line and `SingleLine` **deletes** `\n`/`\r` rather than replacing them.

**What it caused.** A bio typed as `Math nerd` / `3x national finalist` was stored and shown to every viewer as `Math nerd3x national finalist`. Silent — the save succeeded, and the user only found the mangled text on revisiting their profile. Every line break in every bio saved through this form produced a word-join defect.

**Severity — High.** Silent, publicly-visible data corruption with no error surfaced to the user or to monitoring.

**Fix.** `EditProfilePage.sanitize.ts`:
```ts
export const normalizeBioText = (text: string): string =>
  text.replace(/[\r\n]+/g, ' ');
```
Wired into `handleBioChange` (`useEditProfilePage.ts:131`) — runs **per keystroke**, before state.

**Current behaviour.** Enter inserts a single space, not a line break. A run of newlines (`a\n\n\n\nb`) collapses to exactly one space. `\r\n`, lone `\r` and lone `\n` are handled identically. The submitted value can never contain a newline.

---

## A2 — Name field accepts characters the server rejects · Full Name · **High** · ✅ Fixed

**What was wrong.** The name input accepted any character; its only constraint was `maxLength={30}`. The server's `Name()` rejects the value outright — `ErrInvalidNameChars` — on a single rune outside `letters | marks | space - ' ’ .`.

**What it caused.**

| User types | Result |
|---|---|
| `Agent007` | `ErrInvalidNameChars` — whole mutation fails |
| `ada_lovelace` | `ErrInvalidNameChars` |
| `Ada 😀 Lovelace` | `ErrInvalidNameChars` |

Three compounding problems: the failure arrived **after a network round-trip**; the error was **not field-specific**, so the user was told the save failed without being told which character was at fault; and because one bad rune fails the entire `updateUser` mutation, a simultaneous bio edit, country change or link addition was **discarded too**.

**Severity — High.** Hard, unexplained save failure on input a user would consider valid, that additionally destroys unrelated edits in the same submission.

**Fix.** `filterNameText` in `EditProfilePage.sanitize.ts`, a one-to-one mirror of the server's `isNameRune`:
```ts
export const filterNameText = (text: string): string =>
  text.replace(/[^\p{L}\p{M} '’.-]/gu, '');
```
| Server | Client |
|---|---|
| `unicode.IsLetter(r)` | `\p{L}` |
| `unicode.IsMark(r)` | `\p{M}` |
| `' '`, `'-'`, `'\''`, `'’'`, `'.'` | `' '’.-` |

**Current behaviour.** Invalid characters are stripped **as typed** — digits, `_`, `@`, `#`, emoji, HTML brackets, tabs, newlines, null bytes never appear in the field. International names are preserved and test-verified: `Mary-Jane Watson`, `O’Brien`, `J. R. R. Tolkien`, `José Álvarez`, `山田太郎`, `नमस्ते`. The filter does not truncate — length stays `maxLength`'s job. A regression test asserts the output always matches the server's accepted set, so future drift on either side fails CI.

---

## A3 — Social link input has no length cap · Social Link · **Medium** · ✅ Fixed

**What was wrong.** `PlatformLinkEditorView` rendered its `TextInput` with no `maxLength`. The 500 ceiling existed in `validateSocialLink.ts:113` and `updateUser.go:132`, but `MAX_LINK_LENGTH` was module-private so the component could not use it.

**What it caused.** An over-500 pasted URL was accepted into the field, then rejected with a generic "invalid" reason that never mentioned length — and a 600-character URL is not something you eyeball. Unbounded text in a single-line input also churns state on every keystroke.

**Severity — Medium.** Caught before persistence, so no corruption; the cost is a confusing, unactionable rejection on a field where over-long values arrive by ordinary paste.

**Fix.** Exported the existing constant (`validateSocialLink.ts:41`) and applied it at the input (`PlatformLinkEditorView.tsx:62`) — one shared constant, so input cap, validator and server limit cannot drift.

**Current behaviour.** The field stops accepting input at 500 characters. The over-length branch in `validateSocialLink` is now unreachable from the UI, so any "invalid" message shown is a genuine structural problem (bad hostname, multiple links, disallowed characters) and is therefore accurate.

---

## A4 — Bio rendered unclamped on the profile card · Bio (render) · **Medium** · ✅ Fixed

**What was wrong.** `UserInfoCard` rendered the bio as a bare `<Text>` with no `numberOfLines` — the only unclamped text node in a card where everything else was already `numberOfLines={1}` (lines 32, 48).

**What it caused.** 150 runes does not bound rendered height. A value made of narrow characters or many short tokens wraps into far more lines than the layout budgets for, growing the card and pushing stats, social chips and action buttons off-screen — amplified on narrow viewports. Because the card renders to **other users**, a crafted bio degrades the profile page for everyone who views it.

**Severity — Medium.** No data loss, but remotely triggerable (attacker sets their own bio, victim views the profile) and degrades a high-traffic screen for the viewer.

**Fix.** `numberOfLines={BIO_DISPLAY_MAX_LINES}` (6) + `ellipsizeMode="tail"`, chosen comfortably above what 150 runes occupies in normal text.

**Current behaviour.** At most 6 rendered lines regardless of content or viewport. Overflow truncates with a visible ellipsis rather than clipping. Card height is bounded; normal bios render unchanged.

### Part A test coverage

`EditProfilePage/hooks/__tests__/editProfileInputSanitize.test.ts` — **31 tests, all passing**:
```
Test Suites: 1 passed, 1 total
Tests:       31 passed, 31 total
```

---

# PART B — Open findings across the rest of the app

**None of these have a fix implemented.** "Current behaviour" below describes the defect as it ships today.

---

## B1 — Username rules diverge from the server in four independent ways · **High**

**Where:** `onboarding/pages/Username/hooks/useUsernameState.ts`, `profile/hooks/useEditProfileController.ts:47`, `EditProfilePage.constants.ts:24`

**Server truth** (`internal/domain/user/utils.go:61-138`): lowercase + trim; **min 4**; **max 41**; must **start with a letter** (`^[a-z]`); charset `^[a-z0-9-_.]+$` (hyphen allowed).

**Client:**

| Rule | Onboarding | Edit Profile | Server | Verdict |
|---|---|---|---|---|
| Min length | **3** | — | **4** | ✗ too permissive |
| Max length | **20** | **30** | **41** | ✗ three different numbers |
| Charset | `/^[a-zA-Z0-9_.]+$/` | same regex, duplicated | `[a-z0-9-_.]` | ✗ `-` wrongly blocked |
| Leading char | not checked | not checked | must be `a-z` | ✗ not enforced |
| Lowercasing | not applied | `_toLower` only | server lowercases | ✗ inconsistent |

**What it causes.**
1. A 3-character username passes client format validation and reaches the availability check, then fails at save — `len < 4`.
2. `1player`, `_matiks`, `.foo` all pass the client regex and fail server-side on the leading-letter rule, again only after submit.
3. A user who wants `ada-lovelace` is blocked by the client for a character the server explicitly permits.
4. A 25-character username can be set from Edit Profile but not from onboarding, and neither reaches the server's real 41-character allowance.
5. `USERNAME_REGEX` is **defined twice**, in two modules, with no shared constant — the two copies can drift independently, and already differ from the server.

**Severity — High.** Username is a signup-blocking field on the onboarding critical path. Rules 1–3 all produce a post-submit rejection for input the client itself said was fine, on the screen with the least user patience.

**Recommended fix.** One shared module exporting `USERNAME_MIN=4`, `USERNAME_MAX=41`, `USERNAME_REGEX=/^[a-z][a-z0-9\-_.]*$/`, consumed by both call sites, with `_toLower` applied at the input on both paths. Then delete both local copies.

---

## B2 — Forum thread creation has no input constraints at all · **High**

**Where:** `modules/clubs/pages/CreateForumPage/CreateForumPage.tsx`

Both `TextInput`s are raw: **no `maxLength`, no sanitization, no trim**.

```tsx
if (_isEmpty(forumTitle) || _isEmpty(forumDesc)) { … }   // no trim
createForum({ input: { clubId, description: forumDesc, title: forumTitle } });
```

**What it causes — four distinct defects in one screen:**

1. **Whitespace-only title/description passes the client guard.** `_isEmpty('   ')` is `false`, so the mutation fires; the server trims and answers `ErrEmpty` → `invalid thread title: text must not be empty`.
2. **Title is single-line server-side** (`NonEmptySingleLine`), so a pasted multi-line title has its newlines **deleted** — the exact word-join corruption fixed for bio in A1, still live here.
3. **No length cap** on either field (server: 200 runes title, 10000 runes / 200 lines content) → `ErrTooLong` / `ErrTooManyLines` after submit.
4. **The error copy is developer text, unlocalised:** `description: 'forumTitle or forumDesc Can not be empty'` — raw variable names shown to users, bypassing the `react-intl` layer every other screen uses.

**Severity — High.** Same silent-corruption class as A1, plus a validation guard that does not do what it claims, on a content-creation screen.

**Recommended fix.** `maxLength={200}` / `{10000}` on the two inputs; `normalizeBioText`-style newline collapse on the title; `_trim` before the empty check; replace the hardcoded toast with an intl message.

---

## B3 — Forum reply input is unbounded · **Medium**

**Where:** `modules/clubs/pages/ThreadDetailedInfoReplies/ThreadDetailedInfoReplies.tsx:145-160`

`multiline` with no `maxLength`. Server: `NonEmptyMultiLine(10000, 200)`.

**What it causes.** A reply over 10000 runes or over 200 lines is accepted by the UI, sent, and rejected with the raw wrapped error `invalid forum reply content: …`. The user loses the composed reply with no indication of which limit was hit.

The empty check here is correct (`_trim(replyText).length !== 0`) — worth noting, because it is the pattern B2 is missing.

**Severity — Medium.** Requires a long paste to trigger; loses composed content when it does.

**Recommended fix.** `maxLength={10000}` and a line-count guard, or a visible counter as the review form has.

---

## B4 — Chat message over 50 lines fails with a raw error · **Medium**

**Where:** `modules/chatV2/.../MessageListFooter.tsx:40,456-457`

```ts
const CHAR_LIMIT_ON_MESSAGE = 1000;   // matches the server's 1000 runes ✓
```

The **character** limit is correct. The **line** limit is absent: the server also enforces `messageMaxLines = 50`, and 51 newlines is only 51 characters — comfortably inside the 1000 the client allows.

Worse, the server only translates the length error into friendly copy:

```go
if errors.Is(err, textsanitize.ErrTooLong) {
    return nil, errors.New("message is too long")
}
return nil, fmt.Errorf("invalid message content: %w", err)   // ErrTooManyLines lands here
```

**What it causes.** Pasting a 60-line snippet (well under 1000 characters — code, an address block, a list) fails to send and surfaces `invalid message content: text exceeds maximum number of lines` — internal Go error text in a chat UI. The message is not sent and the user has no idea why.

**Severity — Medium.** Reachable through ordinary paste in the app's highest-traffic input, with an internals-leaking error.

**Recommended fix.** Cap line count client-side alongside the char limit, and add an `ErrTooManyLines` branch to the server's friendly-error mapping.

---

## B5 — Review form silently discards an over-limit paste · **Medium**

**Where:** `components/shared/AppReview/.../ReportExperienceForm.tsx:59-61`

```ts
if (text.length > 2000) return;   // rejects the whole change, not the overflow
setDescription(text);
```

**What it causes.** This guard rejects the **entire new value**, not the excess. Pasting a 2500-character description into an empty field does **nothing at all** — the field stays empty, no toast, no error, no counter movement. The user sees their paste vanish. (Typing past 2000 is correctly blocked, so the bug only shows on paste — which is exactly how a long description arrives.)

Secondary: the field is `multiline` with no line cap against the server's `userReviewDescMaxLines = 50`, and `text.length` counts UTF-16 units against the server's rune count.

**Severity — Medium.** Complete, unexplained input loss with zero feedback — the worst failure shape, because nothing tells the user anything happened.

**Recommended fix.** Replace the early return with `maxLength={2000}` on the input (which truncates instead of discarding), keeping the existing `{description.length}/2000` counter.

---

## B6 — Report comments are unbounded and unsanitized on **both** sides · **Medium**

**Where:** `modules/reportPost/pages/ReportPost/ReportPost.tsx:272-279`, `modules/reportUser/components/ReportForm/ReportForm.tsx:229`
**Server:** `internal/domain/userReport/report_user.go:107`

Client: `onChangeText={setAdditionalComments}` with **no `maxLength`**.
Server: `AdditionalComments: input.AdditionalComments` — stored raw. **No `textsanitize` call anywhere in the file**; the only validation is `len(input.ReasonKeys) == 0`.

**What it causes.** Arbitrary-length, arbitrary-content text — control characters, newlines, megabytes of paste — travels from the report form into `UserReport` storage and out to the **admin moderation surfaces** (`get_reports.go:60`, `adminReportedUsers.go:140`) with nothing in the path bounding or cleaning it. This is the only free-text field in the product with no sanitizer on either side.

The submit guards do trim correctly (`additionalComments.trim().length > 0`), so empty-submit is handled; size and content are not.

**Severity — Medium.** No corruption of user-visible content, but it is an unbounded write to a moderation datastore, rendered in an internal tool — the one place where unsanitized input is least likely to be defended downstream.

**Recommended fix.** `maxLength` on both client inputs, and a `textsanitize.OptionalMultiLine` call in `report_user.go` matching the other domains.

---

## B7 — Club description can breach the server line cap; error toast leaks the raw error · **Medium**

**Where:** `modules/clubs/constants/clubRegConstants.ts:127-142`, `modules/clubs/pages/CreateClub/CreateClub.tsx:85-89`

Client rules: name `minLength 5, maxLength 30`; description `minLength 10, maxLength 150`, `multiline`. Server: name 100 runes, description 500 runes / **20 lines**.

**What it causes.**
1. The client is stricter on *length* (no failure there), but enforces **no line count**. Pressing Enter 20 times costs 20 of the 150 allowed characters, so a description that passes every client rule can still trip `ErrTooManyLines`.
2. The failure path renders the raw error into user-facing copy:
   ```ts
   description: `Failed to Create Club ${_toString(e)}`
   ```
   The user is shown the stringified GraphQL/Go error, not a message.
3. These limits are validate-on-submit only — `Form/components/TextInput/TextInput.js` never applies a `maxLength` prop — so over-limit text is typed freely and rejected at the end.

**Severity — Medium.** Reachable, and the error handling turns a recoverable validation failure into an internals dump.

**Recommended fix.** Line-count rule in the form schema; replace the interpolated error with a fixed intl message.

---

## B8 — Two referral-code inputs, two different behaviours · **Low**

**Where:** `modules/referAndEarn/.../RedeemReferralCodePage.tsx:66` vs `modules/onboarding/pages/ReferralCode/components/ReferralInputView.tsx:25`

```ts
// Redeem page — sanitizes
const alphanumericText = _replace(text, /[^a-zA-Z0-9]/g, '');

// Onboarding — no filter, no maxLength
onChangeText={onReferralCodeChange}
```

**What it causes.** The same logical field accepts different input depending on where the user enters it. A code pasted with a trailing space or a zero-width character (both common from chat apps and screenshots) is cleaned on the redeem page and passed through verbatim during onboarding, where it fails the lookup and reads to the user as "invalid code".

**Severity — Low.** Recoverable, but it produces a wrong-looking failure on the referral acquisition path, and the correct implementation already exists 30 lines away in another module.

**Recommended fix.** Extract the redeem page's filter into a shared helper and use it on both.

---

## B9 — Group name limit is 25 on the client, 100 on the server · **Low**

**Where:** `chatV2/pages/FillGroupDetails/FillGroupDetails.tsx:320`, `chatV2/pages/EditGroupDetails/EditGroupDetails.tsx:262`

Both hardcode `maxLength={25}`; the server allows `groupNameMaxRunes = 100`.

**What it causes.** No failure — the client is stricter, so nothing is rejected server-side. The cost is a silently over-restrictive field (a four-word group name does not fit) and a magic number duplicated in two files with no link to the server constant it is supposed to shadow. `EditGroupDetails` does trim correctly before its empty check; `FillGroupDetails` does not.

**Severity — Low.** Product limitation and maintenance hazard, not a defect users hit as an error.

---

## B10 — Live chat and chatV2 disagree on message length · **Low**

**Where:** `modules/liveChat/components/LiveChat/LiveChat.tsx:22` (`MAX_MESSAGE_LENGTH = 500`) vs `MessageListFooter.tsx:40` (`CHAR_LIMIT_ON_MESSAGE = 1000`)

Two message composers, two limits, one server rule (1000). Live chat silently halves the allowance. Both constants are local; neither references a shared source.

**Severity — Low.** No failure; inconsistent product behaviour and duplicated magic numbers.

---

# What the codebase already gets right

Worth recording, because these are the patterns the open findings should be fixed *to* — all four filter at the keystroke **and** cap length:

| Component | Implementation |
|---|---|
| `OTPInput.tsx:57,126` | `_replace(text, /[^0-9]/g, '')` + `maxLength={length}` + `number-pad` |
| `RewardForm/PhoneInput.tsx:19,50` | `text.replace(/[^0-9]/g, '')` + `maxLength={PHONE_MAX_LENGTH}` |
| `RewardForm/PostalCodeInput.tsx:23,53` | `text.replace(/[^0-9]/g, '')` + `maxLength={PINCODE_LENGTH}` |
| `useStakeSelection.ts:87-99` | `text.replace(/\D/g, '')` + clamp to `maxStakeAmount` with a user-facing callback |

The stake input is the strongest example in the codebase: it sanitizes, clamps, **and** notifies the user why the value changed.

---

# Consolidated Findings Table

| ID | Area | Bug | Severity | Status |
|---|---|---|---|---|
| A1 | Profile — bio | Newlines deleted server-side → words concatenate | High | ✅ Fixed |
| A2 | Profile — name | Accepts chars the server rejects; whole mutation fails | High | ✅ Fixed |
| A3 | Profile — link | No length cap → misleading rejection | Medium | ✅ Fixed |
| A4 | Profile — bio render | Unclamped `Text` → card inflation for viewers | Medium | ✅ Fixed |
| B1 | Username | 4 divergences from server rules; regex duplicated | High | ❌ Open |
| B2 | Forum create | No caps, no trim, newline corruption, dev-copy error | High | ❌ Open |
| B3 | Forum reply | Unbounded input vs 10000/200 server limits | Medium | ❌ Open |
| B4 | Chat message | No line cap; >50 lines → raw Go error in chat UI | Medium | ❌ Open |
| B5 | Review form | Over-limit paste silently discarded entirely | Medium | ❌ Open |
| B6 | Report forms | Unbounded + unsanitized on client **and** server | Medium | ❌ Open |
| B7 | Club create | Line cap unenforced; raw error interpolated into toast | Medium | ❌ Open |
| B8 | Referral code | Two inputs, only one sanitizes | Low | ❌ Open |
| B9 | Group name | 25 client vs 100 server; duplicated magic number | Low | ❌ Open |
| B10 | Live chat | 500 vs 1000 char limit for the same server rule | Low | ❌ Open |

---

# Cross-Cutting Root Causes

Every finding above traces to one of four patterns:

1. **Limits are magic numbers, duplicated per call site.** `25`, `30`, `20`, `500`, `1000`, `2000` are hardcoded in the components that use them. Nothing links them to the server constants they shadow, so they drift silently (B1, B9, B10) and get forgotten entirely (B2, B3, B6).
2. **`multiline` is used as a layout choice, not a data contract.** Five inputs are `multiline` against fields the server treats as single-line or line-capped. A1 was the corrupting case; B2 and B4 are the same shape, unfixed.
3. **Emptiness is checked without trimming.** `_isEmpty(value)` on an untrimmed string is `false` for `'   '`, so whitespace-only values pass the client guard and fail server-side as `ErrEmpty` (B2; correct in B3, `EditGroupDetails`, `AddInstitutionForm`).
4. **Sanitize errors are surfaced raw.** Three screens interpolate a stringified error into user copy or show unmapped Go error text (B2, B4, B7).

## Recommended structural fix

A single `src/core/validation/` module exporting the server's limits and rules as named constants, plus the sanitizers already written for the profile fix (`normalizeBioText`, `filterNameText`), consumed everywhere. That converts all ten open findings from "each screen forgot something different" into one shared contract with one place to keep it in sync — which is what A1–A4 did for the profile module and what the 31-test suite already locks down.

---

# Known Gaps in the Part A Fixes

Documented in the test suite rather than fixed, because each changes behaviour in ways worth deciding deliberately:

1. **Non-newline control characters in the bio are not stripped client-side.** The server's `SingleLine` deletes every `unicode.IsControl` rune; `normalizeBioText` handles only `\r`/`\n`. A pasted tab or null byte survives on the client and is silently removed by the server.
2. **Whitespace-only name and bio are not caught client-side.** Space is an allowed name rune, so `filterNameText('   ')` returns `'   '`; the server trims and answers `ErrEmpty`. Neither sanitizer trims. Same root cause as B2.
3. **Length units differ.** React Native's `maxLength` counts UTF-16 code units; the server counts runes. An emoji costs 2 client characters and 1 server rune, so the client is *stricter* — it can only under-allow, never cause a failed save.
4. **The username field was out of scope for the branch** — it has its own server-side availability path. That gap is B1.

---

# Branch State

`feat/input_validation` carries one commit against `origin/dev` (`5fa07007d`) plus uncommitted work:

- **Uncommitted, in scope:** extraction of the two sanitizers into `EditProfilePage.sanitize.ts` and the 31-test suite. A refactor of the committed fix to make it testable — no behaviour change.
- **Uncommitted, unrelated:** a `useLayoutEffect` change to `guardRef` synchronisation in `useEditProfilePage.ts` (exit-guard staleness on discarded renders), plus incidental `.env.development` and `ios/Podfile.lock` edits. Not part of the input-validation work; should be separated before review.
