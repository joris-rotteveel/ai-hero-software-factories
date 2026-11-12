# Knock capabilities for Cadence (in-app feed + SMS)

Research for issue #2, part of the map issue #1 "Wayfinder: In-app Notifications with Knock". Researched 2026-11-12.

Cadence stack: React Router 8.3 (framework mode, SSR), React 19, Tailwind 4 + shadcn, SQLite/Drizzle, Node server. Email goes through Postmark in `app/lib/mailer.server.ts`. When `POSTMARK_SERVER_TOKEN` is not set, that file writes each email to `.outbox/` instead of sending it.

Package versions checked (npm registry, 2026-11-12): `@knocklabs/node` 1.41.1, `@knocklabs/react` 0.15.1, `@knocklabs/react-core` 0.17.1, `@knocklabs/client` 0.23.1.

## TL;DR

- **Free tier ("Developer")**: 10,000 messages a month. A message is one delivery to one user on one channel, so in-app feed items count too. Environments, workflows, users and team members are unlimited. Logs are kept for 30 days. Knock branding stays on.
- **SMS**: you bring your own provider (Twilio and 10 others). There is no Knock-managed SMS. You pay Twilio separately. Phone numbers must be E.164 (`+15551234567`).
- **Test story**: there is no local emulator. You get a separate **development environment** with its own keys, plus **sandbox mode**: per channel, or per trigger with `settings.sandbox_mode: true`. In sandbox mode messages are generated and logged but never reach the provider. Local dev and tests with no key still need a stub of our own, like the `.outbox/` transport.
- **React**: `@knocklabs/react` has prebuilt components (bell, popover, feed) and headless hooks. The bundle starts with `"use client"` and connects over a websocket, so treat it as client-only under React Router SSR. The prebuilt UI is styled with its own CSS (`rnf-` classes and `--rnf-*` variables), not Tailwind. For a shadcn look, use the hooks and build our own UI.

## 1. In-app feed: prebuilt components vs headless hooks

