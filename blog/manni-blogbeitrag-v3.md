---
title: "manni: Wenn die Doku beweisen muss, dass sie noch stimmt"
description: "Ein CLI-Werkzeugkasten für prüfbare Softwaredokumentation: Metadaten gegen JSON Schema, Sätze gegen Quellzeilen, Seiten gegen axe-core."
author: Matthias Eckardt
---

# Wenn die Doku beweisen muss, dass sie noch stimmt

In jeder Docs-as-Code-Pipeline, die ich kenne, gibt es einen blinden Fleck. [Vale](https://vale.sh/) prüft Sprache gegen einen Styleguide. Der Linkchecker prüft Verweise. Der Build prüft, ob die Seite rendert. [Doc Detective](https://docs.doc-detective.com/) prüft, ob sich das Produkt so verhält, wie es beschrieben ist. Und keiner von ihnen merkt, dass der Standardwert für das Fetch-Timeout vor drei Sprints von 10 auf 30 Sekunden gestiegen ist, während auf Seite 9 der API-Doku weiterhin 10 steht.

Das Problem ist weder neu noch anekdotisch. Die Software-Engineering-Forschung untersucht es seit Jahren unter dem Stichwort *code-comment inconsistency*. Wen et al. haben dafür 1,3 Milliarden AST-Änderungen aus der Historie von 1.500 Systemen ausgewertet und gezeigt, wie selten Kommentare tatsächlich mit dem Code fortgeschrieben werden, den sie beschreiben ([Wen et al. 2019](https://www.inf.usi.ch/lanza/Downloads/Wen2019a.pdf)). Benutzerdokumentation liegt weiter vom Code entfernt als der Kommentar direkt über der Funktion. Ihre Ausgangslage ist damit schlechter, nicht besser.

Diese Lücke ist der Grund, warum [manni](https://hawkeyexl.github.io/manni/) einen Blick wert ist.

## Was kann ich mit manni lösen?

`manni` ist eine Familie von Kommandozeilenwerkzeugen für Dokumentation, die geprüft werden soll. Ein npm-Paket, vier Werkzeuge, eine gemeinsame Konfigurationsdatei. Der Aufruf folgt immer demselben Muster:

```bash
npm install -D @hawkeyexl/manni
npx @hawkeyexl/manni meta validate docs/
```

Die Domäne nach `manni` ist das Werkzeug: `meta`, `cite`, `a11y`, `key`. Jedes hat eigene Unterbefehle, einen eigenen Bereich in der Dokumentation und einen eigenen Schlüssel in der `manni.config.yaml` ([Projektübersicht](https://hawkeyexl.github.io/manni/)).

Hinter dem Projekt steht Manny Silva, Entwickler von [Doc Detective](https://docs.doc-detective.com/) und Autor von *Docs as Tests* ([Silva 2025](https://www.barnesandnoble.com/w/docs-as-tests-manny-silva/1147310716)). Das erklärt die Handschrift: Jede Prüfung ist ein CLI-Befehl mit definiertem Exit-Code, gebaut nach den Konventionen von [clig.dev](https://clig.dev/), und jede Prüfung kann einen Build scheitern lassen. Das Paket steht unter [MIT-Lizenz](https://github.com/hawkeyexl/manni/blob/main/LICENSE) und setzt Node.js 24 oder neuer voraus ([README](https://github.com/hawkeyexl/manni#install)).

## meta: sind die Metadaten da, und stimmt ihr Format?

`manni meta` ist der ältere Teil der Familie. Es wurde bis Version 4.13.1 als `docmeta` veröffentlicht und ist unter dem neuen Namen dieselbe Software geblieben. Der alte `docmeta`-Bin läuft weiter, und die alten Schema-URLs werden weiter ausgeliefert ([Migrationshinweise](https://github.com/hawkeyexl/manni#coming-from-docmeta)).

Das Werkzeug liest Frontmatter oder Header aus den Quelldateien und validiert sie gegen ein oder mehrere [JSON Schemas](https://json-schema.org/). Es prüft Präsenz und Format: Ist ein `type` gesetzt? Ist der Zeitstempel wirklich ISO 8601? Ist die `resource` eine URI? Mehr nicht. Es bewertet keine Prosa, prüft keine Rechtschreibung, rendert nichts ([Werkzeugübersicht](https://hawkeyexl.github.io/manni/meta/)).

```
✗ docs/intro.md
    (root)      must have required property 'type'   (line 1)  [google:okf:0.1]
    /timestamp  must match format "date-time"        (line 9)  [google:okf:0.1]

1 file checked, 0 passed, 1 failed, 0 errors
```

Bemerkenswert ist der Schema-Vorrat. 23 Schemas sind eingebaut, und sie decken zwei verschiedene Ebenen ab ([Schema-Registry](https://hawkeyexl.github.io/manni/meta/reference/built-in-schemas/)). Auf der inhaltlichen Ebene stehen Vokabulare wie das [Open Knowledge Format](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md), [Diátaxis](https://diataxis.fr/), [The Good Docs Project](https://www.thegooddocsproject.dev/template) und das [Seven-Action-Modell](https://passo.uno/seven-action-model/). Auf der technischen Ebene stehen die Frontmatter-Schemata der Generatoren: [Hugo](https://gohugo.io/content-management/front-matter/), [Jekyll](https://jekyllrb.com/docs/front-matter/), [Docusaurus](https://docusaurus.io/docs/api/plugins/@docusaurus/plugin-content-docs#markdown-front-matter), [MkDocs Material](https://squidfunk.github.io/mkdocs-material/reference/), [Starlight](https://starlight.astro.build/reference/frontmatter/), [Antora](https://docs.antora.org/antora/latest/page/attributes/) und [Sphinx](https://www.sphinx-doc.org/en/master/usage/restructuredtext/field-lists.html).

Wer also eine Antora-Site betreibt und wissen will, ob die Seitenattribute konsistent gesetzt sind, muss kein eigenes Schema schreiben. Unterstützt werden Markdown, MDX, AsciiDoc, reStructuredText, XML einschließlich [DITA](https://www.oasis-open.org/standard/dita-v1-3/)-Topics und -Maps sowie HTML ([Formatreferenz](https://hawkeyexl.github.io/manni/meta/reference/formats/)).

Der zweite Befehl heißt `fill` und ist der pragmatische Teil: Er lässt ein Sprachmodell die fehlenden Felder vorschlagen und schreibt zurück, was oberhalb einer Konfidenzschwelle liegt (Standard 0,7). Ein Vorschlag muss außerdem das Subschema des Zielfelds erfüllen und das Dokument nach dem Merge valide lassen. Der Anbieter wird erkannt, in der Reihenfolge `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, angemeldete `claude`-CLI, lokales Modell. Mit `--local` läuft die Inferenz auf dem Rechner, und jeder gehostete Anbieter wird abgelehnt ([fill-Referenz](https://hawkeyexl.github.io/manni/meta/reference/cli/#meta-fill)).

Für Umgebungen mit strikter Data-Egress-Policy ist `--local` der entscheidende Schalter. Und ein Detail, das man in CI wissen sollte: Ohne `--provider` fällt ein Runner, der seinen API-Key verliert, stillschweigend auf das lokale Modell zurück und lädt es herunter, statt den Build rot zu machen. Anbieter also festnageln.

## cite: der Satz und die Zeile, auf der er steht

`manni cite` ist der Teil, für den sich der Blick auf das Projekt wirklich lohnt, und zugleich der Teil, der konzeptionell am meisten verlangt.

Die Idee: Ein Satz in der Doku wird an die Quellzeilen gebunden, auf denen er ruht. Ein Zitat hat zwei Enden. Das eine Ende ist die Behauptung, also die Zeilen auf der Seite plus ein Hash darüber. Das andere Ende ist die Quelle, also Datei, Zeilenbereich, Hash und der Commit, in dem gelesen wurde ([Zitatreferenz](https://hawkeyexl.github.io/manni/cite/reference/citations/)).

```bash
npx @hawkeyexl/manni cite add docs/limits.md:9 lib/limits.ts:2 --id fetch-timeout
```

Danach prüft `manni cite check` beide Enden. Ändert jemand das Timeout im Code, meldet der Lauf nicht „diese Seite könnte veraltet sein“, sondern:

```
✗ docs/limits.md
    ✗ fetch-timeout   :9 current   lib/limits.ts:2 changed since 9265563, 1 commit
```

Links steht der Zustand der Doku-Seite, rechts der Zustand der Quelle. Der Satz in Zeile 9 ist unverändert, die Quellzeile nicht. Der Befund hängt damit an genau der Zeile, die ein Mensch anfassen muss: Exit-Code 1, CI rot, Annotation an Ort und Stelle ([Befundstatus und Exit-Codes](https://hawkeyexl.github.io/manni/cite/fix/)).

Drei Eigenschaften entscheiden darüber, ob so etwas brauchbar ist oder nur Alarmmüll produziert:

**Es läuft aus Git allein.** Kein Modell, kein Netzwerk, ein Hash pro Ende. Das heißt: deterministisch, reproduzierbar, ohne Token-Budget und ohne Rate-Limit in der Pipeline. Für ein Gate, das bei jedem Pull Request läuft, ist das kein Nebenaspekt, sondern die Voraussetzung.

**Verschoben ist nicht geändert.** Hat eine Zeile nur Nachbarn über sich bekommen, gilt sie als `moved`, und ein Befehl repariert die Pins. Nur wenn sich die Bytes geändert haben, wird ein Mensch gefragt. Genau an dieser Unterscheidung scheitern selbstgebaute Lösungen meistens.

**Auf der Seite steht der Satz nicht doppelt.** Das Zitat wiederholt den Satz nicht im Frontmatter, sondern hasht seine Zeilen. Eine Korrektur daneben löst deshalb keinen Fehlalarm aus. Wer die Zitate ganz aus der Seite heraushalten will, kann sie in ein [Sidecar-Manifest](https://hawkeyexl.github.io/manni/cite/set-up/citations-in-a-sidecar/) schreiben.

Die Grenzen benennt die Dokumentation selbst: Das Werkzeug beurteilt nicht, ob der Satz jemals richtig war, und es liest die Quelle nicht auf Bedeutung. Ein `changed` ist eine Frage an einen Menschen, nichts weiter. Die Leistung liegt allein im Zuschnitt der Frage.

## key: öffentliche Doku über privaten Code

`manni key` ist das kleinste Werkzeug der Familie und existiert für einen einzigen Fall: Die Dokumentation ist öffentlich, der Code nicht. Ein Zitat müsste dann interne Dateipfade und Zeilennummern preisgeben.

Der Ausweg sind verschlüsselte Quellangaben. `key` setzt und rotiert den Schlüssel, den alle vier Werkzeuge gemeinsam nutzen ([key-Übersicht](https://hawkeyexl.github.io/manni/key/)). Das Zitat bleibt damit prüfbar, ohne die Struktur des Repositories zu verraten ([Public docs, private code](https://hawkeyexl.github.io/manni/cite/set-up/public-docs-private-code/)). Wer Doku und Code ohnehin im selben öffentlichen Repository hält, braucht das Werkzeug nicht.

## a11y: axe-core über die ganze Site

`manni a11y check` nimmt eine URL, crawlt die Site dahinter und lässt [axe-core](https://github.com/dequelabs/axe-core) gegen jede erreichte Seite laufen. Die Seitenliste kommt zuerst aus der Sitemap, dann aus den gerenderten Links. Konfigurierbar sind unter anderem Tag-Filter (etwa nur `wcag2a` und `wcag2aa` nach [WCAG 2.1](https://www.w3.org/TR/WCAG21/)), eine Schwere-Untergrenze und eine Seitenobergrenze ([a11y-Übersicht](https://hawkeyexl.github.io/manni/a11y/)).

Jede Seite bekommt einen Wert zwischen 0 und 100:

```
score = round(100 × passes ÷ (passes + violations))
```

Die Dokumentation warnt ausdrücklich davor, das mit einem Lighthouse-Score zu verwechseln. Der [Lighthouse-Accessibility-Score](https://developer.chrome.com/docs/lighthouse/accessibility/scoring) ist ein gewichteter Mittelwert, dessen Gewichte sich an den User-Impact-Bewertungen von axe orientieren. Bei `manni a11y` zählt nur der Anteil bestandener Regeln. Eine Seite mit einem Fehler und 30 bestandenen Regeln bekommt 97 und fällt trotzdem durch. Der Score ist ein Trend, der Exit-Code ist das Urteil.

Wichtiger als die Formel ist, was ein grüner Lauf nicht beweist. Das axe-core-Repository selbst beziffert die automatisch auffindbaren WCAG-Probleme auf durchschnittlich 57 Prozent und weist alles, was die Engine nicht entscheiden kann, als `incomplete` zur manuellen Prüfung aus ([axe-core README](https://github.com/dequelabs/axe-core)). Diese Zahl stammt aus einer Deque-Studie über rund 13.000 Seiten aus eigenen Erstaudits ([Deque 2021](https://www.deque.com/automated-accessibility-coverage-report/)) und misst den Anteil an der Gesamtzahl gefundener Befunde. Der ältere und vorsichtigere Wert von 20 bis 30 Prozent bezieht sich auf den Anteil der WCAG-Erfolgskriterien, die maschinell überhaupt vollständig prüfbar sind. Beide Zahlen sind korrekt, sie beantworten nur verschiedene Fragen.

Das ist im deutschsprachigen Raum mehr als eine akademische Unterscheidung. Seit dem 28. Juni 2025 gilt das Barrierefreiheitsstärkungsgesetz, mit dem Deutschland den [European Accessibility Act](https://eur-lex.europa.eu/eli/dir/2019/882/oj) umsetzt; maßgeblich für Websites ist die Norm EN 301 549, die ihrerseits auf WCAG 2.1 Level AA verweist ([IHK Köln](https://www.ihk.de/koeln/hauptnavigation/recht-steuern/barrierefreiheit-von-webseiten-dienstleistungen-und-produkten-5921172)). Ob eine Dokumentationsseite darunter fällt, hängt davon ab, ob über sie eine verbraucherbezogene Dienstleistung erbracht wird. Reine B2B-Angebote sind ausgenommen. Wer aber ein grünes `manni a11y` gegenüber Stakeholdern als Konformitätsnachweis präsentiert, präsentiert etwas, das er nicht hat.

Ein `--fix` gibt es nicht, und die Begründung ist sauber: Die Prüfung sieht nur die gerenderte Seite, nie das Markdown, das Template oder den Komponentenbaum, der sie erzeugt hat ([Fix a failing check](https://hawkeyexl.github.io/manni/a11y/fix/)). Der Browser wird beim Paketinstall nicht mitgeliefert und muss einmalig eingerichtet werden.

## Der Gegenentwurf: einbinden statt zitieren

Bevor jemand anfängt, Sätze zu pinnen, gehört eine Alternative auf den Tisch, die es seit Jahren gibt: den Wert nicht beschreiben, sondern einbinden. AsciiDoc kennt [getaggte Include-Regionen](https://docs.asciidoctor.org/asciidoc/latest/directives/include-tagged-regions/), Sphinx hat [`literalinclude`](https://www.sphinx-doc.org/en/master/usage/restructuredtext/directives.html#directive-literalinclude) mit `:start-after:` und `:end-before:`, MkDocs hat [snippets](https://facelessuser.github.io/pymdown-extensions/extensions/snippets/). Was eingebunden ist, kann nicht driften.

Die Grenze dieses Ansatzes ist der Satz selbst. Ein Codeblock lässt sich einbinden. „Der Standardwert beträgt 10 Sekunden, weil längere Wartezeiten den Verbindungspool blockieren“ lässt sich nicht einbinden, denn diese Aussage steht nirgends im Code. Sie ruht nur auf einer Zeile. Genau dafür ist `cite` gemacht, und genau darin liegt die Arbeitsteilung: Includes für das Wörtliche, Zitate für das Erklärende. Wer beides verwechselt, pinnt Dinge, die er hätte einbinden sollen.

## Wo das in die Werkzeugkette passt

Die Werkzeuge beantworten verschiedene Fragen, und keine davon wird von den üblichen Verdächtigen schon beantwortet:

| Frage | Werkzeug |
|---|---|
| Ist die Sprache regelkonform? | [Vale](https://vale.sh/) |
| Verhält sich das Produkt wie beschrieben? | [Doc Detective](https://docs.doc-detective.com/) |
| Trägt die Seite die Metadaten, die die Pipeline braucht? | [`manni meta`](https://hawkeyexl.github.io/manni/meta/) |
| Gilt der Satz über den Code noch? | [`manni cite`](https://hawkeyexl.github.io/manni/cite/) |
| Ist die publizierte Seite barrierefrei? | [`manni a11y`](https://hawkeyexl.github.io/manni/a11y/) |
| Bleiben die Quellpfade dabei privat? | [`manni key`](https://hawkeyexl.github.io/manni/key/) |

`meta` prüft, was eine Seite über sich selbst behauptet. `cite` prüft, was sie über den Code behauptet. Das ist eine saubere Arbeitsteilung, und die beiden greifen ineinander: Zitate werden im Vokabular `manni:citations` geschrieben, `manni meta validate` prüft, ob ein Eintrag wohlgeformt ist, und nur `manni cite check` kann sagen, ob er noch gilt.

## Was der Einsatz kostet

Drei Einwände, bevor jemand das in eine produktive Pipeline hebt.

**Reifegrad.** Die Familie ist neu. Das erste Release des Pakets liegt im September 2026, das [Repository](https://github.com/hawkeyexl/manni) hat einen einzigen Maintainer und noch praktisch keine externe Nutzerbasis. Nur `meta` bringt als früheres `docmeta` echte Historie mit; `cite`, `a11y` und `key` sind wenige Wochen alt. Die Entwicklung ist erkennbar stark agentengestützt, mit einem Proposal-Dokument pro größerer Änderung. Das ist ungewöhnlich gut nachvollziehbar dokumentiert, ersetzt aber keine Betriebserfahrung. Version pinnen, nicht gegen `latest` bauen.

**Pflegeaufwand.** Ein Zitat ist eine Zusage. Jede gepinnte Aussage will bei jedem Refactoring bestätigt oder neu gesetzt werden. Wer 800 Sätze pinnt, hat 800 potenzielle Befunde. Das skaliert nur mit Auswahl: Grenzwerte, Standardwerte, Fehlercodes, Versionsangaben, Endpunktnamen. Konzeptuelle Prosa gehört nicht dazu.

**Voraussetzungen.** Die Prüfung braucht Zugriff auf die Git-Historie des Quell-Repos, in CI also einen Checkout mit voller Historie ([CI-Hinweise](https://hawkeyexl.github.io/manni/cite/ci/)). Getrennte Repositorys für Code und Doku sind vorgesehen, kosten aber Einrichtung. Node 24 ist eine zusätzliche Hürde für ältere Build-Images.

## Ein realistischer Einstieg

Nicht mit `cite` anfangen, sondern mit `meta`. Es läuft ohne Konfiguration gegen ein eingebautes Schema, das nur `type` verlangt, und liefert in einer Minute ein Bild vom Zustand des Docsets ([Get started](https://hawkeyexl.github.io/manni/meta/get-started/)).

Für `cite` würde ich einen Piloten wählen, der beantwortbar ist: die zehn bis zwanzig Aussagen, die in den letzten zwei Jahren am häufigsten falsch waren. Meist sind das Zahlen. Wenn nach einem Quartal die Zahl der echten Befunde die Zahl der reinen Pin-Reparaturen übersteigt, hat sich der Aufwand gelohnt. Wenn nicht, löscht man zwanzig Einträge und hat nichts verloren.

Der eigentliche Gewinn ist ohnehin kein Werkzeug, sondern eine Verschiebung der Beweislast. Bisher muss jemand bemerken, dass die Doku falsch ist. Danach muss die Doku zeigen, dass sie noch stimmt.

---

## Quellen

Alle URLs zuletzt geprüft am 14. September 2026. Geprüfte Paketversion: siehe `npm view @hawkeyexl/manni version`.

**Werkzeug und Dokumentation**

- manni, Projektdokumentation: <https://hawkeyexl.github.io/manni/>
- manni, Quellcode und README (MIT-Lizenz): <https://github.com/hawkeyexl/manni>
- `@hawkeyexl/manni` auf npm: <https://www.npmjs.com/package/@hawkeyexl/manni>
- Doc Detective, Dokumentation: <https://docs.doc-detective.com/>
- Vale, Dokumentation: <https://vale.sh/>
- axe-core, Deque Labs: <https://github.com/dequelabs/axe-core>

**Standards, Normen und Vokabulare**

- JSON Schema: <https://json-schema.org/>
- W3C: *Web Content Accessibility Guidelines (WCAG) 2.1*, W3C Recommendation, 2018: <https://www.w3.org/TR/WCAG21/>
- Richtlinie (EU) 2019/882 (European Accessibility Act): <https://eur-lex.europa.eu/eli/dir/2019/882/oj>
- IHK Köln: *Barrierefreie Websites, Dienstleistungen und Produkte* (Überblick zu BFSG und EN 301 549): <https://www.ihk.de/koeln/hauptnavigation/recht-steuern/barrierefreiheit-von-webseiten-dienstleistungen-und-produkten-5921172>
- OASIS: *Darwin Information Typing Architecture (DITA) Version 1.3*: <https://www.oasis-open.org/standard/dita-v1-3/>
- Procida, Daniele: *Diátaxis. A systematic framework for technical documentation authoring*: <https://diataxis.fr/>
- The Good Docs Project, Templates: <https://www.thegooddocsproject.dev/template>
- Torre, Fabrizio Ferri: *The Seven-Action Model*, passo.uno: <https://passo.uno/seven-action-model/>
- Google Cloud Platform: *Open Knowledge Format (OKF) Specification*: <https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md>
- *Command Line Interface Guidelines*: <https://clig.dev/>

**Im Text belegt**

- Silva, Manny (2025): *Docs as Tests. A Strategy for Resilient Technical Documentation*. Boffin Education. ISBN 978-0-9941693-6-5.
- Wen, Fengcai; Nagy, Csaba; Bavota, Gabriele; Lanza, Michele (2019): *A Large-Scale Empirical Study on Code-Comment Inconsistencies*. In: Proceedings of the 27th International Conference on Program Comprehension (ICPC '19), S. 53–64. DOI: [10.1109/ICPC.2019.00019](https://doi.org/10.1109/ICPC.2019.00019). Volltext: <https://www.inf.usi.ch/lanza/Downloads/Wen2019a.pdf>
- Deque Systems (2021): *The Automated Accessibility Coverage Report*: <https://www.deque.com/automated-accessibility-coverage-report/>
- Google Chrome for Developers: *Lighthouse accessibility score*: <https://developer.chrome.com/docs/lighthouse/accessibility/scoring>

**Weiterführend**

- Tan, Lin; Yuan, Ding; Krishna, Gopal; Zhou, Yuanyuan (2007): *iComment: Bugs or Bad Comments?* In: Proceedings of the 21st ACM Symposium on Operating Systems Principles (SOSP '07), S. 145–158. DOI: [10.1145/1294261.1294276](https://doi.org/10.1145/1294261.1294276)
- Fluri, Beat; Würsch, Michael; Giger, Emanuel; Gall, Harald C. (2009): *Analyzing the co-evolution of comments and source code*. In: Software Quality Journal 17 (4), S. 367–394. DOI: [10.1007/s11219-009-9075-x](https://doi.org/10.1007/s11219-009-9075-x)

**Include-Mechanismen als Alternative**

- Asciidoctor: *Include Tagged Regions*: <https://docs.asciidoctor.org/asciidoc/latest/directives/include-tagged-regions/>
- Sphinx: *literalinclude-Direktive*: <https://www.sphinx-doc.org/en/master/usage/restructuredtext/directives.html#directive-literalinclude>
- PyMdown Extensions: *Snippets*: <https://facelessuser.github.io/pymdown-extensions/extensions/snippets/>
