# Developer Guide

## Objects and how to use them

### Tariff
The main object that describes a single tariff with all its components: `fixedPrice`, `energyPrice`, `powerPrice`, and `efficiencyPrice`.

`powerPrice` and `efficiencyPrice` are required on every tariff, but a tariff that has no effect/power charge or no efficiency charge should still include the object with an empty `components` array rather than omitting it.

### DateInterval (validPeriod)
Describes a typically longer time period (summer/winter) between two dates. Used for the property `validPeriod` in the `Tariff` object and price components.
The valid period of the `Tariff` object is most often a full year and the price components can differ and be valid only during a part of the year. District heating energy prices in particular are often differentiated by calendar month rather than just two seasons — represent this with one `EnergyPriceComponent` per month (or per group of months sharing a rate), each with its own `validPeriod`.

### RecurringPeriod
The `RecurringPeriods` can be added if a price component (`FixedPriceComponent`, `EnergyPriceComponent`, `PowerPriceComponent`, `EfficiencyPriceComponent`)
is not active during all of the specified `ValidPeriod`. If no `RecurringPeriods` are added or the property is missing in the
response, the price component is active during all times within the `ValidPeriod`.

### CostFunction
The `costFunction` property of the different price objects (`FixedPrice`, `EnergyPrice`, `PowerPrice`, `EfficiencyPrice`) is a pseudo code
string that explains how to combine the `components` of each price object. To get the full cost of each price object,
the price components in the `components` array are combined as described in the `costFunction` property.

