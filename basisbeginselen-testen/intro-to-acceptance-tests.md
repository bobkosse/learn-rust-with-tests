# Inleiding tot acceptatietesten

Bij `$WORK` zijn we de noodzaak tegengekomen van een "graceful shutdown" voor onze services. Graceful shutdown zorgt ervoor dat je systeem zijn werk correct afrondt voordat het wordt beëindigd. Een realistische vergelijking zou iemand zijn die een telefoongesprek netjes probeert af te ronden voordat hij doorgaat naar de volgende vergadering, in plaats van midden in een zin op te hangen.

Dit hoofdstuk geeft een introductie tot graceful shutdown in de context van een HTTP-server en hoe je "acceptatietests" schrijft om jezelf het vertrouwen te geven in het gedrag van je code.

Na het lezen hiervan weet je hoe je pakketten kunt delen met uitstekende tests, onderhoudswerkzaamheden kunt verminderen en het vertrouwen in de kwaliteit van je werk kunt vergroten.

## Net genoeg info over Kubernetes

We draaien onze software op [Kubernetes](https://kubernetes.io/) (K8s). K8s beëindigt "pods" (in de praktijk onze software) om verschillende redenen, en een veelvoorkomende reden is wanneer we nieuwe code pushen die we willen implementeren.

We stellen hoge eisen aan [DORA-statistieken](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance), dus we werken op een manier waarbij we kleine, incrementele verbeteringen en functies meerdere keren per dag in productie implementeren.

Wanneer k8s een pod wil beëindigen, start het een ["termination lifecycle"](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace), en een onderdeel daarvan is het verzenden van een SIGTERM-signaal naar onze software. Dit is wat k8s onze code vertelt:

> Je moet jezelf afsluiten en al het werk afmaken waar je mee bezig bent, want na een bepaalde "grace period" stuur ik `SIGKILL` en is het voor jou gedaan.

Bij `SIGKILL` wordt al het werk dat je programma mogelijk aan het doen was, onmiddellijk gestopt.

## Als je geen grace periode hebt

Afhankelijk van de aard van de software kun je problemen ondervinden als je `SIGTERM` negeert.

Ons specifieke probleem betrof in-flight HTTP-verzoeken. Wanneer een geautomatiseerde test onze API uitvoerde en k8s besloot de pod te stoppen, liep de server vast, kreeg de test geen antwoord van de server en mislukte de test.

Dit activeerde een waarschuwing in ons incidentenkanaal, waardoor een ontwikkelaar moest stoppen met wat hij aan het doen was en het probleem moest oplossen. Deze incidentele storingen vormen een vervelende afleiding voor ons team.

Deze problemen doen zich niet alleen voor bij onze tests. Als een gebruiker een verzoek naar je systeem stuurt en het proces halverwege wordt beëindigd, krijgt deze gebruiker waarschijnlijk een 5xx HTTP-foutmelding, niet de gebruikerservaring die je wilt bieden.

## Als je wel een grace periode hebt

Wat we willen doen is luisteren naar `SIGTERM`, en in plaats van de server direct te sluiten, willen we:

- Stoppen met luisteren naar verdere verzoeken
- Alle lopende verzoeken toestaan om te worden voltooid
- *Vervolgens* het proces beëindigen

## Hoe je een grace periode kunt hebben

Gelukkig heeft Go al een mechanisme om een server netjes af te sluiten met [net/http/Server.Shutdown](https://pkg.go.dev/net/http#Server.Shutdown).

> Met Shutdown wordt de server netjes afgesloten zonder actieve verbindingen te onderbreken. Shutdown werkt door eerst alle open listeners te sluiten, vervolgens alle inactieve verbindingen te sluiten en vervolgens oneindig te wachten tot de verbindingen weer inactief zijn en vervolgens af te sluiten. Als de opgegeven context verloopt voordat het afsluiten is voltooid, retourneert Shutdown de fout van de context. Anders retourneert het elke fout die wordt geretourneerd door het sluiten van de onderliggende listener(s) van de server.

Om `SIGTERM` af te handelen, kunnen we [os/signal.Notify](https://pkg.go.dev/os/signal#Notify) gebruiken, dat alle inkomende signalen naar een door ons aangeboden kanaal stuurt.

Door deze twee functies uit de standaardbibliotheek te gebruiken, kun je luisteren naar `SIGTERM` en netjes afsluiten.

## Graceful shutdown-pakket

Om die reden heb ik [https://pkg.go.dev/github.com/quii/go-graceful-shutdown](https://pkg.go.dev/github.com/quii/go-graceful-shutdown) geschreven. Het biedt een decoratorfunctie voor een `*http.Server` om de `Shutdown`-methode aan te roepen wanneer een `SIGTERM`-signaal wordt gedetecteerd.

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// this will typically happen if our responses aren't written before the ctx deadline, not much can be done
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// hopefully, you'll always see this instead
	log.Println("shutdown gracefully! all responses were sent")
}
```

De details rondom de code zijn niet zo belangrijk voor dit hoofdstuk, maar het is de moeite waard om de code kort te bekijken voordat je verdergaat.

## Tests en feedback loops

Toen we het `gracefulshutdown`-pakket schreven, voerden we unittests uit om te bewijzen dat het correct functioneerde, wat ons het vertrouwen gaf om agressief te refactoren. We hadden er echter nog steeds geen vertrouwen in dat het **echt** werkte.

We voegden een `cmd`-pakket toe en maakten een echt programma om het pakket dat we aan het schrijven waren te gebruiken. We startten het handmatig op, stuurden er een HTTP-verzoek naartoe en stuurden vervolgens een `SIGTERM` om te zien wat er zou gebeuren.

**De engineer in jou zou zich ongemakkelijk moeten voelen bij handmatig testen**.
Het is saai, het schaalt niet, het is onnauwkeurig en het is tijdverspilling. Als je een pakket schrijft dat je wilt delen, maar het ook eenvoudig en goedkoop wilt houden om te wijzigen, is handmatig testen niet voldoende.

## Acceptatietests

Als je de rest van dit boek hebt gelezen, zul je voornamelijk "unittests" hebben geschreven. Unittests zijn een fantastische tool om onbevreesd refactoren mogelijk te maken, een goed modulair ontwerp te stimuleren, regressies te voorkomen en snelle feedback te faciliteren.

Van nature testen ze slechts kleine onderdelen van je systeem. Meestal zijn unittests alleen *niet voldoende* voor een effectieve teststrategie. Vergeet niet dat we willen dat onze systemen **altijd leverbaar** zijn. We kunnen niet vertrouwen op handmatige tests, dus hebben we een ander soort testen nodig: **acceptatietests**.

### Wat zijn dat?

Acceptatietests zijn een soort "black-box-test". Ze worden soms ook wel "functionele tests" genoemd. Ze moeten het systeem testen zoals een gebruiker dat zou doen.

De term "black-box" verwijst naar het idee dat de testcode geen toegang heeft tot de interne onderdelen van het systeem; het kan alleen de openbare interface gebruiken en uitspraken doen over het gedrag dat het observeert. Dit betekent dat het systeem alleen als geheel getest kan worden.

Dit is een voordelige eigenschap, omdat het betekent dat de tests het systeem op dezelfde manier testen als een gebruiker; het kan geen speciale workarounds gebruiken die een test zouden kunnen laten slagen, maar niet daadwerkelijk bewijzen wat je moet bewijzen. Dit is vergelijkbaar met het principe om je unit-testbestanden bij voorkeur in een apart testpakket te plaatsen, bijvoorbeeld `package mypkg_test` in plaats van `package mypkg`.

### Voordelen van acceptatietests

- Als ze slagen, weet je dat je hele systeem zich gedraagt zoals jij dat wilt.
- Ze zijn nauwkeuriger, sneller en vergen minder inspanning dan handmatige tests.
- Als ze goed geschreven zijn, fungeren ze als nauwkeurige, geverifieerde documentatie van je systeem. Het trapt niet in de valkuil van documentatie die afwijkt van het werkelijke gedrag van het systeem.
- Geen gezeur! Het is allemaal echt.

### Mogelijke nadelen ten opzichte van unittests

- Ze zijn duur om te schrijven.
- Ze duren langer om uit te voeren.
- Ze zijn afhankelijk van het ontwerp van het systeem.
- Als ze falen, geven ze je meestal geen hoofdoorzaak en kunnen ze moeilijk te debuggen zijn.
- Ze geven je geen feedback over de interne kwaliteit van je systeem. Je zou totale troep kunnen schrijven en toch een acceptatietest kunnen laten slagen.
- Niet alle scenario's zijn praktisch uitvoerbaar vanwege het black-box-karakter.

Daarom is het onverstandig om alleen op acceptatietests te vertrouwen. Ze missen veel van de kwaliteiten van unittests, en een systeem met een groot aantal acceptatietests zal vaak te lijden hebben onder onderhoudskosten en slechtere doorlooptijd.

#### Doorlooptijd?

De doorlooptijd verwijst naar de tijd die nodig is om een commit te mergen in je hoofdbranch en deze te implementeren in productie. Deze tijd kan variëren van weken en zelfs maanden voor sommige teams tot een kwestie van minuten. Bij `$WORK` hechten we waarde aan de bevindingen van DORA en willen we onze doorlooptijd onder de 10 minuten houden.

Een evenwichtige testaanpak is vereist voor een betrouwbaar systeem met een uitstekende doorlooptijd, en dit wordt meestal beschreven in termen van de [Testpiramide](https://martinfowler.com/articles/practical-test-pyramid.html).

## Hoe schrijf je basisacceptatietests?

Wat is de relatie met het oorspronkelijke probleem? We hebben hier net een pakket geschreven en het is volledig unit-testbaar.

Zoals ik al zei, gaven de unit-tests ons niet helemaal het vertrouwen dat we nodig hadden. We willen *echt* zeker weten dat het pakket werkt wanneer het geïntegreerd is met een echt, lopend programma. We zouden de handmatige controles die we uitvoerden, moeten kunnen automatiseren.

Laten we eens kijken naar het testprogramma:

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// this will typically happen if our responses aren't written before the ctx deadline, not much can be done
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// hopefully, you'll always see this instead
	log.Println("shutdown gracefully! all responses were sent")
}
```

Je hebt misschien al geraden dat `SlowHandler` een `time.Sleep` heeft om de reactie uit te stellen, dus ik had tijd om `SIGTERM` te gebruiken en te kijken wat er gebeurt. De rest is vrij standaard:

- Maak een `net/http/Server`;
- Wikkel deze in de bibliotheek (zie: [Decorator patroon](https://en.wikipedia.org/wiki/Decorator_pattern));
- Gebruik de ingepakte versie om `ListenAndServe` uit te voeren.

### Stappen op hoog niveau voor de acceptatietest

- Bouw het programma
- Voer het uit (en wacht tot het luistert op `8080`)
- Stuur een HTTP-verzoek naar de server
- Stuur `SIGTERM` voordat de server een HTTP-antwoord kan sturen
- Kijk of we nog steeds een antwoord krijgen

### Het programma bouwen en uitvoeren

```go
package acceptancetests

import (
	"fmt"
	"math/rand"
	"net"
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
	"time"
)

const (
	baseBinName = "temp-testbinary"
)

func LaunchTestProgram(port string) (cleanup func(), sendInterrupt func() error, err error) {
	binName, err := buildBinary()
	if err != nil {
		return nil, nil, err
	}

	sendInterrupt, kill, err := runServer(binName, port)

	cleanup = func() {
		if kill != nil {
			kill()
		}
		os.Remove(binName)
	}

	if err != nil {
		cleanup() // even though it's not listening correctly, the program could still be running
		return nil, nil, err
	}

	return cleanup, sendInterrupt, nil
}

func buildBinary() (string, error) {
	binName := randomString(10) + "-" + baseBinName

	build := exec.Command("go", "build", "-o", binName)

	if err := build.Run(); err != nil {
		return "", fmt.Errorf("cannot build tool %s: %s", binName, err)
	}
	return binName, nil
}

func runServer(binName string, port string) (sendInterrupt func() error, kill func(), err error) {
	dir, err := os.Getwd()
	if err != nil {
		return nil, nil, err
	}

	cmdPath := filepath.Join(dir, binName)

	cmd := exec.Command(cmdPath)

	if err := cmd.Start(); err != nil {
		return nil, nil, fmt.Errorf("cannot run temp converter: %s", err)
	}

	kill = func() {
		_ = cmd.Process.Kill()
	}

	sendInterrupt = func() error {
		return cmd.Process.Signal(syscall.SIGTERM)
	}

	err = waitForServerListening(port)

	return
}

func waitForServerListening(port string) error {
	for i := 0; i < 30; i++ {
		conn, _ := net.Dial("tcp", net.JoinHostPort("localhost", port))
		if conn != nil {
			conn.Close()
			return nil
		}
		time.Sleep(100 * time.Millisecond)
	}
	return fmt.Errorf("nothing seems to be listening on localhost:%s", port)
}

func randomString(n int) string {
	var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")

	s := make([]rune, n)
	for i := range s {
		s[i] = letters[rand.Intn(len(letters))]
	}
	return string(s)
}
```
`LaunchTestProgram` is verantwoordelijk voor:
- het bouwen van het programma
- het starten van het programma
- het wachten tot het programma luistert op poort `8080`
- het bieden van een `cleanup`-functie om het programma te stoppen en te verwijderen, zodat we na afloop van onze tests een schone staat overhouden - het bieden van een `interrupt`-functie om het programma een `SIGTERM` te sturen, zodat we het gedrag kunnen testen.

Toegegeven, dit is niet de mooiste code ter wereld, maar concentreer je gewoon op de geëxporteerde functie `LaunchTestProgram`; de niet-geëxporteerde functies die het aanroept, zijn oninteressante boilerplate-codes.

Zoals besproken, zijn acceptatietests vaak lastiger op te zetten. Deze code maakt de *test*-code aanzienlijk eenvoudiger te lezen, en vaak is het bij acceptatietests zo dat, zodra je de ceremoniële code hebt geschreven, deze klaar zijn en je er niet meer naar hoeft om te kijken.

### De acceptatietest(s)

We wilden twee acceptatietests voor twee programma's, één met een soepele afsluiting en één zonder, zodat wij en de lezers het verschil in gedrag kunnen zien. Met `LaunchTestProgram` om de programma's te bouwen en uit te voeren, is het vrij eenvoudig om acceptatietests voor beide te schrijven, en we profiteren van hergebruik met enkele hulpfuncties.

Hier is de test voor de server *met* een soepele afsluiting, [je kunt de test zonder vinden op GitHub](https://github.com/quii/go-graceful-shutdown/blob/main/acceptancetests/withoutgracefulshutdown/main_test.go)

```go
package main

import (
	"testing"
	"time"

	"github.com/quii/go-graceful-shutdown/acceptancetests"
	"github.com/quii/go-graceful-shutdown/assert"
)

const (
	port = "8080"
	url  = "<http://localhost:" + port
)

func TestGracefulShutdown(t *testing.T) {
	cleanup, sendInterrupt, err := acceptancetests.LaunchTestProgram(port)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(cleanup)

	// just check the server works before we shut things down
	assert.CanGet(t, url)

	// fire off a request, and before it has a chance to respond send SIGTERM.
	time.AfterFunc(50*time.Millisecond, func() {
		assert.NoError(t, sendInterrupt())
	})
	// Without graceful shutdown, this would fail
	assert.CanGet(t, url)

	// after interrupt, the server should be shutdown, and no more requests will work
	assert.CantGet(t, url)
}
```

Nu de setup is ingekapseld, zijn de tests uitgebreid, beschrijven ze het gedrag en zijn ze relatief eenvoudig te volgen.

`assert.CanGet/CantGet` zijn hulpfuncties die ik heb gemaakt om deze veelvoorkomende bewering voor deze suite op te lossen.

```go
func CanGet(t testing.TB, url string) {
	errChan := make(chan error)

	go func() {
		res, err := http.Get(url)
		if err != nil {
			errChan <- err
			return
		}
		res.Body.Close()
		errChan <- nil
	}()

	select {
	case err := <-errChan:
		NoError(t, err)
	case <-time.After(3 * time.Second):
		t.Errorf("timed out waiting for request to %q", url)
	}
}
```

Dit activeert een `GET` naar `URL` op een goroutine, en als deze binnen 3 seconden zonder fout reageert, mislukt de code niet. `CantGet` is weggelaten vanwege de beknoptheid, [maar je kunt het hier op GitHub bekijken](https://github.com/quii/go-graceful-shutdown/blob/main/assert/assert.go#L61).

Het is belangrijk om nogmaals te vermelden dat Go alle tools heeft die je nodig hebt om acceptatietests direct te schrijven. Je hebt geen speciaal framework *nodig* om acceptatietests te bouwen.

### Kleine investering met een groot rendement

Met deze tests kunnen lezers de voorbeeldprogramma's bekijken en er zeker van zijn dat het voorbeeld *echt* werkt, zodat ze vertrouwen kunnen hebben in de claims van het pakket.

Belangrijk is dat we als auteur **snelle feedback** en **enorme zekerheid** krijgen dat het pakket in de praktijk werkt.

```shell
go test -count=1 ./...
ok  	github.com/quii/go-graceful-shutdown	0.196s
?   	github.com/quii/go-graceful-shutdown/acceptancetests	[no test files]
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withgracefulshutdown	4.785s
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withoutgracefulshutdown	2.914s
?   	github.com/quii/go-graceful-shutdown/assert	[no test files]
```

## Samenvattend

In dit hoofdstuk hebben we acceptatietests geïntroduceerd in je testgereedschapskist. Ze zijn van onschatbare waarde wanneer je echte systemen gaat bouwen en vormen een belangrijke aanvulling op je unittests.

De aard van *hoe* je acceptatietests schrijft, hangt af van het systeem dat je bouwt, maar de principes blijven hetzelfde. Behandel je systeem als een "black box". Als je een website maakt, moeten je tests zich gedragen als een gebruiker. Gebruik daarom een headless webbrowser zoals [Selenium](https://www.selenium.dev/) om op links te klikken, formulieren in te vullen, enz. Voor een RESTful API verstuur je HTTP-verzoeken via een client.

### Verdergaan voor complexere systemen

Niet-triviale systemen zijn meestal geen applicaties met één proces zoals het systeem dat we hebben besproken. Meestal ben je afhankelijk van andere systemen, zoals een database. Voor deze scenario's moet je een lokale omgeving automatiseren om mee te testen. Tools zoals [docker-compose](https://docs.docker.com/compose/) zijn handig voor het opstarten van containers van de omgeving die je nodig hebt om je systeem lokaal te laten draaien.

### Het volgende hoofdstuk

In dit hoofdstuk is de acceptatietest retrospectief geschreven. In [Growing Object-Oriented Software](http://www.growing-object-oriented-software.com) laten de auteurs echter zien dat we acceptatietests in een testgedreven aanpak kunnen gebruiken als een "poolster" om onze inspanningen te sturen.

Naarmate systemen complexer worden, kunnen de kosten van het schrijven en onderhouden van acceptatietests snel uit de hand lopen. Er zijn talloze verhalen over ontwikkelteams die gehinderd worden door dure acceptatietestsuites.

Het volgende hoofdstuk introduceert het gebruik van acceptatietests als leidraad voor ons ontwerp, samen met principes en technieken voor het beheersen van de kosten van acceptatietests.

### De kwaliteit van open source verbeteren

Als je pakketten schrijft die je wilt delen, raad ik je aan om eenvoudige voorbeeldprogramma's te maken die laten zien wat je pakket doet en tijd te investeren in eenvoudig te volgen acceptatietests om jezelf en potentiële gebruikers van je werk vertrouwen te geven.

Net als bij [Testable Examples](https://go.dev/blog/examples) draagt deze kleine extra inspanning in de ontwikkelaarservaring enorm bij aan het opbouwen van vertrouwen in je werk en verlaagt het je eigen onderhoudskosten.

## Wervingsadvertentie voor `$WORK`

Als je zin hebt om samen met andere engineers interessante problemen op te lossen, in de buurt van Londen of Porto woont en de inhoud van dit hoofdstuk en boek interessant vindt, neem dan [contact met me op via Twitter](https://twitter.com/quii) en misschien kunnen we binnenkort samenwerken!