**What's there**
- Prebuilt components: `NotificationIconButton` (bell with an unread/unseen badge), `NotificationFeedPopover`, and `NotificationFeed` (full-page list). They need `KnockProvider` (props `apiKey`, `user={{ id }}`) and `KnockFeedProvider` (props `feedId` = the in-app channel ID, `userToken`, `colorMode`). [docs.knock.app/in-app-ui/react/feed](https://docs.knock.app/in-app-ui/react/feed), [overview](https://docs.knock.app/in-app-ui/react/overview)
- Headless hooks: `useKnockClient`, `useNotifications(client, channelId, { mode: "rich" })` and `useNotificationStore(feedClient)`, which returns `items` and `metadata.unread_count` / `unseen_count`. With hooks you fetch the feed yourself (`feedClient.fetch()` in an effect) and mark items seen yourself (`markAllAsSeen()` / `markAsSeen(item)`). If you skip marking, the badge never clears. [headless/feed](https://docs.knock.app/in-app-ui/react/headless/feed)
- Real-time updates come over a socket. Options can sync tabs through BroadcastChannel and auto-disconnect inactive tabs. [feed](https://docs.knock.app/in-app-ui/react/feed)

**SSR**
- Knock's docs say nothing about SSR.
- `dist/esm/index.mjs` in `@knocklabs/react` starts with `"use client"` (checked in the 0.15.1 tarball). The package also ships optional `./next` and `./tanstack` entry points, but none for React Router.
- Peer dependencies: `react` 17–19. `next` and `@tanstack/react-router` are optional peers.
- **For Cadence**: the feed state is per-user and live, so there is nothing useful to render on the server. Mount `KnockProvider` and the feed only on the client, for example behind a "hydrated" flag or a lazy component. The loader can supply the user ID and signed token. The inbox prototype ticket should confirm this works.

**Styling with Tailwind/shadcn**
- The prebuilt UI needs `import "@knocklabs/react/dist/index.css"`. You theme it by overriding `--rnf-*` CSS variables or by replacing the stylesheet (all classes start with `rnf-`). `renderItem` lets you supply a custom cell. [feed](https://docs.knock.app/in-app-ui/react/feed)
- `@knocklabs/react` pulls in Radix, Knock's "Telegraph" design-system packages, Popper, and `lucide-react@^0.544` (Cadence uses `^0.563`) (from the tarball's `package.json`).
- **For Cadence**: use the headless hooks and render with our shadcn components (Popover, Badge, ScrollArea). The UI then matches the rest of the app with no second design system. The prebuilt popover is fine for a quick spike.

## 2. Identifying users

- **User sync**: a user must exist in Knock before Knock can notify them. Create or update the user with `client.users.update(userId, { name, email, phone_number, timezone, ...props })`, or identify them inline in the trigger by passing full recipient objects instead of IDs. The `id` must be our permanent primary key, not an email address. It cannot change, can be up to 256 characters, and cannot contain `/` or `#`. Extra properties are deep-merged on each update. To remove a property, set it to `null`. `users.delete` removes a user. Old feed items then show a "missing user" marker. [concepts/users](https://docs.knock.app/concepts/users), [triggering via API](https://docs.knock.app/send-notifications/triggering-workflows/api)
- **Signed user tokens** ("enhanced security mode"): this mode is off by default and Knock says to "Enable enhanced security mode before you go live." Without it, anyone with the public key and a user ID can read that user's feed. The token is an RS256 JWT signed with an environment's signing key (stored by convention in `KNOCK_SIGNING_KEY`). Call `signUserToken(userId, { expiresInSeconds })` from `@knocklabs/node`. The default lifetime is 1 hour. Refresh the token with the client's `onUserTokenExpiring` callback. Optional grants can scope a token to tenants or objects. [security-and-authentication](https://docs.knock.app/in-app-ui/security-and-authentication), `@knocklabs/node` README
- **For Cadence**: sign the token in the root loader (or a resource route) for the logged-in user, and pass `userToken` to `KnockFeedProvider`. Sync the user to Knock when they sign up or change their name, email or phone, or identify them inline on every trigger. Inline identification is simplest because it needs no extra write path.

## 3. Workflows: triggering, idempotency, batching

- **Trigger from Node**:
  ```ts
  import Knock from "@knocklabs/node";
  const knock = new Knock({ apiKey: process.env.KNOCK_API_KEY });
  await knock.workflows.trigger("refund-requested", {
    recipients: [{ id: "user_1", name: "…" }],  // IDs or inline-identified objects
    actor: "user_2",
    data: { courseTitle: "…" },
    tenant: "team_7",            // optional
    cancellation_key: "…",       // optional, used to cancel the run later
  }, { idempotencyKey: "refund-requested:42" });
  ```
  The call returns `workflow_run_id`. Runs are **asynchronous**, so a 200 response means "accepted", not "delivered". [API ref: trigger](https://docs.knock.app/api-reference/workflows/trigger), `@knocklabs/node` README
- **Idempotency**: send the `Idempotency-Key` header. Only `POST /workflows/:key/trigger` honours it. Keys are kept for 24 hours and can be up to 255 characters. A retry with the same key gets back the original response, with `idempotent-replayed: true`. If the parameters differ from the first request, Knock returns an error. Only 2xx responses are cached. [idempotent-requests](https://docs.knock.app/api-reference/overview/idempotent-requests)
  - The Node SDK accepts an `idempotencyKey` request option. If you don't pass one, it generates a key for each call, so its own automatic retries (`maxRetries` defaults to 2) are safe. Read from `src/client.ts` and `src/internal/request-options.ts` in the 1.41.1 tarball.
  - **For Cadence**: derive the key from the domain event, e.g. `refund-decided:<refundRequestId>`. A retried action then can't notify twice.
- **Batching**: this is a workflow step, configured in the dashboard, not in code. It groups triggers per recipient, plus an optional batch key (Liquid, e.g. `{{data.courseId}}`). The window can be fixed, dynamic or sliding, and can stay open at most 90 days. You can cap a batch at 2–1,000 items. Templates see up to 10 activities, plus `total_activities`. Rendering more than 10 is Enterprise-only. [batch function](https://docs.knock.app/designing-workflows/batch-function)
- Other trigger settings: `settings.sandbox_mode` and `settings.skip_delay` (see §7). A trigger step can have conditions. You can set a workflow to run "once per recipient". Triggering an inactive workflow returns `workflow_inactive`. [triggering-workflows](https://docs.knock.app/send-notifications/triggering-workflows)

## 4. Preferences API

- Each user has a `PreferenceSet` (normally `"default"`) with these keys: `channel_types` (`in_app_feed`, `sms`, `email`, …), `channels` (by UUID), `categories`, `workflows` (each can have its own `channel_types`), and conditions. [preferences/overview](https://docs.knock.app/preferences/overview)
- **Opt-out by default**: a preference that isn't set counts as `true`. A message sends only if every preference that applies is `true`. A `false` category blocks the message even if the channel is `true`. A workflow can override preferences, for things that must always be sent.
- Defaults are layered: environment default (set in the dashboard or the Management API), then tenant default, then the user's own set.
- Server: `client.users.setPreferences(userId, "default", {...})` overwrites the set by default. Pass `__persistence_strategy__: "merge"` for a partial update. `bulkSetPreferences` updates up to 1,000 users. You can also pass preferences inline on a trigger.
- Client: `knockClient.user.getPreferences()` / `setPreferences(set)` from `@knocklabs/client`. The docs show a hand-rolled `PreferenceCenter`, not a drop-in component. [in-app-ui/react/preferences](https://docs.knock.app/in-app-ui/react/preferences)
- **For Cadence**: SMS should be **opt-in**, but Knock is opt-out by default. Set the environment default to `channel_types.sms: false` (or false for each urgent workflow) and let users turn it on. Even then, SMS can only reach users who have a `phone_number`. The preferences UI can be shadcn checkboxes that write `workflows.<key>.channel_types`, either from the client or through our own action.

## 5. SMS

- **Providers**: Africa's Talking, Amazon SNS, Mailersend, MessageBird, Modica, Plivo, Sinch, Sinch MessageMedia, Telnyx, Twilio and Vonage. Knock has no managed SMS: you always connect your own provider account. [integrations/sms/overview](https://docs.knock.app/integrations/sms/overview)
- **Twilio setup**: in Knock, add an SMS channel with the Twilio Account SID and Auth Token. Choose how to send: a phone number, a short code, or a Messaging Service SID. Configure this per environment, because credentials are not copied between environments. For delivery tracking, enable Knock's incoming webhook and paste its URL as the Status Callback in Twilio. Twilio statuses map to Knock's: delivered → `delivered`, failed → `bounced`, undelivered → `undelivered`. A Twilio trial account can only text numbers you have verified. [integrations/sms/twilio](https://docs.knock.app/integrations/sms/twilio), [environments](https://docs.knock.app/concepts/environments)
- **Phone format**: the user property `phone_number` must be in **E.164** format, e.g. `+15551234567`. [concepts/users](https://docs.knock.app/concepts/users) A channel step checks that the recipient has the data the channel needs before it renders the message. [channel-step](https://docs.knock.app/send-notifications/designing-workflows/channel-step)
- **Costs**:
  - Knock does not charge extra for SMS. Each SMS counts as one message toward the plan limit. [knock.app/pricing](https://knock.app/pricing)
  - Twilio US prices: $0.0083 per outbound segment, plus a carrier fee of about $0.0025–$0.005 per segment. A long-code number costs $1.15 a month, and a toll-free number $2.15 a month. US A2P 10DLC (business texting on local numbers) also has registration fees. [twilio.com/en-us/sms/pricing/us](https://www.twilio.com/en-us/sms/pricing/us)
  - Other countries have different prices and sender rules. Cadence records the buyer's country (ADR 0009), so check Twilio's pricing page for each target country.
- **For Cadence**: add a nullable `phone` column in E.164 format, plus a consent record. Sync the number to Knock as `phone_number`. Put a step condition on the SMS step so it only runs for urgent types (or keep SMS in separate urgent workflows).

## 6. Free-tier limits ("Developer" plan)

[knock.app/pricing](https://knock.app/pricing):
- 10,000 messages a month. A message is one delivery to one user on one channel, so in-app feed items count. Failed or bounced messages don't count.
- Unlimited environments, team members, stored profiles, workflows, broadcasts and channels. Multi-tenancy included.
- 30-day log retention. Knock branding cannot be removed. No priority support and no SAML.
- 500 guide recipients and 500 agent credits a month (not relevant to Cadence).
- The next tier, Starter, costs $250 a month. It includes 50,000 messages, then $0.005 per message.

## 7. Local/test story

- **Environments**: production always exists. Turning on environments adds **development** (this can't be undone), and you can add others such as staging. Each environment has its own data, API keys and channel credentials. Workflows are version-controlled and promoted from development upward. [concepts/environments](https://docs.knock.app/concepts/environments)
- **Sandbox mode**: each channel has a sandbox toggle per environment. The trigger option `settings.sandbox_mode: true` forces sandbox for every channel in that run. Either way, "messages are generated but not delivered to the underlying providers". The dashboard test runner has the same option, plus `skip_delay`. [triggering via API](https://docs.knock.app/send-notifications/triggering-workflows/api), [testing-workflows](https://docs.knock.app/send-notifications/testing-workflows)
- There is **no local emulator or offline mode**. Every Knock call goes to `api.knock.app`. The SDK accepts `baseURL` / `KNOCK_BASE_URL`, which could point at a fake server.
- **For Cadence**: copy the `mailer.server.ts` pattern. Write a `NotificationTransport` interface with two implementations:
  1. a Knock transport, used when `KNOCK_API_KEY` is set;
  2. an outbox transport that writes each trigger (workflow key, recipients, data, idempotency key) to `.outbox/notifications/` as JSON.
  Tests and keyless local dev use (2) and never send an SMS. Shared dev or staging uses Knock's development environment with the SMS channel in sandbox mode, or with no SMS channel configured. With the outbox stub, the in-app feed UI shows nothing locally unless we also add a local feed stub, or developers use a Knock development key. The prototype ticket should decide which.

## Open questions for later tickets

- Should the client feed render only after hydration, or is SSR of the bell possible? Confirm in the inbox prototype.
- Should Notifications be triggered in the action or after the database commit? Idempotency keys make retries safe either way. Knock runs workflows asynchronously, so a 2xx response only means the trigger was accepted.
- For team recipients, should we use Knock tenants or objects/subscriptions? Not researched here.
