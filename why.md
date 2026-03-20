# Waarom unittests en hoe je ze voor jou kunt laten werken

[Hier is een link naar een video waarin ik over dit onderwerp praat](https://www.youtube.com/watch?v=Kwtit8ZEK7U)

Als je niet van video's houdt, is hier de geschreven versie.

## Software

De belofte van software is dat het kan veranderen. Daarom heet het ook _software_; het is kneedbaar in vergelijking met hardware. Een geweldig engineeringteam zou een geweldige aanwinst voor een bedrijf moeten zijn, door systemen te schrijven die met het bedrijf mee kunnen evolueren om waarde te blijven leveren.

Dus waarom zijn we er zo slecht in? Hoeveel projecten hoor je wel eens over projecten die compleet mislukken? Of "legacy" worden en volledig herschreven moeten worden (en die herschrijvingen mislukken vaak ook!)?

Hoe kan een softwaresysteem überhaupt "falen"? Kan het niet gewoon aangepast worden totdat het correct is? Dat is wat ons beloofd wordt!

Veel mensen kiezen ervoor om systemen te bouwen omdat Go een aantal keuzes heeft gemaakt waarvan men hoopt dat ze het legacy-bestendiger maken.

- Vergeleken met mijn vorige leven met Scala, waar [ik beschreef hoe het genoeg touw heeft om jezelf aan op te hangen](http://www.quii.dev/Scala_-_Just_enough_rope_to_hang_yourself), heeft Go slechts 25 trefwoorden en kunnen _veel_ systemen worden gebouwd met de standaardbibliotheek en een paar andere kleine bibliotheken. De hoop is dat je met Go code kunt schrijven en er over 6 maanden mee aan de slag kunt gaan en dat het dan nog steeds zinvol is.
- De tooling met betrekking tot testen, benchmarken, linting en verzending is eersteklas vergeleken met de meeste alternatieven.
- De standaardbibliotheek is briljant.
- Zeer snelle compilatiesnelheid voor strakke feedbackloops.
- De belofte van achterwaartse compatibiliteit met Go. Het lijkt erop dat Go in de toekomst generieke versies en andere functies krijgt, maar de ontwerpers hebben beloofd dat zelfs Go-code die je 5 jaar geleden schreef, nog steeds te compileren en bouwen is. Ik heb letterlijk wekenlang een project geüpgraded van Scala 2.8 naar 2.10.

Zelfs met al deze geweldige eigenschappen kunnen we nog steeds vreselijke systemen maken, dus we moeten terugkijken naar het verleden en lessen in software engineering leren die van toepassing zijn, hoe briljant (of niet) je programmeertaal ook is.

In 1974 schreef een slimme software engineer genaamd [Manny Lehman](https://en.wikipedia.org/wiki/Manny_Lehman_%28computer_scientist%29) [Lehman's laws of software evolution](https://en.wikipedia.org/wiki/Lehman%27s_laws_of_software_evolution].

> Deze wetten beschrijven een evenwicht tussen enerzijds krachten die nieuwe ontwikkelingen stimuleren en anderzijds krachten die de vooruitgang vertragen.

Het lijkt erop dat deze krachten belangrijk zijn om te begrijpen als we niet in een eindeloze cyclus willen belanden van systemen ontwikkelen die verouderd raken en vervolgens steeds opnieuw worden herschreven.

## De wet van continue verandering

> Elk softwaresysteem dat in de praktijk wordt gebruikt, moet veranderen of wordt steeds minder bruikbaar in de omgeving.

Het lijkt vanzelfsprekend dat een systeem _moet_ veranderen of minder bruikbaar wordt, maar hoe vaak wordt dit genegeerd?

Veel teams voelen zich geprikkeld om een project op een bepaalde datum op te leveren en vervolgens door te gaan naar het volgende project. Als de software "geluk" heeft, wordt het tenminste overgedragen aan een andere groep mensen om het te onderhouden, maar zij hebben het natuurlijk niet zelf geschreven.

Mensen zijn vaak bezig met het kiezen van een framework dat hen helpt om "snel te leveren", maar richten zich niet op de levensduur van het systeem in termen van hoe het moet evolueren.

Zelfs als je een geweldige software engineer bent, zul je nog steeds het slachtoffer worden van het niet kennen van de toekomstige behoeften van je systeem. Naarmate de business verandert, is een deel van de briljante code die je schreef nu niet meer relevant.

Lehman was in de jaren 70 op dreef omdat hij ons een nieuwe wet gaf om over na te denken.

## De wet van toenemende complexiteit

> Naarmate een systeem evolueert, neemt de complexiteit ervan toe, tenzij er wordt gewerkt aan het verminderen ervan.

Wat hij hier zegt, is dat we softwareteams niet als blinde functiefabrieken kunnen hebben, die steeds meer functies aan software toevoegen in de hoop dat het op de lange termijn zal overleven.

We **moeten** de complexiteit van het systeem blijven beheren naarmate de kennis van ons domein verandert.

## Refactoring

Er zijn _vele_ facetten van software engineering die software flexibel houden, zoals:

- Empowerment van ontwikkelaars
- Over het algemeen "goede" code. Verstandige scheiding van belangen, enz. enz.
- Communicatievaardigheden
- Architectuur
- Observeerbaarheid
- Implementeerbaarheid
- Geautomatiseerde tests
- Feedbackloops

Ik ga me richten op refactoring. Het is een veelgehoorde uitspraak "we moeten dit refactoren" - gezegd tegen een ontwikkelaar op zijn eerste dag als programmeur, zonder erbij na te denken.

Waar komt die uitspraak vandaan? Wat is het verschil tussen refactoring en het schrijven van code?

Ik weet dat ik en vele anderen _dachten_ dat we refactoring deden, maar we hadden het mis.

[Martin Fowler beschrijft hoe mensen het verkeerd aanpakken](https://martinfowler.com/bliki/RefactoringMalapropism.html)

> De term "refactoring" wordt heel vaak gebruikt wanneer deze niet van toepassing is. Als iemand tijdens het refactoren zegt dat een systeem een paar dagen kapot is, kun je er vrij zeker van zijn dat ze niet refactoren.

Dus wat is het dan?

### Factorisatie

Toen je op school wiskunde leerde, heb je waarschijnlijk al over factorisatie geleerd. Hier is een heel eenvoudig voorbeeld:

Bereken `1/2 + 1/4`

Om dit te doen, _ontbind (factorise)_ je de noemers, waardoor de uitdrukking verandert in

`2/4 + 1/4`, wat je vervolgens kunt omzetten in `3/4`.

We kunnen hieruit een aantal belangrijke lessen trekken. Wanneer we deze expressie _factoriseren_, hebben we **de betekenis van de uitdrukking niet veranderd**. Beide zijn gelijk aan `3/4`, maar we hebben het wel makkelijker gemaakt om ermee te werken; door `1/2` te veranderen in `2/4` past het beter in ons "domein".

Wanneer je je code refactored, probeer je manieren te vinden om je code begrijpelijker te maken en te "passen" bij je huidige begrip van wat het systeem moet doen. Cruciaal is **dat je het gedrag niet moet veranderen**.

#### Een voorbeeld in Go

Hier is een functie die `naam` in een bepaalde `taal` begroet

```go
    func Hello(name, language string) string {

      if language == "es" {
         return "Hola, " + name
      }

      if language == "fr" {
         return "Bonjour, " + name
      }

      // imagine dozens more languages

      return "Hello, " + name
    }
```

Het voelt niet prettig om tientallen `if`-instructies te hebben en we krijgen een duplicatie van het samenvoegen van een taalspecifieke begroeting met `, ` en de `name.` Daarom ga je de code herstructureren.

```go
    func Hello(name, language string) string {
      	return fmt.Sprintf(
      		"%s, %s",
      		greeting(language),
      		name,
      	)
    }

    var greetings = map[string]string {
      "es": "Hola",
      "fr": "Bonjour",
      //etc..
    }

    func greeting(language string) string {
      greeting, exists := greetings[language]

      if exists {
         return greeting
      }

      return "Hello"
    }
```

De aard van deze refactoring is eigenlijk niet belangrijk, wat belangrijk is, is dat ik het gedrag niet heb veranderd.

Bij refactoring kun je doen wat je wilt: interfaces, nieuwe typen, functies, methoden, enz. toevoegen. De enige regel is dat je het gedrag niet verandert.

### Bij het refactoren van code mag je het gedrag niet veranderen

Dit is erg belangrijk. Als je tegelijkertijd het gedrag verandert, doe je _twee_ dingen tegelijk. Als software engineers leren we systemen op te splitsen in verschillende bestanden/pakketten/functies/enz., omdat we weten dat het moeilijk is om een grote hoeveelheid informatie te begrijpen.

We willen niet aan veel dingen tegelijk hoeven te denken, want dan maken we fouten. Ik heb zoveel refactoring-pogingen zien mislukken omdat ontwikkelaars te veel hooi op hun vork nemen.

Toen ik in wiskundelessen factorisaties deed met pen en papier, moest ik handmatig controleren of ik de betekenis van de uitdrukkingen in mijn hoofd niet had veranderd. Hoe weten we dat we het gedrag niet veranderen tijdens het refactoren van code, vooral op een systeem dat niet triviaal is?

Degenen die ervoor kiezen om geen tests te schrijven, zijn doorgaans afhankelijk van handmatige tests. Voor projecten die niet klein zijn, is dit enorm tijdrovend en op de lange termijn is het niet schaalbaar.

**Om veilig te kunnen refactoren heb je unittests nodig** omdat ze zorgen voor:

- Het vertrouwen dat je code kunt aanpassen zonder je zorgen te maken over gedragsverandering
- Documentatie voor mensen over hoe het systeem zich zou moeten gedragen
- Veel snellere en betrouwbaardere feedback dan handmatig testen

#### Een voorbeeld in Go

Een unittest voor onze `Hello`-functie zou er zo uit kunnen zien

```go

    func TestHello(t *testing.T) {
      got := Hello(“Chris”, es)
      want := "Hola, Chris"

      if got != want {
         t.Errorf("got %q want %q", got, want)
      }
    }
```

Op de opdrachtregel kan ik `go test` uitvoeren en direct feedback krijgen of mijn refactoring-inspanningen het gedrag hebben veranderd. In de praktijk is het het beste om de magische knop te leren om je tests binnen je editor/IDE uit te voeren.

Je wilt in een staat komen waarin je bezig bent met

- Kleine refactoring
- Tests uitvoeren
- Herhalen

Alles binnen een zeer strakke feedbacklus, zodat je niet in een valkuil terechtkomt en fouten maakt.

Een project waarbij al je belangrijkste gedragingen unit-tests ondergaan en je ruim binnen een seconde feedback krijgt, is een zeer krachtig vangnet om gedurfde refactoring uit te voeren wanneer dat nodig is. Dit helpt ons de binnenkomende complexiteit, zoals Lehman het beschrijft, te beheersen.

## Als unit tests zo geweldig zijn, waarom is er dan soms weerstand tegen het schrijven ervan?

Aan de ene kant heb je mensen (zoals ik) die zeggen dat unit tests belangrijk zijn voor de gezondheid van je systeem op de lange termijn, omdat ze ervoor zorgen dat je met vertrouwen kunt blijven refactoren.

Aan de andere kant heb je mensen die ervaringen beschrijven waarin unit tests refactoring juist _belemmerden_.

Vraag jezelf af: hoe vaak moet je je tests aanpassen tijdens het refactoren? In de loop der jaren heb ik aan veel projecten gewerkt met een zeer goede testdekking, maar toch aarzelen de engineers om te refactoren vanwege de vermeende inspanning die het aanpassen van tests met zich meebrengt.

Dit is het tegenovergestelde van wat ons beloofd is!

### Waarom gebeurt dit?

Stel je voor dat je gevraagd wordt een vierkant te ontwerpen en wij denken dat de beste manier om dat te bereiken is door twee driehoeken aan elkaar te plakken.

![Twee rechthoekige driehoeken om een ​​vierkant te vormen](https://i.imgur.com/ela7SVf.jpg)

We schrijven onze unit tests rond ons vierkant om er zeker van te zijn dat de zijden gelijk zijn en vervolgens schrijven we een aantal tests rond onze driehoeken. We willen er zeker van zijn dat onze driehoeken correct worden weergegeven, dus we stellen dat de hoeken samen 180 graden zijn, controleren misschien of we er 2 maken, enz. enz. Testdekking is erg belangrijk en het schrijven van deze tests is vrij eenvoudig, dus waarom niet?

Een paar weken later slaat de Wet van Continue Verandering toe in ons systeem en een nieuwe ontwikkelaar brengt een aantal wijzigingen aan. Ze gelooft nu dat het beter zou zijn als vierkanten worden gevormd met 2 rechthoeken in plaats van 2 driehoeken.

![Twee rechthoeken vormen een vierkant](https://i.imgur.com/1G6rYqD.jpg)

Ze probeert deze refactoring uit te voeren en krijgt gemengde signalen van een aantal mislukte tests. Heeft ze hier daadwerkelijk belangrijke gedragingen overtreden? Ze moet nu deze driehoektests doorspitten en proberen te begrijpen wat er aan de hand is.

_Het is niet echt belangrijk dat het vierkant uit driehoeken is gevormd_, maar **onze tests hebben het belang van onze implementatiedetails ten onrechte vergroot**.

## Geef de voorkeur aan testgedrag boven implementatiedetails

Als ik mensen hoor klagen over unittests, komt dat vaak doordat de tests zich op het verkeerde abstractieniveau bevinden. Ze testen implementatiedetails, bespioneren te veel en gebruiken teveel mocks.

Ik denk dat dit voortkomt uit een verkeerd begrip van wat unittests zijn en het najagen van ijdele metrics (testdekking).

Als ik zeg dat het alleen om testgedrag gaat, moeten we dan niet gewoon alleen systeem-/black-boxtests schrijven? Dit soort tests zijn zeer waardevol voor het verifiëren van de belangrijkste gebruikerservaringen, maar ze zijn meestal duur om te schrijven en traag in uitvoering. Daarom zijn ze niet erg nuttig voor _refactoring_, omdat de feedbacklus traag is. Bovendien helpen black-boxtests je over het algemeen niet veel met het vinden van root causes in vergelijking met unittests.

Dus wat _is_ het juiste abstractieniveau?

## Het schrijven van effectieve unittests is een ontwerpprobleem

Laat ik de tests even buiten beschouwing: het is wenselijk om binnen je systeem zelfstandige, losgekoppelde "units" te hebben die gecentreerd zijn rond sleutelconcepten in je domein.

Ik stel me deze units graag voor als simpele Legoblokjes met coherente API's die ik kan combineren met andere blokjes om grotere systemen te maken. Onder deze API's kunnen tientallen dingen (typen, functies, enzovoort) samenwerken om ze te laten werken zoals ze moeten.

Als je bijvoorbeeld een bank in Go schrijft, zou je een "account"-pakket kunnen hebben. Dit presenteert een API die geen implementatiedetails lekt en eenvoudig te integreren is.

Als je deze units hebt die aan deze eigenschappen voldoen, kun je unittests schrijven tegen hun publieke API's. _Per definitie_ kunnen deze tests alleen nuttig gedrag testen. Onder deze units ben ik vrij om de implementatie zo vaak te refactoren als nodig is en de tests zouden over het algemeen niet in de weg moeten zitten.

### Zijn dit unittests?

**JA**. Unittests zijn gericht tegen "units" zoals ik al zei. Ze gingen _nooit_ alleen over het testen tegen één klasse/functie/wat dan ook.

## Deze concepten samenbrengen

We hebben het gehad over

- Refactoring
- Unittests
- Unitontwerp

Wat we beginnen te zien, is dat deze facetten van softwareontwerp elkaar versterken.

### Refactoring

- Geeft ons signalen over onze unittests. Als we handmatige controles moeten uitvoeren, hebben we meer tests nodig. Als tests ten onrechte falen, bevinden onze tests zich op het verkeerde abstractieniveau (of hebben ze geen waarde en moeten ze worden verwijderd).
- Helpt ons de complexiteit binnen en tussen onze units te beheersen.

### Unit tests

- Bieden een vangnet voor refactoren.
- Verifieren en documenteren het gedrag van onze units.

### (Goed ontworpen) units

- Gemakkelijk om zinvolle unit tests voor te schrijven.
- Gemakkelijk te refactoren.

Bestaat er een proces dat ons helpt om onze code continu te refactoren om de complexiteit te beheersen en onze systemen flexibel te houden?

## Waarom Test Driven Development (TDD)

Sommige mensen nemen Lehmans citaten over hoe software moet veranderen en denken te veel na over uitgebreide ontwerpen, waardoor ze veel tijd verspillen aan het proberen te creëren van het "perfecte" uitbreidbare systeem, maar uiteindelijk de fout ingaan en nergens komen.

Dit zijn de slechte oude tijden van software, waarin een analistenteam 6 maanden besteedde aan het schrijven van een requirementsdocument en een architectenteam nog eens 6 maanden aan het bedenken van een ontwerp, en een paar jaar later het hele project mislukte.

Ik zeg slechte oude tijden, maar dit gebeurt nog steeds!

Agile leert ons dat we iteratief moeten werken, klein moeten beginnen en de software moeten doorontwikkelen, zodat we snel feedback krijgen op het ontwerp van onze software en hoe deze werkt met echte gebruikers; TDD dwingt deze aanpak af.

TDD pakt de wetten aan waar Lehman het over heeft en andere lessen die we door de geschiedenis heen hebben geleerd, door een methodologie aan te moedigen van constant refactoren en iteratief leveren.

### Kleine stapjes

- Schrijf een kleine test voor een kleine hoeveelheid gewenst gedrag
- Controleer of de test mislukt met een duidelijke fout (rood)
- Schrijf de minimale hoeveelheid code om de test te laten slagen (groen)
- Refactor
- Herhaal

Naarmate je bedrevener wordt, zal deze manier van werken natuurlijk en snel worden.

Je zult verwachten dat deze feedbacklus niet erg lang duurt en je zult je ongemakkelijk voelen als je in een staat verkeert waarin het systeem niet "groen" is, omdat dit aangeeft dat je mogelijk in een konijnenhol terecht bent gekomen.

Je zult altijd kleine en nuttige functionaliteit ontwikkelen, ondersteund door de feedback van je tests.

## Samenvattend

- De kracht van software is dat we het kunnen veranderen. _De meeste_ software zal in de loop der tijd op onvoorspelbare manieren moeten worden aangepast; maar probeer niet te over-engineeren, want het is te moeilijk om de toekomst te voorspellen.
- In plaats daarvan moeten we het zo maken dat we onze software kneedbaar kunnen houden. Om software te veranderen, moeten we deze refactoren naarmate deze evolueert, anders wordt het een puinhoop.
- Een goede testsuite kan je helpen om sneller en minder stressvol te refactoren.
- Het schrijven van goede unit tests is een ontwerpprobleem, dus denk na over het structureren van je code zodat je zinvolle units hebt die je als legoblokjes kunt integreren.
- TDD kan je helpen en dwingen om goed gefactoreerde software iteratief te ontwerpen, ondersteund door tests om toekomstig werk te ondersteunen zodra het beschikbaar is.
