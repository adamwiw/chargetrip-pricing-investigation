# ChargeTrip Charging-Time ("Charging Curve") Estimation

Reverse-engineered description of how the ChargeTrip white-label front-end estimates the
**session charging duration** shown for a connector, and why a naive
`energy / plugPower` calculation will not match it.

- **Application:** `https://arval-production.discover.chargetrip.com`
- **Source:** client-side JavaScript bundle `index47-DkvWNWyr.js`
  (function exported as `n`, internally `Lt`), consumed by
  `helpers-ByS83r-8.js` and `StationPricing-DKLf_VVk.js`.

---

## 1. Summary

ChargeTrip does **not** integrate a per-state-of-charge charging curve on the client.
Instead it applies a **closed-form, power-dependent de-rating** to the connector's rated
power, divides a fixed usable-energy window by it, and caps the result by the vehicle's
own adapter/connector power.

The two inputs a naive calculation typically omits are:

1. a **de-rating fraction** derived from the connector power (≈0.80–0.97), and
2. the **usable-energy window** (default 70% of usable battery capacity).

---

## 2. The formula

### 2.1 Power de-rating ("curve") function

```js
const effectiveFraction = P => (80 + 17 / (1 + (P / 40) ** 1.5)) / 100;
```

`P` = connector rated power in kW. The fraction asymptotes to **0.80** at high power and
rises toward **≈0.97** at low power:

| Rated power `P` (kW) | `effectiveFraction(P)` | Effective power (kW) |
|---:|---:|---:|
| 11  | 0.949 | 10.4 |
| 22  | 0.921 | 20.3 |
| 50  | 0.871 | 43.6 |
| 100 | 0.834 | 83.4 |
| 150 | 0.821 | 123.1 |
| 250 | 0.810 | 202.4 |
| 350 | 0.806 | 282.0 |

This single function is the "curve": it approximates the average delivered power over a
session (i.e. the effect of taper) as a function of the nominal plug power alone.

### 2.2 Duration calculation

```js
const estimate = ({ standard, vehicle, connectorPower, usableBattery = 0.7 }) => {
  const frac   = effectiveFraction(connectorPower);
  let   power  = connectorPower * frac;                       // (a) de-rated connector power
  const energy = vehicle.battery.usable_kwh * usableBattery;  // (b) usable-energy window

  if (standard) {
    // (c) cap by the vehicle's adapter, if present
    const adapter = vehicle.adapters?.find(a => a.standard === standard) || null;
    let cap;
    if (adapter?.power) cap = adapter.power * frac;

    // (c) cap by the vehicle's connector for the same standard
    const vehConn = vehicle.connectors.find(c => c.standard === standard);
    if (vehConn?.power) cap = vehConn.power * frac;

    if (cap && connectorPower > cap) power = cap;            // effective = min(connector, vehicle)
  }

  return Math.round((energy / power) * 60);                  // minutes
};
```

### 2.3 Inputs

| Input | Source | Notes |
|---|---|---|
| `connectorPower` | connector `power` | rated kW of the station plug |
| `vehicle.battery.usable_kwh` | vehicle model data | **usable** capacity, not nominal |
| `usableBattery` | constant | default **0.7** (i.e. a 10%→80% window) |
| `standard` | connector standard | used to find the vehicle adapter/connector |
| `vehicle.adapters[].power` | vehicle model data | optional power cap |
| `vehicle.connectors[].power` | vehicle model data | optional power cap |

---

## 3. Why `energy / plugPower` does not match

A naive `energy / plugPower` typically:

1. **Omits the de-rating fraction.** Rated plug power is multiplied by a value between
   ≈0.80 and ≈0.97 depending on power, lengthening the estimate (5–20%).
2. **Ignores the vehicle/adapter power cap.** If the vehicle cannot accept the station's
   full power for the chosen standard, the estimate is capped by the vehicle's rated power
   (also de-rated).
3. **Uses a different energy basis.** ChargeTrip uses `usable_kwh * 0.7`, not the full
   (or nominal) pack capacity.

The net difference therefore depends on the relative sizes of the de-rating and the
window: higher-power chargers de-rate more (larger gap), while the 0.7 window reduces the
energy side. Both must be reproduced for parity.

---

## 4. Notes and caveats

- **It is a heuristic, not a curve.** There is no per-SoC array in the client bundle; the
  only "curve" is the analytic `effectiveFraction(P)`. If a per-SoC curve is used anywhere,
  it is server-side (route-level `durations.charging`), not in this station estimate.
- **Rounding** is to the nearest minute (`Math.round`).
- The same function feeds the connector list ("10% to 80% in N minutes") and the pricing
  breakdown's session-time input, so a single discrepancy propagates to both the
  connector labels and the cost estimate.

---

## 5. Suggested parity test

For a given connector + vehicle pair, verify the implementation reproduces:

```
minutes = round( (usable_kwh * 0.7) / (min(connectorPower, vehicleCap) * effectiveFraction(min(connectorPower, vehicleCap))) * 60 )
```

where the de-rating is applied to whichever power becomes the limiting value, and
`vehicleCap` is the vehicle's adapter/connector power for the connector's standard (if any).

Values to compare, at minimum:

| P (kW) | expected de-rating |
|---:|---:|
| 11 | 0.949 |
| 50 | 0.871 |
| 150 | 0.821 |
| 350 | 0.806 |

---

## 6. Disclaimer

Derived from publicly served, unauthenticated client-side resources for interoperability
analysis. Only the minimal expressions required to describe the calculation are
reproduced. All trademarks belong to their respective owners.
