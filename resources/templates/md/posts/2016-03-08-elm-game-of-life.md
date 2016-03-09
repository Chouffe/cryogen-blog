{:title "Elm - Conway's Game of Life - Functional Programming in the browser"
 :layout :post
 :tags  ["Functional Programming" "Elm"]
 :toc true}

This is the second post of my series describing functional programming abstraction in the browser. Elm is a relatively new programming language. Elm code compiles to JS, HTML and CSS. It embraces Reactive Programming in a Functional Programming mindset.

## Elm Language

To be honest, I am pretty excited about Elm! It looks very promising. Elm's syntax is very close to Haskell's syntax. So if you happen to know Haskell you will start writing Elm code in no time.

Here is a list of core features that makes it interesting imo:
* Purely Functional
  * Higher order functions
  * Currying
  * Laziness
  * Immutable Data Structures
* Small core language
  * The Elm core packages are very simple and small. A couple of hours only is needed to dive into it. Not as small as a lisp though but still very small.
  * The Elm source code is amazingly well writen (in Haskell). It is available on Github.
* Expressive language
* Amazing Type System
  * Safety: No runtime exceptions
  * Algebraic Data Types
  * Parametricity
  * Hindley Milner
* Reactive Programming

## Elm Architecture

The Elm Architecture is a simple yet sophisticated pattern for writing infinitely nestable components. This is something anyone would like to have for modularity, code reuse and testing.

The architecture of an Elm program is the composition of 3 separated parts:
* **Model**: Data Structure (usually a combination of algebraic data types) that describes the state of the program
* **Update**: Reducer functions from a model to a new model
* **View**: pure function of the model + event handlers

I tend to see it as a proper implementation of the MVC pattern that is commonly found in other frameworks. The business logic and data are completely separated from the view code.

## Conway's Game of Life

I decided to code a little Elm program for Conway's Game of Life to get a better understanding of Elm. My code is available on [github](https://github.com/Chouffe/elm-game-of-life) as always.

Let's first consider how it works
* There is a board containing alive and dead cells
* A particular cell on the board will die or live according to Conway's rules
* Time is perceived as a sequence of transitions between board states

Here are all the actions I wanted to enable:
* Changing a cell from dead to alive by clicking on the board
* Simulating a time change by clicking a *Next* button
* Loading predefined boards to quickly visualize some interesting shapes in Conway's Game of Life

Disclaimer: I am an Elm beginner (This is actually my first Elm program). Please forgive my bad coding style if you are more experienced. I would love to know the proper ways to achieve some data transformation. Feel free to reach out.

### Modeling State

I model the game as follows:
* Cell: Dead/Alive are modeled with booleans
* Board: 2 dimensional array of Cells
* Position: a position on the board is a hashmap with the keys *x* and *y*. I could be using a tuple for that but I wanted to play with the hashmap datastructure.

```
type alias Position = { x : Int, y : Int }

type alias Model = A.Array (A.Array Bool)
```

### Manipulating State
Maybe I am reinventing the wheel but I need some helper functions to manipulate my states.

#### Board
I need to be able to tell the width and height of a board.

```
height : Model -> Int
height model = A.length model

width : Model -> Int
width model =
    case A.get 0 model of
        Nothing -> 0
        Just row -> A.length row
```
I also need to update a Cell value on the board given a Position.

```
getIn : A.Array (A.Array a) -> Position -> Maybe a
getIn model position =
    case A.get position.x model of
        Nothing -> Nothing
        Just row -> A.get position.y row

assocIn : A.Array (A.Array a) -> Position -> a -> A.Array (A.Array a)
assocIn model position x =
    case A.get position.x model of
        Nothing -> model
        Just row ->
            let updatedRow = A.set position.y x row
            in
                A.set position.x updatedRow model

updateIn : A.Array (A.Array a) -> Position -> (a -> a) -> A.Array (A.Array a)
updateIn model position f =
    case getIn model position of
        Nothing -> model
        Just b -> assocIn model position (f b)
```

I found the Array core package lacking some important functions. I reimplemented some functions to manipulate 2d arrays (Clojure like functions).

