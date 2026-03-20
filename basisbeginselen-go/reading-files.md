# Bestanden lezen

* [**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/reading-files)
* [Hier is een video waarin ik (Chris James) het probleem aanpak en vragen beantwoord uit de Twitch-stream](https://www.youtube.com/watch?v=nXts4dEJnkU)

In dit hoofdstuk gaan we leren hoe we bestanden kunnen lezen, er gegevens uit kunnen halen en er iets nuttigs mee kunnen doen.

Stel je voor dat je samen met een vriend(in) aan het bloggen bent. Het idee is dat een auteur zijn of haar berichten in markdown schrijft, met wat metadata bovenaan het bestand. Bij het opstarten leest de webserver een map om een ​​aantal berichten te maken, waarna een aparte `NewHandler`-functie die berichten gebruikt als gegevensbron voor de webserver van de blog.

Ons is gevraagd een pakket te maken waarmee een bepaalde map met blogberichtbestanden kan worden omgezet in een verzameling berichten (`Post`s).

### Voorbeeld data

`hello world.md`

```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

### Verwachte data

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

## Iteratieve, test-gestuurde ontwikkeling

We hanteren een iteratieve aanpak, waarbij we steeds eenvoudige, veilige stappen zetten om ons doel te bereiken.

Dat vereist dat we ons werk opsplitsen, maar we moeten wel oppassen dat we niet in de valkuil trappen van een ['bottom-up'](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design)-benadering.

We moeten niet op onze overactieve verbeelding vertrouwen wanneer we aan het werk gaan. We zouden in de verleiding kunnen komen om een ​​soort abstractie te maken die pas gevalideerd wordt als we alles aan elkaar plakken, zoals een soort `BlogPostFileParser`.

Dit is _niet_ iteratief en mist de strakke feedback lussen die TDD ons zou moeten opleveren.

Kent Beck zegt hierover:

> Optimisme is een beroepsrisico van programmeren. Feedback is de behandeling.

In plaats daarvan moeten we ernaar streven om zo snel mogelijk zo dicht mogelijk bij het leveren van _echte consumentenwaarde_ te komen (vaak een "happy path" genoemd). Zodra we een kleine hoeveelheid consumentenwaarde van begin tot eind hebben opgeleverd, is verdere iteratie van de rest van de eisen meestal eenvoudig.

## Nadenken over het soort test dat we willen zien

Laten we onszelf herinneren aan onze mindset en doelen wanneer we beginnen:

* **Schrijf de test die we willen zien.** Denk na over hoe we de code die we gaan schrijven willen gebruiken vanuit het perspectief van de consument.
* Concentreer je op het _wat_ en _waarom_, maar laat je niet afleiden door het _hoe_.

Ons pakket moet een functie bieden waarmee naar een map kan worden verwezen en die ons berichten terugstuurt.

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS("some-folder")
```

Om hier een test over te schrijven, hebben we een soort testmap nodig met een aantal voorbeeldberichten. _Daar is niets mis mee_, maar je maakt wel een paar compromissen:

* Voor elke test moet je mogelijk nieuwe bestanden maken om een ​​bepaald gedrag te testen
* sommige gedragingen zullen moeilijk te testen zijn, zoals het niet kunnen laden van bestanden
* de tests zullen iets langzamer verlopen omdat ze toegang nodig hebben tot het bestandssysteem

Bovendien koppelen we onszelf onnodig aan een specifieke implementatie van het bestandssysteem.

### Bestandssysteemabstracties geïntroduceerd in Go 1.16&#x20;

Go 1.16 introduceerde een abstractie voor bestandssystemen: het [io/fs](https://golang.org/pkg/io/fs/)-pakket.

> Pakket fs definieert basisinterfaces voor een bestandssysteem. Een bestandssysteem kan worden geleverd door het hostbesturingssysteem, maar ook door andere pakketten.

Hiermee kunnen we de koppeling met een specifiek bestandssysteem versoepelen, waarna we verschillende implementaties kunnen injecteren op basis van onze behoeften.

> [Aan de producerkant van de interface implementeert het nieuwe type fs.FS, net als zip.Reader. De nieuwe functie os.DirFS biedt een implementatie van fs.FS, ondersteund door een boomstructuur van besturingssysteem bestanden.](https://golang.org/doc/go1.16#fs)

Met deze interface hebben gebruikers van ons pakket een aantal ingebouwde opties in de standaardbibliotheek tot hun beschikking. Het leren gebruiken van interfaces die zijn gedefinieerd in de standaardbibliotheek van Go (bijv. `io.fs`, [`io.Reader`](https://golang.org/pkg/io/#Reader), [`io.Writer`](https://golang.org/pkg/io/#Writer)) is essentieel voor het schrijven van los gekoppelde pakketten. Deze pakketten kunnen vervolgens opnieuw worden gebruikt in andere contexten dan je je had voorgesteld, met minimale moeite voor je gebruikers.

In ons geval wil onze gebruiker misschien dat de berichten in het Go-bestand worden ingesloten in plaats van bestanden in een "echt" bestandssysteem? _Hoe dan ook, onze code hoeft zich er geen zorgen over te maken._

Voor onze tests biedt het pakket [testing/fstest](https://golang.org/pkg/testing/fstest/) ons een implementatie van [io/FS](https://golang.org/pkg/io/fs/#FS) die we kunnen gebruiken, vergelijkbaar met de hulpmiddelen die we kennen van [net/http/httptest](https://golang.org/pkg/net/http/httptest/).

Gezien deze informatie lijkt het volgende een betere aanpak:

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS(someFS)
```

## Schrijf eerst je test

We moeten de scope zo klein en bruikbaar mogelijk houden. Als we bewijzen dat we alle bestanden in een directory kunnen lezen, is dat een goed begin. Dit geeft ons vertrouwen in de software die we schrijven. We kunnen controleren of het aantal geretourneerde `[]Post`-bestanden gelijk is aan het aantal bestanden in ons nepbestandssysteem.

Omdat dit project wat serieuzer wordt, gaan we dit als een nieuw project aanpakken.&#x20;

* `mkdir blogposts`
* `cd blogposts`
* `go mod init github.com/{your-name}/blogposts`
* `touch blogposts_test.go`

```go
package blogposts_test

import (
	"testing"
	"testing/fstest"
)

func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts := blogposts.NewPostsFromFS(fs)

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}

```

Merk op dat het pakket van onze test `blogposts_test` is. Onthoud dat wanneer TDD goed wordt toegepast, we een consumentgerichte aanpak hanteren: we willen geen interne details testen, omdat consumenten daar niets om geven. Door `_test` toe te voegen aan de beoogde pakketnaam, hebben we alleen toegang tot geëxporteerde onderdelen van ons pakket - net als een echte gebruiker van ons pakket.

We hebben [`testing/fstest`](https://golang.org/pkg/testing/fstest/) geïmporteerd, wat ons toegang geeft tot het [`fstest.MapFS`](https://golang.org/pkg/testing/fstest/#MapFS)-type. Ons nep-bestandssysteem zal `fstest.MapFS` aan ons pakket doorgeven.

> Een MapFS is een eenvoudig in-memory bestandssysteem voor gebruik in tests. Het wordt weergegeven als een map van padnamen (argumenten voor Open) naar informatie over de bestanden of mappen die ze vertegenwoordigen.

Dit voelt eenvoudiger dan het bijhouden van een map met testbestanden en de test wordt sneller uitgevoerd.

Ten slotte hebben we het gebruik van onze API vanuit het oogpunt van de consument vastgelegd en gecontroleerd of het het juiste aantal berichten genereert.

## Probeer de test uit te voeren

```
./blogpost_test.go:15:12: undefined: blogposts
```

## Schrijf de minimale hoeveelheid code om de test uit te voeren met een _falend test resultaat_

Het pakket bestaat niet. Maak een nieuw bestand `blogposts.go` aan en plaats het `package blogposts` in je testbestand. Je moet dat pakket vervolgens importeren in je tests. Voor mij zien de imports er nu zo uit (let op de github naamgeving):

```go
import (
	blogposts "github.com/quii/learn-go-with-tests/reading-files"
	"testing"
	"testing/fstest"
)
```

De tests zullen nu weer niet worden gecompileerd, omdat ons nieuwe pakket geen `NewPostsFromFS`-functie heeft die een soort verzameling retourneert.

```
./blogpost_test.go:16:12: undefined: blogposts.NewPostsFromFS
```

Dit dwingt ons om het skelet van onze functie te maken om de test uit te voeren. Denk er op dit punt niet te veel over na; we proberen alleen een test uit te voeren en ervoor te zorgen dat deze faalt zoals verwacht. Als we deze stap overslaan, doen we mogelijk aannames over de oplossing en schrijven we een test die niet nuttig is.

```go
package blogposts

import "testing/fstest"

type Post struct {
}

func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return nil
}
```

De test zal nu correct falen:

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:48: got 0 posts, wanted 2 posts
```

## Schrijf genoeg code om de test te laten slagen

We zouden dit kunnen '[slijmen](https://deniseyu.github.io/leveling-up-tdd/)' om het te laten slagen:

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return []Post{{}, {}}
}
```

Maar, zoals Denise Yu schreef:

> Sliming is handig om je object een 'skelet' te geven. Het ontwerpen van een interface en het uitvoeren van logica zijn twee aspecten, en door sliming-tests strategisch te maken, kun je je op één aspect tegelijk concentreren.

We hebben onze structuur al. Wat doen we dan in plaats daarvan?

Omdat we de scope hebben beperkt, hoeven we alleen nog maar de directory te lezen en een bericht te maken voor elk bestand dat we tegenkomen. We hoeven ons nog geen zorgen te maken over het openen en parsen van bestanden.

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

[`fs.ReadDir`](https://golang.org/pkg/io/fs/#ReadDir) leest een directory in een gegeven `fs.FS` en retourneert [`[]DirEntry`](https://app.gitbook.com/u/0vinjXPm7bXI716vVEHCE1JF7q02).

Ons ideale wereldbeeld is al in duigen gevallen, omdat er fouten kunnen ontstaan die we hier niet afvangen. Maar bedenk dat we ons nu vooral richten op _het laten slagen van de test_, niet op het veranderen van het ontwerp. Daarom negeren we de fout voor nu.

De rest van de code is eenvoudig: herhaal de stappen over de items, maak een `Post` voor elk item en retourneer de slice.

## Refactor

Hoewel onze tests succesvol zijn, kunnen we ons nieuwe pakket niet buiten deze context gebruiken, omdat het gekoppeld is aan een concrete implementatie `fstest.MapFS`. Maar dat hoeft niet. Wijzig het argument in onze functie `NewPostsFromFS` om de interface van de standaardbibliotheek te accepteren.

```go
func NewPostsFromFS(fileSystem fs.FS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

Voer de tests opnieuw uit: alles zou moeten werken.

### Fout afhandeling

We hebben de foutafhandeling eerder geparkeerd toen we ons concentreerden op het werkend maken van het happy-path. Voordat we verder gaan met itereren op de functionaliteit, moeten we erkennen dat er fouten kunnen optreden bij het werken met bestanden. Naast het lezen van de directory kunnen we ook problemen tegenkomen bij het openen van individuele bestanden. Laten we onze API aanpassen (eerst via onze tests, uiteraard) zodat deze een `error` kan retourneren.

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts, err := blogposts.NewPostsFromFS(fs)

	if err != nil {
		t.Fatal(err)
	}

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

Voer de test uit: er zou een foutmelding moeten verschijnen over het verkeerde aantal retourwaarden. Het is eenvoudig om de code te repareren.

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts, nil
}
```

Dit zorgt ervoor dat de test slaagt. De TDD-beoefenaar in jou zal zich misschien irriteren aan het feit dat we geen falende test zagen voordat we de code schreven om de fout vanuit `fs.ReadDir` uit te dragen. Om dit "correct" te doen, hebben we een nieuwe test nodig waarbij we een falende `fs.FS` test-dubbel injecteren om `fs.ReadDir` een `error` te laten retourneren.

```go
type StubFailingFS struct {
}

func (s StubFailingFS) Open(name string) (fs.File, error) {
	return nil, errors.New("oh no, i always fail")
}
```

```go
// later
_, err := blogposts.NewPostsFromFS(StubFailingFS{})
```

Dit zou vertrouwen moeten geven in onze aanpak. De interface die we gebruiken heeft één methode, waardoor het maken van test-dubbels om verschillende scenario's te testen eenvoudig is.

In sommige gevallen is het testen van foutbehandeling de meest praktische oplossing, maar in ons geval doen we niets _interessants_ met de fout; we propageren hem alleen maar. Het is dus niet de moeite waard om een ​​nieuwe test te schrijven.

Logischerwijs zullen we in de volgende stappen ons `Post` type uitbreiden, zodat het bruikbare gegevens bevat.

## Schrijf eerst je test

We beginnen met de eerste regel in het voorgestelde blogpostschema: het titelveld.

We moeten de inhoud van de testbestanden aanpassen, zodat deze overeenkomt met wat is opgegeven. Pas dan kunnen we vaststellen dat de bestanden correct zijn geparseerd.

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("Title: Post 1")},
		"hello-world2.md": {Data: []byte("Title: Post 2")},
	}

	// rest van de testcode ingekort voor de overzicht
	got := posts[0]
	want := blogposts.Post{Title: "Post 1"}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

## Probeer de test uit te voeren

```
./blogpost_test.go:58:26: unknown field 'Title' in struct literal of type blogposts.Post
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Voeg het nieuwe veld toe aan ons `Post`-type, zodat de test kan worden uitgevoerd

```go
type Post struct {
	Title string
}
```

Voer de test opnieuw uit en je zou een duidelijke, falende test moeten krijgen

```
=== RUN   TestNewBlogPosts
=== RUN   TestNewBlogPosts/parses_the_post
    blogpost_test.go:61: got {Title:}, want {Title:Post 1}
```

## Schrijf genoeg code om de test te laten slagen

We moeten elk bestand openen en vervolgens de titel eruit halen

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f)
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()

	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Houd er rekening mee dat het op dit moment niet onze bedoeling is om elegante code te schrijven, maar om tot een punt te komen waarop we werkende software hebben.

Hoewel dit als een kleine stap vooruit voelt, moesten we toch behoorlijk wat code schrijven en een aantal aannames doen met betrekking tot foutafhandeling. Dit is een moment waarop je met je collega's moet overleggen om de beste aanpak te bepalen.

Dankzij de iteratieve aanpak kregen we snel feedback en weten we dat ons begrip van de eisen nog niet volledig is.

`fs.FS` biedt ons een manier om een ​​bestand erin op naam te openen met de `Open`-methode. Van daaruit lezen we de gegevens uit het bestand en voorlopig hebben we geen geavanceerde parsing nodig. We hoeven alleen de tekst `Title:` te verwijderen door de string te 'slicen'.

## Refactor

Door de 'open het bestand code' te scheiden van de 'code voor het verwerken van de bestandsinhoud' wordt de code eenvoudiger te begrijpen en te gebruiken.

```go
func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile fs.File) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Wanneer je nieuwe functies of methoden refactoren, denk dan goed na over de argumenten. Je bent hier aan het ontwerpen en je kunt goed nadenken over wat geschikt is, omdat je tests hebt die voldoen aan de eisen. Denk na over koppeling en samenhang. In dit geval moet je jezelf het volgende afvragen:

> Moet `newPost` gekoppeld worden aan een `fs.File`? Gebruiken we alle methoden en data van dit type? Wat hebben we _echt_ nodig?

In ons geval gebruiken we het alleen als argument voor `io.ReadAll`, waarvoor een `io.Reader` nodig is. We moeten de koppeling in onze functie dus losmaken en om een ​​`io.Reader` vragen.

```go
func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Je kunt een soortgelijk argument gebruiken voor onze `getPost`-functie, die een `fs.DirEntry`-argument accepteert, maar simpelweg `Name()` aanroept om de bestandsnaam op te halen. We hebben dat allemaal niet nodig; laten we loskoppelen van dat type en de bestandsnaam als een string doorgeven. Hier is de volledig gerefactoriseerde code:

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f.Name())
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, fileName string) (Post, error) {
	postFile, err := fileSystem.Open(fileName)
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Vanaf nu kunnen de meeste van onze inspanningen netjes in `newPost` worden ondergebracht. De zorgen over het openen en itereren van bestanden zijn voorbij en we kunnen ons nu richten op het extraheren van de gegevens voor ons `Post`-type. Hoewel technisch gezien niet noodzakelijk, zijn bestanden een handige manier om gerelateerde zaken logisch te groeperen. Daarom heb ik het `Post`-type en `newPost` verplaatst naar een nieuw bestand `post.go`.

### Test helper

We moeten ook goed op onze tests letten. We gaan veel beweringen doen over `Posts`, dus we moeten wat code schrijven om dat te ondersteunen

```go
func assertPost(t *testing.T, got blogposts.Post, want blogposts.Post) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

```go
assertPost(t, posts[0], blogposts.Post{Title: "Post 1"})
```

## Schrijf eerst je test

Laten we onze test verder uitbreiden om de volgende regel uit het bestand te halen, de beschrijving. Totdat de test slaagt, zou het nu comfortabel en vertrouwd moeten aanvoelen.

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1`
		secondBody = `Title: Post 2
Description: Description 2`
	)

	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte(firstBody)},
		"hello-world2.md": {Data: []byte(secondBody)},
	}

	// rest van de testcode ingekort voor overzicht
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
	})

}
```

## Probeer de test uit te voeren

```
./blogpost_test.go:47:58: unknown field 'Description' in struct literal of type blogposts.Post
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Voeg het nieuwe veld toe aan `Post`.

