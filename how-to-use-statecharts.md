---
author: Erik Mogensen
---

# How to use statecharts

This page tries to describe some aspects of employing statecharts in your day-to-day coding routine.

## Determine scope

When you first learn about statecharts, you might get the feeling that statecharts can be used to describe the _entire_ behaviour of an application, all the way from which screens show as part of logging in, to the state of each checkbox and text field in every screen, all represented in a statechart.  That would be a statechart from hell, and an even bigger maintenance burden.  Instead, the focus should be to get a grip on the **behaviour at the component level**, whatever a component might be.  A single screen would be a component, for sure.  A single text field that might have some particular internal behaviour (e.g. it changes color based on various flags like _required_ or _invalid_) might warrant that to be wrapped in a component with a statechart to describe its behaviour.

In general, structure your code as you did before, by dividing things up into components.  Use statecharts to describe the behaviour of each component in isolation.  Use events and so on to get communication between components going just like before; keep the statechart _internal_ to the component itself.  The _user_ of the component shouldn't need to know that there's a statechart within it, other than by inspecting the code or guessing (since it behaves so well).

## Distill the API between the component and the statechart

To start using a statechart, the tangled mess that might be your component and its behaviour need to be disentangled: The _what_ / _how_ needs to be separated from the _why_.  You should end up with a business object that exposes functions that each does one useful part.

The communication between this business object happen in three distinct ways, and they usually execute in this order:

- Your object tells the statechart about an [event](glossary/event.html){:.glossary} — something happening either outside or inside the component. e.g. a keystroke, or a HTTP request response arrived.
- The statechart asks the world about some thing, known as a [guard](glossary/guard.html){:.glossary} — is the text field empty, or is the HTTP response complete.
- The statechart tells your object to perform some [action](glossary/action.html){:.glossary} or control some [activity](glossary/activity.html){:.glossary} — tell the field that it's "invalid", or start parsing the results, or stop a HTTP request.
- The statechart tells you _which states_ are active.

These are the four touchpoints between the statechart and the outside world (your component).  Statecharts fit into an event driven system.  It accepts events, and turns them into actions.

## Designing a statechart

This is the biggest hurdle if you're new to statecharts, mainly because it is often such a foreign concept. You need to decide on the notation, if you want the benefit of a graphical representation, and ultimately which tools you'll choose.

Tools aside, the process of designing a component's statechart is to start by discovering the _modes_ of the component in question.  Those are good candidates for top level states of your statechart.  You can start to think about what events that take you between those states.  Remember, at the "top level" things don't need to be 100% precise; this is what the substates are for.  Finally you can add what _happens_ in each state: the entry and exit handlers.  These are basically just hooks in your component to start and stop things.

You should already have some crude state machine which does at least _some_ of the things you want the component to do.  The next step is to refine the states.

This is done by concentrating on each top level state and trying to discover if the component behaves in different ways when _in_ that state; if so then introduce substates, and repeat the process outlined above for that.

## Example

In order to explain this process, I'm going to try to walk you through the process that went into the design of the following component:  A simple search form.  It is modeled after the same UI that is presented in [Robust React User Interfaces with Finite State Machines](https://css-tricks.com/robust-react-user-interfaces-with-finite-state-machines/).

To repeat the requirements from that article:

> * Show a search input and a search button that allows the user to search for photos
> * When the search button is clicked, fetch photos with the search term from Flickr
> * Display the search results in a grid of small sized photos
> * When a photo is clicked/tapped, show the full size photo
> * When a full-sized photo is clicked/tapped again, go back to the gallery view

Off the top of my head I can think of the following top level "modes" or "states" of this application:

* Initial — no search results are available
* Searching — when the search button was clicked
* Displaying results — when displaying results
* Zoomed in — when a photo is zoomed in on

If I put my statechart hat on, these can be thought of as "top level states":

![Initial set of states](how-to-use-statecharts-initial-states.svg)

