# Opdrachtregel- en projectstructuur

**[Je vindt alle code voor dit hoofdstuk hier](https://github.com/quii/learn-go-with-tests/tree/main/command-line)**

Onze Product Owner wil nu een _pivot_ maken door een tweede applicatie te introduceren: een opdrachtregelapplicatie.

Voorlopig hoeft deze alleen de winst van een speler te kunnen registreren wanneer de gebruiker `Ruth wint` typt. Het is de bedoeling dat het uiteindelijk een tool wordt die gebruikers helpt poker te spelen.

De Product Owner wil dat de database wordt gedeeld tussen de twee applicaties, zodat de competitie wordt bijgewerkt op basis van de winsten die in de nieuwe applicatie worden geregistreerd.

## Een herinnering aan de code

We hebben een applicatie met een `main.go`-bestand dat een HTTP-server start. De HTTP-server is voor ons niet interessant voor deze oefening, maar de abstractie die deze gebruikt wel. Deze is afhankelijk van een `PlayerStore`.

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() League
}
```

In het vorige hoofdstuk hebben we een `FileSystemPlayerStore` gemaakt die die interface implementeert. We zouden een deel hiervan moeten kunnen hergebruiken voor onze nieuwe applicatie.

## Eerst wat projectrefactoring

Ons project moet nu twee binaire bestanden aanmaken: onze bestaande webserver en de commandline-app.

Voordat we aan ons nieuwe werk beginnen, moeten we ons project structureren om dit mogelijk te maken.

Tot nu toe stond alle code in één map, in een pad dat er als volgt uitziet

`$GOPATH/src/github.com/your-name/my-app`

Om een applicatie in Go te maken, heb je een `main`-functie nodig binnen een `package main`. Tot nu toe stond al onze `domein`-code in `package main` en onze `func main` kan naar alles verwijzen.

Dit werkte tot nu toe prima en het is een goede gewoonte om niet te overdrijven met de pakketstructuur. Als je de tijd neemt om de standaardbibliotheek te bekijken, zul je weinig mappen en structuur zien.

Gelukkig is het vrij eenvoudig om structuur toe te voegen _wanneer je die nodig hebt_.

Maak binnen het bestaande project een `cmd`-directory aan met daarin een `webserver`-directory (bijv. `mkdir -p cmd/webserver`).

Verplaats `main.go` daarheen.

Als je `tree` hebt geïnstalleerd, voer je het uit en je structuur zou er zo uit moeten zien.

```
.
|-- file_system_store.go
|-- file_system_store_test.go
|-- cmd
|   |-- webserver
|       |-- main.go
|-- league.go
|-- server.go
|-- server_integration_test.go
|-- server_test.go
|-- tape.go
|-- tape_test.go
```

We hebben nu effectief een scheiding tussen onze applicatie en de bibliotheekcode, maar we moeten nu enkele pakketnamen wijzigen. Onthoud dat wanneer je een Go-applicatie bouwt, het pakket `main` moet zijn.

Wijzig alle andere code zodat het een pakket met de naam `poker` heeft.

Ten slotte moeten we dit pakket importeren in `main.go`, zodat we het kunnen gebruiken om onze webserver te creëren. Vervolgens kunnen we onze bibliotheekcode gebruiken met behulp van `poker.FunctionName`.

De paden zullen op jouw computer anders zijn, maar het zou er ongeveer zo uit moeten zien:

```go
// cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v1"
	"log"
	"net/http"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	server := poker.NewPlayerServer(store)

	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Het volledige pad lijkt misschien wat verwarrend, maar zo kun je _elke_ openbaar beschikbare bibliotheek in je code importeren.

Door onze domeincode op te splitsen in een apart pakket en dit te committen naar een openbare repository zoals GitHub, kan elke Go-ontwikkelaar zijn eigen code schrijven die de functies die we `public` hebben geschreven, importeert uit dat pakket. De eerste keer dat je het probeert uit te voeren, krijg je een melding dat het niet bestaat, maar je hoeft alleen maar `go get` uit te voeren.

