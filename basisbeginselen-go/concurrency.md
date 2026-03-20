# Concurrency

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/concurrency)

Dit is de opzet: een collega heeft een functie geschreven, `CheckWebsites`, die de status van een lijst met URL's controleert.

<pre class="language-go"><code class="lang-go">package concurrency
<strong>
</strong>type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		results[url] = wc(url)
	}

	return results
}
</code></pre>

Hiermee wordt een map van elke gecontroleerde URL's geretourneerd met een Booleaanse waarde: `true` voor een goed response, `false` voor een slecht response.

Je moet ook een `WebsiteChecker` opgeven die één URL accepteert en een boolean retourneert. Deze wordt door de functie gebruikt om alle websites te controleren.

Door [dependency injection](dependency-injection.md) te gebruiken, konden ze de functie testen zonder echte HTTP-aanroepen te doen, waardoor deze betrouwbaar en snel is geworden.

Hier is de test die ze hebben geschreven:

```go
package concurrency

import (
	"reflect"
	"testing"
)

func mockWebsiteChecker(url string) bool {
	return url != "waat://furhurterwe.geds"
}

func TestCheckWebsites(t *testing.T) {
	websites := []string{
		"http://google.com",
		"http://blog.gypsydave5.com",
		"waat://furhurterwe.geds",
	}

	want := map[string]bool{
		"http://google.com":          true,
		"http://blog.gypsydave5.com": true,
		"waat://furhurterwe.geds":    false,
	}

	got := CheckWebsites(mockWebsiteChecker, websites)

	if !reflect.DeepEqual(want, got) {
		t.Fatalf("wanted %v, got %v", want, got)
	}
}
```

De functie is in productie en wordt gebruikt om honderden websites te controleren. Maar je collega krijgt klachten dat de functie traag is, dus hebben ze je gevraagd om te helpen de snelheid te verbeteren.

## Schrijf eerst je test

Laten we een benchmark gebruiken om de snelheid van `CheckWebsites` te testen, zodat we het effect van onze wijzigingen kunnen zien.

```go
package concurrency

import (
	"testing"
	"time"
)

func slowStubWebsiteChecker(_ string) bool {
	time.Sleep(20 * time.Millisecond)
	return true
}

func BenchmarkCheckWebsites(b *testing.B) {
	urls := make([]string, 100)
	for i := 0; i < len(urls); i++ {
		urls[i] = "a url"
	}

	for b.Loop() {
		CheckWebsites(slowStubWebsiteChecker, urls)
	}
}
```

De benchmark test `CheckWebsites` met een slice van honderd URL's en gebruikt een nieuwe nep-implementatie van `WebsiteChecker`. `slowStubWebsiteChecker` is opzettelijk traag. Het gebruikt `time.Sleep` om precies twintig milliseconden te wachten en retourneert vervolgens true.

