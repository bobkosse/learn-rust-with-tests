# Hello, World

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/hello-world)

Het is een traditie om de eerste stappen in een nieuwe programmeertaal te zetten met het programma [Hello, World](https://en.m.wikipedia.org/wiki/%22Hello,_World!%22_program). Dit is zelfs zo'n traditie dat ik er in deze vertaling niet eens "Hallo, Wereld" van gemaakt heb.

* Maak een map aan op je computer. Het maakt niet uit waar
* Maak een bestand aan met de naam `hello.go` in die map en plaats in het bestand de volgende code:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, world")
}
```

Ga in je terminal naar de map die je aangemaakt hebt en typ: `go run hello.go`.

## Hoe dit werkt

Wanneer je een programma in Go schrijft, heb je een `main` gedefinieerd met daarin een `main` functie. Pakketten zijn manieren om gerelateerde Go-code te groeperen.

Het `func` keyword definieert een functie met een naam en een body.

Met `import "fmt"` importeren we een pakket dat de `Println` functie bevat. Deze gebruiken we om tekst op het scherm te printen.

## Hoe test je dit?

Hoe test je dit? Het is verstandig om "domein" code los te maken van de buiten wereld (de bij-effecten). De `fmt.Println` noemen we een bij-effect (het printen naar stdout). De tekst (string) die we naar de `fmt.println` functie sturen is ons domain.

Door domein-code en bij-effecten te splitsen, wordt de code eenvoudiger om te testen.

```go
package main

import "fmt"

func Hello() string {
	return "Hello, world"
}

func main() {
	fmt.Println(Hello())
}
```

We hebben nu een nieuwe functie aangemaakt met `func`, maar deze keer hebben we een nieuw sleutelwoord, `string`, toegevoegd aan de definitie. Dit betekent dat de functie als resultaat een `string` retourneert.

Maak nu een nieuw bestand aan met de naam `hello_test.go` . Hierin gaan we een test schrijven om onze `Hello` functie te testen.

```go
package main

import "testing"

