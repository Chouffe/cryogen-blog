{:title "Fractals - Exploration of the Mandelbrot Set with ClojureScript and React"
 :layout :post
 :tags  ["Functional Programming" "React" "ClojureScript" "Reagent" "Re-frame" "Fractals" "Math"]
 :toc true}

## Introduction

A fractal is a mathematical object that exhibits a pattern that repeats itself at every scale. They are very interesting mathematical concepts because they differ from regular geometrical figures. How does one measure the length a fractal curve? The Mandelbrot Set is a closed area of the complex plane but still, its perimeter is not a finite number.
Sophisticated structures arise from very simple rules!

![Mandelbrot Set](https://upload.wikimedia.org/wikipedia/commons/2/21/Mandel_zoom_00_mandelbrot_set.jpg)

The goal of this post is to see how we can build a simple UI tool to visualize the Mandelbrot Set.

![Mandelbrot Repeating Pattern](https://upload.wikimedia.org/wikipedia/commons/c/ce/Mandelbrot_zoom.gif)

## Mandelbrot Set

The **Mandelbrot Set** is the set of complex numbers such that the function f(z) = z^2 + c does not diverge when iterated.

A practical test is often to iterate n times and to check whether the value remains bouded (eg. < 2)

* f(z)
* f(f(z))
* f(f(f(z))) ...

This fractal is particularly interesting because it is one of the best-known examples of mathematical visualization or mathematical art. It has also some real world applications in complex dynamics.

## Modeling complex numbers

The Mandelbrot Set is a subset of the Complex plane. We need a way to represent [complex numbers](https://en.wikipedia.org/wiki/Complex_number) in Clojure. As a reminder, complex numbers are numbers that can be expressed in the form a + b*i* where a and b are real numbers and *i* is the imaginary unit.

I will not get really fancy implementing the complex operations required to generate the Mandelbrot Set. I may implement a robust library for complex numbers in the future but for now, we will just go with 3 operations:

* add: adds two complex numbers
* mult: multiplies two complex numbers
* abs: norm of a complex number

Also, there are two common ways to represent complex numbers. The first one is a tuple of imaginary part (Im) and real part (Re). The second one uses the distance from O (origin in the Complex plane) and the angle of OP and the x-axis (which is known as the argument of the complex number). Sometimes, operations can be performed more efficiently in one representation or the other. For now, I will only provide one representation for the complex numbers:

```
(ns mandelbrot.complex)

(defn z [x y] [x y])

(defn add [z1 z2]
  (let [[x1 y1] z1
        [x2 y2] z2]
    [(+ x1 x2) (+ y1 y2)]))

(defn mult [z1 z2]
  (let [[x1 y1] z1
        [x2 y2] z2]
    [(- (* x1 x2)
        (* y1 y2))
     (+ (* y1 x2)
        (* y2 x1))]))

(defn abs [z]
  (let [[x y] z]
    (#?(:clj Math/sqrt :cljs js/Math.sqrt) (+ (* x x) (* y y)))))
```

## Set generation

We only need a function `mandelbrot?` that takes in a complex number `z` and returns a boolean. `z` is in the set if and only if it does not diverge when repeatedly applying the function f.

f(u) = u^2 + z

```
(defn mandelbrot? [z]
  (let [max-iter 20]
    (loop [k 1
           m (iterate #(c/add z (c/mult % %)) (c/z 0 0))]
      (if (and (> max-iter k) (< (c/abs (first m)) 2))
        (recur (inc k) (rest m))
        (= max-iter k)))))
```

We can already generate the mandelbrot set with this `mandelbrot?` function. In order to make it look nicer, we can return the number of iterations needed to see that it diverges.
We will then be able to display a gradient of colors instead of only 2 colors.

```
(defn ^:private mandelbrot-n-iter-aux
  [z u n-max k]
  (cond
    (> (c/abs u) 2) [false k]
    (>= k n-max) [true k]
    :else (recur z (c/add z (c/mult u u)) n-max (inc k))))

(defn ^:private mandelbrot-n-iter [z]
  (mandelbrot-n-iter-aux z (c/z 0 0) 20 1))

(defn ^:private mandelbrot-value [range-x range-y]
  (for [y range-y
        x range-x]
    (let [[in-set? k] (mandelbrot-n-iter (c/z x y))]
      (if in-set? 0 k))))
```

## Demo

I built a simple visualization tool with ClojureScript, React, re-frame and the mandelbrot functions discussed above. The colors can be changed by sliding the ranges, zooming is enabled and clicking on the fractal moves it accordingly.

<link href="/posts/css/mandelbrot.css" rel="stylesheet" type="text/css">
<div id="mandelbrot"></div>
<script src="/posts/js/mandelbrot.js"></script>
<script>mandelbrot.core.init();</script>

The code is fully available on [my github](https://github.com/Chouffe/fractals). Feel free to explore the code and explore your own fractal objects.

## The End

The Mandelbrot Set is a very simple yet sophisticated mathematical object that derives from very simple rules. I'd like to generate other fractal objects (Julia, Sierpinski Gasket) in a next project.
