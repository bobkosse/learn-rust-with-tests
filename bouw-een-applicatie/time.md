# Time

**[Je kunt alle code voor dit hoofdstuk hier vinden](https://github.com/quii/learn-go-with-tests/tree/main/time)**

De producteigenaar wil dat we de functionaliteit van onze opdrachtregelapplicatie uitbreiden door een groep mensen te helpen Texas Holdem Poker te spelen.

## Net genoeg informatie over poker

Je hoeft niet veel over poker te weten, alleen dat alle spelers op bepaalde tijdstippen geïnformeerd moeten worden over een gestaag oplopende "blinde" waarde.

Onze applicatie helpt bij het bijhouden wanneer de blinde inzet verhoogd moet worden en hoeveel deze moet bedragen.

- Bij aanvang wordt gevraagd hoeveel spelers er spelen. Dit bepaalt de tijd voordat de "blinde" inzet verhoogd wordt.
- Er is een basistijd van 5 minuten.
- Voor elke speler wordt 1 minuut toegevoegd.
- Bijvoorbeeld: 6 spelers is gelijk aan 11 minuten voor de blinde.
- Nadat de blinde inzet is verstreken, moet het spel de spelers waarschuwen voor het nieuwe bedrag van de blinde inzet. - De blinde inzet begint bij 100 chips, dan 200, 400, 600, 1000, 2000 en blijft verdubbelen tot het spel eindigt (onze eerdere functionaliteit van "Ruth wint" zou het spel nog steeds moeten voltooien).

## Herinnering aan de code

In het vorige hoofdstuk zijn we begonnen met de opdrachtregelapplicatie die al de opdracht `{name} wint` accepteert. Zo ziet de huidige `CLI`-code eruit, maar zorg ervoor dat je ook de andere code kent voordat je begint.

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
}

func NewCLI(store PlayerStore, in io.Reader) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
	}
}

