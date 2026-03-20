# TDD-antipatronen

Van tijd tot tijd is het nodig om je TDD-technieken te herzien en jezelf te herinneren aan gedrag dat je moet vermijden.

Het TDD-proces is conceptueel eenvoudig te volgen, maar je zult merken dat het je ontwerpvaardigheden op de proef stelt. **Verwar dit niet met TDD als moeilijk, het is juist het ontwerp dat moeilijk is!**

Dit hoofdstuk somt een aantal TDD- en test-antipatronen op, en hoe je deze kunt verhelpen.

## Helemaal geen TDD gebruiken

Natuurlijk is het mogelijk om geweldige software te schrijven zonder TDD, maar veel problemen die ik heb gezien met het ontwerp van code en de kwaliteit van tests zouden erg moeilijk op te lossen zijn als er geen gedisciplineerde aanpak voor TDD was gebruikt.

Een van de sterke punten van TDD is dat het je een formeel proces biedt om problemen op te splitsen, te begrijpen wat je probeert te bereiken (rood), het voor elkaar te krijgen (groen) en vervolgens goed na te denken over hoe je het goed kunt maken (blauw/refactoren).

Zonder dit is het proces vaak ad-hoc en losjes, wat engineering moeilijker _kan_ maken dan het _zou_ kunnen_ zijn.

## De beperkingen van de refactoringstap verkeerd begrijpen

Ik heb een aantal workshops, mobbing- of pairingsessies bijgewoond waarbij iemand een testpass had gemaakt en zich in de refactoringfase bevond. Na enig nadenken dachten ze dat het een goed idee zou zijn om wat code te abstraheren tot een nieuwe struct; een beginnende betweter schreeuwde:

> Dit mag je niet doen! Je zou hier eerst een test voor moeten schrijven, we zijn bezig met TDD!

Dit lijkt een veelvoorkomend misverstand te zijn. **Je mag doen wat je wilt met de code als de tests groen zijn**, het enige wat je niet mag doen is **gedrag toevoegen of wijzigen**.

Het doel van deze tests is om je de _vrijheid te geven om te refactoren_, de juiste abstracties te vinden en de code gemakkelijker te veranderen en te begrijpen te maken.

## Tests hebben die niet falen (oftewel evergreen tests)

Het is verbazingwekkend hoe vaak dit voorkomt. Je begint met debuggen of het wijzigen van tests en realiseert je: er zijn geen scenario's waarin deze test kan falen. Of in ieder geval, hij zal niet falen op de manier waarop de test _hoort_ te beschermen.

Dit is _bijna onmogelijk_ met TDD als je **de eerste stap** volgt,

> Schrijf een test, zie hem falen

Dit gebeurt bijna altijd wanneer ontwikkelaars tests schrijven _nadat_ de code is geschreven, en/of om testdekking na te streven in plaats van een bruikbare testsuite te creëren.

## Nutteloze vergelijkingen

Heb je ooit aan een systeem gewerkt en een test stuk gemaakt, en dan zie je dit?

> `false was not equal to true`

Ik weet dat false niet gelijk is aan true. Dit is geen nuttige melding; het vertelt me niet wat ik stuk gemaakt heb. Dit is een symptoom van het niet volgen van het TDD-proces en het niet lezen van de foutmelding.

Terug naar de tekentafel:

> Schrijf een test, zie hem falen (en schaam je niet voor de foutmelding)


## Vergelijkingen op basis van irrelevante details

Een voorbeeld hiervan is het doen van een vergelijking op basis van een complex object, terwijl het in de praktijk alleen om de waarde van een van de velden gaat.

```go
// not this, now your test is tightly coupled to the whole object
if !cmp.Equal(complexObject, want) {
	t.Error("got %+v, want %+v", complexObject, want)
}

// be specific, and loosen the coupling
got := complexObject.fieldYouCareAboutForThisTest
if got != want {
	t.Error("got %q, want %q", got, want)
}
```