Note that this set of states might change, since we haven't really understood what a "mode" is.  The main thing to look for is a different _behaviour_, i.e. that the component in question _reacts in a differnt way_ to events.  For example, the search app should react differently to a click on a photo when displaying results vs when zoomed in on a photo.

From the requirements it's also pretty easy to think up the transitions between those states too.  Here's the "happy path":

* Initial → Searching: someone typed something and hit the _Search_ button — I'll call this the **search** event
* Searching → Displaying results: the HTTP request completed with some data, the UI can be populated with stuff—I'll call this the **results** event.
* Displaying results → Zoomed in: The user clicked a photo and we now _zoom in_ on a particular photo.  I'll call this the **zoom** event
* Zoomed in → Displaying results: The user clicked a zoomed in photo and we now _zoom out_ back to the results.  I can call this the **zoom_out** event.

Again, these can be drawn into the statechart's "top level":

![Initial set of states with transitions](how-to-use-statecharts-initial-transitions.svg)

And similarly, I can sum up the _activities_ that should happen in each state:

* Searching:
  * An HTTP request should be active
* Displaying results:
  * Some search results should be shown
* Zoomed in:
  * A single photo should be visible

A lot of statechart libraries don't support activities natively, so you might need to translate them into _actions_ instead:

* Searching:
  * on entry: fire HTTP request
  * on exit: cancel HTTP request (if still running)
* Displaying results:
  * on entry: parse the HTTP response and show some results
* Zoomed in:
  * on entry: zoom in on a particular photo
  * on exit: remove the zoomed-in photo

So with that, here's my initial stab at the statechart:

![Initial stab at statechart, depicting the above information](how-to-use-statecharts-initial-actions.svg)

Now, if you're an experienced statechart designer, you can probably already see one big shortcoming of this statechart.  Luckily, with a diagram, they are extremely easy to discover:  There is no way to get from the "results" state back into the searching state.  There are no direct arrows, and there is no path to get there.  (It was also not explicitly stated in the requirements document!)  I'm going to ignore this problem for now, because I want to show (later) how you can fix such problems _purely_ by making changes to the state machine.  So, if you spotted this by yourself, pat yourself on the back now.

### Initial implementation

At this point we have enough stuff to work on to be able to get an initial implementation running too, just to get the happy path running.  We can then check them off the list of "problems" that typically plague a quick implementation.

First off my preference is to code this statechart up in SCXML.  It'll give us a nice diagram and it's an _executable_ statechart, meaning I don't have to do any manual translation from this representation to code.  The SCION toolset can _run_ an SCXML file.

First off, the top level states:

SCXML:
```xml
<scxml>
  <state id="initial">
  </state>
  <state id="searching">
  </state>
  <state id="displaying_results">
  </state>
  <state id="zoomed_in">
  </state>
</scxml>
```

xstate:
```json
{ "initial": "initial",
  "states" : {
    "initial": {},
    "searching": {},
    "displaying_results": {},
    "zoomed_in": {}
  }
}
```

Let's add the transitions

SCXML:
```xml
<scxml>
  <state id="initial">
    <transition event="search" target="searching"/>
  </state>
  <state id="searching">
    <transition event="results" target="displaying_results"/>
  </state>
  <state id="displaying_results">
    <transition event="zoom" target="zoomed_in"/>
  </state>
  <state id="zoomed_in">
    <transition event="zoom_out" target="displaying_results"/>
  </state>
</scxml>
```

xstate:
```json
{ "initial": "initial",
  "states" : {
    "initial": {
      "on": {
        "search": "searching"
      }
    },
    "searching": {
      "on": {
        "results": "displaying_results"
      }
    },
    "displaying_results": {
      "on": {
        "zoom": "zoomed_in"
      }
    },
    "zoomed_in": {
      "on": {
        "zoom_out": "displaying_results"
      }
    }
  }
}
```

And then the entry/exit handlers.  Here's the full SCXML file, with namespace declaration and all.

