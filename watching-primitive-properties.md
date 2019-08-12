# Watching Primitive Class Properties With TypeScript Getters and Setters

Ian Sanders - _8/9/2019_

This is the beginning of a series on watching class properties and taking action
when they change in TypeScript.

## Introduction

I fairly often encounter situations where I want to watch changes to a class
property. These properties could be anything, but for the first article in this
series I'll focus on primitives (properties with non-object or function types
like `string`, `boolean`, `number`, etc). For example, here is a class that will
be used as part of a drawing API. In later articles, we'll use this class to
draw on a canvas.

```ts
interface IShape {
  color: string;
  opacity: number;
  visible: boolean;
}
```

I want to design this drawing API so it's extremely intuitive to use. Notice how
the interface has no API methods for updating - there's no `setVisibility()` or
`redraw()` methods. Instead, I want to build this API to 'automagically' update
the drawing whenever a property changes:

```ts
circle.visible = false;
// => Canvas gets redrawn automatically.
```

## Introduction to Getters and Setters

Fortunately, this is a common pattern in JavaScript and TypeScript. Classes
offer handy features called 'getters' and 'setters' that allow us to do just
this. These are functions that, when installed on a class, act publicly just
like normal properties, even though internally they are functions. Class
consumers don't ever have to call these methods directly. Here's an example:

```ts
class Example {
  public get test() {
    return 5;
  }
  public set test(value: number) {
    console.log(`Test is always 5, so it can't be set to ${value}.`);
  }
}

const ex = new Example();
console.log(ex.test); // logs: 5
ex.test = 45; // logs: "Test is always 5, so it can't be set to 45."
console.log(ex.test); // logs: 5
```

Notice how in the class definition, `test` is defined twice - once with `get`
and once with `set`, and notice how they are defined as methods but are treated
exactly the same as normal properties when using the class.

## Using Getters and Setters to Watch Properties

So now how might we use this feature to watch a property? Think of what might
happen if we have a private property that the consumer cannot access directly,
and then a public getter and a public setter that write to that private property.
The getter could just return the property value, while the setter could
simultaneously set the property value and do anything else we desire.

In the following example, I've prefixed the private property (which the
class consumer never sees) with an underscore. It can't have the exact same name
as the getter and setter.
```ts
class Example {
  private _test: number = 1;

  public get test(): number {
    return this._test;
  }
  public set test(value: number) {
    this._test = value;
    // You could do anything you want here
    console.log(`Test property was set to ${value}.`);
  }
}

const ex = new Example();
console.log(ex.test); // logs: 1
ex.test = 45; // logs: "Test property was set to 45."
console.log(ex.test); // logs: 45
```

## Typing Getters and Setters
In the previous example, I typed all three properties with `number`. If we wanted
to change the type of `test` later on, we'd have to change the type of `_test`,
the return value of `get test()`, and the parameter type of `set test(value)`.
Instead of statically setting the same type three times, we can use TypeScript's
type accessor feature to automatically pull the type from `Example.test`:
```ts
class Example {
  private _test: number = 1;

  // `Example["_test"]` exacts exactly the same way as `number` because it
  // pulls the type from the `_test` property
  public get test(): Example["_test"] {
    return this._test;
  }
  public set test(value: Example["_test"]) {
    this._test = value;
    console.log(`Test property was set to ${value}.`);
  }
}
```

## Emitting Events

Finally let's take a quick look at Node's [`events`](https://nodejs.org/api/events.html) library so we can get
familiar with event emission and use that in this example. Of course, we could
use a simple callback function like `IShape.onChange()` but then anyone could
overwrite our event handling and break the API. Instead, the `events` library
allows us to emit a single event from a class and it can have as many listeners
as desired, so we could add a listener for our API _and_ the user could add a
listener for their purposes.

Here's how this would work in our `Example` class:
```ts
import {EventEmitter} from 'events';

class Example extends EventEmitter {
  private _test: Example["_test"] = 1;

  public get test() {
    return this._test;
  }
  public set test(value: Example["_test"]) {
    this._test = value;
    // Emit an event
    this.emit("change", value);
  }
}

const ex = new Example();

// Attach an event listener
ex.on("change", (newValue): void => {
  console.log(`Test property was set to ${newValue}.`)
});

ex.test = 45; // logs: "Test property was set to 45."
```

## Tying it All Together

Now that you are familiar with the concepts involved, it should be pretty easy
to see how we can combine all of this together to make our `Shape` class. Here
we go:

```ts
import {EventEmitter} from 'events';

interface IShape {
  color: string;
  opacity: number;
  visible: boolean;
}

class Shape extends EventEmitter implements IShape {
  // Save the event name as a public property so users can see it
  public static changeEvent = "change";

  private _color: string;
  private _opacity: number;
  // Defaults to visible on creation:
  private _visible: boolean = true;

  // Note how we can use the bracket notation in the constructor too:
  public constructor(color: Shape["_color"], opacity: Shape["_opacity"]) {
    // Call super to initiate the parent EventEmitter
    super();
    this._color = color;
    this._opacity = opacity;
  }

  public get color(): Shape["_color"] {
    return this._color;
  }
  public set color(value: Shape["_color"]) {
    this._color = value;
    this.emit(Shape.ChangeEvent);
  }

  public get opacity(): Shape["_opacity"] {
    return this._opacity;
  }
  public set opacity(value: Shape["_opacity"]) {
    this._opacity = value;
    this.emit(Shape.ChangeEvent);
  }

  public get visible(): Shape["_visible"] {
    return this._visible;
  }
  public set visible(value: Shape["_visible"]) {
    this._visible = value;
    this.emit(Shape.ChangeEvent);
  }
}
```

Let's try it out:
```ts
const exampleShape = new Shape("red", 0.75);
exampleShape.on(Shape.changeEvent, (): void => {
  // Put redrawing logic here
  console.log("Redrew the shape.");
});

exampleShape.visible = false; // Logs: "Redrew the shape."
exampleShape.visible = true; // Logs: "Redrew the shape."
exampleShape.color = "green"; // Logs: "Redrew the shape."
exampleShape.opacity = 1.0; // Logs: "Redrew the shape."
```

Looks great! Stay tuned for the next article, which will show you how you can
watch more complex properties like objects and arrays.