func (cli *CLI) PlayPoker() {
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```

### `time.AfterFunc`

We willen ons programma zo kunnen inplannen dat de waarden van de blinde inzet met bepaalde tijdsduren worden afgedrukt, afhankelijk van het aantal spelers.

Om de reikwijdte van wat we moeten doen te beperken, laten we het aantal spelers even buiten beschouwing en gaan we ervan uit dat er 5 spelers zijn. We testen dan of _elke 10 minuten_ de nieuwe waarde van de blinde inzet wordt afgedrukt.

Zoals gebruikelijk is de standaardbibliotheek voorzien van [`func AfterFunc(d Duration, f func()) *Timer`](https://golang.org/pkg/time/#AfterFunc)

> `AfterFunc` wacht tot de tijdsduur is verstreken en roept vervolgens f aan in zijn eigen goroutine. Het retourneert een `Timer` die kan worden gebruikt om de aanroep te annuleren met behulp van de Stop-methode.

### [`time.Duration`](https://golang.org/pkg/time/#Duration)

> A Duration represents the elapsed time between two instants as an int64 nanosecond count.

The time library has a number of constants to let you multiply those nanoseconds so they're a bit more readable for the kind of scenarios we'll be doing

```
5 * time.Second
```

Wanneer we `PlayPoker` aanroepen, plannen we al onze blind alerts.

Het testen hiervan kan echter lastig zijn. We willen controleren of elke tijdsperiode is gepland met het juiste blindbedrag. Maar als je naar de handtekening van `time.AfterFunc` kijkt, zie je dat het tweede argument de functie is die wordt uitgevoerd. Je kunt geen functies vergelijken in Go, dus we kunnen niet testen welke functie is verzonden. We moeten dus een soort wrapper rond `time.AfterFunc` schrijven die de tijd om uit te voeren en de hoeveelheid om af te drukken opneemt, zodat we dat kunnen zien.

## Schrijf eerst de test

Voeg een nieuwe test toe aan onze suite

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	if len(blindAlerter.alerts) != 1 {
		t.Fatal("expected a blind alert to be scheduled")
	}
})
```

Je zult merken dat we een `SpyBlindAlerter` hebben gemaakt die we in onze `CLI` proberen te injecteren en vervolgens controleren of er een waarschuwing is gepland nadat we `PlayPoker` hebben aangeroepen.

(Onthoud dat we eerst het eenvoudigste scenario uitproberen en dat we dit vervolgens herhalen.)

Hier is de definitie van `SpyBlindAlerter`

```go
type SpyBlindAlerter struct {
	alerts []struct {
		scheduledAt time.Duration
		amount      int
	}
}

func (s *SpyBlindAlerter) ScheduleAlertAt(duration time.Duration, amount int) {
	s.alerts = append(s.alerts, struct {
		scheduledAt time.Duration
		amount      int
	}{duration, amount})
}

```


## Probeer de test uit te voeren

```
./CLI_test.go:32:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *strings.Reader, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader)
```

## Schrijf de minimale hoeveelheid code om de test uit te voeren en controleer de mislukte testuitvoer.

We hebben een nieuw argument toegevoegd en de compiler klaagt. Strikt genomen is de minimale hoeveelheid code bedoeld om `NewCLI` een `*SpyBlindAlerter` te laten accepteren, maar laten we een beetje vals spelen en de afhankelijkheid gewoon als interface definiëren.

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

En voeg het dan toe aan de constructor

```go
func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI
```

Je andere tests zullen nu mislukken omdat ze geen `BlindAlerter` hebben doorgegeven aan `NewCLI`.

Spioneren op BlindAlerter is niet relevant voor de andere tests, dus voeg in het testbestand toe

```go
var dummySpyAlerter = &SpyBlindAlerter{}
```

Gebruik dat vervolgens in de andere tests om de compilatieproblemen op te lossen. Door het als "dummy" te labelen, is het voor de lezer van de test duidelijk dat het niet belangrijk is.

[> Dummy-objecten worden doorgegeven, maar nooit daadwerkelijk gebruikt. Meestal worden ze alleen gebruikt om parameterlijsten te vullen.](https://martinfowler.com/articles/mocksArentStubs.html)

De tests zouden nu moeten compileren en onze nieuwe test mislukt.

```
=== RUN   TestCLI
=== RUN   TestCLI/it_schedules_printing_of_blind_values
--- FAIL: TestCLI (0.00s)
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
    	CLI_test.go:38: expected a blind alert to be scheduled
```

## Schrijf voldoende code om de test te laten slagen

We moeten `BlindAlerter` als veld toevoegen aan onze `CLI`, zodat we ernaar kunnen verwijzen in onze `PlayPoker`-methode.

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		alerter:     alerter,
	}
}
```

Om de test te laten slagen, kunnen we onze `BlindAlerter` aanroepen met wat we maar willen

```go
func (cli *CLI) PlayPoker() {
	cli.alerter.ScheduleAlertAt(5*time.Second, 100)
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

Vervolgens willen we controleren of het alle gewenste meldingen inplant voor 5 spelers.

## Schrijf eerst de test

```go
	t.Run("it schedules printing of blind values", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}
		blindAlerter := &SpyBlindAlerter{}

		cli := poker.NewCLI(playerStore, in, blindAlerter)
		cli.PlayPoker()

		cases := []struct {
			expectedScheduleTime time.Duration
			expectedAmount       int
		}{
			{0 * time.Second, 100},
			{10 * time.Minute, 200},
			{20 * time.Minute, 300},
			{30 * time.Minute, 400},
			{40 * time.Minute, 500},
			{50 * time.Minute, 600},
			{60 * time.Minute, 800},
			{70 * time.Minute, 1000},
			{80 * time.Minute, 2000},
			{90 * time.Minute, 4000},
			{100 * time.Minute, 8000},
		}

		for i, c := range cases {
			t.Run(fmt.Sprintf("%d scheduled for %v", c.expectedAmount, c.expectedScheduleTime), func(t *testing.T) {

				if len(blindAlerter.alerts) <= i {
					t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
				}

				alert := blindAlerter.alerts[i]

				amountGot := alert.amount
				if amountGot != c.expectedAmount {
					t.Errorf("got amount %d, want %d", amountGot, c.expectedAmount)
				}

				gotScheduledTime := alert.scheduledAt
				if gotScheduledTime != c.expectedScheduleTime {
					t.Errorf("got scheduled time of %v, want %v", gotScheduledTime, c.expectedScheduleTime)
				}
			})
		}
	})
```

De tabelgebaseerde test werkt hier prima en illustreert duidelijk onze vereisten. We doorlopen de tabel en controleren de `SpyBlindAlerter` om te zien of de waarschuwing met de juiste waarden is gepland.

## Probeer de test uit te voeren

Je zou veel fouten moeten zien die er zo uitzien.

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s (0.00s)
        	CLI_test.go:71: got scheduled time of 5s, want 0s
=== RUN   TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s (0.00s)
        	CLI_test.go:59: alert 1 was not scheduled [{5000000000 100}]
```

## Schrijf voldoende code om de test te laten slagen

```go
func (cli *CLI) PlayPoker() {

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

Het is niet veel ingewikkelder dan wat we al hadden. We itereren nu gewoon over een array van `blinds` en roepen de scheduler aan met een oplopende `blindTime`.

## Refactor

We kunnen onze geplande waarschuwingen in een methode inkapselen, zodat `PlayPoker` wat duidelijker leesbaar is.

```go
func (cli *CLI) PlayPoker() {
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts() {
	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}
}
```

Uiteindelijk lijken onze tests wat onhandig. We hebben twee anonieme structuren die hetzelfde vertegenwoordigen: een `ScheduledAlert`. Laten we dat refactoren naar een nieuw type en vervolgens wat hulpmiddelen maken om ze te vergelijken.

```go
type scheduledAlert struct {
	at     time.Duration
	amount int
}

func (s scheduledAlert) String() string {
	return fmt.Sprintf("%d chips at %v", s.amount, s.at)
}

type SpyBlindAlerter struct {
	alerts []scheduledAlert
}

func (s *SpyBlindAlerter) ScheduleAlertAt(at time.Duration, amount int) {
	s.alerts = append(s.alerts, scheduledAlert{at, amount})
}
```

We hebben een `String()`-methode aan ons type toegevoegd, zodat deze netjes wordt afgedrukt als de test mislukt.

We hebben onze test bijgewerkt om ons nieuwe type te gebruiken.

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{10 * time.Minute, 200},
		{20 * time.Minute, 300},
		{30 * time.Minute, 400},
		{40 * time.Minute, 500},
		{50 * time.Minute, 600},
		{60 * time.Minute, 800},
		{70 * time.Minute, 1000},
		{80 * time.Minute, 2000},
		{90 * time.Minute, 4000},
		{100 * time.Minute, 8000},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

Implementeer `assertScheduledAlert` zelf.

We hebben hier behoorlijk wat tijd besteed aan het schrijven van tests en zijn een beetje stout geweest door niet te integreren met onze applicatie. Laten we dat aanpakken voordat we meer eisen stellen.

Probeer de app te draaien, maar hij compileert niet en klaagt over onvoldoende argumenten voor `NewCLI`.

Laten we een implementatie van `BlindAlerter` maken die we in onze applicatie kunnen gebruiken.

Maak `blind_alerter.go` aan, verplaats onze `BlindAlerter`-interface en voeg de nieuwe dingen hieronder toe.

```go
package poker

import (
	"fmt"
	"os"
	"time"
)

type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

type BlindAlerterFunc func(duration time.Duration, amount int)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}

func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

Onthoud dat elk _type_ een interface kan implementeren, niet alleen `structs`. Als je een bibliotheek maakt die een interface met één gedefinieerde functie beschikbaar stelt, is het een gebruikelijke idioom om ook een `MyInterfaceFunc`-type beschikbaar te stellen.

Dit type zal een `func` zijn die ook je interface implementeert. Zo hebben gebruikers van je interface de mogelijkheid om je interface te implementeren met slechts één functie, in plaats van een leeg `struct`-type te hoeven aanmaken.

Vervolgens maken we de functie `StdOutAlerter` aan, die dezelfde handtekening heeft als de functie, en gebruiken we `time.AfterFunc` om te plannen dat deze naar `os.Stdout` wordt afgedrukt.

We werken `main` bij waar we `NewCLI` aanmaken om dit in actie te zien.

```go
poker.NewCLI(store, os.Stdin, poker.BlindAlerterFunc(poker.StdOutAlerter)).PlayPoker()
```

Voordat je de test uitvoert, kun je de `blindTime`-increment in de `CLI` wijzigen naar 10 seconden in plaats van 10 minuten, zodat je het in actie kunt zien.

Je zou de blinde waarden elke 10 seconden moeten zien afdrukken zoals we verwachten. Merk op dat je nog steeds `Shaun wint` in de CLI kunt typen en het programma stopt zoals we verwachten.

Het spel wordt niet altijd met 5 spelers gespeeld, dus we moeten de gebruiker vragen om een ​​aantal spelers in te voeren voordat het spel begint.

## Schrijf eerst de test

Om te controleren of we het aantal spelers vragen, willen we vastleggen wat er naar StdOut wordt geschreven. We hebben dit nu een paar keer gedaan en we weten dat `os.Stdout` een `io.Writer` is, dus we kunnen controleren wat er wordt geschreven als we dependency injection gebruiken om een ​​`bytes.Buffer` in onze test door te geven en zien wat onze code zal schrijven.

We maken ons nog geen zorgen over onze andere medewerkers in deze test, dus hebben we een paar dummy's in ons testbestand gemaakt.

We moeten er wel rekening mee houden dat we nu vier afhankelijkheden hebben voor `CLI`, wat misschien te veel verantwoordelijkheden met zich meebrengt. Laten we er voorlopig mee leren leven en kijken of er een refactoring ontstaat wanneer we deze nieuwe functionaliteit toevoegen.

```go
var dummyBlindAlerter = &SpyBlindAlerter{}
var dummyPlayerStore = &poker.StubPlayerStore{}
var dummyStdIn = &bytes.Buffer{}
var dummyStdOut = &bytes.Buffer{}
```

Hier is onze nieuwe test:

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	cli := poker.NewCLI(dummyPlayerStore, dummyStdIn, stdout, dummyBlindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := "Please enter the number of players: "

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

We geven `os.Stdout` door in `main` en kijken wat er geschreven wordt.

## Probeer de test uit te voeren

```
./CLI_test.go:38:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *bytes.Buffer, *bytes.Buffer, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader, poker.BlindAlerter)
```

## Schrijf de minimale hoeveelheid code voor de test om uit te voeren en controleer de mislukte testuitvoer.

We hebben een nieuwe afhankelijkheid, dus we moeten `NewCLI` bijwerken.

```go
func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI
```

De _other_ tests zullen nu niet compileren omdat ze geen `io.Writer` hebben die wordt doorgegeven aan `NewCLI`.

Voeg `dummyStdout` toe voor de andere tests.

De nieuwe test zou als volgt moeten mislukken

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
    	CLI_test.go:46: got '', want 'Please enter the number of players: '
FAIL
```

## Schrijf voldoende code om het te laten slagen

We moeten onze nieuwe afhankelijkheid toevoegen aan onze `CLI`, zodat we ernaar kunnen verwijzen in `PlayPoker`

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	out         io.Writer
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		out:         out,
		alerter:     alerter,
	}
}
```

Dan kunnen we eindelijk onze prompt aan het begin van het spel schrijven

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, "Please enter the number of players: ")
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

## Refactor

We hebben een dubbele string voor de prompt die we moeten extraheren naar een constante

```go
const PlayerPrompt = "Please enter the number of players: "
```

Gebruik dit in zowel de testcode als de `CLI`.

Nu moeten we een nummer invoeren en eruit halen. De enige manier om te weten of het het gewenste effect heeft gehad, is door te kijken welke blinde waarschuwingen er zijn gepland.

## Schrijf eerst de test

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("7\n")
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(dummyPlayerStore, in, stdout, blindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := poker.PlayerPrompt

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{12 * time.Minute, 200},
		{24 * time.Minute, 300},
		{36 * time.Minute, 400},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

Au! Veel veranderingen.

- We verwijderen onze dummy voor StdIn en sturen in plaats daarvan een nagebootste versie die onze gebruiker 7 laat invoeren.
- We verwijderen ook onze dummy voor de blinde alerter, zodat we kunnen zien dat het aantal spelers effect heeft gehad op de planning.
- We testen welke alerts er gepland zijn.

## Probeer de test uit te voeren.

De test zou nog steeds moeten compileren en mislukken door te melden dat de geplande tijden onjuist zijn, omdat we de game hard hebben geprogrammeerd om te worden gebaseerd op 5 spelers.

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s
        --- PASS: TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/200_chips_at_12m0s
```

