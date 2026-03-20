# Refactoring stap, checklist voor beginners

Refactoring is een vaardigheid die, eenmaal voldoende geoefend, in de meeste gevallen een tweede natuur wordt.

De activiteit wordt vaak verward met ingrijpendere ontwerpwijzigingen, maar die staan los van elkaar. Het is nuttig om onderscheid te maken tussen refactoring en andere programmeeractiviteiten, omdat het me in staat stelt om helder en gedisciplineerd te werken.

## Refactoring versus andere activiteiten

Refactoring is gewoon het verbeteren van bestaande code en _niet het veranderen van gedrag_; tests zouden daarom niet hoeven te veranderen.

Daarom is het de derde stap in de TDD-cyclus. Zodra je een gedrag en een test hebt toegevoegd om dit te ondersteunen, zou refactoring een activiteit moeten zijn die geen wijziging in je testcode vereist. **Je doet iets anders** als je code "refactored" en tegelijkertijd tests moet wijzigen.

Veel zeer nuttige refactorings zijn eenvoudig te leren en gemakkelijk uit te voeren (je IDE automatiseert er vaak al veel volledig automatisch), maar na verloop van tijd hebben ze een enorme impact op de kwaliteit van ons systeem.

### Andere activiteiten, zoals "groot" ontwerp

> Dus ik verander het "echte" gedrag niet, maar ik moet mijn tests wel aanpassen? Wat is dat?

Stel dat je aan een type werkt en de kwaliteit van de code wilt verbeteren. *Refactoring zou niet moeten vereisen dat je de tests aanpast*, dus je kunt niet:

- Gedrag aanpassen
- Methodehandtekeningen aanpassen

...omdat je tests aan die twee dingen gekoppeld zijn, maar je kunt wel:

- Private methoden, velden en zelfs nieuwe typen en interfaces introduceren
- De interne werking van publieke methoden aanpassen

Wat als je de handtekening van een methode wilt aanpassen?

```go
func (b BirthdayGreeter) WishHappyBirthday(age int, firstname, lastname string, email Email) {
	// some fascinating emailing code
}
```

Mogelijk vindt je de lijst met argumenten te lang en wil je meer samenhang en betekenis aan de code toevoegen.

```go
func (b BirthdayGreeter) WishHappyBirthday(person Person)
```

Nou, je bent nu aan het **ontwerpen** en moet ervoor zorgen dat je voorzichtig te werk gaat. Als je dit niet gedisciplineerd doet, kun je een puinhoop maken van je code, de test erachter, *en* waarschijnlijk ook van de dingen die ervan afhankelijk zijn. Onthoud: het zijn niet alleen je tests die `WishHappyBirthday` gebruiken. Hopelijk wordt het ook door "echte" code gebruikt!

**Je zou deze wijziging eerst met een test moeten kunnen doorvoeren**. Je kunt erover twisten of dit een "gedrags"-wijziging is, maar je wilt dat je methode zich anders gedraagt.

Omdat dit een gedragsverandering is, pas je hier ook het TDD-proces toe. Een voordeel van TDD is dat het je een eenvoudige, veilige en herhaalbare manier biedt om gedragsverandering in je systeem door te voeren; waarom zou je het in deze situaties laten varen alleen maar omdat het *anders* aanvoelt?

In dit geval verander je je bestaande tests om het nieuwe type te gebruiken. De iteratieve, kleine stappen die je normaal gesproken met TDD zet om risico's te verminderen en discipline en duidelijkheid te brengen, zullen je ook in deze situaties helpen.

De kans is groot dat je meerdere tests hebt die `WishHappyBirthday` aanroepen. In deze scenario's raad ik aan om alle tests behalve één uit te schakelen, de wijziging door te voeren en vervolgens de rest van de tests naar eigen inzicht af te werken.

### Ingrijpend ontwerp

Ontwerp kan ingrijpender wijzigingen en uitgebreidere gesprekken vereisen en kent meestal een zekere mate van subjectiviteit. Het wijzigen van het ontwerp van onderdelen van je systeem is meestal een langer proces dan refactoring; desalniettemin moet je proberen risico's te beperken door na te denken over hoe je het in kleine stapjes kunt doen.

