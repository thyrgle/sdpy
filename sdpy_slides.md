---
title: Formulaic Programming with `pypagate`
---

# Welcome!

- My name is Christopher Sumnicht.
- I have some *interesting* programming ideas (and make a lot of small game libraries!).
- Today, I want to talk about how cool spreadhseets are and how spreadsheets can help Python.
- (Surpsingly, this is not a talk about Pandas or Polars.)

---

# A Spreadsheet

|  X  |     A           |
| :-: | :-------------: |
|  1  |    80           |
|  2  |    85           |
|  3  |    95           |
|  4  |   100           |
|  5  | `MEAN($A)` = 90 |

---

# A Spreadsheet (part 2)

|  X  |     A           |
| :-: | :-------------: |
|  1  |    60           |
|  2  |    85           |
|  3  |    95           |
|  4  |   100           |
|  5  | `MEAN($A)` = 85 |

---


# A Python Program

```py
>>> from statistics import mean
>>> scores = [80, 85, 95, 100]
>>> m = mean(scores)
>>> m
90
```

- What happens if we change it?

```py
>>> scores[0] = 60
>>> m
90
```

- Need to recalculate!

---

# Assignment

- The fundamental problem is assignment *is not* equality in the algebraic sense.
- But what if it were?

```py
>>> from pypagate import Term
>>> x = Term(5)
>>> y = x + 1
>>> print(x.unwrap())
5
>>> y.unwrap()
6
>>> x += 5
>>> x.unwrap()
10
>>> y.unwrap()
11
```

---

## A Slightly More Complicated Example

```py
>>> from dataclasses import dataclass
>>> from pypagate import Term
>>> @dataclass
... class Point:
...     x: Term
...     y: Term
...
>>> @dataclass
>>> class Circle:
        x: Term
        y: Term
        r: Term
...
>>> p = Point(x=Term(0), y=Term(0))
>>> e = Circle(x=Term(10), y=Term(10), r=Term(5))
>>> @fire_on(((p.x ** 2 - e.x) ** 2 + (p.y ** 2 - e.y) ** 2) ** 0.5 <= e.r)
... def collision():
...     print("Collision happened!")
...

>>> p.x.change(7) # Note: Must use change otherwise ref is changed to new obj
>>> p.y.change(7) # Note: Must use change otherwise ref is changed to new obj
Collision happened!
```

---

# Event Listeners

- Event listeners are routines that are called once a specified type or kind of event occurs.
- Popular examples from Javascript:
```js
aButton.addEventListener("click", (e) => {
  console.log("Clicked a button!");
});
```
- Here is what it *might* look like in Python:
```py
@on_click(a_button)
def clicked(e):
    print("Clicked a button!");
```

- Uses a decorator to indicate the function should be fired when the click happens.

---

# Game loops

```py
dt = 0 # Change in time per iteration of loop.

while True:
    get_input(dt)
    update(dt)
    draw(dt)
    dt = clock.tick(60) / 1000 # Cap at 60 FPS, but be sure to get the dt.
```

- Fairly simple
- Challenges in organization

---

## A Closer look at `update(dt)`

```py
def update(dt: float):
    # Just worry about main game scene!
    # Player can get hurt.
    handle_collision()
    if player.health == 0:
        game_over = True
    # Other logic.
```
- Could result in a long chain of if statements if there are lots of different events!
- I would also do not want to check the if statement each `update` call.
- Might actually be inconvenient to *centralize* logic.
- How can we decentralize?

---

## Enter `pypagate` Again!

```py
>>> from pypagate import Term, fire_on
>>> class Player:
...     def __init__(self, health):
...         self.health = Term(health)
...
>>> player = Player(3)
>>> @fire_on(player.health == 0)
>>> def game_over():
...     print("Game over")
>>> player.health -= 1
>>> player.health -= 1
>>> player.health -= 1
Game over
```
---

## Revisiting ``update(dt)``

```py
@fire_on(player.health == 0)
def game_over():
    # Switch to game over scene or just:
    exit()

def update(dt: float):
    handle_collision()
```
- Decentralized the core logic.

---
## But the World Isn't Built Like This

- Basically all pre-existing libraries do not use `Term` objects and `Formula` objects.
- How do we survive a world that is not like a spreadsheet?
- `SourceMap` objects!

---
## `SourceMap`: A Simple Example

```py
>>> from pypagate.source import SourceMap, exec_always
>>> source = SourceMap({})
>>> x = 0
>>> @exec_always(source)
... def inc():
...     global x
...     x += 1
...
>>> for _ in range(3):
...     print(x)
...     source.listen({})
...
0
1
2
```
---

## What about the `{}`?

- `SourceMap` is *almost* a dictionary.
- Constructor takes dictionary of attributes and initial values.
- Values are converted to Terms.
- `.listen` updates the values of the `Term` attributes for a particular source.

---

### What about the `{}`? (Example)

```py
>>> from pypagate.source import SourceMap, exec_while
>>> source = SourceMap({"x": 0})
>>> @exec_while(source.x < 2, source)
... def is_lt_two():
...     print("x < 2")
...
>>> @exec_while(source.x >= 2, source)
... def is_gte_two():
...     print("x >= 2")
...
>>> for _ in range(4):
...     source.listen({"x": source.x.unwrap() + 1})
...
x < 2
x >= 2
x >= 2
x >= 2
```

---

## "The" Pygame Example

```py
import pygame

pygame.init()
screen = pygame.display.set_mode((1280, 720))
clock = pygame.time.Clock()
running = True
while running:
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
    screen.fill("purple")
    pygame.display.flip()
    clock.tick(60)
pygame.quit()
```
---

## What Can ``SourceMap`` do for us?

- We can flatten:

```py
for event in pygame.event.get():
    if event.type == pygame.QUIT:
        running = False
```

---

## What Can ``SourceMap`` do for us? (part 2)

```py
...
source = SourceMap({
    "quit_event": False
})
running = Term(True)
...
while running.unwrap():
    source.listen({
        "quit_event": pygame.event.peek(eventtype=pygame.QUIT)
    })
    ...
```
---

## What Can ``SourceMap`` do for us? (part 3)

```py
@fire_on(source.quit_event) # i.e. fire when the quit event becomes True.
def quit_game():
    running.change(False)
```
---

# Conclusion

- `pypagate` is a library for creating event listeners that operate based on terms and formulas.
- Really new library!
- A lot of room to convert existing libraries to have extensive "formulaic listeners."
- There are more "event decorators"!
- We still have not touched on `Universe` and `Variable` objects for managing *collections* of `Term` and `Formula` objects (decentralize `handle_collision()`?)!
---

# Thanks!
