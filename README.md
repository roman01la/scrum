<img src="logo.png" width="251" height="36" alt="scrum logo" />

*State Coordination for [Rum](https://github.com/tonsky/rum/)*

## Table of Contents

- [Motivation](#motivation)
- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
- [How it works](#how-it-works)
  - [Model state with the Reconciler](#model-state-with-the-reconciler)
  - [Dispatch event with the Dispatcher](#dispatch-event-with-the-dispatcher)
  - [Handle events with Controllers](#handle-events-with-controllers)
  - [Query state reactively with Subscriptions](#query-state-reactively-with-subscriptions)
- [Roadmap](#roadmap)
- [License](#license)

## Motivation

Have a simple, [re-frame](https://github.com/Day8/re-frame) like state management facilities to build web apps with Rum library while leveraging its API.

## Features

- Decoupled application state in a single atom
- Reactive queries
- A notion of *controller* to keep application domains separate
- No global state, everything lives in `Reconciler` instance

## Installation

Add to project.clj/build.boot: `[org.roman01la/scrum "1.0.0-SNAPSHOT"]`

## Usage

```clojure
(ns counter.core
  (:require [rum.core :as rum]
            [scrum.core :as scrum]))

;;
;; define controller & action handlers
;;

(def initial-state 0)

(defmulti control (fn [action] action))

(defmethod control :init []
  initial-state)

(defmethod control :inc [_ _ counter]
  (inc counter))

(defmethod control :dec [_ _ counter]
  (dec counter))


;;
;; define UI component
;;

(rum/defc Counter < rum/reactive [r]
  [:div
   [:button {:on-click #(scrum/dispatch! r :counter :dec)} "-"]
   [:span (rum/react (scrum/subscription r [:counter]))]
   [:button {:on-click #(scrum/dispatch! r :counter :inc)} "+"]])


;;
;; start up
;;

;; create Reconciler instance
(defonce reconciler
  (scrum/reconciler {:state (atom {}) ;; initial state
                     :controllers {:counter control}}))

;; initialize controllers
(defonce init-ctrl (scrum/broadcast-sync! reconciler :init))

;; render
(rum/mount (Counter reconciler)
           (. js/document (getElementById "app")))
```

## How it works

With _Scrum_ you build everything around a well known architecture pattern in modern SPA development:

*MODEL STATE* (with `reconciler`)

↓

*DISPATCH EVENT* (with `dispatch!`, `dispatch-sync!`, `broadcast!`, `broadcast-sync!`)

↓

*HANDLE EVENT* (with `:controllers` functions)

↓

*QUERY STATE REACTIVELY* (with `subscription`, `rum/react` and `rum/reactive`)

↓

*RENDER* (automatic ! profit :+1:)

### Model state with the Reconciler

Reconciler is the core of _Scrum_. An instance of `Reconciler` takes care of application state, handles actions and subscriptions, and performs batched updates (via `requestAnimationFrame`):

```clojure
(defonce reconciler
  (scrum/reconciler {:state (atom {})
                     :controllers {:counter control}}))
```

The value at the `:state` key is the initial state. You can pass either an atom of empty map that will be populated with the `:init` event or an atom with a map containing the whole initial state, modeled at your convenience.

The value at the `:controllers` key is a map from key to controller function. The controller stores its state as a value of its key from this map. So the keys in the `:controllers` will be reflected in the `:state` atom. This is where modeling state happens and application domains keep separated.

*NOTE*: the `:init` event pattern isn't enforced at all by scrum, but we consider it a good idea for 2 reasons :
- it decouples the setup of scrum with the gathering of the initial state, gathering that could happen in several ways (hardcoded, read from some global JSON/Transit data pasted in HTML from the server, a user event, etc.)
- it allows setting a global watcher in the atom for ad-hoc stuff outside of the normal scrum cycle for maximum flexibility.

### Dispatch event with the Dispatcher

Dispatcher communicates intention to perform an action, whether it is updating the state or performing a network request. By default an action is executed asynchronously, use `dispatch-sync!` when synchronous action is required:

```clojure
(scrum.core/dispatch! reconciler :controller-name :action-name &args)
(scrum.core/dispatch-sync! reconciler :controller-name :action-name &args)
```

`broadcast!` and its synchronous counterpart `broadcast-sync!` should be used to broadcast an action to all controllers:

```clojure
(scrum.core/broadcast! reconciler :action-name &args)
(scrum.core/broadcast-sync! reconciler :action-name &args)
```

### Handle events with Controllers

Controller is a multimethod which executes actions against application state. A controller usually have at least an initial state and `:init` method.

```clojure
(def initial-state 0)

(defmulti control (fn [action] action))

(defmethod control :init [action &args db]
  (assoc db :counter initial-state))

(defmethod control :inc [action &args db]
  (update db :counter inc))

(defmethod control :dec [action &args db]
  (update db :counter dec))
```

It's important to understand that the value returned by a controller won't affect the whole state, but only the part corresponding to its associated key in the `:controllers` map of the reconciler.

### Query state reactively with Subscriptions

A subscription is a reactive query into application state. It is basically an atom which holds a part of the state value. Optional second argument is an aggregate function which computes a materialized view. You can also do parameterized and aggregate subscriptions.

Actual subscription happens in Rum component via `rum/reactive` mixin and `rum/react` function which hooks in a watch function to update a component when an atom gets an update.

```clojure
;; normal subscription
(defn fname [reconciler]
  (scrum.core/subscription reconciler [:users 0 :fname]))

;; a subscription with aggregate function
(defn full-name [reconciler]
  (scrum.core/subscription reconciler [:users 0] #(str (:fname %) " " (:lname %))))

;; parameterized subscription
(defn user [reconciler id]
  (scrum.core/subscription reconciler [:users id]))

;; aggregate subscription
(defn discount [reconciler]
  (scrum.core/subscription reconciler [:user :discount]))

(defn goods [reconciler]
  (scrum.core/subscription reconciler [:goods :selected]))

(defn shopping-cart [reconciler]
  (rum/derived-atom [(discount reconciler) (goods reconciler)] ::key
    (fn [discount goods]
      (let [price (->> goods (map :price) (reduce +))]
        (- price (* discount (/ price 100)))))))

;; usage
(rum/defc NameField < rum/reactive [reconciler]
  (let [user (rum/react (user reconciler 0))])
    [:div
     [:div.fname (rum/react (fname reconciler))]
     [:div.lname (:lname user)]
     [:div.full-name (rum/react (full-name reconciler))]
     [:div (str "Total: " (rum/react (shopping-cart reconciler)))]])
```

## Best practices

- Pass the reconciler explicity from parents components to children. Since it is reference type it won't affect shouldComponentUpdate aka rum/static optimization. But if you prefer to do it Redux-way, you can use context in Rum as well https://github.com/tonsky/rum/#interop-with-react
- Set up the initial state by `broadcast-sync!`ing an `:init` event before first rendering. This way you're free to gather the initial as you need in each controller and can setup a global watcher in the atom passed to `:state` in the reconciler.

## Roadmap
- <strike>Get rid of global state</strike>
- Make scrum isomorphic (in progress, see [this issue](#3))
- Storage agnostic architecture? (Atom, DataScript, etc.)
- Better effects handling (network, localStorage, etc.)

## License

Copyright © 2017 Roman Liutikov

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
