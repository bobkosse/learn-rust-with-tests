# Go Installaren

De officiële installatie instructies voor Go zij [hier](https://golang.org/doc/install) te vinden.

## Go Omgeving

### Go Modules

Go 1.11 introduceerde [Modules](https://go.dev/wiki/Modules). Deze aanpak is de standaard bouw methode sinds Go 1.16, daardoor wordt het gebruik van  `GOPATH` afgeraden vanaf Go 1.16.

Modules zijn bedoeld om problemen op te lossen die te maken hebben met afhankelijkheidsbeheer, versies-electie en reproduceerbare builds. Daarnaast maken ze het voor gebruikers mogelijk om Go-code buiten `GOPATH` uit te voeren.

Het gebruik van Modules is behoorlijk recht-toe-recht-aan. Selecteer een directory buiten `GOPATH` als basis voor je project en maak een nieuwe module met het commando `go mod init`.

Een bestand met de naam `go.mod` wordt aangemaakt. Dit bestand bevat het path van de module, de Go versie en de vereiste afhankelijkheden. De vereiste afhankelijkheden zijn de andere modules die nodig zijn voor een succesvolle bouw van het project.

Als er geen `<modulepath>` is gespecificeerd, zal `go mod init` proberen het module path te achterhalen vanuit de directory structuur. Dit kan ook overruled worden door een argument aan het commando toe te voegen.

```sh
mkdir my-project
cd my-project
go mod init <modulepath>
```

Een `go.mod` bestand zou er zo uit moeten zien:

```
module cmd

go 1.16

```

De ingebouwde documentatie geeft een overzicht van alle beschikbare `go mod` commando's.

```sh
go help mod
go help mod init
```

## Go Linting

Een verbetering van de standaard linter kan worden geconfigureerd met [GolangCI-Lint](https://golangci-lint.run).

Deze kan als volgt worden geïnstalleerd:

```sh
brew install golangci-lint
```

## Refactoring en jouw tools

In dit boek wordt de nadruk gelegd op het belang van refactoring.

De juiste tools kunnen je helpen het grotere refactor werk met zelfverzekerdheid te doen.

Je moet voldoende bekend zijn met je editor om het volgende met een eenvoudige toetsencombinatie te kunnen doen:

* **Extract/Inline variabele**. Door magische waarden een naam te geven, kun je je code snel vereenvoudigen.
* **Extract methoden/functies**. Het is van groot belang om een deel van de code te kunnen nemen en daar functies/methoden uit te halen
* **Hernoemen**. Je zou symbolen in verschillende bestanden met vertrouwen moeten kunnen hernoemen.
* **go fmt**. Go heeft een eigenzinnige formatter genaamd `go fmt`. Je editor zou dit op elk opgeslagen bestand moeten uitvoeren.
* **Tests uitvoeren**. Je zou een van de bovenstaande handelingen moeten kunnen uitvoeren en daarna snel je tests opnieuw moeten uitvoeren om er zeker van te zijn dat je refactoring niets kapot heeft gemaakt.

Om je te helpen met je code, moet je daarnaast het volgende kunnen:

* **Bekijken function signature**. Je hoeft nooit te twijfelen hoe je een functie in Go aanroept. Je IDE moet een functie beschrijven in termen van de documentatie, de parameters en wat deze retourneert.
* **Toon function definition**. Als het nog steeds niet duidelijk is wat een functie doet, kun je het beste naar de broncode gaan en proberen het zelf uit te zoeken.
* **Vind toepassingen van functionaliteit.** Inzicht in de context van een functie kan je helpen bij het nemen van beslissingen bij het refactoren.

Wanneer je je tools beheerst, kun je je beter concentreren op de code en hoeft je minder vaak van context te wisselen.

## Samenvattend

Op dit punt zou Go geïnstalleerd moeten zijn, zou er een editor beschikbaar moeten zijn en zou er wat basistooling aanwezig moeten zijn. Go heeft een zeer uitgebreid ecosysteem van producten van derden. We hebben hier een paar nuttige componenten geïdentificeerd. Voor een completere lijst verwijs ik je naar: [https://awesome-go.com](https://awesome-go.com).
