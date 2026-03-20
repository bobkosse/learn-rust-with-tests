# Integers

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/integers)

Integers werken zoals je zou verwachten. Laten we een `Add` functie schrijven om wat dingen te proberen. Maak eerst een bestand aan met de naam `adder_test.go` en schrijf de code voor de test.

**Note:** Go bronbestanden kunnen slechts één `package` per directory hebben. Verzeker je zelf ervan dat je bestanden zijn georganiseerd binnen hun eigen packages. [Hier is een goede uitleg hoe je dat doet](https://dave.cheney.net/2014/12/01/five-suggestions-for-setting-up-a-go-project).

Je project directory zou er nu zo uit moeten zien:

```
learnGoWithTests
    |
    |-> helloworld
    |    |- hello.go
    |    |- hello_test.go
    |
    |-> integers
    |    |- adder_test.go
    |
    |- go.mod
    |- README.md
```

## Schrijf eerst je test

```go
package integers

import "testing"

func TestAdder(t *testing.T) {
	sum := Add(2, 2)
	expected := 4

	if sum != expected {
		t.Errorf("expected '%d' but got '%d'", expected, sum)
	}
}
```

Het valt je misschien op dat we nu `%d` gebruiken als format strings, in plaats van `%q`. Dat komt omdat we een integer waarde willen printen in plaats van een string.

Merk ook op dat we niet langer de main package gebruiken. In plaats daarvan definieren we een package met de naam `integers`. Zoals de naam al doet vermoeden, dit groepeert functies (zoals onze `Add` functie) om met integers te werken samen.

## Probeer de test uit te voeren

Voer de test uit met het commando: `go test`

Inspecteer de compilatie fout:

`./adder_test.go:6:9: undefined: Add`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Schrijf voldoende code om de compiler te laten werken, _en niet meer!_ Onthoud dat we willen controleren of onze tests falen om de juiste reden!

Maak een nieuw bestand aan in de integers directory met de naam `integers.go` en voer de volgende code in:

```go
package integers

func Add(x, y int) int {
	return 0
}
```

Onthoud, wanneer je meer dan 1 argument hebt van hetzelfde type (in ons geval 2 integers) kun je, in plaats van `(x int, y int)` dit korter schrijven als `(x, y int)`.

Voer de test opnieuw uit en we zouden moeten zien dat de test goed draait en netjes aangeeft wat er fout is.

`adder_test.go:10: expected '4' but got '0'`

Misschien heb je gemerkt dat we in de [vorige](hello-world.md#een...laatste...refactor) sectie over _named return value_ hebben geleerd, maar dat je die hier niet gebruikt. Named return values zouden over het algemeen gebruikt moeten worden wanneer de betekenis van het resultaat niet duidelijk is uit de context. In ons geval is het vrijwel duidelijk dat de functie `Add` de opgegeven parameters zal optellen. Raadpleeg [deze](https://go.dev/wiki/CodeReviewComments#named-result-parameters) wiki voor meer informatie.

## Schrijf genoeg code om de test te laten slagen

Volgens de meest strikte definitie van TDD moeten nu de _minimale hoeveelheid code schrijven om te test te laten slagen_. Een programmeur die zich heel strikt aan de regels wil houden zou nu iets kunnen schrijven als:

```go
func Add(x, y int) int {
	return 4
}
```

Augh, is TDD dan toch niet zo'n goed idee?

We kunnen nu een andere test schrijven, met andere getallen, om de test weer te laten falen maar dat voelt toch wel als [een spelletje kat en muis](https://en.m.wikipedia.org/wiki/Cat_and_mouse).

Zodra we wat bekender zijn met de syntax van Go, zal ik een techniek genaamd "_Property Based Testing_" introduceren, wat vervelende ontwikkelaars tegenhoud en je helpt om fouten te vinden.

Voor nu, laten we de `Add` functie op de juiste manier schrijven:

```go
func Add(x, y int) int {
	return x + y
}
```

Wanneer je de test nu opnieuw uitvoert, zou deze moeten slagen.

## Refactor

Er is niet veel in de code dat we echt kunnen verbeteren.

We hebben eerder besproken hoe het return-argument wordt weergegeven in de documentatie, maar ook in de meeste teksteditors voor ontwikkelaars.

Dit is geweldig, omdat het de bruikbaarheid van de code die je schrijft ten goede komt. Het is wenselijk dat een gebruiker het gebruik van je code kan begrijpen door alleen naar de typesignatuur en documentatie te kijken.

Je kunt documentatie aan functies toevoegen met opmerkingen. Deze worden dan in Go Doc weergegeven, net zoals je de documentatie van de standaardbibliotheek bekijkt.

```go
// Add takes two integers and returns the sum of them.
func Add(x, y int) int {
	return x + y
}
```

### Testbare voorbeelden

Wanneer je echt de extra stap wilt zetten, kun je [Testable Examples](https://blog.golang.org/examples) toevoegen. Hiervan staan veel voorbeelden in de documentatie van de standard library.

Vaak raken codevoorbeelden die buiten de codebase te vinden zijn, zoals een readme-bestand, verouderd en onjuist in vergelijking met de daadwerkelijke code, omdat ze niet consequent worden gecontroleerd.

Voorbeeldfuncties worden gecompileerd wanneer tests worden uitgevoerd. Omdat dergelijke voorbeelden worden gevalideerd door de Go-compiler, kun je erop vertrouwen dat de voorbeelden in de documentatie altijd het huidige codegedrag weerspiegelen.

Voorbeeldfuncties beginnen met `Example` (vergelijkbaar met testfuncties die met `Test` beginnen), en staan opgelagen in de `_test.go` bestanden van de package. Voeg de volgende `ExampleAdd` functie toe aan het `adder_test.go` bestand.

```go
func ExampleAdd() {
	sum := Add(1, 5)
	fmt.Println(sum)
	// Output: 6
}
```

(Als je editor de pakketten niet automatisch voor je importeert, mislukt de compilatiestap omdat `import "fmt"` ontbreekt in `adder_test.go`. Het is ten zeerste aan te raden om uit te zoeken hoe je dit soort fouten automatisch kunt laten oplossen in de editor die je gebruikt.)

Door deze code toe te voegen, verschijnt het voorbeeld in je documentatie, waardoor je code nog toegankelijker wordt. Als je code ooit verandert, waardoor het voorbeeld niet meer geldig is, mislukt je build.

Wanneer we de testsuite van het pakket uitvoeren, zien we dat de voorbeeldfunctie `ExampleAdd` wordt uitgevoerd zonder verdere acties van onze kant:

```bash
$ go test -v
=== RUN   TestAdder
--- PASS: TestAdder (0.00s)
=== RUN   ExampleAdd
--- PASS: ExampleAdd (0.00s)
```

Let op de speciale opmaak van de opmerking, `// Output: 6`. Hoewel het voorbeeld altijd gecompileerd zal worden, betekent het toevoegen van deze opmerking dat het voorbeeld ook uitgevoerd zal worden. Verwijder de opmerking tijdelijk `// Output: 6`, voer vervolgens go test uit en je zult zien dat ExampleAdd niet langer wordt uitgevoerd.

Voorbeelden zonder uitvoercommentaar zijn handig voor het demonstreren van code die niet als unit test kan worden uitgevoerd, zoals code die toegang nodig heeft tot het netwerk, terwijl toch de garantie wordt geboden dat het voorbeeld op zijn minst compileert.

Om voorbeelddocumentatie te bekijken, bekijken we `pkgsite` even snel. Navigeer naar de map van je project en voer `pkgsite -open` uit. Dit opent een webbrowser die verwijst naar `http://localhost:8080`. Hierin zie je een lijst met alle pakketten van de standaardbibliotheek van Go, plus pakketten van derden die je hebt geïnstalleerd. Daaronder zou je de voorbeelddocumentatie voor `github.com/quii/learn-go-with-tests` moeten vinden. Volg die link en kijk vervolgens onder `Integers`, vervolgens onder `func Add` en vouw `Example` uit. Je zou nu het voorbeeld moeten zien dat je hebt toegevoegd voor `sum := Add(1, 5)`.

Als je je code met voorbeelden publiceert op een openbare URL, kun je de documentatie van jouw code delen op [pkg.go.dev](https://pkg.go.dev/). [Hier](https://pkg.go.dev/github.com/quii/learn-go-with-tests/integers/v2) is bijvoorbeeld de definitieve API voor dit hoofdstuk. Met deze web interface kun je zoeken naar documentatie van standaard bibliotheek pakketten en pakketten van derden.

## Samenvattend

Wat we hebben besproken:

* Meer geoefend met de TDD werkwijze
* Integers, optellen
* Het schrijven van betere documentatie, zodat gebruikers van onze code het gebruik ervan snel kunnen begrijpen
* Voorbeelden van hoe je de code kunt gebruiken, die word gecontroleerd als onderdeel van de tests