```go
type Post struct {
	Title       string
	Description string
}
```

De test zou nu moeten werken, en falen.

```
=== RUN   TestNewBlogPosts
    blogpost_test.go:47: got {Title:Post 1
        Description: Description 1 Description:}, want {Title:Post 1 Description:Description 1}
```

## Schrijf genoeg code om de test te laten slagen

De standaardbibliotheek heeft een handige functie waarmee je regel voor regel door gegevens kunt scannen; [`bufio.Scanner`](https://golang.org/pkg/bufio/#Scanner)

> Scanner biedt een handige interface voor het lezen van gegevens, zoals een bestand met door nieuwe regels gescheiden tekst.

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	scanner.Scan()
	titleLine := scanner.Text()

	scanner.Scan()
	descriptionLine := scanner.Text()

	return Post{Title: titleLine[7:], Description: descriptionLine[13:]}, nil
}
```

Het handige is dat je ook een `io.Reader` nodig hebt om alles te lezen (nogmaals bedankt, loose-coupling), dus we hoeven onze functieargumenten niet te wijzigen.

Roep `Scan` aan om een ​​regel te lezen en haal vervolgens de gegevens op met behulp van `Text`.

Deze functie kan nooit een `error` retourneren. Het zou verleidelijk zijn om hem op dit punt uit het retourtype te verwijderen, maar we weten dat we later ongeldige bestandsstructuren moeten verwerken, dus we kunnen hem net zo goed laten staan.

## Refactor

We hebben herhaling rond het scannen van een regel en het vervolgens lezen van de tekst. We weten dat we deze bewerking minstens nog één keer gaan uitvoeren, het is een simpele refactoring om door te voeren en aan de DRY-principes te houden, dus laten we daarmee beginnen.

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[7:]
	description := readLine()[13:]

	return Post{Title: title, Description: description}, nil
}
```

