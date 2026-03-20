# Templates

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/blogrenderer)

We leven in een wereld waarin iedereen webapplicaties wil bouwen met het nieuwste frontend-framework, gebouwd op gigabytes aan getranspileerde JavaScript, werkend met een Byzantijns bouwsysteem; [maar misschien is dat niet altijd nodig](https://quii.dev/The_Web_I_Want).

Ik denk dat de meeste Go-ontwikkelaars waarde hechten aan een eenvoudige, stabiele en snelle toolchain, maar de frontend-wereld schiet hier vaak tekort.

Veel websites hebben helemaal geen [SPA](https://en.wikipedia.org/wiki/Single-page_application) oplossing nodig. **HTML en CSS zijn fantastische manieren om content te leveren** en je kunt Go gebruiken om een website te bouwen die content in HTML levert.

Als je toch nog dynamische elementen wilt hebben, kun je er nog wat client-side JavaScript aan toevoegen. Je kunt ook experimenteren met [Hotwire](https://hotwired.dev), waarmee je een dynamische ervaring kunt bieden met een server-side benadering.

Je kunt je HTML in Go genereren met behulp van [`fmt.Fprintf`](https://pkg.go.dev/fmt#Fprintf), maar in dit hoofdstuk leer je dat de standaardbibliotheek van Go een aantal tools bevat om HTML op een eenvoudigere en beter te onderhouden manier te genereren. Je leert ook effectievere manieren om dit soort code te testen, die je misschien nog niet eerder bent tegengekomen.

## Wat we gaan bouwen

In het hoofdstuk [Bestanden lezen](reading-files.md) hebben we code geschreven die [`fs.FS`](https://pkg.go.dev/io/fs) (een bestandssysteem) gebruikt en een deel van `Post` retourneert voor elk markdown-bestand dat het tegenkomt.

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

Hier is hoe we `Post` definiëren

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

Hier is een voorbeeld van een van de markdown-bestanden die verwerkt kunnen worden.

```markdown
Title: Welcome to my blog
Description: Introduction to my blog
Tags: cooking, family, live-laugh-love
---
# First recipe!
Welcome to my **amazing recipe blog**. I am going to write about my family recipes, and make sure I write a long, irrelevant and boring story about my family before you get to the actual instructions.
```

Nu we onze blogsoftware verder ontwikkelen, gaan we deze data gebruiken om HTML te genereren die onze webserver kan retourneren als reactie op HTTP-verzoeken.

Voor onze blog willen we twee soorten pagina's genereren:

1. **View post**. Geeft een specifiek bericht weer. Het `Body`-veld in `Post` is een tekenreeks met markdown, dus die moet worden omgezet naar HTML.
2. **Index**. Geeft een overzicht van alle berichten, met hyperlinks om het specifieke bericht te bekijken.

We willen ook een consistente uitstraling op onze hele site. Daarom gebruiken we voor elke pagina de gebruikelijke HTML-elementen, zoals `<html>` en een `<head>` met links naar CSS-stijlblad en alles wat we verder nog nodig hebben.

Wanneer je blogsoftware bouwt, heb je verschillende opties wat betreft de manier waarop je HTML opbouwt en naar de browser van de gebruiker verzendt.

We ontwerpen onze code zo dat deze een `io.Writer` accepteert. Dit betekent dat de aanroeper van onze code de flexibiliteit heeft om:

* output naar een [os.File](https://pkg.go.dev/os#File) te schrijven, zodat deze statisch kunnen worden weergegeven.
* De HTML rechtstreeks naar een [`http.ResponseWriter`](https://pkg.go.dev/net/http#ResponseWriter) te schrijven
* Gewoon naar iets anders weg te schrijven! Zolang het `io.Writer` implementeert, kan de aanroeper HTML genereren uit een `Post`

## Schrijf eerst je test

Zoals altijd is het belangrijk om na te denken over de vereisten voordat je  te snel in de oplossing duikt. Hoe kunnen we deze vrij grote set vereisten opsplitsen in een kleine, haalbare stap waar we ons eerst op kunnen richten?

Naar mijn mening heeft het daadwerkelijk bekijken van content een hogere prioriteit dan een indexpagina. We zouden dit product kunnen lanceren en directe links naar onze fantastische content kunnen delen. Een indexpagina die niet naar de daadwerkelijke content kan linken, is niet nuttig.

Toch voelt het renderen van een bericht zoals eerder beschreven nog steeds groot aan. Al het HTML-materiaal, het omzetten van de body-markdown naar HTML, het weergeven van tags, enzovoort.

Op dit moment maak ik me niet al te druk om de specifieke markup, en een eenvoudige eerste stap zou zijn om te controleren of we de titel van het bericht als een `<h1>` kunnen weergeven. Dit _voelt_ als die kleinste eerste stap die ons een stap verder kan brengen.

```go
package blogrenderer_test

import (
	"bytes"
	"github.com/quii/learn-go-with-tests/blogrenderer"
	"testing"
)

func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>`
		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
}
```

Doordat we ervoor hebben gekozen om een ​​`io.Writer` te gebruiken, wordt testen ook eenvoudiger. In dit geval schrijven we naar een [`bytes.Buffer`](https://pkg.go.dev/bytes#Buffer) waarvan we de inhoud later kunnen bekijken.

## Probeer de test uit te voeren

Als je de voorgaande hoofdstukken van dit boek hebt gelezen, zou je hier nu goed in geoefend moeten zijn. Je zult de test niet kunnen uitvoeren omdat we het pakket of de `Render`-functie niet hebben gedefinieerd. Probeer zelf de compilermeldingen te volgen en bereik een staat waarin je de test kunt uitvoeren en ziet dat deze faalt met een duidelijke melding.

Het is erg belangrijk dat je je tests oefent met falen. Je zult jezelf dankbaar zijn als je 6 maanden later per ongeluk een test laat falen, omdat je _nu_ de moeite hebt genomen om te controleren of deze faalt met een duidelijke melding.

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Dit is de minimale code om de test uit te voeren.

```go
package blogrenderer

// if you're continuing from the read files chapter, you shouldn't redefine this
type Post struct {
	Title, Description, Body string
	Tags                     []string
}

func Render(w io.Writer, p Post) error {
	return nil
}
```

De test moet aangeven dat een lege string niet gelijk is aan wat we willen.

## Schrijf genoeg code om de test te laten slagen

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1>", p.Title)
	return err
}
```

Vergeet niet dat softwareontwikkeling in de eerste plaats een leeractiviteit is. Om te ontdekken en te leren terwijl we werken, moeten we op een manier werken die ons regelmatige, hoogwaardige feedback oplevert. De makkelijkste manier om dat te doen, is door in kleine stapjes te werken.

We maken ons dus op dit moment geen zorgen over het gebruik van templatebibliotheken. Je kunt prima HTML genereren met "normale" stringtemplates. Door het templategedeelte over te slaan, kunnen we een klein beetje bruikbaar gedrag valideren en hebben we een klein beetje ontwerpwerk gedaan voor de API van ons pakket.

## Refactor

Er valt nog niet veel te refactoren, dus laten we doorgaan naar de volgende iteratie

## Schrijf eerst je test

Nu we een zeer basale versie hebben die werkt, kunnen we itereren op de test om de functionaliteit uit te breiden. In dit geval door meer informatie uit de `Post` te renderen.

```go
	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>
<p>This is a description</p>
Tags: <ul><li>go</li><li>tdd</li></ul>`

		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
```

Merk op dat het schrijven hiervan _onhandig_ aanvoelt. Het zien van al die markup in de test voelt niet fijn, en we hebben nog niet eens de body toegevoegd, of de HTML die we nodig hebben met alle `<head>`-content en alle pagina-indelingen die we nodig hebben.

Laten we de pijn desondanks _voorlopig_ maar accepteren.

## Probeer de test uit te voeren

Er zou een foutmelding moeten verschijnen, met de melding dat de string die we verwachten niet aanwezig is. De beschrijving en tags worden namelijk niet weergegeven.

## Schrijf genoeg code om de test te laten slagen

Probeer dit zelf te doen in plaats van de code te kopiëren. Je zult merken dat het _een beetje vervelend_ is om deze test te laten slagen! Toen ik het probeerde, kreeg ik deze foutmelding.

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:32: got '<h1>hello world</h1><p>This is a description</p><ul><li>go</li><li>tdd</li></ul>' want '<h1>hello world</h1>
        <p>This is a description</p>
        Tags: <ul><li>go</li><li></li></ul>'
```

Nieuwe regels! Wat maakt het uit? Nou, onze test wel, omdat hij een exacte stringwaarde vergelijkt. Zou dat moeten? Ik heb de nieuwe regels voorlopig verwijderd, gewoon om de test te laten slagen.

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1><p>%s</p>", p.Title, p.Description)
	if err != nil {
		return err
	}

	_, err = fmt.Fprint(w, "Tags: <ul>")
	if err != nil {
		return err
	}

	for _, tag := range p.Tags {
		_, err = fmt.Fprintf(w, "<li>%s</li>", tag)
		if err != nil {
			return err
		}
	}

	_, err = fmt.Fprint(w, "</ul>")
	if err != nil {
		return err
	}

	return nil
}
```

**Oei**. Niet de mooiste code die ik ooit geschreven heb, en we zitten nog maar in een heel vroeg stadium van de implementatie van onze markup. We hebben zoveel meer content en dingen op onze pagina nodig, dat we snel inzien dat deze aanpak niet geschikt is.

Belangrijk is echter dat we een voldoende hebben gehaald en dat de software werkt.

## Refactor

Nu we het vangnet hebben van een geslaagde test voor werkende code, kunnen we nadenken over het wijzigen van onze implementatieaanpak in de refactoringfase.

### Introductie tot templates

Go heeft twee templatepakketten: [text/template](https://pkg.go.dev/text/template) en [html/template](https://pkg.go.dev/html/template) en ze delen dezelfde interface. Beide maken het mogelijk om een ​​template en wat data te combineren tot een string.

Wat is het verschil tussen de text- en HTML-versie?

> Het pakket (html/template) implementeert datagestuurde templates voor het genereren van HTML-uitvoer die veilig is tegen code-injectie. Het biedt dezelfde interface als pakket text/template en dient in plaats van text/template te worden gebruikt wanneer de uitvoer HTML is. 

De templatetaal lijkt sterk op [Mustache](https://mustache.github.io) en stelt je in staat om dynamisch content te genereren op een zeer overzichtelijke manier, met een goede scheiding van verantwoordelijkheden. Vergeleken met andere templatetalen die je mogelijk hebt gebruikt, is het erg beperkt of "logicaloos", zoals Mustache het graag noemt. Dit is een belangrijke, **en bewuste** ontwerpbeslissing.

Hoewel we ons hier richten op het genereren van HTML, kun je, als je project complexe string-concatenaties en incantaties gebruikt, beter `text/template` gebruiken om je code schoon te houden.

### Terug naar de code

Hier is een sjabloon voor onze blog:

`<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`

Waar definiëren we deze string? Nou, we hebben een paar opties, maar om de stappen klein te houden, beginnen we gewoon met een gewone string.

```go
package blogrenderer

import (
	"html/template"
	"io"
)

const (
	postTemplate = `<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`
)

func Render(w io.Writer, p Post) error {
	templ, err := template.New("blog").Parse(postTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

We maken een nieuwe template met een naam en parseren vervolgens onze templatestring. We kunnen er vervolgens de `Execute`-methode op gebruiken en onze data doorgeven, in dit geval de `Post`.

De template vervangt zaken als `{{.Description}}` door de inhoud van `p.Description`. Templates bieden je ook een aantal programmeerprimitieven zoals `range` om waarden te loopen, en `if`. Je vindt meer informatie in de [text/template documentatie](https://pkg.go.dev/text/template).

_Dit zou een pure refactoring moeten zijn._ We zouden onze tests niet hoeven te wijzigen en ze zouden moeten blijven slagen. Belangrijk is dat onze code gemakkelijker te lezen is en veel minder vervelende foutafhandeling heeft.

Mensen klagen vaak over de omslachtigheid van foutafhandeling in Go, maar je zult merken dat je betere manieren kunt vinden om je code te schrijven zodat deze in de eerste plaats minder foutgevoelig is, zoals hier.

### Meer refactor werk

Het gebruik van `html/template` is zeker een verbetering, maar het als stringconstante in onze code gebruiken is niet geweldig:

* Het is nog steeds vrij moeilijk te lezen.
* Het is niet IDE/editor-vriendelijk. Geen syntax highlighting, mogelijkheid tot herformatteren, refactoren, enz.
* Het lijkt op HTML, maar je kunt er niet echt mee werken zoals met een "normaal" HTML-bestand.

Wat we willen, is onze templates in aparte bestanden plaatsen, zodat we ze beter kunnen ordenen en ermee kunnen werken alsof het HTML-bestanden zijn.

Maak een map aan met de naam "templates" en maak daarin een bestand aan met de naam `blog.gohtml`. Plak onze template in het bestand.

Wijzig nu onze code om de bestandssystemen te embedden met behulp van de [embeddingfunctionaliteit in go 1.16](https://pkg.go.dev/embed).


```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

Door een "bestandssysteem" in onze code te integreren, kunnen we meerdere sjablonen laden en vrij combineren. Dit is handig wanneer we renderinglogica willen delen tussen verschillende sjablonen, zoals een header bovenaan de HTML-pagina en een footer.

### Embed?

Embed werd terloops aangestipt in [Bestanden lezen](reading-files.md). De [documentatie van de standaardbibliotheek legt dit verder uit](https://pkg.go.dev/embed)

> Pakket embed biedt toegang tot bestanden die zijn ingesloten in het actieve Go-programma.
>
> Go-bronbestanden die "embed" importeren, kunnen de //go:embed-richtlijn gebruiken om een ​​variabele van het type string, \[]byte of FS te initialiseren met de inhoud van bestanden die tijdens het compileren uit de pakketdirectory of subdirectory's worden gelezen.

Waarom zouden we dit willen gebruiken? Het alternatief is dat we onze sjablonen _kunnen_ laden vanuit een "normaal" bestandssysteem. Dit betekent echter dat we ervoor moeten zorgen dat de sjablonen zich in het juiste bestandspad bevinden, waar we deze software ook willen gebruiken. In jouw werk heb je mogelijk verschillende omgevingen, zoals ontwikkeling, staging en live. Om dit te laten werken, moet je ervoor zorgen dat je sjablonen naar de juiste locatie worden gekopieerd.

Met embed worden de bestanden opgenomen in je Go-programma wanneer je het bouwt. Dit betekent dat zodra je je programma hebt gebouwd (wat je maar één keer zou moeten doen), de bestanden altijd voor je beschikbaar zijn.

Het handige is dat je niet alleen individuele bestanden kunt embedden, maar ook bestandssystemen; en dat bestandssysteem implementeert [io/fs](https://pkg.go.dev/io/fs), wat betekent dat je code zich geen zorgen hoeft te maken over het type bestandssysteem waarmee het werkt.

Als je echter verschillende sjablonen wilt gebruiken, afhankelijk van de configuratie, kun je ervoor kiezen om sjablonen op de meer conventionele manier vanaf schijf te laden.

## Volgende: Maak de sjabloon "leuk"

We willen niet dat onze template wordt gedefinieerd als een string van één regel. We willen hem kunnen uitspreiden om hem leesbaarder en gebruiksvriendelijker te maken, bijvoorbeeld:

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

Maar als we dit doen, mislukt onze test. Dit komt doordat onze test een heel specifieke string verwacht.

Maar eigenlijk maakt witruimte ons niet zoveel uit. Het onderhouden van deze test wordt een nachtmerrie als we de assertion string nauwgezet moeten blijven bijwerken telkens wanneer we kleine wijzigingen in de markup aanbrengen. Naarmate de template groeit, worden dit soort bewerkingen moeilijker te beheren en lopen de kosten ervan uit de hand.

## Introductie van Approval Tests

[Go Approval Tests](https://github.com/approvals/go-approval-tests)

> Met ApprovalTests kun je eenvoudig grotere objecten, strings en alles wat in een bestand kan worden opgeslagen (afbeeldingen, geluiden, CSV, enz.) testen.

Het idee is vergelijkbaar met "gouden" bestanden, of snapshot-testen. In plaats van het onhandig bijhouden van strings in een testbestand, kan de goedkeuringstool de uitvoer voor je vergelijken met een "goedgekeurd" bestand dat je hebt gemaakt. Je kopieert vervolgens eenvoudig de nieuwe versie als je deze goedkeurt. Voer de test opnieuw uit en je bent weer helemaal up-to-date.

Voeg een afhankelijkheid toe met `"github.com/approvals/go-approval-tests"` aan je project en bewerk de test als volgt:

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := blogrenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

De eerste keer dat je de test uitvoert, zal deze mislukken omdat we nog niets hebben goedgekeurd

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:29: Failed Approval: received does not match approved.
```

Er worden twee bestanden aangemaakt die er als volgt uitzien

* `renderer_test.TestRender.it_converts_a_single_post_into_HTML.received.txt`
* `renderer_test.TestRender.it_converts_a_single_post_into_HTML.approved.txt`

Het ontvangen bestand bevat de nieuwe, niet-goedgekeurde versie van de uitvoer. Kopieer deze naar het lege, goedgekeurde bestand en voer de test opnieuw uit.

Door de nieuwe versie te kopiëren, heb je de wijziging "goedgekeurd" en is de test geslaagd.

Om de workflow in actie te zien, bewerk je de sjabloon zoals we hebben besproken om deze leesbaarder te maken (maar die semantisch gezien hetzelfde is).

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

Voer de test opnieuw uit. Er wordt een nieuw "received" bestand gegenereerd omdat de uitvoer van onze code afwijkt van de goedgekeurde versie. Bekijk ze eens en als je tevreden bent met de wijzigingen, kopieer dan de nieuwe versie en voer de test opnieuw uit. Zorg ervoor dat je de goedgekeurde bestanden commit naar de broncode.

Deze aanpak maakt het beheren van wijzigingen in grote, lelijke dingen zoals HTML veel eenvoudiger. Je kunt een diff-tool gebruiken om de verschillen te bekijken en te beheren, en het houd je testcode overzichtelijker.

![Gebruik een diff tool om veranderingen te beheren](https://i.imgur.com/0MoNdva.png)

Dit is eigenlijk een vrij beperkte toepassing van goedkeuringstests, die een uiterst nuttig hulpmiddel zijn in je testarsenaal. [Emily Bache](https://twitter.com/emilybache) heeft een [interessante video waarin ze goedkeuringstests gebruikt om een ​​ongelooflijk uitgebreide set tests toe te voegen aan een complexe codebase die nul tests bevat](https://www.youtube.com/watch?v=zyM2Ep28ED8). "Combinatorial Testing" is zeker de moeite waard om te onderzoeken.

Nu we deze wijziging hebben doorgevoerd, hebben we nog steeds baat bij goed geteste code, maar de tests zullen niet al te veel in de weg zitten wanneer we aan de markup sleutelen.

### Doen we eigenlijk nog steeds TDD?

Een interessant neveneffect van deze aanpak is dat het ons afleidt van TDD. Natuurlijk _zou_ je de goedgekeurde bestanden handmatig kunnen bewerken naar de gewenste staat, je tests kunnen uitvoeren en vervolgens de sjablonen kunnen aanpassen zodat ze de resultaten opleveren die je hebt gedefinieerd.

Maar dat is gewoon onzin! TDD is een methode om werk te doen, met name ontwerpen; maar dat betekent niet dat we het dogmatisch voor **alles** moeten gebruiken.

Het belangrijkste is dat we het juiste hebben gedaan en TDD als een **ontwerptool** hebben gebruikt om de API van ons pakket te ontwerpen. Voor wijzigingen in sjablonen kan ons proces als volgt zijn:

* Een kleine wijziging in het sjabloon aanbrengen
* De goedkeuringstest uitvoeren
* De uitvoer visueel bekijken om te controleren of deze correct is
* De goedkeuring uitvoeren
* Herhalen

We moeten de waarde van werken in kleine, haalbare stappen echter niet opgeven. Probeer manieren te vinden om de wijzigingen klein te houden en blijf de tests opnieuw uitvoeren om echte feedback te krijgen op wat je doet.

Als we bijvoorbeeld de code _rondom_ de sjablonen gaan veranderen, kan dat natuurlijk een reden zijn om terug te gaan naar onze TDD-werkmethode.

## De markup uitbreiden

De meeste websites hebben rijkere HTML dan wij nu hebben. Om te beginnen een `html`-element, samen met een `head`, misschien ook wat `nav`. Meestal is er ook een idee voor een footer.

Als onze site verschillende pagina's gaat bevatten, willen we deze zaken op één plek definiëren om ervoor te zorgen dat onze site er consistent uitziet. Go-sjablonen ondersteunen ons bij het definiëren van secties die we vervolgens in andere sjablonen kunnen importeren.

Bewerk onze bestaande sjabloon om een `top` en `bottom` sjabloon te importeren.

```handlebars
{{template "top" .}}
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
{{template "bottom" .}}
```

Maak vervolgens `top.gohtml` aan met de volgende code

```handlebars
{{define "top"}}
<!DOCTYPE html>
<html lang="en">
<head>
    <title>My amazing blog!</title>
    <meta charset="UTF-8"/>
    <meta name="description" content="Wow, like and subscribe, it really helps the channel guys" lang="en"/>
</head>
<body>
<nav role="navigation">
    <div>
        <h1>Budding Gopher's blog</h1>
        <ul>
            <li><a href="/">home</a></li>
            <li><a href="about">about</a></li>
            <li><a href="archive">archive</a></li>
        </ul>
    </div>
</nav>
<main>
{{end}}
```

En `bottom.gohtml`

```handlebars
{{define "bottom"}}
</main>
<footer>
    <ul>
        <li><a href="https://twitter.com/quii">Twitter</a></li>
        <li><a href="https://github.com/quii">GitHub</a></li>
    </ul>
</footer>
</body>
</html>
{{end}}
```

(Natuurlijk, voel je vrij om elke gewenste opmaak te gebruiken!)

We moeten nu een specifieke sjabloon specificeren die moet worden uitgevoerd. Wijzig in de blogrenderer de opdracht `Execute` in `ExecuteTemplate`.

```go
if err := templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
	return err
}
```

Voer je test opnieuw uit. Er moet een nieuw "ontvangen" bestand worden aangemaakt en de test zal mislukken. Controleer het en als je tevreden bent, keur het goed door het over de oude versie heen te kopiëren (en plakken). Voer de test opnieuw uit en deze zou moeten slagen.

## Een excuus om te rommelen met benchmarking

Laten we, voordat we verdergaan, eens kijken wat onze code doet.


```go
func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
		return err
	}

	return nil
}
```

* De sjablonen parsen
* Gebruik het sjabloon om een ​​bericht te renderen naar een `io.Writer`

Hoewel de prestatie-impact van het opnieuw parsen van de sjablonen voor elk bericht in de meeste gevallen vrijwel verwaarloosbaar zal zijn, is de moeite om dit _niet_ te doen ook vrijwel verwaarloosbaar en zou de code ook wat op orde moeten brengen.

Om de impact te zien van het niet steeds opnieuw parsen, kunnen we de benchmarktool gebruiken om te kijken hoe snel onze functie is.

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	for b.Loop() {
		blogrenderer.Render(io.Discard, aPost)
	}
}
```

Op mijn computer zijn dit de resultaten

```
BenchmarkRender-8 22124 53812 ns/op
```

Om te voorkomen dat we de sjablonen steeds opnieuw moeten parsen, maken we een type dat het geparsed sjabloon bevat en dat een methode heeft om de rendering uit te voeren

```go
type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {

	if err := r.templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
		return err
	}

	return nil
}
```

Dit verandert de interface van onze code, dus we zullen onze test moeten bijwerken

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		t.Fatal(err)
	}

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := postRenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

En onze benchmark

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		b.Fatal(err)
	}

	for b.Loop() {
		postRenderer.Render(io.Discard, aPost)
	}
}
```

De test zou moeten slagen. Hoe zit het met onze benchmark?

`BenchmarkRender-8 362124 3131 ns/op`. De oude NS per op waren `53812 ns/op`, dus dit is een behoorlijke verbetering! Naarmate we andere methoden toevoegen om te renderen, bijvoorbeeld een indexpagina, zou dit de code moeten vereenvoudigen, omdat we de template-parsing niet hoeven te dupliceren.

## Terug naar het echte werk

Wat het renderen van berichten betreft, is het belangrijkste dat overblijft het renderen van de `Body`. Zoals je je herinnert, zou dat markdown moeten zijn die de auteur heeft geschreven, dus die moet naar HTML worden omgezet.

We laten dit als oefening over aan jou, de lezer. Je zou een Go-bibliotheek moeten kunnen vinden die dit voor je kan doen. Gebruik de goedkeuringstest om te valideren wat je doet.

### Over het testen van bibliotheken van derden

**Opmerking**. Maak je niet te veel zorgen over het expliciet testen van hoe een externe bibliotheek zich gedraagt ​​in unit tests.

Het schrijven van tests tegen code die je niet zelf beheert, is verspilling en verhoogt de onderhoudskosten. Soms kun je [dependency injection](dependency-injection.md) gebruiken om een ​​afhankelijkheid te controleren en het gedrag ervan te simuleren voor een test.

In dit geval beschouw ik het omzetten van de markdown naar HTML echter als een implementatiedetail van de rendering, en onze goedkeuringstests zouden ons voldoende vertrouwen moeten geven.

### Render index

De volgende functionaliteit die we gaan ontwikkelen, is het weergeven van een index, waarbij de berichten worden weergegeven als een geordende HTML-lijst.

We breiden onze API uit, dus we zetten onze TDD-hoed weer op.

## Schrijf eerst je test

Op het eerste gezicht lijkt een indexpagina eenvoudig, maar het schrijven van de test zet ons toch aan tot het maken van een aantal ontwerpkeuzes

```go
t.Run("it renders an index of posts", func(t *testing.T) {
	buf := bytes.Buffer{}
	posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

	if err := postRenderer.RenderIndex(&buf, posts); err != nil {
		t.Fatal(err)
	}

	got := buf.String()
	want := `<ol><li><a href="/post/hello-world">Hello World</a></li><li><a href="/post/hello-world-2">Hello World 2</a></li></ol>`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
})
```

1. We gebruiken het titelveld van `Post` als onderdeel van het pad van de URL, maar we willen eigenlijk geen spaties in de URL, dus vervangen we ze door koppeltekens.
2. We hebben een `RenderIndex`-methode toegevoegd aan onze `PostRenderer` die wederom een ​`io.Writer` en een deel van `Post` gebruikt.

Als we hier een test-after-goedkeuringstestaanpak hadden aangehouden, zouden we deze vragen niet in een gecontroleerde omgeving beantwoorden. **Tests geven ons ruimte om na te denken**.

## Probeer de test uit te voeren

```
./renderer_test.go:41:13: undefined: blogrenderer.RenderIndex
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return nil
}
```

Het bovenstaande zou de volgende testfout moeten opleveren

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
```

## Schrijf genoeg code om de test te laten slagen

Hoewel dit _voelt_ alsof het makkelijk zou moeten zijn, is het toch een beetje lastig. Ik heb het in meerdere stappen gedaan.

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

Ik wilde in eerste instantie geen gedoe met aparte templatebestanden, ik wilde het gewoon werkend krijgen. Ik zie het parsen en scheiden van templates vooraf als een refactoring die ik later kan doen.

Dit lukt niet, maar het komt in de buurt.

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "<ol><li><a href=\"/post/Hello%20World\">Hello World</a></li><li><a href=\"/post/Hello%20World%202\">Hello World 2</a></li></ol>" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
    --- FAIL: TestRender/it_renders_an_index_of_posts (0.00s)
```

Je ziet dat de templatecode de spaties in de `href`-attributen 'escaped'. We hebben een manier nodig om spaties door koppeltekens te vervangen. We kunnen niet zomaar door de `[]Post` heen loopen en ze in het geheugen vervangen, omdat we de spaties in de links nog steeds aan de gebruiker willen laten zien.

We hebben een paar opties. De eerste die we zullen verkennen, is het doorgeven van een functie aan onze template.

### Functies doorgeven aan sjablonen

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{sanitiseTitle .Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Funcs(template.FuncMap{
		"sanitiseTitle": func(title string) string {
			return strings.ToLower(strings.Replace(title, " ", "-", -1))
		},
	}).Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

_Voordat je een template parsed_ kun je een `template.FuncMap` aan je template toevoegen, waarmee je functies kunt definiëren die binnen je template kunnen worden aangeroepen. In dit geval hebben we een `sanitiseTitle`-functie gemaakt die we vervolgens binnen onze template aanroepen met `{{sanitiseTitle .Title}}`.

Dit is een krachtige functie. Door functies naar je template te sturen, kun je een aantal hele coole dingen doen, maar is dat wel de moeite waard? Terug naar de principes van Mustache en logica-loze templates: waarom pleitten ze voor logica-loze templates? **Wat is er mis met logica in templates?**

Zoals we hebben aangetoond, moesten we om onze templates te testen, _een heel ander soort testen introduceren_.

Stel je voor dat je een functie in een template introduceert met een paar verschillende permutaties van gedrag en randgevallen, **hoe ga je die dan testen**? Met dit huidige ontwerp kun je deze logica alleen testen door HTML te _renderen en strings te vergelijken_. Dit is geen gemakkelijke of verstandige manier om logica te testen, en zeker niet wat je zou willen voor _belangrijke_ business logic.

Hoewel de goedkeuringstesttechniek de onderhoudskosten van deze tests heeft verlaagd, zijn ze nog steeds duurder in onderhoud dan de meeste unittests die je schrijft. Ze zijn nog steeds gevoelig voor kleine markupwijzigingen die je aanbrengt, maar we hebben het beheer ervan eenvoudiger gemaakt. We moeten er nog steeds naar streven om onze code zo te ontwerpen dat we niet veel tests rond onze templates hoeven te schrijven, en proberen om aandachtspunten te scheiden, zodat alle logica die niet in onze renderingcode hoeft te zitten, goed gescheiden is.

Wat door Mustache beïnvloede template engines je bieden, is een nuttige beperking; probeer deze niet te vaak te omzeilen; **ga niet tegen de stroom in**. Omarm in plaats daarvan het idee van [view models](https://stackoverflow.com/a/11074506/3193), waarbij je specifieke typen construeert die de data bevatten die je moet renderen, op een manier die geschikt is voor de templatetaal.

Op deze manier kan alle belangrijke business logic die je gebruikt om die data te genereren, afzonderlijk worden getest, los van de rommelige wereld van HTML en templates.

### Scheiding van belangen

Dus, wat kunnen dan doen?

#### Voeg een methode toe aan `Post` en roep die vervolgens aan in de template.

We kunnen methoden aanroepen in onze templatecode voor de typen die we verzenden, dus we zouden een `SanitisedTitle`-methode kunnen toevoegen aan `Post`. Dit zou de template vereenvoudigen en we zouden deze logica desgewenst eenvoudig apart kunnen testen. Dit is waarschijnlijk de makkelijkste oplossing, hoewel niet per se de eenvoudigste.

Een nadeel van deze aanpak is dat dit nog steeds _view_ logica is. Het is niet interessant voor de rest van het systeem, maar het wordt nu onderdeel van de API voor een kerndomeinobject. Deze aanpak kan er na verloop van tijd toe leiden dat je [God Objects](https://en.wikipedia.org/wiki/God_object) creëert.

#### Maak een speciaal weergavemodeltype, zoals `PostViewModel`, met precies de gegevens die we nodig hebben.

In plaats van dat onze renderingcode gekoppeld is aan het domeinobject `Post`, gebruikt deze een weergavemodel.

```go
type PostViewModel struct {
	Title, SanitisedTitle, Description, Body string
	Tags                                     []string
}
```

Aanroepers van onze code zouden een mapping moeten maken van `[]Post` naar `[]PostView`, waarmee de `SanitizedTitle` wordt gegenereerd. Een manier om dit overzichtelijk te houden, is door een `func NewPostView(p Post) PostView` te gebruiken die de mapping inkapselt.

Dit zou onze renderingcode logischer maken en is waarschijnlijk de striktste scheiding van taken die we konden doen, maar het nadeel is een iets ingewikkelder proces om onze berichten te renderen.

Beide opties zijn prima, in dit geval ben ik geneigd om voor de eerste te kiezen. Naarmate het systeem zich verder ontwikkelt, moet je voorzichtig zijn met het toevoegen van steeds meer ad-hocmethoden om het renderen soepeler te laten verlopen; speciale weergavemodellen worden nuttiger wanneer de transformatie tussen het domeinobject en de weergave complexer wordt.

We kunnen onze methode dus toevoegen aan `Post`


```go
func (p Post) SanitisedTitle() string {
	return strings.ToLower(strings.Replace(p.Title, " ", "-", -1))
}
```

En dan kunnen we teruggaan naar een eenvoudigere wereld in onze renderingcode

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

## Refactor

De test zou eindelijk moeten slagen. We kunnen onze template nu naar een bestand verplaatsen (`templates/index.gohtml`) en deze één keer laden wanneer we onze renderer construeren.

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", p)
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}
```

Door meer dan één template in `templ` te parsen, moeten we nu `ExecuteTemplate` aanroepen en specificeren welke template we willen renderen. Hopelijk ben je het ermee eens dat de code die we hebben samengesteld er geweldig uitziet.

Er is een klein risico dat een bug ontstaat als iemand een van de templatebestanden hernoemt, maar onze snel uit te voeren unittests zouden dit snel opsporen.

Nu we tevreden zijn met het API-ontwerp van ons pakket en een aantal basisgedragingen hebben ontwikkeld met TDD, kunnen we onze test aanpassen om goedkeuringen te gebruiken.

```go
	t.Run("it renders an index of posts", func(t *testing.T) {
		buf := bytes.Buffer{}
		posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

		if err := postRenderer.RenderIndex(&buf, posts); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
```

Vergeet niet de test uit te voeren om te zien of het mislukt en keur de wijziging vervolgens goed.

Ten slotte kunnen we onze pagina-indeling toevoegen aan onze indexpagina:

```handlebars
{{template "top" .}}
<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>
{{template "bottom" .}}
```

Voer de test opnieuw uit, keur de wijziging goed en de index is klaar!

## De markdown-body renderen

Ik heb je aangemoedigd om het zelf te proberen. Dit is de aanpak die ik uiteindelijk heb gekozen.

```go
package blogrenderer

import (
	"embed"
	"github.com/gomarkdown/markdown"
	"github.com/gomarkdown/markdown/parser"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ    *template.Template
	mdParser *parser.Parser
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	extensions := parser.CommonExtensions | parser.AutoHeadingIDs
	parser := parser.NewWithExtensions(extensions)

	return &PostRenderer{templ: templ, mdParser: parser}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", newPostVM(p, r))
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}

type postViewModel struct {
	Post
	HTMLBody template.HTML
}

func newPostVM(p Post, r *PostRenderer) postViewModel {
	vm := postViewModel{Post: p}
	vm.HTMLBody = template.HTML(markdown.ToHTML([]byte(p.Body), r.mdParser, nil))
	return vm
}
```

Ik heb de uitstekende bibliotheek [gomarkdown](https://github.com/gomarkdown/markdown) gebruikt, die precies werkte zoals ik had gehoopt.

Als je dit zelf hebt geprobeerd, heb je mogelijk gemerkt dat de HTML in je body-render is 'escaped'. Dit is een beveiligingsfunctie van Go's html/template-pakket om te voorkomen dat schadelijke HTML van derden wordt uitgevoerd.

Om dit te omzeilen, moet je in het type dat je naar de render stuurt, je vertrouwde HTML omsluiten in [template.HTML](https://pkg.go.dev/html/template#HTML)

> HTML omsluit een bekend, veilig HTML-documentfragment. Het mag niet worden gebruikt voor HTML van derden, of HTML met niet-afgesloten tags of opmerkingen. De uitvoer van een degelijke HTML-sanitiser en een sjabloon die door dit pakket wordt geëscapete, zijn prima voor gebruik met HTML.
>
> Gebruik van dit type brengt een beveiligingsrisico met zich mee: de ingesloten content moet afkomstig zijn van een vertrouwde bron, aangezien deze letterlijk in de sjabloonuitvoer wordt opgenomen.

Dus maakte ik een **niet-geëxporteerd** viewmodel (`postViewModel`), omdat ik dit nog steeds zag als een intern implementatiedetail voor de rendering. Ik hoef dit niet apart te testen en ik wil niet dat het mijn API vervuilt.

Ik maak er een tijdens het renderen, zodat ik de `Body` kan parsen naar `HTMLBody` en vervolgens gebruik ik dat veld in de template om de HTML te renderen.

## Samenvattend

Als je de kennis uit het hoofdstuk [Bestanden lezen](reading-files.md) combineert met dit hoofdstuk, kun je gemakkelijk een goed geteste, eenvoudige, statische sitegenerator maken en je eigen blog beginnen. Zoek een paar CSS-tutorials en je kunt het er ook nog eens mooi uit laten zien.

Deze aanpak gaat verder dan blogs. Gegevens uit elke bron halen, of het nu een database, een API of een bestandssysteem is, en deze converteren naar HTML en terugsturen vanaf een server, is een eenvoudige techniek die al tientallen jaren bestaat. Mensen klagen graag over de complexiteit van moderne webontwikkeling, maar weet je zeker dat je jezelf die complexiteit niet alleen maar bezorgt?

Go is geweldig voor webontwikkeling, vooral als je goed nadenkt over wat je werkelijke vereisten zijn voor de website die je maakt. Het genereren van HTML op de server is vaak een betere, eenvoudigere en efficiëntere aanpak dan het maken van een "webapplicatie" met technologieën zoals React.

### Wat we geleerd hebben

* Hoe je HTML-sjablonen maakt en rendert.
* Hoe je sjablonen samenstelt en gerelateerde markup [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself) toevoegt en ons helpt een consistente look-and-feel te behouden.
* Hoe je functies in sjablonen opneemt en waarom je daar nog eens goed over na moet denken.
* Hoe je "goedkeuringstests" schrijft, waarmee we de grote, lelijke output van bijvoorbeeld sjabloonrenderers kunt testen.

### Over sjablonen zonder logica

Zoals altijd draait het hier om **scheiding van belangen**. Het is belangrijk dat we nadenken over de verantwoordelijkheden van de verschillende onderdelen van ons systeem. Te vaak lekken mensen belangrijke business logic in sjablonen, waardoor belangen door elkaar raken en systemen moeilijk te begrijpen, onderhouden en testen zijn.

### Niet alleen voor HTML

Onthoud dat go `text/template` heeft om andere soorten gegevens uit een sjabloon te genereren. Als je gegevens moet omzetten in een gestructureerde uitvoer, kunnen de technieken die in dit hoofdstuk worden beschreven nuttig zijn.

### Referenties en verder materiaal

* [John Calhoun's 'Learn Web Development with Go'](https://www.calhoun.io/intro-to-templates-p1-contextual-encoding/) bevat een aantal uitstekende artikelen over templates.
* [Hotwire](https://hotwired.dev) - Je kunt deze technieken gebruiken om Hotwire-webapplicaties te maken. Het is gebouwd door Basecamp/37Signals, dat voornamelijk Ruby-on-Rails apps bouwt, maar omdat het server-side is, kunnen we het met Go gebruiken.