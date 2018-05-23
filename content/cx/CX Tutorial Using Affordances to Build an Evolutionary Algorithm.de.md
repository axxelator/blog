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

# Affordance System

Another function must coordinate the function calls. In this case,
CX's affordance system is used to determine if an action is allowed to
run or not.

```
yes := true
no := false

remArg("walk")
affExpr("walk", "yes|no", 0)
:tag walk;
walk(false)
```

In the code above, *remArg()* looks for an expression with the "walk"
tag and removes its argument. This is done in order to make the
affordance system list the arguments that can be sent to the
expression's operator. Next, *affExpr()* is telling CX "among all the
arguments that can be sent to *walk*, tell me if *yes* or no *no* can
be used as arguments, and apply the *0th* option from the affordance
list that you return."

The previous procedure is applied to all the actions that can happen
during the traveler's adventure. For each of these actions, the
following rules are queried to determine if the action should be
allowed or not:

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

The first rule can be read as "I will be queried if you're considering
to send the *yes* argument to the *walk* action. If the object
*monster* is present, then this argument is *not* an option."

The rules in the second block (the 4 rules after the first empty line)
tell the affordance system to "never" accept a *yes* argument. We do
this because we want this to be the default behaviour, but we can
later declare rules that override this behaviour. This override
process happens with the last 4 rules. Basically, this block of rules
is telling CX to accept *yes* as arguments if a particular object is
present in the object stack.

# Objects

Some of the actions add or remove objects from the object stack. For
example, whenever the *noise* action decides to make the monster
appear, *addObject("monster")* is executed. If the traveler decides to
run away from the fight, the "monster" object is removed from the
stack.

In the case of the *chance* action, the monster can decide to spare
the traveler a few more seconds to see what he will decide to do
next. To do this, the "fight" object is removed (as the monster does
not want to start a fight yet), but the "monster" object remains.

# Conclusion

CX's affordance system uses objects and rules to make complex
decisions about how affordances are going to be filtered.

By using objects, we can decide what actions will be activated or
deactivated. For this example, a small amount of actions are being
considered for this activation process, and the benefit of using this
architecture could seem worthless at first sight. Nevertheless, more
complex rules involving more objects could be defined, and a single
rule could be in charge of activating several nodes in a big network
of actions. Also, in this example only two possible arguments are
considered: *yes* and *no*; we could have more arguments, and actions
that accept different types of arguments other than booleans.