scxml:
```xml
<?xml version="1.0"?>
<scxml xmlns="http://www.w3.org/2005/07/scxml">
  <state id="initial">
    <transition event="search" target="searching"/>
  </state>
  <state id="searching">
    <onentry>
      <script>startHttpRequest();</script>
    </onentry>
    <onexit>
      <script>cancelHttpRequest();</script>
    </onexit>
    <transition event="results" target="displaying_results"/>
  </state>
  <state id="displaying_results">
    <onentry>
      <script>showResults();</script>
    </onentry>
    <transition event="zoom" target="zoomed_in"/>
  </state>
  <state id="zoomed_in">
    <onentry>
      <script>zoomIn();</script>
    </onentry>
    <onexit>
      <script>zoomOut();</script>
    </onexit>
    <transition event="zoom_out" target="displaying_results"/>
  </state>
</scxml>
```

xstate:
```json
{ "initial": "initial",
  "states" : {
    "initial": {
      "on": {
        "search": "searching"
      }
    },
    "searching": {
      "onEntry": "startHttpRequest",
      "onExit": "cancelHttpRequest",
      "on": {
        "results": "displaying_results"
      }
    },
    "displaying_results": {
      "onEntry": "showResults",
      "on": {
        "zoom": "zoomed_in"
      }
    },
    "zoomed_in": {
      "onEntry": "zoomIn",
      "onExit": "zoomOut",
      "on": {
        "zoom_out": "displaying_results"
      }
    }
  }
}
```

I've [extracted this SCXML into its own file](how-to-use-statecharts-initial-actions.scxml.xml), styled for a certain amount of interactivity and ability to explore:

<iframe src="how-to-use-statecharts-initial-actions.scxml.xml" width="100%" height="500px"></iframe>

### API Surface

If we look at our _API surface_—the set of events, guards and actions that we have—we can start to compile a list of things that our UI needs to provide:

* Events: `search`, `results`, `zoom`, and `zoom_out`
* Guards: none (it's still quite a crude solution)
* Actions: `startHttpRequest`, `cancelHttpRequest`, `showResults`, `zoomIn`, and `zoomOut`
* States: `initial`, `searching`, `showing_results` and `zoomed_in`

> #### Note
> As described in [decoupled statecharts](#decoupled-statecharts-ftw), below, you could consider the states as being part of this API surface.  The entire set of states is rarely the API, because there will almost always be states that dictate behaviour that won't be _visible_ in the component.  To begin with, we will use a _coupled_ approach, and expose the _current state_ to the component, and allow it to do stuff on its own purely based on this.

### Absence of data transfer

Note the absence of any data being passed back and forth: The events themselves are pretty anonymous; this is about high level things that happen in the UI. Likewise, the actions are no-arg function calls; no data is being passed from the statechart to the model.  They don't have to be function calls; it really depends on how you implement it. If you choose to, you can get the statechart to emit _events_ instead, meaning that the component _listens_ for certain events from the state machine.  This is the approach I will be using later in this post.

This absence of data transfer also means that the component still needs to keep track of the "business state" — the variables and stuff that the component is busy working on.  The statechart doesn't need or care about those things, it is only concerned with triggering the right actions at the right times.

As an example: the `startHttpRequest` action is called.  It will have to:

* grab the text from the text field that the user has typed a search term into
* construct a URL to search using flickr's API
* fire off a HTTP request

These three things are hidden from the statechart; it doesn't need to know that all of this is happening.  Only if any of those things need to be treated as _behaviour_ should they be exposed to the statechart.

### Decouple the component from the statechart?

At this point in time I think it's very useful to point out dependency that might come creeping.  The component in question easily becomes dependent on the states in the statechart.  A decision has to be made, or else it will be made for you.

The statechart invariably starts out as a reflection of the _modes_ that the component has, e.g. enabled, disabled, loading and so on.  It is therefore common to use the "current state" of the statechart reflect it in the component somewhere.  For a HTML based component, this might translate into a top level CSS element, like `state-enabled` and `state-loading`.  For a React app, it might be that your app renders different things based on the top level state.  This is completely natural, and introduces an implied coupling between the statechart and the component.

This coupling may or may not be beneficial, depending on how you end up using the statechart, but you should be aware of the coupling and the problems it introduces.  

| Decoupled | Coupled |
| --------- | -------- |
| The component **doesn't know** which state it's in | The component **knows** which state it's in |
| The component is explicitly _told_ when to change its mode, because the statechart says when _entering_ this state, _enter this mode_ | The component changes its mode automatically: whenever the statechart has handled an event, the component asks the statechart which state it's in and uses that |
| The component is explicitly _told_ when to do stuff, because the statechart says when _entering_ this state, _do this_ | The component does things based on the "current state": Whenever the statechart has handled an event, the component asks the statechart which state it's in and executes various functions |

* If you decide to keep them **decoupled**, it comes at the increased cost of having to define additional actions—an increased "API surface" if you will.  Additionally the statechart needs to have entry handlers (and possibly exit handlers) to turn on (and off) modes in the component.  The statechart needs to be able to control _explicitly_ what the component should be doing at any time.
* If you decide to keep them **coupled**, it comes at the cost of being unable to make changes to the statechart itself.  Introducing a new state to make a behavioural change can no longer be done _purely_ on the statechart side, because this new state might _affect the component_ when it **should not**, or it might _not affect it_ when it **should**.  Often a change in the statechart has to be done along with a change in the component.

### Decoupled statecharts FTW

When starting out with statecharts, it's often easiest to get going with **coupled** statecharts, since it's one less thing to learn.

For long term durability and maintainability of statecharts, it is probably best to go **decoupled**: remove the direct dependency between the statechart and the component.  If you choose to do this, there are some notable things that the component _should not_ be worrying about, such as:

- _Which state is the statechart in?_ — It really doesn't matter. What matters are the actions that are called.
- _Which transition just fired?_ — This too doesn't matter.

The things that matter in a decoupled statechart are: events, guards and actions.

## Coding up the API

The choice of UI framework should really not be very important, since what we're trying to describe is _what to do_ when certain events happen.  We aim to replace code that that litter the UI code to hide and show components based on the "state" of various variables, and replace them with actions that control the behaviour of the UI component.

### Actions

First off, let's take a look at the actions; the desired output of the state machine.  We need a component that can _react_ to what the state machine tells it to do.

To recap, the actions that the state machine can perform are: `startHttpRequest`, `cancelHttpRequest`, `showResults`, `zoomIn`, and `zoomOut`

A simple `performAction` function can do nicely:

```js
function performAction(event) {
  if (event === "startHttpRequest") {
    // get value of text field
    // construct URL
    // send HTTP request
  }

  if (event === "cancelHttpRequest") {
    // cancel HTTP request (if it's still active)
  }

  if (event === "showResults") {
  }

  if (event === "zoomIn") {
  }

  if (event === "zoomOut") {
  }
}
```

Forgive the naive if handler, as I opted for something readable.  I'll leave it to the readers to figure out ways of improving this code.

### Events

The next part of the API is actually sending events _from_ the world and _to_ the state machine.  This is also somewhat independent of the actual state machine library, as most of them take a string event.

To recap the events were: `search`, `results`, `zoom`, and `zoom_out` — these are all things that "happen" in the real world, that need to _tell the statechart_ about 

We want something that's similar to this:

```js
searchButton.onclick = statemachine.send("search");
httpRequest.then(() => statemachine.send("results"));
resultspanel.onclick = statemachine.send("zoom_in");
zoomedphoto.onclick = statemachine.send("zoom_out");
```

Essentially the buttons, HTTP requests and other things that _generate events_ don't need to know what's going to happen; they shouldn't.  They should just blindly send the information to the state machine, and let the state machine tell us what to do.

### States

Finally, the component is allowed to see which state is the "current" state.  This of course depends on the statechart library, but most will happily tell you the _name_ of the current state, typically as a string:

```js
currentState = machine.currentState; // 
```

## Implemention

We've now explained all of the parts necessary to get our UI and the component talking together.  (to be continued...)
