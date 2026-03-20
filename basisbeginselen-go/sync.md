# Sync

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/sync)

Wij willen een teller maken die veilig is voor gelijktijdig gebruik.

We beginnen met een onveilige teller en controleren of het gedrag ervan werkt in een single-threaded omgeving.

Vervolgens gaan we de onveiligheid ervan testen door met meerdere goroutines de teller te gebruiken en het probleem op te lossen.

## Schrijf eerst je test

We willen dat onze API ons een methode geeft om de teller te verhogen en vervolgens de waarde ervan op te halen.

```go
func TestCounter(t *testing.T) {
	t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
		counter := Counter{}
		counter.Inc()
		counter.Inc()
		counter.Inc()

		if counter.Value() != 3 {
			t.Errorf("got %d, want %d", counter.Value(), 3)
		}
	})
}
```

## Probeer de test uit te voeren

```
./sync_test.go:9:14: undefined: Counter
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Laten we `Counter` definiëren.

```go
type Counter struct {
}
```

Probeer de test opnieuw en het zal falen met de volgende melding

```
./sync_test.go:14:10: counter.Inc undefined (type Counter has no field or method Inc)
./sync_test.go:18:13: counter.Value undefined (type Counter has no field or method Value)
```

Om de test uiteindelijk uit te voeren, kunnen we die methoden definiëren

```go
func (c *Counter) Inc() {

}

func (c *Counter) Value() int {
	return 0
}
```

De test zou nu uitgevoerd moeten worden, maar moeten falen

```
=== RUN   TestCounter
=== RUN   TestCounter/incrementing_the_counter_3_times_leaves_it_at_3
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/incrementing_the_counter_3_times_leaves_it_at_3 (0.00s)
    	sync_test.go:27: got 0, want 3
```

## Schrijf genoeg code om de test te laten slagen

Dit zou voor Go-experts zoals wij eenvoudig moeten zijn. We moeten een status voor de teller in ons datatype bijhouden en deze vervolgens bij elke `Inc`-aanroep verhogen.

```go
type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}

func (c *Counter) Value() int {
	return c.value
}
```

## Refactor

Er valt niet veel te refactoren, maar aangezien we meer tests rondom `Counter` gaan schrijven, schrijven we een kleine assert-functie `assertCount`, zodat de test wat duidelijker leesbaar is.

```go
t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
	counter := Counter{}
	counter.Inc()
	counter.Inc()
	counter.Inc()

	assertCounter(t, counter, 3)
})
```

```go
func assertCounter(t testing.TB, got Counter, want int) {
	t.Helper()
	if got.Value() != want {
		t.Errorf("got %d, want %d", got.Value(), want)
	}
}
```

## Vervolg stapen

Dat was eenvoudig genoeg, maar nu hebben we de eis dat het veilig moet zijn voor gebruik in een gelijktijdige omgeving. We zullen een falende test moeten schrijven om dit te testen.

## Write the test first

```go
t.Run("it runs safely concurrently", func(t *testing.T) {
	wantedCount := 1000
	counter := Counter{}

	var wg sync.WaitGroup
	wg.Add(wantedCount)

	for i := 0; i < wantedCount; i++ {
		go func() {
			counter.Inc()
			wg.Done()
		}()
	}
	wg.Wait()

	assertCounter(t, counter, wantedCount)
})
```

Deze test zal door onze `wantedCount` heen lopen en een goroutine activeren om `counter.Inc()` aan te roepen.

We gebruiken [`sync.WaitGroup`](https://golang.org/pkg/sync/#WaitGroup), een handige manier om gelijktijdige processen te synchroniseren.

> Een WaitGroup wacht tot een verzameling goroutines klaar is. De hoofdgoroutine roept Add aan om het aantal te wachten goroutines in te stellen. Vervolgens wordt elke goroutine uitgevoerd en roept Done aan wanneer deze klaar is. Tegelijkertijd kan Wait worden gebruikt om te blokkeren totdat alle goroutines klaar zijn.

Door te wachten tot `wg.Wait()` klaar is voordat we onze beweringen doen, weten we zeker dat al onze goroutines hebben geprobeerd de teller te verhogen.

## Probeer de test uit te voeren

```
=== RUN   TestCounter/it_runs_safely_in_a_concurrent_envionment
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/it_runs_safely_in_a_concurrent_envionment (0.00s)
    	sync_test.go:26: got 939, want 1000
FAIL
```

De test zal _waarschijnlijk_ mislukken met een ander getal, maar het laat wel zien dat de test niet werkt als meerdere goroutines tegelijkertijd proberen de waarde van de teller te veranderen.

## Schrijf genoeg code om de test te laten slagen

Een eenvoudige oplossing is om een ​​slot aan onze `Counter` toe te voegen, zodat slechts één goroutine tegelijk de teller kan verhogen. Go's [`Mutex`](https://golang.org/pkg/sync/#Mutex) biedt zo'n slot:

> Een Mutex is een wederzijdse uitsluitingsvergrendeling. De waarde nul voor een Mutex is een ontgrendelde mutex.

```go
type Counter struct {
	mu    sync.Mutex
	value int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}
