# Načrt eksperimenta: Primerjava strategij spreminjanja shem v sistemih Event Sourcing

## 1. Raziskovalna vprašanja

- **RQ1:** Kakšna je cena posamezne strategije v osnovnem scenariju?
- **RQ2:** Kako različni dejavniki, kot so dolžina zgodovine dogodkov, porazdelitev dostopa, uporaba posnetkov in kompleksnost transformacij, vplivajo na učinkovitost strategij?
- **RQ3:** Kje je točka preloma med strategijami na branju in na pisanju? Ali meritve potrjujejo priporočilo iz Overeem et al. (2021), da se začne z versioned events in weak schema, upcasting in copy-and-transform pa pride kasneje?

## 2. Strategije

### S1 – Weak schema

Dogodki uporabljajo prilagodljivo shemo, ki dopušča dodajanje novih atributov in ignoriranje neznanih atributov. Starejši dogodki lahko ostanejo nespremenjeni, bralci pa manjkajoče atribute obravnavajo z uporabo privzetih vrednosti.

Strategija ne zahteva eksplicitne transformacije zgodovinskih dogodkov ob dodajanju neobveznih atributov.

Omejitve:
- Preimenovanje atributov zahteva dodatno združljivostno logiko.
- Razcep in združevanje dogodkov ni podprto.

### S2 – Versioned Events

Vsaka sprememba sheme ustvari novo verzijo dogodka. Zgodovinski dogodki ohranijo svojo prvotno obliko, aplikacija pa vsebuje ločene handlerje za posamezne verzije.

Strategija ne zahteva transformacije zgodovinskih dogodkov ob branju, vendar število potrebnih handlerjev narašča s številom verzij.

Omejitve:
- Logika rekonstrukcije mora podpirati vse uporabljene verzije, kompleksnost implementacije narašča s številom verzij.

### S3 – Upcasting

Zgodovinski dogodki se ob branju transformirajo v trenutno verzijo sheme. Transformacije so organizirane v verigo, pri čemer posamezen transformacijski korak pretvori dogodek iz ene verzije v naslednjo.

Zgodovinski dogodki ostanejo v shrambi nespremenjeni.

Omejitve:
- Strošek branja narašča z dolžino verige transformacij.

### S4 – Lazy-transformation

Dogodki oziroma agregati se transformirajo šele ob prvem dostopu. Transformirani podatki se shranijo, tako da naslednji dostopi ne zahtevajo ponovne transformacije.

Strategija se izogne transformaciji neuporabljenih agregatov, vendar povzroči dodatno zakasnitev ob prvem dostopu.

Omejitve:
- Prvo branje agregata je počasnejše.
- Potrebna je evidenca že transformiranih podatkov.

### S5 – Copy-and-transform

Dogodki se preberejo iz obstoječe shrambe, transformirajo in zapišejo v novo shrambo. Po uspešni migraciji se branje preusmeri na novo shrambo.

Omejitve:
- Začetni strošek migracije je višji.
- Med migracijo je potrebnega dodatnega prostora.

---

## 3 Začetna konfiguracija

Osnovna konfiguracija eksperimenta:

| Parameter | Vrednost |
|---|---:|
| Število agregatov | 1.000 |
| Število dogodkov na agregat | 100 |
| Skupno število dogodkov | 100.000 |
| Število verzij sheme | 5 |
| Format dogodkov | JSON |
| Število ponovitev |  10 + 1 (ogrevanje) |
| Porazdelitev dostopa | Enakomerna in Zipf (s = 1) |

Za eksperimente, ki preučujejo vpliv dolžine toka dogodkov, se bo število dogodkov spreminjalo neodvisno od osnovne konfiguracije.

Dogodki bodo predstavljeni v enotnem formatu JSON. Transformacije bodo izvedene nad deserializiranim JSON-drevesom, pri čemer bodo vse strategije uporabljale enak postopek deserializacije.

---

## 4. Eksperimentalni faktorji in nivoji

### 4.1 Dolžina toka dogodkov

Faktor določa število dogodkov, ki jih je treba obdelati pri rekonstrukciji agregata.

| Nivo | Število dogodkov na agregat |
|---|---:|
| E1 | 100 |
| E2 | 500 |
| E3 | 1.000 |
| E4 | 5.000 |
| E5 | 10.000 |

Ta faktor se uporablja predvsem pri H1 in H4.

### 4.2 Dolžina verige transformacij

Določa število zaporednih transformacij, potrebnih za pretvorbo dogodka v trenutno verzijo sheme.

| Nivo | Število transformacij |
|---|---:|
| V1 | 0 |
| V2 | 3 |
| V3 | 10 |

