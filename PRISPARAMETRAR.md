# Prisparametrar i fjärrvärmetariffer

Det här dokumentet beskriver de fyra pristyper som en tariff i denna specifikation kan bestå av, och knyter varje pristyp till konkreta exempel från Göteborg Energis publicerade priser för 2026 (privatkund respektive företagskund). Det beskriver också var i tidsstandarden ISO 8601-1 de datum-, tid- och tidslängdsformat som används i specifikationen är definierade, för den som vill verifiera detaljerna själv.

Målgrupp: den som är van vid att arbeta med prisparametrar och tariffkonstruktion, men inte nödvändigtvis med JSON eller tekniska filformat. De korta kodexemplen är enbart illustrativa - poängen är innehållet i fälten, inte syntaxen.

## Sammanfattning

En tariff byggs upp av fyra prisdelar. De två första är gemensamma med elnätstariffer, den tredje är gemensam till strukturen men fjärrvärmespecifik i sina siffror, och den fjärde är unik för fjärrvärme:

| Pristyp | Svenskt begrepp | Enhet | Måste alltid finnas? |
|---|---|---|---|
| `fixedPrice` | Fast avgift / nätavgift | kr per period | Ja |
| `energyPrice` | Energipris | kr/MWh eller kr/kWh | Ja |
| `powerPrice` | Effektavgift | kr/kW | Ja, men kan vara tom |
| `efficiencyPrice` | Effektivitetspris (volym- eller returtemperaturbaserat) | kr/m³ eller kr/(MWh·°C) | Ja, men kan vara tom |

Att en pristyp "kan vara tom" betyder att fältet alltid finns med i datan, men att listan av faktiska priskomponenter i den kan vara tom - det är så man uttrycker att en viss tariff inte har någon effektavgift eller effektivitetsavgift, utan att behöva utelämna hela fältet.

Göteborg Energis egna priser illustrerar hela spannet: privatkundens tariff använder bara de två första pristyperna, medan företagskundens tariff använder alla fyra.

## 1. Fast avgift (`fixedPrice`)

Den fasta avgiften är en periodisk avgift som inte beror på hur mycket värme eller effekt som faktiskt förbrukas - typiskt en årsavgift eller månadsavgift för att vara ansluten till nätet.

**Exempel, privatkund (Göteborg Energi, normalprislista 2026):**
Nätavgift 374 kr/månad för avtalsformen "Äga" (345 kr/månad för "Hyra"). Avgiften är densamma oavsett förbrukning och effektbehov.

**Exempel, företagskund:**
En fast årsavgift som beror på vilket effektintervall (kW-band) kunden abonnerar på, t.ex. 10 870 kr/år för bandet 0-100 kW. Ju högre band, desto högre fast avgift (se effektavgift nedan för hela tabellen) - men inom ett och samma band är avgiften konstant.

I specifikationen anges varje sådan avgift med ett pris, en giltighetsperiod (`validPeriod`) och en period den gäller för (`pricedPeriod`, t.ex. "en månad" eller "ett år"). Flera fasta avgifter kan förekomma i samma tariff om olika delar av avgiften gäller olika tidsperioder.

