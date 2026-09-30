# Tariff Response Samples

This directory contains sample JSON responses demonstrating district heating tariff structures supported by the District Heating Tariff API. The numbers are illustrative, drawn where possible from Göteborg Energi's published 2026 prices for private and business customers, but they are **not** a complete or guaranteed-accurate price list — see the operator's own published prices for that.

## Tariff Structure Examples

**[tariffs-response.json](tariffs-response.json)**
- **Företag 0-100 kW** (business): all four price components.
  - `fixedPrice`: annual fixed effect fee for the 0-100 kW subscribed-effect bracket
  - `energyPrice`: per-MWh heat energy price, differentiated by calendar month (only January, February, December, and July are populated with confirmed published rates — the rest are intentionally omitted rather than guessed)
  - `powerPrice`: effect fee based on the average of the three highest daily mean heat power values over the trailing 12 months
  - `efficiencyPrice`: a `returnTemperature`-method component — the facility's monthly return temperature is compared to the district network's system-average return temperature, priced per MWh·°C, applied only during the heating season (October-April)
- **Villa - Normalprislista (Äga)** (private/residential): only two components are used at all.
  - `fixedPrice`: fixed monthly nätavgift
  - `energyPrice`: per-kWh heat energy price, differentiated by season (summer/shoulder/winter)
  - `powerPrice` and `efficiencyPrice` are both present but empty - Göteborg Energi's residential tariff has no effect charge and no efficiency charge at all

## Why these two examples

Comparing a real business and private tariff from the same operator shows the range the spec needs to cover:
- Not every tariff uses every component - `powerPrice` and `efficiencyPrice` are commonly empty for residential products.
- `efficiencyPrice` components can use either `measurementMethod`: a facility that only has a flow meter would be billed by `volume` (SEK/m³), while one with temperature sensors - like Göteborg Energi's business customers - can be billed by `returnTemperature` (SEK/(MWh·°C)) instead. See [DEVELOPER_GUIDE.md](../DEVELOPER_GUIDE.md#why-a-unified-efficiencyprice-instead-of-separate-volumetemperature-components) for why both live in one component type.
- Energy prices can be differentiated far more finely than a simple summer/winter split (Göteborg Energi's business price list varies essentially every month) - this only requires adding more `EnergyPriceComponent` entries with narrower `validPeriod`s, no schema change.

## Common Elements Across Tariffs

- **Fixed prices**: subscription/network fee, billed monthly or annually
- **Energy prices**: price per unit of delivered heat energy (kWh for residential-scale, MWh for business-scale in these samples), which may vary by season or by calendar month
- **Power prices**: effect/demand charge based on peak heat power (kW) during a billing period - required on every tariff, but may be an empty component list
- **Efficiency prices**: district-heating-specific charge for how efficiently a facility uses the heating medium, either per m³ delivered or per °C of return-temperature deviation from a reference - required on every tariff, but may be an empty component list
- **Valid periods**: date ranges for price applicability
- **Billing periods**: typically monthly (`P1M`)
- **Time zones**: all samples use `Europe/Stockholm`