## Schrijf genoeg code om het te laten slagen

Onthoud dat we vrij zijn om elke zonde te begaan die we nodig hebben om dit te laten werken. Zodra we werkende software hebben, kunnen we beginnen met het refactoren van de puinhoop die we gaan maken!

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)

	numberOfPlayers, _ := strconv.Atoi(cli.readLine())

	cli.scheduleBlindAlerts(numberOfPlayers)

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}
```

- We lezen `numberOfPlayersInput` in als een string.
- We gebruiken `cli.readLine()` om de invoer van de gebruiker op te halen en roepen vervolgens `Atoi` aan om deze om te zetten naar een geheel getal, waarbij we eventuele foutscenario's negeren. We moeten later een test voor dat scenario schrijven.
- Vanaf hier wijzigen we `scheduleBlindAlerts` om een _aantal spelers te accepteren_. Vervolgens berekenen we een `blindIncrement`-tijd die we toevoegen aan `blindTime` terwijl we itereren over de blinde aantallen.

Hoewel onze nieuwe test is gerepareerd, zijn veel andere mislukt, omdat ons systeem nu alleen werkt als het spel begint met een gebruiker die een getal invoert. Je moet de tests repareren door de gebruikersinvoer te wijzigen, zodat er een getal gevolgd door een nieuwe regel wordt toegevoegd (dit brengt nu nog meer tekortkomingen in onze aanpak aan het licht).

## Refactor

Dit voelt allemaal een beetje afschuwelijk, toch? Laten we **luisteren naar onze tests**.

- Om te testen of we bepaalde meldingen inplannen, hebben we vier verschillende afhankelijkheden ingesteld. Wanneer je veel afhankelijkheden hebt voor iets in je systeem, betekent dit dat het te veel doet. Visueel kunnen we dit zien aan hoe rommelig onze test is.
- Het voelt alsof we een duidelijkere abstractie moeten maken tussen het lezen van gebruikersinvoer en de bedrijfslogica die we willen uitvoeren.
- Een betere test zou zijn: _roepen we, gegeven deze gebruikersinvoer, een nieuw type 'Game' aan met het juiste aantal spelers_?
- Vervolgens zouden we de tests van de planning extraheren naar de tests voor onze nieuwe 'Game'.

We kunnen eerst refactoren naar onze 'Game' en onze test zou moeten slagen. Zodra we de gewenste structurele wijzigingen hebben aangebracht, kunnen we nadenken over hoe we de tests kunnen refactoren om onze nieuwe scheiding van aandachtspunten te weerspiegelen.

Onthoud dat je bij het aanbrengen van wijzigingen in de refactoring moet proberen deze zo klein mogelijk te houden en de tests steeds opnieuw uit te voeren.

Probeer het eerst zelf. Denk na over de grenzen van wat een 'Game' zou bieden en wat onze 'CLI' zou moeten doen.

Verander voorlopig de externe interface van 'NewCLI' niet, omdat we de testcode en de clientcode niet tegelijkertijd willen wijzigen. Dat is te veel werk en we zouden dingen kapot kunnen maken.

Dit is wat ik bedacht heb:

```go
// game.go
type Game struct {
	alerter BlindAlerter
	store   PlayerStore
}