Extra vergelijkingen maken je test niet alleen moeilijker leesbaar door 'ruis' in je documentatie te creëren, maar koppelen de test ook onnodig aan data die er niet toe doet. Dit betekent dat als je de velden voor je object of het gedrag ervan wijzigt, je onverwachte compilatieproblemen of fouten in je tests kunt krijgen.

Dit is een voorbeeld van het niet strikt genoeg volgen van de rode fase.

- Een bestaand ontwerp laten beïnvloeden hoe je je test schrijft **in plaats van na te denken over het gewenste gedrag**
- Niet genoeg aandacht besteden aan de foutmelding van de falende test

## Veel vergelijkingen binnen één scenario voor unittests

Veel vergelijkingen kunnen tests moeilijk leesbaar en debuggend maken wanneer ze mislukken.

Ze sluipen er vaak geleidelijk in, vooral als de testopstelling ingewikkeld is, omdat je aarzelt om dezelfde vreselijke opstelling te repliceren om vergelijkingen op iets anders te doen. In plaats daarvan zou je de problemen in je ontwerp moeten oplossen die het moeilijk maken om beweringen op nieuwe dingen te doen.

Een handige vuistregel is om te streven naar één vergelijking per test. Maak in Go gebruik van subtests om een duidelijk onderscheid te maken tussen vergelijkingen wanneer dat nodig is. Dit is ook een handige techniek om vergelijkingen over gedrag te scheiden van implementatiedetails.

Voor andere tests waarbij de opzet- of uitvoeringstijd een vergelijking kan zijn (bijvoorbeeld een acceptatietest die een webbrowser bestuurt), moet je de voor- en nadelen van iets lastiger te debuggen tests afwegen tegen de uitvoeringstijd van de test.

## Niet luisteren naar je tests

