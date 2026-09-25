# ChargeTrip White-Label App — Pricing Component Discrepancy

An investigation into why the ChargeTrip "no-code" white-label web app displays an
**idle fee** and **no session-time cost** for certain charging stations, while the raw
GraphQL response for the same station contains a **`TIME` price component** and **no
`PARKING_TIME` component**.

- **Application analysed:** `https://arval-production.discover.chargetrip.com`
- **API:** `https://api.chargetrip.io/graphql`
- **Method:** static analysis of the publicly served client-side JavaScript bundles
  (Chrome DevTools Protocol capture + de-minification), corroborated against the
  GraphQL schema/fragments embedded in the bundle.

> **Result:** the behaviour is **not** a client-side caching artefact. It is a
> deliberate, operator-specific normalisation in the shared Chargetrip front-end that
> **relabels OCPI `TIME` price components as `PARKING_TIME` for Shell operators**
> before they are rendered.

---

## 1. Observed discrepancy

| | Raw GraphQL response | Rendered UI |
|---|---|---|
| Session time (`TIME`) | present | not shown (`-`) |
| Parking / idle (`PARKING_TIME`) | absent | shown as an **Idle fee** |

Because the API and the UI appear to contradict one another, a client-side caching bug
(e.g. a stale value retained from a previously viewed station) was a reasonable first
hypothesis. The bundle analysis shows otherwise.

---

## 2. Root cause

### 2.1 Operator-aware relabelling (`TIME` → `PARKING_TIME`)

In the shared station chunk (`station-DzQQNbHy.js`), a post-processing function maps the
station's OCPI `tariff` into the app's internal `prices` model. For operators whose
normalised name is `shell`, and only when specific conditions hold, it rewrites the price
component type:

```js
K = async e => {
  let t = (await s())?.name === `shell`;                       // operator is Shell?
  return { ...e, prices: e.tariff?.filter(i).map(el => {
    // Skip if not Shell, or if a PARKING_TIME component already exists
    if (!t || el.elements?.some(x => x?.price_components?.some(c => c?.type === `PARKING_TIME`)))
      return g(el) || [];

    let n = el.elements?.some(x => x?.price_components?.some(c => c?.type === `ENERGY`));
    // Rewrite TIME -> PARKING_TIME when an ENERGY component exists
    // and the element carries a min/max duration restriction
    ... price_components.map(c => {
      let r = el.restrictions?.[0]?.min_duration || el.restrictions?.[0]?.max_duration;
      return c?.type === `TIME` && r && n ? { ...c, type: `PARKING_TIME` } : c;
    })
  })}
}
```

The `shell` operator name is assigned per country in the station database
(`FR`, `AT`, `HU`, `SK`, `DE`, `IT`, `CZ`, `PL`, `BE` → `name: "shell"`).

**Net effect:** the component received from the API as `TIME` is stored internally as
`PARKING_TIME`. The raw GraphQL response is unchanged; only the app's in-memory model
is rewritten.

### 2.2 Rendering consequences (`StationPricing-DKLf_VVk.js`)

Two independently rendered sections consume the normalised model:

- **`IdleFees`** reads the **`parking_time`** component:
  ```js
  const { element: h, component: _ } = J(pricingModule, `parking_time`);
  ```
  Since the relabelled component is now `parking_time`, it populates the **Idle fees**
  section.

- **`Breakdown`** renders the session-time row only when a `time` component survives and
  the `isTimeAllowed` predicate passes:
  ```js
  w.isTimeAllowed && timeValue && timeComponent ? formattedTime : `-`
  ```
  After relabelling, no `time` component exists, so the row renders **`-`** — i.e. "no
  cost for time spent charging".

### 2.3 Supporting predicate

The pricing model also gates the session-time contribution:

```js
get isTimeAllowed() {
  const time    = this.getComponentByType(`time`);
  const parking = this.getComponentByType(`parking_time`);
  const energy  = this.getComponentByType(`energy`);
  return !(time?.price_excl_vat === parking?.price_excl_vat && energy && energy.price_excl_vat > 0);
}
```

When `time` and `parking_time` share a price and an energy component is present, the
session-time value is zeroed in the breakdown and therefore not displayed.

---

## 3. Caching hypothesis — ruled out

The price model is **recreated** whenever the active price set or external price id
changes, rather than being mutated in place:

```js
watch([priceExternalId, prices], () => {
  if (!priceExternalId.value || !prices.value) { module.value = null; return }
  module.value = new PricingModule({
    strategy: `external`,
    timeZone: station.value?.time_zone,
    externalId: priceExternalId.value,
    prices: prices.value
  })
}, { immediate: true })
```

A fresh instance is constructed per station/price change, and the module is reset on
dispose. This makes a stale cross-station cache the **unlikely** cause of the observed
behaviour.

### 3.1 Latent (non-triggering) cache bug — noted for completeness

The pricing module exposes setters, one of which clears only part of its memo state:

```js
clearCaches() { this.#normalized.clear(); this.#price.clear(); this.#breakdown.clear(); }
setPrices(p) { this.prices = p; this.#normalized.clear(); }   // does NOT clear #price / #breakdown
```

If `setPrices()` were ever called on a reused instance, the memoised `price` and
`breakdown` would remain stale until `setConsumption`/`setDuration`/`clearCaches` ran.
No caller of `setPrices()` was found in the shipped bundle, so this is currently
dormant — but it is a real defect worth reporting upstream.

---

## 4. Implications

1. **API/UI parity.** For Shell stations, the app's interpretation of an OCPI `TIME`
   component is that it represents an **idle / overstay** charge, not a charge for time
   spent charging. Any integration that treats Shell `TIME` as a session-time charge
   will systematically disagree with the app.

2. **Not a data bug at the API layer.** The raw GraphQL response is internally
   consistent; the divergence is introduced entirely in the client normalisation layer.

3. **Reproducibility.** The relabelling fires only when **all** of the following hold:
   - `operator.name === "shell"`
   - the tariff element contains an `ENERGY` component
   - the element carries a `min_duration` **or** `max_duration` restriction
   - no `PARKING_TIME` component is already present

---

## 5. How to verify

For the affected station, confirm in the raw GraphQL response:

- [ ] `operator.name == "shell"`
- [ ] an `ENERGY` price component is present
- [ ] a `min_duration` or `max_duration` restriction is present
- [ ] no `PARKING_TIME` component is present

If all four are true, the app will render an **idle fee** and a **`-`** for session time.
This is expected behaviour of the current front-end, not a rendering or caching fault.

---

## 6. Files analysed (client chunks, public assets)

| Chunk | Role |
|---|---|
| `station-DzQQNbHy.js` | Station post-processing (`TIME`→`PARKING_TIME`), GraphQL fragments, pricing model |
| `StationPricing-DKLf_VVk.js` | `Breakdown` and `IdleFees` renderers |
| `station-n6afTM1v.js` | Pricing module lifecycle (per-station instantiation) |
| `index47-DkvWNWyr.js` | OCPI price-component normalisation and restrictions |
| `en-US-pNCLjXyY.js` | UI strings (idle fees, standard fees, session time) |

Chunk filenames are content-hashed and change between deployments; the module contents
and the logic above are stable.

---

## 7. Disclaimer

This analysis was performed against publicly served, unauthenticated client-side
resources, solely to understand an observed behavioural discrepancy for interoperability
purposes. Only minimal excerpts necessary to explain the behaviour are reproduced.
All trademarks belong to their respective owners.