Priset är antingen ett enda belopp, som privatkundens nätavgift, eller ett stafflat pris med ett belopp per effektintervall, som företagskundens årsavgift. Se avsnittet [Stafflade priser](#stafflade-priser) nedan.

## 2. Energipris (`energyPrice`)

Energipriset är priset per levererad energienhet (kWh eller MWh) värme. Det är den del av notan som direkt speglar hur mycket värme kunden faktiskt förbrukat.

**Exempel, privatkund:**
Från och med 2026 har Göteborg Energi ett säsongsdifferentierat energipris för villakunder:
- Sommar (juni-augusti): 13 öre/kWh
- Mellansäsong (maj, september): 36 öre/kWh
- Vinter (oktober-april): 94 öre/kWh

**Exempel, företagskund:**
Företagskundens energipris varierar månad för månad, inte bara mellan tre säsonger. Två bekräftade exempel: 531 kr/MWh i januari, februari och december (kallaste, dyraste månaderna), och 46 kr/MWh i juli (då spillvärme/återvunnen värme gör energin billig). Övriga månader ligger däremellan enligt Göteborg Energis fullständiga prislista.

Att uttrycka detta i specifikationen kräver inget särskilt stöd för "säsonger" - varje period med ett eget pris (en månad, en säsong, eller vad prislistan råkar dela in året i) blir helt enkelt en egen priskomponent med sin egen giltighetsperiod. En tariff med tolv olika månadspriser har alltså tolv energipris-komponenter; en tariff med tre säsonger har tre.

## 3. Effektavgift (`powerPrice`)

Effektavgiften debiterar kundens toppeffekt (den högsta värmeeffekt kunden drar) snarare än den totala energimängden. Motivet är att fjärrvärmenätet måste dimensioneras för att klara toppar, även om de bara inträffar enstaka gånger under de kallaste dagarna - därför prissätts toppeffekten separat.

Göteborg Energi definierar en kunds relevanta toppeffekt som **genomsnittet av de tre högsta dygnsmedeleffekterna under de senaste tolv månaderna** (ett rullande år). Effektavgiften för företag har dessutom en bandstruktur: både en fast årsavgift och en rörlig avgift per kW beror på vilket effektintervall kunden ligger i:

| Effektintervall | Fast avgift (kr/år) | Rörlig avgift (kr/kW/år) |
|---|---|---|
| 0-100 kW | 10 870 | 1 185 |
| 100-250 kW | 15 270 | 1 141 |
| 250-500 kW | 27 020 | 1 094 |
| 500-1 000 kW | 51 520 | 1 045 |
| 1 000-2 500 kW | 101 520 | 995 |
| > 2 500 kW | 229 020 | 944 |

Den rörliga delen (kr/kW/år) hör hemma i `powerPrice`, medan den fasta delen hör hemma i `fixedPrice` (se avsnitt 1) - de två kolumnerna i tabellen ovan blir alltså två olika priskomponenter i specifikationen, trots att de kommer från samma prislista.

Båda kolumnerna är stafflade priser: priset beror på vilket effektintervall kundens toppeffekt hamnar i. Hela tabellen kan därför beskrivas i en enda tariff, med en priskomponent per kolumn, i stället för en tariff per effektintervall. Hur det går till beskrivs i avsnittet [Stafflade priser](#stafflade-priser) nedan.

Villakunder hos Göteborg Energi har ingen effektavgift alls - för dem är `powerPrice` en tom lista av komponenter.

## 4. Effektivitetspris (`efficiencyPrice`)

Det här är den pristyp som skiljer fjärrvärme från elnät, och den finns i två varianter som mäter samma sak på olika sätt.

### Bakgrund: varför "effektivitet" prissätts alls

Ju mer en fastighets värmesystem lyckas kyla av fjärrvärmevattnet innan det skickas tillbaka till nätet (dvs. ju större temperaturskillnad mellan inkommande och utgående vatten, "returtemperaturen"), desto mer energi har hämtats ut per kubikmeter cirkulerande vatten. En fastighet som skickar tillbaka för varmt vatten (hög returtemperatur) är ineffektiv - den tvingar fjärrvärmenätet att pumpa runt mer vatten för att leverera samma mängd energi, vilket belastar hela systemets kapacitet.

Sambandet är: **energi = volym × specifik värmekapacitet × temperaturskillnad**. Om man vet hur mycket energi som levererats finns det alltså två likvärdiga sätt att mäta en fastighets effektivitet - antingen genom att mäta volymen direkt, eller genom att mäta temperaturskillnaden. Det är därför de modelleras som samma pristyp i specifikationen, med ett fält (`measurementMethod`) som anger vilken av de två metoderna en viss priskomponent använder:

**a) Volymbaserad ("flödesavgift"/"volympris"), `measurementMethod: "volume"`**
Ett pris per kubikmeter (m³) fjärrvärmevatten som levererats, oavsett temperaturskillnad. Kräver bara en flödesmätare.

**b) Returtemperaturbaserad, `measurementMethod: "returnTemperature"`**
Ett pris per grad (°C) som fastighetens returtemperatur avviker från en referenstemperatur, multiplicerat med den levererade energimängden - anges alltså i kr per (MWh·°C). Kräver temperaturgivare, men ingen volymmätning.

**Exempel, företagskund (Göteborg Energi):**
Göteborg Energi använder metod (b). Varje fastighets returtemperatur jämförs månadsvis med fjärrvärmenätets genomsnittliga returtemperatur för samma månad. Skillnaden prissätts till **7 kr per MWh och grad**, och gäller bara under uppvärmningssäsongen (oktober-april):
- En fastighet som ligger 5 °C **under** nätets medelvärde får en **rabatt** på 5 × 7 = 35 kr/MWh.
- En fastighet som ligger 5 °C **över** nätets medelvärde får ett **tillägg** på 35 kr/MWh.

