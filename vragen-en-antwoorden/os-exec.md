# OS Exec

[**Je kunt alle code voor dit hoofdstuk hier vinden**](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/os-exec)

[keith6014](https://www.reddit.com/user/keith6014) vraagt op [reddit](https://www.reddit.com/r/golang/comments/aaz8ji/testdata_and_function_setup_help/)

> Ik voer een opdracht uit met os/exec.Command() die XML-gegevens genereert. De opdracht wordt uitgevoerd in een functie genaamd GetData().

> Om GetData() te testen, heb ik wat testgegevens aangemaakt.

> In mijn \_test.go heb ik een TestGetData die GetData() aanroept, maar die os.exec gebruikt. Ik wil in plaats daarvan mijn testgegevens gebruiken.

> Wat is een goede manier om dit te bereiken? Moet ik bij het aanroepen van GetData een "test"-vlagmodus gebruiken, zodat het een bestand leest, bijvoorbeeld GetData(mode string)?

Een paar dingen

* Als iets moeilijk te testen is, komt dat vaak doordat de scheiding van aandachtspunten niet helemaal klopt.
* Voeg geen "testmodi" toe aan je code, maar gebruik in plaats daarvan [Dependency Injection](../basisbeginselen-go/dependency-injection.md) zodat je je afhankelijkheden kunt modelleren en aandachtspunten kunt scheiden.

Ik heb de vrijheid genomen om te raden hoe de code eruit zou kunnen zien.

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData() string {
	cmd := exec.Command("cat", "msg.xml")

	out, _ := cmd.StdoutPipe()
	var payload Payload
	decoder := xml.NewDecoder(out)

	// these 3 can return errors but I'm ignoring for brevity
	cmd.Start()
	decoder.Decode(&payload)
	cmd.Wait()

	return strings.ToUpper(payload.Message)
}
```

* Het gebruikt `exec.Command`, waarmee je een externe opdracht aan het proces kunt uitvoeren.
* We vangen de uitvoer op in `cmd.StdoutPipe`, wat ons een `io.ReadCloser` retourneert (dit wordt belangrijk).
* De rest van de code is min of meer gekopieerd en geplakt uit de [uitstekende documentatie](https://golang.org/pkg/os/exec/#example_Cmd_StdoutPipe).
* We vangen alle uitvoer van stdout op in een `io.ReadCloser`, waarna we de opdracht `Starten` en wachten tot alle gegevens zijn gelezen door `Wait` aan te roepen. Tussen deze twee aanroepen decoderen we de gegevens in onze `Payload`-struct.

Dit is wat er in `msg.xml` staat.

```xml
<payload>
    <message>Happy New Year!</message>
</payload>
```

Ik heb een eenvoudige test geschreven om het in de praktijk te laten zien

```go
func TestGetData(t *testing.T) {
	got := GetData()
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

## Testbare code

Testbare code is ontkoppeld en heeft één doel. Het voelt alsof er twee hoofddoelen zijn voor deze code:

1. Het ophalen van de ruwe XML-gegevens
2. Het decoderen van de XML-gegevens en het toepassen van onze bedrijfslogica (in dit geval `strings.ToUpper` op `<bericht>`)

Het eerste deel is gewoon het kopiëren van het voorbeeld uit de standaardbibliotheek.

Het tweede deel is waar onze bedrijfslogica staat en door naar de code te kijken, kunnen we zien waar de "naad" in onze logica begint; daar krijgen we onze `io.ReadCloser`. We kunnen deze bestaande abstractie gebruiken om aandachtspunten te scheiden en onze code testbaar te maken.

**Het probleem met GetData is dat de bedrijfslogica gekoppeld is aan de manier waarop de XML wordt opgehaald. Om ons ontwerp te verbeteren, moeten we ze ontkoppelen**

Onze `TestGetData` kan dienen als onze integratietest tussen onze twee aandachtspunten, dus we zullen die bewaren om ervoor te zorgen dat hij blijft werken.

Dit is hoe de nieuw gescheiden code eruit ziet

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData(data io.Reader) string {
	var payload Payload
	xml.NewDecoder(data).Decode(&payload)
	return strings.ToUpper(payload.Message)
}

func getXMLFromCommand() io.Reader {
	cmd := exec.Command("cat", "msg.xml")
	out, _ := cmd.StdoutPipe()

	cmd.Start()
	data, _ := io.ReadAll(out)
	cmd.Wait()

	return bytes.NewReader(data)
}

func TestGetDataIntegration(t *testing.T) {
	got := GetData(getXMLFromCommand())
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Nu `GetData` zijn invoer alleen van een `io.Reader` haalt, hebben we het testbaar gemaakt en maakt het zich niet langer druk om hoe de gegevens worden opgehaald; mensen kunnen de functie hergebruiken met alles wat een `io.Reader` retourneert (wat heel gebruikelijk is). We zouden bijvoorbeeld de XML kunnen ophalen via een URL in plaats van de opdrachtregel.

```go
func TestGetData(t *testing.T) {
	input := strings.NewReader(`
<payload>
    <message>Cats are the best animal</message>
</payload>`)

	got := GetData(input)
	want := "CATS ARE THE BEST ANIMAL"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

```

Hier is een voorbeeld van een unittest voor `GetData`.

Door de aandachtspunten te scheiden en bestaande abstracties binnen Go te gebruiken, is onze belangrijke bedrijfslogica een fluitje van een cent.
