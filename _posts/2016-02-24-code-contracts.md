---
layout: post
title:  "Code Contracts i .NET"
author: ori
tags: [CSharp]
---

I DIPS har vi en stund hatt litt diskusjon rundt ulike former for enhetstesting. Vi bruker enhetstesting i stor grad i våre prosjekter, men med varierende detaljnivå. Utfordringen her er at enhetstester som opererer på lave detaljnivåer er kostbare å vedlikeholde, selv om de kanskje gir god code coverage. Men hva med [functional coverage](https://coding.abel.nu/2014/07/code-coverage-functional-coverage/)? Kanskje får man dette med på kjøpet, kanskje ikke. Det kommer an på hvor flink man er til å skrive gode tester.

<!--more-->

Ifm. denne diskusjonen har jeg eksperimentert litt med innføring av Code Contracts i [Kjernejournal](https://helsenorge.no/kjernejournal)-integrasjonen for DIPS Arena. Målet med dette var å finne ut hva Code Contracts kan bidra med i vår kode, og hva som eventuelt er kostnaden ved å innføre dette. Jeg har tidligere aldri brukt Code Contracts, og har generelt lite erfaring med statisk sjekking av kode utover det ReSharper tilbyr, så dette ble et nytt tankesett. Oppsummeringen som følger er en blanding av mine egne og andres erfaringer ved bruk av Code Contracts i .NET.

## Litt om Code Contracts i .NET
[Code Contracts](https://github.com/Microsoft/CodeContracts) er et verktøy for å innføre *conditions* i koden din som alltid må være sanne når de kjøres. Dette kan sjekkes både statisk og runtime. Kort fortalt bruker man *preconditions* for å verifisere input og state ved starten av en metode, *postconditions* for å verifsere output og state ved slutten av en metode, og *invariants* for å definere conditions som alltid skal gjelde for en gitt klasse. Det finnes flere gode kilder til informasjon om .NET Code Contracts, og det anbefales å bla gjennom disse før man går ordentlig i gang med Code Contracts:

- [Code Contracts på MSDN](https://msdn.microsoft.com/en-us/library/dd264808(v=vs.110).aspx)
- [Code Contracts User Manual]
- [The CodeContracts static checker](http://research.microsoft.com/en-US/projects/contracts/cccheck.pdf)

Å innføre Code Contracts i en eksisterende kodebase kan være en utfordring, spesielt for å få på plass god statisk analyse, og da er det viktig at man bruker en [strukturert framgangsmåte](http://blog.xoc.net/2014/03/10-steps-for-implementing-code.html).

Code Contracts tools kan installeres som [NuGet-pakke](https://www.nuget.org/packages/DotNet.Contracts/) eller som [Visual Studio extension](https://visualstudiogallery.msdn.microsoft.com/1ec7db13-3363-46c9-851f-1ce455f66970). Sistnevnte ser av en eller annen grunn ikke ut til å være oppdatert med nyeste versjon per i dag.

Ute i den store verden finnes det litt ulike syn på nytten av Code Contracts i .NET, og mye bærer preg av at .NET Code Contracts ikke er helt modent enda. Noen eksempler, som taler både for og imot bruk av .NET Code Contracts:

- [Jarrett Meyer, Learning Code Contracts](http://jarrettmeyer.com/blog/2013/12/30/learning-code-contracts):

    > Code contracts are a debugging tool. They are not a replacement for unit tests. Good code contracts and good unit tests work together.

- [Microsoft Code Contracts: Not with a Ten-foot Pole](http://blogs.encodo.ch/news/view_article.php?id=170) (denne er fra 2009, men noe av det som tas opp er fortsatt gjeldende)

    > Without explicit language support, a DBC solution couched in terms of assertions and/or exceptions quickly leads to clutter that obscures the actual program logic.

- [Code Contracts is the next coding practice you should learn and use](http://codebetter.com/patricksmacchia/2013/12/18/code-contracts-is-the-next-coding-practice-you-should-learn-and-use/)

    > By using contracts you make unit-tests much more effective.

- [Code Contracts and Test-Driven Development](http://discoveringcode.blogspot.no/2014/03/code-contracts-and-test-driven.html)

    > The great thing about Code Contracts is that you don’t need to redefine them on all of your sub-types.

## Runtime checking
Når man først starter med Code Contracts, anbefales det å begynne med runtime checking. Det er flere måter å gjøre runtime checking på, og [Code Contracts User Manual] gir følgende hjelp til å beslutte hva som passer for et gitt prosjekt:

<img usage_guidelines.PNG>

Når man gjør denne vurderingen er det noen faktorer man må ta hensyn til. [Runtime checking vil kunne påvirke ytelsen](http://www.codeproject.com/Articles/181411/Code-Contract-Performance-Analysis), men omfanget vil variere avhengig av hvilke kontrakter man implementerer. `null` checks og range-checks på f.eks. integers koster ganske lite, mens collection quantifiers gjerne koster litt mer. Hvis man ikke ønsker runtime checking i release, går man for **Usage 1**.

Merk også følgende advarsel, som kan påvirke et eventuelt valg mellom **Usage 2** og **Usage 3**:

> The risk of using the contract tools in your release build is that you depend on tools that have not reached production quality level.

Når man innfører Code Contracts i en eksisterende kodebase kan det være greit å bruke *legacy requires* i større grad enn det som vises i figuren over, fordi det kanskje allerede finnes en god del *if-then-throw* -kode.

For Kjernejournal-integrasjonen i Arena valgte jeg å gå for **Usage 2**. Siden dette er "pilotprosjekt" for Code Contracts hos oss, ville jeg prøve å få til mest mulig validering. Kjernejournal-integrasjonen i Arena er ikke i produksjon enda, så det er foreløpig ikke noe problem å være avhengig av Code Contract tools for release-bygg. I tillegg er det ingen andre prosjekter som er avhengige av Kjernejournal-integrasjonen, så *public surface methods* er i praksis ikke-eksisterende.

Et par andre ting som er greit å vite om runtime checking:

- Når runtime checking er skrudd på, gjør Code Contracts en *rewrite* av assemblyet. Fra manualen:
> It applies the contract rewriter ccrewrite to the target assembly, performing well-formedness checks on
the contracts, instrumenting contract runtime checks into the appropriate places, including contract
inheritance

- De fleste kontraktmetodene er i utgangspunktet *conditionally compiled*, definert av CONTRACTS\_FULL. I Visual Studio er det mulig å styre dette fra project properties i stedet for å definere CONTRACTS\_FULL overalt.

### Hvilke fordeler gir det oss å bruke runtime checking?

**Bedre enhetstester:** Med Code Contracts kan man definere en del forventa oppførsel direkte i koden, i stedet for å skrive haugevis av tester som f.eks. verifiserer riktig oppførsel gitt ugyldige parametre. Typiske kandidater her er `null` eller range-checks. I stedet kan man fokusere på mer funksjonelle enhetstester, fordi kontraktene kjøres uansett. Dersom en kontrakt feiler, kastes en `ContractException`, og testen feiler (selv om testen i utgangspunktet er ment for å teste noe annet).

**Enklere å identifisere feil under testing:** Når det oppstår feil, er det ikke alltid like enkelt å finne ut hvor denne oppstår (spesielt ikke med omfattende bruk av async/await). Med Code Contracts er det enklere å identifisere når tilstanden i en gitt klasse eller parametre inn i en metode ikke er som forventa (før det eventuelt feiler et annet sted), og gi tydelig tilbakemelding om dette.

## Static checking
Først, en advarsel fra [Code Contracts User Manual]:
> Static code checking or verification is a difficult endeavor. It requires a relatively large effort in terms of writing contracts, determining why a particular property cannot be proven, and finding a way to help the checker see the light. Before you start using the static contract checker in earnest, we suggest you spend enough time using contracts for runtime checking to familiarize yourself with contracts and the benefits they bring in that
domain.

Med static checking forsøker Code Contracts å analysere koden din for å finne ut om det er sannsynlig at en kontrakt vil holde under kjøring. Dette er ganske krevende, og baserer seg både på kontrakter andre steder i koden og antakelser om oppførsel i eksterne APIer. Merk at static checking har en del begrensninger, og analyzeren er ikke veldig smart. Mer om dette senere.

I Visual Studio er det mange forskjellige options for static checking man kan skru på. Man kan for eksempel få analyzeren til å sjekke spesifikke ting, automatisk gjøre en del antakelse, eller foreslå contract definitions. I tillegg kan man velge hvilket nivå av warnings som skal vises, og om bygget skal feile dersom analyzeren gir warnings (merk at i gjeldende versjon feiler denne også på warnings som ikke er relatert til Code Contracts). Her er det lurt å starte på et lavt nivå av warnings, og jobbe seg oppover etter hvert som man får kontraktene til å fungere. På høyere nivåer vil analyzeren prøve å verifisere mer og mer av koden for å sjekke at kontraktene holder, og det blir etter hvert ganske vanskelig å "bevise" for den at det stemmer.

### Hvilke fordeler gir det oss å bruke static checking?
Static checking hjelper oss i hovedsak med å oppdage feil i programflyten basert på de kontraktene vi har satt opp, compile-time.

Da jeg skulle innføre Code Contracts i Kjernejournal-integrasjonen endte jeg opp med å skrive om en del kode for å få det til å fungere bedre med Code Contracts, og for å hjelpe analyzeren med å verifisere resultatet. For eksempel oppdaget jeg et par tilfeller der jeg hadde unødvendig state i en klasse, og fjernet dermed denne. Stort sett vil jeg påstå at koden ble *bedre* av dette, og at det å innføre/bruke Code Contracts gir en effekt tilsvarende TDD når det gjelder kodekvalitet.

### Hvilke problemer/ulemper kan static checking føre til?
**Tid:** Static checking er en ganske krevende operasjon, og avhengig av størrelsen på prosjektet kan det ta ganske lang tid. I Visual Studio kan dette kjøre i bakgrunnen, men når man kjører bygg f.eks. på byggeserver skjer dette synkront. I Kjernejournal-integrasjonen økte byggetiden fra 25s til 35s (inkludert kjøring av enhetstester, NuGet restore, og lignende). 10 sekunder er ikke mye, men Kjernejournal-integrasjonen er ganske liten, og det tilsvarer en økning på 40%.

**Contracts er vanskelige å bevise / jeg får for mange warnings**: Sånn er det bare. Sett warning-nivået til det du mener er fornuftig for ditt prosjekt. I Kjernejournal-integrasjonen endte jeg opp med nivå 3 *(more warnings)*.

At contracts er vanskelige å bevise skyldes i noen grad at Code Contracts for .NET ikke er helt modent enda. Jeg opplevde for eksempel at analyzeren aksepterte følgende kode når det ble sendt inn gyldig verdi:

{% highlight csharp %}
Contract.Requires(byteArray != null && byteArray.Length >= 0);
{% endhighlight %}

... mens den ikke klarte å verifisere dette:

{% highlight csharp %}
Contract.Requires(ByteArrayIsValid(byteArray));

...

private static void ByteArrayIsValid(byte[] byteArray)
{
	return byteArray != null && byteArray.Length >= 0;
}
{% endhighlight %}

Jeg har også hatt trøbbel med å verifisere kontrakter på følgende form:

{% highlight csharp %}
Contract.Requires((someCondition && someAdditionalCondition) || !someCondition);
{% endhighlight %}

Code Contracts sliter også med **extension methods**. Gitt følgende kode:

{% highlight csharp %}
public static async Task<T> ExtractAndDeserializeContentAsync<T>
(this HttpResponseMessage httpResponseMessage)
{
	...
}
{% endhighlight %}

... påsto analyzeren at den ikke kunne verifisere at `httpResponseMessage` ikke var null.

`String.IsNullOrWhiteSpace()`: Hvis du ser følgende warning:
{% highlight csharp %}
Detected call to method 'System.String.IsNullOrWhiteSpace(System.String)' without [Pure] in contracts of method [...]
{% endhighlight %}

... skjer dette pga en feil i Code Contracts for .NET 4.6.1. Se [tilsvarende spørsmål på StackOverflow](https://stackoverflow.com/questions/34612382/how-to-deal-with-code-contracts-warning-cc1036-when-using-string-isnullorwhitesp/) for mulige løsninger.

**Andre problemer?** Sjekk [åpne issues på GitHub](https://github.com/Microsoft/CodeContracts/issues), eller [spør StackOverflow](https://stackoverflow.com/search?q=code+contracts).

## Andre tanker og kommentarer
**Invariants og readonly:** I project properties kan man velge et alternativ som heter *Infer invariants for readonly*. Dette betyr at analyzeren antar at `readonly`-felter alltid har samme verdi når den kjører static checking. Dette fungerte ikke som forventa i Kjernejournal-prosjektet, og jeg måtte eksplisitt definere `Contract.Invariant()` for readonly-felter for å få det til å fungere. Dette fører til en del støy i koden.

**Kontrakter på interfaces**: Hvis man ønsker kontrakter på metoder som defineres av interfaces, må disse plasseres i en egen abstrakt kontraktklasse, markert med attributtet `[ContractClassFor(...)]`. Dette kan være både en fordel og en ulempe. Fordelen er at kontrakten defineres et annet sted, slik at koden blir mer ryddig. Ulempen er at kontrakten defineres et annet sted, slik at den blir mindre synlig. I klasser det man i tillegg vil ha kontrakter på private metoder (kan være relevant for store klasser med mye logikk) vil det bety at noen kontrakter er definert inline, mens andre er definert i kontraktklassen, og da kan det fort bli uoversiktlig.

Merk at når kontraktene er definert for et interface, betyr dette også at de gjelder for [alle implementasjoner](https://en.wikipedia.org/wiki/Liskov_substitution_principle).

En relatert utfordring er at preconditions i public-metoder ikke kan referere private felter, fordi dette ikke kan verifiseres av den som kaller metoden. Det betyr at hvis man ønsker å sjekke et privat felt som en del av en kontrakt, må denne eksponeres gjennom en public property.

## Noen tips til slutt

- Bruk [[Pure]](https://msdn.microsoft.com/en-us/library/system.diagnostics.contracts.pureattribute(v=vs.110).aspx)!
- Hvis samme kontrakt brukes flere steder, vurder [[ContractAbbreviator]](https://msdn.microsoft.com/en-us/library/system.diagnostics.contracts.contractabbreviatorattribute(v=vs.110).aspx).
- Bruk `Contract.Ensures()` for å definere hvilke returverdier som er gyldige. Dette er til stor hjelp for analyzeren ved static checking.
- Hvis det ikke finnes andre måter å verifisere en kontrakt på (f.eks. ved bruk av eksterne APIer), bruk `Contract.Assume()` for å hjelpe analyzeren med å verifisere koden ved static checking.
- Ved innføring i eksisterende prosjekter, start på laveste nivå i koden din, med klasser og metoder som ikke har så mange avhengigheter, og jobb deg oppover etter hvert som kontraktene kan verifiseres.

[Code Contracts User Manual]: http://research.microsoft.com/en-us/projects/contracts/userdoc.pdf

## Concluding remarks
Selv om Code Contracts for .NET ikke er helt modent enda, og det til tider kan være veldig tungvint å få verifisert en kontrakt, er jeg positiv til å ta det i bruk, i hvert fall i noen grad. Kontraktene bidrar til et tankesett som kan skape bedre kode, og static checking hjelper til med å finne en del logiske feil eller mangler. Best av alt, så bidrar runtime checking til litt ryddigere enhetstester. Den store ulempen er nok at det krever mye tid å innføre dette i eksisterende prosjekter, og tid er ikke noe vi har for mye av.