func (p *Game) Start(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		p.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}

func (p *Game) Finish(winner string) {
	p.store.RecordWin(winner)
}

// cli.go
type CLI struct {
	in   *bufio.Scanner
	out  io.Writer
	game *Game
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		in:  bufio.NewScanner(in),
		out: out,
		game: &Game{
			alerter: alerter,
			store:   store,
		},
	}
}

const PlayerPrompt = "Please enter the number of players: "

func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)

	numberOfPlayersInput := cli.readLine()
	numberOfPlayers, _ := strconv.Atoi(strings.Trim(numberOfPlayersInput, "\n"))

	cli.game.Start(numberOfPlayers)

	winnerInput := cli.readLine()
	winner := extractWinner(winnerInput)

	cli.game.Finish(winner)
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins\n", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```
Vanuit een domeinperspectief:
- We willen een `Game` starten en aangeven hoeveel mensen er spelen.
- We willen een `Game` beëindigen en de winnaar bekendmaken.

Het nieuwe `Game`-type omvat dit voor ons.

Met deze wijziging hebben we `BlindAlerter` en `PlayerStore` doorgegeven aan `Game`, omdat dit nu verantwoordelijk is voor het melden en opslaan van resultaten.

Onze `CLI` houdt zich nu alleen bezig met:

