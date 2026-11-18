# Community contributor and commit attribution policy v1

**Status: proposed for DevLoot maintainer review; not adopted or implemented.** This decision record is intended to resolve LootTown tracking issue #3 and guide a future DevLoot public projection. It does not describe a deployed endpoint or prove that the current provider can supply complete history.

## Objective and boundary

Define one stable contributor identity per DevLoot community and a defensible count of that person's distinct attributed commits. The result is a participation signal for LootTown's approved avatar bands, not a measure of quality, value, employment, or online presence. One GitHub ID represents one GitHub account; GitHub cannot prove that one account equals one real person or that multiple accounts belong to one person.

This policy concerns publicly linked repositories only. It does not grant additional GitHub permissions, authorize direct browser access, include bounty/support/ownership events as commits, or define venue-size thresholds.

## Proposed policy decisions

### Community and repository scope

- A community is one DevLoot `Project`, identified by its stable database ID. The repository set is the set linked to that Project when a snapshot is created; DevLoot currently caps this set at 10.
- Include a repository only when it is public at snapshot time. If a linked repository is private, inaccessible, deleted, or cannot be checked for visibility, do not expose its identity or contributors. Mark community coverage partial because the public set no longer proves full community coverage.
- Record the stable DevLoot repository ID for each source. Do not identify a repository by a mutable URL or owner/name slug alone.
- A public fork or mirror explicitly linked to the community is in scope. Identical commit SHAs across linked repositories are deduplicated within the community. A repository linked to two communities contributes independently to each; deduplication never crosses community IDs.
- Each snapshot is scoped to a precise repository-set version. Adding a repository makes coverage partial until it is fully scanned. Removing a repository does not erase previously accepted lifetime attribution: a SHA already recorded for the community remains recorded, preserving permanent avatar bands. A later re-link does not count that SHA again.

### Contributor identity and eligibility

- The identity key is the numeric GitHub user ID, scoped by community. A username or email is never an identity key. Renames update the display handle for future snapshots without changing the ID; retain the last known handle when a deleted account no longer has a current public handle.
- A person is eligible when at least one in-scope commit has been attributed to a resolvable GitHub user ID that is not classified as a bot. Repository owner, supporter, and bounty-winner roles alone do not make a person eligible.
- Exclude an author when GitHub identifies the account as a bot (`type=Bot` or an explicit bot marker). Do not classify an account as a bot based only on commit volume, organization membership, or an unverified name pattern. If bot status cannot be resolved, coverage is partial; do not silently include or exclude that author's contribution as known.
- An unlinked author (no stable GitHub ID) is not returned as a person and is not assigned to a guessed account by email, handle, or commit message. Its presence makes coverage partial because eligible people and counts may be missing.
- A deleted account with a previously resolved stable ID remains the same contributor for historical attribution. If history cannot be linked to a stable ID, it is unresolved and makes coverage partial.
- A commit's co-authors count only when provider data explicitly resolves them to stable GitHub IDs. Do not infer identity from email addresses or parse arbitrary commit text as proof. Each resolved eligible co-author receives one attribution for that SHA. An unresolved claimed co-author makes coverage partial.

### Commit attribution and deduplication

- The attribution window is lifetime history available from the provider for in-scope public repositories, with no rolling cutoff. Attribution is based on commits exposed by the provider's documented history source; it must not be inferred from the current `RepositoryContributorSync.contributions` aggregate.
- A commit is eligible when it is part of the provider's documented repository history and has a stable SHA and at least one resolved eligible author. Exclude bounty records, pull-request counts, issue activity, and repository-level contribution totals.
- Count a SHA at most once per `(communityId, githubId)`. The same SHA seen in two linked repositories adds one commit for that person, not two. One SHA with two resolved eligible co-authors contributes one to each co-author's personal count; therefore the sum of per-person counts is not a count of globally unique community SHAs.
- Persist a community-scoped seen-SHA attribution ledger so overlap, unlinks, re-links, mirrors, and later syncs cannot inflate counts or remove already awarded bands. For each SHA, freeze the resolved author-ID set from its first successful canonical provider observation; duplicate observations add repository provenance but do not rewrite authors or add counts. Conflicting metadata for the same SHA must be surfaced for review and make coverage partial until resolved.
- Preserve source provenance for each attribution: community ID, repository ID(s) where the SHA was observed, SHA, resolved GitHub ID(s), provider observation time, and policy version. Do not expose the SHA ledger in the public response by default; use it for deduplication and audit only.

### Coverage, freshness, and unknown values

Coverage describes whether the projection can make a complete claim for the current community, not whether a provider request merely returned successfully.

| State | Meaning | Person counts |
| --- | --- | --- |
| Complete and fresh | Every currently linked public repository is in the snapshot; its full in-scope history is scanned; identities and bot status are resolved; snapshot is within the freshness target. | Exact nonnegative counts. A truly empty, fully scanned community has no people and a known zero-person result. |
| Partial | A repository is missing, newly linked, inaccessible, history is truncated, sync failed for any repository, an author/co-author is unresolved, or bot classification is uncertain. | `null` for every returned person's community count; do not emit a partial lower bound as exact. |
| Stale | Last complete snapshot is older than the freshness limit. Its timestamp and prior provenance may be retained, but it is not current coverage. | `null` for current counts. A stale result must be visibly distinguishable from a fresh empty community. |
| Unavailable | No usable snapshot can be produced, such as provider outage or no successful baseline. | No people/counts; report unavailable explicitly. Never encode this as an empty complete community. |

