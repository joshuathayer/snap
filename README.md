snap
====
Snap is an HTML generation library loosely inspired by Facebook's [react](http://facebook.github.io/react/). It's implemented in [Joxa](https://github.com/erlware/joxa), a lisp which runs on the Erlang VM.


Installation
------------

After installing Erlang and Joxa, this repo can be cloned and Snap can be compiled using [rebar](https://github.com/basho/rebar):

```
$ rebar make-deps
$ rebar compile
```

This will leave `snap.beam` and `snap-util.beam` in the `ebin` directory, which can be used in any other erlang or joxa project.

Examples
--------

### Basic tag functions
Snap generates HTML by calling functions. Every HTML5 tag has a corresponding Snap function. Calling a "tag" function on its own returns an empty HTML tag:

```clojure
(html)
;; "<html></html>"
(h1)
;; "<h1></h1>"
```

Content can be added to the generated HTML by providing arguments to the function calls:

```clojure
(h2 "Great News")
;; "<h2>Great News</h2>"
```

Function calls may be nested, to generate nested HTML tags:

```clojure
(div (p "A paragraph within a div"))
;; "<div><p>A paragraph within a div</p></div>"
```

Functions are of arbitrary arity, so mutiple tags (and text blocks) may be nested:

```clojure
(html (head (title "An interesting page")) (body (p "Interesting content")))
;; "<html><head><title>An interesting page</title></head><body><p>Interesting content</p></body></html>"
```

### Attributes
Attributes can be added by passing an erlang [proplist](http://www.erlang.org/doc/man/proplists.html) as the first argument to any tag function:

```clojure
(a [{:href "https://firstlook.org"} {:class "offsite-link"}] "First Look Media")
;; <a href="https://firstlook.org" class="offsite-link">First Look Media</a>
```

Components and Composition
--------------------------
Snap was inspired by React's [component model](http://facebook.github.io/react/docs/multiple-components.html), which encourages composition of parts into complete documents.

Any function which returns HTML may be used as an argument to a Snap tag function. Consider a "red-button" function:

```clojure
(defn+ red-button (button-text)
  (button [{:class "big red rounded-corners"}] button-text))
```

That function can be used in any other Snap tags, now:

```clojure
(ul
  (li (red-button "panic"))
  (li (red-button "light hair on fire")))

;; <ul>
;;   <li><button class="big red rounded-corners">panic</button></li>
;;   <li><button class="big red rounded-corners">light hair on fire</button></li>
;; </ul>
```

Data Binding
------------
React is strict about how data flows between components, modeling data flow as a directed graph from parent to child, and requring that data is passed in a "Props" dict.

I have not had enough experience using Snap to generate complex pages to have developed rules for how data should be passed between components. It might end up being the case that strict rules are not needed- Joxa is a functional language which encourages the kind of parent->child data flow that React enforces on a non-functional platform (javascript).


On-the-fly Compilation
----------------------
An interesting feature of Snap is on-the-fly compilation of arbitrary Snap code, which enables generating HTML from content (code) which didn't exist at system boot time, even if that code/content doesn't exist on disk. In practical terms, this means we can display HTML generated from Snap code which itself was generated from structured data stored in a database.

Consider a blogging platform, where an in-browser rich text editor (say, the Guardian's [Scribe](https://github.com/guardian/scribe)) POSTs JSON-formatted structured data back to a RESTy server, which saves the content to its database.

How, then, would a Snap-based system display that content?

Snap implements a json-to-snap function which takes a JSON document which represents a snippit of HTML (see [JSON format](#)) and generates valid Snap source which, when evaluated, will render the corresponding HTML.

That generated source is then compiled on the fly after being wrapped in a simple server function. An Erlang process is spawned with the newly-compiled code, which upon receipt of a message will run the created Snap function.

The HTML snippit generated by the Snap function is then sent back to the caller, which can wrap that content in its own Snap functions, and eventually send data back to the caller.

In this way, an "article" is only ever fetched from the DB, rendered in to source, compiled, and wrapped into a process once (per lifetime of the server). Informal testing on my MacBook Pro shows simple page generation time on the order of 40ms for the first request (involving the DB, compilation, etc), and 15ms for subsequent requests.

This is very rudimentary code at the moment, but here's an example of the process

```clojure
(let (
  ;; given an article ID, either return the existing erlang process responsible
  ;; for generating the article's HTML snippet. The second argument here is a
  ;; closure that will return the article's structured data: we'll use this function
  ;; in the case where rendering function is not yet spawned.
  render-pid (snap-util/find-or-spawn-renderer
              article-id
              (fn () (article/load-draft-content article-id)))

  ;; send a message to the renderer process, which will trigger content generation
  _          (erlang/send render-pid {(erlang/self)})

  ;; receive rendered content from process
  content    (receive ({res} res))

  ;; wrap the content in a full page
  full-content (html (head (title "An Article"))
                     (body (h1 "Your article") (div content)))))
  
```

## Known Issues
  * no tests
  * fragile
  * generated HTML is ugly: no newlines or indentations
  * many others...
