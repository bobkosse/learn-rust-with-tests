# Context-aware Reader

[**Je kunt alle code uit dit hoofdstuk hier vinden**](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/context-aware-reader)

In dit hoofdstuk wordt gedemonstreerd hoe je een contextbewuste `io.Reader` kunt testen, zoals geschreven door Mat Ryer en David Hernandez in [The Pace Dev Blog](https://pace.dev/blog/2020/02/03/context-aware-ioreader-for-golang-by-mat-ryer).

## Contextbewuste lezer?

Allereerst een korte introductie tot `io.Reader`.

Als je andere hoofdstukken in dit boek hebt gelezen, ben je `io.Reader` vast wel eens tegengekomen bij het openen van bestanden, het coderen van JSON en diverse andere veelvoorkomende taken. Het is een simpele abstractie van het lezen van data uit _iets_

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

Door `io.Reader` te gebruiken, kun je veel hergebruik uit de standaardbibliotheek halen; het is een veelgebruikte abstractie (samen met zijn tegenhanger `io.Writer`).

### Contextbewust?

[In een eerder hoofdstuk](../basisbeginselen-go/context.md) hebben we besproken hoe we `context` kunnen gebruiken om te annuleren. Dit is vooral handig als je taken uitvoert die rekenintensief kunnen zijn en je ze wilt kunnen stoppen.

Als je een `io.Reader` gebruikt, heb je geen garanties over de snelheid; het kan 1 nanoseconde of honderden uren duren. Het kan handig zijn om dit soort taken in je eigen applicatie te kunnen annuleren, en dat is waar Mat en David over schreven.

Ze combineerden twee eenvoudige abstracties (`context.Context` en `io.Reader`) om dit probleem op te lossen.

Laten we proberen wat functionaliteit te TDD'en, zodat we een `io.Reader` zo kunnen inpakken dat deze geannuleerd kan worden.

Het testen hiervan vormt een interessante uitdaging. Normaal gesproken geef je bij het gebruik van een `io.Reader` deze meestal door aan een andere functie en houd je je niet echt bezig met de details, zoals `json.NewDecoder` of `io.ReadAll`.

Wat we willen demonstreren is zoiets als:

> Gegeven een `io.Reader` met "ABCDEF", wanneer ik halverwege een annulatiesignaal stuur, wanneer ik probeer verder te lezen, krijg ik niets anders, dus krijg ik alleen "ABC".

Laten we de interface nog eens bekijken.


```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

De `Read`-methode van de `Reader` leest de inhoud ervan in een `[]byte` die we opgeven.

In plaats van alles te lezen, kunnen we dus:

* Een byte-array met vaste grootte opgeven die niet alle inhoud bevat
* Een annulatiesignaal verzenden
* Probeer het opnieuw te lezen. Dit zou een foutmelding moeten opleveren met 0 gelezen bytes

Laten we nu gewoon een "happy path"-test schrijven zonder annulering, zodat we vertrouwd kunnen raken met het probleem zonder dat we al productiecode hoeven te schrijven.

```go
func TestContextAwareReader(t *testing.T) {
	t.Run("lets just see how a normal reader works", func(t *testing.T) {
		rdr := strings.NewReader("123456")
		got := make([]byte, 3)
		_, err := rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "123")

		_, err = rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "456")
	})
}

func assertBufferHas(t testing.TB, buf []byte, want string) {
	t.Helper()
	got := string(buf)
	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

* Maak een `io.Reader` van een string met wat data.
* Een byte-array om in te lezen die kleiner is dan de inhoud van de reader.
* Roep read aan, controleer de inhoud en herhaal.

Hieruit kunnen we ons voorstellen dat er vóór de tweede leesbewerking een soort annulatiesignaal wordt verzonden om het gedrag te veranderen.

Nu we hebben gezien hoe het werkt, zullen we de rest van de functionaliteit TDD'en.

## Schrijf eerst de test

We willen een `io.Reader` kunnen samenstellen met een `context.Context`.

Met TDD is het het beste om te beginnen met het bedenken van de gewenste API en er een test voor te schrijven.

Laat de compiler en de mislukte testuitvoer ons vervolgens naar een oplossing leiden

```go
t.Run("behaves like a normal reader", func(t *testing.T) {
	rdr := NewCancellableReader(strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	_, err = rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "456")
})
```

## Probeer de test uit te voeren

```
./cancel_readers_test.go:12:10: undefined: NewCancellableReader
```

## Schrijf de minimale hoeveelheid code voor de test om uit te voeren en controleer de uitvoer van de mislukte test.

We moeten deze functie definiëren en deze moet een `io.Reader` retourneren.

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return nil
}
```

Wanneer je dit probeert uit te voeren

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/behaves_like_a_normal_reader
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x10f8fb5]
```

Zoals verwacht

## Schrijf voldoende code om het te laten slagen