Nivo V1 predstavlja osnovni primer brez upcastinga. Vsi dogodki so že v trenutni verziji sheme.

### 4.3 Porazdelitev dostopa do agregatov

Uporabljeni bosta dve osnovni porazdelitvi.

Enakomerna porazdelitev (Uniform): Vsak agregat ima enako verjetnost dostopa

Zipfova porazdelitev: Dostopi so neenakomerno porazdeljeni, pri čemer imajo agregati z nižjim rangom večjo verjetnost dostopa.

### 4.4 Razmerje med številom dostopov in številom agregatov

Ta faktor določa, kolikokrat v povprečju dostopamo do agregatov glede na njihovo skupno število.

R = # dostopov / # agregatov

Pri 10.000 agregatih uporabimo naslednje nivoje:

| Nivo | Število dostopov | Razmerje R |
|---|---:|---:|
| R1 | 1.000 | 0,1 |
| R2 | 10.000 | 1 |
| R3 | 50.000 | 5 |
| R4 | 100.000 | 10 |
| R5 | 1.000.000 | 100 |

Pri razmerju R = 0,1 bo dostopano le do manjšega dela agregatov. Pri R = 1 bo število dostopov enako številu agregatov, pri višjih razmerjih pa bo veliko agregatov dostopanih večkrat.

Ta faktor je posebej pomemben za primerjavo lazy transformation z ostalimi strategijami.

### 4.5 Interval posnetkov (Snapshots)

Posnetki omogočajo rekonstrukcijo agregata iz njegovega zadnjega shranjenega stanja in samo dogodkov, ki so nastali po njem.

| Nivo | Interval posnetkov |
|---|---:|
| S0 | Brez posnetkov |
| S1 | Vsakih 100 dogodkov |
| S2 | Vsakih 500 dogodkov |
| S3 | Vsakih 1.000 dogodkov |

Ob spremembi sheme stanja agregata se stari posnetki zavržejo in ponovno zgradijo.

### 4.6 Vrsta transformacije sheme

Za preverjanje vpliva kompleksnosti spremembe sheme bodo uporabljeni naslednji primeri:

| Nivo | Vrsta spremembe | Primer |
|---|---|---|
| T1 | Dodajanje atributa | Dodajanje atributa `email` |
| T2 | Preimenovanje | `firstName` → `givenName` |
| T3 | Razcep (1→N) | `AddressChanged` → `StreetChanged` + `CityChanged` |
| T4 | Združevanje (N→1) | `FirstNameChanged` + `LastNameChanged` → `FullNameChanged` |

Pri združevanju se uporabijo različne velikosti medpomnilnika oziroma transformacijskega okna:

| Nivo | Velikost okna |
|---|---:|
| W1 | 2 dogodka |
| W2 | 4 dogodki |
| W3 | 8 dogodkov |
| W4 | 16 dogodkov |

### 4.7 Število verzij dogodkov

Ta faktor se uporablja pri primerjavi weak schema in versioned events.

| Nivo | Število verzij |
|---|---:|
| G1 | 1 |
| G3 | 3 |
| G5 | 10 |

Pri versioned events bo vsaka verzija zahtevala ustrezen handler. Pri weak schema bo preverjena združljivost med različnimi oblikami dogodkov.

---

## 5. Metrike

### 5.1 Časovne metrike

- Čas rekonstrukcije agregata (vse strategije)
- Čas obnove projekcije (vse strategije)
- Čas migracije (vse strategije)
- Čas deserializacije (vse strategije)
- Čas transformacije (lazy-trnsformation, copy-and-transform)
- Čas prvega branja (lazy-transformation)
- P95 zakasnitve (vse strategije)
- Čas na dogodek (vse strategije)
- Čas na transformacijski korak (upversioning)

### 5.2 Prostorske metrike

- Velikost shrambe (vse strategije)
- Dodatna poraba prostora (copy-and-transform, lazy-transformation)
- Največja poraba prostora (vse strategije)
- Poraba pomnilnika (vse strategije)
- Velikost medpomnilnika (upcasting ((N→1)))

### 5.3 Metrike števila operacij

- Število transformiranih dogodkov (lazy-transformation)
- Število transformiranih agregatov (lazy-transformation)
- Odstotek transformiranih agregatov (lazy-transformation)
- Število dodatnih zapisov (lazy-transformation)
- Število branj iz shrambe

- Število ustvarjenih posnetkov

### 5.4 Metrike kompleksnosti implementacije

- Število vrstic kode, namenjenih strategiji
- Število implementiranih handlerjev
- Število dodatnih podatkovnih struktur
- Število posebnih primerov, ki jih mora implementacija obravnavati

