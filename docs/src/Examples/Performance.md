# Making Buttons: A look at Performance

This tutorial's web application comes with the following
ingredients: A validated text input where users are
supposed to enter a positive natural number.
When they hit `Enter` after entering their
number, a corresponding number of buttons is created, each
of which can be clicked exactly once before being disabled,
and the sum of the values clicked will be accumulated and
displayed at the UI. The sum should be reset when a new set
of buttons is created and the input field should be cleared
when the run button is clicked.

Since we want to have a look at the performance of this,
we also include an output field for the time it took to
create and display the buttons.

We are going to iterate over large lists of
items, therefore we need to be conscious about stack space and make
use of the tail-recursive functions.

Here's the list of imports:

```idris
module Examples.Performance

import Data.Either
import Data.List.TR
import Data.Nat
import Data.Refined.Integer
import Data.String

import Derive.Prelude
import Derive.Refined

import Examples.CSS.Performance
import Examples.Util

import Web.MVC
import Web.MVC.Animate

%default total
%language ElabReflection
```

## Model

As before, we first define our event type. We use a refinement type
from the [idris2-refined](https://github.com/stefan-hoeck/idris2-refined)
library for the number of buttons:


```idris

MinBtns, MaxBtns : Integer
MinBtns  = 1
MaxBtns  = 100_000

record NumBtns where
  constructor B
  value : Integer
  {auto 0 prf : FromTo MinBtns MaxBtns value}

%runElab derive "NumBtns" [Show,Eq,Ord,RefinedInteger]

public export
data PerfEv : Type where
  PerfInit   : PerfEv
  NumChanged : Either String NumBtns -> PerfEv
  Reload     : PerfEv
  Set        : Nat -> PerfEv
```

We also require a function for input validation:

```idris
read : String -> Either String NumBtns
read =
  let err := "expected integer between \{show MinBtns} and \{show MaxBtns}"
   in maybeToEither err . refineNumBtns . cast
```

The application state consists of the currently validated input
plus the current sum.

```idris
public export
record PerfST where
  constructor P
  sum : Nat
  num : Maybe NumBtns

export
init : PerfST
init = P 0 Nothing
```

## View

The CSS rules and reference IDs have again been moved
to their [own module](CSS/Performance.idr), to declutter
the code here. We also use labeled lines of input elements
as in the [previous example](Reset.idr). For the
grid of buttons, we need a reference for each button,
since we want to disable them after they have been clicked:

```idris
btnRef : Nat -> Ref Tag.Button
btnRef n = Id "BTN\{show n}"

btn : Nat -> Node PerfEv
btn n =
  button
    [Id (btnRef n), onClick (Set n), classes [widget,btn,inc]]
    [Text $ show n]
```

Next, we write the function to create a grid of buttons.
Since we plan to create thousands of buttons at once, we must
make sure to do this in a stack-safe manner.
Some list functions in the standard libraries are not (yet)
stack safe, so we use stack-safe `iterateTR`
from the [idris2-tailrec](https://github.com/stefan-hoeck/idris2-tailre)
project.
Luckily, in recent commits of the Idris project,
`map` for `List` *is* stack-safe, so we can use it without further
modification.

```idris
btns : NumBtns -> Node PerfEv
btns (B n) = div [class grid] $ map btn (iterateTR (cast n) (+1) 1)
```

And, finally, the overall layout of the application:

```idris
content : Node PerfEv
content =
  div [ class performanceContent ]
    [ lbl "Number of buttons:" numButtonsLbl
    , input [ Id natIn
            , onInput (NumChanged . read)
            , onEnterDown Reload
            , classes [widget, textIn]
            , placeholder "Enter a positive integer"
            ] []
    , button [Id btnRun, onClick Reload, classes [widget, btn]] ["Run"]
    , lbl "Sum:" sumLbl
    , div [Id out] []
    , div [Id time] []
    , div [Id buttons] []
    ]
```

We register two events at the text field: Whenever users enter
some text, the field should fire an event to get the validation
routine started. If the *Enter* key is pressed, the grid of
buttons should be generated. This should also happen if the
*Run* button is clicked.

## Controller

As before, we define several pure functions for updating
the state and the DOM depending on the current event.

```idris
dispTime : NumBtns -> Integer -> String
dispTime 1 ms = "\Loaded one button in \{show ms} ms."
dispTime n ms = "\Loaded \{show n.value} buttons in \{show ms} ms."
```

Adjusting the application state is very simple:

```idris
adjST : PerfEv -> PerfST -> PerfST
adjST PerfInit       = const init
adjST (NumChanged e) = {num := eitherToMaybe e}
adjST Reload         = {sum := 0}
adjST (Set k)        = {sum $= (+k)}
```

Updating the DOM is not much harder. Here it is very useful that we
do not use a virtual DOM: Since we don't recreate the whole view on
every event, we don't have to keep track of the disabled buttons,
nor do we have redraw thousands of buttons, which would drastically
slow down the user interface.

```idris
displayST : PerfST -> List (DOMUpdate PerfEv)
displayST s = [disabledM btnRun s.num, show out s.sum]

displayEv : PerfEv -> PerfST -> DOMUpdate PerfEv
displayEv PerfInit       _ = child exampleDiv content
displayEv (NumChanged e) _ = validate natIn e
displayEv (Set k)        _ = disabled (btnRef k) True
displayEv Reload         s = maybe noAction (child buttons . btns) s.num

display : PerfEv -> PerfST -> List (DOMUpdate PerfEv)
display e s = displayEv e s :: displayST s
```

The main controller is slightly more involved because we want to
record the time taken to create the buttons to get a feeling for
the performance we can achieve. Function `Web.MVC.Animate.timed` is used
for this: It calculates the time difference (in milliseconds) spent
within an `IO` action.

However, we update the displayed time only after a set of new buttons
was created successfully:

```idris
export
runPerf : Handler PerfEv => Controller PerfST PerfEv
runPerf e s = do
  (s2,dt) <- timed (runDOM adjST display e s)
  case (e,s2.num) of
    (Reload,Just n) => updateDOM {e = PerfEv} [text time $ dispTime n dt]
    _               => pure ()
  pure s2
```

<!-- vi: filetype=idris2:syntax=markdown
-->