Daarnaast kunnen gebruikers de [documentatie op pkg.go.dev](https://pkg.go.dev/github.com/quii/learn-go-with-tests/command-line/v1) bekijken.

### Laatste controles

- Voer binnen de root `go test` uit en controleer of ze nog steeds slagen.
- Ga naar onze `cmd/webserver` en voer `go run main.go` uit.
- Ga naar `http://localhost:5000/league` en je zou moeten zien dat het nog steeds werkt.

### Wandelend skelet

Voordat we beginnen met het schrijven van tests, voegen we een nieuwe applicatie toe die ons project zal bouwen. Maak een nieuwe directory binnen `cmd` aan met de naam `cli` (command line interface) en voeg een `main.go` toe met het volgende.

```go
// cmd/cli/main.go
package main

import "fmt"

func main() {
	fmt.Println("Let's play poker")
}
```

De eerste vereiste die we zullen aanpakken, is het registreren van een winst wanneer de gebruiker `{PlayerName} wins` typt.

## Schrijf eerst de test

We weten dat we iets moeten maken dat `CLI` heet, waarmee we poker kunnen spelen. Het moet gebruikersinvoer lezen en vervolgens winsten registreren in een `PlayerStore`.

Maar voordat we te ver vooruitlopen, laten we eerst een test schrijven om te controleren of deze integreert met de `PlayerStore` zoals we willen.

Binnen `CLI_test.go` (in de root van het project, niet binnen `cmd`)

```go
// CLI_test.go
package poker

import "testing"

func TestCLI(t *testing.T) {
	playerStore := &StubPlayerStore{}
	cli := &CLI{playerStore}
	cli.PlayPoker()

	if len(playerStore.winCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}
}
```

- We kunnen onze `StubPlayerStore` gebruiken vanuit andere tests
- We geven onze afhankelijkheid door aan ons nog niet bestaande `CLI`-type
- Het spel activeren met een ongeschreven `PlayPoker`-methode
- Controleren of er een winst is geregistreerd

## Probeer de test uit te voeren

```
# github.com/quii/learn-go-with-tests/command-line/v2
./cli_test.go:25:10: undefined: CLI
```

## Schrijf de minimale hoeveelheid code om de test uit te voeren en controleer de mislukte testuitvoer.

Op dit punt zou je voldoende vertrouwd moeten zijn met het maken van onze nieuwe `CLI`-structuur met het betreffende veld voor onze afhankelijkheid en het toevoegen van een methode.

Je zou uiteindelijk code zoals deze moeten krijgen.

```go
// CLI.go
package poker

type CLI struct {
	playerStore PlayerStore
}

func (cli *CLI) PlayPoker() {}
```

Bedenk dat we alleen maar proberen de test te laten draaien, zodat we kunnen controleren of de test mislukt zoals we hadden gehoopt.

```
--- FAIL: TestCLI (0.00s)
    cli_test.go:30: expected a win call but didn't get any
FAIL
```

## Schrijf genoeg code om de test te laten slagen

```go
//CLI.go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Cleo")
}
```

Daarmee zou het moeten lukken.

Vervolgens moeten we het lezen van `Stdin` (de invoer van de gebruiker) simuleren, zodat we de winst van specifieke spelers kunnen registreren.

Laten we onze test uitbreiden om dit te oefenen.

## Schrijf eerst de test

```go
//CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	if len(playerStore.winCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}

	got := playerStore.winCalls[0]
	want := "Chris"

	if got != want {
		t.Errorf("didn't record correct winner, got %q, want %q", got, want)
	}
}
```

`os.Stdin` gebruiken we in `main` om de invoer van de gebruiker vast te leggen. Het is een `*File` onder de motorkap, wat betekent dat het `io.Reader` implementeert, wat, zoals we inmiddels weten, een handige manier is om tekst vast te leggen.

We maken in onze test een `io.Reader` aan met behulp van de handige `strings.NewReader` en vullen deze met wat we verwachten dat de gebruiker typt.

## Probeer de test uit te voeren

`./CLI_test.go:12:32: too many values in struct initializer`

## Schrijf de minimale hoeveelheid code die nodig is om de test uit te voeren en controleer de mislukte testuitvoer.

We moeten onze nieuwe afhankelijkheid toevoegen aan `CLI`.

```go
//CLI.go
type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}
```

```
--- FAIL: TestCLI (0.00s)
    CLI_test.go:23: didn't record the correct winner, got 'Cleo', want 'Chris'
FAIL
```

## Schrijf voldoende code om het te laten slagen

Onthoud dat je eerst het strikt eenvoudigste moet doen

```go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Chris")
}
```

De test is geslaagd. We voegen nog een test toe om ons te dwingen echte code te schrijven, maar laten we eerst refactoren.

## Refactor

In `server_test` hebben we eerder controles uitgevoerd om te zien of er winsten worden geregistreerd, zoals hier. Laten we die bewering `DRY` maken tot een helper.

```go
//server_test.go
func assertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}
```

Vervang nu de beweringen in zowel `server_test.go` als `CLI_test.go`.

De test zou er nu als volgt uit moeten zien:

```go
//CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	assertPlayerWin(t, playerStore, "Chris")
}
```

Laten we nu nog een test schrijven met andere gebruikersinvoer om ons te dwingen de test daadwerkelijk te lezen.

## Schrijf eerst de test

```go
//CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Cleo")
	})

}
```

## Probeer de test uit te voeren

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/record_chris_win_from_user_input
    --- PASS: TestCLI/record_chris_win_from_user_input (0.00s)
=== RUN   TestCLI/record_cleo_win_from_user_input
    --- FAIL: TestCLI/record_cleo_win_from_user_input (0.00s)
        CLI_test.go:27: did not store correct winner got 'Chris' want 'Cleo'
FAIL
```

## Schrijf genoeg code om het te laten slagen

We gebruiken een [`bufio.Scanner`](https://golang.org/pkg/bufio/) om de invoer van de `io.Reader` te lezen.

> Pakket bufio implementeert gebufferde I/O. Het verpakt een io.Reader- of io.Writer-object en creëert daarmee een ander object (Reader of Writer) dat ook de interface implementeert, maar buffering en enige hulp biedt voor tekstuele I/O.

Werk de code bij naar het volgende


```go
//CLI.go
type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}

