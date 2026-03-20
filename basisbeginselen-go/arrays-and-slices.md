# Arrays en slices

[**Je kunt hier alle code voor dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/arrays)

Met arrays kun je meerdere elementen van hetzelfde type in een variabele in een bepaalde volgorde opslaan.

Wanneer je arrays hebt, is het heel gebruikelijk om eroverheen te itereren. Laten we daarom onze [nieuwe kennis van `for`](iteration.md) gebruiken om een ​​`Sum`-functie te maken. `Sum` neemt een array met getallen en retourneert het totaal.

Natuurlijk gebruiken we onze TDD vaardigheden!

## Schrijf eerst je test

Maak een nieuwe map aan om in te werken. Maak daarin een nieuw bestand aan met de naam `sum_test.go` en zet daarin de volgende codesum.:

```go
package main

import "testing"

func TestSum(t *testing.T) {

	numbers := [5]int{1, 2, 3, 4, 5}

	got := Sum(numbers)
	want := 15

	if got != want {
		t.Errorf("got %d want %d given, %v", got, want, numbers)
	}
}
```

Arrays hebben _vaste capaciteit_ welke je definieert bij het declareren van de variabele. We kunnen een array op twee manieren initialiseren:

* \[N]type{value1, value2, ..., valueN} bijv. `numbers := [5]int{1, 2, 3, 4, 5}`
* \[...]type{value1, value2, ..., valueN} bijv. `numbers := [...]int{1, 2, 3, 4, 5}`

Soms is het nuttig om ook de invoer van de functie in de foutmelding weer te geven. Hier gebruiken we de tijdelijke aanduiding `%v` om de "standaard"-indeling weer te geven, wat goed werkt voor arrays.

