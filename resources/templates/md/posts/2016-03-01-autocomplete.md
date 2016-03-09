{:title "Autocomplete - Trie Datastructure in Clojure and ClojureScript"
 :layout :post
 :tags  ["Data Structures" "Functional Programming" "React" "ClojureScript" "Reagent" "Re-frame"]
 :toc true}

A [trie](https://en.wikipedia.org/wiki/Trie) is a powerful datastructure often used for dictionary representation. The goal of this post is to implement a trie in Clojure and ClojureScript. Then we will build an autocomplete system.

![Trie Datastructure](https://upload.wikimedia.org/wikipedia/commons/thumb/b/be/Trie_example.svg/600px-Trie_example.svg.png)

## Trie

* A trie or prefix tree (as they can be searched by prefixes) is an ordered datastructure that is used to store an associative array where the keys are ususally strings.
* All the descendants of a node have a common prefix of the string associated with that node.
* The root node is the empty string
* A prefix tree allows for quick search, insert and delete operations O(log k) where k is the length of the prefix string

## Demo

<link href="/posts/css/autocomplete.css" rel="stylesheet" type="text/css">
<div id="autocomplete-app"></div>
<script src="/posts/js/autocomplete.js"></script>
<script>autocomplete.core.init();</script>

* The left input field allows you to add strings in the trie
* the right input field performs autocompletes
* The trie is represented as nested list for visualization
* Some preset lists of strings can be loaded by clicking on the buttons `Reset`, `Small Alphabet`, `Countries` or `Languages`.
* The colors of the elements in the trie change if they match the prefix-string

## Implementation

You can find my trie implementation on [github](https://github.com/Chouffe/clj-trie) and [clojars](https://clojars.org/trie).

### Datastructure

Each node of the trie will contain the following keys:
* **value**: the current prefix string at that node, eg "te", "in"...
* **children**: a hashmap from char to node
* **key**: flag to say if the value of that node is in the dictionary

The empty trie is then
```
(defn- make-empty-trie
  "Creates an empty trie with empty lexicon."
  []
  {:value ""})
```

### Operations

#### Conj - Adding to a trie
When adding a string to a trie, we need to perform the following:
* Go down the tree following the characters of the string
* Create nodes (if they do not exist yet) unitl the last character of the string is reached
* Add/Update the node with a flag to say that this word (path from root to the node) has been added to the trie

```
(defn- conj-trie
  [{:keys [value children] :as trie} [c & cs :as string]]
  (let [ch (str c)]
    (cond
      ; A leaf is reached and last letter in string
      (and (not (get children ch)) (not (seq cs)))
      (assoc-in trie [:children (str c)] {:value (str value c) :key (gensym)})

      ; A leaf is reached and still some letters
      (and (not (get children ch)) (seq cs))
      (assoc-in trie [:children ch] (conj-trie {:value (str value c)} cs))

      ; No more letters but already present in the trie
      (and (get children ch) (not (seq cs)))
      (assoc-in trie [:children ch :key] (gensym))

      ; Already in trie and still letters
      (and (get children ch) (seq cs))
      (update-in trie [:children ch] conj-trie cs))))
```

Then adding a set of strings is just a fold over the collection

```
(defn- into-trie
  [trie coll]
  (reduce conj-trie trie coll))
```

Here is the Clojure Datastructure that represents the trie
```
{:value "",
 :children
 {"t"
  {:value "t",
   :children
   {"e"
    {:value "te",
     :children
     {"a" {:value "tea", :key G__22475},
      "d" {:value "ted", :key G__22477},
      "n" {:value "ten", :key G__22478}}},
    "o" {:value "to", :key G__22476}}},
  "a" {:value "a", :key G__22479},
  "i"
  {:value "i",
   :key G__22480,
   :children
   {"n"
    {:value "in",
     :key G__22481,
     :children {"n" {:value "inn", :key G__22482}}}}}}}
```
### Search
To search into the trie (autocomplete given a prefix string), we only need to go down the trie following the prefix string characters and then collect the elements in this subtree.
#### apply-trie

The `apply-trie` function goes down the trie following the characters and then call the funtion `trie->seq` on this subtree.

```
(defn- apply-trie
  [{:keys [value key children] :as trie} [c & cs :as string]]
  (cond
    (not (seq string))           (trie->seq trie)
    (not (get children (str c))) []
    :else                        (recur (get children (str c)) cs)))
```
#### trie->seq
This function is just a DFS on the given trie and returns the element in order.

```
(defn- trie->seq
  "Returns all the elements inserted in trie as a sequence."
  [{:keys [value children key] :as trie}]
  (loop [stack [trie]
         words []]
    (if (not (seq stack))
      (seq words)
      (let [{:keys [value children key]} (peek stack)]
        (recur (into (pop stack) (vals children))
               (if key (conj words value) words))))))
```
### Protocols and Interfaces
#### Rationale
To make it feel like a built-in clojure Datastructure, we can define a Trie type that will implement some common clojure interfaces

* IPersistentCollection
* ILookup
* IFn

The goal is to be able to do the following

```
;; Creating a trie
(def autocomplete (trie ["doo" "foo" "bar" "baz"]))

;; Autocomplete given a prefix-string
(autocomplete "ba") => ["bar" "baz"]

;; Adding to the trie
((conj autocomplete "baf") "ba") => ["baf" "bar" "baz"]
```

#### Implementation
Clojure implementation

```
(deftype Trie [trie lexicon-set]

      clojure.lang.ILookup
      (valAt [self prefix-string] (get lexicon-set prefix-string))
      (valAt [self prefix-string default] (get lexicon-set prefix-string default))

      clojure.lang.IPersistentCollection
      (seq [self]     (seq lexicon-set))
      (cons [self o]  (Trie. (conj-trie trie o) (conj lexicon-set o)))
      (empty [self]   (Trie. (make-empty-trie) #{}))
      (equiv [self o] (and (instance? Trie o) (= trie (.trie o))))

      clojure.lang.IFn
      (invoke [self prefix-string]
        (let [completions (apply-trie trie prefix-string)]
          (if (seq completions) completions nil)))

      Object
      (toString [self] (str trie)))
```

ClojureScript implementation

```
(deftype Trie [trie lexicon-set]

      cljs.core/ILookup
      (-lookup [self prefix-string] (get lexicon-set prefix-string))
      (-lookup [self prefix-string default] (get lexicon-set prefix-string default))

      cljs.core/ICollection
      (-conj [self o] (Trie. (conj-trie trie o) (conj lexicon-set o)))

      cljs.core/ICounted
      (-count [self] (count lexicon-set))

      cljs.core/IEmptyableCollection
      (-empty [self] (Trie. (make-empty-trie) #{}))

      cljs.core/IFn
      (-invoke [self prefix-string]
        (let [completions (apply-trie trie prefix-string)]
          (if (seq completions) completions nil)))

      Object
      (toString [self] (str trie)))
```

Then the use of conditional readers and .cljc files allow us to make our trie work for Clojure and ClojureScript

```

#?(:clj
    (deftype Trie [trie lexicon-set]

      clojure.lang.ILookup
      (valAt [self prefix-string] (get lexicon-set prefix-string))
      (valAt [self prefix-string default] (get lexicon-set prefix-string default))

      clojure.lang.IPersistentCollection
      (seq [self]     (seq lexicon-set))
      (cons [self o]  (Trie. (conj-trie trie o) (conj lexicon-set o)))
      (empty [self]   (Trie. (make-empty-trie) #{}))
      (equiv [self o] (and (instance? Trie o) (= trie (.trie o))))

      clojure.lang.IFn
      (invoke [self prefix-string]
        (let [completions (apply-trie trie prefix-string)]
          (if (seq completions) completions nil)))

      Object
      (toString [self] (str trie))))

#?(:cljs
    (deftype Trie [trie lexicon-set]

      cljs.core/ILookup
      (-lookup [self prefix-string] (get lexicon-set prefix-string))
      (-lookup [self prefix-string default] (get lexicon-set prefix-string default))

      cljs.core/ICollection
      (-conj [self o] (Trie. (conj-trie trie o) (conj lexicon-set o)))

      cljs.core/ICounted
      (-count [self] (count lexicon-set))

      cljs.core/IEmptyableCollection
      (-empty [self] (Trie. (make-empty-trie) #{}))

      cljs.core/IFn
      (-invoke [self prefix-string]
        (let [completions (apply-trie trie prefix-string)]
          (if (seq completions) completions nil)))

      Object
      (toString [self] (str trie))))
```

Finally, the main factory function is the `trie` function

```
(defn trie
  ([]     (Trie. (make-empty-trie) #{}))
  ([coll] (into (trie) (set (map s/lower-case coll)))))
```

## Autocomplete System and UI

### Introduction

I wanted to build a small [UI interface](#demo) for my trie datastructure. The hard part was the trie visualization. I decided to use nested unordered lists to replicate the tree structure of the trie. I am still not sure convinced that is the best way to visualize a trie but it is good enough for the scope of this project.

### Implementation

* ClojureScript
* re-frame (React + Reagent)
* trie library developed before
* Bootstrap for CSS style

Actually, it took me more time to build the UI than to build and deploy the trie datastructure. It is hard to come up with good and simple UI components to help the reader understand how this datastructure works.

It is also on my [github](https://github.com/Chouffe/autocomplete) if you want to take a look at the code :)