- Het construeren van `Game` met de bestaande afhankelijkheden (die we vervolgens zullen refactoren).
- Het interpreteren van gebruikersinvoer als methodeaanroepen voor `Game`.

We willen `grote` refactoren vermijden, waardoor we gedurende langere tijd in een staat van falende tests terechtkomen, omdat dit de kans op fouten vergroot. (Als je in een groot/gedistribueerd team werkt, is dit extra belangrijk.)

Het eerste wat we doen, is `Game` refactoren, zodat we het in de `CLI` injecteren. We zullen de kleinste wijzigingen in onze tests doorvoeren om dit te vergemakkelijken en vervolgens zullen we bekijken hoe we de tests kunnen opsplitsen in de thema's van het parsen van gebruikersinvoer en gamebeheer.

Het enige wat we nu hoeven te doen, is `NewCLI` wijzigen.

```go
func NewCLI(in io.Reader, out io.Writer, game *Game) *CLI {
	return &CLI{
		in:   bufio.NewScanner(in),
		out:  out,
		game: game,
	}
}
```

Dit voelt al als een verbetering. We hebben minder afhankelijkheden en _onze afhankelijkheidslijst weerspiegelt ons algemene ontwerpdoel_: de CLI houdt zich bezig met invoer/uitvoer en delegeert gamespecifieke acties aan een `Game`.

Als je probeert te compileren, treden er problemen op. Je zou deze problemen zelf moeten kunnen oplossen. Maak je nu nog geen zorgen over het maken van mocks voor `Game`, initialiseer gewoon de _echte_ `Game` om alles te compileren en de tests groen te maken.

Om dit te doen, moet je een constructor maken

```go
func NewGame(alerter BlindAlerter, store PlayerStore) *Game {
	return &Game{
		alerter: alerter,
		store:   store,
	}
}
```

Hier is een voorbeeld van een van de opstellingen voor de tests die worden gerepareerd

```go
stdout := &bytes.Buffer{}
in := strings.NewReader("7\n")
blindAlerter := &SpyBlindAlerter{}
game := poker.NewGame(blindAlerter, dummyPlayerStore)

cli := poker.NewCLI(in, stdout, game)
cli.PlayPoker()
```

Het zou niet veel moeite moeten kosten om de tests te repareren en weer groen te zijn (dat is juist de bedoeling!), maar zorg er wel voor dat je ook `main.go` repareert voordat je doorgaat naar de volgende fase.

```go
// main.go
game := poker.NewGame(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
cli := poker.NewCLI(os.Stdin, os.Stdout, game)
cli.PlayPoker()
```

Nu we `Game` hebben geëxtraheerd, moeten we onze gamespecifieke beweringen verplaatsen naar tests die losstaan van de CLI.

