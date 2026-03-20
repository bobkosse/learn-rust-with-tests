# Iteratie

[**Je kunt hier alle code voor dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/for)

Om dingen herhaaldelijk te doen in Go, heb je `for` nodig. In Go zijn er geen `while`, `do` en `until` trefwoorden, je kunt alleen for gebruiken. En dat is maar goed ook!

Laten we een test schrijven voor een functie die een teken 5 keer herhaalt.

Er is nog niets nieuws, dus probeer het zelf te schrijven om te oefenen. Let op dat je een nieuwe map en package aanmaakt. Lees de [deze sectie](integers.md#schrijf-eerst-je-test) nog eens door om te terug te lezen hoe je dat doet.

## Schrijf eerst je test

```go
package iteration

import "testing"

func TestRepeat(t *testing.T) {
	repeated := Repeat("a")
	expected := "aaaaa"

	if repeated != expected {
		t.Errorf("expected %q but got %q", expected, repeated)
	}
}
```

## Probeer de test uit te voeren

`./repeat_test.go:6:14: undefined: Repeat`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

_Behoud de discipline!_ Je hoeft niets nieuws te weten om de op de juiste manier te laten falen.

Alles wat je op dit moment moet doen is&#x20;

Het enige dat je nu hoeft te doen, is de test te laten compileren, zodat je kunt controleren of de test goed is geschreven.

```go
package iteration

func Repeat(character string) string {
	return ""
}
```

Is het niet fijn om te weten dat je al genoeg Go-kennis hebt om tests te schrijven voor een aantal basisproblemen? Dit betekent dat je nu zoveel met de productiecode kunt spelen als je wilt en weet dat deze zich gedraagt zoals je hoopt.

`repeat_test.go:10: expected 'aaaaa' but got ''`

## Schrijf genoeg code om de test te laten slagen

De `for` syntaxis is zeer onopvallend en volgt die van de meeste C-achtige talen.

```go
func Repeat(character string) string {
	var repeated string
	for i := 0; i < 5; i++ {
		repeated = repeated + character
	}
	return repeated
}
```

In tegenstelling tot andere talen zoals C, Java of JavaScript staan er geen haakjes rond de drie componenten van de for-instructie en zijn de accolades `{ }` altijd vereist. Je vraagt je misschien af wat er in deze regel gebeurt,

```go
	var repeated string
```

omdat we tot nu toe `:=` hebben gebruikt om variabelen te declareren en te initialiseren. `:=` is echter gewoon een [afkorting voor beide stappen](https://gobyexample.com/variables). Hier declareren we alleen een `string` variabele. Vandaar de expliciete versie. We kunnen `var` ook gebruiken om functies te declareren, zoals we later zullen zien.

Voer de test uit en deze zou moeten slagen.

Aanvullende varianten van de for-lus worden [hier](https://gobyexample.com/for) beschreven.

## Refactor

Nu is het tijd om te refactoren en een andere constructie te introduceren: `+=` toewijzingsoperator.

```go
const repeatCount = 5

func Repeat(character string) string {
	var repeated string
	for i := 0; i < repeatCount; i++ {
		repeated += character
	}
	return repeated
}
```

`+=`, ook wel "_de Add AND assignment operator_" genoemd, voegt de rechter operand toe aan de linker operand en wijst het resultaat toe aan de linker operand. Het werkt ook met andere typen, zoals gehele getallen.

### Benchmarking

Het schrijven van [benchmarks](https://golang.org/pkg/testing/#hdr-Benchmarks) in Go is een andere geweldige functie van de taal en lijkt erg op het schrijven van tests.

```go
func BenchmarkRepeat(b *testing.B) {
	for b.Loop() {
		Repeat("a")
	}
}
```

Je ziet dat de code erg overeen komt met een test.

Met `testing.B` krijg je toegang tot de lusfunctie. `Loop()` retourneert true zolang de benchmark moet blijven draaien.

Wanneer de benchmarkcode wordt uitgevoerd, meet deze hoe lang het duurt. Nadat `Loop()` false retourneert, bevat `b.N` het totale aantal iteraties dat is uitgevoerd.

Het maakt voor jou niet uit hoe vaak de code wordt uitgevoerd. Het framework bepaalt zelf wat een 'goede' waarde is, zodat je nuttige resultaten krijgt.

Om de benchmarks uit te voeren, voer je `go test -bench=.` uit. (of als je Windows PowerShell gebruikt, voer je `go test -bench="."` uit.)

To run the benchmarks do `go test -bench=.` (or if you're in Windows Powershell `go test -bench="."`)

```
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           136 ns/op
PASS
```

Wat `136 ns/op` betekent, is dat onze functie gemiddeld 136 nanoseconden nodig heeft om te draaien (op mijn computer). Dat is best oké! Om dit te testen, draaide hij de functie 10000000 keer.

**Noot:** Benchmarks worden standaard sequentieel uitgevoerd.

Alleen de body van de lus wordt getimed; setup- en opschoon-code wordt automatisch uitgesloten van benchmark timing. Een typische benchmark is als volgt opgebouwd:

```go
func Benchmark(b *testing.B) {
	//... setup ...
	for b.Loop() {
		//... code to measure ...
	}
	//... cleanup ...
}
```

Strings in Go zijn niet-muteerbaar, wat betekent dat elke samenvoeging, zoals in onze `Repeat`-functie, geheugenkopieerbewerkingen vereist om de nieuwe string te kunnen verwerken. Dit heeft invloed op de prestaties, met name bij intensieve samenvoeging van strings.

De standaardbibliotheek biedt het type `strings.Builder` [stringsBuilder](https://pkg.go.dev/strings#Builder), dat kopiëren van geheugen minimaliseert. Het implementeert een `WriteString`-methode die we kunnen gebruiken om strings samen te voegen:

```go
const repeatCount = 5

func Repeat(character string) string {
	var repeated strings.Builder
	for i := 0; i < repeatCount; i++ {
		repeated.WriteString(character)
	}
	return repeated.String()
}
```

**Noot**: We moeten de `String` methode aanroepen om het laatste resultaat op te halen.

We kunnen `BenchmarkRepeat` gebruiken om te bevestigen dat `strings.Builder` de prestaties aanzienlijk verbetert. Voer `go test -bench=. -benchmem` uit:

```
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           25.70 ns/op           8 B/op           1 allocs/op
PASS
```

De `-benchmem` flag rapporteert informatie over het toekennen van geheugen:

* `B/op`: het aantal bytes dat is toegekend per iteratie
* `allocs/op`: het aantal geheugen toekenningen per iteratie

## Oefeningen

* Verander de test zodat bij de aanroep kan worden aangeven hoe vaak het teken wordt herhaald. Pas vervolgens de code aan zodat de test slaagt.
* Schrijf `ExampleRepeat` om de functie te documenteren.
* Bekijk de [strings](https://golang.org/pkg/strings) package. Zoek functies waarvan je denkt dat ze nuttig kunnen zijn en experimenteer ermee door tests te schrijven zoals we hier hebben gedaan. Tijd investeren in het leren van de standaardbibliotheek zal zich op den duur zeker terugbetalen.

## Samenvattend

* Meer TDD oefeningen
* De `for` loop geleerd&#x20;
* Geleerd hoe je benchmarks kunt schrijven