Wanneer we de benchmark uitvoeren met go `test -bench=.` (of als je Windows PowerShell gebruikt, `go test -bench="."`) zie je dit resultaat:

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v0
BenchmarkCheckWebsites-4               1        2249228637 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v0        2.268s
```

`CheckWebsites` is getest op 2249228637 nanoseconden, wat ongeveer twee en een kwart seconde is.

Laten we proberen dit sneller te maken.

### Schrijf genoeg code om de test te laten slagen

Nu kunnen we het eindelijk over _concurrency_ hebben, wat in het vervolg betekent dat er "meer dan één proces tegelijk is". Dit is iets wat we van nature elke dag doen.

Zo zette ik vanochtend bijvoorbeeld een kop thee. Ik zette de waterkoker aan en terwijl ik wachtte tot het water kookte, pakte ik de melk uit de koelkast, de thee uit de kast, pakte mijn favoriete mok, deed het theezakje in de kop en toen de waterkoker kookte, deed ik er water in.

Wat ik _niet deed_, was de waterkoker aanzetten en er vervolgens wezenloos naar staren totdat het water kookte, en daarna alle het anders doen zodra het water kookte.

Als je begrijpt waarom het sneller is om thee op de eerste manier te zetten, dan begrijp je ook hoe we `CheckWebsites` sneller gaan maken. In plaats van te wachten tot een website reageert voordat we een verzoek naar de volgende website sturen, zullen we onze computer opdracht geven om het volgende verzoek te doen terwijl hij wacht.

Normaal gesproken wachten we in Go, wanneer we een functie `doSomething()` aanroepen, tot deze retourneert (zelfs als er geen waarde is om te retourneren, wachten we nog steeds tot de functie is voltooid). We noemen deze bewerking _blocking_ - het laat ons wachten tot de bewerking is voltooid. Een bewerking die in Go niet blokkeert, wordt uitgevoerd in een apart proces, _een goroutine genaamd_. Stel je een proces voor als het van boven naar beneden lezen van de pagina met Go-code, waarbij elke functie die wordt aangeroepen 'binnenin' wordt gelezen wat deze doet. Wanneer een apart proces start, is het alsof een andere lezer binnenin de functie begint te lezen, terwijl de oorspronkelijke lezer de pagina verder kan aflopen.

Om Go te vertellen een nieuwe goroutine te starten, zetten we een functieaanroep om in een `go`-instructie door het sleutelwoord go ervoor te zetten: `go doSomething()`.

```go
package concurrency

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	return results
}
```

Omdat de enige manier om een ​​goroutine te starten is door go voor een functieaanroep te plaatsen, gebruiken we vaak _anonieme functies_ wanneer we een goroutine willen starten. Een anonieme functieliteral ziet er precies hetzelfde uit als een normale functiedeclaratie, maar dan zonder naam (niet verrassend). Je ziet er hierboven een in de body van de `for`-lus.

Anonieme functies hebben een aantal nuttige eigenschappen, waarvan we er hierboven twee hebben gebruikt. Ten eerste kunnen ze worden uitgevoerd op hetzelfde moment dat ze worden gedeclareerd - dit is wat de `()` aan het einde van de anonieme functie doet. Ten tweede behouden ze toegang tot de lexicale scope waarin ze zijn gedefinieerd - alle variabelen die beschikbaar zijn op het moment dat je de anonieme functie declareert, zijn ook beschikbaar in de body van de functie.

De body van de anonieme functie hierboven is precies hetzelfde als de body van de lus. Het enige verschil is dat elke iteratie van de lus een nieuwe goroutine start, gelijktijdig met het huidige proces (de `WebsiteChecker`-functie). Elke goroutine voegt zijn resultaat toe aan de resultaten map.

Maar als we `go test` uitvoeren:

```sh
--- FAIL: TestCheckWebsites (0.00s)
        CheckWebsites_test.go:31: Wanted map[http://google.com:true http://blog.gypsydave5.com:true waat://furhurterwe.geds:false], got map[]
FAIL
exit status 1
FAIL    github.com/gypsydave5/learn-go-with-tests/concurrency/v1        0.010s

```

### Een korte opmerking over het concurrency universum...

Het kan zijn dat je dit resultaat niet krijgt. Je krijgt mogelijk een paniekmelding waar we het zo meteen over hebben. Maak je geen zorgen als je die krijgt, blijf de test gewoon uitvoeren totdat je het bovenstaande resultaat krijgt. Of doe alsof je dat wel krijgt. De keuze is aan jou. Welkom bij concurrency: als het niet correct wordt afgehandeld, is het moeilijk te voorspellen wat er gaat gebeuren. Maak je geen zorgen - daarom schrijven we tests, zodat we weten wanneer we concurrency voorspelbaar afhandelen.

### ... en we zijn weer terug.

We zijn gepakt door de originele test `CheckWebsites`, die nu een lege map retourneert. Wat is er misgegaan?

Geen van de goroutines die onze `for`-lus startte, had genoeg tijd om hun resultaat toe te voegen aan de `resultaten`. De `WebsiteChecker`-functie is te snel voor hen en retourneert een nog steeds lege map.

Om dit op te lossen, kunnen we gewoon wachten tot alle goroutines hun werk doen en dan terugkeren. Twee seconden zou voldoende moeten zijn, toch?

```go
package concurrency

import "time"

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	time.Sleep(2 * time.Second)

	return results
}
```

Als je nu geluk hebt krijg je:

```sh
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v1        2.012s
```

Maar als je pech hebt (dit is waarschijnlijker als je ze uitvoert met de benchmark, aangezien je dan meer pogingen krijgt):

```sh
fatal error: concurrent map writes