```

Dit betekent dat elke goroutine die `Inc` aanroept, de lock op `Counter` krijgt als deze als eerste is. Alle andere goroutines moeten wachten tot de `Unlock` uitgevoerd is voordat ze toegang krijgen.

Als je de test nu opnieuw uitvoert, zou deze moeten slagen, omdat elke goroutine op zijn beurt moet wachten voordat er een wijziging kan worden doorgevoerd.

## Ik heb andere voorbeelden gezien waarbij `sync.Mutex` in de struct is ingebed.

Je ziet wellicht voorbeeldig als

```go
type Counter struct {
	sync.Mutex
	value int
}
```

Je zou kunnen stellen dat het de code wat eleganter zou kunnen maken.

```go
func (c *Counter) Inc() {
	c.Lock()
	defer c.Unlock()
	c.value++
}
```

Dit _ziet_ er leuk uit, maar hoewel programmeren een enorm subjectief vakgebied is, **is dit slecht en fout**.

Soms vergeten mensen dat het insluiten van typen betekent dat de methoden van dat type onderdeel worden van de publieke interface; en dat wil je vaak niet. Onthoud dat we heel voorzichtig moeten zijn met onze publieke API's; het moment dat we iets openbaar maken, is het moment waarop andere code eraan kan koppelen. We willen onnodige koppeling altijd vermijden.

Het blootstellen van `Lock` en `Unlock` is in het beste geval verwarrend, maar kan in het slechtste geval zeer schadelijk zijn voor de software als aanroepen van je type deze methoden beginnen aan te roepen.

![Showing how a user of this API can wrongly change the state of the lock](https://i.imgur.com/SWYNpwm.png)

_Het lijkt echt een heel slecht idee te zijn_

## Mutexes kopiëren

Onze test is geslaagd, maar onze code is nog steeds een beetje gevaarlijk.

Als je `go vet` op je code uitvoert, zou je een foutmelding moeten krijgen zoals de volgende

```
sync/v2/sync_test.go:16: call of assertCounter copies lock value: v1.Counter contains sync.Mutex
sync/v2/sync_test.go:39: assertCounter passes lock by value: v1.Counter contains sync.Mutex
```

Een blik op de documentatie van [`sync.Mutex`](https://golang.org/pkg/sync/#Mutex) vertelt ons waarom

> Een Mutex mag na het eerste gebruik niet meer worden gekopieerd.

Wanneer we onze `Counter` (op waarde) doorgeven aan `assertCounter`, zal deze proberen een kopie van de mutex te maken.

Om dit op te lossen, moeten we in plaats daarvan een pointer naar onze `Counter` doorgeven, dus de parameters van `assertCounter` wijzigen

```go
func assertCounter(t testing.TB, got *Counter, want int)
```

Onze tests compileren niet meer omdat we een `Counter` proberen mee te geven in plaats van een `*Counter`. Om dit op te lossen, maak ik liever een constructor die lezers van je API laat zien dat het beter is om het type niet zelf te initialiseren.

```go
func NewCounter() *Counter {
	return &Counter{}
}
```

Gebruik deze functie wanneer in je tests wanneer je `Counter` initialiseert. Let op dat je de initatie aanpast van `counter := Counter{}` naar `counter := NewCounter()`. Je gebruikt nu dus hookjes in plaats van accolades omdat je via een functie initieert in plaats van via een type.&#x20;

## Samenvattend

We hebben een paar onderdelen besproken uit het [sync package](https://golang.org/pkg/sync/)

* `Mutex` stelt je in staat om locks op je data te zetten
* `WaitGroup` is een manier om te wachten tot goroutines hun taken hebben afgerond

### Wanneer gebruik je locks op kanalen en goroutines?

[We hebben goroutines eerder behandeld in het eerste hoofdstuk over concurrency](concurrency.md), waarmee we veilige concurrent code konden schrijven. Waarom zou je dan locks gebruiken? [De go-wiki heeft een pagina gewijd aan dit onderwerp: Mutex of Channel.](https://go.dev/wiki/MutexOrChannel)

> Een veelvoorkomende fout van beginners in Go is het overmatig gebruiken van kanalen en goroutines, alleen maar omdat het mogelijk is en/of omdat het leuk is. Wees niet bang om een ​​`sync.Mutex` te gebruiken als dat het beste bij je probleem past. Go is pragmatisch: je kunt de tools gebruiken die je probleem het beste oplossen en dwingt je niet tot één codestijl.

Dus:

* **Gebruik kanalen bij het doorgeven van eigendom van gegevens**
* **Gebruik mutexes voor het beheren van states**

### go vet

Vergeet niet om go vet te gebruiken in je buildscripts. Hiermee wordt je gewaarschuwd voor subtiele bugs in je code voordat ze je arme gebruikers treffen.

### Gebruik geen embedding omdat het handig is

* Denk eens na over het effect dat insluiten heeft op je openbare API.
* Wilt je deze methoden _echt_ openbaar maken en mensen hun eigen code eraan laten koppelen?
* Met betrekking tot mutexen zou dit op zeer onvoorspelbare en vreemde manieren rampzalig kunnen zijn. Stel je voor dat een kwaadaardige code een mutex ontgrendelt terwijl dat niet de bedoeling is; dit zou een aantal zeer vreemde bugs veroorzaken die moeilijk op te sporen zijn.
