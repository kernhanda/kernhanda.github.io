---
title:  "Creating a canvas to draw on using Typescript"
date:   2018-01-03
toc:    true
---
A quick tutorial on how to create a small web application to handle the user
drawing on HTML `canvas` elements.

## Introduction

The aim of this tutorial is to demonstrate how to work with `canvas`
elements. This will be done by creating a simple web page and using
typescript to write the code that powers the logic of the page, which will
allow the user to draw on the `canvas` element like is possible in a trivial
paint application.

## Why Typescript

I'm not an expert in web development, so I really appreciate the static
typing available in Typescript. Combined with VS Code's Intellisense for
Typescript, it makes it fairly easy to start writing new functionality and
discovering new APIs while being reasonably certain of its correctness.

## The files

There are three files needed for this demo:

* `tsconfig.json` - configures the typescript compiler
* `index.html` - simple HTML page with a `canvas` and `button` element
* `main.ts` - the typescript file that we will write and compile

### `tsconfig.json`

This file is relatively straightforward. It sets the target for the compiled
Javascript code (ES5), turns on source map for easier debugging, and a few
other relatively inconsequential things. If you'd like to read more, a nice
explanation of `tsconfig.json` is available [here][1] and a page explaining
the compiler options is available [here][2].

```json
{
    "compilerOptions": {
        "target": "es5",
        "sourceMap": true,
        "lib": [
            "es5",
            "dom"
        ],
        "noUnusedLocals": true,
        "module": "commonjs"
    }
}
```

### `index.html`

Another relatively straightforward file, the contents `index.html` will serve
as a foreshadowing of what we're going to make.

```html
<html>

<head>
    <title>Canvas demo</title>
</head>

<body>
    <canvas id="canvas" width="490" height="490"
            style="border: 1px solid black;">
    </canvas>
    <p id="clear">clear</p>
    <script src="main.js"></script>
</body>

</html>
```

This creates a `canvas` element (with the aptly-named id, `canvas`). It also
creates a text label (the `p` element with id `clear`) that will clear the
drawing area when clicked.

### `main.ts`

Finally, we start writing the main file. We begin with the skeleton of the
class that is going to contain the functionality:

```typescript
class DrawingApp {

}

new DrawingApp();
```

`DrawingApp` is the name of the class we're going to fill in. At the end, you
can see we instantiate a new class. Upon instantiation, we will find the
relevant DOM elements and register event handlers for the events that we're
interested in.

This is done by writing a constructor for the class. Add the following
snippet to the body of the `DrawingApp` class:

```typescript
private canvas: HTMLCanvasElement;
private context: CanvasRenderingContext2D;
private paint: boolean;

private clickX: number[] = [];
private clickY: number[] = [];
private clickDrag: boolean[] = [];

constructor() {
    let canvas = document.getElementById('canvas') as
                 HTMLCanvasElement;
    let context = canvas.getContext("2d");
    context.lineCap = 'round';
    context.lineJoin = 'round';
    context.strokeStyle = 'black';
    context.lineWidth = 1;

    this.canvas = canvas;
    this.context = context;

    this.redraw();
    this.createUserEvents();
}
```

The `constructor` method is called automatically when instantiating the
class, the same as constructor methods in other OOP languages. In the
constructor, the first thing we do is get a handle to the element that has
`canvas` as the id. We then get a 2D rendering context from the canvas. Other
modes are also [available][4]. For the 2D context, we then set some defaults
for our drawing app. Finally, we store the handles in the class's variables
and call a couple of methods that we will define next (add these to the
class):

```typescript
private createUserEvents() {
    let canvas = this.canvas;

    canvas.addEventListener("mousedown", this.pressEventHandler);
    canvas.addEventListener("mousemove", this.dragEventHandler);
    canvas.addEventListener("mouseup", this.releaseEventHandler);
    canvas.addEventListener("mouseout", this.cancelEventHandler);

    canvas.addEventListener("touchstart", this.pressEventHandler);
    canvas.addEventListener("touchmove", this.dragEventHandler);
    canvas.addEventListener("touchend", this.releaseEventHandler);
    canvas.addEventListener("touchcancel", this.cancelEventHandler);

    document.getElementById('clear')
            .addEventListener("click", this.clearEventHandler);
}

private redraw() {
    let clickX = this.clickX;
    let context = this.context;
    let clickDrag = this.clickDrag;
    let clickY = this.clickY;
    for (let i = 0; i < clickX.length; ++i) {
        context.beginPath();
        if (clickDrag[i] && i) {
            context.moveTo(clickX[i - 1], clickY[i - 1]);
        } else {
            context.moveTo(clickX[i] - 1, clickY[i]);
        }

        context.lineTo(clickX[i], clickY[i]);
        context.stroke();
    }
    context.closePath();
}
```

