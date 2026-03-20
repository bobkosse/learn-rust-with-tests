# Mocking

[Je kunt hier alle code van dit hoofdstuk vinden](https://github.com/quii/learn-go-with-tests/tree/main/mocking)

Je wordt gevraagd een programma te schrijven dat aftelt vanaf 3, waarbij elk getal op een nieuwe regel wordt afgedrukt (met een pauze van 1 seconde) en dat, zodra het programma nul bereikt, "Go!" afdrukt en afsluit.

```
3
2
1
Go!
```

We gaan dit aanpakken door een functie te schrijven met de naam `Countdown`. Deze functie plaatsen we vervolgens in een hoofdprogramma (`main`), zodat het er ongeveer zo uitziet:

```go
package main

func main() {
	Countdown()
}
```

Hoewel dit een vrij eenvoudig programma is, moeten we, om het volledig te testen, zoals altijd een _iteratieve, testgedreven aanpak_ hanteren.

Wat bedoel ik met iteratief? We zorgen ervoor dat we de kleinst mogelijke stappen zetten om _bruikbare_ software te schrijven.

We willen niet veel tijd besteden aan code die theoretisch gezien werkt na wat hacken, want dat is vaak de manier waarop ontwikkelaars in het diepe springen. **Het is een belangrijke vaardigheid om eisen zo klein mogelijk te kunnen opdelen, zodat je werkende software krijgt.**

Zo kunnen we ons werk opdelen en er iteraties op uitvoeren:

* Print 3
* Print 3, 2, 1 en Go!
* Wacht een seconde tussen iedere regel

## Schrijf eerst je test

Onze software moet naar stdout afdrukken en we hebben bekeken hoe we Dependency Injection (DI) kunnen gebruiken om dit in de DI-sectie te testen.

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}

	Countdown(buffer)

	got := buffer.String()
	want := "3"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Als je niet bekend bent met het begrip `buffer`, lees dan [het vorige gedeelte](dependency-injection.md) nog eens door.

We weten dat we willen dat onze `Countdown`-functie ergens gegevens naartoe schrijft en `io.Writer` is de de facto manier om die gegevens vast te leggen als een interface in Go.



* In `main` sturen we het bericht naar `os.Stdout` waardoor onze gebruikers het aftellen in de terminal kunnen zien.
* In de test sturen we dit naar `bytes.Buffer` waardoor onze teste de gegenerereerde data kunnen afvangen.

## Probeer de test uit te voeren

`./countdown_test.go:11:2: undefined: Countdown`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Defineer de functiecount `Countdown`&#x20;

```go
func Countdown() {}
```

Voer de test opnieuw uit:

```
./countdown_test.go:11:11: too many arguments in call to Countdown
    have (*bytes.Buffer)
    want ()
```

De compiler vertelt je welke argumenten je aan de functie mee moet geven. Pas je code aan:

```go
func Countdown(out *bytes.Buffer) {}
```

`countdown_test.go:17: got '' want '3'`

Perfect!

## Schrijf genoeg code om de test te laten slagen

```go
func Countdown(out *bytes.Buffer) {
	fmt.Fprint(out, "3")
}
```

We gebruiken `fmt.Fprint`, dat een `io.Writer` (zoals `*bytes.Buffer`) gebruikt en er een `string` naartoe stuurt. De test zou moeten slagen.

## Refactor

We weten dat `*bytes.Buffer` wel werkt, maar dat het beter is om een ​​algemene interface te gebruiken.

```go
func Countdown(out io.Writer) {
	fmt.Fprint(out, "3")
}
```

Voer de tests opnieuw uit. Ze zouden moeten slagen.

Om het geheel af te ronden, gaan we onze functie nu verbinden met een `main`-functie, zodat we werkende software hebben om onszelf ervan te verzekeren dat we vooruitgang boeken.

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func Countdown(out io.Writer) {
	fmt.Fprint(out, "3")
}