Dit heeft nauwelijks regels code bespaard, maar dat is zelden de bedoeling van refactoring. Wat ik hier probeer te doen, is het _wat_ van het _hoe_ van het lezen scheiden om de code wat duidelijker te maken voor de lezer.

De magische getallen 7 en 13 zijn weliswaar voldoende, maar ze zijn niet erg beschrijvend.

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
)

func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[len(titleSeparator):]
	description := readLine()[len(descriptionSeparator):]

	return Post{Title: title, Description: description}, nil
}
```

Nu ik met mijn creatieve refactoring-geest naar de code staar, wil ik proberen onze readLine-functie de tag te laten verwijderen. Er is ook een beter leesbare manier om een ​​voorvoegsel van een string te verwijderen met de functie `strings.TrimPrefix`.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
	}, nil
}
```

Misschien vind je dit idee wel of niet leuk, maar ik wel. Het punt is dat we in de refactoring-status vrij zijn om met de interne details te spelen, en je kunt je tests blijven uitvoeren om te controleren of alles nog steeds correct werkt. We kunnen altijd teruggaan naar eerdere statussen als we niet tevreden zijn. De TDD-aanpak geeft ons de vrijheid om regelmatig met ideeën te experimenteren, zodat we meer kansen hebben om geweldige code te schrijven.

