# Voice shopping and ticket booking

Status: approved planned extension in Phase 4 of [the feature roadmap](./feature-ideas.md). No shopping, booking, payment, or provider adapter is implemented. [The architecture](./architecture.md#voice-shopping-and-ticket-booking-planned-extension) defines its relationship to the voice coordinator and action cards.

Follow [the runtime recovery design](./scaling.md#durable-work-and-background-pools) for durable approval/operation records, outbox dispatch, callback validation and reconciliation. These workflows are independent of temporary voice history and are introduced with M7–M8.

## Intended experience

Users describe a shopping or travel request by voice, refine it conversationally, review the exact proposal, and authorize a specific purchase or booking. Initial target providers are Flipkart, Blinkit, Zepto, and IRCTC or an authorized ticketing partner. Support is limited to verified provider capabilities, available locations/catalogs, and the user's authorized account; the target list is not a promise that any item or ticket can already be purchased.

Examples of intended requests:

- “Find two 1-litre milk packs and eggs within my total budget.”
- “Compare this headphone model on the supported stores.”
- “Find a train from Delhi to Lucknow next Friday; I need two seats.”

Resolve ambiguous brands, variants, quantities/units, delivery location, dates/timezones, passengers, class/seats, and budget before proposing execution. Confirm absolute travel dates and the actual route. Ask about substitutions or waitlisted tickets when relevant; do not silently change what the user requested.

## Workflow and approval

1. **Prepare a local draft.** Parse the finalized utterance once, keep ownership and current-turn checks, and ask for missing information. Preferences and spending limits constrain the search; they do not authorize a purchase.
2. **Discover and quote.** Use supported read-only search/catalog access, showing provider, location, freshness, stock/seat availability, and seller/fare details. Obtain a current checkout quote when the provider supports it. Treat cached estimates as estimates. A paid reservation or inventory hold is an external action requiring its own appropriate authorization.
3. **Review.** Display and read back the relevant cart/itinerary, quantities, variants, address or passenger details, provider, currency, all known charges/discounts, delivery/travel timing, and cancellation/refund terms. Keep the purchase budget separate from the app's speech/model/video budget. State unknown fees or availability rather than claiming a complete live quote.
4. **Confirm a specific proposal.** Approval is bound server-side to an authenticated owner, action ID, version, exact payload/quote, currency, maximum authorized total, scope, and expiry. Material item, price, address, itinerary, availability, or terms changes invalidate approval. Voice confirmation is allowed only when it unambiguously identifies the current reviewed proposal in an authenticated context; otherwise use an explicit on-screen confirmation. A familiar/cloned voice, background audio, an unrelated “yes,” or silence is not authentication or approval.
5. **Complete supported checkout.** Execute only through a verified supported adapter/flow and current authorization. For providers without suitable access, offer a clearly labelled handoff with the reviewed list/itinerary and a supported destination. Do not imply that a generic link fills a cart or completes checkout. Required login, payment approval, OTP, or CAPTCHA steps stay in the provider's secure user flow.
6. **Verify the result.** Report an order/booking as placed only after a verified provider result. Keep order/booking, seat status, payment, and refund status separate; an issued waitlisted ticket is not a confirmed seat. A handoff or user-reported completion is labelled accordingly until verified. Return an owned receipt/reference and supported tracking options.

Each utterance remains a normal bounded voice turn. Pending approval or secure checkout belongs to the separately owned action workflow; do not keep an agent run open indefinitely waiting for the user. Register capability-specific, runtime-validated tools only when an adapter is implemented, with provider logic behind backend services. Frontend pages connect search results, comparison cards, draft carts/itineraries, and transaction status to atomic UI components through feature hooks.

## Provider access readiness

The public documentation reviewed does not establish universal consumer checkout access. Recheck terms, account permissions, authentication, and supported endpoints during implementation.

| Target | Current documentation finding | Implementation gate |
| --- | --- | --- |
| Flipkart | The [public Marketplace documentation](https://flipkart.github.io/fk-api-platform-docs/docs/intro) describes seller operations. Those APIs do not establish a buyer checkout integration. | Verify supported consumer discovery/checkout access independently. Keep seller/affiliate/catalog access distinct from customer purchasing. |
| Blinkit | Public consumer checkout integration has not been verified for this project. | Begin with draft preparation and supported handoff; enable live account/cart/checkout operations only after supported access is established. |
| Zepto | Public consumer checkout integration has not been verified for this project. | Apply the same per-capability gate; do not infer purchase access from a storefront or seller/brand portal. |
| IRCTC rail tickets | The [reviewed B2C PSP policy](https://contents.irctc.co.in/en/B2C_PSP_Policy_other_than_Startup_MSME.pdf) describes authorized providers and affiliate approval. | Establish a current authorized direct/partner route before live rail booking. A personal website login is not proof of developer API authorization. |
| Other ticket providers | No flight, bus, or event provider has been selected. | Verify search, quote/hold, booking, payment, tracking, cancellation, and refund capabilities independently. |

Provider capability status must be visible as unavailable, handoff-only, or verified direct execution. Use supported official/partner access or an explicitly supported user-controlled flow; do not bypass authentication/security controls or rely on undocumented private endpoints. Search, cart changes, booking, payment, cancellation, and refund are separate permissions.

## Credentials, payment, and saved details

Keep integration credentials and account authorization references on the server. Use provider-supported account linking and hosted checkout where available. Do not request spoken passwords, OTPs, card security codes, or payment PINs; do not put them into model context, transcripts, tools, or logs. Pause capture, transcription forwarding, and speech playback before secure-input handoff, invalidate outstanding input, and resume listening only through a valid user action.

Collect only required address/passenger fields and explain their transfer to the provider. Saved address/traveller preferences require an explicit storage choice and ownership/removal controls. Retain only the transaction references/status needed for receipt, reconciliation, and tracking under a stated retention policy; transaction authorization does not opt the user into saving every conversation or raw audio. Access to a provider account remains revocable.

## Execution and failure handling

Track a durable owned operation ID/version independently of voice-turn and playback generations, with states such as `draft`, `awaiting_confirmation`, `awaiting_user_checkout`, `submitting`, `confirmed`, `failed`, `unknown`, `cancelled`, and `expired`. Atomically consume an approval once and use provider idempotency/status lookup where supported. Repeated confirmation, duplicate callbacks, reconnect, or a lost response must not create duplicate orders/bookings.

Revalidate the quote and permission immediately before execution. On an uncertain result, keep status `unknown`, reconcile using the owned provider reference, and require resolution before another purchase attempt. Do not interpret a timeout or interrupted microphone session as proof that no order was placed. Retain verified external outcomes even if the conversational turn is cancelled; attach them to the original action without overwriting a newer answer or autoplaying late speech. Cancellation of an already placed order/ticket is a new authorized operation with current terms, not rollback of a voice turn.

Record approval consumption and the durable `submitting` operation/dispatch intent atomically before provider dispatch; do not hold database locks during network calls. A crash, expired worker lease or restored backup may follow an already submitted external effect. Suspend unsafe retries, reconcile using the original operation/idempotency reference, and invalidate stale restored approvals rather than blindly resubmitting.

Acceptance checks include ambiguous quantities/dates, wrong variants/passengers, location/stock/seat changes, fee/price changes, stale/edited/expired approval, replayed or ambiguous voice confirmation, login/OTP/payment handoff, revoked access, user cancellation, duplicate confirmation/callbacks, unknown provider outcomes, waitlisted seats, failed payments/refunds, and unsupported-provider handoff. Test isolation across users and purchase budgets. The first live milestone enables one verified provider or a clear handoff flow; mock providers verify behavior before any real-money test.
