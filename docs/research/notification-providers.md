# Which notification providers fit Cadence?

Research for [issue #2](https://github.com/ibrahim-envelio/cohort-005-project/issues/2), part of the map [issue #1](https://github.com/ibrahim-envelio/cohort-005-project/issues/1).
Sources were read on 2026-11-10.
Prices and limits change often, so recheck them before signing up.

## Recommendation

**1. Novu**\
It is the only provider whose free plan includes EU data residency.
Its paid plan starts lowest, at $30 per month.
Its React SDK has both a ready-made Inbox and headless hooks.

**2. Courier**\
It has a 10,000-send free plan and pay-as-you-go pricing after it.
Its EU datacenter covers the in-app Inbox, but only on the paid Business plan.
Its browser token is scoped to one user and can be made short-lived.

**Close third: Knock**\
Knock has the most fine-grained browser token and a mature React SDK.
It has no EU hosting today, and its first paid plan costs $250 per month.

No shortlisted provider fails a must-have.
The differences are in EU hosting, price after the free tier, and SDK details.

## What Cadence needs

Cadence is a React Router 8 app on React 19.
It has server-side loaders and actions, so the server can call a provider's API and sign a token for the browser.

The must-haves from the map are:

- a free tier for development
- a React SDK or plain API usable from React Router 8
- real-time delivery to the browser
- read/unread state handled by the provider
- a path to add email later without a migration

EU hosting is a plus, not a must-have.

Every provider below is multi-channel, so email can be added later as another channel on the same workflow.

## Terms used below

- **Workflow / trigger**: a server call that says "this event happened to this user". The provider then delivers it to each channel.
- **Idempotency key**: a unique string sent with a request. If the same request is retried with the same key, the provider does not send the notification twice.
- **JWT (JSON Web Token)**: a signed token. Cadence's server signs it, and the browser sends it to the provider to prove who the user is.
- **HMAC (hash-based message authentication code)**: a hash of the user id made with a server-only secret. It proves the browser was given that user id by the server.
- **WebSocket**: a long-lived browser connection that the provider uses to push new notifications.

## Novu

**Free tier and pricing**\
The Free plan is $0 with 10,000 workflow runs per month.
In-app is one of its included channels.
Pro starts at $30 per month with 30,000 runs.
Team starts at $250 per month with 250,000 runs.
Extra runs on Pro and Team cost $1.20 per 1,000.
The pricing page does not say what happens past the Free limit.
Source: https://novu.co/pricing

**EU hosting**\
The pricing page lists "US + EU data residency" on the Free plan, and higher plans include it.
The EU API is `https://eu.api.novu.co` and the EU socket is `wss://eu.socket.novu.co`.
Sources: https://novu.co/pricing, https://docs.novu.co/platform/inbox/prepare-for-production, https://github.com/novuhq/novu-ts

**Retries and delivery guarantees**\
Novu supports an `Idempotency-Key` header on POST requests, including trigger.
Keys are checked before the request is queued.
Successful and error responses are cached for 24 hours.
A duplicate sent while the first request is still running gets `409 Conflict` with a `Retry-After` header.
The trigger endpoint returns `201` with an `acknowledged` flag and a `status`.
The docs show a manual retry loop that reuses the same key.
The TypeScript SDK (`@novu/api`) says "some of the endpoints" support retries, but the default retry values are not stated.
What Novu does when a channel delivery fails is not covered on the pages read, so it is unknown.
Sources: https://docs.novu.co/api-reference/idempotency, https://docs.novu.co/api-reference/events/trigger-event, https://github.com/novuhq/novu-ts

**React SDK**\
The package is `@novu/react`, and it supports React 18 and 19.
The prebuilt `<Inbox />` has a bell, a feed and user preferences.
Its parts (Bell, Notifications, Preferences) can be used on their own.
Headless hooks include `useNotifications`, `useCounts` and `useNovu`.
`useNotifications` returns the list, paging, and bulk actions such as `readAll` and `archiveAll`.
New notifications arrive over a WebSocket.
Read, seen, archived and snoozed state is tracked by Novu.
Sources: https://docs.novu.co/platform/inbox/overview, https://docs.novu.co/platform/sdks/react/hooks/use-notifications, https://docs.novu.co/platform/sdks/react/hooks/use-counts, https://docs.novu.co/platform/inbox/headless-mode

**Browser authentication**\
The browser uses a public application identifier plus a `subscriberHash`.
The server computes the hash as `HMAC-SHA256(secretKey, subscriberId)`.
You turn on "Security HMAC encryption" in the In-App provider settings.
Once it is on, the Inbox will not load without the hash.
Without it, someone could guess another user's id and read their feed.
The hash does not expire, unlike a JWT.
Source: https://docs.novu.co/platform/inbox/prepare-for-production

## Courier

**Free tier and pricing**\
The Developer plan is free with up to 10,000 sends per month.
Business is pay-as-you-go at $0.005 per send.
Enterprise is custom.
The pricing page does not say whether Inbox is on the Developer plan, but it lists in-app as a channel that counts toward sends.
It also does not say what happens past 10,000 sends on Developer.
Source: https://www.courier.com/pricing

**EU hosting**\
An EU datacenter is available on Business and Enterprise, not on Developer.
It covers the API, the Inbox and its real-time socket (`wss://realtime.eu.courier.io`).
The US and EU workspaces are separate, and data does not move between them.
Source: https://www.courier.com/docs/platform/workspaces/eu-datacenter

**Retries and delivery guarantees**\
`POST /send` accepts an `Idempotency-Key` header.
A retry with the same key returns the stored response and does not send again.
The docs disagree on how long keys last: one page says 25 hours, others say 24 hours.
The Node SDK retries up to 2 times with exponential backoff on connection errors, 408, 409, 429 and 5xx responses.
Sources: https://www.courier.com/docs/reference/idempotent-requests, https://www.courier.com/docs/send/overview, https://www.courier.com/docs/sdk-libraries/node

**React SDK**\
The package is `@trycourier/courier-react`, for React 18 and later.
Prebuilt components are `CourierInbox`, `CourierInboxPopupMenu`, `CourierToast` and `CourierPreferences`.
The `useCourier()` hook gives access to the client, auth, inbox, toast and preferences, for custom UI.
Real-time updates start only after you call `inbox.listenForUpdates()`.
Users can mark messages read and archive them, and state syncs across devices.
The docs do not say React 19 explicitly; they say React 18+.
Sources: https://www.courier.com/docs/sdk-libraries/courier-react-web, https://www.courier.com/docs/platform/inbox/inbox-overview

**Browser authentication**\
The server calls `POST /auth/issue-token` with the API key and a scope string.
The scope includes `user_id:<id>`, `inbox:read:messages` and `inbox:write:events`.
The browser signs in with `signIn({ userId, jwt })`.
The docs say to always set `expires_in`, because a token without it never expires.
The SDK does not refresh tokens itself, so Cadence must mint a new one before expiry.
Source: https://www.courier.com/docs/platform/inbox/authentication

## Knock

**Free tier and pricing**\
The Free plan is $0 with 10,000 messages per month.
Knock does not hard-cap messages; it contacts you to add billing.
Starter is $250 per month with 50,000 messages, then $0.005 each.
Enterprise is custom.
Source: https://knock.app/pricing

**EU hosting**\
Not available today.
Knock's plans page says: "We expect to support EU data residency by the end of 2026."
Its subprocessor list puts hosting (AWS) in the USA.
Sources: https://docs.knock.app/manage-your-account/knock-plans, https://knock.app/legal/subprocessors

**Retries and delivery guarantees**\
`POST /workflows/:key/trigger` accepts an `Idempotency-Key` header.
Knock keeps the key for 24 hours and replays the original response.
Only 2xx responses are cached, so a retry after a 4xx or 5xx runs again.
The Node SDK retries 2 times by default with backoff, on connection errors, 408, 409, 429 and 5xx.
What Knock does when a channel delivery fails is not covered on the pages read, so it is unknown.
Sources: https://docs.knock.app/api-reference/overview/idempotent-requests, https://github.com/knocklabs/knock-node

**React SDK**\
The package is `@knocklabs/react`.
Prebuilt components are `NotificationFeedPopover`, `NotificationFeed` and `NotificationIconButton`.
Headless hooks include `useKnockClient`, `useNotifications` and `useNotificationStore`.
Real-time updates come over a WebSocket from Knock's managed service.
Seen, read, interacted and archived state is tracked by Knock and exposed as counts.
Sources: https://docs.knock.app/in-app-ui/react/overview, https://docs.knock.app/in-app-ui/react/headless/feed, https://docs.knock.app/in-app-ui/feeds/overview

**Browser authentication**\
You turn on "enhanced security mode", which rejects browser requests without a signed user token.
The server signs an RS256 JWT with a signing key from the dashboard.
The `sub` claim is the user id.
With `explicit_grants: true`, the token can be limited to reading that user's feed only.
Source: https://docs.knock.app/in-app-ui/security-and-authentication

## SuprSend

**Free tier and pricing**\
The Free plan is $0 with 10,000 notifications per month.
Its Inbox shows SuprSend branding.
Essentials is $100 per month with 50,000 notifications, then $2 per 1,000.
Business is $250 per month with 50,000 notifications, then $5 per 1,000.
Source: https://www.suprsend.com/pricing

**EU hosting**\
The docs say "Each workspace has a data region, including the EU."
No page read says which plans include the EU region, so plan availability is unknown.
Sources: https://docs.suprsend.com/, https://docs.suprsend.com/docs/security

**Retries and delivery guarantees**\
Triggers accept an `idempotency_key`.
The docs say a duplicate with the same key within 24 hours is rejected.
Whether the SDKs retry on their own is not stated, so it is unknown.
Sources: https://docs.suprsend.com/docs/node-trigger-workflow-from-api, https://docs.suprsend.com/docs/events

**React SDK**\
The package is `@suprsend/react`; the older `@suprsend/react-inbox` is end of support.
It has drop-in components and a headless guide, but the page read did not name them.
The real-time transport is not described on the pages read.
Source: https://docs.suprsend.com/docs/inbox-react

**Browser authentication**\
With enhanced security mode on, the server signs an ES256 JWT.
Claims are `entity_type: "subscriber"`, `entity_id` (the user id), `iat` and `exp`.
The SDK has a `refreshUserToken` callback to fetch a new token before expiry.
Enhanced security mode is off in Sandbox, so a broken token still appears to work there.
Source: https://docs.suprsend.com/docs/client-authentication

## MagicBell

**Free tier and pricing**\
The free Builder plan has 1,000 deliveries per month and 1 project.
That is the smallest free tier here.
Startup is $249 per month with 50,000 deliveries, then $0.0025 each.
Source: https://www.magicbell.com/pricing

**EU hosting**\
No EU hosting is documented.
The only "eu" region setting found belongs to the Mailgun email integration.
Sources: https://www.magicbell.com/pricing, https://www.magicbell.com/docs/api/reference

**Retries and delivery guarantees**\
The API supports an `Idempotency-Key` header, cached for 24 hours (v1 docs).
The v2 docs say only "some endpoints" support it.
SDK retry behaviour is not documented, so it is unknown.
Sources: https://www.magicbell.com/docs/rest-api/idempotent-requests, https://www.magicbell.com/docs/api

**React SDK**\
The package is `@magicbell/react`.
Components include `ContextProvider`, `FloatingInbox`, `Inbox`, `InlineNotification`, `ToastContainer` and `UserPreferences`.
Headless hooks and the real-time transport are not described on the page read.
Source: https://www.magicbell.com/docs/libraries/magicbell-react

**Browser authentication**\
The server signs an HS256 User JWT with the project's secret key.
Claims are `api_key` plus `user_email` or `user_external_id`.
Source: https://www.magicbell.com/docs/api/authentication/user

## Why this order

Novu is first because EU hosting comes free, and the paid step is small.
Its HMAC hash is simpler to produce than a JWT but never expires.

Courier is second because pricing after the free tier is pay-as-you-go with no monthly base.
EU hosting exists, but needs the paid plan, and its EU workspace is separate from the US one.
Whether Inbox is on the free plan should be confirmed at sign-up.

Knock would be first on SDK and token design alone.
It drops to third because it has no EU hosting yet and its first paid plan is $250 per month.

SuprSend is a reasonable fallback, but its EU plan availability is unclear and the free Inbox is branded.

MagicBell is last because its free tier is only 1,000 deliveries and it has no documented EU hosting.

## Open questions for sign-up

- Does Courier's free Developer plan include Inbox?
- What happens past each free limit (hard stop or soft)?
- How does each provider retry a failed channel delivery after a trigger is accepted?
- Do the React SDKs render cleanly in React Router 8 with server rendering, or only on the client?