Te metrike bodo uporabljene kot dopolnilna ocena kompleksnosti implementacije in ne kot neposredno merilo zmogljivosti.

---

## 6. Hipoteze

### H1 – Strošek upcastinga

**Raziskovalno vprašanje: RQ2**

Upcasting povečuje čas rekonstrukcije agregata sorazmerno s številom dogodkov in dolžino verige transformacij. Pri enakem številu dogodkov je dodatna zakasnitev približno linearno odvisna od števila transformacijskih korakov. Predvidevam, da posamezen transformacijski korak povzroči dodatno zakasnitev med 0,1 in 1 µs na dogodek.

Upcaster dela nad deserializiranim objektom. Čas deserializacije je izmerjen ločeno od časa transformacije.

**Metrike:**
- Čas rekonstrukcije brez upcastinga.
- Čas rekonstrukcije z upcastingom.
- Dodatni čas rekonstrukcije.
- Dodatni čas na dogodek.
- Dodatni čas na transformacijski korak.
- Čas deserializacije, merjen ločeno.

**Odločitveno pravilo:**

Hipoteza je podprta, če:
1. Linearni regresijski model med številom transformacijskih korakov in dodatnim časom rekonstrukcije doseže R^2 > 0,90.

Osnovni primer predstavlja rekonstrukcija toka, v katerem so vsi dogodki že v zadnji verziji sheme.

### H2 – Točka preloma

**Raziskovalno vprašanje: RQ3**

Copy-and-transform ima zaradi začetne migracije višji začetni strošek kot upcasting, vendar omogoča hitrejše nadaljnje branje. Po določenem številu polnih rekonstrukcij agregatov postane njegov skupni strošek nižji. Točka preloma se zmanjšuje z večanjem dolžine verige transformacij.

**Metrike:**
- Čas začetne migracije.
- Čas rekonstrukcije z upcastingom.
- Čas rekonstrukcije po migraciji.
- Skupni čas po več ponovitvah.
- Dodatna poraba prostora.
- Število rekonstrukcij do točke preloma.

**Odločitveno pravilo:**

Najprej se iz posameznih komponent izračuna napovedana točka preloma:

k = (cena migracije na dogodek) / (dodatna cena upcastinga na dogodek pri enem predvajanju)

Nato se napoved preveri z dejanskimi ponovitvami rekonstrukcije.

Hipoteza je podprta, če:
1. Je začetni strošek copy-and-transform višji od začetnega stroška upcastinga.
2. Se točka preloma zmanjšuje z večanjem dolžine verige transformacij.

Med migracijo se dodatno spremlja začasna poraba prostora, ki je okoli 2x večja.

Polno predvajanje je definirano kot rekonstrukcija vseh agregatov iz celotne zgodovine njihovih dogodkov.

### H3 – Vpliv porazdelitve dostopa na lazy transformation

**Raziskovalno vprašanje: RQ2**

Pri neenakomerni porazdelitvi dostopa do agregatov (Zipfova porazdelitev) ima lazy transformation nižji skupni strošek kot copy-and-transform in upcasting, saj se transformirajo samo agregati, do katerih dejansko dostopamo. Pri enakomerni porazdelitvi dostopa se njegova prednost zmanjša ali izgine.

**Metrike:**
- Skupni čas izvajanja.
- Čas začetne migracije.
- Čas posameznega branja.
- P95 zakasnitve prvega branja agregata.
- Število in odstotek transformiranih agregatov.
- Število transformiranih dogodkov.
- Število dodatnih zapisov.
- Dodatna poraba prostora.

**Odločitveno pravilo:**

Skupni strošek je definiran kot:

čas migracije + vsota časov vseh branj + čas dodatnih pisanj

Primerjajo se vse tri strategije:
- Upcasting.
- Copy-and-transform.
- Lazy transformation.

Uporabita se enakomerna porazdelitev in Zipfova porazdelitev s parametrom s=1, pri različnih razmerjih števila dostopov proti številu agregatov.

Hipoteza je podprta, če:
1. Lazy transformation doseže nižji skupni strošek kot preostali strategiji pri Zipfovi porazdelitvi.
2. Se njena prednost zmanjša pri enakomerni porazdelitvi.
3. Se delež transformiranih agregatov pri neenakomerni porazdelitvi razlikuje od deleža pri enakomerni porazdelitvi.

Posebej se preveri vpliv razmerja med številom dostopov in številom agregatov. Pri velikem številu dostopov se postopoma transformira vedno več agregatov, zaradi česar lahko prednost lazy transformation izgine.

### H4 – Vpliv posnetkov

**Raziskovalno vprašanje: RQ2**