func main() {
	Countdown(os.Stdout)
}
```

Probeer het programma eens uit en wees verbaasd over je werk.

Ja, dit lijkt nogal basic, maar deze aanpak zou ik voor elk project aanraden. **Neem een ​​klein stukje functionaliteit en zorg dat het van begin tot eind werkt, ondersteund door tests.**

Vervolgens kunnen we 2,1 en dan "Go!" laten printen.

## Schrijf eerst je test

Door te investeren in het goed laten werken van de algehele installatie, kunnen we onze oplossing veilig en eenvoudig itereren. We hoeven het programma niet langer te stoppen en opnieuw te draaien om er zeker van te zijn dat het werkt, omdat alle logica wordt getest.

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}

	Countdown(buffer)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

De backtick-syntaxis is een andere manier om een ​​`string` te maken, maar hiermee kun je zaken als nieuwe regels gebruiken, wat perfect is voor onze test.

## Probeer de test uit te voeren

```
countdown_test.go:21: got '3' want '3
        2
        1
        Go!'
```

## Schrijf genoeg code om de test te laten slagen

```go
func Countdown(out io.Writer) {
	for i := 3; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, "Go!")
}
```

Gebruik een `for`-lus die terugtelt met `i--` en gebruik `fmt.Fprintln` om naar `out` te printen met ons nummer gevolgd door een nieuwe regel. Gebruik ten slotte `fmt.Fprint` om "Go!" naar de volgende regel te sturen.

## Refactor

Er valt niet veel te refactoren, behalve het omzetten van een aantal magische waarden in benoemde constanten.

```go
const finalWord = "Go!"
const countdownStart = 3

func Countdown(out io.Writer) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, finalWord)
}
```

Als je het programma nu uitvoert, zou je het gewenste resultaat moeten krijgen, maar het gaat niet om een ​​dramatische aftelling met pauzes van 1 seconde.

Met Go kun je dit bereiken met `Time.Sleep`. Probeer het eens toe te voegen aan onze code.

```go
func Countdown(out io.Writer) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, finalWord)
}
```

Als je het programma uitvoert, werkt het zoals wij dat willen.

## Mocking

De tests zijn nog steeds succesvol en de software werkt zoals bedoeld, maar er zijn enkele problemen:

* De tests duren 3 seconden om uit te voeren.
  * Elk vooruitstrevend bericht over softwareontwikkeling benadrukt het belang van snelle feedbackloops.
  * **Langzame testen verpesten de productiviteit van ontwikkelaars**
  * Stel je voor dat de eisen geavanceerder worden en er meer tests nodig zijn. Zijn we dan nog steeds tevreden met de toevoeging van 3 seconden aan de testrun voor elke nieuwe test van `Countdown`?
* We hebben een belangrijke eigenschap van onze functie niet getest.

We hebben een afhankelijkheid van `Sleep`ing die we moeten extraheren, zodat we deze vervolgens in onze tests kunnen controleren.

Als we `time.Sleep` kunnen _mocken_, kunnen we _dependency injection_ gebruiken om deze tijd te gebruiken in plaats van de 'echte' `time.Sleep`. Vervolgens kunnen we de **aanroepen bespioneren** om er beweringen over te doen.

## Schrijf eerst je test

Laten we onze afhankelijkheid definiëren als een interface. Dit stelt ons in staat om een ​​echte Sleeper in `main` en een _spy sleeper_ in onze tests te gebruiken. Door een interface te gebruiken, heeft onze `Countdown`-functie hier geen weet van en voegt wat flexibiliteit toe voor de aanroeper.

```go
type Sleeper interface {
	Sleep()
}
```

Ik heb een ontwerpbeslissing genomen waarbij onze `Countdown`-functie niet verantwoordelijk is voor hoe lang de pauze duurt. Dit vereenvoudigt onze code in ieder geval voorlopig een beetje en betekent dat een gebruiker van onze functie die pauze naar wens kan configureren.

Nu moeten we er een simulatie van maken voor onze tests.

```go
type SpySleeper struct {
	Calls int
}

func (s *SpySleeper) Sleep() {
	s.Calls++
}
```

_Spies_ zijn een soort _mocks_ die kunnen registreren hoe een afhankelijkheid wordt gebruikt. Ze kunnen de meegestuurde argumenten registreren, hoe vaak deze is aangeroepen, enzovoort. In ons geval houden we bij hoe vaak `Sleep()` wordt aangeroepen, zodat we dit in onze test kunnen controleren.

Werk de tests bij om een ​​afhankelijkheid van onze Spy te injecteren en te bevestigen dat de sleep 3 keer is aangeroepen.

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}
	spySleeper := &SpySleeper{}

	Countdown(buffer, spySleeper)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}

	if spySleeper.Calls != 3 {
		t.Errorf("not enough calls to sleeper, want 3 got %d", spySleeper.Calls)
	}
}
```