[Dave Farley wijst er in zijn video "When TDD goes wrong"](https://www.youtube.com/watch?v=UWtEVKVPBQ0&feature=youtu.be) op:

> TDD geeft je de snelst mogelijke feedback op je ontwerp

Uit eigen ervaring weet ik dat veel ontwikkelaars proberen TDD te oefenen, maar vaak de signalen negeren die ze uit het TDD-proces krijgen. Daardoor zitten ze nog steeds vast aan kwetsbare, irritante systemen met een slechte testsuite.

Simpel gezegd: als het testen van je code moeilijk is, dan is het _gebruiken_ van je code ook moeilijk. Beschouw je tests als de eerste gebruiker van je code en dan zul je zien of je code prettig is om mee te werken of niet.

Ik heb dit vaak benadrukt in het boek, en ik herhaal het nog een keer: **luister naar je tests**.

### Overmatige setup, te veel testdubbels, enz.

Heb je ooit naar een test gekeken met 20, 50, 100, 200 regels setupcode voordat er iets interessants in de test gebeurt? Moet je dan de code aanpassen, de chaos opnieuw bekijken en wensen dat je een andere carrière had?

Wat zijn hier de signalen? _Luister_, gecompliceerde tests `==` gecompliceerde code. Waarom is je code ingewikkeld? Moet dat zo zijn?

- Als je veel testdubbels in je tests hebt, betekent dit dat de code die je test veel afhankelijkheden heeft, wat betekent dat je ontwerp moet worden aangepast.
- Als je test afhankelijk is van het instellen van verschillende interacties met mocks, betekent dit dat je code veel interacties met zijn afhankelijkheden maakt. Vraag jezelf af of deze interacties eenvoudiger kunnen.

#### Lekkende interfaces

Als je een `interface` hebt gedeclareerd die veel methoden heeft, verwijst dat naar een lekkende abstractie. Denk na over hoe je die samenwerking zou kunnen definiëren met een meer geconsolideerde set methoden, idealiter één.

#### Interfacevervuiling

Zoals een Go-spreekwoord luidt: *hoe groter de interface, hoe zwakker de abstractie*. Als je een enorme interface beschikbaar stelt aan de gebruikers van je pakket, dwing je ze om in hun tests een stub/mock te maken die overeenkomt met de volledige API, en zo ook een implementatie te bieden voor methoden die ze niet gebruiken (soms raken ze gewoon in paniek om duidelijk te maken dat ze niet gebruikt moeten worden). Deze situatie is een antipatroon dat bekend staat als [interfacevervuiling](https://rakyll.org/interface-pollution/) en dit is de reden waarom de standaardbibliotheek je slechts kleine interfaces biedt.

In plaats daarvan zou je vanuit je pakket een kale struct met alle relevante methoden geëxporteerd moeten maken, zodat de clients van je API de vrijheid hebben om hun eigen interfaces te declareren en te abstraheren over de subset van de methoden die ze nodig hebben: bijvoorbeeld [go-redis](https://github.com/redis/go-redis) stelt een struct (`redis.Client`) beschikbaar aan de API-clients.

Over het algemeen moet je een interface alleen aan clients beschikbaar stellen wanneer:
- de interface bestaat uit een kleine en samenhangende set functies.
- de interface en de implementatie ervan losgekoppeld moeten zijn (bijvoorbeeld omdat gebruikers kunnen kiezen uit meerdere implementaties of omdat ze een externe afhankelijkheid moeten simuleren).

#### Denk na over de soorten testdoubles die je gebruikt

- Mocks zijn soms nuttig, maar ze zijn extreem krachtig en daardoor gemakkelijk te misbruiken. Probeer jezelf de beperking op te leggen om in plaats daarvan stubs te gebruiken.
- Het verifiëren van implementatiedetails met spionnen is soms nuttig, maar probeer dit te vermijden. Onthoud dat je implementatiedetails meestal niet belangrijk zijn en dat je je tests er indien mogelijk niet aan wilt koppelen. Probeer je tests te koppelen aan **nuttig gedrag in plaats van incidentele details**.
- [Lees mijn berichten over het benoemen van testdoubles](https://quii.dev/Start_naming_your_test_doubles_correctly) als de taxonomie van testdoubles wat onduidelijk is

#### Consolideer afhankelijkheden

Hier is wat code voor een `http.HandlerFunc` om nieuwe gebruikersregistraties voor een website af te handelen.

```go
type User struct {
	// Some user fields
}

type UserStore interface {
	CheckEmailExists(email string) (bool, error)
	StoreUser(newUser User) error
}

type Emailer interface {
	SendEmail(to User, body string, subject string) error
}

func NewRegistrationHandler(userStore UserStore, emailer Emailer) http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		// extract out the user from the request body (handle error)
		// check user exists (handle duplicates, errors)
		// store user (handle errors)
		// compose and send confirmation email (handle error)
		// if we got this far, return 2xx response
	}
}
```

Bij de eerste poging is het redelijk om te zeggen dat het ontwerp niet zo slecht is. Het heeft maar twee afhankelijkheden!

Evalueer het ontwerp opnieuw door rekening te houden met de verantwoordelijkheden van de handler:

- Parseer de aanvraagbody naar een `User` :white_check_mark:
- Gebruik `UserStore` om te controleren of de gebruiker bestaat :question:
- Gebruik `UserStore` om de gebruiker op te slaan :question:
- Stel een e-mail op :question:
- Gebruik `Emailer` om de e-mail te verzenden :question:
- Retourneer een passend http-antwoord, afhankelijk van succes, fouten, enz. :white_check_mark:

Om deze code te gebruiken, moet je veel tests schrijven met verschillende niveaus van testdubbelconfiguraties, spionnen, enz.

- Wat als de vereisten worden uitgebreid? Vertalingen voor de e-mails? Ook een sms-bevestiging versturen? Lijkt het je logisch dat je een HTTP-handler moet aanpassen om deze wijziging mogelijk te maken?
- Voelt het goed dat de belangrijke regel "we moeten een e-mail sturen" zich in een HTTP-handler bevindt?
- Waarom moet je de hele ceremonie van het aanmaken van HTTP-verzoeken en het lezen van reacties doorlopen om die regel te verifiëren?

**Luister naar je tests**. Het schrijven van tests voor deze code op een TDD-manier zou je al snel een ongemakkelijk gevoel moeten geven (of in ieder geval de luie ontwikkelaar in jou geïrriteerd moeten maken). Als het pijnlijk aanvoelt, stop dan even en denk na.

Wat als het ontwerp er in plaats daarvan zo uitzag?

```go
type UserService interface {
	Register(newUser User) error
}

func NewRegistrationHandler(userService UserService) http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		// parse user
		// register user
		// check error, send response
	}
}
```

- Eenvoudig om de handler te testen ✅
- Wijzigingen in de regels rondom registratie zijn geïsoleerd van HTTP, waardoor ze ook eenvoudiger te testen zijn ✅

## Inkapseling schenden

Inkapseling is erg belangrijk. Er is een reden waarom we niet alles in een pakket exporteren (of openbaar maken). We willen coherente API's met een klein oppervlak om nauwe koppeling te voorkomen.

Mensen komen soms in de verleiding om een functie of methode openbaar te maken om iets te testen. Door dit te doen, verslechter je je ontwerp en stuur je verwarrende berichten naar beheerders en gebruikers van je code.

Een gevolg hiervan kan zijn dat ontwikkelaars een test proberen te debuggen en er uiteindelijk achter komen dat de geteste functie _alleen vanuit tests_ wordt aangeroepen. Wat natuurlijk **een vreselijke uitkomst en tijdverspilling** is.

Beschouw in Go je standaardpositie voor het schrijven van tests als _vanuit het perspectief van een gebruiker van je pakket_. Je kunt dit een compile-time beperking maken door je tests in een testpakket te plaatsen, bijvoorbeeld `package gocoin_test`. Als je dit doet, heb je alleen toegang tot de geëxporteerde leden van het pakket, waardoor het niet mogelijk is om jezelf te koppelen aan implementatiedetails.

## Gecompliceerde tabeltests

Tabeltests zijn een geweldige manier om een aantal verschillende scenario's te oefenen wanneer de testopstelling hetzelfde is en je alleen de invoer wilt variëren.

_Maar_ ze kunnen lastig te lezen en te begrijpen zijn wanneer je andere soorten tests probeert te combineren onder de noemer van één fantastische tabel.

```go
cases := []struct {
	X                int
	Y                int
	Z                int
	err              error
	IsFullMoon       bool
	IsLeapYear       bool
	AtWarWithEurasia bool
}{}
```

**Wees niet bang om je tabel te verlaten en nieuwe tests te schrijven** in plaats van nieuwe velden en booleans toe te voegen aan de tabel `struct`.

Een ding om in gedachten te houden bij het schrijven van software is:

> [Eenvoudig is niet makkelijk](https://www.infoq.com/presentations/Simple-Made-Easy/)

"Gewoon" een veld toevoegen aan een tabel is misschien makkelijk, maar het kan de zaken verre van eenvoudig maken.

## Samenvatting

De meeste problemen met unit tests zijn normaal gesproken te wijten aan:

- Ontwikkelaars die het TDD-proces niet volgen
- Slecht ontwerp

Dus, leer over goed softwareontwerp!

Het goede nieuws is dat TDD je kan helpen _je ontwerpvaardigheden te verbeteren_, want zoals in het begin al werd gezegd:

**Het belangrijkste doel van TDD is om feedback te geven op je ontwerp.** Luister voor de miljoenste keer naar je tests, ze reflecteren je ontwerp.

Wees eerlijk over de kwaliteit van je tests door te luisteren naar de feedback die ze je geven, en je zult er een betere ontwikkelaar door worden.