De volgende vereiste is het extraheren van de tags van het bericht. Als je meedoet, raad ik je aan om dit zelf te proberen voordat je verder leest. Je zou nu een goed, iteratief ritme moeten hebben en je zelfverzekerd genoeg moeten voelen om de volgende regel te extraheren en de data te parseren.

Voor de leesbaarheid, zal ik de TDD-stappen niet doornemen, maar hier is de test met toegevoegde tags.

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker`
	)

	// rest of test code cut for brevity
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
	})
}
```

Je houdt jezelf alleen maar voor de gek als je gewoon kopieert en plakt wat ik schrijf. Om er zeker van te zijn dat we allemaal op dezelfde pagina zitten, is hier mijn code, inclusief het extraheren van de tags.

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
	tagsSeparator        = "Tags: "
)

func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
	}, nil
}
```

Hopelijk geen verrassingen hier. We konden `readMetaLine` hergebruiken om de volgende regel voor de tags te krijgen en deze vervolgens opsplitsen met `strings.Split`.

De laatste stap op ons happy path is het extraheren van de inhoud.

Hierbij een herinnering aan het voorgestelde bestandsformaat.

```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

We hebben de eerste drie regels al gelezen. Vervolgens moeten we nog één regel lezen, deze weglaten en dan bevat de rest van het bestand de tekst van het bericht.