goroutine 8 [running]:
runtime.throw(0x12c5895, 0x15)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/panic.go:605 +0x95 fp=0xc420037700 sp=0xc4200376e0 pc=0x102d395
runtime.mapassign_faststr(0x1271d80, 0xc42007acf0, 0x12c6634, 0x17, 0x0)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:783 +0x4f5 fp=0xc420037780 sp=0xc420037700 pc=0x100eb65
github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1(0xc42007acf0, 0x12d3938, 0x12c6634, 0x17)
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x71 fp=0xc4200377c0 sp=0xc420037780 pc=0x12308f1
runtime.goexit()
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/asm_amd64.s:2337 +0x1 fp=0xc4200377c8 sp=0xc4200377c0 pc=0x105cf01
created by github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xa1

        ... many more scary lines of text ...
```

Dit is lang en eng, maar het enige wat we hoeven te doen is even ademhalen en de stacktrace lezen: `fatal error: concurrent map writes`. Soms, wanneer we onze tests uitvoeren, schrijven twee goroutines tegelijkertijd naar de resultatenmap. Maps in Go vinden het niet prettig als meer dan één ding er tegelijk naar probeert te schrijven, wat resulteert in een `fatal error`.

Dit is een _raceconditie_, een bug die optreedt wanneer de uitvoer van onze software afhankelijk is van de timing en volgorde van gebeurtenissen waarover we geen controle hebben. Omdat we niet precies kunnen bepalen wanneer elke goroutine naar de resultatenmap schrijft, zijn we kwetsbaar voor twee goroutines die er tegelijkertijd naartoe schrijven.

Go kan ons helpen raceomstandigheden te detecteren met de ingebouwde [racedetector](https://blog.golang.org/race-detector). Om deze functie in te schakelen, voer je de tests uit met de `race`vlag: `go test -race`.

Je zou een uitvoer moeten krijgen die er ongeveer zo uitziet:

```sh
==================
WARNING: DATA RACE
Write at 0x00c420084d20 by goroutine 8:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Previous write at 0x00c420084d20 by goroutine 7:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Goroutine 8 (running) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c

Goroutine 7 (finished) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c
==================
```

De details zijn wederom moeilijk te lezen, maar `WARNING: DATA RACE` is vrij duidelijk. Als we de tekst van de fout lezen, zien we twee verschillende goroutines die schrijfbewerkingen op een map uitvoeren:

`Write at 0x00c420084d20 by goroutine 8:`

schrijft naar hetzelfde geheugenblok als

`Previous write at 0x00c420084d20 by goroutine 7:`

Daarboven kunnen we de regel code zien waar de schrijfbewerking plaatsvindt:

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12`

en de regel code waar goroutines 7 en 8 worden gestart:

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11`

Alles wat je moet weten wordt op je terminal afgedrukt. Je hoeft alleen maar geduldig genoeg te zijn om de meldingen te lezen.

### Kanalen

We kunnen deze datarace oplossen door onze goroutines te coördineren met behulp van kanalen. Kanalen vormen een Go-datastructuur die waarden kan ontvangen en verzenden. Deze bewerkingen, samen met hun details, maken communicatie tussen verschillende processen mogelijk.

In dit geval willen we nadenken over de communicatie tussen het bovenliggende proces en elk van de goroutines die het aanmaakt om de `WebsiteChecker`-functie met de url uit te voeren.

```go
package concurrency

