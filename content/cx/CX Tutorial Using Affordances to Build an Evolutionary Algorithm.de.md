+++
title = "CX Tutorial: Using Affordances to Build a Small Text-based Adventure"
tags = [
    "CX",
    "CX Tutorials",
    "Affordances"
]
bounty = 5
date = "2017-09-20"
categories = [
    "Tutorials",
]
+++

<!-- MarkdownTOC autolink="true" bracket="round" depth="2" -->

- [Einführung](#Einführung)
- [Aufforderung-Antwort-Verfahren](#Aufforderung-Antwort-Verfahren)
- [Angebotssystem](#Angebotssystem)
- [Objekte](#Objekte)
- [Fazit](#Fazit)

<!-- /MarkdownTOC -->

# Einführung

Dieses Tutorial zeigt ein textbasiertes "Spiel" (der Benutzer interagiert nicht mit dem Programm und 
kann die Entscheidungen der Spielfigur nicht beeinflussen), das ein
[Aufforderung-Antwort-Verfahren](#Aufforderung-Antwort-Verfahren) verwendet, um zu bestimmen, 
welche möglichen Aktionen die Spielfigur ausführen kann. Der vollständige Quellcode ist im
[CX's repository](https://github.com/skycoin/cx), in der Datei *examples/text-based-adventure.cx* zu finden.

Das Spiel beschreibt das Abenteuer eines Reisenden, der vor einem Monster flieht (nächsten Monat ist Halloween). 
Wenn der Reisende eine bestimmte Anzahl von Stunden überlebt (nun, das sind nichts weiter 
als Iterationen in einer *for* Schleife), hört das Monster auf, den Reisenden zu jagen. 
Unten ein Beispiel für eine Sitzung:


```
The traveler keeps following the lane, making sure to ignore any pain.
Howling and growling, the monster is coming.
Bravery comes into sight, in the hope of living for another night.
Naive, and even dumb, but the traveler's act leaves the monster numb.
North, east, west, south. Any direction is good,
as long as no monster can be found.
Howling and growling, the monster is coming.
The traveler runs away, and cowardice lets him live for another day.

You survived.
```

Wenn der Reisende sich entscheidet, das Monster zu bekämpfen und sein heroischer Versuch
scheitert, endet das Spiel. Ein Beispiel für ein Spielende ist:

```
North, east, west, south. Any direction is good,
as long as no monster can be found.
Howling and growling, the monster is coming.
Bravery comes into sight, in the hope of living for another night.
But failure describes this fend and, suddenly, this adventure comes to an end.

You died.

Call's State:
flag:			true
nonAssign_32:		""

halt() Arguments:
0: "You died."

65: call to halt
```

Wie Sie sehen können, wird ein Fehler ausgelöst, wenn Sie sterben (das ist gut so, 
da es für den Programmierer eine beängstigende Situation ist).


# Aufforderung-Antwort-Verfahren

In diesem Verfahren wird eine Frage gestellt und verschiedene Agenten (in diesem Fall 
Funktionen) müssen auf diese Frage antworten. Eine einfache Frage, die gestellt werden 
kann, lautet: "Wer kann in diesem Moment ausgeführt werden?" und die Funktionen, die 
ausgeführt werden dürfen, tun dies.

Die folgenden Funktionsprototypen stellen die möglichen Aktionen dar, die während des 
Abenteuers des Reisenden auftauchen können.


```
func walk (flag bool) () {}
func noise (flag bool) () {}
func consider (flag bool) () {}
func chance (flag bool) () {}
func fightResult (flag bool) () {}
func theEnd (flag bool) () {}
```

# Angebotssystem

Eine andere Funktion muss die Funktionsaufrufe koordinieren. In diesem 
Fall wird das Angebotssystem von CX verwendet, um zu bestimmen, ob eine 
Aktion ausgeführt werden darf oder nicht.

```
yes := true
no := false

remArg("walk")
affExpr("walk", "yes|no", 0)
:tag walk;
walk(false)
```

Im obigen Code sucht *remArg()* nach dem Ausdruck mit der "walk"-Markierung
und entfernt deren Argument. Dies geschieht, damit das Angebotssystem die 
Argumente auflistet, die an den Operator des Ausdrucks gesendet werden können. 
Als nächstes sagt *affExpr()* CX: "Unter allen Argumenten die gesendet werden 
können, um zu ´gehen´ (*walk*), sag mir ob ´ja´ (*yes*) oder ´nein´ (*no*) als 
Argumente benutzt werden können, und wende die nullte (*0th*) Option der von dir 
zurückgegebenen Angebotsliste an.

Das vorherige Verfahren wird auf alle Aktionen angewendet, die während des Abenteuers 
des Reisenden passieren können. Für jede dieser Aktionen werden die folgenden Regeln 
abgefragt, um zu bestimmen, ob die Aktion erlaubt ist oder nicht:


```
setClauses("
          aff(walk, yes, X, R) :- X = monster, R = false.
          aff(noise, yes, X, R) :- X = monster, R = false.

          aff(consider, yes, X, R) :- R = false.
          aff(chance, yes, X, R) :- R = false.
          aff(fightResult, yes, X, R) :- R = false.
          aff(theEnd, yes, X, R) :- R = false.

          aff(consider, yes, X, R) :- X = monster, R = true.
          aff(chance, yes, X, R) :- X = fight, R = true.
          aff(fightResult, yes, X, R) :- X = fight, R = true.
          aff(theEnd, yes, X, R) :- X = died, R = true.
        ")
```

Die erste Regel kann wie folgt gelesen werden: "Ich werde abgefragt, 
wenn du in Betracht ziehst das Ja-Argument an die Geh-Aktion zu senden. 
Wenn das aktuelle Objekt *monster* ist, dann ist dieses Argument keine Option."

Die Regeln im zweiten Block (die 4 Regeln nach der ersten leeren Zeile) sagen 
dem Angebotssystem, "niemals" ein "Ja"-Argument zu akzeptieren. Wir tun das, weil 
wir wollen, dass dies das Standardverhalten ist, aber wir können später Regeln 
deklarieren, die dieses Verhalten überschreiben. Dieser Überschreibungsprozess 
erfolgt mit den letzten 4 Regeln. Grundsätzlich sagt dieser Regelblock CX, dass er *ja* 
als Argumente akzeptiert, wenn ein bestimmtes Objekt im Objektstapel vorhanden ist.


# Objekte

Einige der Aktionen fügen Objekte aus dem Objektstapel hinzu oder entfernen sie. 
Zum Beispiel, wenn die Rausch-Aktion *noise* entscheidet, das Monster erscheinen 
zu lassen, wird *addObject("monster")* ausgeführt. Wenn der Reisende sich entscheidet, 
vor dem Kampf wegzulaufen, wird das "monster"-Objekt vom Stapel entfernt.

Im Falle der "zufälligen" Aktion *chance* kann das Monster entscheiden, dem Reisenden 
noch ein paar weitere Sekunden zu geben, um zu sehen, was er als nächstes tun wird. 
Um dies zu tun, wird das "fight"-Objekt entfernt (weil das Monster den Kampf noch nicht 
beginnen möchte), aber das "monster"-Objekt bleibt.


# Fazit

CX's Angebotssystem benutzt Objekte und Regeln, um komplexe Entscheidungen
zu treffen, wie Angebote gefiltert werden.

Benutzen wir Objekte, können wir entscheiden, welche Aktionen aktiviert 
oder deaktiviert werden. In diesem Beispiel wird für den Aktivierungsprozess
nur eine kleine Anzahl von Aktionen herangezogen und der Vorteil dieses 
Verfahrens mag auf den ersten Blick wertlos erscheinen. Aber dessen ungeachtet: 
Komplexere Regeln, die wesentlich mehr Objekte beinhalten, können definiert 
werden und eine einzige davon könnte dafür verantwortlich sein, eine ganze Anzahl 
von Knoten in einem grossen Aktionsnetzwerk zu aktivieren. Ausserdem sind in diesem 
Beispiel nur zwei mögliche Argumente *ja* und *nein* beteiligt, dabei könnten wir 
viel mehr Argumente verwenden und Aktionen, die andere als boolsche Arten von Argumenten
akzeptieren.