Dit is slechts een oefening in het kopiëren van onze `CLI`-tests, maar dan met minder afhankelijkheden.

```go
func TestGame_Start(t *testing.T) {
	t.Run("schedules alerts on game start for 5 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(5)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 10 * time.Minute, Amount: 200},
			{At: 20 * time.Minute, Amount: 300},
			{At: 30 * time.Minute, Amount: 400},
			{At: 40 * time.Minute, Amount: 500},
			{At: 50 * time.Minute, Amount: 600},
			{At: 60 * time.Minute, Amount: 800},
			{At: 70 * time.Minute, Amount: 1000},
			{At: 80 * time.Minute, Amount: 2000},
			{At: 90 * time.Minute, Amount: 4000},
			{At: 100 * time.Minute, Amount: 8000},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

	t.Run("schedules alerts on game start for 7 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(7)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 12 * time.Minute, Amount: 200},
			{At: 24 * time.Minute, Amount: 300},
			{At: 36 * time.Minute, Amount: 400},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

}

func TestGame_Finish(t *testing.T) {
	store := &poker.StubPlayerStore{}
	game := poker.NewGame(dummyBlindAlerter, store)
	winner := "Ruth"

	game.Finish(winner)
	poker.AssertPlayerWin(t, store, winner)
}
```

De bedoeling achter wat er gebeurt wanneer een pokerspel begint, is nu veel duidelijker.

Zorg ervoor dat je ook de test verplaatst wanneer het spel eindigt.

Zodra we tevreden zijn met de verplaatsing van de tests voor spellogica, kunnen we onze CLI-tests vereenvoudigen, zodat ze onze beoogde verantwoordelijkheden duidelijker weergeven.

- Gebruikersinvoer verwerken en de methoden van `Game` aanroepen wanneer dat nodig is.
- Uitvoer verzenden.
- Cruciaal is dat de CLI niet op de hoogte is van de daadwerkelijke werking van spellen.

Om dit te doen, moeten we ervoor zorgen dat `CLI` niet langer afhankelijk is van een concreet `Game`-type, maar in plaats daarvan een interface accepteert met `Start(aantalSpelers)` en `Finish(winnaar)`. We kunnen dan een spion van dat type aanmaken en controleren of de juiste calls worden gemaakt.

Hier realiseren we ons dat het benoemen soms lastig is. Hernoem `Game` naar `TexasHoldem` (aangezien dat het _soort_ spel is dat we spelen) en de nieuwe interface zal `Game` heten. Hiermee wordt trouw gebleven aan het idee dat onze CLI geen weet heeft van het spel dat we spelen en wat er gebeurt als u op `Start` en `Finish` klikt.

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

Vervang alle verwijzingen naar `*Game` in `CLI` door `Game` (onze nieuwe interface). Blijf zoals altijd de tests opnieuw uitvoeren om te controleren of alles groen is tijdens het refactoren.

Nu we `CLI` hebben losgekoppeld van `TexasHoldem`, kunnen we spies gebruiken om te controleren of `Start` en `Finish` worden aangeroepen wanneer we dat verwachten, met de juiste argumenten.

Maak een spy die `Game` implementeert

```go
type GameSpy struct {
	StartedWith  int
	FinishedWith string
}

func (g *GameSpy) Start(numberOfPlayers int) {
	g.StartedWith = numberOfPlayers
}

func (g *GameSpy) Finish(winner string) {
	g.FinishedWith = winner
}
```

Vervang elke `CLI`-test die gamespecifieke logica test door controles op de aanroep van onze `GameSpy`. Dit zal dan duidelijk de verantwoordelijkheden van de CLI in onze tests weergeven.

Hier is een voorbeeld van een van de tests die wordt gerepareerd; probeer de rest zelf uit te voeren en controleer de broncode als je vastloopt.

```go
	t.Run("it prompts the user to enter the number of players and starts the game", func(t *testing.T) {
		stdout := &bytes.Buffer{}
		in := strings.NewReader("7\n")
		game := &GameSpy{}

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		gotPrompt := stdout.String()
		wantPrompt := poker.PlayerPrompt

		if gotPrompt != wantPrompt {
			t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
		}

		if game.StartedWith != 7 {
			t.Errorf("wanted Start called with 7 but got %d", game.StartedWith)
		}
	})
```

Nu we een duidelijke scheiding van aandachtspunten hebben, zou het controleren van grensgevallen rond IO in onze `CLI` eenvoudiger moeten zijn.

We moeten het scenario aanpakken waarbij een gebruiker een niet-numerieke waarde invoert wanneer er om het aantal spelers wordt gevraagd:

Onze code zou het spel niet moeten starten en een handige foutmelding aan de gebruiker moeten tonen en vervolgens moeten afsluiten.

## Schrijf eerst de test

We beginnen met ervoor te zorgen dat het spel niet start