## Schrijf eerst je test

Wijzig de testgegevens zodat ze een scheidingsteken bevatten en een hoofdtekst met een paar nieuwe regels om te controleren of alle inhoud is vastgelegd.

```go
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go
---
Hello
World`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker
---
B
L
M`
	)
```

Voeg toe aan onze bewering zoals de anderen

```go
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
		Body: `Hello
World`,
	})
```

## Probeer de test uit te voeren

```
./blogpost_test.go:60:3: unknown field 'Body' in struct literal of type blogposts.Post
```

Zoals we al verwachtte

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Voeg `Body` toe aan `Post` en de test zou nu moeten falen.

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:38: got {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:}, want {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:Hello
        World}
```

## Schrijf genoeg code om de test te laten slagen

1. Scan de volgende regel om het `---` scheidingsteken te negeren.
2. Blijf scannen tot er niets meer is om te scannen.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	title := readMetaLine(titleSeparator)
	description := readMetaLine(descriptionSeparator)
	tags := strings.Split(readMetaLine(tagsSeparator), ", ")

	scanner.Scan() // ignore a line

	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	body := strings.TrimSuffix(buf.String(), "\n")

	return Post{
		Title:       title,
		Description: description,
		Tags:        tags,
		Body:        body,
	}, nil
}
```

