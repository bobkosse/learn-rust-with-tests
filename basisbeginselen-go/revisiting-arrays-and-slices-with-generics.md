# Arrays en slices opnieuw bekijken met generieke typen

**[De code voor dit hoofdstuk is een voortzetting van Arrays en Slices, hier te vinden](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

Bekijk zowel `SumAll` als `SumAllTails` die we schreven in [arrays en slices](arrays-and-slices.md). Als je je versie niet hebt, kopieer dan de code uit het hoofdstuk [arrays en slices](arrays-and-slices.md) samen met de tests.

```go
// Sum calculates the total from a slice of numbers.
func Sum(numbers []int) int {
	var sum int
	for _, number := range numbers {
		sum += number
	}
	return sum
}

// SumAllTails calculates the sums of all but the first number given a collection of slices.
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

Zie je een terugkerend patroon?

- Creëer een soort "initiële" resultaatwaarde.
- Itereer over de verzameling en pas een bewerking (of functie) toe op het resultaat en het volgende item in de slice, waarbij een nieuwe waarde voor het resultaat wordt ingesteld.
- Retourneer het resultaat.

Dit idee wordt vaak besproken in kringen van functioneel programmeren, vaak 'reduce' of [fold](https://en.wikipedia.org/wiki/Fold_(higher-order_function)) genoemd.

> In functioneel programmeren verwijst fold (ook wel reduce, accumulate, aggregate, compress of inject genoemd) naar een familie van hogere-orde functies die een recursieve datastructuur analyseren en, door middel van een gegeven combinatiebewerking, de resultaten van de recursieve verwerking van de samenstellende delen recombineren en zo een retourwaarde opbouwen. Een fold wordt doorgaans gepresenteerd met een combinatiefunctie, een bovenste knooppunt van een datastructuur en mogelijk enkele standaardwaarden die onder bepaalde omstandigheden moeten worden gebruikt. De fold combineert vervolgens elementen van de hiërarchie van de datastructuur, waarbij de functie op een systematische manier wordt gebruikt.

Go heeft altijd al hogere-orde functies gehad, en vanaf versie 1.18 heeft het ook [generics](./generics.md), waardoor het nu mogelijk is om enkele van deze functies die in ons bredere vakgebied worden besproken, te definiëren. Het heeft geen zin om je kop in het zand te steken; dit is een veelvoorkomende abstractie buiten het Go-ecosysteem en het zal nuttig zijn om het te begrijpen.

Ik weet dat sommigen van jullie hier waarschijnlijk een beetje huiverig voor zijn.

> Go is bedoeld om eenvoudig te zijn

**Verwar gemak niet met eenvoud**. Loops maken en code kopiëren en plakken is makkelijk, maar niet per se simpel. Bekijk [Rich Hickey's meesterwerk van een presentatie - Simple Made Easy](https://www.youtube.com/watch?v=SxdOUGdseq4) voor meer informatie over simpel versus makkelijk.

**Verwar onbekendheid niet met complexiteit**. Fold/reduce klinkt misschien in eerste instantie eng en computerwetenschappelijk, maar het is eigenlijk niets meer dan een abstractie van een veelvoorkomende bewerking. Een verzameling nemen en deze combineren tot één item. Als je een stapje terug doet, zul je je realiseren dat je dit waarschijnlijk _vaak_ doet.

## Een generic refactoring

Een fout die mensen vaak maken met gloednieuwe taalfuncties, is dat ze ze meteen gebruiken zonder een concrete use-case te hebben. Ze vertrouwen op speculatie en giswerk om hun inspanningen te sturen.

Gelukkig hebben we onze "nuttige" functies geschreven en tests eromheen, dus we kunnen nu vrij experimenteren met ideeën in de refactoringfase van TDD en weten dat wat we ook proberen, de waarde ervan wordt geverifieerd via onze unittests.

Het gebruik van generieke functies als hulpmiddel om code te vereenvoudigen via de refactoringstap leidt veel waarschijnlijker tot nuttige verbeteringen dan tot voorbarige abstracties.

We kunnen veilig dingen uitproberen, onze tests opnieuw uitvoeren, en als de wijziging ons bevalt, kunnen we die implementeren. Zo niet, dan draaien we de wijziging gewoon terug. Deze vrijheid om te experimenteren is een van de echt grote voordelen van TDD.

Je dient bekend te zijn met de generics syntaxis [uit het vorige hoofdstuk](generics.md). Probeer je eigen `Reduce`-functie te schrijven en gebruik deze in `Sum` en `SumAllTails`.

### Tips

Als je eerst nadenkt over de argumenten van je functie, krijg je een zeer kleine set geldige oplossingen.
- De array die je wilt reduceren.
- Een soort combinerende functie.

"Reduce" is een ongelooflijk goed gedocumenteerd patroon, je hoeft het wiel niet opnieuw uit te vinden. [Lees de wiki, met name de sectie met lijsten](https://en.wikipedia.org/wiki/Fold_(higher-order_function)), het zou je in de richtin gmoeten duwen om een ​​ander argument te noemen dat je nodig hebt.

> In de praktijk is het handig en natuurlijk om een ​​beginwaarde te hebben

### Mijn eerste poging tot `Reduce`

```go
func Reduce[A any](collection []A, f func(A, A) A, initialValue A) A {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

Reduce legt de essentie van het patroon vast. Het is een functie die een verzameling, een optellende functie, een beginwaarde gebruikt en één waarde retourneert. Er zijn geen rommelige afleidingen rond concrete typen.

Als je de syntaxis van generieke typen begrijpt, zou je geen probleem moeten hebben met het begrijpen van wat deze functie doet. Door de erkende term `Reduce` te gebruiken, begrijpen programmeurs uit andere talen ook de bedoeling.

### Het gebruik

```go
// Sum calculates the total from a slice of numbers.
func Sum(numbers []int) int {
	add := func(acc, x int) int { return acc + x }
	return Reduce(numbers, add, 0)
}

// SumAllTails calculates the sums of all but the first number given a collection of slices.
func SumAllTails(numbers ...[]int) []int {
	sumTail := func(acc, x []int) []int {
		if len(x) == 0 {
			return append(acc, 0)
		} else {
			tail := x[1:]
			return append(acc, Sum(tail))
		}
	}

	return Reduce(numbers, sumTail, []int{})
}
```

`Sum` en `SumAllTails` beschrijven nu het gedrag van hun berekeningen als de functies die respectievelijk op hun eerste regels zijn gedeclareerd. Het uitvoeren van de berekening op de verzameling is geabstraheerd in `Reduce`.

## Verdere toepassingen van reduce

Met behulp van tests kunnen we experimenteren met onze reduce-functie om te zien hoe herbruikbaar deze is. Ik heb onze generic assertiefuncties uit het vorige hoofdstuk overgenomen.


```go
func TestReduce(t *testing.T) {
	t.Run("multiplication of all elements", func(t *testing.T) {
		multiply := func(x, y int) int {
			return x * y
		}

		AssertEqual(t, Reduce([]int{1, 2, 3}, multiply, 1), 6)
	})

	t.Run("concatenate strings", func(t *testing.T) {
		concatenate := func(x, y string) string {
			return x + y
		}

		AssertEqual(t, Reduce([]string{"a", "b", "c"}, concatenate, ""), "abc")
	})
}
```

### De nulwaarde

In het vermenigvuldigingsvoorbeeld laten we zien waarom er een standaardwaarde als argument voor `Reduce` is. Als we de standaardwaarde 0 van Go voor `int` zouden gebruiken, zouden we onze beginwaarde met 0 vermenigvuldigen, en vervolgens de daarop volgende, zodat je altijd 0 overhoudt. Door het op 1 in te stellen, blijft het eerste element in de slice hetzelfde en worden de restelementen vermenigvuldigd met de volgende elementen.

Als je slim wilt overkomen bij je nerdvrienden, noem je dit [Het Identiteitselement](https://en.wikipedia.org/wiki/Identity_element).

> In de wiskunde is een identiteitselement, of neutraal element, van een binaire bewerking die op een verzameling wordt uitgevoerd, een element van de verzameling dat elk element van de verzameling onveranderd laat wanneer de bewerking wordt toegepast.

Hier is het identiteitselement bijvoorbeeld 0.

`1 + 0 = 1`

Met vermenigvuldigen, is het 1.

`1 * 1 = 1`

## Wat als we willen reduceren tot een ander type dan `A`?

Stel dat we een lijst met transacties `Transactie` hebben en we een functie willen die deze transacties, plus een naam, verwerkt om hun banksaldo te berekenen.

Laten we het TDD-proces volgen.

## Schrijf eerst je test

```go
func TestBadBank(t *testing.T) {
	transactions := []Transaction{
		{
			From: "Chris",
			To:   "Riya",
			Sum:  100,
		},
		{
			From: "Adil",
			To:   "Chris",
			Sum:  25,
		},
	}

	AssertEqual(t, BalanceFor(transactions, "Riya"), 100)
	AssertEqual(t, BalanceFor(transactions, "Chris"), -75)
	AssertEqual(t, BalanceFor(transactions, "Adil"), -25)
}
```

## Probeer de test uit te voeren
```
# github.com/quii/learn-go-with-tests/arrays/v8 [github.com/quii/learn-go-with-tests/arrays/v8.test]
./bad_bank_test.go:6:20: undefined: Transaction
./bad_bank_test.go:18:14: undefined: BalanceFor
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

We hebben nog geen typen of functies, voeg deze toe om de test uit te voeren.

```go
type Transaction struct {
	From string
	To   string
	Sum  float64
}

func BalanceFor(transactions []Transaction, name string) float64 {
	return 0.0
}
```

Wanneer je de test uitvoert, zou je het volgende moeten zien:

```
=== RUN   TestBadBank
    bad_bank_test.go:19: got 0, want 100
    bad_bank_test.go:20: got 0, want -75
    bad_bank_test.go:21: got 0, want -25
--- FAIL: TestBadBank (0.00s)
```

## Schrijf genoeg code om de test te laten slagen

Laten we de code schrijven alsof we geen `Reduce`-functie hebben.

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	var balance float64
	for _, t := range transactions {
		if t.From == name {
			balance -= t.Sum
		}
		if t.To == name {
			balance += t.Sum
		}
	}
	return balance
}
```

## Refactor

Zorg op dit punt voor wat broncodebeheer en commit je werk. We hebben werkende software, klaar om Monzo, Barclays en anderen uit te dagen.

Nu ons werk gecommit is, kunnen we ermee experimenteren en verschillende ideeën uitproberen in de refactoringfase. Om eerlijk te zijn, de code die we hebben is niet bepaald slecht, maar voor deze oefening wil ik dezelfde code demonstreren met `Reduce`.

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	adjustBalance := func(currentBalance float64, t Transaction) float64 {
		if t.From == name {
			return currentBalance - t.Sum
		}
		if t.To == name {
			return currentBalance + t.Sum
		}
		return currentBalance
	}
	return Reduce(transactions, adjustBalance, 0.0)
}
```

Maar dit comileert niet.

```
./bad_bank.go:19:35: type func(acc float64, t Transaction) float64 of adjustBalance does not match inferred type func(Transaction, Transaction) Transaction for func(A, A) A
```

De reden is dat we proberen te reduceren naar een _ander_ type dan het type van de collectie. Dit klinkt eng, maar vereist eigenlijk alleen dat we de typesignatuur van `Reduce` aanpassen om het te laten werken. We hoeven de functiebody niet aan te passen, en we hoeven ook geen van onze bestaande aanroepen te wijzigen.

```go
func Reduce[A, B any](collection []A, f func(B, A) B, initialValue B) B {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

We hebben een tweede typebeperking toegevoegd waarmee we de beperkingen voor `Reduce` konden versoepelen. Dit stelt ons in staat om te `Reduce` te gebruiken van een verzameling `A` naar een `B`. In ons geval van `Transaction` naar `float64`.

Dit maakt `Reduce` algemener en herbruikbaarder, en nog steeds typeveilig. Als je de tests opnieuw uitvoert, zouden ze moeten compileren en slagen.

## De bank uitbreiden

Voor de lol wilde ik de ergonomie van de bankcode verbeteren. Om het kort te houden, heb ik het TDD-proces weggelaten.

```go
func TestBadBank(t *testing.T) {
	var (
		riya  = Account{Name: "Riya", Balance: 100}
		chris = Account{Name: "Chris", Balance: 75}
		adil  = Account{Name: "Adil", Balance: 200}

		transactions = []Transaction{
			NewTransaction(chris, riya, 100),
			NewTransaction(adil, chris, 25),
		}
	)

	newBalanceFor := func(account Account) float64 {
		return NewBalanceFor(account, transactions).Balance
	}

	AssertEqual(t, newBalanceFor(riya), 200)
	AssertEqual(t, newBalanceFor(chris), 0)
	AssertEqual(t, newBalanceFor(adil), 175)
}
```

En hier is de bijgewerkte code

```go
package main

type Transaction struct {
	From string
	To   string
	Sum  float64
}

func NewTransaction(from, to Account, sum float64) Transaction {
	return Transaction{From: from.Name, To: to.Name, Sum: sum}
}

type Account struct {
	Name    string
	Balance float64
}

func NewBalanceFor(account Account, transactions []Transaction) Account {
	return Reduce(
		transactions,
		applyTransaction,
		account,
	)
}

func applyTransaction(a Account, transaction Transaction) Account {
	if transaction.From == a.Name {
		a.Balance -= transaction.Sum
	}
	if transaction.To == a.Name {
		a.Balance += transaction.Sum
	}
	return a
}
```

Ik vind dat dit echt de kracht laat zien van concepten zoals `Reduce`. `NewBalanceFor` voelt meer _declaratief_ aan en beschrijft _wat_ er gebeurt in plaats van _hoe_. Vaak bladeren we tijdens het lezen van code door talloze bestanden en proberen we te begrijpen _wat_ er gebeurt in plaats van _hoe_, en deze codestijl maakt dit goed mogelijk.

Als ik me in de details wil verdiepen, kan ik dat doen, en ik kan de _business logic_ van `applyTransaction` zien zonder me zorgen te maken over lussen en muterende statussen; `Reduce` zorgt daar apart voor.

### Fold/reduce zijn behoorlijk universeel

De mogelijkheden zijn eindeloos™️ met `Reduce` (of `Fold`). Het is niet voor niets een veelgebruikt patroon, het is niet alleen voor rekenkunde of het samenvoegen van strings. Probeer eens een paar andere toepassingen.

- Waarom meng je `color.RGBA` niet tot één kleur?
- Tel het aantal stemmen in een poll of de items in een winkelmandje op.
- Zo'n beetje alles wat met het verwerken van een lijst te maken heeft.

## Find

Nu Go generieke functies heeft en deze combineert met functies van hogere orde, kunnen we veel boilerplate code binnen onze projecten reduceren, waardoor onze systemen gemakkelijker te begrijpen en te beheren zijn.

Je hoeft niet langer specifieke `Find`-functies te schrijven voor elk type collectie dat je wilt doorzoeken; in plaats daarvan kun je ze hergebruiken of een `Find`-functie schrijven. Als je de bovenstaande `Reduce`-functie begreep, is het schrijven van een `Find`-functie een fluitje van een cent.

Hier is een test

```go
func TestFind(t *testing.T) {
	t.Run("find first even number", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

		firstEvenNumber, found := Find(numbers, func(x int) bool {
			return x%2 == 0
		})
		AssertTrue(t, found)
		AssertEqual(t, firstEvenNumber, 2)
	})
}
```

En hier is de implementatie

```go
func Find[A any](items []A, predicate func(A) bool) (value A, found bool) {
	for _, v := range items {
		if predicate(v) {
			return v, true
		}
	}
	return
}
```

Omdat het een generiek type aanneemt, kunnen we het op veel manieren hergebruiken.

```go
type Person struct {
	Name string
}

t.Run("Find the best programmer", func(t *testing.T) {
	people := []Person{
		Person{Name: "Kent Beck"},
		Person{Name: "Martin Fowler"},
		Person{Name: "Chris James"},
	}

	king, found := Find(people, func(p Person) bool {
		return strings.Contains(p.Name, "Chris")
	})

	AssertTrue(t, found)
	AssertEqual(t, king, Person{Name: "Chris James"})
})
```

Zoals je kunt zien, is deze code foutloos.

## Samenvattend

Wanneer ze met smaak worden uitgevoerd, maken hogere-orde functies zoals deze je code eenvoudiger leesbaar en onderhoudbaar. Onthoud echter de volgende vuistregel:

Gebruik het TDD-proces om echt, specifiek gedrag te definiëren dat je daadwerkelijk nodig hebt. In de refactoringfase _ontdek_ je vervolgens mogelijk enkele nuttige abstracties om de code op te schonen.

Oefen met het combineren van TDD met goede gewoonten van broncodebeheer. Commit je werk wanneer je test slaagt, _voordat_ je probeert te refactoren. Op deze manier kun je, als je een puinhoop maakt, gemakkelijk terugkeren naar je werkende staat.

### Naamgeving doet ertoe

Doe je best om wat onderzoek buiten Go te doen, zodat je bestaande patronen niet opnieuw uitvindt met een reeds bestaande naam.

Een functie schrijven die een verzameling `A` omzet naar `B`? Noem het dan niet `Convert`, dat is [`Map`](https://en.wikipedia.org/wiki/Map_(higher-order_function)). Door de "juiste" naam voor deze items te gebruiken, wordt de cognitieve belasting voor anderen verminderd en wordt het zoekmachinevriendelijker om meer te leren.

### Voelt dit niet idiomatisch aan?

Probeer een open geest te hebben.

Hoewel de idiomen van Go niet _radicaal_ zullen veranderen en dat ook niet zouden moeten doen door de release van generieke versies, _zullen_ ze wel veranderen, door de taalverandering! Dit zou geen controversieel punt moeten zijn.

Zeggen

> Dit is niet idiomatisch

Zonder verdere details is het niet uitvoerbaar of nuttig om te zeggen. Vooral niet bij het bespreken van nieuwe taalfuncties.

Bespreek patronen en codestijlen met je collega's op basis van hun verdiensten in plaats van dogma's. Zolang je goed ontworpen tests hebt, kun je altijd dingen herstructureren en aanpassen naarmate je begrijpt wat goed werkt voor jou en je team.

### Bronnen

Fold is een echte basis in de computerwetenschap. Hier zijn enkele interessante bronnen als je je er verder in wilt verdiepen.
- [Wikipedia: Fold](https://en.wikipedia.org/wiki/Fold)
- [Een tutorial over de universaliteit en expressiviteit van fold (ENGELSTALIG)](http://www.cs.nott.ac.uk/~pszgmh/fold.pdf)