Konstruktionen är alltså intäktsneutral för Göteborg Energi i grunden - den flyttar kostnad mellan kunder utifrån deras faktiska effektivitet, snarare än att vara en ren intäktskälla.

**Göteborg Energis privatkunder har varken volym- eller returtemperaturbaserad avgift** - för dem är `efficiencyPrice` en tom lista av komponenter, precis som `powerPrice`.

### Varför inte två separata pristyper?

Eftersom volym och returtemperaturavvikelse är två sätt att mäta samma underliggande effektivitet, och en given fjärrvärmeleverantör normalt bara använder den ena metoden (aldrig båda samtidigt för samma kund), är det både enklare och mer korrekt att modellera dem som en gemensam pristyp med en metodangivelse, snarare än att tvinga varje implementation att ta ställning till två helt separata fält där bara ett någonsin är relevant.

## Stafflade priser

Ett stafflat pris är ett pris som beror på hur stor en viss mängd är, till exempel kundens toppeffekt eller årets energivolym. Mängden delas in i nivåer, och varje nivå har sitt eget pris. Det är inte en femte pristyp, utan ett sätt att ange priset i en fast avgift, ett energipris eller en effektavgift. Ett pris som inte är stafflat anges precis som tidigare, med ett enda belopp.

Ett stafflat pris anges med fyra uppgifter:

| Uppgift | Fält | Innebörd |
|---|---|---|
| Enhet | `tierUnit` | Enheten för nivåernas gränser, t.ex. kW eller MWh. |
| Underlag | `tierBasis` | Vilken mängd som avgör nivån, t.ex. toppeffekten i effektavgiften. Kan utelämnas när det är komponentens egen mängd som avses. |
| Beräkningssätt | `tierMode` | Hur nivåns pris används, se nedan. |
| Nivåer | `tiers` | En rad per nivå, med nivåns nedre gräns (`fromIncluding`) och dess pris. |

För en mängd som summeras över tid, som energi, anges dessutom vilken period mängden räknas över (`tierPeriod`, t.ex. ett år).

### Nivåernas gränser

Varje nivå anges bara med sin nedre gräns. Nivån gäller från och med den gränsen fram till nästa nivås gräns, och den sista nivån saknar övre gräns. Den första nivån börjar alltid på 0. På så sätt hamnar varje mängd i exakt en nivå, utan luckor och utan överlapp.

En mängd som ligger precis på en gräns tillhör den övre nivån. Med prislistans intervall "0-100 kW" och "100-250 kW" räknas alltså exakt 100 kW till nivån 100-250 kW.

### Tre beräkningssätt

Det som skiljer stafflade priser åt är vad nivåns pris betyder. Specifikationen har tre beräkningssätt:

| `tierMode` | Nivåns pris är | Kostnaden blir |
|---|---|---|
| `stepped` | Ett belopp för hela perioden | Beloppet för den nivå mängden hamnar i. Kostnaden är densamma inom hela nivån och ändras språngvis vid varje gräns, som en trappa. |
| `volume` | Ett pris per enhet | Hela mängden gånger priset för den nivå mängden hamnar i. |
| `graduated` | Ett pris per enhet | Varje del av mängden gånger priset för sin egen nivå. Kostnaden växer utan språng, men lutningen ändras vid varje gräns. |

### Exempel: Göteborg Energis effekttabell

Tabellen i avsnitt 3 innehåller två stafflade priser som båda styrs av samma toppeffekt:

- **Fast avgift (kr/år)** är ett stafflat pris med beräkningssättet `stepped` i `fixedPrice`. Underlaget är toppeffekten i effektavgiften.
- **Rörlig avgift (kr/kW/år)** är ett stafflat pris med beräkningssättet `volume` i `powerPrice`. Hela toppeffekten prissätts med priset för den nivå den hamnar i.

Den fasta avgiften ser ut så här i datan (den rörliga är uppbyggd på samma sätt, med `volume` och kolumnens priser per kW):

```json
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
```

**Räkneexempel:** en kund med toppeffekten 180 kW hamnar i nivån 100-250 kW.

- Fast avgift: 15 270 kr/år.
- Rörlig avgift: 180 × 1 141 = 205 380 kr/år.
- Summa: 220 650 kr/år exklusive moms.