* `scanner.Scan()` retourneert een `bool` die aangeeft of er nog meer gegevens zijn om te scannen, zodat we dit kunnen gebruiken met een `for`-lus om de gegevens tot het einde te blijven lezen.
* Na elke `Scan()` schrijven we de gegevens naar de buffer met behulp van `fmt.Fprintln`. We gebruiken de versie die een nieuwe regel toevoegt, omdat de scanner de nieuwe regels uit elke regel verwijdert, maar we moeten ze behouden.
* Om bovenstaande redenen moeten we de laatste nieuwe regel afkappen, zodat er geen nieuwe regel achteraan komt.

## Refactor

Door het idee om de rest van de gegevens in een functie te verwerken, begrijpen toekomstige lezers snel _wat_ er in `newPost` gebeurt, zonder dat ze zich druk hoeven te maken over de implementatiedetails.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
		Body:        readBody(scanner),
	}, nil
}

func readBody(scanner *bufio.Scanner) string {
	scanner.Scan() // ignore a line
	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	return strings.TrimSuffix(buf.String(), "\n")
}
```

## Verder itereren

We hebben onze 'rode draad' van functionaliteit gemaakt en nemen de kortste route om ons happy path te bereiken. Maar er moet uiteraard nog een lange weg worden afgelegd voordat dit pakket klaar is voor productie.

We hebben geen rekening gehouden met:

* wanneer de bestandsindeling niet correct is
* het bestand geen `.md` bestand is
* wat als de volgorde van de metadata velden anders is? Moet dat toegestaan ​​zijn? Moeten we ermee kunnen omgaan?

Cruciaal is echter dat we werkende software hebben en onze interface hebben gedefinieerd. Bovenstaande zijn slechts verdere iteraties, meer tests om te schrijven en het gedrag van de software te sturen. Om bovenstaande te ondersteunen, hoeven we ons ontwerp niet aan te passen, alleen de implementatiedetails.

Door ons te blijven richten op het doel, nemen we de belangrijke beslissingen en toetsen we deze aan het gewenste gedrag. We raken niet te veel bezig met zaken die het algehele ontwerp niet beïnvloeden.

## Samenvattend

`fs.FS` en de andere wijzigingen in Go 1.16 bieden ons een aantal elegante manieren om gegevens uit bestandssystemen te lezen en deze eenvoudig te testen.

Als je de code "echt" wilt uitproberen:

* Maak een `cmd`-map binnen het project en voeg een `main.go`-bestand toe
* Voer daar de onderstaande code in

```go
import (
	blogposts "github.com/quii/fstest-spike"
	"log"
	"os"
)

