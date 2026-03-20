# Inleiding tot property based testen

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/roman-numerals)

Sommige bedrijven vragen je om de [Romeinse cijferkata](http://codingdojo.org/kata/RomanNumerals/) te doen als onderdeel van het sollicitatiegesprek. Dit hoofdstuk laat zien hoe je dit met TDD kunt aanpakken.

We gaan een functie schrijven die een [Arabisch getal](https://en.wikipedia.org/wiki/Arabic_numerals) (de cijfers 0 tot en met 9) omzet naar een Romeins cijfer.

Als je nog nooit van [Romeinse cijfers](https://en.wikipedia.org/wiki/Roman_numerals) hebt gehoord: dit is de manier hoe de Romeinen vroeger getallen schreven.

Je bouwt ze door symbolen aan elkaar te plakken en die symbolen stellen getallen voor

Zo staat `I` voor "éen" en staat `III` voor drie.

Lijkt makkelijk, maar er zijn een paar interessante regels. `V` betekent vijf, maar `IV` is 4 (niet `IIII`).

`MCMLXXXIV` is 1984. Dat ziet er ingewikkeld uit en het is moeilijk voor te stellen hoe we code kunnen schrijven om dit vanaf het begin uit te zoeken.

Zoals dit boek benadrukt, is een belangrijke vaardigheid voor softwareontwikkelaars het identificeren van "dunne verticale segmenten" van _nuttige_ functionaliteit en vervolgens te **itereren** naar resultaat. De TDD-workflow vergemakkelijkt iteratieve ontwikkeling.

Dus, in plaats van te starten met 1984, laten we beginnen met 1.

## Schrijf eerst de test

```go
func TestRomanNumerals(t *testing.T) {
	got := ConvertToRoman(1)
	want := "I"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Als je tot hier in het boek bent gekomen, vind je het hopelijk erg saai en routineus. Dat is goed.

## Probeer de test uit te voeren

```console
./numeral_test.go:6:9: undefined: ConvertToRoman
```

Laat de compiler je de weg wijzen.

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Creëer onze functie, maar zorg ervoor dat de test nog niet slaagt. Zorg er altijd voor dat de tests falen zoals je verwacht.

```go
func ConvertToRoman(arabic int) string {
	return ""
}
```

De test zou nu moeten werken

```console
=== RUN   TestRomanNumerals
--- FAIL: TestRomanNumerals (0.00s)
    numeral_test.go:10: got '', want 'I'
FAIL
```

## Schrijf genoeg code om de test te laten slagen

```go
func ConvertToRoman(arabic int) string {
	return "I"
}
```

## Refactor

Er is nog niet zoveel te refactoren op dit moment.

_Ik weet_ dat het vreemd voelt om het resultaat gewoon hard te coderen, maar met TDD willen we zo lang mogelijk uit de "rode" situatie blijven. Het voelt misschien alsof we niet veel hebben bereikt, maar we hebben onze API gedefinieerd en een test uitgevoerd die een van onze regels vastlegt, ook al is de "echte" code behoorlijk dom.

Gebruik dat ongemakkelijke gevoel nu om een ​​nieuwe test te schrijven die ons dwingt om iets minder domme code te schrijven.

## Schrijf eerst je test

We kunnen subtests gebruiken om onze tests netjes te groeperen

```go
func TestRomanNumerals(t *testing.T) {
	t.Run("1 gets converted to I", func(t *testing.T) {
		got := ConvertToRoman(1)
		want := "I"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("2 gets converted to II", func(t *testing.T) {
		got := ConvertToRoman(2)
		want := "II"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

## Probeer de test uit te voeren

```console
=== RUN   TestRomanNumerals/2_gets_converted_to_II
    --- FAIL: TestRomanNumerals/2_gets_converted_to_II (0.00s)
        numeral_test.go:20: got 'I', want 'II'
```

Niet heel veel verbazends hier

## Schrijf genoeg code om de test te laten slagen

```go
func ConvertToRoman(arabic int) string {
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

Ja, het voelt nog steeds alsof we het probleem niet echt aanpakken. Dus moeten we meer tests schrijven om vooruit te komen.

## Refactor

We hebben wat herhaling in onze tests. Wanneer je iets test waarvan je het gevoel hebt dat het een kwestie is van "gegeven input X, verwachten we Y", kun je beter tabelgebaseerde tests gebruiken.

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Description string
		Arabic      int
		Want        string
	}{
		{"1 gets converted to I", 1, "I"},
		{"2 gets converted to II", 2, "II"},
	}

	for _, test := range cases {
		t.Run(test.Description, func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Want {
				t.Errorf("got %q, want %q", got, test.Want)
			}
		})
	}
}
```

We kunnen nu eenvoudig meer cases toevoegen zonder dat we nieuwe testboilerplates hoeven te schrijven.

Laten even doorzetten en verder gaan met 3

## Schrijf eerst je test

Voeg het onderstaande toe aan je testcases:

```
{"3 gets converted to III", 3, "III"},
```

## Probeer de test uit te voeren

```console
=== RUN   TestRomanNumerals/3_gets_converted_to_III
    --- FAIL: TestRomanNumerals/3_gets_converted_to_III (0.00s)
        numeral_test.go:20: got 'I', want 'III'
```

## Schrijf genoeg code om de test te laten slagen

```go
func ConvertToRoman(arabic int) string {
	if arabic == 3 {
		return "III"
	}
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

## Refactor

Oké, ik begin deze if-statements niet meer zo leuk te vinden. En als je goed naar de code kijkt, zie je dat we een string van `I`'s bouwen die gebaseerd is op de waarde van `arabic`.

We "weten" dat we voor ingewikkeldere getallen een soort rekenkunde en het samenvoegen van tekenreeksen zullen moeten toepassen.

Laten we met deze gedachten een refactoring proberen. Het is _misschien niet_ geschikt voor de uiteindelijke oplossing, maar dat is oké. We kunnen onze code altijd weggooien en opnieuw beginnen met de tests die we als leidraad hebben.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := 0; i < arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

Je herinnert je misschien [`strings.Builder`](https://golang.org/pkg/strings/#Builder) nog van onze discussie over [benchmarking](iteration.md#benchmarking)

> Een Builder wordt gebruikt om efficiënt een string te bouwen met behulp van Write-methoden. Dit minimaliseert het kopiëren van geheugen.

Normaal gesproken zou ik pas met dit soort optimalisaties beginnen als ik daadwerkelijk een prestatieprobleem heb. Maar de hoeveelheid code is niet veel groter dan een "handmatige" toevoeging aan een string, dus we kunnen net zo goed de snellere aanpak gebruiken.

De code ziet er wat mij betreft beter uit en beschrijft het domein _zoals wij dat nu kennen_.

### De Romeinen kende de DRY-principes ook...

Het wordt nu ingewikkelder. De Romeinen dachten in hun wijsheid dat herhalende tekens moeilijk te lezen en te tellen zouden worden. Een regel met Romeinse cijfers is dus dat je hetzelfde teken niet vaker dan drie keer achter elkaar mag herhalen.

In plaats daarvan neem je het op één na hoogste symbool en verminder je het vervolgens door er een symbool links van te plaatsen. Niet alle symbolen kunnen voor vermindering worden gebruikt; alleen I (1), X (10) en C (100).

Bijvoorbeeld, `5` in Romeinse cijfers is een `V`. Om 4 te maken, doe je niet `IIII`, maar `IV`.

## Schrijf eerst je test

```
{"4 gets converted to IV (can't repeat more than 3 times)", 4, "IV"},
```

## Probeer de test uit te voeren

```console
=== RUN   TestRomanNumerals/4_gets_converted_to_IV_(cant_repeat_more_than_3_times)
    --- FAIL: TestRomanNumerals/4_gets_converted_to_IV_(cant_repeat_more_than_3_times) (0.00s)
        numeral_test.go:24: got 'IIII', want 'IV'
```

## Schrijf genoeg code om de test te laten slagen

```go
func ConvertToRoman(arabic int) string {

	if arabic == 4 {
		return "IV"
	}

	var result strings.Builder

	for i := 0; i < arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

## Refactor

Ik vind het niet prettig dat we het patroon van het opbouwen van strings hebben doorbroken, en ik zou graag op die manier willen doorgaan.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```

Om 4 te laten "passen" bij mijn huidige denkwijze, tel ik nu af vanaf het Arabische getal en voeg ik naarmate we vorderen symbolen toe aan de reeks. Ik weet niet zeker of dit op de lange termijn zal werken, maar we zullen zien!

Laten we zorgen dat 5 ook werkt&#x20;

## Schrijf eerst de test

```
{"5 gets converted to V", 5, "V"},
```

## Probeer de test uit te voeren

```console
=== RUN   TestRomanNumerals/5_gets_converted_to_V
    --- FAIL: TestRomanNumerals/5_gets_converted_to_V (0.00s)
        numeral_test.go:25: got 'IIV', want 'V'
```

## Schrijf genoeg code om de test te laten slagen

Kopieer gewoon de aanpak die we voor 4 hebben gebruikt

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 5 {
			result.WriteString("V")
			break
		}
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```

## Refactor

Herhaling in lussen zoals deze is meestal een teken dat een abstractie wacht om benoemd te worden. Kortsluitende lussen kunnen een effectief hulpmiddel zijn voor leesbaarheid, maar ze kunnen je ook iets anders vertellen.

We gaan over ons Arabische getal heen en als we bepaalde symbolen tegenkomen roepen we `break` aan, maar wat we _eigenlijk_ doen is op een onhandige `i` verlagen.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for arabic > 0 {
		switch {
		case arabic > 4:
			result.WriteString("V")
			arabic -= 5
		case arabic > 3:
			result.WriteString("IV")
			arabic -= 4
		default:
			result.WriteString("I")
			arabic--
		}
	}

	return result.String()
}
```

* Als ik goed kijk naar de code, en onze tests van een aantal zeer eenvoudige scenario's, kan ik zien dat ik, om een ​​Romeins cijfer te maken, iets van `arabic` moet aftrekken terwijl ik symbolen toepas.
* De `for`-lus is niet langer afhankelijk van een `i` en in plaats daarvan blijven we de string opbouwen totdat we genoeg waarden van `arabic` hebben afgetrokken.

Ik ben er vrij zeker van dat deze aanpak ook voor 6 (VI), 7 (VII) en 8 (VIII) zal gelden. Voeg de cases desalniettemin toe aan onze testsuite en controleer dit (ik zal de code niet opnemen om dit hoofdstuk niet te lang te maken. Kijk op GitHub voor voorbeelden als je het niet zeker weet hoe je dit doet).

9 volgt dezelfde regel als 4 in die zin dat we 1 moeten aftrekken van de representatie van het volgende getal. 10 wordt in Romeinse cijfers weergegeven met `X`; dus 9 zou `IX` moeten zijn.

## Schrijf eerst je test

```
{"9 gets converted to IX", 9, "IX"},
```

## Probeer de test uit te voeren

```console
=== RUN   TestRomanNumerals/9_gets_converted_to_IX
    --- FAIL: TestRomanNumerals/9_gets_converted_to_IX (0.00s)
        numeral_test.go:29: got 'VIV', want 'IX'
```

## Schrijf genoeg code om de test te laten slagen

We zouden dezelfde aanpak moeten kunnen hanteren als voorheen

```
case arabic > 8:
    result.WriteString("IX")
    arabic -= 9
```

## Refactor

Het _lijkt_ erop dat de code ons nog steeds vertelt dat er ergens een refactoring plaats moet vinden, maar voor mij is dat niet helemaal duidelijk waar, dus laten we verdergaan.

Ik sla de code hiervoor ook over, maar voeg aan je testcases een test voor `10` toe die `X` zou moeten zijn en zorg dat deze slaagt voordat je verder leest.

Hier zijn een paar tests die ik heb toegevoegd, omdat ik er vertrouwen in heb dat onze code tot 39 zou moeten werken

```
{"10 gets converted to X", 10, "X"},
{"14 gets converted to XIV", 14, "XIV"},
{"18 gets converted to XVIII", 18, "XVIII"},
{"20 gets converted to XX", 20, "XX"},
{"39 gets converted to XXXIX", 39, "XXXIX"},
```

Als je ooit OO-programmering hebt gedaan, weet je dat je `switch`-statements met enige argwaan moet bekijken. Meestal leg je een concept of data vast in een imperatieve code, terwijl het in werkelijkheid in een klassenstructuur zou kunnen worden vastgelegd.

Go is strikt genomen niet OO, maar dat betekent niet dat we de lessen die OO biedt volledig negeren (hoe graag sommigen dat ook zouden willen beweren).

Onze switch-statement beschrijft enkele waarheden over Romeinse cijfers en hun gedrag.

We kunnen dit herstructureren door de data los te koppelen van het gedrag.

```go
type RomanNumeral struct {
	Value  int
	Symbol string
}

var allRomanNumerals = []RomanNumeral{
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for _, numeral := range allRomanNumerals {
		for arabic >= numeral.Value {
			result.WriteString(numeral.Symbol)
			arabic -= numeral.Value
		}
	}

	return result.String()
}
```

Dit voelt veel beter. We hebben een aantal regels rond de cijfers als data gedeclareerd in plaats van verborgen in een algoritme, en we kunnen zien hoe we gewoon het Arabische getal berekenen en proberen symbolen aan ons resultaat toe te voegen als ze passen.

Werkt deze abstractie voor grotere getallen? Breid de testsuite uit zodat deze ook werkt voor het Romeinse getal 50, namelijk `L`.

Hier zijn enkele testgevallen, probeer ze te laten slagen.

```
{"40 gets converted to XL", 40, "XL"},
{"47 gets converted to XLVII", 47, "XLVII"},
{"49 gets converted to XLIX", 49, "XLIX"},
{"50 gets converted to L", 50, "L"},
```

Hulp nodig? Je kunt zien welke symbolen je moet toevoegen in [deze gist](https://gist.github.com/pamelafox/6c7b948213ba55332d86efd0f0b037de).

## En de rest!

Hier zijn de resterende symbolen

| Arabisch | Romeins |
| -------- | :-----: |
| 100      |    C    |
| 500      |    D    |
| 1000     |    M    |

Gebruik dezelfde aanpak voor de overige symbolen. Het enige wat je hoeft te doen, is gegevens toevoegen aan zowel de tests als aan onze reeks symbolen.

Werkt je code ook voor `1984`: `MCMLXXXIV` ?

Hier is mijn laatste testsuite

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Arabic int
		Roman  string
	}{
		{Arabic: 1, Roman: "I"},
		{Arabic: 2, Roman: "II"},
		{Arabic: 3, Roman: "III"},
		{Arabic: 4, Roman: "IV"},
		{Arabic: 5, Roman: "V"},
		{Arabic: 6, Roman: "VI"},
		{Arabic: 7, Roman: "VII"},
		{Arabic: 8, Roman: "VIII"},
		{Arabic: 9, Roman: "IX"},
		{Arabic: 10, Roman: "X"},
		{Arabic: 14, Roman: "XIV"},
		{Arabic: 18, Roman: "XVIII"},
		{Arabic: 20, Roman: "XX"},
		{Arabic: 39, Roman: "XXXIX"},
		{Arabic: 40, Roman: "XL"},
		{Arabic: 47, Roman: "XLVII"},
		{Arabic: 49, Roman: "XLIX"},
		{Arabic: 50, Roman: "L"},
		{Arabic: 100, Roman: "C"},
		{Arabic: 90, Roman: "XC"},
		{Arabic: 400, Roman: "CD"},
		{Arabic: 500, Roman: "D"},
		{Arabic: 900, Roman: "CM"},
		{Arabic: 1000, Roman: "M"},
		{Arabic: 1984, Roman: "MCMLXXXIV"},
		{Arabic: 3999, Roman: "MMMCMXCIX"},
		{Arabic: 2014, Roman: "MMXIV"},
		{Arabic: 1006, Roman: "MVI"},
		{Arabic: 798, Roman: "DCCXCVIII"},
	}
	for _, test := range cases {
		t.Run(fmt.Sprintf("%d gets converted to %q", test.Arabic, test.Roman), func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Roman {
				t.Errorf("got %q, want %q", got, test.Roman)
			}
		})
	}
}
```

* Ik heb de `description` verwijderd, omdat ik vond dat de gegevens de informatie voldoende beschreven en ik steeds dezelfde regel aan het herhalen was.
* Ik heb een paar andere randgevallen toegevoegd om mezelf wat meer vertrouwen te geven. Met tabel-gebaseerde tests is dit heel goedkoop toe te voegen.

Ik heb het algoritme niet veranderd. Het enige wat ik moest doen was de `allRomanNumerals`-array uitbreiden.

```go
var allRomanNumerals = []RomanNumeral{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}
```

## Romeinse cijfers ontleden

We zijn er nog niet. Nu gaan we een functie schrijven die een Romeins cijfer _naar_ een `int` converteert.

## Schrijf eerst de test

We kunnen onze testcases hier opnieuw gebruiken met een kleine refactoring

Verplaats de `cases`-variabele buiten de test als een pakketvariabele in een `var`-blok.

```go
func TestConvertingToArabic(t *testing.T) {
	for _, test := range cases[:1] {
		t.Run(fmt.Sprintf("%q gets converted to %d", test.Roman, test.Arabic), func(t *testing.T) {
			got := ConvertToArabic(test.Roman)
			if got != test.Arabic {
				t.Errorf("got %d, want %d", got, test.Arabic)
			}
		})
	}
}
```

Let op, ik gebruik de slice-functionaliteit om nu slechts één van de tests uit te voeren (`cases[:1]`), omdat het een te grote stap is om al die tests in één keer te laten slagen.

## Probeer de test uit te voeren

```console
./numeral_test.go:60:11: undefined: ConvertToArabic
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Voeg de nieuwe functie definitie toe

```go
func ConvertToArabic(roman string) int {
	return 0
}
```

De test kan nu worden uitgevoerd en zal falen

```console
--- FAIL: TestConvertingToArabic (0.00s)
    --- FAIL: TestConvertingToArabic/'I'_gets_converted_to_1 (0.00s)
        numeral_test.go:62: got 0, want 1
```

## Schrijf genoeg code om de test te laten slagen

Je weet wat je te doen staat

```go
func ConvertToArabic(roman string) int {
	return 1
}
```

Wijzig vervolgens de slice-index in onze test om naar de volgende testcase te gaan (bijv. `cases[:2]`). Laat deze test zelf slagen met de domste code die je kunt bedenken, en blijf ook voor de derde case domme code schrijven (het beste boek ooit, toch?). Hier is mijn domme code.

```go
func ConvertToArabic(roman string) int {
	if roman == "III" {
		return 3
	}
	if roman == "II" {
		return 2
	}
	return 1
}
```

Door de domheid van _echte code die werkt_, kunnen we een patroon beginnen te zien zoals eerder. We moeten door de input itereren en _iets_ bouwen, in dit geval een totaal.

```go
func ConvertToArabic(roman string) int {
	total := 0
	for range roman {
		total++
	}
	return total
}
```

## Schrijf eerst je test

Vervolgens gaan we naar de `cases[:4]` (`IV`), die nu niet meer werken omdat er 2 terugkomt, aangezien dat de lengte van de string is.

## Schrijf genoeg code om de test te laten slagen

```go
// earlier..
var allRomanNumerals = []RomanNumerals{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

// later..
func ConvertToArabic(roman string) int {
	var arabic = 0

	for _, numeral := range allRomanNumerals {
		for strings.HasPrefix(roman, numeral.Symbol) {
			arabic += numeral.Value
			roman = strings.TrimPrefix(roman, numeral.Symbol)
		}
	}

	return arabic
}
```

Het is in feite het algoritme van `ConvertToRoman(int)`, maar dan achterstevoren geïmplementeerd. Hier loopen we over de gegeven Romeinse cijferreeks:

* We zoeken naar Romeinse cijfersymbolen uit `allRomanNumerals`, van hoog naar laag, aan het begin van de reeks.
* Als we het voorvoegsel vinden, voegen we de waarde ervan toe aan `arabic` en kappen we het voorvoegsel af.

Uiteindelijk geven we de som terug als een Arabisch getal.

`HasPrefix(s, prefix)` controleert of string `s` begint met `prefix` en `TrimPrefix(s, prefix)` verwijdert het `prefix` van `s`, zodat we verder kunnen met de resterende Romeinse cijfersymbolen. Het werkt met `IV` en alle andere testcases.

Je kunt dit implementeren als een recursieve functie, wat eleganter is (naar mijn mening), maar mogelijk ook langzamer. Ik laat dit aan jou over, samen met wat `benchmark...` tests.

Nu we de functies hebben om een ​​Arabisch getal naar een Romeins cijfer om te zetten en omgekeerd, kunnen we onze tests nog een stap verder uitvoeren:

## Een introductie tot op eigenschappen gebaseerde tests

Er zijn een paar regels op het gebied van Romeinse cijfers waarmee we in dit hoofdstuk hebben gewerkt

* Er kunnen niet meer dan 3 dezelfde opeenvolgende symbolen zijn
* Alleen I (1), X (10) en C (100) kunnen worden afgetrokken
* Als we het resultaat van `ConvertToRoman(N)` nemen en doorgeven aan `ConvertToArabic`, moeten we `N` terugkrijgen.

De tests die we tot nu toe hebben geschreven, kunnen worden omschreven als 'voorbeeldtests', waarbij we voorbeelden aanleveren om de tooling te verifiëren.

Wat als we de regels die we over ons domein kennen, zouden kunnen gebruiken om deze op de een of andere manier toe te passen op onze code?

Eigenschap gebaseerde tests (Property based tests) helpen je hierbij door willekeurige data aan je code toe te voegen en te controleren of de regels die je beschrijft altijd kloppen. Veel mensen denken dat eigenschap gebaseerde tests voornamelijk om willekeurige data gaan, maar dat is niet waar. De echte uitdaging bij eigenschapsgebaseerde tests is een goed begrip van je domein, zodat je deze eigenschappen kunt schrijven.

Genoeg woorden, laten we eens kijken wat code

```go
func TestPropertiesOfConversion(t *testing.T) {
	assertion := func(arabic int) bool {
		roman := ConvertToRoman(arabic)
		fromRoman := ConvertToArabic(roman)
		return fromRoman == arabic
	}

	if err := quick.Check(assertion, nil); err != nil {
		t.Error("failed checks", err)
	}
}
```

### Grondslag van eigendom

Onze eerste test controleert of als we een getal naar het Romeins omzetten, we met onze andere functie het getal weer terugkrijgen naar het oorspronkelijke getal.

* Gegeven een willekeurig getal (bijv. `4`).
* Roep `ConvertToRoman` aan met het willekeurige getal (zou `IV` terug moeten geval bij `4`).
* Neem het verkregen resultaat en geef het door aan `ConvertToArabic`.
* Het bovenstaande zou ons de originele input terug moeten geven (`4`).

Dit voelt als een goede test om ons vertrouwen te vergroten, want het zou moeten werken als er een bug in een van beide zit. De enige manier waarop het zou kunnen slagen is als ze dezelfde soort bug hebben; wat niet onmogelijk is, maar onwaarschijnlijk lijkt.

### Technische uitleg

We gebruiken de [testing/quick](https://golang.org/pkg/testing/quick/) package uit de standaard bibliotheek

Als we van onderaf lezen, bieden we snel een `quick.Check` aan voor een functie die wordt uitgevoerd op een aantal willekeurige invoeren. Als de functie `false` retourneert, wordt deze beschouwd als mislukt door de controle.

Onze bovenstaande `assertion` neemt een willekeurig getal en voert onze functies uit om de eigenschap te testen.

### De test uitvoeren

Probeer het eens uit; je computer kan dan even vastlopen, dus sluit het programma af als je je verveelt :)

Wat is er aan de hand? Probeer het volgende toe te voegen aan de assertiecode.

```go
assertion := func(arabic int) bool {
   if arabic < 0 || arabic > 3999 {
   	log.Println(arabic)
   	return true
   }
   roman := ConvertToRoman(arabic)
   fromRoman := ConvertToArabic(roman)
   return fromRoman == arabic
}
```

Je zou dan zoiets moeten zien:

```console
=== RUN   TestPropertiesOfConversion
2019/07/09 14:41:27 6849766357708982977
2019/07/09 14:41:27 -7028152357875163913
2019/07/09 14:41:27 -6752532134903680693
2019/07/09 14:41:27 4051793897228170080
2019/07/09 14:41:27 -1111868396280600429
2019/07/09 14:41:27 8851967058300421387
2019/07/09 14:41:27 562755830018219185
```

Alleen al het uitvoeren van deze zeer eenvoudige eigenschap heeft een fout in onze implementatie blootgelegd. We gebruikten int als invoer, maar:

* Je kunt geen negatieve getallen gebruiken met Romeinse cijfers
* Gegeven onze regel van maximaal 3 opeenvolgende symbolen kunnen we geen waarde groter dan 3999 weergeven (nou ja, [soort van](https://www.quora.com/Which-is-the-maximum-number-in-Roman-numerals)) en `int` heeft een veel hogere maximumwaarde dan 3999.

Geweldig! We zijn gedwongen om dieper na te denken over ons domein, wat een echte kracht is van eigenschap gebaseerde tests.

Het is duidelijk dat `int` niet het beste type is voor deze toepassing. Wat als we iets geschikters zouden proberen?

### [`uint16`](https://golang.org/pkg/builtin/#uint16)

Go heeft typen voor _unsigned integers_, wat betekent dat ze niet negatief kunnen zijn; dat sluit meteen een bug in onze code uit. Door 16 toe te voegen, wordt het een 16-bits integer die maximaal de waarde `65535` kan opslaan. Dat is nog steeds te groot, maar brengt ons wel dichter bij wat we nodig hebben.

Probeer de code bij te werken zodat deze `uint16` gebruikt in plaats van `int`. Ik heb de `assertion` in de test bijgewerkt om iets meer zichtbaarheid te geven.

> Let erop dat je in `roman.go` ook de variabele `arabic` moet aanpassen naar `uint16` (de test zal je dit vertellen). Wat misschien een grotere zoektocht is, is de foutmelding die je krijgt voor de regel `arabic += numeral.Value`. Deze melding krijg je omdat we `arabic`  in `ConvertToArabic` hebben gedeclareerd met `arabic := 0`. Deze manier van declareren is goed, maar Go zal er vanuit gaan dat we de `0` moeten behandelen als een `int` waarde. De foutmelding gaat er dus over dat je een `int` waarde en een `uint16` waarde bij elkaar op probeert te tellen. Omdat Go een typed language is, zal dat niet gaan. Pas daarom `arabic := 0` aan naar var arabic `uint16 = 0` om de code te laten werken. &#x20;

```go
assertion := func(arabic uint16) bool {
	if arabic > 3999 {
		return true
	}
	t.Log("testing", arabic)
	roman := ConvertToRoman(arabic)
	fromRoman := ConvertToArabic(roman)
	return fromRoman == arabic
}
```

Merk op dat we nu de invoer loggen met de `log`methode van het test-framework. Zorg ervoor dat je de opdracht `go test` uitvoert met de vlag `-v` om de extra uitvoer te tonen (`go test -v`).

Als je de test uitvoert, worden ze daadwerkelijk uitgevoerd en kun je zien wat er getest wordt. Je kunt de test meerdere keren uitvoeren om te zien of onze code goed presteert op basis van de verschillende waarden! Dit geeft me veel vertrouwen dat onze code werkt zoals we willen.

Het standaard aantal runs dat `quick.Check` uitvoert is 100, maar je kunt dit wijzigen via een configuratie.

```go
if err := quick.Check(assertion, &quick.Config{
	MaxCount: 1000,
}); err != nil {
	t.Error("failed checks", err)
}
```

### Further work

* Kun je een eigenschappen tests schrijven die de andere eigenschappen controleren die we hebben beschreven?
* Kun je een manier bedenken om het zo te maken dat het voor iemand onmogelijk is om de code aan te roepen met een getal groter dan 3999?
  * Je zou een foutmelding terug kunnen geven
  * Of je maakt een nieuw type aan dat getallen groter dan 3999 niet kan vertegenwoordigen
    * Wat zou je beste oplossing zijn denk je?

## Samenvattend

### Meer TDD-oefeningen met iteratieve ontwikkeling

Vond je het idee om code te schrijven die 1984 omzet in MCMLXXXIV in het begin intimiderend? Voor mij wel, en ik schrijf al heel lang software.

De truc is, zoals altijd, om **met iets eenvoudigs te beginnen** en **kleine stapjes te zetten**.

Op geen enkel punt in dit proces hebben we grote sprongen gemaakt, grote refactoringen doorgevoerd of een puinhoop gemaakt.

Ik hoor iemand cynisch zeggen: "Dit is maar een kata." Daar kan ik niet tegenin gaan, maar ik hanteer nog steeds dezelfde aanpak voor elk project waaraan ik werk. Ik lever nooit meteen een groot gedistribueerd systeem af; ik zoek het simpelste wat het team kan leveren (meestal een "Hello world"-website) en itereer dan op kleine stukjes functionaliteit in beheersbare brokken, net zoals we hier deden.

De kunst is om te weten hoe je het werk moet opsplitsen. Met wat oefening en een aantal fijne TDD-technieken kun je dat leren en op weg helpen.

### Eigenschap gebaseerd tests

* Ingebouwd in de standaardbibliotheek
* Als je manieren kunt bedenken om je domeinregels in code te beschrijven, vormen deze een uitstekend hulpmiddel om je meer zelfvertrouwen te geven over je code
* Dwingt je om goed na te denken over je domein
* Potentieel een mooie aanvulling op je testsuite

## Wat extra's

Dit boek is afhankelijk van waardevolle feedback van de community. [Dave](http://github.com/gypsydave5) is een enorme hulp in vrijwel elk hoofdstuk. Maar hij had een flinke tirade over mijn gebruik van 'Arabische cijfers' in dit hoofdstuk, dus, in het belang van volledige openheid, hier is wat hij zei.

> Ik ga gewoon uitleggen waarom een ​​waarde van het type `int` niet echt een 'Arabisch cijfer' is. Misschien ben ik wel veel te precies, dus ik begrijp het helemaal als je me compleet negeert.
>
> Een cijfer is een teken dat gebruikt wordt bij het weergeven van getallen – van het Latijnse woord voor 'vinger', omdat we er meestal tien hebben. In het Arabische (ook wel Hindoe-Arabische) getallenstelsel zijn er tien. Deze Arabische cijfers zijn:
>
> ```console
>   0 1 2 3 4 5 6 7 8 9
> ```
>
> Een _nummer_ is de weergave van een getal met behulp van een verzameling cijfers. Een Arabisch nummer is een getal dat wordt weergegeven door Arabische cijfers in een tientallig positioneel getallenstelsel. We zeggen 'positioneel' omdat elk cijfer een andere waarde heeft, afhankelijk van de positie in het cijfer. Dus
>
> ```console
>   1337
> ```
>
> De `1` heeft de waarde duizend omdat het het eerste cijfer is van een getal met vier cijfers.
>
> Romeinse cijfers bestaan ​​uit een beperkt aantal cijfers (`I`, `V`, enz.), voornamelijk als waarden om het cijfer te vormen. Er is wat positionele informatie, maar meestal staat `I` altijd voor 'één'.
>
> Dus, met deze informatie, is een `int` dan een 'Arabisch getal'? Het idee van een getal is helemaal niet verbonden met de representatie ervan. We kunnen dit zien als we ons afvragen wat de juiste representatie van dit getal is:
>
> ```console
> 255
> 11111111
> twee-honderd en vijf-en-vijftig
> FF
> 377
> ```
>
> Ja, dit is een strikvraag. Ze zijn allemaal correct. Ze representeren hetzelfde getal in respectievelijk het decimale, binaire, Nederlandse, hexadecimale en octale talstelsel.
>
> De representatie van een getal als cijfer is onafhankelijk van zijn eigenschappen als getal. Dit zien we bijvoorbeeld als we kijken naar gehele getallen in Go:
>
> ```go
> 	0xFF == 255 // true
> ```
>
> En hoe we gehele getallen in een formatstring kunnen afdrukken:
>
> ```go
> n := 255
> fmt.Printf("%b %c %d %o %q %x %X %U", n, n, n, n, n, n, n, n)
> // 11111111 ÿ 255 377 'ÿ' ff FF U+00FF
> ```
>
> We kunnen hetzelfde gehele getal zowel als hexadecimaal als als Arabisch (decimaal) cijfer schrijven.
>
> Dus wanneer de functie aanroep eruitziet als `ConvertToRoman(arabic int) string`, gaat het om een ​​aanname over hoe deze wordt aangeroepen. Omdat `arabic` soms wordt geschreven als een decimaal geheel getal (literal):
>
> ```go
> 	ConvertToRoman(255)
> ```
>
> Maar het zou net zo goed geschreven kunnen zijn als hexadecimaal getal:
>
> ```go
> 	ConvertToRoman(0xFF)
> ```
>
> Eigenlijk 'converteren' we helemaal geen Arabisch cijfer, we 'printen' - en representeren - een `int` als een Romeins cijfer - en `int`s zijn geen cijfers, Arabisch of anderszins; het zijn gewoon getallen. De functie `ConvertToRoman` lijkt meer op `strconv.Itoa` in die zin dat het een `int` omzet in een `string`.
>
> Maar elke andere versie van de kata maakt geen onderscheid tussen deze twee, dus ¯\_(ツ)\_/¯
