# Minimal Auto Location Refresh Design

Status: DRAFT - requires approval before implementation.

Audience: BUCKS storefront shoppers.

## Goal

When location-based auto-switch is enabled, refreshing the same tab requests fresh location unless the shopper has an explicit currency choice. A VPN change can then affect displayed currency if the location provider returns the new country and the existing conversion flow supports the switch.

This is a narrow cache/selection fix, not a redesign of Shopify localization or widget reconversion.

## Current problem

1. `autoSwitchCurrency.js` restores `buckscc_customer_currency` before attempting location detection.
2. `convert/convertToLocalCurrency.js` also returns early when `hxoGeoCurrency` exists.
3. Automatic conversion and manual selection both save the current currency. The saved value alone cannot distinguish intent.
4. Session storage survives a same-tab refresh; a VPN connection does not invalidate it.

The existing `buckscc_auto_currency` and `buckscc_last_manual` localStorage entries are analytics records and cannot reliably identify selection intent.

## Recommended solution

Add one session boolean, `buckscc_manual_currency_selected`, using `eStore` directly in the existing files. Do not introduce a storage helper or shared request-state module.

| Action | Flag | Result |
| --- | --- | --- |
| Automatic first visit | Absent | Request location and apply a supported detected currency |
| Refresh in automatic mode | Absent | Request location even when both currency caches exist |
| Explicit dropdown currency | `true` | Preserve the existing saved currency on refresh |
| Accepted URL currency | `true` | Preserve the accepted URL choice on refresh |
| Auto Location click | Remove flag | Request location immediately rather than applying the cached dropdown ID |

Read the flag as strict boolean true. Keep the formats of `buckscc_customer_currency`, `hxoGeoCurrency`, and `hxoGeoCountry` unchanged.

### Why this approach

Always refreshing location without the flag would undo manual choices on navigation. A geo-cache expiry alone would still leave the saved-current-currency shortcut and would not guarantee detection on the next refresh. One flag and four existing production files are the smallest proposed scope that preserves explicit choices.

No new merchant settings or shopper UI are needed. Existing currency labels and formatting remain intact.

## Detection and fallback

- Automatic initialization and Auto Location explicitly request a fresh lookup, bypassing the cached-geo shortcut for those requests.
- Preserve existing merchant-preview transport and behavior; the freshness requirement is for storefront shoppers, not a preview redesign.
- Bound a fresh storefront attempt, including response-body parsing, to **2500 ms**.
- On a successful valid result, update the country/currency cache pair and invoke the existing conversion flow with its merchant restrictions and multi-currency behavior.
- On rejection, unsuccessful HTTP status, malformed response, unavailable rate, disallowed currency, or timeout: use the previously applied currency if usable, otherwise the cached geo currency if usable. If neither is usable, leave current prices unchanged.
- A fallback does not set the manual flag and does not claim fresh location detection succeeded. Do not introduce additional fallback analytics dispatches; keep existing analytics behavior inside shared conversion paths.
- Clear the timeout after settlement. Late results cannot change storage, labels, prices, or events after timeout.
- Retain existing visit-tracking completion behavior; do not attribute a failed lookup to a newly detected country.

## Request ordering without another file

Keep a module-local request counter in `convertToLocalCurrency.js`. Export a small invalidation function from that same file for explicit dropdown selections.

- A new lookup supersedes older lookups.
- Explicit manual selection invalidates pending lookups before applying the choice.
- Each response and fallback checks that its request is still current and no manual flag is active.
- An auto -> manual -> auto sequence must not allow the first automatic response to overwrite the second. The flag alone is insufficient, hence the counter.
- Keep network parsing free of storage/UI side effects; apply those only after the timeout race and request checks succeed.

## URL behavior

Preserve the existing handler's precedence: an already-saved currency prevents accepting a new URL override. Otherwise accepted `currency` or `bucks_currency` parameters save the currency and set the flag. Invalid or disallowed parameters do not set it.

This records accepted URL intent; it does not change which links take priority.

## Auto Location interaction

Clear the manual flag, close the existing dropdown as appropriate, and request fresh detection. Do not immediately display or dispatch the cached item ID as if it were a fresh result. Update the selected label and existing currency-change event through the accepted conversion/fallback path. Refresh the cart banner after current-request completion when enabled, not from a stale response.

An Auto Location click does not change the merchant's auto-switch setting. If that setting is disabled, later page loads retain their existing behavior.

## Four-file production boundary

All paths below are relative to `widgets/src/apps/widgets/buckscc/`:

1. `autoSwitchCurrency.js`: flag-aware initial routing and explicit fresh lookup request.
2. `convert/convertToLocalCurrency.js`: fresh-request option, timeout, fallback, and local request counter.
3. `template/refreshTriggerEvent.js`: record manual intent, invalidate requests, and re-detect on Auto Location.
4. `common/convertCurrencyByUrlParams.js`: record accepted URL intent.

Tests and these documents are additional files, but production modifications stay within these four files.

## Deliberate limitations

- `common/reconvert.js` is unchanged. Existing page-load/DOM handlers may briefly reapply cached currency while detection is pending. This version does not promise flicker-free pending behavior.
- `common/changeMultiCurrency.js` and its path-based reload guard are unchanged. A Shopify localization change can still be blocked after an earlier automatic submission on the same path. Fresh detection is fixed; successful checkout/localization switching is not guaranteed in that case.
- No manual multi-currency restoration redesign, same-currency country switching changes, or retry-history migration.
- No live VPN listener, polling, cache TTL, or new request-state helper.
- Existing rerender callers may also initiate detection through `autoSwitchCurrency`; this version does not promise exactly one request per page or change resize handling. Superseded results remain ignored.
- Provider country accuracy and HTTP cache behavior need live verification. A new request is not proof that the provider has detected the VPN country.

## Existing sessions

Missing flag means automatic mode when auto-switch is enabled. An old manual choice may therefore reset once after deployment because existing data cannot reliably identify its source. New explicit choices remain preserved. Approving this design includes accepting this rollout tradeoff.

## Acceptance checks

1. Saved automatic INR + fresh US response on reload -> USD in the normal display conversion path.
2. Manual INR, even if equal to detected INR, remains INR on refresh without a switching geo request.
3. Accepted URL currency sets the flag; rejected URL and already-saved-currency precedence remain unchanged.
4. Auto Location after manual EUR requests location and uses the result rather than cached item ID.
5. Rejected, malformed, disallowed, or never-settling lookup uses the defined fallback; no usable fallback leaves prices unchanged.
6. A response after the 2500 ms deadline is ignored; a later manual choice beats a pending response.
7. Two requests resolving out of order, including auto -> manual -> auto, apply only the latest intent.
8. Preview, auto-switch-disabled behavior, dropdown closure, and banner completion remain functional.
9. Verify existing localization and pending-reconversion limitations separately; do not label them fixed.

## Approval

Approve this design and its companion plan before implementation. The previous reset implementation is not to be restored.