func main() {
	posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
	if err != nil {
		log.Fatal(err)
	}
	log.Println(posts)
}
```

* Voeg een aantal markdown-bestanden toe aan een `posts`-map en voer het programma uit!

Let op de gelijkenis tussen de productiecode

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

En de tests

```go
posts, err := blogposts.NewPostsFromFS(fs)
```

Dan _voelt_ consument gestuurde, top-down TDD als de juiste keuze.

Een gebruiker van ons pakket kan onze tests bekijken en snel op de hoogte raken van wat het moet doen en hoe het te gebruiken. Als beheerders kunnen we erop _vertrouwen dat onze tests nuttig zijn_, omdat ze vanuit het perspectief van de gebruiker zijn geschreven. We testen geen implementatiedetails of andere bijkomstige details, dus we kunnen er redelijk zeker van zijn dat onze tests ons zullen helpen in plaats van hinderen bij het refactoren.

Door gebruik te maken van goede software engineering-praktijken, zoals [**dependency injection**](dependency-injection.md), is onze code eenvoudig te testen en hergebruiken.

Geef bij het maken van pakketten, zelfs als ze alleen intern voor je project zijn, de voorkeur aan een top-down, consumentgerichte aanpak. Dit voorkomt dat je te veel nadenkt over ontwerpen en abstracties maakt die je misschien niet eens nodig hebt, en zorgt ervoor dat de tests die je schrijft nuttig zijn.

Dankzij de iteratieve aanpak bleef elke stap klein en de voortdurende feedback zorgde ervoor dat we onduidelijke vereisten mogelijk eerder ontdekten dan met andere, meer ad-hocbenaderingen.

### Schrijven?

Het is belangrijk om te weten dat deze nieuwe functies alleen bewerkingen voor het lezen van bestanden bevatten. Als je werk schrijfwerk vereist, zul je elders moeten zoeken. Houd er rekening mee dat je moet blijven nadenken over wat de standaardbibliotheek momenteel biedt. Als je data schrijft, kun je waarschijnlijk beter gebruikmaken van bestaande interfaces zoals `io.Writer` om je code los gekoppeld en herbruikbaar te houden.

### Verder lezen

* Dit was een lichte introductie tot `io/fs`. [Ben Congdon heeft een uitstekende tekst geschreven](https://benjamincongdon.me/blog/2021/01/21/A-Tour-of-Go-116s-iofs-package/) dat erg behulpzaam was bij het schrijven van dit hoofdstuk.
* [Discussie over de bestandssysteem interfaces](https://github.com/golang/go/issues/41190)
