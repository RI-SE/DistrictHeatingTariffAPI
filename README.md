# District Heating Tariff API
_Specification for an API for District Heating Tariffs in Sweden._

This project follows the same model as [Eltariff-API](https://github.com/RI-SE/Eltariff-API), adapted for the specific needs of district heating (fjärrvärme) — most notably an **efficiency price** component, in addition to the fixed, energy, and power/effect price components shared with the electricity grid tariff model. District heating operators price how efficiently a facility uses the heating medium (water) either per m³ delivered, or per degree of return-temperature deviation from a network reference — see [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md#why-a-unified-efficiencyprice-instead-of-separate-volumetemperature-components) for why both are modeled as one component.

__Please note that the provision of data in this API is in a development phase. It is likely not legally binding, used at your own risk, and does not entail any obligations for the district heating operators unless otherwise stated. For more information, contact the respective district heating company.__

# Documentation

The District Heating Tariff API is based on district heating operators publishing their tariffs according to a shared [API specification](specification/districtheatingtariffapi.json).

See [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) for how the tariff objects, cost functions, and peak functions work, and [samples/](samples/) for example tariff responses — including a business tariff that uses all four price components (fixed, energy, power, efficiency) modeled on Göteborg Energi's published prices, and a private/residential tariff that has no power or efficiency charge at all.

For a Swedish-language walkthrough of the price components aimed at tariff/pricing specialists (rather than developers) — with references to Göteborg Energi's actual 2026 prices and to the relevant ISO 8601-1 clauses for the date/time/duration formats used — see [PRISPARAMETRAR.md](PRISPARAMETRAR.md).

## Tariff components

| Component | Typical Swedish term | Unit | Required |
|---|---|---|---|
| `fixedPrice` | Fast avgift | SEK/period | Yes |
| `energyPrice` | Energipris | SEK/MWh | Yes |
| `powerPrice` | Effektavgift | SEK/kW | Yes (may be empty) |
| `efficiencyPrice` | Effektivitet / Volympris / Flödesavgift | SEK/m³ or SEK/(MWh·°C) | Yes (may be empty) |

# Contribute
Run the following commands to set up your dev environment

    npm install

## Validate the specification
    npx @redocly/cli lint specification/districtheatingtariffapi.json

## Bundle the specification (single-file OpenAPI document)
    npx @redocly/cli bundle specification/districtheatingtariffapi.json --output dist/openapi.json --ext json

## Preview the documentation
    npx @redocly/cli preview-docs specification/districtheatingtariffapi.json

# Status

The current focus of this project is the OpenAPI specification and JSON Schema definitions under [specification/](specification/). Reference server/client code (in the style of Eltariff-API's `ControllerGenerator`/`SwaggerUI` .NET projects) has not been set up yet and can be added once the specification stabilizes.