#### Methods
|Method|Description|
|-|-|
|`sum(expression(c))`|The sum of the calculated expression for each component (`c`).|
|`price(c)`|The price of a component. For a component with a tiered price, this is the price resolved from its tiers, see [TieredPrice](#tieredprice-stafflade-priser).|
|`energy(c)`|The heat energy consumption/production in `unit` (e.g. MWh) for a component.|
|`volume(c)`|The volume of heating medium (e.g. m³) delivered for a component. Used for `EfficiencyPrice` components where `measurementMethod` is `volume`.|
|`temperatureDeviation(c)`|The difference, in °C, between the facility's measured return temperature and the reference temperature defined by the component's `returnTemperatureSettings`, for the component's `measurementPeriod`. Positive when the facility's return temperature is *above* the reference (a less efficient installation, resulting in a surcharge); negative when *below* (a more efficient installation, resulting in a credit). Used for `EfficiencyPrice` components where `measurementMethod` is `returnTemperature`.|
|`peak(c)`|The heat power peak value for a power price component as calculated by `peakIdentificationSettings`, `activePeriods` in `recurringPeriods` and `validPeriod`. For example, the `peak(c)` value may be the average heat power of the three highest daily mean effects over the trailing 12 months.|
|`energy(p)`|The average heat energy consumption/production for a period (`p`), where the period is a distinct time interval with start and end. For example one hour or 15 minutes.|
|`power(p)`|The average heat power value for a period (`p`), where the period is a distinct time interval with start and end. For example one hour or 15 minutes.|
|`price(p)`|The price for a period (`p`), where the period is a distinct time interval with start and end. For example one hour or 15 minutes.|

#### Examples
|Example|Description|
|-|-|
|`sum(price(c))`|The sum of the price of each component within this price segment. Often used for the `FixedPrice` object where there are only fixed prices for a year or month where the properties `validPeriod`, `price` and `pricedPeriod` defines the price for a point in time.|
|`sum(energy(c)*price(c))`|The sum of the calculated cost for each component where the cost is the heat energy consumption/production multiplied with the price for each component. Used for the `EnergyPrice` object.|
|`sum(peak(c)*price(c))`|The sum of the calculated cost for each component where the cost is the heat power peak multiplied with the price for each component. Used for the `PowerPrice` object.|
|`sum(volume(c)*price(c))`|The sum of the calculated cost for each component where the cost is the delivered volume of heating medium multiplied with the price for each component. Used for `EfficiencyPrice` components with `measurementMethod` `volume`.|
|`sum(energy(c)*temperatureDeviation(c)*price(c))`|The sum of the calculated cost for each component where the cost is the delivered heat energy multiplied by the temperature deviation and the price for each component. Can be negative overall (a net rebate) when facilities run cooler than the reference. Used for `EfficiencyPrice` components with `measurementMethod` `returnTemperature`.|

### TieredPrice (stafflade priser)
The `price` of a `FixedPriceComponent`, `PowerPriceComponent` or `EnergyPriceComponent` is either a constant `Price` or a `TieredPrice`.
A tiered price depends on which tier a quantity falls in, for example the heat power peak or the energy volume of a year.

|Property|Description|
|-|-|
|`currency`|The currency of all tier prices.|
|`tierUnit`|The unit of the thresholds, for example `kW` or `MWh`. It does not have to be the unit of the component itself: a fixed price can be tiered on power.|
|`tierBasis`|Optional. A pseudo code expression for the quantity that is compared with the thresholds. When omitted, the quantity priced by the component itself is used: `peak(c)` for a power price component and `energy(c)` for an energy price component. A fixed price component has no quantity of its own and must state `tierBasis`.|
|`tierMode`|How the price of a tier is applied: `stepped`, `volume` or `graduated`, see below.|
|`tierPeriod`|Optional. The period over which the quantity is accumulated before it is compared with the thresholds, for example `P1Y` for the energy volume of a year. Needed when the quantity is a sum over time, such as energy. Not needed for a peak, whose period is given by `peakIdentificationSettings`.|
|`tiers`|The tiers, each with `fromIncluding`, `priceExVat` and `priceIncVat`.|

#### Tiers
Each tier states only its lower threshold, `fromIncluding`. A tier ends where the next one begins and the last tier has no upper limit.
The tiers must be listed in ascending order of `fromIncluding` and the first tier must start at `0`, so that every quantity falls in exactly one tier.
A quantity equal to a threshold belongs to the tier that starts there: with thresholds `0` and `100`, a quantity of exactly 100 falls in the second tier.

#### Tier modes
|`tierMode`|Tier price is|Cost|Typical use|
|-|-|-|-|
|`stepped`|An amount for the priced period.|The price of the tier the quantity falls in. The quantity is not multiplied with anything, so the cost is a staircase.|`FixedPriceComponent`|
|`volume`|A price per unit.|The whole quantity multiplied with the price of the tier it falls in. The cost can jump at a threshold.|`PowerPriceComponent`, `EnergyPriceComponent`|
|`graduated`|A price per unit.|Each part of the quantity multiplied with the price of its own tier. The cost is a continuous line that changes slope at each threshold.|`PowerPriceComponent`, `EnergyPriceComponent`|

The cost functions are the same as for constant prices. `price(c)` for a tiered component is resolved like this:

|`tierMode`|`price(c)`|
|-|-|
|`stepped`|The price of the tier the quantity falls in.|
|`volume`|The price of the tier the quantity falls in.|
|`graduated`|The average price per unit over all parts of the quantity, so that the quantity multiplied with `price(c)` equals the sum of each part multiplied with the price of its tier.|

#### Methods for tierBasis
|Method|Description|
|-|-|
|`peak(r)`|The heat power peak of the power price component with `reference` name (`r`) in the same tariff.|
|`energy(r)`|The heat energy of the energy price component with `reference` name (`r`) in the same tariff, accumulated over `tierPeriod`.|

#### Example: constant price
For contrast, a fixed monthly fee that is the same for every customer uses a constant `Price`.
The component examples in this section leave out `id` and `description` for brevity.

```json
{
  "name": "Network fee",
  "type": "public",
  "reference": "main",
  "validPeriod": { "fromIncluding": "2026-01-01", "toExcluding": "2027-01-01" },
  "pricedPeriod": "P1M",
  "price": { "priceExVat": 299.20, "priceIncVat": 374.00, "currency": "SEK" }
}
```

#### Example: stepped fixed price
An annual fixed fee that depends on the heat power peak. The price of the tier is the fee for the whole `pricedPeriod`.

```json
{
  "name": "Fixed effect fee",
  "type": "public",
  "reference": "main",
  "validPeriod": { "fromIncluding": "2026-01-01", "toExcluding": "2027-01-01" },
  "pricedPeriod": "P1Y",
  "price": {
    "currency": "SEK",
    "tierUnit": "kW",
    "tierBasis": "peak(main)",
    "tierMode": "stepped",
    "tiers": [
      { "fromIncluding": 0,    "priceExVat": 10870,  "priceIncVat": 13587.50 },
      { "fromIncluding": 100,  "priceExVat": 15270,  "priceIncVat": 19087.50 },
      { "fromIncluding": 250,  "priceExVat": 27020,  "priceIncVat": 33775.00 },
      { "fromIncluding": 500,  "priceExVat": 51520,  "priceIncVat": 64400.00 },
      { "fromIncluding": 1000, "priceExVat": 101520, "priceIncVat": 126900.00 },
      { "fromIncluding": 2500, "priceExVat": 229020, "priceIncVat": 286275.00 }
    ]
  }
}
```

With the cost function `sum(price(c))`, a facility with a peak of 180 kW falls in the tier that starts at 100 kW and pays 15 270 SEK per year excluding VAT. So does a facility with a peak of 240 kW.

#### Example: volume tiered power price
A power price where the whole peak is priced at the rate of the tier it falls in. `tierBasis` is omitted, so the tier is selected by the peak of the component itself.

```json
{
  "name": "Effect fee",
  "type": "peak",
  "reference": "main",
  "validPeriod": { "fromIncluding": "2026-01-01", "toExcluding": "2027-01-01" },
  "unit": "kW",
  "price": {
    "currency": "SEK",
    "tierUnit": "kW",
    "tierMode": "volume",
    "tiers": [
      { "fromIncluding": 0,    "priceExVat": 1185, "priceIncVat": 1481.25 },
      { "fromIncluding": 100,  "priceExVat": 1141, "priceIncVat": 1426.25 },
      { "fromIncluding": 250,  "priceExVat": 1094, "priceIncVat": 1367.50 },
      { "fromIncluding": 500,  "priceExVat": 1045, "priceIncVat": 1306.25 },
      { "fromIncluding": 1000, "priceExVat": 995,  "priceIncVat": 1243.75 },
      { "fromIncluding": 2500, "priceExVat": 944,  "priceIncVat": 1180.00 }
    ]
  },
  "peakIdentificationSettings": {
    "peakFunction": "peak(main)",
    "peakIdentificationPeriod": "P1Y",
    "peakDuration": "P1D",
    "numberOfPeaksForAverageCalculation": 3
  }
}
```

With the cost function `sum(peak(c)*price(c))`, a peak of 180 kW costs 180 × 1 141 = 205 380 SEK per year excluding VAT.

This example and the stepped fixed price above are the two columns of the same price list, and they are designed to be used together.
On its own the power cost drops at a threshold (99.9 kW costs about 118 380 SEK, 100 kW costs 114 100 SEK), but the fixed fee steps up by the same amount, so the sum of the two is continuous: 129 370 SEK at 100 kW from either side.
The complete tariff is available in [samples/tiered-tariffs-response.json](samples/tiered-tariffs-response.json).

#### Example: graduated tiered energy price
An energy price where each part of the yearly volume is priced at the rate of its own tier. The numbers are invented for illustration.

```json
{
  "name": "Heat energy price",
  "type": "fixed",
  "reference": "main",
  "validPeriod": { "fromIncluding": "2026-01-01", "toExcluding": "2027-01-01" },
  "unit": "MWh",
  "price": {
    "currency": "SEK",
    "tierUnit": "MWh",
    "tierMode": "graduated",
    "tierPeriod": "P1Y",
    "tiers": [
      { "fromIncluding": 0,   "priceExVat": 600, "priceIncVat": 750.00 },
      { "fromIncluding": 50,  "priceExVat": 550, "priceIncVat": 687.50 },
      { "fromIncluding": 200, "priceExVat": 500, "priceIncVat": 625.00 }
    ]
  }
}
```

A facility that uses 120 MWh during the year pays 50 × 600 + 70 × 550 = 68 500 SEK excluding VAT, which makes `price(c)` 68 500 / 120 ≈ 570.83 SEK/MWh.
With `tierMode` `volume`, the same tiers would give 120 × 550 = 66 000 SEK.

### PeakFunction
The `peakFunction` property of the `PeakIdentificationSettings` object is a pseudo code
string that explains how to calculate the heat power peak for a component in the `PowerPrice` object.

#### Methods
|Method|Description|
|-|-|
|`peak(r)`|The heat power value of the power peak for the `recurringPeriod` with `reference` name (`r`).|
|`max(peak(a),peak(b))`|Get the highest value of the result of two different peak functions.|

#### Examples
|Example|Description|
|-|-|
|`peak(main)`|Gets the peak of the `recurringPeriod` referenced as `main`. This is used when there is only one single `recurringPeriod` entry.|
|`max(peak(high),peak(low)/2)`|Selects the heat power peak from the `low` time period if the value is more than double the value of that of the peak from the `high` time period.|

## Why a unified EfficiencyPrice instead of separate volume/temperature components?

District heating operators price a customer's heat-exchange efficiency using one of two proxies, and both express the same physics: `energy = volume × specific heat × temperature differential`. Given how much energy a facility used, its water volume and its temperature differential are two ways of measuring the same thing, so an operator picks one or the other (or, in principle, both):

- **`measurementMethod: "volume"`** — price per m³ of heating medium delivered, independent of temperature (appears on price lists as "flödesavgift" or "volympris"). A flow-only meter is enough to bill this.
- **`measurementMethod: "returnTemperature"`** — price per MWh delivered multiplied by the deviation between the facility's return temperature and a reference (e.g. the district network's system-average return temperature that month), which can be a rebate or a surcharge. Requires temperature sensors, not just a flow meter, but doesn't require knowing the volume at all.

For example, Göteborg Energi's business tariff uses the `returnTemperature` method: it compares each facility's monthly return temperature against the network's system average and applies ±7 kr/(MWh·°C), only during the heating season (October–April) — and has no volume-based charge at all. Its private/residential tariff has neither: only a fixed nätavgift and a seasonally-varying energy price. Both are valid `Tariff` objects in this API; a residential tariff simply has an empty `efficiencyPrice.components` array.

Because both proxies are structured and time-varied the same way as `energyPrice` and `powerPrice`, they share one wrapper (`EfficiencyPrice`) and one component array — an operator that bills strictly by volume never touches `returnTemperatureSettings`, and vice versa.