[Lees meer over het formatteren van string waarden](https://golang.org/pkg/fmt/)

## Probeer de test uit te voeren

Als je go mod hebt geïnitialiseerd met `go mod init main`, krijg je de foutmelding `_testmain.go:13:2: cannot import "main"`. Dit komt doordat pakket main volgens de gangbare werkwijze alleen de integratie van andere pakketten bevat en geen unit-testbare code. Daarom staat Go het importeren van een pakket met de naam main niet toe.

Om dit probleem op te lossen, kun je de hoofdmodule in `go.mod` een andere naam geven.

Zodra bovenstaande fout is verholpen, zal de compiler bij het uitvoeren van `go test` de bekende `./sum_test.go:10:15: undefined: Sum`-fout laten zien. Nu kunnen we verdergaan met het schrijven van de daadwerkelijk te testen methode.

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

In `sum.go`

```go
package main

func Sum(numbers [5]int) int {
	return 0
}
```

Je test zou nu moeten mislukken met een _duidelijke foutmelding_

`sum_test.go:13: got 0 want 15 given, [1 2 3 4 5]`

## Schrijf genoeg code om de test te laten slagen

```go
func Sum(numbers [5]int) int {
	sum := 0
	for i := 0; i < 5; i++ {
		sum += numbers[i]
	}
	return sum
}
```

Om de waarde uit een array op een bepaalde index te halen, gebruik je de syntaxis `array[index]`. In dit geval gebruiken we `for` om vijf keer te itereren om de array te doorlopen en elk item aan `sum` toe te voegen.

## Refactor

Een mooi moment om [`range`](https://gobyexample.com/range) te introduceren om de code op te schonen

```go
func Sum(numbers [5]int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

`range` laat je over een array ittereren. Bij iedereen iteratie geeft  `range` twee resultaten terug: de index en de waarde. We kiezen er hier voor om de index waarde te negeren door gebruik te maken van `_`, diut wordt een [blank identifier](https://golang.org/doc/effective_go.html#blank) genoemd.

### Arrays en hun type

Een interessante eigenschap van arrays is dat de grootte in het type is gecodeerd. Als je een `[4]int` probeert door te geven aan een functie die een `[5]int` verwacht, zal de code niet compileren. Het zijn verschillende typen, dus het is precies hetzelfde als een `string` proberen door te geven aan een functie die een `int` wil ontvangen.

Misschien vind je het nogal omslachtig dat arrays een vaste lengte hebben, en in de meeste gevallen zul je ze daarom waarschijnlijk niet gebruiken!

Go heeft _slices_ die de grootte van de verzameling niet coderen, maar in plaats daarvan elke gewenste grootte kunnen hebben.

De volgende opdracht om te bouwen is het optellen van verzamelingen van verschillende lengtes.

## Schrijf eerst je test

We gebruiken nu het [slice-type](https://golang.org/doc/effective_go.html#slices), waarmee we verzamelingen van elke grootte kunnen hebben. De syntaxis is vergelijkbaar met die van arrays; je laat alleen de grootte weg bij het declareren.

`mySlice := []int{1,2,3}` in plaats van `myArray := [3]int{1,2,3}`

```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := [5]int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

## Probeer de test uit te voeren

Deze zal niet compileren

`./sum_test.go:22:13: cannot use numbers (type []int) as type [5]int in argument to Sum`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Het probleem hier is dat we ofwel

* We breken de bestaande API door het argument `Sum` te wijzigen naar een slice in plaats van een array. Als we dit doen, verpesten we mogelijk iemands dag, omdat onze andere test niet meer compileert!
* Een nieuwe functie maken die hetzelfde doet

In ons geval gebruikt niemand anders onze functie. In plaats van twee functies te onderhouden, houden we er maar één.

```go
func Sum(numbers []int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

Als je de tests probeert uit te voeren, worden ze nog steeds niet gecompileerd. Je moet ook de eerste test wijzigen, zodat deze een slice bevat in plaats van een array.

## Schrijf genoeg code om de test te laten slagen

Het blijkt dat het oplossen van de compilerproblemen het enige was wat we hoeven te doen, om de tests de laten slagen!

## Refactor

We hebben `Sum` al gerefactored. We hebben alleen arrays vervangen door slices, dus er zijn geen extra wijzigingen nodig. Vergeet niet dat we onze testcode niet mogen verwaarlozen tijdens de refactoringfase. Ook de code voor de tests willen we zo goed en efficiënt mogelijk maken. We kunnen onze `Sum`-tests verder verbeteren.

```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

Het is belangrijk om de waarde van je tests in twijfel te trekken. Het doel moet niet zijn om zoveel mogelijk tests te hebben, maar om zoveel mogelijk vertrouwen in je codebase te hebben. Te veel tests kunnen een echt probleem vormen en zorgen alleen maar voor meer overhead in onderhoud. **Elke test brengt kosten met zich mee.**

In ons geval zie je dat het overbodig is om twee tests voor deze functie te hebben. Als het werkt voor een slice van één grootte, is de kans groot dat het ook werkt voor een slice van elke grootte (binnen redelijke grenzen).

De ingebouwde testtoolkit van Go bevat een [coverage tool](https://blog.golang.org/cover). Hoewel het streven naar 100% dekking niet je einddoel zou moeten zijn, kan de coverage tool je helpen bij het identificeren van delen van je code die niet door tests worden gedekt. ​​Als je strikt de TDD werkwijze hebt gevolgd, is de kans groot dat je toch bijna 100% dekking hebt.

Voer het onderstaande commando uit

`go test -cover`

You should see

```bash
PASS
coverage: 100.0% of statements
```

Verwijder nu een van de tests en controleer opnieuw de coverage.

Nu we tevreden zijn dat we een goed geteste functie hebben, is het belangrijk je code weer vast te leggen in versiebeheer, voordat je doorgaat naar de volgende uitdaging.

We hebben een nieuwe functie nodig met de naam `SumAll` die een wisselend aantal slices accepteert en een nieuwe slice retourneert met de totalen van elke doorgegeven slice.

Bijvoorbeeld:

`SumAll([]int{1,2}, []int{0,9})` moet `[]int{3, 9}`teruggeven

of

`SumAll([]int{1,1,1})` moet `[]int{3}` teruggeven

## Schrijf eerst je test

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if got != want {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## Voer de test uit

`./sum_test.go:23:9: undefined: SumAll`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

We moeten `SumAll` definiëren op basis van wat onze test wil.

Met Go kun je [variadic functies](https://gobyexample.com/variadic-functions) schrijven die een variabel aantal argumenten kunnen aannemen.

```go
func SumAll(numbersToSum ...[]int) []int {
	return nil
}
```

Dit is geldig, maar onze tests compileren nog steeds niet!

`./sum_test.go:26:9: invalid operation: got != want (slice can only be compared to nil)`

Go staat het gebruik van vergelijkingsoperatoren met slices niet toe. Je zou een functie kunnen schrijven om over elke `got`- en `want`-slice te itereren en hun waarden te controleren, maar voor het gemak kunnen we [reflect.DeepEqual](https://golang.org/pkg/reflect/#DeepEqual) gebruiken, wat handig is om te zien of twee variabelen hetzelfde zijn.

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

(zorg ervoor dat je `import reflect` bovenin je bestand toevoegd om toegang te krijgen tot `DeepEqual`)

Het is belangrijk om te weten dat `reflect.DeepEqual` niet "typesafe" is. De code compileert zelfs als je iets onbenulligs hebt gedaan. Om dit in de praktijk te zien, wijzig je de test tijdelijk in:

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := "bob"

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

Wat we hier hebben gedaan, is proberen een `slice` te vergelijken met een `string`. Dit slaat nergens op, maar de test compileert wel! Dus hoewel `reflect.DeepEqual` een handige manier is om slices (en andere dingen) te vergelijken, moet je voorzichtig zijn bij het gebruik ervan.

(Vanaf Go 1.21 is het [Slices](https://pkg.go.dev/slices#pkg-overview)-standaardpakket beschikbaar, met de [slices.Equal](https://pkg.go.dev/slices#Equal)-functie om een ​​eenvoudige, oppervlakkige vergelijking uit te voeren op slices, waarbij je je geen zorgen hoeft te maken over de typen zoals in het bovenstaande geval. Houd er rekening mee dat deze functie verwacht dat de elementen [vergelijkbaar](https://pkg.go.dev/slices#Equal) zijn. Deze functie kan dus niet worden toegepast op slices met niet-vergelijkbare elementen, zoals 2D-slices.)

Verander de test naar de originele staat en voer deze uit. Je zou een resultaat moeten zien dat er als volgt uitziet:

`sum_test.go:30: got [] want [3 9]`

## Schrijf genoeg code om de test te laten slagen

Wat we moeten doen is itereren over de `varargs`, de som berekenen met behulp van onze bestaande `Sum`-functie en deze vervolgens toevoegen aan de slice die we zullen retourneren

```go
func SumAll(numbersToSum ...[]int) []int {
	lengthOfNumbers := len(numbersToSum)
	sums := make([]int, lengthOfNumbers)

	for i, numbers := range numbersToSum {
		sums[i] = Sum(numbers)
	}

	return sums
}
```

Dat zijn veel nieuwe dingen om te leren!

Er is een nieuwe manier om een ​​slice te maken. Met `make` kun je een slice maken met een startcapaciteit van de lengte ( `len` ) van de `numbersToSum` die we moeten doorlopen. De lengte van een slice is het aantal elementen dat de slice bevat: `len(mySlice)`, terwijl de capaciteit het aantal elementen is dat het kan bevatten in de onderliggende array `cap(mySlice)`. Bijvoorbeeld: `make([]int, 0, 5)` maakt een slice met lengte 0 en capaciteit 5.

Je kunt slices indexeren als arrays met `mySlice[N]` om de waarde eruit te halen of er een nieuwe waarde aan toe te wijzen met `=`

De tests zouden nu moeten slagen.

## Refactor

Zoals vermeld, hebben slices een capaciteit. Als je een slice hebt met een capaciteit van 2 en je probeert `mySlice[10] = 1` uit te voeren, krijg je een _runtime_-fout.

Je kunt echter de `append`-functie gebruiken, die een segment en een nieuwe waarde als invoer neemt en vervolgens een nieuw segment retourneert met alle items erin. De code is te verbeteren naar:

```go
func SumAll(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		sums = append(sums, Sum(numbers))
	}

	return sums
}
```

In deze implementatie maken we ons minder zorgen over capaciteit. We beginnen met een lege slice, `sums`, en voegen daar het resultaat van `Sum` aan toe terwijl we de varargs doorlopen.

Onze volgende opdracht is om `SumAll` te wijzigen in `SumAllTails`, zodat het de totalen van de "tails" van elke slice berekent. De tail van een collectie bestaat uit alle items in de collectie, behalve de eerste (de "head").

## Schrijf je eerste test

```go
func TestSumAllTails(t *testing.T) {
	got := SumAllTails([]int{1, 2}, []int{0, 9})
	want := []int{2, 9}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## Voer de test uit

`./sum_test.go:26:9: undefined: SumAllTails`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

De opdracht was de bestaande functie te wijzigen. Hernoem daarom de functie `SumAll` naar `SumAllTails` en voer de test opnieuw uit.&#x20;

`sum_test.go:30: got [3 9] want [2 9]`&#x20;

**Let op:** Pas ook de aanroep van de `TestSumAll` functie aan!

## Schrijf genoeg code om de test te laten slagen

```go
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		tail := numbers[1:]
		sums = append(sums, Sum(tail))
	}

	return sums
}
```

Slices kunnen worden gesliced! De syntaxis is `slice[laag:hoog]`. Als je de waarde aan een van de zijkanten van de `:` weglaat, wordt alles aan die kant vastgelegd. In ons geval zeggen we "neem van 1 tot het einde" met `numbers[1:]`. Je kunt wat tijd besteden aan het schrijven van andere tests rond slices en experimenteren met de slice-operator om er meer vertrouwd mee te raken.

## Refactor

Deze keer valt er niet veel te refactoren.

Wat denk je dat er zou gebeuren als je een lege slice aan onze functie zou toevoegen? Wat is de "staart" van een lege slice? Wat gebeurt er als je Go opdracht geeft om alle elementen uit `myEmptySlice[1:]` vast te leggen?

## Schrijf eerst je test

```go
func TestSumAllTails(t *testing.T) {

	t.Run("make the sums of some slices", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

}
```

## Voer de test uit

```
panic: runtime error: slice bounds out of range [recovered]
    panic: runtime error: slice bounds out of range
```

O nee! Het is belangrijk om op te merken dat de test weliswaar is _gecompileerd_, maar dat er een _runtime-fout_ is opgetreden.

Fouten tijdens het compileren zijn onze vriend, omdat ze ons helpen werkende software te schrijven.\
Rundtime-fouten zijn onze vijanden, omdat ze onze gebruikers beïnvloeden.

## Schrijf genoeg code om de test te laten slagen

```go
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

## Refactor

Onze tests bevatten opnieuw herhaalde code rondom de beweringen. Die gaan we in een functie extraheren.

```go
func TestSumAllTails(t *testing.T) {

	checkSums := func(t testing.TB, got, want []int) {
		t.Helper()
		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	}

	t.Run("make the sums of tails of", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}
		checkSums(t, got, want)
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}
		checkSums(t, got, want)
	})

}
```

We hadden een nieuwe functie `checkSums` kunnen maken zoals we normaal doen, maar in dit geval laten we een nieuwe techniek zien: het toewijzen van een functie aan een variabele. Het ziet er misschien vreemd uit, maar het is niet anders dan het toewijzen van een variabele aan een `string` of een `int`. Functies zijn in feite ook waarden.

Het wordt hier niet getoond, maar deze techniek kan handig zijn wanneer je een functie wilt binden aan andere lokale variabelen in de "scope" (bijvoorbeeld tussen enkele `{}`). Hiermee kun je ook het oppervlak van je API verkleinen.

Door deze functie binnen de test te definiëren, kan deze niet door andere functies in dit pakket worden gebruikt. Het verbergen van variabelen en functies die niet geëxporteerd hoeven te worden, is een belangrijke ontwerpoverweging.

Een handig neveneffect hiervan is dat het een beetje typesafety aan onze code toevoegt. Als een ontwikkelaar per ongeluk een nieuwe test toevoegt met `checkSums(t, got, "dave")`, zal de compiler hem onmiddellijk stoppen.

```bash
$ go test
./sum_test.go:52:21: cannot use "dave" (type string) as type []int in argument to checkSums
```

## Samenvattend

We hebben het gehad over:

* Arrays
* Slices
  * De verschillende manieren om ze aan te maken
  * Hoe ze een _vaste capaciteit_ hebben, maar je kunt nieuwe slices maken van oude slices met behulp van  `append`
  * Hoe je slices kunt slicen!
* `len` om de lengte van een array of slice te krijgen
* Test coverage tool
* `reflect.DeepEqual` en waarom het nuttig is, maar de typeveiligheid van je code kan verminderen

We hebben slices en arrays met gehele getallen gebruikt, maar ze werken ook met elk ander type, inclusief arrays/slices zelf. Je kunt dus een variabele van `[][]string` declareren als dat nodig is.

[Bekijk de Go-blogpost ](https://blog.golang.org/go-slices-usage-and-internals)over slices voor een diepere uitleg over slices. Probeer meer tests te schrijven om te ontdekken wat je ervan hebt geleerd.

Een andere handige manier om met Go te experimenteren, naast het schrijven van tests, is de Go Playground. Je kunt er de meeste dingen uitproberen en je kunt je code gemakkelijk delen als je vragen hebt. [Ik heb een Go Playground gemaakt met een stukje erin, zodat je ermee kunt experimenteren.](https://play.golang.org/p/ICCWcRGIO68)

[Hier is een voorbeeld](https://play.golang.org/p/bTrRmYfNYCp) van het slicen van een array en hoe het wijzigen van de slice de originele array beïnvloedt; een "kopie" van de slice heeft echter geen invloed op de originele array. [Nog een voorbeeld ](https://play.golang.org/p/Poth8JS28sc)van waarom het een goed idee is om een ​​kopie van een slice te maken nadat je een zeer grote slice hebt gesliced.
