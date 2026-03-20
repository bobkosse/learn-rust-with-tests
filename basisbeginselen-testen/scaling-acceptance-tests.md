# Leer Go met Tests - Acceptatietests opschalen (en een korte introductie tot gRPC)

Dit hoofdstuk is een vervolg op [Inleiding tot acceptatietests](https://bobkosse.gitbook.io/leer-go-met-tests/basisbeginselen-testen/intro-to-acceptance-tests). Je kunt [de voltooide code voor dit hoofdstuk vinden op GitHub](https://github.com/quii/go-specs-greet).

Acceptatietests zijn essentieel en hebben een directe invloed op je vermogen om je systeem in de loop der tijd met vertrouwen te ontwikkelen, tegen redelijke kosten.

Ze vormen ook een fantastische tool om te werken met legacy code. Wanneer je te maken hebt met een slechte codebase zonder tests, weersta dan de verleiding om te beginnen met refactoren. Schrijf in plaats daarvan een aantal acceptatietests om je een vangnet te bieden om de interne werking van het systeem vrijelijk te kunnen wijzigen zonder het functionele externe gedrag te beïnvloeden. Acceptatietests hoeven zich geen zorgen te maken over de interne kwaliteit, dus ze zijn in deze situaties een uitstekende keuze.

Nadat je dit hebt gelezen, zul je begrijpen dat acceptatietests nuttig zijn voor verificatie en dat ze ook kunnen worden gebruikt in het ontwikkelingsproces. Ze helpen ons om ons systeem doelbewuster en methodischer te veranderen, waardoor er minder moeite wordt verspild.

## Vereist materiaal

De inspiratie voor dit hoofdstuk is voortgekomen uit jarenlange frustratie met acceptatietests. Twee video's die ik je aanraad om te bekijken zijn:

- Dave Farley - [Hoe schrijf je acceptatietests](https://www.youtube.com/watch?v=JDD5EEJgpHU)
- Nat Pryce - [E2E functionele tests die in milliseconden kunnen worden uitgevoerd](https://www.youtube.com/watch?v=Fk4rCn4YLLU)

"Growing Object Oriented Software" (GOOS) is een belangrijk boek voor veel software engineers, waaronder ikzelf. De aanpak die het voorschrijft, is de aanpak die ik de engineers met wie ik werk, aanraad te volgen.

- [GOOS](http://www.growing-object-oriented-software.com) - Nat Pryce & Steve Freeman

Ten slotte spraken [Riya Dattani](https://twitter.com/dattaniriya) en ik over dit onderwerp in de context van BDD in onze lezing [Acceptatietests, BDD en Go](https://www.youtube.com/watch?v=ZMWJCk_0WrY).

## Samenvatting

We hebben het over "black-box"-tests die controleren of je systeem zich van buitenaf gedraagt zoals verwacht, vanuit een "**zakelijk perspectief**". De tests hebben geen toegang tot de interne onderdelen van het systeem dat ze testen; ze zijn alleen geïnteresseerd in **wat** je systeem doet in plaats van **hoe**.

## Anatomie van slechte acceptatietests

Ik heb jarenlang voor verschillende bedrijven en teams gewerkt. Elk van hen erkende de noodzaak van acceptatietests; een manier om een systeem vanuit het perspectief van de gebruiker te testen en te verifiëren dat het werkt zoals bedoeld. Maar bijna zonder uitzondering vormden de kosten van deze tests een echt probleem voor het team.

- Langzaam draaiend
- Broos
- Onstabiel
- Duur in onderhoud en het lijkt erop dat het lastiger is om de software te wijzigen dan nodig is
- Kan alleen draaien in een specifieke omgeving, wat zorgt voor trage en slechte feedbackloops

Stel dat je van plan bent een acceptatietest te schrijven voor een website die je bouwt. Je besluit een headless webbrowser (zoals [Selenium](https://www.selenium.dev)) te gebruiken om te simuleren dat een gebruiker op knoppen op je website klikt om te controleren of deze doet wat hij moet doen.

Na verloop van tijd moet de opmaak van je website veranderen naarmate er nieuwe functies worden ontdekt, en technici discussiëren voor de miljardste keer over de vraag of iets een `<artikel>` of een `<sectie>` moet zijn.

Hoewel je team slechts kleine wijzigingen in het systeem aanbrengt, die nauwelijks merkbaar zijn voor de daadwerkelijke gebruiker, verspil je toch veel tijd aan het bijwerken van je acceptatietests.

### Tight-coupling

Denk na over wat de aanleiding is voor acceptatietests om te veranderen:

- Een externe gedragsverandering. Als je wilt veranderen wat het systeem doet, lijkt het aanpassen van de acceptatietestsuite redelijk, zo niet wenselijk.
- Een wijziging in de implementatiedetails/refactoring. Idealiter zou dit geen verandering moeten veroorzaken, of als dat wel gebeurt, een kleine.

Maar al te vaak is dit laatste de reden dat acceptatietests moeten veranderen. Zo erg zelfs dat engineers aarzelen om hun systeem te veranderen vanwege de ervaren inspanning die het updaten van tests met zich meebrengt!

![Riya en ik praten over het scheiden van aandachtspunten in onze tests](https://i.imgur.com/bbG6z57.png)

Deze problemen komen voort uit het niet toepassen van gevestigde en beoefende engineeringgewoonten die door de bovengenoemde auteurs zijn geschreven. **Je kunt acceptatietests niet schrijven zoals unittests**; ze vereisen meer denkwerk en andere werkwijzen.

## Anatomie van goede acceptatietests

Als we acceptatietests willen die alleen veranderen wanneer we het gedrag veranderen en niet de implementatiedetails, dan is het logisch dat we die aandachtspunten moeten scheiden.

### Over soorten complexiteit

Als software engineers hebben we te maken met twee soorten complexiteit.

- **Accidental complexity** is de complexiteit waarmee we te maken hebben omdat we werken met computers, zaken als netwerken, schijven, API's, enz.

- **Essentiële complexiteit** wordt soms ook wel "domeinlogica" genoemd. Het zijn de specifieke regels en waarheden binnen je domein.
    - Bijvoorbeeld: "als een rekeninghouder meer geld opneemt dan beschikbaar is, is er sprake van een roodstand". Deze bewering zegt niets over computers; deze bewering was al waar voordat computers überhaupt in banken werden gebruikt!

Essentiële complexiteit zou voor een niet-technisch persoon begrijpelijk moeten zijn, en het is waardevol om deze te modelleren in onze "domein"-code en in onze acceptatietests.

### Scheiding van aandachtspunten

Wat Dave Farley eerder in de video voorstelde, en wat Riya en ik ook bespraken, is dat we het idee van **specificaties** zouden moeten hebben. Specificaties beschrijven het gedrag van het systeem dat we willen, zonder dat ze gepaard gaan met onbedoelde complexiteit of implementatiedetails.

Dit idee zou aannemelijk voor je moeten zijn. In productiecode streven we er vaak naar om aandachtspunten te scheiden en werkeenheden te ontkoppelen. Zou je niet aarzelen om een `interface` te introduceren zodat je `HTTP`-handler deze kan ontkoppelen van niet-HTTP-aandachtspunten? Laten we dezelfde denkwijze volgen voor onze acceptatietests.

Dave Farley beschrijft een specifieke structuur.

![Dave Farley over acceptatietests](https://i.imgur.com/nPwpihG.png)

Op GopherconUK hebben Riya en ik dit in Go-termen vertaald.

![Scheiding van aandachtspunten](https://i.imgur.com/qdY4RJe.png)

### Testen op steroïden

Door de uitvoering van de specificatie te ontkoppelen, kunnen we deze in verschillende scenario's hergebruiken. We kunnen:

#### Onze drivers configureerbaar maken

Dit betekent dat je je acceptatietests lokaal, in je staging- en (idealiter) productieomgevingen kunt uitvoeren.
- Te veel teams ontwerpen hun systemen zo dat acceptatietests onmogelijk lokaal kunnen worden uitgevoerd. Dit introduceert een ondraaglijk trage feedbacklus. Zou je er niet liever zeker van zijn dat je acceptatietests slagen _voordat_ je je code integreert? Als de tests mislukken, is het dan acceptabel dat je de fout niet lokaal kunt reproduceren en in plaats daarvan wijzigingen moet committen en hopen dat het 20 minuten later in een andere omgeving wel lukt?
- Onthoud dat het feit dat je tests in staging slagen, niet betekent dat je systeem ook werkt. Dev/Prod-pariteit is op zijn best een leugentje om bestwil. [Ik test in prod](https://increment.com/testing/i-test-in-production/).
- Er zijn altijd verschillen tussen de omgevingen die het *gedrag* van je systeem kunnen beïnvloeden. Een CDN kan cacheheaders onjuist hebben ingesteld; Een downstream service waarvan je afhankelijk bent, kan zich anders gedragen; een configuratiewaarde kan onjuist zijn. Maar zou het niet handig zijn als je je specificaties in productie kon uitvoeren om deze problemen snel op te sporen?

#### Plug _verschillende_ drivers in om andere delen van je systeem te testen

Deze flexibiliteit stelt ons in staat om gedragingen op verschillende abstractie- en architectuurlagen te testen, waardoor we gerichtere tests kunnen uitvoeren dan alleen black-box-tests.
- Je kunt bijvoorbeeld een webpagina hebben met een API erachter. Waarom zou je dan niet dezelfde specificatie gebruiken om beide te testen? Je kunt een headless webbrowser gebruiken voor de webpagina en HTTP-aanroepen voor de API.
- Als we dit idee verder uitwerken, willen we idealiter dat de **code essentiële complexiteit modelleert** (als "domein"-code), zodat we onze specificaties ook voor unittests kunnen gebruiken. Dit geeft snelle feedback dat de essentiële complexiteit in ons systeem is gemodelleerd en correct functioneert.

### Acceptatietests veranderen om de juiste redenen

Met deze aanpak hoeven je specificaties alleen te veranderen als het gedrag van het systeem verandert, wat logisch is.

- Als je HTTP API moet veranderen, is er één voor de hand liggende plek om deze bij te werken: de driver.
- Als je markup verandert, werk dan ook de specifieke driver bij.

Naarmate je systeem groeit, zul je merken dat je drivers voor meerdere tests hergebruikt. Dit betekent wederom dat als implementatiedetails veranderen, je slechts één, meestal voor de hand liggende, plek hoeft bij te werken.

Als deze aanpak goed wordt uitgevoerd, biedt deze ons flexibiliteit in onze implementatiedetails en stabiliteit in onze specificaties. Belangrijk is dat het een eenvoudige en duidelijke structuur biedt voor het beheren van wijzigingen, wat essentieel wordt naarmate een systeem en het bijbehorende team groeien.

### Acceptatietests als methode voor softwareontwikkeling

Tijdens ons gesprek bespraken Riya en ik acceptatietests en hun relatie met BDD. We bespraken hoe je je werk kunt beginnen met het proberen _het probleem dat je probeert op te lossen te begrijpen_ en dit te verwoorden in een specificatie, je helpt je intentie te focussen en een geweldige manier is om je werk te starten.

Ik maakte voor het eerst kennis met deze manier van werken in GOOS. Een tijdje geleden heb ik de ideeën samengevat op mijn blog. Hier is een fragment uit mijn bericht [Waarom TDD](https://quii.dev/The_Why_of_TDD)

---

TDD is erop gericht je iteratief te laten ontwerpen voor het gedrag dat je precies nodig hebt. Wanneer je een nieuw gebied start, moet je het belangrijkste, noodzakelijk gedrag identificeren en de scope agressief beperken.

Volg een top-downbenadering, beginnend met een acceptatietest (AT) die het gedrag van buitenaf test. Dit zal dienen als een leidraad voor je inspanningen. Het enige waar je je op moet richten, is ervoor zorgen dat die test slaagt. Deze test zal waarschijnlijk een tijdje falen terwijl je voldoende code ontwikkelt om hem te laten slagen.

![](https://i.imgur.com/pxTaYu4.png)

Zodra je AT is ingesteld, kun je het TDD-proces starten om voldoende units te genereren voor de AT-pass. De truc is om je op dit punt niet te veel zorgen te maken over het ontwerp; zorg dat je voldoende code hebt om de AT te laten slagen, omdat je nog steeds bezig bent met het leren en verkennen van het probleem.

Het zetten van deze eerste stap is vaak omvangrijker dan je denkt, met het opzetten van webservers, routing, configuratie, enz. Daarom is het essentieel om de scope van het werk beperkt te houden. We willen die eerste positieve stap op ons lege canvas zetten en deze laten ondersteunen door een AT die slaagt, zodat we snel en veilig kunnen blijven itereren.

![](https://i.imgur.com/t5y5opw.png)

Luister tijdens je ontwikkeling naar je tests. Ze zouden je signalen moeten geven die je helpen je ontwerp in een betere richting te sturen, maar wederom gebaseerd op het gedrag in plaats van op onze verbeelding.

Je eerste "eenheid" die het zware werk doet om de AT te laten slagen, zal doorgaans te groot worden om comfortabel te zijn, zelfs voor dit kleine beetje gedrag. Dit is het moment waarop je kunt gaan nadenken over hoe je het probleem kunt opsplitsen en nieuwe samenwerkingspartners kunt introduceren.

![](https://i.imgur.com/UYqd7Cq.png)

Hierbij zijn testdubbels (bijvoorbeeld fakes, mocks) handig, omdat de meeste complexiteit die intern in software leeft doorgaans niet in de implementatiedetails zit, maar 'tussen' de eenheden en de manier waarop ze met elkaar interacteren.

#### De gevaren van bottom-up

Dit is een "top-down"-benadering in plaats van een "bottom-up". Bottom-up heeft zijn nut, maar brengt ook een risico met zich mee. Door "services" en code te bouwen zonder dat deze snel in je applicatie worden geïntegreerd en zonder een high-level test te verifiëren, **loop je het risico veel moeite te verspillen aan niet-gevalideerde ideeën**.

Dit is een cruciale eigenschap van de acceptatietestgestuurde aanpak, waarbij tests worden gebruikt om onze code daadwerkelijk te valideren.

Ik ben te vaak engineers tegengekomen die een stuk code, geïsoleerd en bottom-up, hebben gemaakt waarvan ze denken dat het een probleem oplost, maar het:

- Werkt niet zoals we willen
- Doet dingen die we niet nodig hebben
- Is niet gemakkelijk te integreren
- Vereist sowieso veel herschrijven

Dat is verspilling.

## Genoeg gepraat, tijd om te coderen

In tegenstelling tot andere hoofdstukken moet je [Docker](https://www.docker.com) geïnstalleerd hebben, omdat we onze applicaties in containers zullen draaien. We gaan ervan uit dat je op dit punt in het boek vertrouwd bent met het schrijven van Go-code, het importeren vanuit verschillende pakketten, enz.

Maak een nieuw project aan met `go mod init github.com/quii/go-specs-greet` (je kunt hier plaatsen wat je wilt, maar als je het pad wijzigt, moet je alle interne imports aanpassen).

Maak een map `specifications` aan voor onze specificaties en voeg een bestand `greet.go` toe.

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet() (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet()
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, world")
}
```

Mijn IDE (Goland) zorgt voor het toevoegen van afhankelijkheden, maar als je het handmatig moet doen, moet je het volgende in de terminal invoeren:

`go get github.com/alecthomas/assert/v2`

Gegeven Farley's acceptatietestontwerp (Specificatie->Domain Specific Language(DSL)->Stuurprogramma->Systeem) hebben we nu een specificatie die losgekoppeld is van de implementatie. Het weet niet _hoe_ we 'Greet'-en en maakt zich er ook niet druk om; het houdt zich alleen bezig met de essentiële complexiteit van ons domein. Toegegeven, deze complexiteit is nu nog niet zo groot, maar we zullen de specificatie uitbreiden om meer functionaliteit toe te voegen naarmate we verder itereren. Het is altijd belangrijk om klein te beginnen!

Je zou de interface kunnen zien als onze eerste stap in een DSL; naarmate het project groeit, kun je de behoefte voelen om anders te abstraheren, maar voor nu is dit prima.

Op dit punt zou deze mate van ceremonie om onze specificatie los te koppelen van de implementatie sommigen ertoe kunnen aanzetten ons te beschuldigen van "overmatig abstraheren". **Ik beloof je dat acceptatietests die te veel gekoppeld zijn aan de implementatie een echte last worden voor engineeringteams**. Ik ben ervan overtuigd dat de meeste acceptatietests die in de praktijk worden uitgevoerd, duur zijn om te onderhouden vanwege deze ongepaste koppeling; in plaats van het omgekeerde van overmatig abstract zijn.

We kunnen deze specificatie gebruiken om elk 'systeem' te verifiëren dat kan 'Greet'-en.

### Eerste systeem: HTTP API

We moeten een "greeter service" via HTTP aanbieden. We moeten dus het volgende aanmaken:

1. Een **driver**. In dit geval werkt men met een HTTP-systeem door een **HTTP-client** te gebruiken. Deze code weet hoe het met onze API moet werken. Drivers vertalen DSL's naar systeemspecifieke aanroepen; in ons geval implementeert de driver de gedefinieerde interfacespecificaties.
2. Een **HTTP-server** met een greet API
3. Een **test**, die verantwoordelijk is voor het beheer van de levenscyclus van het opstarten van de server en het vervolgens koppelen van de driver aan de specificatie om deze als test uit te voeren.

## Schrijf eerst de test

Het initiële proces voor het maken van een black-boxtest die je programma compileert en uitvoert, de test uitvoert en vervolgens alles opschoont, kan behoorlijk arbeidsintensief zijn. Daarom is het beter om dit aan het begin van je project te doen met minimale functionaliteit. Ik start al mijn projecten meestal met een "hello world"-serverimplementatie, waarbij al mijn tests klaarstaan om de daadwerkelijke functionaliteit snel te bouwen.

Het mentale model van "specificaties", "drivers" en "acceptatietests" kan even wennen zijn, dus volg het zorgvuldig. Het kan nuttig zijn om "achteruit te werken" door eerst de specificatie aan te roepen.

Creëer een structuur voor het programma dat we willen uitbrengen.

`mkdir -p cmd/httpserver`

Maak in de nieuwe map een nieuw bestand `greeter_server_test.go` en voeg het volgende toe.

```go
package main_test

import (
	"testing"

	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	specifications.GreetSpecification(t, nil)
}
```

We willen onze specificatie uitvoeren in een Go-test. We hebben al toegang tot een `*testing.T`, dus dat is het eerste argument, maar hoe zit het met het tweede?

`specifications.Greeter` is een interface die we zullen implementeren met een `Driver` door de nieuwe TestGreeterServer-code als volgt te wijzigen:

```go
import (
	go_specs_greet "github.com/quii/go-specs-greet"
)

func TestGreeterServer(t *testing.T) {
	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

Het zou gunstig zijn als onze `Driver` configureerbaar zou zijn om deze in verschillende omgevingen te kunnen gebruiken, waaronder lokaal. Daarom hebben we een `BaseURL`-veld toegevoegd.

## Probeer de test uit te voeren

```
./greeter_server_test.go:46:12: undefined: go_specs_greet.Driver
```

We zijn hier nog steeds bezig met TDD! Het is een belangrijke eerste stap die we moeten zetten; we moeten een paar bestanden aanmaken en misschien meer code schrijven dan we gewend zijn, maar als je net begint, is dit vaak het geval. Het is daarom belangrijk dat we de regels van de rode stap onthouden.

> Bega zoveel zonden als nodig is om de test te laten slagen.

## Schrijf de minimale hoeveelheid code voor de test om uit te voeren en controleer de mislukte testuitvoer.

Hou je mond; onthoud dat we kunnen refactoren wanneer de test is geslaagd. Hier is de code voor de driver in `driver.go` die we in de projectroot plaatsen:

```go
package go_specs_greet

import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
}

func (d Driver) Greet() (string, error) {
	res, err := http.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

Opmerkingen:

- Je zou kunnen stellen dat ik tests zou moeten schrijven om de verschillende `if err != nil`-fouten te omzeilen, maar in mijn ervaring zijn tests die zeggen "je retourneert de fout die je krijgt" relatief laagwaardig, zolang je niets met de `err`-fout doet.
- **Je zou de standaard HTTP-client niet moeten gebruiken**. Later zullen we een HTTP-client toevoegen om deze te configureren met time-outs enz., maar voor nu proberen we gewoon een geslaagde test te krijgen.
- In onze `greeter_server_test.go` hebben we de Driver-functie aangeroepen vanuit het `go_specs_greet`-pakket dat we nu hebben aangemaakt. Vergeet niet `github.com/quii/go-specs-greet` aan de import toe te voegen.
Probeer de tests opnieuw uit te voeren; ze zouden nu moeten compileren, maar niet slagen.

```
Get "http://localhost:8080/greet": dial tcp [::1]:8080: connect: connection refused
```

We hebben een `Driver`, maar onze applicatie is nog niet gestart, dus deze kan geen HTTP-verzoek verwerken. We hebben onze acceptatietest nodig om het bouwen, draaien en uiteindelijk afsluiten van ons systeem te coördineren voordat de test kan worden uitgevoerd.

### Onze applicatie draaien

Het is gebruikelijk dat teams Docker-images van hun systemen bouwen om te implementeren, dus voor onze test doen we hetzelfde.

Om Docker in onze tests te kunnen gebruiken, gebruiken we [Testcontainers](https://golang.testcontainers.org). Testcontainers biedt ons een programmatische manier om Docker-images te bouwen en de levenscycli van containers te beheren.

`go get github.com/testcontainers/testcontainers-go`

Je kunt nu `cmd/httpserver/greeter_server_test.go` bewerken, zodat het er als volgt uitziet:

```go
package main_test

import (
	"context"
	"testing"

	"github.com/alecthomas/assert/v2"
	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func TestGreeterServer(t *testing.T) {
	ctx := context.Background()

	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "./cmd/httpserver/Dockerfile",
			// set to false if you want less spam, but this is helpful if you're having troubles
			PrintBuildLog: true,
		},
		ExposedPorts: []string{"8080:8080"},
		WaitingFor:   wait.ForHTTP("/").WithPort("8080"),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})

	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

Probeer de test uit te voeren.

```
=== RUN   TestGreeterHandler
2022/09/10 18:49:44 Starting container id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Waiting for container id 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Container is ready id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
    greeter_server_test.go:32: Did not expect an error but got:
        Error response from daemon: Cannot locate specified Dockerfile: ./cmd/httpserver/Dockerfile: failed to create container
--- FAIL: TestGreeterHandler (0.59s)
```

We moeten een Dockerfile voor ons programma maken. Maak in onze map `httpserver` een `Dockerfile` en voeg het volgende toe.

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
# For example, golang:1.22.1-alpine.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/httpserver/*.go

EXPOSE 8080
CMD [ "./svr" ]
```

Maak je niet te veel zorgen over de details; het kan verfijnd en geoptimaliseerd worden, maar voor dit voorbeeld is het voldoende. Het voordeel van onze aanpak is dat we later ons Dockerfile kunnen verbeteren en een test kunnen uitvoeren om te bewijzen dat het werkt zoals we willen. Dit is een echte kracht van black-box-tests!

Probeer de test opnieuw uit te voeren; hij zou moeten klagen dat de image niet gebouwd kan worden. Dat komt natuurlijk omdat we nog geen programma hebben geschreven om te bouwen!

Om de test volledig uit te voeren, moeten we een programma maken dat luistert naar `8080`, maar **dat is alles**. Houd je aan de TDD-discipline en schrijf de productiecode die de test zou laten slagen pas als we hebben geverifieerd dat de test faalt zoals verwacht.

Maak een `main.go` aan in onze `httpserver`-map met het volgende

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

Probeer de test opnieuw uit te voeren. Deze zou dan moeten mislukken en het volgende resultaat moet verschijnen.

```
    greet.go:16: Expected values to be equal:
        +Hello, World
        \ No newline at end of file
--- FAIL: TestGreeterHandler (2.09s)
```

## Schrijf voldoende code om het te laten slagen

Werk de handler bij zodat deze zich gedraagt zoals onze specificatie dat voorschrijft

```go
import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		fmt.Fprint(w, "Hello, world")
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

## Refactor

Hoewel dit technisch gezien geen refactor is, moeten we niet vertrouwen op de standaard HTTP-client. Laten we daarom onze driver aanpassen, zodat we er een kunnen leveren die onze test zal opleveren.

```go
import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
	Client  *http.Client
}

func (d Driver) Greet() (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

In onze test in `cmd/httpserver/greeter_server_test.go`, wordt de aanmaak van de driver bijgewerkt om deze door te geven aan een client.

```go
client := http.Client{
	Timeout: 1 * time.Second,
}

driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080", Client: &client}
specifications.GreetSpecification(t, driver)
```

Het is een goede gewoonte om `main.go` zo eenvoudig mogelijk te houden; het zou zich alleen moeten bezighouden met het samenvoegen van de bouwstenen die je maakt tot een applicatie.

Maak een bestand in de projectroot met de naam `handler.go` en verplaats onze code daarheen.

```go
package go_specs_greet

import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello, world")
}
```

Werk `main.go` bij om in plaats daarvan de handler te importeren en te gebruiken.

```go
package main

import (
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet"
)

func main() {
	handler := http.HandlerFunc(go_specs_greet.Handler)
	http.ListenAndServe(":8080", handler)
}
```

## Reflectie

De eerste stap voelde als een inspanning. We hebben verschillende `go`-bestanden gemaakt om een HTTP-handler te maken en te testen die een hardgecodeerde string retourneert. Deze "iteratie 0"-ceremonie en -configuratie zullen ons goed van pas komen bij verdere iteraties.

Het wijzigen van functionaliteit moet eenvoudig en gecontroleerd zijn door deze via de specificatie te sturen en alle wijzigingen die we daardoor moeten aanbrengen, te verwerken. Nu zijn de `DockerFile` en `testcontainers` ingesteld voor onze acceptatietest; we zouden deze bestanden niet hoeven te wijzigen, tenzij de manier waarop we onze applicatie bouwen verandert.

We zullen dit zien met onze volgende eis: begroet een specifieke persoon.

## Schrijf eerst de test

Bewerk onze specificatie

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet(name string) (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet("Mike")
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, Mike")
}
```

Om specifieke mensen te kunnen begroeten, moeten we de interface van ons systeem aanpassen zodat deze een `naam`-parameter accepteert.

## Probeer de test uit te voeren

```
./greeter_server_test.go:48:39: cannot use driver (variable of type go_specs_greet.Driver) as type specifications.Greeter in argument to specifications.GreetSpecification:
	go_specs_greet.Driver does not implement specifications.Greeter (wrong type for Greet method)
		have Greet() (string, error)
		want Greet(name string) (string, error)
```

De wijziging in de specificatie betekent dat onze driver moet worden bijgewerkt.

## Schrijf de minimale hoeveelheid code om de test uit te voeren en controleer de mislukte testuitvoer.

Werk de driver bij zodat deze een querywaarde voor `name` in de aanvraag specificeert om te vragen of een specifieke `name` als begroeting moet worden gebruikt.


```go
import "io"

func (d Driver) Greet(name string) (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet?name=" + name)
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

De test zou nu moeten worden uitgevoerd, maar mislukken.

```
    greet.go:16: Expected values to be equal:
        -Hello, world
        \ No newline at end of file
        +Hello, Mike
        \ No newline at end of file
--- FAIL: TestGreeterHandler (1.92s)
```

## Schrijf voldoende code om het te laten slagen

Extraheer de `naam` uit het verzoek en begroet.

```go
import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %s", r.URL.Query().Get("name"))
}
```

De test zou nu moeten slagen

## Refactor

In [HTTP Handlers Revisited,](https://github.com/quii/learn-go-with-tests/blob/main/http-handlers-revisited.md) hebben we besproken hoe belangrijk het is dat HTTP-handlers alleen verantwoordelijk zijn voor het afhandelen van HTTP-problemen; alle "domeinlogica" moet zich buiten de handler bevinden. Dit stelt ons in staat om domeinlogica geïsoleerd van HTTP te ontwikkelen, waardoor het eenvoudiger te testen en te begrijpen is.

Laten we deze problemen eens nader bekijken.

We werken onze handler in `./handler.go` als volgt bij:

```go
func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, Greet(name))
}
```

maak een nieuw bestand aan `./greet.go`:

```go
package go_specs_greet

import "fmt"

func Greet(name string) string {
	return fmt.Sprintf("Hello, %s", name)
}
```

## Een kleine uitweiding over het "adapter"-ontwerppatroon

Nu we onze domeinlogica voor het begroeten van mensen in een aparte functie hebben ondergebracht, kunnen we nu unittests schrijven voor onze greet-functie. Dit is ongetwijfeld een stuk eenvoudiger dan het testen via een specificatie die via een driver gaat die een webserver benadert, om een string te verkrijgen!

Zou het niet mooi zijn als we onze specificatie hier ook zouden kunnen hergebruiken? Het punt van de specificatie is immers losgekoppeld van de implementatiedetails. Als de specificatie onze **essentiële complexiteit** vastlegt en onze "domein"-code deze moet modelleren, zouden we ze samen moeten kunnen gebruiken.

Laten we het proberen door `./greet_test.go` als volgt aan te maken:

```go
package go_specs_greet_test

import (
	"testing"

	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(t, go_specs_greet.Greet)
}

```

Dit zou leuk zijn, maar werkt helaas niet

```
./greet_test.go:11:39: cannot use go_specs_greet.Greet (value of type func(name string) string) as type specifications.Greeter in argument to specifications.GreetSpecification:
	func(name string) string does not implement specifications.Greeter (missing Greet method)
```

Onze specificatie wil iets met een methode `Greet()`, geen functie.

De compilatiefout is frustrerend; we hebben iets waarvan we "weten" dat het een `Greeter` is, maar het is niet helemaal in de juiste **vorm** om door de compiler gebruikt te kunnen worden. Dit is waar het **adapter**-patroon voor zorgt.

> In [software engineering](https://en.wikipedia.org/wiki/Software_engineering) is het **adapterpatroon** een [softwareontwerppatroon](https://en.wikipedia.org/wiki/Software_design_pattern) (ook bekend als [wrapper](https://en.wikipedia.org/wiki/Wrapper_function), een alternatieve naamgeving die wordt gedeeld met het [decoratorpatroon](https://en.wikipedia.org/wiki/Decorator_pattern)) waarmee de [interface](https://en.wikipedia.org/wiki/Interface_(computer_science)) van een bestaande [klasse](https://en.wikipedia.org/wiki/Class_(computer_science)) kan worden gebruikt als een andere interface.[[1\]](https://en.wikipedia.org/wiki/Adapter_pattern#cite_note-HeadFirst-1) Het wordt vaak gebruikt om bestaande klassen met andere klassen te laten werken zonder hun [bron code](https://en.wikipedia.org/wiki/Source_code) te wijzigen.

Veel mooie woorden voor iets relatief eenvoudigs, wat vaak het geval is bij ontwerppatronen, waardoor mensen er vaak met hun ogen rollen. De waarde van ontwerppatronen zit niet in specifieke implementaties, maar in een taal die specifieke oplossingen beschrijft voor veelvoorkomende problemen waar engineers mee te maken krijgen. Als je een team hebt dat een gedeelde woordenschat heeft, vermindert dat de communicatieproblemen.

Voeg deze code toe aan `./specifications/adapters.go`

```go
type GreetAdapter func(name string) string

func (g GreetAdapter) Greet(name string) (string, error) {
	return g(name), nil
}
```

We kunnen onze adapter nu in onze test gebruiken om onze `Greet`-functie in de specificatie op te nemen.

```go
package go_specs_greet_test

import (
	"testing"

	gospecsgreet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(
		t,
		specifications.GreetAdapter(gospecsgreet.Greet),
	)
}
```

Het adapterpatroon is handig wanneer je een type hebt dat het gewenste gedrag vertoont voor een interface, maar niet de juiste vorm heeft.

## Reflecteren

De gedragsverandering voelde eenvoudig, toch? Oké, misschien lag het gewoon aan de aard van het probleem, maar deze werkwijze geeft je discipline en een eenvoudige, herhaalbare manier om je systeem van top tot teen te veranderen:

- Analyseer je probleem en identificeer een kleine verbetering aan je systeem die je in de goede richting duwt
- Leg de nieuwe essentiële complexiteit vast in een specificatie
- Volg de compilatiefouten totdat de acceptatietest draait
- Werk je implementatie bij zodat het systeem zich gedraagt volgens de specificatie
- Refactor

Na de pijn van de eerste iteratie hoefden we onze acceptatietestcode niet aan te passen, omdat we de specificaties, drivers en implementatie van elkaar gescheiden houden. Het wijzigen van onze specificatie vereiste dat we onze driver en uiteindelijk onze implementatie moesten bijwerken, maar de boilerplate-code over _hoe_ het systeem als container moest worden opgestart, bleef onaangetast.

Zelfs met de overhead van het bouwen van een docker-image voor onze applicatie en het opstarten van de container, is de feedbacklus voor het testen van onze **hele** applicatie erg strak:

```
quii@Chriss-MacBook-Pro go-specs-greet % go test ./...
ok  	github.com/quii/go-specs-greet	0.181s
ok  	github.com/quii/go-specs-greet/cmd/httpserver	2.221s
?   	github.com/quii/go-specs-greet/specifications	[no test files]
```

Stel je nu voor dat je CTO heeft besloten dat gRPC _de toekomst_ is. Ze wil dat je dezelfde functionaliteit beschikbaar stelt via een gRPC-server, terwijl je de bestaande HTTP-server behoudt.

Dit is een voorbeeld van **toevallige complexiteit**. Onthoud dat toevallige complexiteit de complexiteit is waarmee we te maken hebben omdat we werken met computers, zaken als netwerken, schijven, API's, enzovoort. **De essentiële complexiteit is niet veranderd**, dus we zouden onze specificaties niet hoeven te wijzigen.

Veel repositorystructuren en ontwerppatronen houden zich voornamelijk bezig met het scheiden van soorten complexiteit. Zo vragen "poorten en adapters" je om je domeincode te scheiden van alles wat met toevallige complexiteit te maken heeft; die code staat in een map "adapters".

### De wijziging eenvoudig maken

Soms is het zinvol om wat refactoring uit te voeren _voordat_ een wijziging wordt aangebracht.

> Eerst de wijziging eenvoudig maken, dan de eenvoudige wijziging doorvoeren

~Kent Beck

Laten we daarom onze `http`-code - `driver.go` en `handler.go` - verplaatsen naar een pakket met de naam `httpserver` in een map `adapters` en hun pakketnamen wijzigen in `httpserver`.

Je moet nu het root-pakket importeren in `handler.go` om te verwijzen naar de Greet-methode...

```go
package httpserver

import (
	"fmt"
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet/domain/interactions"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, go_specs_greet.Greet(name))
}

```

Importeer je httpserveradapter in main.go:

```go
package main

import (
	"net/http"

	"github.com/quii/go-specs-greet/adapters/httpserver"
)

func main() {
	handler := http.HandlerFunc(httpserver.Handler)
	http.ListenAndServe(":8080", handler)
}
```

en werk de import en verwijzing naar `Driver` in greeter_server_test.go bij:

```go
driver := httpserver.Driver{BaseURL: "http://localhost:8080", Client: &client}
```

Tot slot is het handig om ook onze code op domeinniveau in een eigen map te verzamelen. Wees niet lui en maak een map `domain` aan in je projecten met honderden niet-gerelateerde typen en functies. Neem de moeite om na te denken over je domein en groepeer ideeën die bij elkaar horen. Dit maakt je project begrijpelijker en verbetert de kwaliteit van je imports.

In plaats van te zien

```go
domain.Greet
```

Wat gewoon een beetje raar is, in plaats daarvan heeft dit de voorkeur:

```go
interactions.Greet
```

Maak een map `domain` aan voor al je domeincode, en daarbinnen een map `interactions`. Afhankelijk van je tooling moet je mogelijk enkele imports en code bijwerken.

Onze projectboom zou er nu zo uit moeten zien:

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   └── httpserver
|       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── adapters.go
    └── greet.go

```

Onze domeincode, **essentiële complexiteit**, vormt de basis van onze go-module, en code die ons in staat stelt deze in "de echte wereld" te gebruiken, is georganiseerd in **adapters**. De map `cmd` is waar we deze logische groeperingen kunnen samenvoegen tot praktische applicaties, die black-box tests hebben om te verifiëren dat alles werkt. Mooi!

Ten slotte kunnen we onze acceptatietest een _klein_ beetje opschonen. Als je de hoofdstappen van onze acceptatietest bekijkt:

- Bouw een docker-image
- Wacht tot deze luistert op _een_ poort
- Maak een driver die begrijpt hoe de DSL moet worden vertaald naar systeemspecifieke aanroepen
- Sluit de driver aan op de specificatie

... je zult je realiseren dat we dezelfde vereisten hebben voor een acceptatietest voor de gRPC-server!

De map `adapters` lijkt een goede plek, dus in een bestand met de naam `docker.go` kapselen we de eerste twee stappen in een functie in die we hierna zullen hergebruiken.

```go
package adapters

import (
	"context"
	"fmt"
	"testing"
	"time"

	"github.com/alecthomas/assert/v2"
	"github.com/docker/go-connections/nat"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func StartDockerServer(
	t testing.TB,
	port string,
	dockerFilePath string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:       "../../.",
			Dockerfile:    dockerFilePath,
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

Dit geeft ons de gelegenheid om onze acceptatietest een beetje op te schonen

```go
func TestGreeterServer(t *testing.T) {
	var (
		port           = "8080"
		dockerFilePath = "./cmd/httpserver/Dockerfile"
		baseURL        = fmt.Sprintf("http://localhost:%s", port)
		driver         = httpserver.Driver{BaseURL: baseURL, Client: &http.Client{
			Timeout: 1 * time.Second,
		}}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, driver)
}
```

Dit zou het schrijven van de _volgende_ test eenvoudiger moeten maken.

## Schrijf eerst de test

Deze nieuwe functionaliteit kan worden gerealiseerd door een nieuwe `adapter` aan te maken die communiceert met onze domeincode. Om die reden:

- hoeven we de specificatie niet te wijzigen;
- moeten we de specificatie kunnen hergebruiken;
- moeten we de domeincode kunnen hergebruiken.

Maak een nieuwe map `grpcserver` aan binnen `cmd` om ons nieuwe programma en de bijbehorende acceptatietest te huisvesten. Voeg binnen `cmd/grpc_server/greeter_server_test.go` een acceptatietest toe, die sterk lijkt op onze HTTP-servertest. Dat is niet toevallig maar een bewuste ontwerpkeuze.

```go
package main_test

import (
	"fmt"
	"testing"

	"github.com/quii/go-specs-greet/adapters"
	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	var (
		port           = "50051"
		dockerFilePath = "./cmd/grpcserver/Dockerfile"
		driver         = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, &driver)
}
```

De enige verschillen zijn:

- We gebruiken een ander dockerbestand, omdat we een ander programma bouwen.
- Dit betekent dat we een nieuwe driver nodig hebben die gRPC gebruikt om met ons nieuwe programma te communiceren.

## Probeer de test uit te voeren

```
./greeter_server_test.go:26:12: undefined: grpcserver
```

We hebben nog geen `Driver` aangemaakt, dus deze zal niet compileren.

## Schrijf de minimale hoeveelheid code voor de test om uit te voeren en controleer de mislukte testuitvoer.

Maak een map `grpcserver` aan in `adapters` en maak daarin `driver.go` aan.

```go
package grpcserver

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	return "", nil
}
```

Als je het opnieuw uitvoert, zou het nu moeten _compileren_, maar zal de test niet slagen, omdat we geen Dockerfile en bijbehorend programma hebben aangemaakt om uit te voeren.

Maak een nieuwe `Dockerfile` aan in `cmd/grpcserver`.

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/grpcserver/*.go

EXPOSE 50051
CMD [ "./svr" ]
```

En een `main.go`

```go
package main

import "fmt"

func main() {
	fmt.Println("implement me")
}
```

Je zou nu moeten merken dat de test mislukt omdat onze server niet op de poort luistert. Nu is het tijd om onze client en server met gRPC te bouwen.

## Schrijf voldoende code om de test te laten slagen

### gRPC

Als je niet bekend bent met gRPC, zou ik beginnen met het bekijken van de [gRPC-website](https://grpc.io). Voor dit hoofdstuk is het echter gewoon een soort adapter in ons systeem, een manier voor andere systemen om onze uitstekende domeincode aan te roepen (**r**emote **p**rocedure **c**all).

Het probleem is dat je een "servicedefinitie" definieert met behulp van protocolbuffers. Vervolgens genereer je server- en clientcode op basis van de definitie. Dit werkt niet alleen voor Go, maar ook voor de meeste gangbare talen. Dit betekent dat je een definitie kunt delen met andere teams in je bedrijf die misschien niet eens Go schrijven, en toch soepel service-to-servicecommunicatie kunt uitvoeren.

Als je gRPC nog niet eerder hebt gebruikt, moet je een **Protocolbuffercompiler** en een aantal **Go-plug-ins** installeren. [De gRPC-website bevat duidelijke instructies hiervoor](https://grpc.io/docs/languages/go/quickstart/).

Voeg in dezelfde map als onze nieuwe driver een `greet.proto`-bestand toe met de volgende inhoud:

```protobuf
syntax = "proto3";

option go_package = "github.com/quii/adapters/grpcserver";

package grpcserver;

service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
}

message GreetRequest {
  string name = 1;
}

message GreetReply {
  string message = 1;
}
```

Om deze definitie te begrijpen, hoef je geen expert te zijn in protocolbuffers. We definiëren een service met een Greet-methode en beschrijven vervolgens de inkomende en uitgaande berichttypen.

Voer binnen `adapters/grpcserver` het volgende uit om de client- en servercode te genereren.

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

Als het werkt, zouden we code hebben moeten gegenereren die we kunnen gebruiken. Laten we beginnen met het gebruiken van de gegenereerde clientcode in onze `Driver`.

```go
package grpcserver

import (
	"context"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	//todo: we shouldn't redial every time we call greet, refactor out when we're green
	conn, err := grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return "", err
	}
	defer conn.Close()

	client := NewGreeterClient(conn)
	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

Nu we een client hebben, moeten we onze `main.go` bijwerken om een server aan te maken. Onthoud: op dit moment proberen we alleen onze test te laten slagen en maken we ons geen zorgen over de kwaliteit van de code.

```go
package main

import (
	"context"
	"log"
	"net"

	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"google.golang.org/grpc"
)

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}
	s := grpc.NewServer()
	grpcserver.RegisterGreeterServer(s, &GreetServer{})

	if err := s.Serve(lis); err != nil {
		log.Fatal(err)
	}
}

type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: "fixme"}, nil
}
```

To create our gRPC server, we have to implement the interface it generated for us

```go
// GreeterServer is the server API for Greeter service.
// All implementations must embed UnimplementedGreeterServer
// for forward compatibility
type GreeterServer interface {
	Greet(context.Context, *GreetRequest) (*GreetReply, error)
	mustEmbedUnimplementedGreeterServer()
}
```

Onze `main`-functie:

- Luistert naar een poort
- Maakt een `GreetServer` aan die de interface implementeert en registreert deze vervolgens bij `grpcServer.RegisterGreeterServer`, samen met een `grpc.Server`.
- Gebruikt de server met de listener

Het zou geen enorme extra moeite kosten om onze domeincode aan te roepen in `greetServer.Greet` in plaats van `fix-me` hard te coderen in het bericht, maar ik wil eerst onze acceptatietest uitvoeren om te zien of alles werkt op transportniveau en de mislukte testuitvoer te verifiëren.

```
greet.go:16: Expected values to be equal:
-fixme
\ No newline at end of file
+Hello, Mike
\ No newline at end of file
```

Goed! We zien dat onze driver in de test verbinding kan maken met onze gRPC-server.

Noem nu onze domeincode in onze `GreetServer`

```go
type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

Eindelijk is het gelukt! We hebben een acceptatietest die bewijst dat onze gRPC-begroetingsserver zich gedraagt zoals we willen.

## Refactor

We hebben verschillende fouten gemaakt om de test te laten slagen, maar nu ze slagen, hebben we het vangnet om te refactoren.

### Vereenvoudig main

Net als voorheen willen we niet dat `main` te veel code bevat. We kunnen onze nieuwe `GreetServer` verplaatsen naar `adapters/grpcserver`, aangezien die daar hoort. Om de samenhang te behouden: als we de servicedefinitie wijzigen, willen we dat de "blast-radius" van de wijziging beperkt blijft tot dat deel van onze code.

### Roep niet elke keer opnieuw onze driver aan

We hebben maar één test, maar als we onze specificatie uitbreiden (wat we zullen doen), is het niet zinvol dat de driver bij elke RPC-oproep opnieuw aanroept.

```go
package grpcserver

import (
	"context"
	"sync"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string

	connectionOnce sync.Once
	conn           *grpc.ClientConn
	client         GreeterClient
}

func (d *Driver) Greet(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}

func (d *Driver) getClient() (GreeterClient, error) {
	var err error
	d.connectionOnce.Do(func() {
		d.conn, err = grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
		d.client = NewGreeterClient(d.conn)
	})
	return d.client, err
}
```

Hier laten we zien hoe we [`sync.Once`](https://pkg.go.dev/sync#Once) kunnen gebruiken om ervoor te zorgen dat onze `Driver` slechts één keer verbinding probeert te maken met onze server.

Laten we eerst de huidige status van onze projectstructuur bekijken voordat we verdergaan.

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   ├── docker.go
│   ├── grpcserver
│   │   ├── driver.go
│   │   ├── greet.pb.go
│   │   ├── greet.proto
│   │   ├── greet_grpc.pb.go
│   │   └── server.go
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   ├── grpcserver
│   │   ├── Dockerfile
│   │   ├── greeter_server_test.go
│   │   └── main.go
│   └── httpserver
│       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── greet.go
```

- `adapters` hebben samenhangende functionaliteitseenheden die gegroepeerd zijn
- `cmd` bevat onze applicaties en bijbehorende acceptatietests
- Onze code is volledig ontkoppeld van elke toevallige complexiteit

### `Dockerfile` consolideren

Je hebt waarschijnlijk gemerkt dat de twee `Dockerfiles` vrijwel identiek zijn, afgezien van het pad naar de binaire code die we willen bouwen.

`Dockerfiles` kunnen argumenten accepteren, zodat we ze in verschillende contexten kunnen hergebruiken, wat perfect klinkt. We kunnen onze twee Dockerfiles verwijderen en er in plaats daarvan één in de root van het project plaatsen met de volgende code:

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
FROM golang:1.18-alpine

WORKDIR /app

ARG bin_to_build

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/${bin_to_build}/main.go

CMD [ "./svr" ]
```

We zullen onze `StartDockerServer`-functie moeten bijwerken om het argument door te geven wanneer we de images bouwen

```go
func StartDockerServer(
	t testing.TB,
	port string,
	binToBuild string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "Dockerfile",
			BuildArgs: map[string]*string{
				"bin_to_build": &binToBuild,
			},
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

En werk ten slotte onze tests bij zodat de te bouwen image wordt doorgegeven (doe dit voor de andere test en verander `grpcserver` in `httpserver`).

```go
func TestGreeterServer(t *testing.T) {
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
}
```

### Verschillende soorten tests scheiden

Acceptatietests zijn geweldig omdat ze de werking van het hele systeem testen vanuit een puur gebruikersgericht, gedragsmatig perspectief, maar ze hebben ook hun nadelen ten opzichte van unittests:

- Ze zijn langzamer
- De kwaliteit van de feedback is vaak niet zo gericht als bij een unittest
- Helpt je niet met de interne kwaliteit of het ontwerp

[De Testpiramide](https://martinfowler.com/articles/practical-test-pyramid.html) geeft ons richtlijnen voor de mix die we willen voor onze testsuite. Lees Fowlers bericht voor meer details, maar de simplistische samenvatting van dit bericht is "veel unittests en een paar acceptatietests".

Om die reden kan het voorkomen dat je, naarmate een project groeit, in situaties terechtkomt waarin de acceptatietests een paar minuten duren. Om een prettige ontwikkelaarservaring te bieden aan mensen die je project bekijken, kun je ontwikkelaars de mogelijkheid bieden om de verschillende soorten tests afzonderlijk uit te voeren.

Het is beter dat `go test ./...` uitgevoerd kan worden zonder verdere configuratie door een engineer, afgezien van bijvoorbeeld een paar belangrijke afhankelijkheden zoals de Go-compiler (uiteraard) en eventueel Docker.

Go biedt engineers een mechanisme om alleen "korte" tests uit te voeren met de [short flag](https://pkg.go.dev/testing#Short)

`go test -short ./...`

We kunnen onze acceptatietests uitbreiden om te zien of de gebruiker onze acceptatietests wil uitvoeren door de waarde van de vlag te inspecteren

```go
if testing.Short() {
	t.Skip()
}
```

Ik heb een `Makefile` gemaakt om dit gebruik te laten zien

```makefile
build:
	golangci-lint run
	go test ./...

unit-tests:
	go test -short ./...
```

### Wanneer moet ik acceptatietests schrijven?

De beste aanpak is om de voorkeur te geven aan veel snellopende unittests en een paar acceptatietests, maar hoe bepaal je wanneer je een acceptatietest moet schrijven en niet unittests?

Het is moeilijk om een concrete regel te geven, maar de vragen die ik mezelf meestal stel zijn:

- Is dit een edge case? Ik zou liever unittests uitvoeren.
- Is dit iets waar mensen zonder computerkennis veel over praten? Ik zou er liever zeker van zijn dat het belangrijkste "echt" werkt, dus ik zou een acceptatietest toevoegen.
- Beschrijf ik een gebruikersreis in plaats van een specifieke functie? Acceptatietest.
- Zouden unittests me voldoende vertrouwen geven? Soms neem je een bestaande reis die al een acceptatietest heeft, maar voeg je andere functionaliteit toe om met verschillende scenario's om te gaan vanwege verschillende invoer. In dit geval brengt het toevoegen van nog een acceptatietest kosten met zich mee, maar levert het weinig waarde op, dus ik zou de voorkeur geven aan enkele unittests.

## Itereren op ons werk

Met al deze moeite hoop je dat het uitbreiden van ons systeem nu eenvoudig zal zijn. Het maken van een systeem dat eenvoudig te gebruiken is, is niet per se makkelijk, maar het is de tijd waard en aanzienlijk eenvoudiger om te doen wanneer je een project start.

Laten we onze API uitbreiden met een "curse"-functionaliteit.

## Schrijf eerst de test

Dit is gloednieuw gedrag, dus we moeten beginnen met een acceptatietest. Voeg het volgende toe aan ons specificatiebestand:

```go
type MeanGreeter interface {
	Curse(name string) (string, error)
}

func CurseSpecification(t *testing.T, meany MeanGreeter) {
	got, err := meany.Curse("Chris")
	assert.NoError(t, err)
	assert.Equal(t, got, "Go to hell, Chris!")
}
```

Kies een van onze acceptatietests en probeer de specificatie te gebruiken

```go
func TestGreeterServer(t *testing.T) {
	if testing.Short() {
		t.Skip()
	}
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	t.Cleanup(driver.Close)
	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
	specifications.CurseSpecification(t, &driver)
}
```

## Probeer de test uit te voeren

```
# github.com/quii/go-specs-greet/cmd/grpcserver_test [github.com/quii/go-specs-greet/cmd/grpcserver.test]
./greeter_server_test.go:27:39: cannot use &driver (value of type *grpcserver.Driver) as type specifications.MeanGreeter in argument to specifications.CurseSpecification:
	*grpcserver.Driver does not implement specifications.MeanGreeter (missing Curse method)
```

Onze `Driver` ondersteunt `Curse` nog niet.

## Schrijf de minimale hoeveelheid code om de test uit te voeren en controleer de uitvoer van de mislukte test.

Onthoud dat we alleen proberen de test uit te voeren, dus voeg de methode toe aan `Driver`

```go
func (d *Driver) Curse(name string) (string, error) {
	return "", nil
}
```

Als je het opnieuw probeert, moet de test compileren, wordt deze uitgevoerd en zal deze mislukken

```
greet.go:26: Expected values to be equal:
+Go to hell, Chris!
\ No newline at end of file
```

## Schrijf voldoende code om het te laten slagen

We moeten onze protocolbufferspecificatie bijwerken met een `Curse`-methode en vervolgens onze code opnieuw genereren.

```protobuf
service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
  rpc Curse (GreetRequest) returns (GreetReply) {}
}
```

Je zou kunnen stellen dat het hergebruiken van de typen `GreetRequest` en `GreetReply` een ongepaste koppeling is, maar daar kunnen we in de refactoringfase mee omgaan. Zoals ik blijf benadrukken: we proberen gewoon de test te laten slagen, zodat we controleren of de software werkt, _en_ dan kunnen we de code goed maken.

Genereer onze code opnieuw met (in `adapters/grpcserver`).

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

### Driver bijwerken

Nu de clientcode is bijgewerkt, kunnen we `Curse` aanroepen in onze `Driver`

```go
func (d *Driver) Curse(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Curse(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

### Server bijwerken

Ten slotte moeten we de `Curse`-methode toevoegen aan onze `Server`

```go
package grpcserver

import (
	"context"
	"fmt"

	"github.com/quii/go-specs-greet/domain/interactions"
)

type GreetServer struct {
	UnimplementedGreeterServer
}

func (g GreetServer) Curse(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: fmt.Sprintf("Go to hell, %s!", request.Name)}, nil
}

func (g GreetServer) Greet(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

De tests zouden nu moeten slagen.

## Refactor

Probeer dit zelf.

- Extraheer de `Curse`-domeinlogica, weg van de grpc-server, zoals we dat voor `Greet` hebben gedaan. Gebruik de specificatie als een unittest voor je domeinlogica.
- Zorg voor verschillende typen in de protobuf om ervoor te zorgen dat de berichttypen voor `Greet` en `Curse` ontkoppeld zijn.

## `Curse` implementeren voor de HTTP-server

Nogmaals, een oefening voor jou, de lezer. We hebben onze specificatie op domeinniveau en onze logica op domeinniveau netjes gescheiden. Als je dit hoofdstuk hebt gevolgd, zou dit heel eenvoudig moeten zijn.

- Voeg de specificatie toe aan de bestaande acceptatietest voor de HTTP-server
- Werk je `Driver` bij
- Voeg het nieuwe eindpunt toe aan de server en hergebruik de domeincode om de functionaliteit te implementeren. Je kunt `http.NewServeMux` gebruiken om de routering naar de afzonderlijke eindpunten af te handelen.

Vergeet niet om in kleine stapjes te werken, commit en voer je tests regelmatig uit. Als je er echt niet uitkomt [kun je mijn implementatie vinden op GitHub](https://github.com/quii/go-specs-greet).

## Verbeter beide systemen door de domeinlogica bij te werken met een unittest

Zoals gezegd, hoeft niet elke wijziging in een systeem via een acceptatietest te worden uitgevoerd. Permutaties van bedrijfsregels en grensgevallen moeten eenvoudig via een unittest kunnen worden uitgevoerd als je de aandachtspunten goed hebt gescheiden.

Voeg een unittest toe aan onze `Greet`-functie om de `name` standaard in te stellen op `World` als deze leeg is. Je zult zien hoe eenvoudig dit is, en de bedrijfsregels worden vervolgens "gratis" in beide applicaties weergegeven.

## Samenvattend

Het bouwen van systemen met redelijke wijzigingskosten vereist dat AT's zo zijn ontworpen dat ze je helpen en geen onderhoudslast vormen. Ze kunnen worden gebruikt als middel om je software te begeleiden, of zoals een GOOS het noemt, methodisch te laten "groeien".

Hopelijk zie je met dit voorbeeld de voorspelbare, gestructureerde workflow van onze applicatie voor het stimuleren van verandering en hoe je deze voor je werk kunt gebruiken.

Je kunt je voorstellen dat je met een stakeholder praat die het systeem waaraan je werkt op de een of andere manier wil uitbreiden. Leg dit domeingericht en implementatieonafhankelijk vast in een specificatie en gebruik dit als leidraad voor je inspanningen. Riya en ik beschrijven het gebruik van BDD-technieken zoals "Example Mapping" [in onze GopherconUK-lezing](https://www.youtube.com/watch?v=ZMWJCk_0WrY) om je te helpen de essentiële complexiteit beter te begrijpen en je in staat te stellen gedetailleerdere en zinvollere specificaties te schrijven.

Het scheiden van essentiële en onvoorziene complexiteitskwesties maakt je werk minder ad-hoc en meer gestructureerd en weloverwogen; Dit zorgt voor de veerkracht van je acceptatietests en zorgt ervoor dat ze minder onderhoudsintensief worden.

Dave Farley geeft een uitstekende tip:

> Stel je voor dat de minst technische persoon die je kunt bedenken, die het probleemdomein begrijpt, je acceptatietests leest. De tests moeten voor die persoon logisch zijn.

Specificaties dienen dan ook als documentatie. Ze moeten duidelijk specificeren hoe een systeem zich moet gedragen. Dit idee is het principe achter tools zoals [Cucumber](https://cucumber.io), die je een DSL biedt om gedragingen als code vast te leggen, en die DSL vervolgens omzet in systeemaanroepen, net zoals we hier deden.

### Wat is behandeld

- Het schrijven van abstracte specificaties stelt je in staat de essentiële complexiteit van het probleem dat je oplost uit te drukken en onbedoelde complexiteit te verwijderen. Dit stelt je in staat de specificaties in verschillende contexten te hergebruiken.
- Hoe je [Testcontainers](https://golang.testcontainers.org) kunt gebruiken om de levenscyclus van je systeem voor AT's te beheren. Dit stelt je in staat om de image die je wilt verzenden grondig te testen op je computer, wat je snelle feedback en vertrouwen geeft.
- Een korte introductie tot het containeriseren van je applicatie met Docker
- gRPC
- In plaats van te jagen op standaard mappenstructuren, kun je je ontwikkelaanpak gebruiken om de structuur van je applicatie op natuurlijke wijze te ontwikkelen, gebaseerd op je eigen behoeften.

### Verder materiaal

- In dit voorbeeld is onze "DSL" niet echt een DSL; we hebben gewoon interfaces gebruikt om onze specificatie los te koppelen van de echte wereld en om domeinlogica helder uit te drukken. Naarmate je systeem groeit, kan dit abstractieniveau onhandig en onduidelijk worden. [Lees het "Screenplay Pattern"](https://cucumber.io/blog/bdd/understanding-screenplay-(part-1)/) als je meer ideeën wilt over hoe je je specificaties kunt structureren.
- Ter verduidelijking: [Growing Object-Oriented Software, Guided by Tests,](http://www.growing-object-oriented-software.com) is een klassieker. Het laat zien hoe deze "London style", "top-down"-benadering, wordt toegepast op het schrijven van software. Iedereen die Learn Go with Tests heeft gelezen, zal veel baat hebben bij het lezen van GOOS. - [In de voorbeeldcode repository](https://github.com/quii/go-specs-greet) staat meer code en ideeën die ik hier nog niet heb beschreven, zoals de multi-stage docker build. Misschien wil je dit eens bekijken.
- In het bijzonder heb ik *voor de lol* een **derde programma** gemaakt, een website met een aantal HTML-formulieren om te `Greeten` en `Vloeken`. De `Driver` maakt gebruik van de uitstekend uitziende [https://github.com/go-rod/rod](https://github.com/go-rod/rod) module, waardoor het met de website kan werken via een browser, net zoals een gebruiker dat zou doen. Als je naar de git-geschiedenis kijkt, zie je hoe ik begon met het niet gebruiken van templates "gewoon om het werkend te krijgen". Toen ik eenmaal geslaagd was voor mijn acceptatietest, had ik de vrijheid om dat te doen zonder bang te zijn dingen kapot te maken. -->
