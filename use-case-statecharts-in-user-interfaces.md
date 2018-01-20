# Statecharts in User Interfaces

Statecharts are a good fit for solving certain problems with coding user interfaces.  Both statecharts and most UI technologies are event driven, but the technologies complement one another very well.

## Event Action paradigm

Most UIs are event driven, meaning that the user interface components themselves generate events when the user interacts with them.  These events typically trigger actions attached directly to the UI components.  In some systems, the events are passed around things like component hierarchies, or forwarded to an event loop that gets to decide what _action_ to perform.  The implementation of the UI component somehow decides what to do, what to show, what to stop doing and what to stop showing.

This is called the **Event Action** paradigm, because the action is tightly coupled to the event.

Statecharts inject themselves between the event being generated and the action being performed.  A statechart's main function is to take an event created somewhere, and make a decision on _doing stuff_.  It does precisely what most event driven UIs _don't_ do, or provide a framework for.

## A comparison

In a traditional event driven user interface component, say, the ubiquitous HTML `<input>` element, the UI component generates a lot of events, that the developer can subscribe to, from gaining or losing focus, editing, selecting, mouse movement and so on.  A developer has a plethora of events to choose between.  There is a similar almost infinite set of things that can be _done_ to a user interface.  Each element has a wide range of possible mutations.  It is up to the developer to decide _what to do_ based on whatever event that happens.

In simple user interfaces, the event-action paradigm is fine.  Here, for example is an input handler that turns the text green when text is entered into the field:

```css
.green {
    box-shadow: 0 0 30px green
}
```

```html
<input id="my_editor">
```

```js
var field = document.getElementById("my_editor");
function handleChange(e) {
  field.classList.add("green")
}
field.onchange = handleChange
```

Don't cringe, it's a simple component that turns green (an action) whenever a particular event happens (the field is modified).  This is the component's **behaviour**.

However, in order for this code to support _anything_ other than turning it green when it's modified, it will need conditional logic.  If the field should only turn green if the has any text, then the event handler needs an _if_ test, _and_ it needs to check the "current state" of the component.  This is the beginning of the complexity creep.

```js
function handleChange(e) {
  if (field.value == "") {
    field.classList.add("green")
  }
}
```

## Statecharts

As mentioned earlier, when implementing statecharts, the events are passed on to the statechart instead of being acted upon directly. The statechart then determines what to do.

The simplest state machine (a simple form of a statechart) is shown below. It has the same behaviour as the example above:

```js
var currentState = "not_green";
var stateMachine = {
  "not_green": () => field.classList.add("green")
}
function handleChange(e) {
  stateMachine[currentState]()
}
```

Now this state machine is extremely simple, it only has one state and knows only one way to behave.  But it is now a lot easier to make this component do more.

```js
var stateMachine = {
  "not_green": () => { field.classList.add("green"); currentState = "green"; },
  "green": () => { field.classList.remove("green"); currentState = "not_green"; }
}
```

By changing the state machine alone, we can now change the behaviour of the component.  It now alternates between having the "green" class every time the field changes.

We can introduce a _guard_ in order to prevent the event from having an effect.  Let's extend it so that the field is green only when text has been added to it:


```js
var stateMachine = {
  "not_green": () => {
    if (field.value == "") return;
    currentState = "green";
    field.classList.add("green");
  },
  "green": () => {
    if (field.value != "") return;
    currentState = "not_green";
    field.classList.remove("green");
  }
}
```

This is a crude approximation of a state machine, but it _is_ a state machine, and has a lot of the moving parts of a statechart too:

* It accepts events, although it simply treats all events equal. Real statecharts have named events.
* It has several states, `green` and `not_green`
* It has an "active" state (`currentState`)
* It reacts differently depending on which "active state" state it's in
* It also reacts differently depending on "real world" information (`if (field.value == "") return;`)
* It changes the "current" state when it deals with an event
* It has side effects (known as actions, it adds and removes the `green` class)

