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
|`price(c)`|The price of a component.|
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