func TestHello(t *testing.T) {
	got := Hello()
	want := "Hello, world"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

## Go modules?

De volgende stap is om de test uit te voeren. Type `go test` in je terminal. Als de test slaagt, heb je waarschijnlijk een oudere versie van Go. Maar, als je Go 1.16 of nieuwer gebruikt, is het waarschijnlijker dat de test niet slaagt. In plaats van een geslaagde test krijg je een foutmelding zoals deze te zien in je terminal:

```shell
$ go test
go: cannot find main module; see 'go help modules'
```

Wat is hier het probleem? In een woord: [modules](https://blog.golang.org/go116-module-changes). Gelukkig is dit probleem eenvoudig op te lossen. Type  `go mod init example.com/hello` in je terminal. Dat maakt een nieuw bestand aan met de volgende inhoud:

```
module example.com/hello

go 1.16
```

Dit bestand geeft de Go-tools essentiële informatie over je code. Als je van plan bent je applicatie te distribueren, moet je vermelden waar de code beschikbaar is om te downloaden, evenals informatie over afhankelijkheden. De naam van de module, example.com/hello, verwijst meestal naar een URL waar de module kan worden gevonden en gedownload. Voor compatibiliteit met tools die we binnenkort gaan gebruiken, zorg ervoor dat de naam van je module ergens een punt bevat, zoals de punt in .com van example.com/hello. Voorlopig is je modulebestand minimaal en kun je het zo laten. Voor meer informatie over modules kun je de [referentie in de Golang-documentatie raadplegen](https://golang.org/doc/modules/gomod-ref). We kunnen nu verder met het testen en leren van Go, aangezien de tests zouden moeten werken, zelfs op Go 1.16.

In toekomstige hoofdstukken, zul je `go mod init MODULENAAM` in elke nieuwe map moeten uitvoeren, voordat je commando's als `go test` of `go build` kunt uitvoeren.

## Terug naar testen

Voer het commando `run test` nogmaals uit via de terminal. Nu zouden de tests moeten slagen! Om dit te controleren of het echt goed werkt, kun je proberen de test opzettelijk te laten falen door de `want`-string te wijzigen.

Merk op dat je niet hoeft te kiezen tussen meerdere test frameworks en vervolgens hoeft uit te zoeken hoe je ze moet installeren. Alles wat je nodig hebt, is ingebouwd in de taal en de syntaxis is hetzelfde als de rest van de code die je schrijft.

### Een test schrijven

Een test schrijven is vergelijkbaar met het schrijven van een functie, met een paar extra regels:

* De functie moet in een bestand staan dat het volgende format volgt als naam `xxx_test.go`
* De test functie moet starten met het woord `Test`
* De test functie krijgt slechts 1 argument mee `t *testing.T`
* Om het type  `*testing.T` te kunnen gebruiken moet je `import "testing"`, bovenaan je code zetten, net zoals we deden met `fmt` in onze eerste bestand

Voor nu is het voldoende om te weten dat  `t` van het type `*testing.T` je "haak" is met het test framework. Dit zorgt er voor dat je dingen kunt doen als `t.Fail()` wanneer je een test wilt laten falen.

We behandelen ook een paar nieuwe onderwerpen:

#### `if`

If commando's zijn in Go vergelijkbaar met If commando's in andere programmeertalen.

#### Variabelen declareren

We declareren een paar variabelen met de syntax `varName := value`. Dit helpt ons om de test leesbaarder te maken door waarden opnieuw te gebruiken.

#### `t.Errorf`

We roepen de `Errorf`-methode aan op onze `t`. Dit zal een bericht afdrukken en de test laten mislukken. De `f` staat voor format, waarmee we een string kunnen samenstellen met waarden die in de tijdelijke aanduiding `%q` zijn ingevoegd. Wanneer je de test laat mislukken zul je vrij snel zien hoe dit werkt.

Meer informatie over de tijdelijke tekenreeksen vindt je in de [fmt-documentatie](https://pkg.go.dev/fmt#hdr-Printing). Voor tests is `%q` erg handig, omdat het je waarden tussen dubbele aanhalingstekens plaatst. Dit maakt het makkelijker deze in de terminal te lezen.

Later zullen we het verschil tussen methodes en functies ontdekken.

### Go's documentatie

Een andere kwaliteitsfunctie van Go is de documentatie. We hebben zojuist de documentatie voor het fmt-pakket bekeken op de officiële website voor pakketweergave, en Go biedt ook mogelijkheden om de documentatie snel offline te raadplegen.

Go heeft een ingebouwde tool, doc, waarmee je elk pakket dat op je systeem is geïnstalleerd, of de module waaraan je momenteel werkt, kunt bekijken. Om dezelfde documentatie voor de print opties te bekijken:

```
$ go doc fmt
package fmt // import "fmt"

Package fmt implements formatted I/O with functions analogous to C's printf and
scanf. The format 'verbs' are derived from C's but are simpler.

# Printing

The verbs:

General:

    %v	the value in a default format
    	when printing structs, the plus flag (%+v) adds field names
    %#v	a Go-syntax representation of the value
    %T	a Go-syntax representation of the type of the value
    %%	a literal percent sign; consumes no value
...
```

Go's tweede tool voor het bekijken van documentatie is de pkgsite-opdracht, die de officiële website voor het bekijken van pakketten van Go aanstuurt. Je kunt pkgsite installeren met `go install golang.org/x/pkgsite/cmd/pkgsite@latest` en het vervolgens uitvoeren met `pkgsite -open .`. De installatieopdracht van Go downloadt de bronbestanden uit de repository en bouwt ze om tot een uitvoerbaar bestand. Voor een standaardinstallatie van Go staat dat uitvoerbare bestand in `$HOME/go/bin` voor Linux en macOS, en `%USERPROFILE%\go\bin` voor Windows. Als je deze paden nog niet aan je $PATH-variabele hebt toegevoegd, kun je dat doen om het uitvoeren van via go geïnstalleerde opdrachten te vereenvoudigen.

Het overgrote deel van de standaardbibliotheek beschikt over uitstekende documentatie met voorbeelden. Het is de moeite waard om naar [http://localhost:8080/testing](http://localhost:8080/testing) te gaan om te zien wat er beschikbaar is.

### Hallo, Jij

Nu we een test hebben, kunnen we veilig itereren op onze software.

In het laatste voorbeeld hebben we de test geschreven nadat de code was geschreven, zodat je een voorbeeld krijgt van hoe je een test schrijft en een functie declareert. Vanaf nu schrijven we eerst tests.

Onze volgende vereiste is dat we de ontvanger van de begroeting kunnen specificeren.

Laten we beginnen met het vastleggen van deze vereisten in een test. Dit is basis test-driven development en stelt ons in staat om ervoor te zorgen dat onze test daadwerkelijk test wat we willen. Wanneer je achteraf tests schrijft, bestaat het risico dat je test alsnog slaagt, zelfs als de code niet werkt zoals bedoeld. Je schrijft dan de test naar het gewenst resultaat toe, in plaats van de code.

```go
package main

import "testing"

func TestHello(t *testing.T) {
	got := Hello("Chris")
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Voer nu `go test` uit. Je zou daarbij een compilatie fout moeten krijgen

```
./hello_test.go:6:18: too many arguments in call to Hello
    have (string)
    want ()
```

Bij het gebruik van een statisch getypeerde taal zoals Go is het belangrijk om naar de _compiler te luisteren_. De compiler begrijpt hoe je code in elkaar moet zitten en moet werken, zodat jij dat niet hoeft te doen.

In dit geval vertelt de compiler je wat je moet doen om verder te gaan. We moeten onze functie `Hello` aanpassen om een argument te accepteren.

Bewerk de `Hello`-functie zodat deze een argument van het type string accepteert

```go
func Hello(name string) string {
	return "Hello, world"
}
```

Als je je tests opnieuw probeert uit te voeren, mislukt de compilatie van `hello.go` omdat je geen argument meegeeft. Stuur "world" mee in je `main`-functie om de compilatie te laten slagen.

```go
func main() {
	fmt.Println(Hello("world"))
}
```

Wanneer je nu je tests uitvoert, zou je iets moeten zien als:

```
hello_test.go:10: got 'Hello, world' want 'Hello, Chris''
```

We hebben eindelijk een compilerend programma, maar het voldoet niet aan onze eisen volgens de test.\
\
Laten we de test laten slagen door het argument 'name' te gebruiken en dit samen te voegen met `'Hello,'`

```go
func Hello(name string) string {
	return "Hello, " + name
}
```

Wanneer je de tests uitvoert, zouden ze nu moeten slagen. Normaal gesproken zouden we, als onderdeel van de TDD-cyclus, nu moeten _refactoren_.

### Een opmerking over source control

Op dit punt, als je source control (bijvoorbeeld Git) gebruikt (wat je zou moeten doen!), zou ik de code committen zoals die is. We hebben werkende software, ondersteund door een test.

Ik zou echter niet pushen naar de _main branch_, omdat ik van plan ben om de volgende keer te refactoren. Het is handig om nu alvast te committen, mocht je op de een of andere manier in de problemen komen met de refactoring, kun je altijd terug naar de werkende versie.

Er valt hier niet veel te veranderen, maar we kunnen wel een andere taalfunctie introduceren: _constanten_.

### Constanten

Constanten worden als volgt gedefinieerd

```go
const englishHelloPrefix = "Hello, "
```

We kunnen nu onze code refactoren:

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
	return englishHelloPrefix + name
}
```

Voer na het refactoren de tests opnieuw uit om er zeker van te zijn dat je niets kapot hebt gemaakt.

Het is de moeite waard om na te denken over het creëren van constanten om de betekenis van waarden vast te leggen en soms om de prestaties te verbeteren.

## Hello, world... nog een keer

De volgende vereiste is dat wanneer onze functie wordt aangeroepen met een lege string, deze standaard "Hello, World" afdrukt in plaats van "Hello, ".

Start met het schrijven van een falende test:

```go
func TestHello(t *testing.T) {
	t.Run("saying hello to people", func(t *testing.T) {
		got := Hello("Chris")
		want := "Hello, Chris"

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})
	t.Run("say 'Hello, World' when an empty string is supplied", func(t *testing.T) {
		got := Hello("")
		want := "Hello, World"

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})
}
```

Hier introduceren we een nieuwe tool in ons testarsenaal: subtests. Soms is het handig om tests rond een 'ding' te groeperen en vervolgens subtests te hebben die verschillende scenario's beschrijven.

Een voordeel van deze aanpak is dat je gedeelde code kunt opzetten die in de andere tests kan worden gebruikt.

Nu de test mislukt is, gaan we de code repareren met behulp van een `if`.

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
	if name == "" {
		name = "World"
	}
	return englishHelloPrefix + name
}
```

Als we onze tests uitvoeren, zien we dat het aan de nieuwe vereisten voldoet en dat we niet per ongeluk de andere functionaliteit hebben gebroken.

Het is belangrijk dat je tests _duidelijke specificaties_ bevatten van wat de code moet doen. Maar er is sprake van herhaalde code wanneer we controleren of de boodschap overeenkomt met wat we verwachten.

Refactoring gaat _niet alleen_ op voor productie code!

Nu de tests succesvol zijn, kunnen en moeten we onze tests refactoren.

```go
func TestHello(t *testing.T) {
	t.Run("saying hello to people", func(t *testing.T) {
		got := Hello("Chris")
		want := "Hello, Chris"
		assertCorrectMessage(t, got, want)
	})

	t.Run("empty string defaults to 'world'", func(t *testing.T) {
		got := Hello("")
		want := "Hello, World"
		assertCorrectMessage(t, got, want)
	})

}

func assertCorrectMessage(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Wat hebben we hier gedaan?

We hebben onze vergelijking geherstructureerd tot een nieuwe functie. Dit vermindert duplicatie en verbetert de leesbaarheid van onze tests. We moeten `t *testing.T` doorgeven, zodat we de testcode kunnen laten falen wanneer dat nodig is.

Voor hulpfuncties is het een goed idee om een `testing.TB` te accepteren. Dit is een interface waaraan `*testing.T` en `*testing.B` beide voldoen, zodat je hulpfuncties kunt aanroepen vanuit een test of een benchmark (maak je geen zorgen als woorden als "interface" je nu nog niets zeggen, dat komt later aan bod).

`t.Helper()` is nodig om de testsuite te laten weten dat deze methode een helper is. Hierdoor zal, wanneer de methode faalt, het gerapporteerde regelnummer in onze functieaanroep staan in plaats van in onze testhelper. Dit helpt andere ontwikkelaars om problemen gemakkelijker op te sporen. Als je dit nog steeds niet helemaal begrijpt, maak dan commentaar van deze regel, laat een test mislukken en bekijk de testuitvoer. Commentaar in Go is een geweldige manier om extra informatie aan je code toe te voegen, of in dit geval een snelle manier om de compiler te vertellen een regel te negeren. Je kunt de `t.Helper()`-code uitschakelen door er commentaar van te maken. Dat doe je door twee slashes `//` aan het begin van de regel toe te voegen. Je zou die regel grijs moeten zien worden of een andere kleur moeten krijgen dan de rest van je code, om aan te geven dat deze nu is uitgecommentarieerd.

Wanneer je meer dan één argument van hetzelfde type hebt (in ons geval twee strings), kun je in plaats van `(got string, want string)` dit verkorten tot `(got, want string)` in de functie aanroep.

### Terug naar source control

Nu we tevreden zijn met de code, zou ik de vorige commit aanpassen zodat we alleen de 'mooie' versie van onze code met zijn test inchecken.

### Discipline

Laten we de hele cyclus van TDD eens bekijken:

* Schrijf een test
* Zorg dat de compiler slaagt
* Voer de test uit, bekijk wat er faalt en controleer of de foutmelding iets waardevols vertelt
* Schrijf voldoende code om de test te laten slagen
* Refactor

Op het eerste gezicht lijkt dit misschien vervelend, maar het is belangrijk om je aan de feedback loop te houden.

Hiermee ben je er niet alleen zeker van dat je _relevante tests_ hebt, maar helpt je ook bij _het ontwerpen van goede software_ door te refactoren met de veiligheid van tests.

Het zien van een mislukte test is een belangrijke controle, omdat je dan ook kunt zien hoe de foutmelding eruitziet. Als ontwikkelaar kan het erg lastig zijn om met een codebase te werken wanneer mislukte tests geen duidelijk beeld geven van wat het probleem is.

Door ervoor te zorgen dat je tests snel zijn en je hulpmiddelen zo in te stellen dat het uitvoeren van tests eenvoudig is, kun je in een flow komen tijdens het schrijven van code.

Door geen tests te schrijven, verplicht je je tot het handmatig controleren van je code door je software te draaien, wat je flow verstoort. Met het achterwegen laten van testen bespaart jezelf geen tijd, zeker niet op de lange termijn.

## We gaan door! Meer vereisten

Hemeltjelief, we hebben meer vereisten. We moeten nu een tweede parameter ondersteunen, die de taal van de begroeting specificeert. Als er een taal wordt doorgegeven die we niet herkennen, wordt standaard Engels gebruikt.

We zijn er zeker van dat we TDD eenvoudig kunnen gebruiken om deze functionaliteit uit te breiden!

Schrijf een test voor de situatie waar een gebruiker de taal Spaans meegeeft (Spanish in de code). Voeg de test toe aan de bestaande testsuite.

```go
	t.Run("in Spanish", func(t *testing.T) {
		got := Hello("Elodie", "Spanish")
		want := "Hola, Elodie"
		assertCorrectMessage(t, got, want)
	})
```

Onthoud dat je niet valsspeelt! _Eerst testen._ Wanneer je probeert de test uit te voeren, _zou_ de compiler moeten klagen omdat we `Hello` aanroepen met twee parameters in plaats van één.&#x20;

```
./hello_test.go:27:19: too many arguments in call to Hello
    have (string, string)
    want (string)
```

Repareer de compilatie problemen door een nieuw string argument toe te voegen aan `Hello`

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}
	return englishHelloPrefix + name
}
```

Wanneer je de test opnieuw probeert uit te voeren, zal deze klagen dat er in je andere tests en in `hello.go` niet voldoende argumenten zijn doorgegeven aan `Hello`.

```
./hello.go:15:19: not enough arguments in call to Hello
    have (string)
    want (string, string)
```

Los dit op door lege strings door te geven. Nu zouden al je tests moeten compileren en slagen, behalve ons nieuwe scenario.

```
hello_test.go:29: got 'Hello, Elodie' want 'Hola, Elodie'
```

We kunnen hier `if` gebruiken om te controleren of de taal gelijk is aan 'Spanish' en zo ja, het bericht wijzigen

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	if language == "Spanish" {
		return "Hola, " + name
	}
	return englishHelloPrefix + name
}
```

De test zou nu moeten slagen.

Nu is het tijd om te _refactoren_. Je zou wat problemen in de code moeten zien, "magische" strings, waarvan sommige herhaald worden. Probeer het zelf te refactoren. Zorg ervoor dat je bij elke wijziging de tests opnieuw uitvoert om er zeker van te zijn dat je refactoring niets kapotmaakt.

```go
	const spanish = "Spanish"
	const englishHelloPrefix = "Hello, "
	const spanishHelloPrefix = "Hola, "

	func Hello(name string, language string) string {
		if name == "" {
			name = "World"
		}

		if language == spanish {
			return spanishHelloPrefix + name
		}
		return englishHelloPrefix + name
	}
```

### Frans (French in de code)

* Schrijf een test waarin je beweert dat als je slaagt voor `French`, je `Bonjour ,` krijgt.
* Zie het mislukken, controleer of de foutmelding gemakkelijk te lezen is
* Voer de kleinste redelijke verandering in de code door

Je hebt misschien iets geschreven dat er ongeveer zo uitziet

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	if language == spanish {
		return spanishHelloPrefix + name
	}
	if language == french {
		return frenchHelloPrefix + name
	}
	return englishHelloPrefix + name
}
```

## `switch`

Wanneer je veel `if`-statements hebt die een bepaalde waarde controleren, is het gebruikelijk om in plaats daarvan een `switch`-statement te gebruiken. We kunnen `switch` gebruiken om de code te refactoren, zodat deze beter leesbaar en uitbreidbaar wordt, mocht je later meer taalondersteuning willen toevoegen.

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	prefix := englishHelloPrefix

	switch language {
	case spanish:
		prefix = spanishHelloPrefix
	case french:
		prefix = frenchHelloPrefix
	}

	return prefix + name
}
```

Schrijf nu een test om een begroeting in de taal van je keuze toe te voegen en je zult zien hoe eenvoudig het is om onze _geweldige_ functionaliteit uit te breiden.

### een...laatste...refactor?

Je zou kunnen stellen dat onze functie misschien een beetje groot wordt. De eenvoudigste refactoring hiervoor zou zijn om wat functionaliteit over te brengen naar een andere functie.

```go

const (
	spanish = "Spanish"
	french  = "French"

	englishHelloPrefix = "Hello, "
	spanishHelloPrefix = "Hola, "
	frenchHelloPrefix  = "Bonjour, "
)

func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	return greetingPrefix(language) + name
}

func greetingPrefix(language string) (prefix string) {
	switch language {
	case french:
		prefix = frenchHelloPrefix
	case spanish:
		prefix = spanishHelloPrefix
	default:
		prefix = englishHelloPrefix
	}
	return
}
```

Een paar nieuwe concepten:

* In onze functiesignatuur hebben we een benoemde retourwaarde `(prefix string)` gemaakt.
* Hiermee wordt een variabele met de naam `prefix` binnen je functie aangemaakt.
  * Er wordt de waarde "zero" aan toegekend. Dit is afhankelijk van het type, bijvoorbeeld `int`s zijn 0 en voor `string`s is het `""`.
  * Je kunt de ingestelde waarde retourneren door gewoon `return` aan te roepen in plaats van `return prefix`.
  * Dit wordt weergegeven in het Go Doc voor je functie, zodat de bedoeling van je code duidelijker wordt.
* `default` is een tak binnen switch waar naartoe verwezen word als geen van de andere `case` statements matcht.
* De functienaam begint met een kleine letter. In Go beginnen `public` functies met een hoofdletter en `private` functies met een kleine letter. We willen niet dat de interne werking van ons algoritme openbaar wordt gemaakt, dus hebben we deze functie privé gemaakt.
* We kunnen constanten ook in een blok groeperen in plaats van ze op een aparte regel te declareren. Voor de leesbaarheid is het een goed idee om een regel te gebruiken tussen sets van gerelateerde constanten.

## Afronden

Wie had gedacht dat je zoveel kon leren uit het simpele `Hello, world` programma?

Tot nu toe zou je begrip moeten hebben van:

### Een deel van de syntaxis van Go

* Schrijven van tests
* Functies declareren, met argumenten en return types
* `if`, `const` en `switch`
* Variable en constanten declareren

### Het TDD-proces en waarom de stappen belangrijk zijn

* _Schrijf een falende test en zie deze falen_, zodat we weten dat we een _relevante test_ voor onze vereisten hebben geschreven en dat deze een _gemakkelijk te begrijpen beschrijving van de mislukking_ oplevert.
* Het schrijven van de kleinste hoeveelheid code om de test te laten slagen, zodat we weten dat we werkende software hebben
* Vervolgens _refactoren_, ondersteund door de veiligheid van onze tests, om te garanderen dat we goed geschreven code hebben waarmee gemakkelijk te werken is

In ons geval zijn we van `Hello()` naar `Hello("name")` en vervolgens naar `Hello("name", "French")` gegaan in kleine, gemakkelijk te begrijpen stappen.

Natuurlijk is dit basaal vergeleken met "echte" software, maar de principes blijven overeind. TDD is een vaardigheid die oefening vereist om te ontwikkelen, maar door problemen op te delen in kleinere componenten die je kunt testen, wordt het veel gemakkelijker om software te schrijven.