Voorlopig retourneren we gewoon de `io.Reader` die we doorgeven

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return rdr
}
```

De test zou nu moeten slagen.

Ik weet het, ik weet het, dit klinkt misschien gek en pedant, maar voordat we aan de slag gaan, is het belangrijk dat we _enige_ verificatie hebben dat we het "normale" gedrag van een `io.Reader` niet hebben verstoord. Deze test geeft ons vertrouwen voor de toekomst.

## Schrijf eerst de test

Vervolgens moeten we proberen te annuleren.

```go
t.Run("stops reading when cancelled", func(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	rdr := NewCancellableReader(ctx, strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	cancel()

	n, err := rdr.Read(got)

	if err == nil {
		t.Error("expected an error after cancellation but didn't get one")
	}

	if n > 0 {
		t.Errorf("expected 0 bytes to be read after cancellation but %d were read", n)
	}
})
```

We kunnen de eerste test min of meer kopiëren, maar nu:

* We maken een `context.Context` met annulering, zodat we na de eerste keer lezen kunnen `cancel`.
* Om onze code te laten werken, moeten we `ctx` aan onze functie doorgeven.
* Vervolgens bevestigen we dat er na `cancel` niets is gelezen.

## Probeer de test uit te voeren

```
./cancel_readers_test.go:33:30: too many arguments in call to NewCancellableReader
	have (context.Context, *strings.Reader)
	want (io.Reader)
```

## Schrijf de minimale hoeveelheid code voor de test om uit te voeren en controleer de mislukte testuitvoer.

De compiler vertelt ons wat we moeten doen: onze handtekening bijwerken om een context te accepteren.

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return rdr
}
```

(Je moet de eerste test ook bijwerken om te slagen in `context.Background`)

Je zou nu een zeer duidelijke uitvoer van de mislukte test moeten zien

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/stops_reading_when_cancelled
--- FAIL: TestCancelReaders (0.00s)
    --- FAIL: TestCancelReaders/stops_reading_when_cancelled (0.00s)
        cancel_readers_test.go:48: expected an error but didn't get one
        cancel_readers_test.go:52: expected 0 bytes to be read after cancellation but 3 were read
```

## Schrijf voldoende code om het te laten slagen

Op dit punt is het kopiëren en plakken van het originele bericht van Mat en David, maar we gaan het rustig en iteratief aanpakken.

We weten dat we een type nodig hebben dat de `io.Reader` waaruit we lezen en de `context.Context` omvat, dus laten we dat aanmaken en proberen het vanuit onze functie te retourneren in plaats van de originele `io.Reader`

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return &readerCtx{
		ctx:      ctx,
		delegate: rdr,
	}
}

type readerCtx struct {
	ctx      context.Context
	delegate io.Reader
}
```

Zoals ik al vaker in dit boek heb benadrukt: ga langzaam te werk en laat de compiler je helpen

```
./cancel_readers_test.go:60:3: cannot use &readerCtx literal (type *readerCtx) as type io.Reader in return argument:
	*readerCtx does not implement io.Reader (missing Read method)
```

De abstractie voelt goed, maar implementeert niet de interface die we nodig hebben (`io.Reader`). Daarom voegen we de methode toe.

```go
func (r *readerCtx) Read(p []byte) (n int, err error) {
	panic("implement me")
}
```

Voer de tests uit en ze zouden moeten _compileren_, maar paniek. Dit is nog steeds een stap in de goede richting.

Laten we de eerste test laten slagen door de aanroep te _delegeren_ naar onze onderliggende `io.Reader`.

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	return r.delegate.Read(p)
}
```

Op dit punt is onze 'happy path'-test opnieuw geslaagd en het voelt alsof we alles goed hebben geabstraheerd.

Om onze tweede test te laten slagen, moeten we de `context.Context` controleren om te zien of deze is geannuleerd.

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	if err := r.ctx.Err(); err != nil {
		return 0, err
	}
	return r.delegate.Read(p)
}
```

Alle tests zouden nu moeten slagen. Je zult zien hoe we de fout van `context.Context` retourneren. Dit stelt aanroepers van de code in staat om de verschillende redenen voor de annulering te onderzoeken. Dit wordt uitgebreider behandeld in het oorspronkelijke bericht.

## Samenvattend

* Kleine interfaces zijn goed en zijn eenvoudig samen te stellen.
* Wanneer je één ding (bijv. `io.Reader`) met een ander probeert uit te breiden, kun je meestal het [delegatiepatroon](https://en.wikipedia.org/wiki/Delegation_pattern) gebruiken.

> In software engineering is het delegatiepatroon een objectgeoriënteerd ontwerppatroon waarmee objectcompositie hetzelfde codehergebruik als overerving mogelijk maakt.

* Een eenvoudige manier om met dit soort werk te beginnen, is door je gedelegeerde te omhullen en een test te schrijven die beweert dat deze zich gedraagt ​​zoals de gedelegeerde zich normaal gedraagt, voordat je andere onderdelen gaat samenstellen om het gedrag te wijzigen. Dit helpt je om ervoor te zorgen dat alles correct blijft werken terwijl je codeert naar je doel.