`createUserEvents` does what the name says -- it sets up the handlers for the
canvas events that we're interested in. We register for both mouse and touch
events so that the app can work not only on normal computers but on mobile
devices as well. `redraw` is a little bit more complicated. The method uses
all the stored information about where the user clicks and drags the mouse
and uses that to draw on the `canvas` element. If the user was dragging
(clicking and moving the cursor at the same time), we draw a line through all
the points one-by-one. However, if they just clicked in a spot, we draw a
point one pixel wide. The stored state of the user's action is managed by the
`addClick` and `clearCanvas` methods:

```typescript
private addClick(x: number, y: number, dragging: boolean) {
    this.clickX.push(x);
    this.clickY.push(y);
    this.clickDrag.push(dragging);
}

private clearCanvas() {
    this.context
        .clearRect(0, 0, this.canvas.width, this.canvas.height);
    this.clickX = [];
    this.clickY = [];
    this.clickDrag = [];
}
```

These functions are used to add to and clear the state, respectively, of the user's actions on the `canvas` element. We store the X and Y coordinates along with a `boolean` representing whether the user was in the middle of dragging the cursor.

When the user does perform an action on the canvas (other than simply moving the cursor over it), our event handlers will get triggered, which we registered with the browser with the `createUserEvents` method. We'll start with the three simple handlers:

```typescript
private clearEventHandler = () => {
    this.clearCanvas();
}

private releaseEventHandler = () => {
    this.paint = false;
    this.redraw();
}

private cancelEventHandler = () => {
    this.paint = false;
}
```

`clearEventHandler` is registered to be called whenever the "clear" link is
clicked. All we have to do is call `clearCanvas` and the canvas is cleared.
`releaseEventHandler` is responsible for handling `mouseup` or `touchend`
events. These occur when the user stops holding the mouse button over the
`canvas` element. The event handler stores the state the user is no longer
drawing on the canvas and does a final `redraw` call. `cancelEventHandler` is
called whenever the user moves his mouse or finger outside of the `canvas`
element. This handler method stops the processing of all future mouse events
until the user performs another `mousedown` event on the `canvas` element.

Note the difference in syntax between these methods and the ones that were
written earlier. This is because of the way Javascript (and thus Typescript)
handle the binding of the `this` variable. Creating a closure for the event
handlers allows for the handler to keep a correct reference to the instance
when it's run as an event handler. There's a pretty good write-up available
[here][3].

Finally, we write the two methods responsible for handling the initial
clicking or touch event and the moving of the cursor while in a down state,
respectively:

```typescript
private pressEventHandler = (e: MouseEvent | TouchEvent) => {
    let mouseX = (e as TouchEvent).changedTouches ?
                 (e as TouchEvent).changedTouches[0].pageX :
                 (e as MouseEvent).pageX;
    let mouseY = (e as TouchEvent).changedTouches ?
                 (e as TouchEvent).changedTouches[0].pageY :
                 (e as MouseEvent).pageY;
    mouseX -= this.canvas.offsetLeft;
    mouseY -= this.canvas.offsetTop;

    this.paint = true;
    this.addClick(mouseX, mouseY, false);
    this.redraw();
}

private dragEventHandler = (e: MouseEvent | TouchEvent) => {
    let mouseX = (e as TouchEvent).changedTouches ?
                 (e as TouchEvent).changedTouches[0].pageX :
                 (e as MouseEvent).pageX;
    let mouseY = (e as TouchEvent).changedTouches ?
                 (e as TouchEvent).changedTouches[0].pageY :
                 (e as MouseEvent).pageY;
    mouseX -= this.canvas.offsetLeft;
    mouseY -= this.canvas.offsetTop;

    if (this.paint) {
        this.addClick(mouseX, mouseY, true);
        this.redraw();
    }

    e.preventDefault();
}
```

The methods essentially do the same thing with a couple of minor differences:

* get the X and Y coordinates of the event
* transform the coordinates to be relative to the `canvas` element itself
* add the click to the stored state
* call `redraw` thereby updating the drawing surface

One difference is that `dragEventHandler` makes sure the `paint` variable is
`true`. This is to prevent cases where the user performed a click event
outside of the `canvas` element but then moved the mouse inside of the
element. We also call `preventDefault`
([reference](https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault))
in `dragEventHandler` to improve the performance of the app. Without this,
dragging the cursor can cause a degradation in responsiveness.

Note that both event handlers do a little bit of [type
punning](https://en.wikipedia.org/wiki/Type_punning) to work around the fact
that the event handler is called for both mouse and touch events and thus the
type of `e` can vary. We detect the type of the event argument by checking
for the presence of the `changedTouches` field; if it's present, we have a
touch event, otherwise it's a mouse event.

## Demo

You can see a live demo of this tutorial [here](https://kernhanda.github.io/blog-demos/canvas/).

## Further reading

* [tsconfig.json][1]
* [Typescript compiler options][2]
* [`this` in Typescript][3]

[1]: https://www.typescriptlang.org/docs/handbook/tsconfig-json.html
[2]: https://www.typescriptlang.org/docs/handbook/compiler-options.html
[3]: https://github.com/Microsoft/TypeScript/wiki/'this'-in-TypeScript
[4]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/getContext