## Probeer de test uit te voeren

```
too many arguments in call to Countdown
    have (*bytes.Buffer, *SpySleeper)
    want (io.Writer)
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

We moeten `Countdown` updaten om onze `Sleeper` te accepteren

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, finalWord)
}
```

Als je het opnieuw probeert, zal je hoofdbestand om dezelfde reden niet meer compileren

```
./main.go:26:11: not enough arguments in call to Countdown
    have (*os.File)
    want (io.Writer, Sleeper)
```

Laten we een _echte slaper_ creëren die de interface implementeert die we nodig hebben

```go
type DefaultSleeper struct{}

func (d *DefaultSleeper) Sleep() {
	time.Sleep(1 * time.Second)
}
```

We kunnen deze vervolgens gebruiken in onze echte toepassing zoals hier

```go
func main() {
	sleeper := &DefaultSleeper{}
	Countdown(os.Stdout, sleeper)
}
```

## Schrijf genoeg code om de test te laten slagen

De test compileert nu, maar slaagt niet omdat we nog steeds de `time.Sleep`-afhankelijkheid aanroepen in plaats van de geïnjecteerde afhankelijkheid. Laten we dat oplossen.

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		sleeper.Sleep()
	}

	fmt.Fprint(out, finalWord)
}
```

De test zou moeten slagen en niet geen 3 seconden meer mogen duren.

### Nog steeds enkele problemen

Er is nog een belangrijke eigenschap die we nog niet hebben getest.

`Countdown` moet pauzeren voor elke volgende afdruk, bijvoorbeeld:

* `Print N`
* `Sleep`
* `Print N-1`
* `Sleep`
* `Print Go!`
* etc

Onze laatste wijziging bevestigt alleen dat het proces 3 keer gepauzeert heeft, maar die pauzes kunnen ook in een andere volgorde plaatsvinden.

Als je bij het schrijven van tests niet zeker weet of je tests je voldoende zekerheid geven, maak het dan gewoon stuk! (Zorg er wel voor dat je je wijzigingen eerst hebt vastgelegd in de broncode). Wijzig de code als volgt:

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		sleeper.Sleep()
	}

	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
	}

	fmt.Fprint(out, finalWord)
}
```

Als je je tests uitvoert, zouden ze nog steeds moeten slagen, ook al is de implementatie verkeerd.

Laten we spying opnieuw gebruiken met een nieuwe test om te controleren of de volgorde van de bewerkingen correct is.

We hebben twee verschillende afhankelijkheden en we willen al hun bewerkingen in één lijst vastleggen. Daarom maken we voor beide één spion.

```go
type SpyCountdownOperations struct {
	Calls []string
}

func (s *SpyCountdownOperations) Sleep() {
	s.Calls = append(s.Calls, sleep)
}

func (s *SpyCountdownOperations) Write(p []byte) (n int, err error) {
	s.Calls = append(s.Calls, write)
	return
}

const write = "write"
const sleep = "sleep"
```

Onze `SpyCountdownOperations` implementeert zowel `io.Writer` als `Sleeper` en registreert elke aanroep in één slice. In deze test kijken we alleen naar de volgorde van de bewerkingen, dus is het voldoende om ze op te nemen als een lijst met benoemde bewerkingen.

We kunnen nu een subtest toevoegen aan onze testsuite die verifieert dat onze slaap- en afdruktaken in de volgorde werken die we hopen

```go
t.Run("sleep before every print", func(t *testing.T) {
	spySleepPrinter := &SpyCountdownOperations{}
	Countdown(spySleepPrinter, spySleepPrinter)

	want := []string{
		write,
		sleep,
		write,
		sleep,
		write,
		sleep,
		write,
	}

	if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
		t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
	}
})
```

Deze test zou nu moeten mislukken. Draai `Countdown` terug naar de oorspronkelijke staat om de test te herstellen.