Uporaba posnetkov zmanjša razliko v času rekonstrukcije agregatov med upcastingom in copy-and-transform, saj je treba obdelati samo dogodke, ki nastanejo po zadnjem posnetku.

Posnetki ne odpravijo potrebe po obdelavi celotne zgodovine dogodkov pri obnovi projekcije, zato pričakujem bistveno manjši vpliv na čas obnove projekcije.

**Metrike:**
- Čas rekonstrukcije agregata brez in z posnetki.
- Čas obnove projekcije.
- Število dogodkov v agregatu.
- Število dogodkov po zadnjem posnetku.
- Čas gradnje posnetka.
- Čas invalidacije in ponovne izgradnje posnetka.

**Odločitveno pravilo:**

Primerjata se upcasting in copy-and-transform pri naslednjih konfiguracijah:
- Brez posnetkov.
- Posnetek na vsakih 100 dogodkov.
- Posnetek na vsakih 500 dogodkov.
- Posnetek na vsakih 1.000 dogodkov.

Hipoteza je podprta, če:
1. Posnetki zmanjšajo razliko v času rekonstrukcije agregata med strategijama.
2. Posnetki ne povzročijo primerljivega zmanjšanja časa obnove projekcije.

Ob spremembi sheme stanja agregata se stari posnetki zavržejo in ponovno zgradijo.

### H5 – Razcep in združevanje dogodkov

**Raziskovalno vprašanje: RQ2**

Pri razcepu dogodka (1→N) lahko upcasting izvede transformacijo brez ohranjanja dodatnega stanja med dogodki.

Pri združevanju dogodkov (N→1) mora upcaster ohranjati dodatno stanje, kar poveča porabo pomnilnika in lahko povzroči dodatno zakasnitev. Pričakujemo, da bo copy-and-transform pri združevanju dogodkov učinkovitejši z vidika zakasnitve rekonstrukcije in dodatnega delovnega pomnilnika.

**Metrike:**
- Čas transformacije.
- Čas rekonstrukcije agregata.
- Največja poraba pomnilnika med transformacijo.
- Velikost medpomnilnika.
- Čas začetne migracije.
- Kompleksnost implementacije.

**Odločitveno pravilo:**

Pri združevanju se spreminja velikost transformacijskega okna (2, 4, 8 in 16 dogodkov).

Hipoteza je podprta, če:
2. Združevanje povzroči merljivo povečanje porabe pomnilnika pri upcastingu.
3. Copy-and-transform pri združevanju doseže nižjo zakasnitev rekonstrukcije ali nižjo porabo pomnilnika kot upcasting.

Rezultati se interpretirajo ločeno za zakasnitev in porabo pomnilnika, saj copy-and-transform ni nujno ugodnejši pri obeh metrikah.

### H6 – Weak schema in versioned events

**Raziskovalno vprašanje: RQ1**

Weak schema ima pri dodajanju atributa zanemarljivo dodatno ceno ob branju, preimenovanja, razcepa in združevanja pa ne podpira. Versioned events nima pretvorbe ob branju, zato se kompleksnost kode (število handlerjev) linearno povečuje s številom verzij.

**Metrike:**
- Čas branja dogodkov.
- Čas rekonstrukcije agregatov.
- Čas deserializacije.
- Število implementiranih handlerjev.
- Kompleksnost kode.
- Podpora posameznim vrstam sprememb sheme.

**Odločitveno pravilo:**

Primerjajo se različne vrste sprememb:
1. Dodajanje atributa.
2. Preimenovanje atributa.
3. Razcep dogodka (1→N).
4. Združevanje dogodkov (N→1).

Za versioned events se spreminja tudi število verzij (1, 2, 3, 5 in 10).

Hipoteza je podprta, če:
1. Dodajanje neobveznega atributa pri weak schema ne zahteva transformacije zgodovinskih dogodkov in povzroči zanemarljiv dodatni čas branja glede na osnovni primer.
3. Versioned events omogočajo branje brez transformacije zgodovinskih dogodkov.
4. Število potrebnih handlerjev pri versioned events linearno narašča s številom podprtih verzij.

---

## 7. Domena - Bančni računi


| Faktor | izbira  |
|---|---|
|Domena	 |Bančni računi |
|Agregat |BankAccount |
|Identifikator agregata |	accountId |
|Format dogodkov | JSON |
|Skupno |eventId, accountId, eventType, version, timestamp, data |
|T1 | Dodajanje ownerEmail |
|T2 | Preimenovanje ownerId v customerId |
|T3 | AccountContactDetailsChanged → dva dogodka |
|T4 | Dva dogodka spremembe imena → AccountHolderNameChanged |