De två kolumnerna hör ihop. Den rörliga avgiften för sig sjunker vid en gräns: strax under 100 kW kostar den omkring 118 400 kr, vid 100 kW 114 100 kr. Den fasta avgiften stiger samtidigt med lika mycket, så summan av de två blir densamma på båda sidor om gränsen: 129 370 kr vid 100 kW. Den totala kostnaden växer alltså utan språng, med en lägre kostnad per ytterligare kW ju högre effekten är.

Hela tariffen finns som exempelfil i `samples/tiered-tariffs-response.json`.

### Exempel: energipris där varje del prissätts för sig

Beräkningssättet `graduated` förekommer inte i Göteborg Energis prislista. Siffrorna nedan är påhittade för att visa skillnaden mot `volume`:

| Årsvolym | Energipris (kr/MWh) |
|---|---|
| 0-50 MWh | 600 |
| 50-200 MWh | 550 |
| > 200 MWh | 500 |

En kund som använder 120 MWh under året betalar:

- Med `graduated`: 50 × 600 + 70 × 550 = 68 500 kr. De första 50 MWh kostar 600 kr/MWh och de följande 70 MWh kostar 550 kr/MWh.
- Med `volume`: 120 × 550 = 66 000 kr. Hela volymen får priset för den nivå som 120 MWh hamnar i.

Samma tabell ger alltså olika kostnad beroende på beräkningssättet, och därför måste det alltid anges.

## Källor

Siffrorna ovan är hämtade från Göteborg Energis publikt tillgängliga sidor för 2026 års priser (hämtade 2026-08-31):