```go
t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("Pies\n")
	game := &GameSpy{}

	cli := poker.NewCLI(in, stdout, game)
	cli.PlayPoker()

	if game.StartCalled {
		t.Errorf("game should not have started")
	}
})
```

Je moet aan onze `GameSpy` een veld `StartCalled` toevoegen, dat alleen wordt aangemaakt als `Start` wordt aangeroepen.

## Probeer de test uit te voeren

```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:62: game should not have started
```

## Schrijf voldoende code om het te laten slagen

Rond de plek waar we `Atoi` aanroepen, hoeven we alleen nog maar te controleren op de fout

```go
numberOfPlayers, err := strconv.Atoi(cli.readLine())

if err != nil {
	return
}
```

Vervolgens moeten we de gebruiker informeren over wat hij/zij fout heeft gedaan, zodat we een bewering kunnen doen op basis van wat er naar `stdout` is gestuurd.

## Schrijf eerst de test

We hebben eerder een bewering gedaan op basis van wat er naar `stdout` is gestuurd, dus we kunnen die code nu kopiëren.

```go
gotPrompt := stdout.String()

wantPrompt := poker.PlayerPrompt + "you're so silly"

if gotPrompt != wantPrompt {
	t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
}
```

We slaan _alles_ op dat naar stdout wordt geschreven, dus we verwachten nog steeds de `poker.PlayerPrompt`. Vervolgens controleren we of er nog iets wordt afgedrukt. We maken ons nu niet al te druk om de exacte bewoording, we pakken het aan wanneer we de refactoring uitvoeren.

## Probeer de test uit te voeren

```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:70: got 'Please enter the number of players: ', want 'Please enter the number of players: you're so silly'
```

## Schrijf voldoende code om het te laten slagen

Wijzig de code voor foutverwerking

```go
if err != nil {
	fmt.Fprint(cli.out, "you're so silly")
	return
}
```

## Refactor

Refactor het bericht nu naar een constante zoals `PlayerPrompt`

```go
wantPrompt := poker.PlayerPrompt + poker.BadPlayerInputErrMsg
```

en een meer passende boodschap plaatsen

```go
const BadPlayerInputErrMsg = "Bad value received for number of players, please try again with a number"
```

Ten slotte is ons testen van wat er naar `stdout` is verzonden vrij uitgebreid. Laten we een assert-functie schrijven om het op te schonen.

```go
func assertMessagesSentToUser(t testing.TB, stdout *bytes.Buffer, messages ...string) {
	t.Helper()
	want := strings.Join(messages, "")
	got := stdout.String()
	if got != want {
		t.Errorf("got %q sent to stdout but expected %+v", got, messages)
	}
}
```

Het gebruik van de vararg-syntaxis (`...string`) is hier handig, omdat we moeten asserten op verschillende aantallen berichten.

Gebruik deze helper in beide tests waarin we asserten op berichten die naar de gebruiker worden verzonden.

Er zijn een aantal tests die geholpen kunnen worden met sommige `assertX`-functies, dus oefen je refactoring door onze tests op te schonen zodat ze goed leesbaar zijn.

Neem even de tijd om na te denken over de waarde van een aantal van de tests die we hebben uitgevoerd. Onthoud dat we niet meer tests willen dan nodig is. Kun je een aantal ervan refactoren/verwijderen _en er toch zeker van zijn dat alles werkt_?

Dit is wat ik heb bedacht.

```go
func TestCLI(t *testing.T) {

	t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
		game := &GameSpy{}
		stdout := &bytes.Buffer{}

		in := userSends("3", "Chris wins")
		cli := poker.NewCLI(in, stdout, game)

		cli.PlayPoker()

		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt)
		assertGameStartedWith(t, game, 3)
		assertFinishCalledWith(t, game, "Chris")
	})

	t.Run("start game with 8 players and record 'Cleo' as winner", func(t *testing.T) {
		game := &GameSpy{}

		in := userSends("8", "Cleo wins")
		cli := poker.NewCLI(in, dummyStdOut, game)

		cli.PlayPoker()

		assertGameStartedWith(t, game, 8)
		assertFinishCalledWith(t, game, "Cleo")
	})

	t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
		game := &GameSpy{}

		stdout := &bytes.Buffer{}
		in := userSends("pies")

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		assertGameNotStarted(t, game)
		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt, poker.BadPlayerInputErrMsg)
	})
}
```
De tests weerspiegelen nu de belangrijkste mogelijkheden van `CLI`. Het kan gebruikersinvoer lezen in termen van hoeveel spelers er spelen en wie er gewonnen heeft, en handelt af wanneer er een onjuiste waarde voor het aantal spelers wordt ingevoerd. Hierdoor is het voor de lezer duidelijk wat `CLI` doet, maar ook wat het niet doet.

Wat gebeurt er als de gebruiker in plaats van `Ruth wint` `Lloyd is een moordenaar` invoert?

