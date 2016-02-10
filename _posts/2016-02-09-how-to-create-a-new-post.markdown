---
layout: post
title:  "How to create a new post!"
categories: basics
author: igl
img: http://jekyllrb.com/img/logo-2x.png
---

For å lage en ny post så kan du starte med å legge til en ny fil i _posts folderen.

##Front Matter

Alle blogposter må ha en YAML Front Matter blokk øverst i fila, denne forteller Jekyll at det er en spesiell fil og at den skal behandles i forhold til det som er definert i YAML-blokka. 
{% highlight markdown %}
---
layout: post
title:  "How to create a new post!"
---
{% endhighlight %}

Mellom disse triple-dashed linjene kan du sette [predefinerte variabler](http://jekyllrb.com/docs/frontmatter/) eller lage egne variabler. Disse variablene vil være tilgjengelig ved å bruke Liquid tags enten lenger ned i fila eller i layoutfiler.

For denne posten ser YAML-blokka slik ut:
{% highlight markdown %}
---
layout: post
title:  "How to create a new post!"
date:   2016-02-09 12:21:09 +0100
categories: basics
author: Ivar G.Lervold
authorlink: http://vg.no
img: http://jekyllrb.com/img/logo-2x.png
---
{% endhighlight %}

Det er lagt til et par egendefinerte variabler, img og authorlink.

* **img** definerer hvilket bilde som skal brukes for posten på førstesiden
* **authorlink** burkes for å legge til link til skribenten (twitter, linkedin, privat blogg etc)

Disse variablene blir brukt i index.html-fila og legges til posten i listen over poster.

#Navngiving av fila

Jekyll krever at fila navngis etter følgende format:
```
ÅR-MÅNED-DAG-tittel.MARKUP
```

Der ``ÅR`` er et firesifret nummer, ``MÅNED`` og ``DAG`` er begge tosifrede nummer, og ``MARKUP`` er fil-extension typen. 