We hebben nu twee tests die de `Sleeper` bespioneren, dus we kunnen onze test nu refactoren. De ene test test wat er wordt geprint en de andere zorgt ervoor dat we tussen de prints door slapen. Ten slotte kunnen we onze eerste spion verwijderen, omdat deze niet meer wordt gebruikt.

```go
func TestCountdown(t *testing.T) {

	t.Run("prints 3 to Go!", func(t *testing.T) {
		buffer := &bytes.Buffer{}
		Countdown(buffer, &SpyCountdownOperations{})

		got := buffer.String()
		want := `3
2
1
Go!`

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})

	t.Run("sleep before every print", func(t *testing.T) {
		spySleepPrinter := &SpyCountdownOperations{}
		Countdown(spySleepPrinter, spySleepPrinter)

		want := []string{
			write,
			sleep,
			write,
			sleep,
			write,
			sleep,
			write,
		}

		if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
			t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
		}
	})
}
```

We hebben nu onze functie en de 2 belangrijke eigenschappen ervan goed getest.

## Sleeper uitbreiden om configureerbaar te zijn

Een mooie feature zou zijn dat de `Sleeper` configureerbaar zou zijn. Dit betekent dat we de slaaptijd in ons hoofdprogramma kunnen aanpassen.

### Schrijf eerst je test

Laten we eerst een nieuw type voor `ConfigurableSleeper` maken dat accepteert wat we nodig hebben voor configuratie en testen.

```go
type ConfigurableSleeper struct {
	duration time.Duration
	sleep    func(time.Duration)
}
```

We gebruiken `duration` om de pauze tijd te configureren en `sleep` als een manier om deze in een slaapfunctie door te geven. De signatuur van `sleep` is hetzelfde als die van `time.Sleep`, waardoor we `time.Sleep` in onze echte implementatie en de volgende spion in onze tests kunnen gebruiken:

```go
type SpyTime struct {
	durationSlept time.Duration
}

func (s *SpyTime) SetDurationSlept(duration time.Duration) {
	s.durationSlept = duration
}
```

Nu we onze spion op zijn plek hebben, kunnen we een nieuwe test voor de configureerbare slaper maken.

```go
func TestConfigurableSleeper(t *testing.T) {
	sleepTime := 5 * time.Second

	spyTime := &SpyTime{}
	sleeper := ConfigurableSleeper{sleepTime, spyTime.SetDurationSlept}
	sleeper.Sleep()

	if spyTime.durationSlept != sleepTime {
		t.Errorf("should have slept for %v but slept for %v", sleepTime, spyTime.durationSlept)
	}
}
```

Er zal niets nieuws zijn in deze test en de opzet ervan lijkt erg op de vorige mock testen.

### Probeer de test uit te voeren

```
sleeper.Sleep undefined (type ConfigurableSleeper has no field or method Sleep, but does have sleep)

```

Je zou een duidelijke foutmelding moeten zien die aangeeft dat er geen `Sleep`methode is aangemaakt in onze `ConfigurableSleeper`.

### Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func (c *ConfigurableSleeper) Sleep() {
}
```

Nu onze nieuwe `Sleep`functie is geïmplementeerd, hebben we een test die faalt.

```
countdown_test.go:56: should have slept for 5s but slept for 0s
```

### Schrijf genoeg code om de test te laten slagen

Het enige wat we nu nog hoeven te doen is de `Sleep`functie voor `ConfigurableSleeper` implementeren.

```go
func (c *ConfigurableSleeper) Sleep() {
	c.sleep(c.duration)
}
```

Met deze wijziging zouden alle tests weer moeten slagen en je vraagt ​​je misschien af ​​waarom al die moeite, aangezien het hoofdprogramma helemaal niet is veranderd. Hopelijk wordt het duidelijk na het volgende gedeelte.

### Opschonen en refactor

Het laatste wat we moeten doen is onze `ConfigurableSleeper` gebruiken in de hoofdfunctie.

```go
func main() {
	sleeper := &ConfigurableSleeper{1 * time.Second, time.Sleep}
	Countdown(os.Stdout, sleeper)
}
```

Als we de tests en het programma handmatig uitvoeren, zien we dat het gedrag hetzelfde blijft.

Omdat we de `ConfigurableSleeper` gebruiken, is het nu veilig om de `DefaultSleeper`-implementatie te verwijderen. We ronden ons programma af en hebben een meer [generieke](https://stackoverflow.com/questions/19291776/whats-the-difference-between-abstraction-and-generalization) Sleeper met willekeurige lange aftellingen.

## Maar is mocken niet slecht?

Je hebt misschien gehoord dat mocken slecht is. Net als alles in softwareontwikkeling kan het voor slechte doeleinden worden gebruikt, net als concepten als [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself).

Mensen raken vaak in een slechte staat als ze _niet naar hun tests luisteren_ en _de refactoringfase niet respecteren_.

Als je mocking code ingewikkeld wordt of als je veel dingen moet mocken om iets te testen, moet je naar dat slechte gevoel luisteren en nadenken over je code. Meestal is het een teken van

* teveel ineens moeten testen (omdat er te veel afhankelijkheden zijn om te simuleren).
  * Breek je module dan op zodat deze minder omvangrijk is
* De afhankelijkheden zijn te fijnmazig
  * Denk na over hoe je een aantal van deze afhankelijkheden kunt samenvoegen in één zinvolle module
* Je test is te veel gericht op implementatiedetails
  * Geef de voorkeur aan het testen van verwacht gedrag in plaats van de implementatie ervan.

Normaal gesproken wijst veel mocken op een slechte abstractie in je code.

**Wat vaak gezien wordt als een zwakte van TDD, maar het is eigenlijk juist krachtig.** Slechte testcode is vaak het resultaat van slecht ontwerp. Beter gezegd: _goed ontworpen code is eenvoudig te testen_.

### Maar de mocks en testen maken mijn leven nog steeds moeilijk!

Herken je deze situatie?

* Je wilt wat refactoring doorvoeren
* Om dit te doen moet je veel testen aanpassen
* Je stelt TDD ter discussie en plaatst een bericht op Medium met de titel "Mocken wordt als schadelijk beschouwd"

Dit is meestal een teken dat je te veel _implementatiedetails_ test. Probeer ervoor te zorgen dat je tests nuttig gedrag testen, tenzij de implementatie echt belangrijk is voor de werking van het systeem.

Soms is het lastig om te bepalen op welk niveau je precies moet testen, maar ik probeer wel een aantal denkprocessen en regels te volgen:

* **De definitie van refactoring is dat de code verandert, maar het gedrag hetzelfde blijft.** Als je in theorie hebt besloten om te refactoren, zou je de commit moeten kunnen maken zonder testwijzigingen. Dus stel jezelf bij het schrijven van een test de volgende vraag:
  * Test ik het gedrag dat ik wil, of de implementatiedetails?
  * Als ik deze code zou refactoren, moet ik dan veel wijzigingen aanbrengen in de tests?
* Hoewel je met Go private-functies kunt testen, zou ik dit vermijden, omdat private-functies implementatiedetails zijn ter ondersteuning van openbaar gedrag. Test het openbare gedrag. Sandi Metz beschrijft private-functies als "minder stabiel" en je wilt je tests er niet aan koppelen.
* Ik heb het gevoel dat als een test met **meer dan drie mocks werkt, dit een waarschuwingssignaal is**: tijd voor een heroverweging van het ontwerp.
* Gebruik spionnen met de nodige voorzichtigheid. Spionnen laten je de binnenkant van het algoritme dat je schrijft zien, wat erg nuttig kan zijn, maar het betekent wel een nauwere koppeling tussen je testcode en de implementatie. **Zorg ervoor dat je deze details echt belangrijk vindt als je ze gaat bespioneren.**

#### Kan ik niet gewoon een mocking-framework gebruiken?

Mocking vereist geen magie en is relatief eenvoudig; het gebruik van een framework kan mocking ingewikkelder laten lijken dan het is. We gebruiken in dit hoofdstuk geen automocking, dus krijgen we:

* een beter begrip van hoe je kunt mocken
* oefening met het implementeren van interfaces

In samenwerkingsprojecten is het waardevol om automatisch mocks te genereren. In een team zorgt een mockgeneratietool voor consistentie rond de testdubbels. Dit voorkomt inconsistent geschreven testdubbels, wat kan leiden tot inconsistent geschreven tests.

Gebruik alleen een mock generator die testdubbels genereert tegen een interface. Elke tool die overdreven dicteert hoe tests geschreven moeten worden, of die veel 'magie' gebruikt, kan je in de problemen brengen.

## Samenvattend

### Meer over de TDD aanpak

* Wanneer je met minder triviale voorbeelden te maken krijgt, deel het probleem dan op in _"dunne verticale plakjes"_. Probeer zo snel mogelijk een punt te bereiken waarop je _werkende software hebt die wordt ondersteund door tests_, om te voorkomen dat je in valkuilen terechtkomt en een "big bang"-aanpak kiest.
* Zodra je over werkende software beschikt, kun je makkelijker kleine stapjes maken totdat je de software hebt gevonden die je nodig hebt.

> "Wanneer gebruik je iteratieve ontwikkeling? Gebruik iteratieve ontwikkeling alleen voor projecten die je succesvol wilt maken."

Martin Fowler.

### Mocking

* **Zonder mocking blijven belangrijke delen van je code ongetest.** In ons geval zouden we niet kunnen testen of onze code pauzeert tussen elke afdruk, maar er zijn talloze andere voorbeelden. Een service aanroepen die kan falen? Je systeem in een bepaalde status willen testen? Het is erg moeilijk om deze scenario's te testen zonder mocking.
* Zonder mocks moet je mogelijk databases en andere externe zaken opzetten om eenvoudige bedrijfsregels te testen. Je krijgt dan waarschijnlijk langzame tests, wat resulteert in **langzame feedbackloops**.
* Als je een database of webservice moet opstarten om iets te testen, is de kans groot dat de **tests kwetsbaar** zijn vanwege de onbetrouwbaarheid van dergelijke services.

Zodra een ontwikkelaar eenmaal leert over mocking, wordt het heel gemakkelijk om elk facet van een systeem te overtesten op basis van de _manier waarop het werkt_ in plaats van _wat het doet_. Houd altijd rekening met de **waarde van je tests** en de impact die ze zouden kunnen hebben op toekomstige refactoring.

In dit hoofdstuk over mocken hebben we het alleen gehad over **Spies**, een soort mock. Mocks zijn een soort "testdubbel".

> [Test Double is een algemene term voor alle gevallen waarin je een productieobject vervangt voor testdoeleinden.](https://martinfowler.com/bliki/TestDouble.html)

Onder testdubbels vallen verschillende typen, zoals stubs, spies en zelfs mocks! Bekijk de post van [Martin Fowler](https://martinfowler.com/bliki/TestDouble.html) voor meer informatie.

## Bonus - Voorbeeld van iteratoren uit go 1.23

In Go 1.23 werden [iterators geïntroduceerd](https://tip.golang.org/doc/go1.23). We kunnen iterators op verschillende manieren gebruiken. In dit geval kunnen we een `countdownFrom`-iterator maken, die de aftelgetallen in omgekeerde volgorde retourneert.

Voordat we ingaan op hoe we aangepaste iteratoren schrijven, laten we eerst eens kijken hoe we ze gebruiken. In plaats van een vrij imperatieve lus te schrijven om af te tellen vanaf een bepaald getal, kunnen we deze code expressiever maken door een `range`-over te zetten met onze aangepaste `countdownFrom`-iterator.

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := range countDownFrom(3) {
		fmt.Fprintln(out, i)
		sleeper.Sleep()
	}

	fmt.Fprint(out, finalWord)
}
```

Om een ​​iterator zoals `countDownFrom` te schrijven, moet je een functie op een bepaalde manier schrijven. Uit de documentatie:

```
The “range” clause in a “for-range” loop now accepts iterator functions of the following types
    func(func() bool)
    func(func(K) bool)
    func(func(K, V) bool)
```

(De `K` en `V` staan ​​respectievelijk voor sleutel(Key)- en waarde(value)typen.)

In ons geval hebben we geen sleutels, alleen waarden. Go biedt ook een handig type `iter.Seq[T]`, een typealias voor `func(func(T) bool)`.

```go
func countDownFrom(from int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := from; i > 0; i-- {
			if !yield(i) {
				return
			}
		}
	}
}
```

Dit is een eenvoudige iterator die getallen in omgekeerde volgorde oplevert, beginnend bij, `from` - perfect voor ons gebruik.