type WebsiteChecker func(string) bool
type result struct {
	string
	bool
}

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)
	resultChannel := make(chan result)

	for _, url := range urls {
		go func() {
			resultChannel <- result{url, wc(url)}
		}()
	}

	for i := 0; i < len(urls); i++ {
		r := <-resultChannel
		results[r.string] = r.bool
	}

	return results
}
```

Naast de `results` map hebben we nu een `resultChannel`, die we op dezelfde manier maken. `chan result` is het type van het kanaal - een channel van `result`. Het nieuwe type `result` is gemaakt om de retourwaarde van de `WebsiteChecker` te koppelen aan de URL die wordt gecontroleerd - het is een struct van `string` en `bool`. Omdat we geen van beide waarden hoeven te benoemen, is elke waarde anoniem binnen de struct; dit kan handig zijn wanneer het moeilijk is om te weten hoe een waarde moet worden genoemd.

Wanneer we nu over de URL's itereren, sturen we in plaats van rechtstreeks naar de map te schrijven, een resultaatstructuur voor elke aanroep van `wc` naar het `resultChannel` met een _send-statement_. Hiervoor gebruiken we de `<-` operator, waarbij een kanaal aan de linkerkant en een waarde aan de rechterkant worden geschreven:

```go
// Send statement
resultChannel <- result{u, wc(u)}
```

De volgende `for`-lus itereert één keer voor elk van de URL's. Binnenin gebruiken we een ontvangstexpressie, die een waarde die van een kanaal is ontvangen, toewijst aan een variabele. Dit gebruikt ook de `<-` operator, maar nu met de twee operanden omgedraaid: het kanaal staat nu rechts en de variabele waaraan we toewijzen staat links:

```go
// Receive expression
r := <-resultChannel
```

Vervolgens gebruiken we het ontvangen resultaat om de map bij te werken.

Door de resultaten naar een kanaal te sturen, kunnen we de timing van elke schrijfbewerking in de resultatenmap bepalen, zodat deze één voor één plaatsvindt. Hoewel elke aanroep van `wc` en elke verzending naar het resultatenkanaal gelijktijdig binnen een eigen proces plaatsvindt, worden de resultaten één voor één verwerkt terwijl we waarden uit het resultatenkanaal halen met de ontvangstexpressie.

We hebben gelijktijdigheid gebruikt voor het deel van de code dat we sneller wilden maken, terwijl we ervoor zorgden dat het deel dat niet gelijktijdig kon gebeuren, toch lineair gebeurde. En we hebben de communicatie tussen de verschillende betrokken processen verzorgd via kanalen.

Wanneer we de benchmark uitvoeren:

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v2
BenchmarkCheckWebsites-8             100          23406615 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v2        2.377s
```

23406615 nanoseconden - 0,023 seconden, ongeveer honderd keer zo snel als de oorspronkelijke functie. Een groot succes.

## Samenvattend

Deze oefening was iets minder intensief wat betreft de TDD dan normaal. We hebben in zekere zin deelgenomen aan één lange refactoring van de `CheckWebsites`-functie; de ​​invoer en uitvoer zijn nooit veranderd, het is alleen sneller geworden. Maar de tests die we hadden, evenals de benchmark die we schreven, stelden ons in staat `CheckWebsites` zo te refactoren dat we er zeker van konden zijn dat de software nog steeds werkte, terwijl we tegelijkertijd aantoonden dat deze daadwerkelijk sneller was geworden.

Door deze functionaliteit sneller te maken leerde we over:

* _goroutines_, de basiseenheid voor concurrency in Go, waarmee we meer dan één websitecontroleverzoek kunnen beheren.
* _anonieme functies_, die we gebruikten om elk van de gelijktijdige processen te starten die websites controleren.
* _kanalen_, om de communicatie tussen de verschillende processen te helpen organiseren en controleren, zodat we een _race condition_ bug kunnen voorkomen.
* de _racedetector_ die ons hielp problemen met gelijktijdige code te debuggen

### Maak het snel

Eén formulering van een agile manier van software bouwen, die vaak ten onrechte aan Kent Beck wordt toegeschreven, is \*:

> [Make it work, make it right, make it fast](http://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast)

Waar 'werk' het slagen van de tests inhoudt, 'goed' het refactoren van de code is, en 'snel' het optimaliseren van de code om deze bijvoorbeeld snel te laten werken. We kunnen het pas 'snel maken' als we het werkend en goed hebben gemaakt. We hadden geluk dat de code die we kregen al bewezen werkte en niet gerefactored hoefde te worden. We zouden nooit moeten proberen het 'snel te maken' voordat de andere twee stappen zijn uitgevoerd, omdat \*:

> [Premature optimization is the root of all evil](http://wiki.c2.com/?PrematureOptimization) -- Donald Knuth

\*) Deze teksten zijn zo iconisch in de software development gemeenschap, dat ik ze bewust in het Engels heb laten staan.&#x20;