Sluit dit hoofdstuk af door een test voor dit scenario te schrijven en deze te laten slagen.

## Samenvattend

### Een korte samenvatting van het project

De afgelopen 5 hoofdstukken hebben we langzaam een behoorlijke hoeveelheid code TDD'd.

- We hebben twee applicaties: een opdrachtregelapplicatie en een webserver.
- Beide applicaties maken gebruik van een `PlayerStore` om winnaars bij te houden.
- De webserver kan ook een ranglijst weergeven van wie de meeste games wint.
- De opdrachtregelapplicatie helpt spelers een potje poker te spelen door bij te houden wat de huidige blindwaarde is.

### time.Afterfunc

Een zeer handige manier om een functieaanroep na een bepaalde tijdsduur te plannen. Het is zeker de moeite waard om er tijd in te steken [bekijk de documentatie voor `time`](https://golang.org/pkg/time/) omdat het veel tijdbesparende functies en methoden bevat waarmee je kunt werken.

Een paar van mijn favorieten zijn:

- `time.After(duration)` retourneert een `chan Time` wanneer de tijdsduur is verstreken. Dus als je iets _na_ een bepaalde tijd wilt doen, kan dit helpen.
- `time.NewTicker(duration)` retourneert een `Ticker` die vergelijkbaar is met de bovenstaande, in die zin dat het een kanaal retourneert, maar deze "tikt" elke tijdsduur, in plaats van slechts één keer. Dit is erg handig als je elke `N tijdsduur` code wilt uitvoeren.

### Meer voorbeelden van een goede scheiding van aandachtspunten

_Over het algemeen_ is het een goede gewoonte om de verantwoordelijkheden voor het omgaan met gebruikersinvoer en -reacties te scheiden van de domeincode. Je ziet dat hier in onze commandline-applicatie en ook op onze webserver.

Onze tests raakten rommelig. We hadden te veel assertions (controleer deze invoer, plan deze waarschuwingen, enz.) en te veel afhankelijkheden. We konden visueel zien dat het rommelig was; het is **zo belangrijk om naar je tests te luisteren**.

- Als je tests er rommelig uitzien, probeer ze dan te refactoren.
- Als je dit hebt gedaan en ze nog steeds rommelig zijn, wijst dat zeer waarschijnlijk op een fout in je ontwerp.
- Dit is een van de echte sterke punten van tests.

Hoewel de tests en de productiecode wat rommelig waren, konden we vrijelijk refactoren, ondersteund door onze tests.

Onthoud in dergelijke situaties dat je altijd kleine stapjes moet zetten en de tests na elke wijziging opnieuw moet uitvoeren.

Het zou gevaarlijk zijn geweest om zowel de testcode als de productiecode tegelijkertijd te refactoren, dus hebben we eerst de productiecode gerefactored (in de huidige staat konden we de tests niet veel verbeteren) zonder de interface aan te passen, zodat we zoveel mogelijk op onze tests konden vertrouwen terwijl we dingen aanpasten. _Vervolgens_ hebben we de tests gerefactored nadat het ontwerp was verbeterd.

Na de refactoring weerspiegelde de afhankelijkheidslijst ons ontwerpdoel. Dit is een ander voordeel van DI, omdat het vaak de intentie documenteert. Wanneer je afhankelijk bent van globale variabelen, worden de verantwoordelijkheden erg onduidelijk.

## Een voorbeeld van een functie die een interface implementeert

Wanneer je een interface definieert met één methode erin, kun je overwegen om een `MyInterfaceFunc`-type te definiëren als aanvulling hierop, zodat gebruikers je interface met slechts één functie kunnen implementeren.

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

// BlindAlerterFunc allows you to implement BlindAlerter with a function
type BlindAlerterFunc func(duration time.Duration, amount int)

// ScheduleAlertAt is BlindAlerterFunc implementation of BlindAlerter
func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}
```

By doing this, people using your library can implement your interface with just a function. They can use [Type Conversion](https://go.dev/tour/basics/13) to convert their function into a `BlindAlerterFunc` and then use it as a BlindAlerter (as `BlindAlerterFunc` implements `BlindAlerter`).

```go
game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
```

Het algemene punt is dat je in Go methoden aan _typen_ kunt toevoegen, niet alleen aan structs. Dit is een zeer krachtige functie, die je kunt gebruiken om interfaces op een handigere manier te implementeren.

Bedenk dat je niet alleen typen van functies kunt definiëren, maar ook typen rond andere typen, zodat je er methoden aan kunt toevoegen.

```go
type Blog map[string]string

func (b Blog) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, b[r.URL.Path])
}
```

We hebben hier een HTTP-handler gemaakt die een heel eenvoudige "blog" implementeert, waarbij URL-paden als sleutels naar berichten in een kaart worden gebruikt.