It has a few limitations too, though

* It is only a glorified enumeration with side effects.
* It's just _two states_
* It only handles _one event_
* It's extremely tightly coupled to the rest of the system (it talks directly to the DOM)
* It's difficult to extend with substates

While it is possible to "roll your own" state machine, it is likely not worth the effort.  There are numerous edge cases that you need to consider, and if you're not careful, such a one-off state machine becomes more difficult to maintain than the original spaghetti code it was meant to displace.  It is a bit like rolling your own date handling code; it works for the simplest of cases, but is quickly outgrown.

The code shown above was introduced solely to describe how a state machine fits in, in the context of user interfaces.  In order to show more advanced examples, it's necessary to avoid coding the inner workings of the state machine, and try to focus on the important parts, namely the different states, and which events causes the states to change.

In order to help, we'll be introducing a statechart library

## Introducing xstate

Xstate is a javascript library that essentially allows us to hide the inner workings of the state machine.  You provide it with an object that _describes_ the state machine you're interested in, and xstate returns a state machine that provides _pure functions_ that you can use to answer the question: "If I'm in _this_ state, and _this_ happens, _what_ should I do?"

To give you a quick primer, I'll rewrite the green / not green state machine above using xstate:


First of all, we'll need a state machine, constructed by xstate.

```js
const Machine = ...;

var stateMachine = new Machine({
  initial: "not_green",
  states: {
    "not_green": {
      on: {
        "change": "green"
      }
    },
    "green": {
      on: {
        "change": "not_green"
      }
    }
  }
}
```

One thing to note is that in state machines, and statecharts, events are given explicit names.  For our simple example I called the event `change`.

Now, this `stateMachine` variable, after it's constructed provides a _pure function_ interface to the state machine.  This means that this state machine cannot and wiil not have side effects.  Every time you use it, you tell it what the "current" state is, and what happens, and _it_ tells you what happened.

We start off by asking the state machine what the "initial" state is:

```js
var currentState = stateMachine.initialState;
```

`currentState` is an xstate _State_ instance, which has a _name_ which should be `green`.

You can then simulate what happens if you pass it the `change` event:

```js
currentState = statemachine.transition(currentState, "change");
```

`currentState` will describe the `green` state, and if you did it again, it would be the `not_green` state once again.  What we have might seem like a pretty advanced boolean, but don't despair.  It's time to hook this state machine up to our user interface.  To start with, we'll let any change in the text field will trigger the "change" event.

```js
var field = document.getElementById("my_editor");

function handleChange(e) {
  newState = stateMachine.transition(currentState, "change");
  if (newState) {
    currentState = newState;
    if (currentState.name == "green") {
      field.classList.add("green");
    }
    if (currentState.name == "not_green") {
      field.classList.remove("green");
    }
  }
}

field.onchange = handleChange
```




### UI modeling

When you have a state machine or statechart that "drives" your UI, it is quite common for the states in the statechart to (at least at the highest level) correspond to "modes" of the user interface.

... TKTK some words from the "render from state" vs "render from action" discussion:
* Actions should be used for side effects
* User interface changes could be deemed a side effect, so _can_ be controlled via actions
* The "current state" can be thought of as an implicit side effect
* It's possible to take the "current state" and control user interface changes based on it

Especially in declarative UI frameworks like HTML or React, it makes a lot of sense to model the statechart based on different "modes" of the UI, and use normal statechart mechanisms to control which is the "current state".  It therefore makes a lot of sense to re-use the "current state" and pass this knowledge on to the declarative UI, basically asking the UI to render the "current state" UI.

This has the benefit of keeping the statechart very much in line with the major modes of the UI.  When a component gets a new "mode of operation", it also gets a new "top level state".  This makes it easier when showing e.g. a statechart to non-developers, like QA or designers, since they will quickly recognise the states and be able to relate to them.

TKTK not finished yet.