#### Positions
I need a constructor for creating Position values
```
position : Int -> Int -> Position
position i j = { x = i, y = j }

positions : Model -> List Position
positions model =
    let w = width model
        h = height model
    in List.concatMap (\i -> (List.map (position i) (range 0 w))) (range 0 h)

```
I wish Elm had list comprehensions or at least monadic operations to operate on Lists, Arrays and Maybe values.
Here I had to basically reimplement the Haskell >>= for lists. I am sure there is a rationale for not having them in Elm. I have not figured it out yet though ;)

```
List.concatMap (\i -> (List.map (position i) (range 0 w))) (range 0 h)
```

### Update

The update part of the Elm program is usually really short and can be seen as a FSM (Finite State Machine) with Actions and Models.
The Action type is an union type that describes the possible actions that can be performed on the model.
The `update` function is a reducer function from actions to models. This pattern can be found in re-frame (ClojureScript) and React as well.

```
type Action = Next
            | LoadBoard Model
            | ToggleValue Position

update : Action -> Model -> Model
update action model =
    case action of
        Next -> next model
        LoadBoard model -> model
        ToggleValue pos -> updateIn model pos not
```

#### Business Logic (Update Logic)

All the logic is composition of pure functions and is located in one place. Here is the business logic needed to transition from one board state to the next.

```
isValid : Model -> Position -> Bool
isValid model position =
    let w = width model
        h = height model
        x = position.x
        y = position.y
    in
        if x >= h then False
        else if y >= w then False
        else if x < 0 then False
        else if y < 0 then False
        else True

neighbors : Model -> Position -> List Position
neighbors model pos =
    let w = width model
        h = height model
        x = pos.x
        y = pos.y
    in
        [ (x + 1, y), (x + 1, y + 1), (x + 1, y - 1)
        , (x , y + 1), (x, y - 1)
        , (x - 1, y), (x - 1, y + 1), (x - 1, y - 1)
        ]
            |> List.map (\(i, j) -> position i j)
            |> List.filter (isValid model)

isDead : Model -> Position -> Bool
isDead model position =
    case getIn model position of
        Nothing -> False
        Just b -> not b

isAlive : Model -> Position -> Bool
isAlive model = not << isDead model

rules : Bool -> Int -> Int -> Bool
rules self deadCellCount aliveCellCount =
    if self && aliveCellCount < 2 then False
    else if self && aliveCellCount >= 2 && aliveCellCount <= 3 then True
    else if not self && aliveCellCount == 3 then True
    else False

next : Model -> Model
next model =
    positions model
        |> List.foldl (\position m -> assocIn m position (nextCellState model position)) model

nextCellState : Model -> Position -> Bool
nextCellState model position =
    let neighbs = neighbors model position
        deadCellCount = List.length (List.filter (isDead model) neighbs)
        aliveCellCount = List.length (List.filter (isAlive model) neighbs)
    in case (getIn model position) of
        Nothing -> False
        Just self -> rules self deadCellCount aliveCellCount
```

### View
The last missing piece of our Elm program is the view. Which is a pure function of the model.

```
viewCell : Signal.Address Action -> Position -> Int -> Bool -> Html
viewCell address position width b =
    div [ viewCellStyle width b
        , onClick address (ToggleValue position)
        , class ("cell-item" ++ " " ++ (if b then "cell-alive" else "cell-dead"))
        ] []

selectBoardView : Signal.Address Action -> List (String, Model) -> Html
selectBoardView address models =
    div [ class "btn-group pull-right" ]
        (List.map (\(s, m) -> div [ class "btn btn-default"
                                  , onClick address (LoadBoard m) ]
                                  [ text s ]) models)

view : Signal.Address Action -> Model -> Html
view address model =
    let w = width model
    in
        div [class "game-container" ]
            [ css "game.css"
            , css "//maxcdn.bootstrapcdn.com/bootstrap/3.3.0/css/bootstrap.min.css"
            , div [ class "button-container" ]
                  [div [ class "btn btn-default"
                       , onClick address Next]
                        [text "Next"]
                  , selectBoardView address boards
                ]
            , div [ class "board-container" ]
                  (List.indexedMap (\k b -> viewCell address (intToPosition k w) w b) (A.toList (concat model)))
            ]

viewCellStyle : Int -> Bool -> Attribute
viewCellStyle width b =
    style [ ("flex-basis", toString (100.0 / (toFloat width)) ++ "%") ]
```