Proposed operating target for maintainer review: attempt a full or incremental refresh at least every 24 hours and mark a snapshot stale after 48 hours without a successful complete refresh. These are starting targets, not verified provider limits. A failed refresh must not advance `lastSyncedAt` or erase the last successful provenance. The API design must expose freshness/staleness separately from source completeness; proposed `VenueV1` has only `coverage.complete` and `lastSyncedAt`, so an upstream implementation may need to extend or migrate that contract before representing all four states without ambiguity.

An empty result is a known zero only after a complete scan of the full linked public repository set finds no eligible attributed people. No linked repositories, an uninitialized sync, a failed provider call, a partially scanned repository, or an empty current contributor cache is not evidence of zero. Under this proposal, `communityCommitCount` is null whenever coverage is partial, stale, or unavailable.

### Retention and public fields

- Public output may include the stable community ID and name; stable IDs and current/last-known handles for eligible GitHub accounts; eligibility; nullable community commit count; policy version; coverage/freshness state; and stable IDs for the public repositories included in that snapshot.
- Do not return private repository IDs, private affiliations, access tokens, wallet addresses/IDs, commit author emails, raw commit messages, or internal provider credentials. Do not use wallet identity, DevLoot account ID, or email to merge GitHub identities.
- Retain only fields needed for the deduplication/provenance ledger and public projection. Proposed minimum ledger: community ID, SHA, resolved GitHub ID set, source repository IDs, first-observed timestamp, and policy version. Do not persist raw commit payloads or messages for this feature.
- Proposed retention: keep deduplication keys while the community venue remains enabled so all-time counts and permanent bands remain stable; on venue removal, delete the ledger within 30 days unless DevLoot's existing retention obligations require a shorter period. DevLoot maintainers and privacy review must approve or replace this proposal before implementation.
- Public contribution data can reveal a person's association with a community even when the underlying repository is public. The product should disclose that public GitHub contribution metadata is used for the venue projection and provide a removal/correction path consistent with DevLoot policy. Do not claim public source data has no privacy impact.

## Source mapping and implementation boundary

The current mapping below is transcribed from [the inspected wire-contract record](contracts.md), whose inspection basis is DevLoot commit `8dd22dd89f265024dfff5d76f938a2774cc0f4ec` (read 2026-11-03). The DevLoot source checkout was not available in this workspace during this policy draft; paths and behavior must be rechecked against upstream source before implementation.

| DevLoot model/source | Existing meaning | Policy use and gap |
| --- | --- | --- |
| `Project` ↔ `Repository`, `server/prisma/schema.prisma:42-101` | Many-to-many community/repository linkage. | Use stable Project and Repository IDs and snapshot the linked repository set. |
| `AddCommunityRepositoryUseCase`, `server/src/modules/community/application/add-community-repository.usecase.ts:26-36` | Caps a community at 10 linked repositories. | Expected maximum input size; still require a consistent snapshot of all links. |
| Contributor index and sync metadata, `server/prisma/schema.prisma:992-1017` | Stores GitHub ID, per-repository `contributions`, generation, and sync metadata. | Candidate source for identity and freshness, but `contributions` is not a unique-SHA ledger and cannot be summed into community counts. |
| Participation query, `server/src/modules/projects/infrastructure/prisma-project-participation.ts:11-76` | Uses positive per-repository contributions and groups rows by project and GitHub ID. | Useful for current navigation/identity precedent; does not establish bots, complete commit history, or SHA overlap. |
| Participation DTO/controller, `participation-response.dto.ts:1-43`; `project-participation.controller.ts:1-34` | Exposes `person.githubId`, roles, repository affiliations, and freshness `{complete,lastSyncedAt}`. `isVerified` means DevLoot sign-in, not human verification. | Do not treat current `complete` as full Git history; public projection should omit private affiliations and auth-related identity fields. |

No existing field listed above proves commit-SHA uniqueness, historical coverage, author/co-author resolution, or bot classification. A future implementation needs provider feasibility review and an explicit SHA-level data source/ledger; this draft does not claim the current contributor sync can provide it.

## Synthetic reference cases

The deterministic test vectors live in [`fixtures/contributor-attribution-v1.json`](fixtures/contributor-attribution-v1.json). They cover a shared contributor, duplicate SHA across two public repositories, co-authorship, bot exclusion, unresolved authors, stale/partial coverage, an empty-but-complete community, and a 10-repository community. All IDs and SHAs are fabricated placeholders. They are policy examples, not DevLoot data, GitHub fixtures, or an API response.

Expected rules for the fixture:

1. In `complete-overlap`, the shared person is one community identity and the repeated SHA is counted once.
2. In `complete-coauthor`, each resolved eligible co-author receives one count for the shared SHA; the community-wide sum is two person-attributions for one SHA.
3. In `partial-unlinked`, the bot is excluded, the unlinked author is not guessed, and every returned person's count is null because the community is partial.
4. In `empty-complete`, all repositories in the snapshot were successfully scanned; this is a known empty community. In `empty-without-baseline-is-not-zero`, no scan is not evidence of zero.
5. In `ten-repositories`, coverage is complete only if all ten linked public repositories were scanned under one snapshot and each source completed.

## Decisions still requiring maintainer approval

- Provider/history feasibility and rate-limit budget for lifetime SHA enumeration and refreshes.
- Whether the proposed 24-hour refresh / 48-hour stale thresholds are operationally feasible.
- Retention period and deletion timing for the SHA-to-GitHub-ID ledger.
- Exact provider signals for bot and co-author resolution, plus the conflict-review process.
- Whether current and last-known GitHub handles may be publicly returned, and the correction/removal mechanism.
- Whether public linked forks/mirrors are included as stated, and how a Project repository-set change is versioned.
- The public contract fields needed to distinguish complete, partial, stale, unavailable, and truly empty states.

Until these decisions are approved and checked against DevLoot source, this document is a proposal, not an adopted product policy. No GitHub permission expansion, DevLoot API, or public endpoint is implied.