### Door de bomen het bos niet meer zien

> [Als iemand **door de bomen het bos niet meer ziet**, is hij of zij erg betrokken bij de details van iets en ziet daardoor niet wat belangrijk is aan het geheel.](https://www.collinsdictionary.com/dictionary/english/cant-see-the-wood-for-the-trees)

Het praten over de "grote" ontwerpproblemen is toegankelijker wanneer de **onderliggende code goed gefactoriseerd** is. Als jij en je collega's, elke keer als er een bestand geopend wordt, veel tijd moeten besteden aan het mentaal parsen van een wirwar aan code, hoe groot is dan de kans dat je nadenkt over het ontwerp van de code?

Daarom is **constante refactoring zo belangrijk in het TDD-proces**. Als we de kleine ontwerpproblemen niet aanpakken, zullen we het moeilijk vinden om het algehele ontwerp van ons uitgebreidere systeem te ontwikkelen.

Helaas wordt slecht gefactoriseerde code exponentieel slechter naarmate engineers complexiteit opstapelen op een wankel fundament.

## Beginnen met een mentale checklist

**Maak er een gewoonte van om elke TDD-cyclus een mentale checklist door te nemen.** Hoe meer je jezelf dwingt om te oefenen, hoe makkelijker het wordt. **Het is een vaardigheid die oefening vereist.** Onthoud dat al deze veranderingen geen enkele aanpassing in je tests zouden moeten vereisen.

Ik heb snelkoppelingen toegevoegd voor IntelliJ/GoLand, die mijn collega's en ik gebruiken. Wanneer ik een nieuwe engineer coach, moedig ik hem of haar aan om te proberen het spiergeheugen en de gewoonte te ontwikkelen om deze tools te gebruiken om snel en veilig te refactoren.

### Inline variabelen

Als u een variabele aanmaakt, alleen om deze door te geven aan een andere methode/functie:

```go
url := baseURL + "/user/" + id
res, err := client.Get(url)
```

Overweeg om het inline te gebruiken (`command+option+n`) *tenzij* de variabelenaam een waardevolle betekenis toevoegt.

```go
res, err := client.Get(baseURL + "/user/" + id)
```

Wees niet _te_ slim met inlining; het doel is niet om nul variabelen te hebben, maar in plaats daarvan belachelijke oneliners die niemand kan lezen. Als je een waarde een betekenisvolle naam kunt geven, is het misschien het beste om die zo te laten.

### DRY up waarden met geëxtraheerde variabelen

"Herhaal jezelf niet" (Don't Repeat Yourself: DRY). Gebruik je dezelfde waarde meerdere keren in een functie? Overweeg dan om een variabele te extraheren en vast te leggen in een betekenisvolle variabelenaam (`command+option+v`).

Dit verbetert de leesbaarheid en maakt het wijzigen van de waarde in de toekomst eenvoudiger, omdat je niet hoeft te onthouden om dezelfde waarde meerdere keren te wijzigen.

### DRY up dingen in het algemeen

[DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) heeft tegenwoordig een slechte reputatie, en dat is niet helemaal onterecht. DRY is een van die concepten die *te* gemakkelijk te begrijpen is op een oppervlakkig niveau en vervolgens verkeerd wordt toegepast.

Een ontwikkelaar kan DRY gemakkelijk te ver doorvoeren en verbijsterende, verstrengelde abstracties creëren om een paar regels code te besparen in plaats van het *echte* idee van DRY, namelijk het vastleggen van een _idee_ op één plek. Het verminderen van het aantal regels code is vaak een bijwerking van DRY, **maar het is niet het werkelijke doel**.

Dus ja, DRY kan verkeerd worden toegepast, maar het tegenovergestelde van weigeren om iets op de DRY manier te doen, is ook slecht. Herhaalde code voegt ruis toe en verhoogt de onderhoudskosten. Een weigering om gerelateerde concepten of waarden in één ding te verzamelen uit angst voor verkeerd gebruik van DRY veroorzaakt *andere* problemen.

Dus in plaats van extremistisch te zijn over "alles moet DRY" of "DRY is slecht", zet je hersenen aan het werk en denk na over de code die je voor je ziet. Wat wordt herhaald? Moet dat? Ziet de parameterlijst er logisch uit als je herhaalde code in een methode encapsuleert? Voelt het zelfdocumenterend aan en vat het het "idee" duidelijk samen?

Negen van de tien keer kun je naar de argumentenlijst van een functie kijken, en als die er rommelig en verwarrend uitziet, is het waarschijnlijk een slechte toepassing van DRY.

Als het moeilijk voelt om code DRY te maken, maak je het waarschijnlijk complexer; overweeg om te stoppen.

DRY met zorg, **maar door dit regelmatig te oefenen, verbeter je je beoordelingsvermogen**. Ik moedig mijn collega's aan om "het gewoon te proberen" en broncodebeheer te gebruiken om terug te keren naar de veilige haven als het fout gaat.

_**Door deze dingen te proberen leer je meer dan er over te praten**_. Met broncodebeheer in combinatie met goede geautomatiseerde tests heb je de perfecte setting om te experimenteren en te leren.

### Extraheer "Magische" waarden.

> [Unieke waarden met een onverklaarbare betekenis of meerdere waarden die (bij voorkeur) vervangen kunnen worden door benoemde constanten](https://en.wikipedia.org/wiki/Magic_number_(programming))

Gebruik extractvariabele `(command+option+v)` of constante `(command+option+c)` om betekenis te geven aan magische waarden. Dit kan worden gezien als de inverse van de inlining-refactoring. Ik merk dat ik de code vaak "wissel" met inline en extract om te beoordelen wat ik beter vind lezen.

Onthoud dat het extraheren van herhaalde waarden ook een niveau van _koppeling_ toevoegt. Alles wat die waarde gebruikt, is nu gekoppeld. Bekijk de volgende code:

```go
func main() {
	api1Client := http.Client{
		Timeout: 1 * time.Second,
	}
	api2Client := http.Client{
		Timeout: 1 * time.Second,
	}
	api3Client := http.Client{
		Timeout: 1 * time.Second,
	}
	//etc
}
```

We zijn bezig met het instellen van een aantal HTTP-clients voor onze applicatie. Er zijn hier een aantal _magische waarden_, en we zouden de `Time-out` meer DRY kunnen maken door een variabele te extraheren en deze een betekenisvolle naam te geven.

![A screenshot of me extracting variable](https://i.imgur.com/4sgUG7L.png)

Nu ziet de code er zo uit:

```go
func main() {
	timeout := 1 * time.Second
	api1Client := http.Client{
		Timeout: timeout,
	}
	api2Client := http.Client{
		Timeout: timeout,
	}
	api3Client := http.Client{
		Timeout: timeout,
	}
	// etc..
}
```

We hebben geen magische waarde meer; we hebben er een betekenisvolle naam aan gegeven, maar we hebben er ook voor gezorgd dat alle drie de clients **dezelfde time-out delen**. Dat is misschien wat je wilt; refactors zijn behoorlijk contextspecifiek, maar het is iets om rekening mee te houden.

Als je je IDE goed kunt gebruiken, kun je de _inline_ refactoring uitvoeren om de clients weer aparte `Time-out`-waarden te geven.

### Maak openbare methoden/functies eenvoudig te scannen

Heeft je code buitensporig lange openbare methoden of functies?

Verpak de stappen in privémethoden/functies met de extractmethode (`command+option+m`) refactoring.

De onderstaande code bevat een saaie, afleidende ceremonie rond het aanmaken van een JSON-string en het omzetten ervan in een `io.Reader`, zodat we deze in een HTTP-verzoek kunnen `POST`en.


```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	payload := []byte(`{"name": "` + name + `"}`)

	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer(payload),
	)
	//todo: handle codes, err etc
}
```

Gebruik eerst de inline variabele refactor `(command+option+n)` om de `payload` in de buffercreatie te plaatsen.

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer([]byte(`{"name": "`+name+`"}`)),
	)
	// etc
}
```

Nu kunnen we de aanmaak van de JSON-payload extraheren naar een functie met behulp van de extract-methode refactoring (`command+option+m`) om de ruis uit de methode te verwijderen.

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		createWidgetPayload(name),
	)
	// etc
}
```

Publieke methoden en functies zouden moeten beschrijven *wat* ze doen in plaats van *hoe* ze het doen.

> **Wanneer ik moet nadenken om te begrijpen wat de code doet, vraag ik mezelf af of ik de code kan refactoren om dat inzicht directer duidelijk te maken**
>
> -- Martin Fowler

Dit helpt je het algehele ontwerp beter te begrijpen en stelt je vervolgens in staat vragen te stellen over verantwoordelijkheden:

> Waarom doet deze methode X? Zou dat niet in Y moeten staan?

> Waarom voert deze methode zoveel taken uit? Kunnen we dit elders onderbrengen?

Private functies en -methoden zijn geweldig; ze laten je irrelevante "hoe's" samenvatten in "wat's".

#### Maar nu weet ik niet meer hoe het werkt!

Een veelgehoord bezwaar tegen deze refactoring, die de voorkeur geeft aan kleinere functies en methoden die uit andere bestaan, is dat het de werking van de code moeilijk kan begrijpen. Mijn botte antwoord hierop is:

> Heb je geleerd hoe je effectief door codebases kunt navigeren met je tooling?

Als _schrijver_ van `CreateWidget` wil ik bewust niet dat het aanmaken van een specifieke string een essentieel onderdeel is in de beschrijving van de methode. Het is in 99% van de gevallen afleidende, irrelevante ruis voor de lezer.

Maar als iemand het _wel_ interesseert, druk je op `command+b` (of wat "navigeer naar symbool" voor jou ook is) op `createWidgetPayload` ... en lees je het. Druk op `command+pijltje naar links` om weer terug te gaan.

### Verplaats de waardecreatie naar de constructietijd.

Methoden moeten vaak waarde creëren en gebruiken, zoals de `url` in onze `CreateWidget`-methode van eerder.

```go
type WidgetService struct {
	baseURL string
	client  *http.Client
}

func NewWidgetService(baseURL string) *WidgetService {
	client := http.Client{
		Timeout: 10 * time.Second,
	}
	return &WidgetService{baseURL: baseURL, client: &client}
}

func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		createWidgetPayload(name),
	)
	// etc
}
```

Een refactoringtechniek die je hierbij kunt toepassen, is dat als er een waarde wordt aangemaakt **die niet afhankelijk is van de argumenten voor de methode**, je in plaats daarvan een _veld_ in je type kunt maken en deze in je constructorfunctie kunt berekenen.

```go
type WidgetService struct {
	client          *http.Client
	createWidgetURL string
}

func NewWidgetService(baseURL string) *WidgetService {
	client := http.Client{
		Timeout: 10 * time.Second,
	}
	return &WidgetService{
		createWidgetURL: baseURL + "/widgets",
		client:          &client,
	}
}

func (ws *WidgetService) CreateWidget(name string) error {
	req, err := http.NewRequest(
		http.MethodPost,
		ws.createWidgetURL,
		createWidgetPayload(name),
	)
	// etc
}
```

Door ze naar de bouwtijd te verplaatsen, kun je de methoden vereenvoudigen.

#### `CreateWidget` vergelijken en contrasteren

Beginnend met

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	payload := []byte(`{"name": "` + name + `"}`)
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer(payload),
	)
	// etc
}

