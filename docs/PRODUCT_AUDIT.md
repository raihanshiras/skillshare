# SkillSwap Product and Codebase Audit
Audit date: 2026-10-13
Branch: audit/skillswap-foundation-fixes
Base: main

## Summary

Continue from the existing Next.js + React + TypeScript + Supabase application. Do not rewrite the frontend. Prioritize schema correctness and an end-to-end proof of the core loop before adding major features.

## Existing surface area (confirmed in repository)

- Public product landing page.
- Supabase auth pages and callback flow.
- Three screens in signup/onboarding (account draft, profile details, skills offered).
- People discovery/search and profile cards.
- Public and private profile-related database tables are intended.
- Skill offers/needs are represented as user-to-skill links.
- Swap request UI with incoming, outgoing, and history tabs.
- Messaging UI and SQL scripts for conversations/messages.
- Profile editing, avatar upload, and offered skill management.
- Multiple variants of message components and older CSS files remain in the tree.

Repository has multiple overlapping schema/setup files:
- `supabase/schema.sql`
- `SUPABASE_SCHEMA.sql`
- `SUPABASE_ONBOARDING.sql`
- `supabase/messaging-v2.sql`
- `supabase/messaging-permissions-fix.sql`

These should not be treated as safely interchangeable. The current application source uses `profiles.avatar_url`, `user_skills.skill_type`, `skills.name_normalized`, `swap_requests`, `conversations`, and `messages`. A full migration audit must include all of those dependencies.

## Confirmed issues / risks

1. **Inconsistent onboarding metadata contracts.** The onboarding client sends `profile_full_name`, `profile_bio`, `private_phone_number`, and `private_date_of_birth`. The root onboarding SQL trigger reads `full_name`, `bio`, `phone_number`, and `date_of_birth`. The `supabase/schema.sql` trigger does match the newer `profile_*` / `private_*` metadata names.
2. **Different `user_skills` column names.** One SQL file uses `skill_type`; another uses `type`. The application components query `skill_type`. Do not apply the older SQL as if it were compatible.
3. **Avatar column inconsistency.** The client reads/writes `profiles.avatar_url`. The schema at `supabase/schema.sql` does not declare it. The root onboarding SQL does declare it.
4. **Privacy vs discovery.** Discovery queries skill links across users. The current `supabase/schema.sql` selection policy originally restricted `user_skills` to the current user only. The audit branch changes that policy to allow authenticated campus discovery; mutation policies remain self-only. Private contact data remains in `profile_private` with self-only policies.
5. **Swap request schema not found in the inspected canonical SQL.** Client calls `create_swap_request` and `respond_to_swap_request` and queries `swap_requests`; the inspected `supabase/schema.sql` file does not define them. The messaging SQL assumes `swap_requests` and these functions already exist.
6. **Avatar storage assumptions.** Onboarding and profile components use a Storage bucket named `avatars`. The setup SQL reviewed does not create the bucket or its policies.
7. **Auth onboarding uses sessionStorage for an in-progress password.** This avoids sending a password to the database before final signup, but it is still sensitive browser state. Keep its lifetime short, clear it on every successful or abandoned flow, never log it, and do not persist it to localStorage.

## First implementation sequence

1. Determine the actual deployed Supabase schema and whether current data must be preserved.
2. Consolidate schema scripts into a non-destructive, forward-only migration path (never reset a live database without an explicit backup and approval).
3. Align onboarding trigger metadata with the client contract, preserve `avatar_url`, and verify profile trigger constraints.
4. Add/verify the swap-request table and secure RPC functions:
   - caller must be authenticated;
   - a user cannot request themself;
   - prevent duplicate pending/accepted requests;
   - only recipient can accept/decline;
   - only sender can cancel;
   - only participants can read requests;
   - acceptance should create/retrieve a conversation transactionally.
5. Ensure RLS and grants support campus profile and skill discovery without exposing private phone/date-of-birth fields.
6. Ensure avatar bucket setup and storage policies are documented and least-privilege.
7. Test the complete journey using two test accounts: signup, confirmation, login, discover skills, request swap, accept, message in realtime, edit profile, add/remove skill, sign out.
8. Only then add session scheduling, feedback, and the college dashboard.

## MVP non-goals until the core loop is stable

- Native mobile application.
- Custom video infrastructure.
- AI matching.
- Payments or paid tutoring.
- Heavy gamification.
- Advanced college analytics.

## Validation note

This is a static audit of repository files. It is not a live Supabase inspection, production health check, or a successful build/test report. No database has been modified by this audit.
