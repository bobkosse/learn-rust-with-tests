# Aan de slag met Rust

Je gaat aan de slag om de programmeertaal Rust te leren. Er is eigenlijk maar één goede manier om een programmeertaal te leren: doen! Alleen een boek erover lezen gaat je niet helpen. Net als bij autorijden moet je echt oefenen, proberen en fouten maken. Gelukkig zijn de fouten die je met programmeren maakt vaak iets minder ernstig dan met autorijden, tenzij je software voor zelfrijdende voertuigen maakt natuurlijk. Daarnaast ga je aan de slag met het schrijven van testen voor je code. Dit zorgt ervoor dat je code getest is voordat je deze laat uitvoeren. Test Driven Development, of TDD, noemen we dat.

Om te kunnen oefenen met het programmeren in Rust, moet je natuurlijk wel de nodige software op je computer hebben staan. Gelukkig is dat erg eenvoudig gemaakt en kunnen snel aan de slag. Let's dive in!

## Rust installeren

Alle informatie over rust vind je op de website van rust zelf: [https://rust-lang.org](https://rust-lang.org). Op het moment van schrijven zit Rust op versie 1.94.0. Zorg dus dat je minimaal deze versie installeert.

Om Rust op je computer te installeren klik je op de grote knop met GET STARTED erop. Dit brengt je naar [https://rust-lang.org/learn/get-started/](https://rust-lang.org/learn/get-started/), waar je de instructies om Rust te installeren kunt vinden.

De eenvoudigste manier om Rust te installeren is met Rustup, de installatie en versie beheer tool van Rust. Om die te installeren open je de terminal op je computer. Ik hoop niet dat dit je afschrikt, want de terminal gaan we nog veel vaker gebruiken in de lessen.

Typ bij de intimiderende, knipperende, cursor de volgend regel en druk op ENTER:

```bash
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Dit commando ziet er misschien een beetje intimiderend uit, maar wat het doet is niets meer dan het installatie bestand van Rust ophalen en zorgen dat alles op jouw computer gaat werken.

Met dit commando download je Rustup en installeer je meteen de 2 belangrijke tools die je nodig hebt:

- *Rustc*: De compiler van Rust. Deze zet je Rust bestanden om naar echt werkende programma's. Die programma's noemen we ook wel binaries.
- *Cargo*: Cargo is het Zwitserse zakmes van Rust. Hiermee kun je je programma's bouwen, uitvoeren, testen, documentatie bouwen en bibliotheken publiceren. Waar Rustc vaak op de achtergrond haar werk doet, zul je Cargo leren kennen als je nieuwe beste vriend!

**Detail voor Windows gebruikers**: Op Windows werkt de installatie van Rust net een beetje anders dan op MacOs en Linux. Kijk voor de instructies om Rust op Windows te installeren op: [Installeer Rust](https://forge.rust-lang.org/infra/other-installation-methods.html)

Bang om iets stuk te maken op je computer als je Rust installeert? Geen nood, het doet geen gekke dingen. Rust wordt volledig geïnstalleerd in een eigen mapje op je computer met de naam ```.cargo```. Wil je weer helemaal van Rust af, dan weet je nu welk mapje je moet hebben om weg te gooien!

## Cargo werkt niet!

Je hebt net de installatie uitgevoerd, roept ```cargo --version``` aan om de versie die geïnstalleerd hebt te controleren, krijg je een foutmelding.

```bash
    Command not found
```

Huh, je hebt net toch de installatie gedaan?

Soms komt het voor dat de terminal het nieuwe programma nog niet herkent. In dat geval kun je het beste de terminal even helemaal afsluiten en opnieuw opstarten. Daarna werken de nieuwe programma's als een zonnetje!

## Rust updaten

Ook Rust zelf blijft zich ontwikkelen. Op het moment van schrijven is versie 1.94.0 de nieuwste versie, maar er komt natuurlijk regelmatig een nieuwe versie uit om verbeteringen door te voeren. Omdat je Rustup hebt geïnstalleerd, kun updaten met een enkel eenvoudig commando:

```bash
    rustup update
```

## Cargo

Zoals je al hebt gelezen is Cargo je nieuwe beste vriend in de terminal. Hieronder zie je een paar commando's die je met Cargo kunt gebruiken. Lees ze rustig door en houd in je achterhoofd dat de commando's er zijn. Zodra je verder gaat met de lessen zul je steeds meer van de commando's leren gebruiken. Je hoeft ze nu nog niet uit je hoofd te leren.

- Een nieuw project starten doe je met ```cargo new [naam van het project]```.
- Met ```cargo build``` 'bouw' je een uitvoerbaar bestand van de code die je geschreven hebt.
- ```cargo run``` gebruik je om je programma uit te voeren zonder alles helemaal te moeten bouwen.
- Met ```cargo doc``` maak je documentatie van de code die je geschreven hebt.
- Soms wil je controleren of je programma goed geprogrammeerd is, zonder een uitvoerbaar bestand te maken. Dat doe je met het commando ```cargo check```.

Tot slot is er 1 commando wat een beetje extra liefde krijgt in dit boek: ```cargo test```. Dit commando voert de tests uit die je geschreven hebt. Dit commando ga je tijdens de lessen heel vaak gebruiken.

## Wat je nog meer nodig hebt

Je kunt nu projecten aanmaken, bouwen, testen, enzovoort, maar hoe schrijf je nu de code? Of beter gezegd: in welke programma schrijf je de code? Hierin heb je een heel ruime keuze. Je kunt code schrijven in heel eenvoudige editors zoals ```Edit``` op de Apples MacOs, of ```Notepad``` in Microsoft Windows. Dat is de meest basic manier.

Je kunt ook kiezen voor zeer uitgebreide code-schrijf-applicaties. Dat noem je een Integrated Development Environment of, afgekort, IDE. Dit zijn tools die je enorm helpen om snel code te produceren. Hierbij zitten tegenwoordig ook vaak oplossingen om AI te gebruiken om code te schrijven.

Ik ga er in deze lessen natuurlijk vanuit dat je vooral wilt leren programmeren. En zoals je als weet: leren is doen. Laat je al je code door AI schrijven, dan leer je natuurlijk niets. Bovendien: wat is daar nou leuk aan?

Daarom adviseer ik je een middenweg te kiezen. Progamma's als [Visual Studio Code](https://code.visualstudio.com) of [Zed](https://zed.dev) zijn uitstekende editors. Persoonlijk gebruik Zed trouwens om mijn code te schrijven. Ik heb lang gebruik gemaakt van Visual Studio Code, maar ontdekte Zed, ervaarde de hoge snelheid en lage systeemeisen en werd op slag verliefd. Maar, de keus is helemaal aan jou! Als je al een andere editor gebruikt waarin je ook Rust code kunt schrijven, voel je helemaal vrij om dat te doen!

Onder de sectie **other tools** op [https://rust-lang.org/learn/get-started/](https://rust-lang.org/learn/get-started/) vind je informatie over extra hulpmiddelen waarmee je jouw editor nog behulpzamer kunt maken.

## De volgende stap

Je hebt alles geïnstalleerd om een project te starten, code te schrijven en deze te testen en uit te voeren. De hoogste tijd dus om aan de slag te gaan met een eerste project: ```Hello, World!```
