{:title "Snake the game - A Functional Programming Style tutorial for ClojureScript and Reagent"
 :layout :post
 :tags  ["Functional Programming" "React" "ClojureScript" "Reagent" "Re-frame"]
 :toc true}

Functional Programming is finally getting broadly used in the frontend! This is fantastic news! I have been programming in Clojure/ClojureScript for about 2 years. It seems to me that nothing can get quite close to it in terms of productivity and simplicity. So let's give it a try and see what else is out there.

## Rationale
I am starting this series of tutorials to compare different frontend abstractions to build modern real world applications. The goal is to provide the reader with some comparisons on different functional abstractions. I will focus on the following ones:

* ClojureScript + Reagent
* ClojureScript + Om
* Elm
* ReactJS

## ClojureScript and Reagent
The first post will focus on building a Snake Game in ClojureScript and Reagent, using the library re-frame. All the code is available on [my github](https://github.com/Chouffe/snake). Here is a demo of what we are about to build:

<link href="/posts/css/snake.css" rel="stylesheet" type="text/css">
<div id="snake"></div>
<script src="/posts/js/snake.js"></script>
<script>snake.core.init();</script>

## Setup
To get started, just go read about [re-frame](https://github.com/Day8/re-frame). Then run the following lein command to start an re-frame app.

> lein new re-frame <project-name>

That should generate the following project dir:

```
.
├── dev-resources
├── figwheel_server.log
├── project.clj
├── README.md
├── resources
│   └── public
├── src
│   ├── clj
│   └── cljs
├── target
│   ├── classes
│   ├── figwheel_temp
│   └── stale
└── test
    └── cljs
```

## Modeling State
The following concepts are needed to properly model the Snake Game:
* Board
  * width
  * height
* Snake
  * position
  * direction
* Apple position
* Points
* Is the game over?

### Board
The board is defined by a number of rows and columns.  simplest way to express it is to use a tuple of 2 elements `[width height]`.

```
(defn init-board [] [35 25])
```

### Snake
The snake is modeled with a direction in `#{:up :down :right :left}` and a sequence of positions `[[x1 y1] [x2 y2]]`.

```
(defn init-snake []
  {:direction :right
   :body [[3 2] [2 2] [1 2] [0 2]]})
```

### State
The game state is the combination of all the above

```
(defn rand-free-position [board unavailable-positions]
  (->> board
       positions
       (remove (set unavailable-positions))
       rand-nth))

(defn init-state []
  (let [snake (init-snake)
        board (init-board)]
    {:running? true
     :board board
     :snake snake
     :food (rand-free-position board (:body snake))
     :points 0}))
```

## The views
Reagent is a simple wrapper around React. We need to design our views as pure functions of the app-state. Re-frame achieves this by using FRP and subscribing to state changes.
### Score
The view function is very simple. It subscribes to the points value in the app-state.
```
(defn score []
  (let [points (subscribe [:points])]
    [:div.score "Score: " @points]))
```

And here is how a subscription gets registered:

```
(register-sub :points (fn [db] (reaction (:points @db))))
```

### Board
This view will be created with simple html elements. Canvas could have been used for that instead. Here, each cell of the board is a table cell with a given CSS class.
```
(defn board []
  (let [board (subscribe [:board])
        snake (subscribe [:snake])
        food (subscribe [:food])]
    (fn []
      (let [[width height] @board
            {:keys [body]} @snake]
        (into [:table.stage]
              (for [y (range height)]
                (into [:tr {:key y}]
                      (mapv (fn [x] [:td {:key (str x "." y)
                                          :class (cell-class [x y] @food (set body))}])
                            (range width)))))))))

(defn ^:private cell-class [position food-position snake-body-positions]
  (cond
    (= position food-position)          "food"
    (get snake-body-positions position) "snake"
    :else                               "empty"))
```
### Game Over
When the game is over, an overlay is added on top of the board with a replay button.
```
(defn game-over []
  (let [running? (subscribe [:running?])]
    (fn []
      (if @running?
        [:div]
        [:div.overlay
         [:div.play {:on-click #(dispatch [:start-game])}
          [:h1 "↺"]]]))))
```
### Game
The main view is then composed of the different components.
```
(defn game []
  [:div.game
   [score]
   [board]
   [game-over]])
```
The components are then mounted to the DOM with the function `run` defined below.
The main entrypoint of an re-frame app is the `init` function.
```
(defn run []
  (reagent/render [views/game]
                  (.getElementById js/document "app")))

(defn ^:export init []
  (run))
```
## Game Loop
The game loop is handled with a set-interval call that will create a new game state based on the current one. That state will then be handled by React (Reagent) and the view will get updated.
```
(defonce snake-moving
   (js/setInterval #(dispatch [:next-state]) 100))
```
Every 100ms, the message :next-state is sent to the handler.
## Handlers
### Game Initialization
In order to initialize the game, a handler `:start-game` is created. It resets the initial state.
```
(register-handler :start-game (fn  [_ _] (db/init-state)))
```
And the `init` function is updated accordingly.
```
(defn ^:export init []
  (dispatch-sync [:start-game])
  (run))
```
### Changing Direction
When a player interacts with the keyboard (Up, Down, Left, Right or h, j, k, l for vim users), the direction of the snake should get updated.
```
(register-handler
   :change-direction
   (fn [db [_ direction]]
      (assert (get #{:up :down :left :right} direction))
      (assoc-in db [:snake :direction] direction)))
```
A handler is just a reducer function that takes in the value of the app-state `db` and an action and returns a new app-state.
Then, a keyboard listener is defined to catch the keystrokes mentioned above.

```
(def key-code->move
   "Mapping from the integer key code to the direction keyword"
   {38  :up   ; Up Arrow
    75  :up   ; k
    40  :down ; Down Arrow
    74  :down ; j
    39  :right ; Right Arrow
    76  :right ; l
    37  :left  ; Left Arrow
    72  :left  ; h
    })

(defonce key-handler
   (events/listen
      js/window
      "keydown"
      (fn  [e]
         (let [key-code (.-keyCode e)]
            (when (contains? key-code->move key-code)
               (re-frame/dispatch [:change-direction (key-code->move key-code)]))))))

```
### Next State
The last thing to do is to implement the handler next-state that will move the snake given the direction and the state of the game.
#### Collisions
The game is lost when the snake hits a wall or eats itself.
```
(defn collision? [snake board]
    (or (wall? next-head board) (eat-itself? snake)))

(defn eat-itself? [{:keys [body] :as snake}]
  (boolean (get (into #{} body) (next-head-position snake))))

(defn valid? [[x y :as position] [width height :as board]]
  (and (< -1 x width) (< -1 y height)))

(def wall? (complement valid?))
```

```
(re-frame/register-handler
   :next-state
   (fn [db _]
      (let [{:keys [snake food board running?]} db]
         (cond
            (db/collision? snake board) (assoc db :running? false)
            (db/food? (db/next-head-position snake) food) (-> db
                                                              (update :points inc)
                                                              (update :snake db/move-snake)
                                                              (dissoc :food)
                                                              (update :snake db/grow-snake snake)
                                                              (db/rand-food))
            :else (update db :snake db/move-snake)))))
```
Here is the implementation of `move-snake`
```
(defn next-head-position
  [{:keys [body direction] :as snake}]
  (let [[x y :as pos] (first body)]
    (case direction
      :left  [(dec x) y]
      :right [(inc x) y]
      :up    [x (dec y)]
      :down  [x (inc y)])))

(defn move-snake
  [{:keys [body direction] :as snake}]
  (update snake :body (fn [body] (cons (next-head-position snake)
                                       (drop-last body)))))
```
Here is the implementation of `grow-snake`
```
(defn grow-snake
  [snake previous-snake]
  (update snake :body concat [(last (:body previous-snake))]))
```
And finally, here is `rand-food`
```
(defn rand-food [{:keys [snake board] :as state}]
  (assoc state :food (rand-free-position board (:body snake))))
```
## Development Environment
Something that is often not discussed is tooling. Tools are everything. I spent a lot of time fine tuning my environment to increase my productiviy.
To be honest, the ecosystem and tooling around Clojure and ClojureScript are just amazing.

### Figwheel
To develop this game I used [figwheel](https://github.com/bhauman/lein-figwheel) which enables live pushes to the clients. Here is a [quick demo](https://www.youtube.com/watch?v=KZjFVdU8VLI) of interactive development in ClojureScript.

This is what my workflow looks like when developing:
* Type something in my text-editor
* Save changes
* See live changes in the browser
  * live code reloading
  * live CSS reloading
* Repeat until done

I get instant feedback without leaving my text editor!

![Dev Environment](/posts/img/snake/liveupdate.png "Figwheel and Interactive Development")

## The End
Re-frame is a well thought out combination of **reactive programming**, **functional programming** and **immutable data**.

It lets developers focus on the core application in a beautifully simple way.
Re-frame emerged from ideas developed in Om, Elm and Flux that we are going to study next.

### Pros
* Clojure ❤
  * Immutability
  * Lisp
* Local states are just atoms captured in a closure. Local states can be shared between components.
* Reagent is faster than React: All the checks for changes are pointer comparison (thanks to Clojure's immutability)
* Pure functions: the side effects are pushed to the system's boundary. They should only exist in the event handlers.
* Composition via reactive data flows (unidirectional graph)
* Extremely simple (200 lines of code to implement re-frame)
* Decomplects view, state, business logic in a functional way
### Cons
* The views are only in Hiccup Style for now (Clojure Data Structure). That is hard for designers to create their own components...