func (cli *CLI) PlayPoker() {
	reader := bufio.NewScanner(cli.in)
	reader.Scan()
	cli.playerStore.RecordWin(extractWinner(reader.Text()))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}
```

De tests zullen nu slagen.

- `Scanner.Scan()` zal tot een nieuwe regel inlezen.
- Vervolgens gebruiken we `Scanner.Text()` om de `string` te retourneren die de scanner heeft gelezen.

Nu we een aantal geslaagde tests hebben, moeten we dit koppelen aan `main`. Vergeet niet dat we er altijd naar moeten streven om zo snel mogelijk volledig geïntegreerde, werkende software te hebben.

Voeg in `main.go` het volgende toe en voer het uit. (Mogelijk moet je het pad van de tweede afhankelijkheid aanpassen zodat deze overeenkomt met wat er op je computer staat)

```go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")

	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.CLI{store, os.Stdin}
	game.PlayPoker()
}
```

Je zou een fout moeten krijgen

```
command-line/v3/cmd/cli/main.go:32:25: implicit assignment of unexported field 'playerStore' in poker.CLI literal
command-line/v3/cmd/cli/main.go:32:34: implicit assignment of unexported field 'in' in poker.CLI literal
```

Wat hier gebeurt, is dat we proberen toe te wijzen aan de velden `playerStore` en `in` in `CLI`. Dit zijn niet-geëxporteerde (private) velden. We _zouden_ dit in onze testcode kunnen doen, omdat onze test zich in hetzelfde pakket bevindt als `CLI` (`poker`). Maar onze `main` zit in het pakket `main` en heeft dus geen toegang.

Dit benadrukt het belang van het _integreren van je werk_. We hebben de afhankelijkheden van onze `CLI` terecht privé gemaakt (omdat we niet willen dat ze zichtbaar zijn voor gebruikers van `CLI`s), maar hebben geen manier gevonden waarop gebruikers deze kunnen construeren.

Is er een manier om dit probleem eerder te signaleren?

### `package mypackage_test`

In alle andere voorbeelden tot nu toe declareren we, wanneer we een testbestand maken, dat bestand in hetzelfde pakket zit als dat we testen.

Dit is prima en het betekent dat we, in het uitzonderlijke geval dat we iets intern in het pakket willen testen, toegang hebben tot de niet-geëxporteerde typen.

Maar aangezien we hebben gepleit voor het _over het algemeen_ niet testen van interne dingen, kan Go dat helpen afdwingen? Wat als we onze code zouden kunnen testen waar we alleen toegang hebben tot de geëxporteerde typen (zoals onze `main` doet)?

Als je een project met meerdere pakketten schrijft, raad ik je ten zeerste aan om de naam van je testpakket te laten eindigen op `_test`. Wanneer je dit doet, heb je alleen toegang tot de openbare typen in je pakket. Dit zou helpen in dit specifieke geval, maar helpt ook om de discipline te handhaven om alleen openbare API's te testen. Als je toch interne zaken wilt testen, kun je een aparte test maken met het pakket dat je wilt testen.

Een gezegde bij TDD is dat als je je code niet kunt testen, het voor gebruikers van je code waarschijnlijk moeilijk is om ermee te integreren. Het gebruik van `package foo_test` helpt hierbij, omdat je dan gedwongen wordt je code te testen alsof je hem importeert, net zoals gebruikers van je pakket dat doen.

Voordat we `main` aanpassen, wijzigen we eerst het pakket van onze test in `CLI_test.go` naar `poker_test`.

Als je een goed geconfigureerde IDE hebt, zie je plotseling veel rood! Als je de compiler uitvoert, krijg je de volgende foutmeldingen.

```
./CLI_test.go:12:19: undefined: StubPlayerStore
./CLI_test.go:17:3: undefined: assertPlayerWin
./CLI_test.go:22:19: undefined: StubPlayerStore
./CLI_test.go:27:3: undefined: assertPlayerWin
```

We zijn nu op meer vragen over pakketontwerp gestuit. Om onze software te testen, hebben we niet-geëxporteerde stubs en helperfuncties gemaakt die we niet langer kunnen gebruiken in onze `CLI_test`, omdat de helpers gedefinieerd zijn in de `_test.go`-bestanden in het `poker`-pakket.

#### Willen we onze stubs en helpers 'openbaar' maken?

Dit is een subjectieve discussie. Je zou kunnen stellen dat je de API van je pakket niet wilt vervuilen met code om tests mogelijk te maken.

In de presentatie ["Advanced Testing with Go"](https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=53) van Mitchell Hashimoto wordt beschreven hoe HashiCorp dit bepleit, zodat gebruikers van het pakket tests kunnen schrijven zonder het wiel opnieuw te hoeven uitvinden door stubs te schrijven. In ons geval zou dit betekenen dat iedereen die ons `poker`-pakket gebruikt, geen eigen `PlayerStore`-stub hoeft te maken om met onze code te werken.

Ik heb deze techniek in andere gedeelde pakketten gebruikt en het is enorm nuttig gebleken voor gebruikers die tijd besparen bij de integratie met onze pakketten.

Laten we dus een bestand maken met de naam `testing.go` en daar onze stub en helpers aan toevoegen.

```go
// testing.go
package poker