- [Fjärrvärmepriser, privat](https://www.goteborgenergi.se/privat/fjarrvarme/fjarrvarmepriser)
- [Fjärrvärmepriser (privat, prisöversikt)](https://www.goteborgenergi.se/privat/fjarrvarme/priser)
- [Fjärrvärmepriser, företag](https://www.goteborgenergi.se/foretag/fjarrvarme/fjarrvarmepriser)
- [Fjärrvärmepriser abonnerad effekt](https://www.goteborgenergi.se/foretag/fjarrvarme/fjarrvarmepriser/fjarrvarmepriser-abonnerad-effekt)
- [Så bestäms fjärrvärmepriset](https://www.goteborgenergi.se/foretag/fjarrvarme/sa-bestams-fjarrvarmepriset)
- [Fjärrvärme och vad som påverkar kostnaden](https://www.goteborgenergi.se/foretag/fjarrvarme/paverka-fjarrvarmekostnaden)
- [Prisändring för fjärrvärmen i Göteborg år 2026](https://goteborg.se/wps/PA_Pabolagshandlingar/file?id=57137) (Göteborgs Stads handling)

Observera att endast ett fåtal månaders energipris för företagskunder kunde bekräftas (januari, februari, december och juli) - se `samples/README.md` för motsvarande reservation kring exempelfilen.

---

# Datum-, tid- och tidslängdsformat (ISO 8601-1)

Specifikationens fält för datum, tidsstämplar och tidslängder följer [ISO 8601-1:2019](https://www.iso.org/standard/70907.html) (i Sverige fastställd som SS-ISO 8601-1:2022, tillgänglig som `resources/SS_ISO_8601_1_2022_EN.pdf` i det här projektet). Nedan anges exakt vilket klausulnummer i standarden som beskriver respektive format, för den som vill slå upp den fullständiga, formella definitionen.

## Datum (`fromIncluding`/`toExcluding` i giltighetsperioder, samt kalenderdatum i `calendarPatterns`)

Ett datum anges som kalenderdatum, t.ex. `2026-01-01`. Formatet är den "utökade" (extended) representationen av **kalenderdatum**, beskrivet i **klausul 5.2.2 "Calendar date"** (fullständig representation, exempel `1985-04-12`, se även Tabell A.1 i bilaga A). Notera att specifikationen alltid använder den utökade formen (med bindestreck), aldrig den kompakta "basic"-formen (`19850412`) som standarden också tillåter.

## Tidsstämplar med tidszon (t.ex. när en tariff publicerats, eller när prisdata senast uppdaterats)

Fält som anger en exakt tidpunkt - inte bara ett datum - kombinerar ett datum med en klockslagsdel och en tidszonsangivelse, t.ex. `2026-01-01T00:00:00+01:00`. Det här är den utökade formen av **"Date and time of day"**, beskriven i **klausul 5.4** (särskilt 5.4.2.1 "Complete representations for calendar date and time of day", exempel 7: `1985-04-12T23:20:30+04:00`). Själva tidszonsdelen (`+01:00`) - skillnaden mellan lokal tid och UTC - definieras separat i **klausul 5.3.4.1 "Time shift between local time scale and UTC"**.

Specifikationen kräver alltid att någon tidszonsangivelse finns med - antingen `Z` (UTC, klausul 5.3.3 "UTC of day") eller ett numeriskt offset i formen `±hh:mm` som ovan. Ett klockslag helt utan tidszonsangivelse (t.ex. bara `2026-01-01T00:00:00`) godkänns inte, eftersom det annars är omöjligt att veta om tiden avser svensk normaltid, sommartid eller UTC. I det här projektets egna exempel används genomgående det numeriska offsetet, eftersom det gör skiftet mellan svensk normaltid (`+01:00`) och sommartid (`+02:00`) synligt direkt i tidsstämpeln.

## Klockslag utan datum (start/slut på tidsintervall inom ett dygn, t.ex. "07:00" till "20:00" för högtrafiktid)

När bara ett klockslag behövs - utan datum och utan tidszon, eftersom det gäller varje dag inom en redan angiven period - används formatet från **klausul 5.3.1 "Local time of day"**, utökad form, t.ex. `07:00:00` (klausul 5.3.1.2, exempel 2).

## Tidslängder (t.ex. faktureringsperiod, hur länge ett fast pris gäller, eller hur ofta topplast mäts)

Fält som anger en *längd* av tid snarare än en tidpunkt - till exempel att en faktureringsperiod är en månad, eller att en topplast mäts över ett dygn - använder formatet **"Duration"**, beskrivet i **klausul 5.5.2**. En tidslängd inleds alltid med `P` ("period"), följt av siffror och bokstäver för respektive tidsenhet:

| Uttryck | Betydelse |
|---|---|
| `P1M` | En månad |
| `P1Y` | Ett år |
| `P1D` | Ett dygn |
| `PT1H` | En timme (`T` markerar att det som följer är klockenheter, inte kalenderenheter - se nedan) |
| `PT15M` | 15 minuter |

**Varför `T` ibland behövs:** Bokstaven `M` betyder "månad" om den kommer före ett eventuellt `T`, men "minut" om den kommer efter. `P1M` är alltså en månad, medan `PT1M` är en minut. Detta är rakt av definierat i klausul 5.5.2.2 i standarden, och är den vanligaste källan till förvirring vid handpåläggning av dessa värden.

Denna typ av tidslängd används i specifikationen bland annat för: `billingPeriod` (faktureringsperiod), `pricedPeriod` (vilken period ett fast pris gäller för, t.ex. ett år), `peakIdentificationPeriod` och `peakDuration` (hur topplasten mäts, se avsnitt om effektavgift ovan), samt `measurementPeriod` i det returtemperaturbaserade effektivitetspriset (t.ex. `P1M` för att ange att jämförelsen görs per kalendermånad).

## Giltighetsperioder (`validPeriod`) - en avvikelse värd att notera

ISO 8601-1 definierar även ett format för att uttrycka ett *tidsintervall* som en enda textsträng, med start och slut separerade av snedstreck, t.ex. `2018-01-15/2018-02-20` (**klausul 5.5, särskilt 5.5.1 "Means of specifying time intervals"** och **5.5.3.1**). Den här specifikationen använder **inte** den formen. Istället anges giltighetsperioder (`validPeriod`) alltid som två separata fält, `fromIncluding` och `toExcluding`, som var för sig är ett kalenderdatum enligt klausul 5.2.2 ovan. Innebörden är densamma - ett intervall som inkluderar startdatumet men inte slutdatumet - men uttrycket i datan ser alltså inte ut som standardens eget snedstreck-format. Den som är van vid att läsa ISO 8601-intervall som en sammanhängande sträng bör alltså inte leta efter ett `/`-tecken i den här specifikationen.

Motsvarande gäller de återkommande tidsperioder som beskriver när ett pris är aktivt under dygnet eller veckan (`recurringPeriods` i priskomponenterna, och `calendarPatterns` för veckodagar/helgdagar) - dessa är specifikationens egen konstruktion för att uttrycka mönster som "vardagar klockan 07-20", och motsvarar **inte** standardens formella **"Recurring time interval"** i klausul 5.6 (`R12/2026-01-01T00:00:00/P1M` och liknande). De bygger dock på samma grundformat för datum, klockslag och tidslängd som beskrivits ovan.
