# Dependency Injection

[Je kunt hier alle code van dit hoofdstuk vinden](https://github.com/quii/learn-go-with-tests/tree/main/di)

Ik ga ervan uit dat je het gedeelte over [structs](structs-methods-and-interfaces.md) al eerder hebt gelezen, omdat enige kennis van interfaces voor dit hoofdstuk nodig is.

Er bestaan ​​veel misverstanden over dependency injection binnen de programmeergemeenschap. Hopelijk laat deze gids je zien hoe

* Je hebt geen framework nodig
* Het je ontwerp niet onnodig ingewikkeld maakt
* je faciliteert bij het testen
* je hiermee geweldige, algemene functies kunt schrijven.

We willen een functie schrijven die iemand begroet, net zoals we deden in het hoofdstuk over de Hello World. Deze keer gaan we echter het daadwerkelijke _tonen op het scherm_ testen.

Om het nog even samen te vatten: dit is hoe die functie eruit zou kunnen zien

```go
func Greet(name string) {
	fmt.Printf("Hello, %s", name)
}
```

Maar hoe kunnen we dit testen? Door `fmt.Printf` aan te roepen, printen we naar stdout, wat voor ons vrij lastig is om vast te leggen met het testframework.

Wat we moeten doen is de afhankelijkheid van het printen **injecteren** (een mooi woord voor doorgeven).

**Voor onze functie maakt het niet uit waar of hoe het afdrukken plaatsvindt. Daarom moeten we een interface accepteren in plaats van een concrete type.**

Als we dat doen, kunnen we de implementatie aanpassen om te printen naar iets dat we zelf beheren, zodat we het kunnen testen. In de "echte" praktijk zou je iets injecteren dat naar stdout schrijft.

Als je naar de broncode van [`fmt.Printf`](https://pkg.go.dev/fmt#Printf) kijkt, zie je een manier waarop we kunnen aansluiten

```go
// It returns the number of bytes written and any write error encountered.
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
```

Interessant! Onder motorkap roept `Printf` gewoon de `Fprintf` functie aan en geeft het `os.Stdout` door.

Wat is _precies_ een `os.Stdout`? Wat verwacht `Fprintf` dat er als eerste argument aan wordt doorgegeven?

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

Een `io.Writer`

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

Hieruit kunnen we afleiden dat `os.Stdout` `io.Writer` implementeert; `Printf` geeft `os.Stdout` door aan `Fprintf`, die een `io.Writer` verwacht.

Naarmate je meer Go-code schrijft, zul je merken dat deze interface steeds vaker opduikt. Het is namelijk een geweldige, algemene interface om "gegevens ergens neer te zetten".

We weten dus dat we uiteindelijk `Writer` gebruiken om onze groet ergens naartoe te sturen. Laten we deze bestaande abstractie gebruiken om onze code testbaar en herbruikbaarder te maken.

## Schrijf eerst je test

```go
func TestGreet(t *testing.T) {
	buffer := bytes.Buffer{}
	Greet(&buffer, "Chris")

	got := buffer.String()
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Het type `Buffer` uit het `bytes`-pakket implementeert de `Writer`-interface, omdat het de methode `Write(p []byte) (n int, err error)` heeft.

We zullen het dus gebruiken in onze test om het als onze `Writer` in te sturen en dan kunnen we controleren wat erin is geschreven nadat we `Greet` hebben aangeroepen

## Probeer de test uit te voeren

De test zal niet compileren

```
./di_test.go:10:2: undefined: Greet
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

_Luister naar de compiler_ en fix het probleem.

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Printf("Hello, %s", name)
}
```

`Hello, Chris di_test.go:16: got '' want 'Hello, Chris'`

De test mislukt. Merk op dat de naam wel wordt afgedrukt, maar op stdout staat.

## Schrijf genoeg code om de test te laten slagen

Gebruik de writer om de begroeting naar de buffer te sturen in onze test. Onthoud dat `fmt.Fprintf` hetzelfde is als `fmt.Printf`, maar een `Writer` nodig heeft om de string naartoe te sturen, terwijl `fmt.Printf` standaard stdout gebruikt.

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}
```

De test slaagt nu.

## Refactor

Eerder vertelde de compiler ons dat we een pointer naar een `bytes.Buffer` moesten meegeven. Dit is technisch correct, maar niet erg nuttig.

Om dit te demonstreren, kun je proberen de `Greet`-functie aan te sluiten op een Go-toepassing en deze vervolgens op stdout af te drukken.

```go
func main() {
	Greet(os.Stdout, "Elodie")
}
```

`./di.go:14:7: cannot use os.Stdout (type *os.File) as type *bytes.Buffer in argument to Greet`

Zoals eerder besproken, kun je met `fmt.Fprintf` een `io.Writer` doorgeven, waarvan we weten dat zowel `os.Stdout` als `bytes.Buffer` deze implementeren.

Als we onze code aanpassen om de meer algemene interface te gebruiken, kunnen we deze nu zowel in tests als in onze applicatie gebruiken.

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func Greet(writer io.Writer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}

func main() {
	Greet(os.Stdout, "Elodie")
}
```

## Meer over io.Writer

Naar welke andere plekken kunnen we gegevens schrijven met `io.Writer`? Hoe algemeen is onze Greet-functie?

### Het Internet

Voer de volgende code uit:

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
)

func Greet(writer io.Writer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}

func MyGreeterHandler(w http.ResponseWriter, r *http.Request) {
	Greet(w, "world")
}

func main() {
	log.Fatal(http.ListenAndServe(":5001", http.HandlerFunc(MyGreeterHandler)))
}
```

Start het programma en ga naar [http://localhost:5001](http://localhost:5001/). Je zult zien dat je begroetingsfunctie wordt gebruikt.

HTTP-servers worden in een later hoofdstuk besproken, dus maak je niet te druk over de details.

Wanneer je een HTTP-handler schrijft, krijg je een `http.ResponseWriter` en de `http.Request` die is gebruikt om de aanvraag te doen. Wanneer je je server implementeert, _schrijf_ je je antwoord met behulp van de writer.

Je kunt waarschijnlijk wel raden dat `http.ResponseWriter` ook `io.Writer` implementeert en daarom kunnen we onze `Greet`-functie hergebruiken in onze handler.

## Samenvattend

Onze eerste coderonde was niet eenvoudig te testen, omdat de gegevens naar een plek werden geschreven waar we geen controle over hadden.

_Geïnspireerd door onze tests_ hebben we de code aangepast, zodat we konden bepalen waar de data naartoe werd geschreven door door gebruik te maken van **dependency injection**. Hierdoor konden we:

* **Onze code testen**. Als je een functie niet _gemakkelijk_ kunt testen, komt dat meestal door afhankelijkheden die vastliggen in een functie of globale status. Als je bijvoorbeeld een globale databaseverbindingspool hebt die wordt gebruikt door een servicelaag, zal het testen waarschijnlijk moeilijk zijn en zullen de uitvoeringen traag zijn. DI zal je motiveren om een ​​databaseafhankelijkheid te injecteren (via een interface), die je vervolgens kunt mocken met iets dat je in je tests kunt beheren.
* **Scheid onze zorgen** en ontkoppel _waar de data naartoe gaat_ van _hoe deze gegenereerd wordt_. Als je ooit het gevoel hebt dat een methode/functie te veel verantwoordelijkheden heeft (data genereren en naar een database schrijven? HTTP-verzoeken verwerken en logica op domeinniveau uitvoeren?) dan is DI waarschijnlijk de tool die je nodig hebt.
* **Sta toe dat onze code in verschillende contexten hergebruikt kan worden**. De eerste "nieuwe" context waarin onze code gebruikt kan worden, is binnen tests. Maar als iemand later iets nieuws met je functie wil proberen, kan hij of zij zijn of haar eigen afhankelijkheden injecteren.

### Hoe zit het met mocken? Ik hoor dat je dat nodig hebt voor DI en het is ook nog eens slecht.

Mocking wordt later uitgebreid behandeld (en het is niet slecht). Je gebruikt mocking om echte dingen die je injecteert te vervangen door een fictieve versie die je kunt controleren en inspecteren in je tests. In ons geval had de standaardbibliotheek echter iets kant-en-klaars voor ons.

### De standaardbibliotheek van Go is echt goed, neem de tijd om deze te bestuderen

Omdat we enigszins vertrouwd zijn met de `io.Writer`-interface, kunnen we `bytes.Buffer` in onze test gebruiken als onze `Writer`. Vervolgens kunnen we andere `Writers` uit de standaardbibliotheek gebruiken om onze functie in een command-line toepassing of op een webserver te gebruiken.

Naarmate je meer vertrouwd bent met de standaardbibliotheek, zul je vaker algemene interfaces tegenkomen. Je kunt deze vervolgens opnieuw gebruiken in je eigen code, zodat je software in diverse contexten hergebruikt kan worden.

Dit voorbeeld is sterk beïnvloed door een hoofdstuk uit het boek [The Go Programming language](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440). Als je dit leuk vond, koop het dan!
