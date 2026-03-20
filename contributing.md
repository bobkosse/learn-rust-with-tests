# Bijdragen

Bijdragen zijn van harte welkom. Ik hoop dat dit een geweldige plek wordt voor handleidingen over hoe je Go kunt leren door tests te schrijven. Overweeg om een PR in te dienen of een issue aan te maken, wat je [hier](https://github.com/quii/learn-go-with-tests/issues) kunt doen.

## Wat we zoeken

* Go-functies aanleren (bijv. dingen zoals `if`, `select`, structs, methoden, enz.).
* Interessante functionaliteit binnen de standaardbibliotheek laten zien. Laat bijvoorbeeld zien hoe gemakkelijk het is om een HTTP-server te TDD'en.
* Laat zien hoe Go's tooling, zoals benchmarking, racedetectoren, enz. je kunnen helpen bij het ontwikkelen van geweldige software.

Als je je niet zeker genoeg voelt om je eigen handleiding in te dienen, is het indienen van een issue voor iets wat je wilt leren nog steeds een waardevolle bijdrage.

### ⚠️ Krijg snel feedback op nieuwe content ⚠️

- TDD leert ons om iteratief te werken en feedback te krijgen, en ik raad je ten zeerste aan hetzelfde te doen als je wilt bijdragen.
- Open een PR met je eerste test en implementatie, bespreek je aanpak zodat ik feedback kan geven en bij kan sturen.
- Dit is natuurlijk open source, maar ik heb wel een uitgesproken mening over de content. Hoe eerder je met me praat, hoe beter.

## Stijlgids

* Blijf de TDD-cyclus benadrukken. Bekijk de [Hoofdstuksjabloon](template.md).
* De nadruk ligt op iteratie over functionaliteit die door tests wordt aangestuurd. Het Hello, world-voorbeeld werkt goed omdat we het geleidelijk aan geavanceerder maken en nieuwe technieken leren die door de tests worden aangestuurd. Bijvoorbeeld:
  * `Hello()` &lt;- hoe je functies en retourtypen schrijft.
  * `Hello(naam string)` &lt;- argumenten, constanten.
  * `Hello(naam string)` &lt;- standaard "world" met `if`.
  * `Hello(naam, taal string)` &lt;- `switch`.
* Probeer de oppervlakte van de vereiste kennis te minimaliseren.
  * Het is belangrijk om voorbeelden te bedenken die laten zien wat je probeert te onderwijzen zonder de lezer te verwarren met andere functies.
  * Je kunt bijvoorbeeld `struct`s leren zonder pointers te begrijpen.
  * Beknoptheid is koning.
* Volg de [Code Review Comments style guide](https://go.dev/wiki/CodeReviewComments). Dit is belangrijk voor een consistente stijl in alle secties.
* Je sectie moet een uitvoerbare applicatie aan het einde hebben (bijv. `package main` met een `main`-functie) zodat gebruikers deze in actie kunnen zien en ermee kunnen spelen.
* Alle tests moeten slagen.
* Voer `./build.sh` uit voordat je de PR genereert.