import "testing"

type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}

func (s *StubPlayerStore) GetLeague() League {
	return s.league
}

func AssertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}

// todo for you - the rest of the helpers
```

Je moet de helpers `public` maken (onthoud dat exporteren met een hoofdletter begint) als je wilt dat ze zichtbaar zijn voor importeurs van ons pakket.

In onze CLI-test moet je de code aanroepen alsof je deze in een ander pakket gebruikt.

```go
//CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Cleo")
	})

}
```

Je zult nu zien dat we dezelfde problemen hebben als in `main`


```
./CLI_test.go:15:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:15:39: implicit assignment of unexported field 'in' in poker.CLI literal
./CLI_test.go:25:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:25:39: implicit assignment of unexported field 'in' in poker.CLI literal
```

De makkelijkste manier om dit te omzeilen is door een constructor te maken zoals we die voor andere typen hebben. We zullen ook `CLI` aanpassen zodat deze een `bufio.Scanner` opslaat in plaats van de reader, omdat deze nu automatisch wordt ingepakt tijdens de constructie.

```go
//CLI.go
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
```

Door dit te doen, kunnen we onze leescode vereenvoudigen en refactoren

```go
//CLI.go
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

Wijzig de test zodat deze de constructor gebruikt en we zouden weer terug moeten zijn bij de tests die geslaagd zijn.