```

Met een paar basisrefactoringen, die bijna volledig werden aangestuurd door geautomatiseerde tools, hebben we het volgende resultaat bereikt:

```go
func (ws *WidgetService) CreateWidget(name string) error {
	req, err := http.NewRequest(
		http.MethodPost,
		ws.createWidgetURL,
		createWidgetPayload(name),
	)
	// etc
}
```

Dit is een kleine verbetering, maar hij leest ongetwijfeld prettiger. Als je goed geoefend bent, kost dit soort verbeteringen je amper een minuut, en zolang je TDD goed toepast, heb je het vangnet van tests om ervoor te zorgen dat je niets kapotmaakt. Deze voortdurende kleine verbeteringen zijn essentieel voor de gezondheid van een codebase op de lange termijn.

### Probeer comments te verwijderen.

> Een heuristiek die we volgen, is dat wanneer we de behoefte voelen om iets te becommentariëren, we in plaats daarvan een methode schrijven.
>
> -- Martin Fowler

Ook hier kan de extract-methode-refactoring je vriend zijn.

## Uitzonderingen op de regel

Er zijn verbeteringen die je in je code kunt aanbrengen die een wijziging in je tests vereisen. Die zou ik nog steeds graag in de categorie "refactoring" plaatsen, ook al overtreedt het de regel.

Een eenvoudig voorbeeld is het hernoemen van een openbaar symbool (bijvoorbeeld een methode, type of functie) met `shift+F6`. Dit zal natuurlijk de productie- en testcodes wijzigen.

Omdat het echter een **geautomatiseerde en veilige** wijziging is, is het risico dat je in een spiraal terechtkomt van het breken van tests en productiecode, waar zovelen in terechtkomen bij andere soorten *ontwerp*-wijzigingen, minimaal.

Om die reden zou ik alle wijzigingen die je veilig met je IDE/editor kunt uitvoeren, nog steeds graag refactoring noemen.

## Gebruik je tools om te oefenen met refactoren.

- Voer je unit tests uit elke keer dat je een van deze kleine wijzigingen doorvoert. We investeren tijd in het unit-testbaar maken van onze code, en de feedbackloop van een paar milliseconden is een van de belangrijkste voordelen; maak er gebruik van!
- Vertrouw op bronbeheer. Je moet je niet schamen om ideeën uit te proberen. Als je tevreden bent, commit het; zo niet, draai het terug. Dit moet comfortabel en gemakkelijk aanvoelen en geen probleem zijn.
- Hoe beter je je unit tests en bronbeheer benut, hoe gemakkelijker het is om te *oefenen* met refactoren. Zodra je deze discipline onder de knie hebt, **verbeteren je ontwerpvaardigheden snel** omdat je een betrouwbare en effectieve feedbackloop en vangnet hebt.
- Te vaak in mijn carrière heb ik ontwikkelaars horen klagen dat ze geen tijd hebben om te refactoren; helaas is het duidelijk dat het hen zoveel tijd kost omdat ze het niet gedisciplineerd doen, en ze hebben het niet genoeg geoefend.
- Hoewel typen nooit de bottleneck is, zou je elke editor/IDE die je gebruikt veilig en snel moeten kunnen gebruiken om te refactoren. Als je tool bijvoorbeeld niet in staat is om variabelen met één toetsaanslag te extraheren, zul je het minder vaak doen, omdat het arbeidsintensiever en riskanter is.

## Vraag geen toestemming om te refactoren

Refactoren zou een frequente gebeurtenis in je werk moeten zijn, iets wat je constant doet. Het zou ook geen tijdrovende bezigheid moeten zijn, vooral niet als je het in kleine stappen en vaak doet.

Als je niet refactored, zal je interne code kwaliteit eronder lijden, zal de capaciteit van je team afnemen en zal de druk toenemen.

Martin Fowler heeft nog een fantastische quote voor ons.

> Behalve wanneer je heel dicht bij een deadline zit, moet je refactoren niet uitstellen omdat je er geen tijd voor hebt. Ervaring met verschillende projecten heeft geleerd dat refactoren resulteert in een hogere productiviteit. Te weinig tijd is meestal een teken dat je moet refactoren.

## Samenvattend

Dit is geen uitgebreide lijst, maar slechts een begin. Lees Martin Fowler's Refactoring-boek (2e editie) om een professional te worden.

Refactoring zou extreem snel en veilig moeten zijn als je voldoende ervaring hebt, dus er zijn weinig excuses om het niet te doen. Te veel mensen zien refactoring als een beslissing die anderen moeten nemen in plaats van een vaardigheid die je moet leren, waardoor het een vast onderdeel van je werk wordt.

We moeten er altijd naar streven om code in een *voorbeeldige* staat op te leveren.

Goede refactoring leidt tot code die gemakkelijker te begrijpen is. Begrip van de code betekent dat betere ontwerpen gemakkelijker te herkennen zijn. Het is veel moeilijker om ontwerpen te vinden in systemen met enorme functies, onnodig gedupliceerde code, diepe nesting, enz. **Regelmatige, kleine refactoring is noodzakelijk voor een beter ontwerp**.
