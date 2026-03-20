# Leer Go met Tests

<p align="center">
  <img src="red-green-blue-gophers-smaller.png" />
</p>

[Art by Denise](https://twitter.com/deniseyu21)

[![Go Report Card](https://goreportcard.com/badge/github.com/quii/learn-go-with-tests)](https://goreportcard.com/report/github.com/quii/learn-go-with-tests)

[![GitBook](https://img.shields.io/static/v1?message=Documented%20on%20GitBook&logo=gitbook&logoColor=ffffff&label=%20&labelColor=5c5c5c&color=3F89A1)](https://www.gitbook.com/preview?utm_source=gitbook_readme_badge&utm_medium=organic&utm_campaign=preview_documentation&utm_content=link)

## Opmaak

- [Gitbook](https://quii.gitbook.io/learn-go-with-tests)
- [EPUB or PDF](https://github.com/quii/learn-go-with-tests/releases)

## Vertalingen

- [中文](https://studygolang.gitbook.io/learn-go-with-tests)
- [Português](https://larien.gitbook.io/aprenda-go-com-testes/)
- [日本語](https://andmorefine.gitbook.io/learn-go-with-tests/)
- [Français](https://goosegeesejeez.gitbook.io/apprendre-go-par-les-tests)
- [한국어](https://miryang.gitbook.io/learn-go-with-tests/)
- [Türkçe](https://halilkocaoz.gitbook.io/go-programlama-dilini-ogren/)
- [فارسی](https://go-yaad-begir.gitbook.io/go-ba-test/)
- [Nederlands](https://bobkosse.gitbook.io/leer-go-met-tests/)

## Ondersteun me

Ik ben er trots op deze bron gratis aan te bieden, maar als je me wat waardering wilt geven:

- [Tweet me @quii](https://twitter.com/quii)
- <a rel="me" href="https://mastodon.cloud/@quii">Mastodon</a>
- [Buy me a coffee :coffee:](https://www.buymeacoffee.com/quii)
- [Sponsor me on GitHub](https://github.com/sponsors/quii)

## Waarom

* Ontdek de Go taal door het schrijven van tests
* **Krijg een goede basis met TDD**. Go is een goede taal om TDD te leren omdat het een eenvoudige taal is om te leren en testen standaard ingebouwd zijn
* Je raakt vertrouwd en overtuig
* Vertrouw erop dat je in staat zult zijn om robuuste, goed geteste systemen in Go te schrijven
* [Bekijk een video of lees waarom unit testing en TDD belangrijk zijn](why.md)

## Inhoudsopgave

### Basisbeginselen van GO

1. [Go installeren](install-go.md) - Creëer een omgeving voor productiviteit.
2. [Hello, world](hello-world.md) - Creëer een omgeving voor productiviteit. Declareer variabelen, constanten, if/else-instructies, switch-structuur, schrijf je eerste go-programma en schrijf je eerste test. Syntaxis en closures van subtests.
3. [Integers](integers.md) - Verdiep je verder in de syntaxis van functiedeclaraties en ontdek nieuwe manieren om de documentatie van je code te verbeteren.
4. [Iteratie](iteration.md) - Leer meer over `for` en benchmarking.
5. [Arrays en slices](arrays-and-slices.md) - Leer over arrays, slices, `len`, varargs, `range` en test coverage.
6. [Structs, methods & interfaces](structs-methods-and-interfaces.md) - Leer over `struct`, methods, `interface` en table driven tests.
7. [Pointers & errors](pointers-and-errors.md) - Leer over pointers en fouten.
8. [Maps](maps.md) - Leer over het opslaan van waarden in de map data structuur.
9. [Dependency Injection](dependency-injection.md) - Leer over dependency injection, hoe het zich verhoudt tot het gebruik van interfaces en een inleiding tot io.
10. [Mocking](mocking.md) - Neem wat bestaande, ongeteste code en test deze met DI met mocking.
11. [Concurrency](concurrency.md) - Leer hoe je concurrent code schrijft om software sneller te maken.
12. [Select](select.md) - Leer hoe je asynchrone processen op een elegante manier kunt synchroniseren.
13. [Reflection](reflection.md) - Learn about reflection.
14. [Sync](sync.md) - Leer enkele functionaliteiten van het synchronisatiepakket, waaronder `WaitGroup` en `Mutex`
15. [Context](context.md) - Gebruik het contextpakket om langlopende processen te beheren en te annuleren
16. [Introductie tot property based tests](roman-numerals.md) - Oefen wat TDD met de Romeinse cijfers kata en krijg een korte introductie tot op eigenschappen gebaseerde tests
17. [Maths](math.md) - Gebruik de `math` package om een SVG klok te tekenen
18. [Bestanden lezen](reading-files.md) - Lees bestanden en verwerk deze
19. [Templating](html-templates.md) - Gebruik het html/template-pakket van Go om html uit gegevens te renderen en leer ook over 'approval testing'
20. [Generics](generics.md) - Leer hoe je functies schrijft die generieke argumenten gebruiken en je eigen generieke datastructuur maakt
21. [Arrays en slices herzien met generics](revisiting-arrays-and-slices-with-generics.md) - Generieke patronen zijn erg handig bij het werken met verzamelingen. Leer hoe je je eigen 'Reduce'-functie schrijft en een aantal veelvoorkomende patronen opruimt.

### Bouw en applicatie

Hopelijk heb je nu de _Basisbeginselen van Go_ doorgenomen en heb je een goede basiskennis van de meeste taalfuncties van Go en hoe je TDD uitvoert.

In het volgende gedeelte ga je een applicatie bouwen.

Elk hoofdstuk bouwt voort op het vorige en breidt de functionaliteit van de applicatie uit zoals onze Product Owner dat voorschrijft.

Er worden nieuwe concepten geïntroduceerd die het schrijven van goede code vergemakkelijken, maar het meeste nieuwe materiaal gaat over wat er allemaal mogelijk is met de standaardbibliotheek van Go.

Aan het einde van deze cursus weet je waarschijnlijk heel goed hoe je iteratief een applicatie in Go schrijft, ondersteund door tests.

* [HTTP server](http-server.md) - We gaan een applicatie creëren die naar HTTP-verzoeken luistert en hierop reageert.
* [JSON, routing en embedding](json.md) - We zorgen ervoor dat onze eindpunten JSON retourneren en onderzoeken hoe we routing kunnen uitvoeren.
* [IO en sorting](io.md) - We blijven onze gegevens van de schijf lezen en we gaan het sorteren van gegevens behandelen.
* [Command line & project structuur](command-line.md) - Ondersteun meerdere applicaties vanuit één codebase en lees invoer vanaf de opdrachtregel.
* [Time](time.md) - het gebruik van het `time`-pakket om activiteiten te plannen.
* [WebSockets](websockets.md) - Leer hoe je een server schrijft en test die WebSockets gebruikt.

### Basisbeginselen van testen

Andere onderwerpen rondom testen behandelen.

* [Introductie tot acceptatie testen](intro-to-acceptance-tests.md) - Leer hoe je acceptatietests voor je code schrijft, met een praktijkvoorbeeld van het netjes afsluiten van een HTTP-server
* [Schalen van acceptatie testen](scaling-acceptance-tests.md) - Leer technieken om de complexiteit van het schrijven van acceptatietests voor niet-triviale systemen te beheersen.
* [Werken zonder mocks, stubs en spies](working-without-mocks.md) - Leer hoe je fakes en contracten kunt gebruiken om realistischere en beter onderhoudbare tests te maken.
* [Refactoring Checklist](refactoring-checklist.md) - Een discussie over wat refactoring is, en enkele basistips voor hoe je het kunt uitvoeren.

### Vragen en antwoorden

Ik kom op internet vaak vragen tegen zoals

> Hoe test ik mijn fantastische functie die x, y en z doet

Als je zo'n vraag hebt, stel hem dan als issue op GitHub en ik zal proberen tijd te vinden om een kort hoofdstuk te schrijven over het probleem. Ik vind dit soort content waardevol, omdat het de echte vragen van mensen over testen beantwoordt.

* [OS exec](os-exec.md) - Een voorbeeld van hoe we verbinding kunnen maken met het besturingssysteem om opdrachten uit te voeren om gegevens op te halen en onze bedrijfslogica testbaar te houden/
* [Error types](error-types.md) - Voorbeeld van hoe je je eigen fouttypen kunt maken om je tests te verbeteren en je code gebruiksvriendelijker te maken.
* [Context-aware Reader](context-aware-reader.md) - Leer hoe je TDD kunt gebruiken om `io.Reader` uit te breiden met annulering. Gebaseerd op [Context-aware io.Reader for Go](https://pace.dev/blog/2020/02/03/context-aware-ioreader-for-golang-by-mat-ryer)
* [Revisiting HTTP Handlers](http-handlers-revisited.md) - Het testen van HTTP-handlers lijkt voor veel ontwikkelaars een doorn in het oog. Dit hoofdstuk onderzoekt de problemen rond het correct ontwerpen van handlers.

### Meta / Discussie

* [Waarom unittests en hoe je ze voor je kunt laten werken](why.md) - Bekijk een video of lees waarom unit testing en TDD belangrijk zijn
* [Anti-patterns](anti-patterns.md) - Een kort hoofdstuk over TDD en unit testing anti-patterns

## Bijdragen

* _Dit project is work in progress_ Als je het leuk vindt hieraan bij te dragen, neem dan contact op.
* Lees [contributing.md](https://github.com/quii/learn-go-with-tests/tree/842f4f24d1f1c20ba3bb23cbc376c7ca6f7ca79a/contributing.md) voor de richtlijnen.
* Ideeën? Maak een issue aan

## Achtergrond

Ik heb enige ervaring met het introduceren van Go bij ontwikkelteams en heb verschillende benaderingen geprobeerd om een team te laten groeien van mensen die nieuwsgierig waren naar Go tot mensen die zeer effectieve schrijvers van Go-systemen zijn.

### Wat niet werkte

#### _Het_ boek lezen

Een aanpak die we probeerden was om [het blauwe boek](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440) te pakken en elke week het volgende hoofdstuk te bespreken, samen met de oefeningen.

Ik ben dol op dit boek, maar het vereist een hoge mate van toewijding. Het boek legt concepten zeer gedetailleerd uit, wat natuurlijk geweldig is, maar het betekent wel dat de voortgang langzaam en gestaag verloopt, dit is niet voor iedereen weggelegd.

Ik ontdekte dat een kleine groep mensen hoofdstuk X zou lezen en de oefeningen zou maken, maar dat veel mensen dat niet deden.

#### Een paar problemen oplossen

Kata's zijn leuk, maar je leert er meestal maar beperkt een taal mee. Je zult waarschijnlijk geen goroutines gebruiken om een kata op te lossen.

Een ander probleem is wanneer je enthousiasme varieert. Sommige mensen leren gewoon veel meer van de taal dan anderen, en als ze laten zien wat ze hebben gedaan, raken ze in de war met functies die de anderen niet kennen.

Dit zorgt ervoor dat het leren nogal _ongestructureerd_ en _ad hoc_ aanvoelt.

### Wat wel werkt

Veruit de meest effectieve manier was om de basisbeginselen van de taal langzaam te introduceren door [go by example](https://gobyexample.com/) te lezen, ze te verkennen met voorbeelden en ze in groepsverband te bespreken. Dit was een interactievere aanpak dan "lees hoofdstuk x als huiswerk".

Na verloop van tijd had het team een solide basis van de _grammatica_ van de taal ontwikkeld, zodat we konden beginnen met het bouwen van systemen.

Voor mij is dit vergelijkbaar met het oefenen van toonladders wanneer je gitaar probeert te leren spelen.

Het maakt niet uit hoe artistiek je denkt dat je bent, je zult waarschijnlijk geen goede muziek kunnen schrijven als je de basis niet begrijpt en de mechaniek niet oefent.

### Wat voor mij werkte

Als _ik_ een nieuwe programmeertaal leer, begin ik meestal met wat rommelen in een REPL, maar uiteindelijk heb ik meer structuur nodig.

Wat ik graag doe, is concepten verkennen en de ideeën vervolgens met tests onderbouwen. Tests verifiëren of de code die ik schrijf correct is, en documenteren de functionaliteit die ik heb geleerd.

Gebruikmakend van mijn ervaring met leren in een groep en mijn eigen persoonlijke aanpak, ga ik proberen iets te creëren dat hopelijk nuttig is voor andere teams. Leer de basisprincipes door kleine tests te schrijven, zodat je vervolgens je bestaande softwareontwerpvaardigheden kunt gebruiken om geweldige systemen te ontwikkelen.

## Voor wie is dit

* Voor mensen die geïnteresseerd zijn om met Go aan de slag te gaan
* Mensen die al bekend zijn met Go, maar testen met TDD verder willen ontdekken

## Wat je nodig hebt

* Een computer!
* [Go geïnstalleerd](https://golang.org/)
* Een tekst editor
* Enige ervaring met programmeren. Begrip van concepten zoals 'if', variabelen, functies, etc.
* Je moet je comfortable voelen de terminal te gebruiken

## Feedback

* Problemen melden/PR's indienen [hier](https://github.com/quii/learn-go-with-tests) of [tweet me @quii](https://twitter.com/quii)

[MIT license](LICENSE.md)

[Logo is by egonelbre](https://github.com/egonelbre) Wat een ster!

## Over deze vertaling

Deze vertaling is gemaakt door [Bob Kosse](https://bobkosse.nl). Ook deze vertaling wordt, net als het origineel, kosteloos vrijgegeven. Ik hoop hiermee de programmeertaal Go ook in Nederland toegankelijker te maken voor een breder publiek.

Ik heb dit boek vertaald naar het Nederlands, maar enkele begrippen zijn bewust Engelstalig gehouden. Go gebruikt, net als veel andere programmeertalen, veel Engelse benamingen. Deze zijn in takt gelaten. Ook speciefieke woorden waarvan in de markt vaak de Engelse variant wordt gebruikt, heb ik ook in het Engels gehouden.

Heb je vragen over deze vertaling? Neem van vooral contact met mij op:
[LinkedIn](https://linkedin.com/in/bobkosse)
[GitHub](https://github.com/bobkosse)