Ten slotte kunnen we teruggaan naar onze nieuwe `main.go` en de constructor gebruiken die we zojuist hebben gemaakt.

```go
//cmd/cli/main.go
game := poker.NewCLI(store, os.Stdin)
```

Probeer het eens uit te voeren, typ "Bob wins".

### Refactor

We hebben wat herhaling in onze respectievelijke applicaties waarbij we een bestand openen en een `file_system_store` aanmaken op basis van de inhoud. Dit voelt als een kleine zwakte in het ontwerp van ons pakket, dus we zouden er een functie in moeten maken die het openen van een bestand vanaf een pad en het retourneren van de `PlayerStore` inkapselt.

```go
//file_system_store.go
func FileSystemPlayerStoreFromFile(path string) (*FileSystemPlayerStore, func(), error) {
	db, err := os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		return nil, nil, fmt.Errorf("problem opening %s %v", path, err)
	}

	closeFunc := func() {
		db.Close()
	}

	store, err := NewFileSystemPlayerStore(db)

	if err != nil {
		return nil, nil, fmt.Errorf("problem creating file system player store, %v ", err)
	}

	return store, closeFunc, nil
}
```

Refactor nu beide applicaties om deze functie te gebruiken om de opslag te creëren.

#### CLI-applicatiecode

```go
// cmd/cli/main.go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")
	poker.NewCLI(store, os.Stdin).PlayPoker()
}
```

#### Web server applicatie code

```go
// cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"net/http"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	server := poker.NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

Notice the symmetry: despite being different user interfaces the setup is almost identical. This feels like good validation of our design so far.
And notice also that `FileSystemPlayerStoreFromFile` returns a closing function, so we can close the underlying file once we are done using the Store.

## Samenvattend

### Pakketstructuur

Dit hoofdstuk hield in dat we twee applicaties wilden maken, waarbij we de domeincode die we tot nu toe hadden geschreven, hergebruikten. Om dit te doen, moesten we onze pakketstructuur bijwerken, zodat we aparte mappen hadden voor onze respectievelijke `main`-pakketten.

Door dit te doen, liepen we tegen integratieproblemen aan vanwege niet-geëxporteerde waarden, wat de waarde van het werken in kleine "slices" en frequent integreren verder aantoont.

We hebben geleerd hoe `mypackage_test` ons helpt een testomgeving te creëren die dezelfde ervaring biedt als andere pakketten die met je code integreren. Zo kun je integratieproblemen opsporen en zien hoe gemakkelijk (of niet!) je code te gebruiken is.

### Gebruikersinvoer lezen

We zagen hoe gemakkelijk het is om te lezen vanuit `os.Stdin`, omdat het `io.Reader` implementeert. We gebruikten `bufio.Scanner` om gebruikersinvoer regel voor regel eenvoudig te lezen.

### Eenvoudige abstracties leiden tot eenvoudiger hergebruik van code

Het kostte bijna geen moeite om `PlayerStore` in onze nieuwe applicatie te integreren (nadat we de aanpassingen aan het pakket hadden doorgevoerd) en het testen ervan was ook heel eenvoudig, omdat we besloten om ook onze stubversie beschikbaar te stellen.