All the view functions are purely declarative. We never need to take care of DOM mutations because Elm takes care of that for us.

### Entrypoint
The main entrypoint of our program is the following `main` function:

```
main =
  start
    { model = pulsar
    , update = update
    , view = view
    }

pulsar = A.fromList [ A.fromList [ False, False, False, False, False, False, False, False, False, False, False, False, False, False, False ]
                    , A.fromList [ False, False, False, True,  True,  True,  False, False, False, True,  True,  True,  False, False, False ]
                    , A.fromList [ False, False, False, False, False, False, False, False, False, False, False, False, False, False, False ]
                    , A.fromList [ False, True,  False, False, False, False, True,  False, True,  False, False, False, False, True,  False ]
                    , A.fromList [ False, True,  False, False, False, False, True,  False, True,  False, False, False, False, True,  False ]
                    , A.fromList [ False, True,  False, False, False, False, True,  False, True,  False, False, False, False, True,  False ]
                    , A.fromList [ False, False, False, True,  True,  True,  False, False, False, True,  True,  True,  False, False, False ]
                    , A.fromList [ False, False, False, False, False, False, False, False, False, False, False, False, False, False, False ]
                    , A.fromList [ False, False, False, True,  True,  True,  False, False, False, True,  True,  True,  False, False, False ]
                    , A.fromList [ False, True,  False, False, False, False, True,  False, True,  False, False, False, False, True,  False ]
                    , A.fromList [ False, True,  False, False, False, False, True,  False, True,  False, False, False, False, True,  False ]
                    , A.fromList [ False, True,  False, False, False, False, True,  False, True,  False, False, False, False, True,  False ]
                    , A.fromList [ False, False, False, False, False, False, False, False, False, False, False, False, False, False, False ]
                    , A.fromList [ False, False, False, True,  True,  True,  False, False, False, True,  True,  True,  False, False, False ]
                    , A.fromList [ False, False, False, False, False, False, False, False, False, False, False, False, False, False, False ]
                    ]
```

## Result

![Conway's Game of Life in Elm](/posts/img/gameoflife/gameoflife.png)
## The End

I really enjoyed learning a bit of Elm and coding that small application. The learning curve is really smooth and if you happen to know Haskell you can start coding in Elm right away. Frontend is getting fun again thanks to Functional Reactive Programming. We should all be thankful to Elm since it has paved he way to bringing awesomeness to the frontend (Redux, React, re-frame...).

### Pros

* **Type System**: it speeds up development with its awesome error messages and strong guarantees (no runtime exceptions)
* **Persistent Data Structures**: all the most common datastructures are available in Elm core and are persistent (functional):
  * Records (Hashmap)
  * Lists
  * Arrays
  * Sets
* **Fun**: Elm is really fun to use (do not underestimate this because this is the main feature of Ruby).
* **Elm Architecture**: so simple but powerful way to compose and reuse pure functional code.
* **Tooling**: amazing tooling to recompile from the browser and hot swap code
  * Time traveling debugger
  * Hot code swapping

### Cons

* **Lack of adhoc polymorphism** (Haskell Typeclasses/Clojure protocols...): it is painful to use different functions (eg. List.map, Array.map, Set.map, ...) to perform similar operations on different datastructures. I think this is a even a deal breaker not to incorporate this in Elm.
* **Javascript interop**: it seems like a lot of ceremony to interop with javascipt libraries. You need to use `ports` and send messages back and forth. I do not see a simple solution to call javascript code directly from Elm code without using ports though.
* **Very young language**: The Elm community seems very excited and is growing all the time. Some fundamental libraries are not there yet and it may take a long time to get there.
