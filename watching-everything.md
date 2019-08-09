# Watching *Everything* with TypeScript and Proxies
*8/9/2019*

I often encounter a situation where I want to watch changes to a class property
and take action when those properties change. For example, here's API of some
sort of basic drawing canvas that supports rendering rectangles:

```ts
interface IShape {
  coordinates: {
    x: number;
    y: number;
  };
  color: string;
  opacity: number;
  visible: boolean;
  // Exact dimensions structure depends on the shape
  dimensions: Record<string, number>;
  // Called when the shape should be redrawn. In a real app we'd use the Node.js
  // Events API instead of a function property.
  onUpdate: () => void;
}

class IRectangle extends IShape {
  dimensions: {
    height: number;
    width: number;
  };
}

class ICanvas {
  shapes: IShape[];
}
```

I want to design this drawing API so it's extremely intuitive to use. Notice how
the interfaces have no API methods for updating - there's no
`Rectangle.setDimensions()` or `Canvas.addShape()` methods here. Instead, I want
to build this API to 'automagically' update the drawing whenever a property
changes:

```ts
canvas.shapes[0] = rectA;
// => Canvas was redrawn.
```

If you want to skip the explanation, you can
[skip to code / demo.](#putting-it-all-together)

## Watching Primitive Properties

For primitive properties, like `Shape.color`, `Shape.opacity`, and
`Shape.visible`, this is a common pattern. If you've seen public getters and
setters paired with private properties before, you can probably
[skip this section](#watching-complex-properties). If not, here's how we'd do
this:

First, make a `Shape` class with private properties that hold the actual values.
For the purpose of this article, private properties and methods will be prefixed
with an underscore. Don't worry about the constructor; I'm intentionally leaving
it out for simplicity:

```ts
// Abstract because it will always be extended by a more specific shape
abstract class Shape implements IShape {
  private _color: string;
  private _opacity: number;
  private _visible: boolean;

  // Marking a property as abstract requires inheriting classes to implement it
  abstract dimensions: Record<string, number>;

  public onUpdate = function() {};
}
```

For the public-facing properties, classes sice ES6 come with these nifty little
features called getters and setters. By declaring `get` and `set` functions, we
can make it outwardly look as though these are normal properties, but inwardly
we can do whatever we want when they change.

Here's the syntax. Note how we never touch the private variable outside of the
class, and it doesn't look like we're calling a function at all:

```ts
class Example {
  private _exampleProp: string = "example";

  get exampleProp() {
    return this._exampleProp;
  }
  set exampleProp(newValue: string) {
    console.log(`Example.exampleProp changed to ${newValue}`);
    this._exampleProp = newValue;
  }
}

const ex = new Example();

ex.exampleProp = "different";
// => "Example.exampleProp changed to different"
console.log(ex.exampleProp);
// => "different"
```

And here's how we implement it in the `Shape` API. Notice the types in the
following example. They use bracket notation to automatically pull the type from
the private properties. This makes refactoring a breeze in the future as you
only have one type to change for each member. TypeScript exposes these as
`string`, `number`, and `boolean` respectively.

```ts
abstract class Shape implements IShape {
  private _color: string;
  private _opacity: number;
  private _visible: boolean;

  abstract dimensions: Record<string, number>;

  // Default to do nothing on callback
  public onUpdate: () => void = function() {};

  // You have to define a setter and a getter for each property individually
  public get color(): Shape["_color"] {
    return this._color;
  }
  public get opacity(): Shape["_opacity"] {
    return this._opacity;
  }
  public get visible(): Shape["_visible"] {
    return this._visible;
  }

  public set color(value: Shape["_color"]) {
    this._color = value;
    this.onUpdate();
  }
  public set opacity(value: Shape["_opacity"]) {
    this._opacity = value;
    this.onUpdate();
  }
  public set visible(value: Shape["_visible"]) {
    this._visible = value;
    this.onUpdate();
  }
}
```

OK, so far so good. But what about the more complex properties, like
`Shape.coordinates` or the array of shapes in `Canvas.shapes`? Updating an
object's property doesn't change the reference to that object, so updating
`Shape.coordinates.x` would never call a `Shape.coordinates` setter.

## Watching Complex Properties

To watch objects (rather than primitives), we have this other tool called
[proxies](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy).
Without going into too much detail, proxies allow you to intercept ('trap')
actions taken on an object, like `get`, `set`, and `delete`. This is done by
creating wrapper around the object, and then providing that proxy as the public
API instead of the object itself.

In our class, we can keep the actual `Shape.coordinates` object private, and
only provide the consumer with a proxy. Because proxies look just like normal
objects, the user will never know the difference. We make both the proxy and
private object `readonly` to prevent the actual object references from being
changed, which would break the watching behaviour.

TypeScript works great with proxies, as it disguises them as having exactly the
same type as their wrapped object. This makes sure API consumers don't have to
worry about whether they are interacting with a proxy or not.

```ts
abstract class Shape implements IShape {
  // Provide a standard method for updating an object's property and calling
  // onUpdate. This can be reused in multiple different proxies and accessed by
  // inheriting classes. The function should return true for successful sets.
  protected _setObjectProperty(
    object: any,
    prop: string | number | symbol,
    value: any
  ): boolean {
    object[prop] = value;
    if (object[prop] === value) {
      this.onUpdate();
      return true;
    }
    return false;
  }

  private readonly _coordinates: {
    x: number;
    y: number;
  };

  public readonly coordinates: Shape["_coordinates"] = new Proxy(
    this._coordinates,
    {
      // If you don't `bind` the function, it will try to call `onUpdate` on the
      // Proxy, not on the Shape
      set: this._setObjectProperty.bind(this)
      // Unlike before, you don't need a `get` function as Proxy automatically
      // passes untrapped calls through to the underlying object
    }
  );
}
```

Packaging the interception function into its own `protected` method makes it
easy to reuse for inherited classes like `Rectangle`:

```ts
class Rectangle extends Shape implements IRectangle {
  private readonly _dimensions: {
    height: number;
    width: number;
  };

  public readonly dimensions: Rectangle["_dimensions"] = new Proxy(
    this._dimensions,
    {
      set: super._setObjectProperty.bind(this)
    }
  );
}
```

Because an array is just an object with numeric property names and some other
functionality added on, this method works with arrays as well. There are some
caveats with typing, since TypeScript assumes you never want to set a
non-numeric property on an array, however some important internal behaviours of
JavaScript rely on being able to update properties like "length". Also, you need
to provide a hook for `deleteProperty`. We didn't do that before because we can
assume that the user will not try to `delete` a required (non-optional) property
of an object like `Shape.coordinates.x`, but some array methods use this
keyword.

In the `Canvas` class, however, we don't just want to redraw when
`Canvas.shapes` changes; we also want to redraw when each shape changes. To do
that we add an event listener in the intercepter:

```ts
class Canvas implements ICanvas {
  private readonly _shapes: Shape[];
  private _redraw = (): void => {};

  public readonly shapes: Canvas["_shapes"] = new Proxy(this._shapes, {
    set: (shapes, prop, shape: Shape): boolean => {
      // Have to keep in mind that an array also has some string properties
      // like "length" that are set by internal functions. TypeScript doesn't
      // like setting those but you have to handle that, so the type is `any`.
      (shapes as any)[prop] = shape;
      if ((shapes as any)[prop] === shape) {
        // typeof won't work if property is string like "1"
        if (!Number.isNaN(Number(prop))) {
          shape.onUpdate = this._redraw;
          this._redraw();
        }
        return true;
      }
      return false;
    },
    // Also must provide a delete trap, for example when array.pop() is called
    deleteProperty: (shapes, prop): boolean => {
      delete shapes[prop];
      if (!(prop in shapes)) {
        this._redraw();
        return true;
      }
      return false;
    }
  });
}
```

## Allowing Overwrites of Complex Properties

So far we've set `readonly` private properties and provided `readonly` proxies.
This works great for catching modification of those properties, however, what if
the consumer wants to overwrite them altogether? For example, what if you want
to do `Canvas.shapes = []` to clear the canvas after initialization, or you want
to set both coordinates at once with `Shape.coordinates = {x: 10, y: 12}`?

To do this, we have to combine getters, setters, and proxies into one harmonious
cooperative process. We'll keep the base private property the same, but we'll
also make the proxy private and only allow access with getters and setters. A
proxy doesn't provide a way to reassign its target, so we'll have to create a
new one every time the base property is changed.

In summary, we'll have:

- A base private member with an object/array type
- A private proxy member that intercepts property updates on the private object
  and performs a redraw
- A public getter that returns the proxy
- A public setter that simultaneously updates the private member, the private
  proxy, and performs a redraw

It's really not as bad as it sounds. Here's how it looks on `Canvas`:

```ts
class Canvas implements ICanvas {
  private _shapes: Shape[];
  private _shapesProxy: Canvas["_shapes"] = this._generateNewShapesProxy();
  private _redraw = () => {};

  // Put in its own function because we want to call it on construction and changes
  private _generateNewShapesProxy(): Canvas["_shapes"] {
    // This is the same proxy from before
    return new Proxy(this._shapes, {
      set: (shapes, prop, shape: Shape): boolean => {
        (shapes as any)[prop] = shape;
        if ((shapes as any)[prop] === shape) {
          if (!Number.isNaN(Number(prop))) {
            shape.onUpdate = this._redraw;
            this._redraw();
          }
          return true;
        }
        return false;
      },
      deleteProperty: (shapes, prop): boolean => {
        delete shapes[prop];
        if (!(prop in shapes)) {
          this._redraw();
          return true;
        }
        return false;
      }
    });
  }

  public get shapes(): Canvas["_shapes"] {
    return this._shapesProxy;
  }

  public set shapes(value: Canvas["_shapes"]) {
    this._shapes = value;
    this._shapesProxy = this._generateNewShapesProxy();
  }
}
```

## Putting it All Together

Putting it all together, here's what we get (note I've updated the `Shape` and
`Rectangle` classes to allow overwrites as described, added simple constructors
for each class, and added a log statement in `Canvas.redraw`).

To run the code, check it out on the
[TypeScript playground](http://www.typescriptlang.org/play/index.html?target=3#code/JYOwLgpgTgZghgYwgAgJIGUAWcAOKDeAsAFDLIID2FUAJqHJAM4BcyRpZyAHqyAK4BbAEbQA3CU7IAnr0Eio4jgF9FZSgBtqrRmCigA5quQUciYGBnJ+wsROQA3YI2BD1EVkKpu4II3QEQIM4UQawAShCUtAA8Onog+gA0VnLQAHxGIQCqODQM7sgAFACUyAC8aQ4UwDSKSiQkoJCwiCioEQhgPvpuyBBckCA0jGhYuAR2-oHBoWx2ZJgQwPqYYLI2CvPIAO41YJjr8kYqJPXEjeDQ8EhoAMI+9nAj7GSM2Hgso+8QANoAunUGsQ4EI4ogwOR1E8RmM8MhgAIcG4AuARhhvnMODg9I9IMgAPoaLTIOIGIzY4C4lD4kxmCyHWxYnH5AmOZyuAqeCjeEDlZC6PgQcnMvGEqi0ehMVgvTg8FIbIxkSzWI52E5MykssXUOggfKMAAKUAoXEssN+ACJtRK9UwLQCgWQQWDOsgpkFgCFPh0dbFdAZkir0oo7Dg+K5gAhjCAcnlIKxCiVypV7NUaKUysgYHwQJ1PSAk-h1XZKEEBZ1qIUtkSoKxzT8rTX7YktrSEOYzd8GzTTO2LM3q+Ldfq613G0PJRBGPa7KUZWR9k4AHRizRQPk1xX8zDLnt0qR8tsdreLxgryg6ycjTMXm36k87s-W4dMI0mg+Z08r-SBaD5AByEDbG+pqFF+z5XsUxyOsgYYRlGP4QjWJSjuM3ZNn8mKSFAEBgHwUC8uBm5qqG4bqJGyCIcYvYdihyD1laR79ph87IDheEEduu5MVI0FYmRFFUWyLhuHRDH4sJHL2lhnDsfhhGPiukluHxpHwSSuHkNylaPOogqoXg6HaVA9pzlsRHGXyumCg+y7ZLk+QlHxZBweRUaMJpPGFNZBTiTxpkyQuil7n2H4OHAelCuZin2XGEBOSR-HqR5ELKfFPkGZaElOCJEABax4FpVZEU2dFdkxg5kAJcoMHYhQkCdBANAEilADyQgAFaRGAb54FAFhVhwZAUJ13WsD4UgtkNsHGjg2j+gkyAAD7yvIy0klIwjclNkgZcgE2zh4XgQD4gXGKNnQ-HVOCYZmPlbsAMBFCNXWXddt1lHdJUQGZ01BeVsaOVBWxkHJnEClF01nNhuHyVmEUeapTL1d1TUEj+IB-pAgHAcaprRAAKn0AyBMM52vWAaRgXAUCIawBPFPTZ1g7ymPbMgIFSNTtO4ck+Apaw4FtRdPWzdAFhLkIoA0GBj7FEowM1ecSVuZRmm3i+U5iWOEG2lO0msSzXFPhrV6c0jLkCe56sTnrjDed9mVGZedv5WVJu2-qxWRbZHsu-qnN8uBGNYxAOOc7Lu6m3biuSF+sVA3xZwkAgUKMCMHRdAkvT9IMZPmvCiLIoEYBopn3S9DKFJUgS7ozJ8rGLMsqwMps027DQ+yt8cwqaqKdf5oaeOWOX2dZQPXr2iGHClnEfAVlAg2SDW4TdRXWUYTtnA8avnTr92-l-Fvaie1KyCjz0G+n-rR9bBPswX243b39OfyzmdjB8H1hQ1skPHJNHfUsdODgRfnyF+vsVwv0DpmT+fVvy-igABICEdQEImmIPYBycVaCU0i-Oij9x7oI9JPFiWwjZoJRPXc2iU1KqxSm6Yh9cHaRV3lnS+z8mGDzdtNShGCvTe1Krw4K0Dh58jgdABBmMkHYxQcPSOT4X7AP+meBOVUsGnCBCnNOIx7ggEeCMBESIIAolLncB4TwZLVy1G8cYnx6wOg1DXfEtiPic1YHogx3ZXE317s4nCNAkHs0zEmCoZ1Z7cggEuTQ+hCgWk8ZY7YliAlBJAEuC0WDp4nzLFAeeYBKw+M+Akxg3jviv1+pIAA9JU5AABBHASIDz7BQAgCK6ghCIAANb8goNuFAoBzDAAiiSMpWxClLhgNQAAoogTAiYfEZkqD4pcaiUCfmCikuA2xlHGxXIU8RZTIEuLKTA3ZIcZFhyAuaIe74SjuxXJs7Y1UyDQxmn3FA5zkHbGuRHRmyBimlLsQbchsNOJsw5vIoWZS+Ygw0msIohTkjXWSD4zKfyuQ8mTGdSQhR9mWImsUK6s1bojPGFuSQj15llP2iMAlRKTAfVgd8CpkhWWUoAIT-lSFAJcTh-xwH-IULlGxCjXWKOK7FrLXjfBWRVOKQcNlNSCeSqVuz8SPOeaql5sKYYcUIrkyGWrXlSqNvAdQiNYVKGPsgapdTzW9IEHwHQM0KCOBoCgOAboIBuDxLoXAyRJnrn6HAIuKBtiLF5DTJBUglw4BMEmJw5A2lNTvt63CEBeri0sLi6FLqcDouOqdMJrEyDup9SgHNdiaX7RAFIQl70VUctFbNeEvJCkSpLSAxVgStmapNaC-VQipXGtkgO+G5rDWcBHQrC2sErZqwhIUuiAKrSFOBdNChwVCk0KVnQiiDCl17RXccoFfwWUqL2dSr6Pt7knrcWI9Zy5PmyO+Sc+ROzbUABEQgAHIISBqovk-aNBmrNL6PYEuyByI6EQSMIDYHCnsq2D5CZ0zZlUvGIs0leBZWAzxI+p8jydngQ1Vg5AmjlazwhDhTotS+TgsIXEgJFpkgAEYAAM7G+Zyg41x6QrBeNWrYE3FY8KACsnHkgdy7sgAALJxmdKcvSROiRQWJFoqNJv0ZYzM4LimFB+DRsAtSz2iAyUpssWmDH0aAv8ix9tDPdRM4rW1YTik7GSUqrZaSgQRLcKp9TRnalLhrHyAAOhafQOFAgRbM8UEgQWQuWUzJF6LIALSKFc5UdzSSRiPJ88QPzUSYlMac0l-2TAlxcD5BJuLCWyuAMq9VzMtWSBZbs9pkYuW2Jee2AVizzqjMACEbPs0YxaVwgoWPIAABx8fwHKCTyRLAACYFN8xEy3ZAvGpN7AOMgNb7HFOFeU-5krGn7NLnGWGN4hRhtQXM4Vy712nVzPu5lmpbn7Meby71grRWAulc6ENlZNELB8lk2JurxBhug-3BDqHbXPvZe+91-LS4BsqfO60zrV2r3IH+NDnHBi8dVszITpHWKcueZ7X1jHJ2ghY7U3E4nTxScfD5I54HAJHus7PPs8nw3HHtep792n-3TvFeZxd3H1340Pfi092XZTY3y4+1T1HNPUn04B9j57ZSfjsZJUFon+u7GG+N059XX3Os-Z6+LnXkvAcy5J4Ui35W7yVbCpD03yvzdG495rM83vEfEBF5rsX2ugA).

```ts
interface IShape {
  coordinates: {
    x: number;
    y: number;
  };
  color: string;
  opacity: number;
  visible: boolean;
  dimensions: Record<string, number>;
  onUpdate: () => void;
}

interface IRectangle extends IShape {
  dimensions: {
    height: number;
    width: number;
  };
}

interface ICanvas {
  shapes: IShape[];
}

abstract class Shape implements IShape {
  private _color: string;
  private _opacity: number;
  private _visible: boolean = true;
  private _coordinates: {
    x: number;
    y: number;
  };
  private _coordinatesProxy: Shape["_coordinates"];

  abstract dimensions: Record<string, number>;

  public onUpdate: () => void = function() {};

  constructor(
    color: Shape["_color"],
    opacity: Shape["_opacity"],
    coordinates: Shape["_coordinates"]
  ) {
    this._color = color;
    this._opacity = opacity;
    this._coordinates = coordinates;
    this._coordinatesProxy = this._generateNewProxy(this._coordinates);
  }

  public get color(): Shape["_color"] {
    return this._color;
  }
  public get opacity(): Shape["_opacity"] {
    return this._opacity;
  }
  public get visible(): Shape["_visible"] {
    return this._visible;
  }
  public get coordinates(): Shape["_coordinates"] {
    return this._coordinatesProxy;
  }

  public set color(value: Shape["_color"]) {
    this._color = value;
    this.onUpdate();
  }
  public set opacity(value: Shape["_opacity"]) {
    this._opacity = value;
    this.onUpdate();
  }
  public set visible(value: Shape["_visible"]) {
    this._visible = value;
    this.onUpdate();
  }
  public set coordinates(value: Shape["_coordinates"]) {
    this._coordinates = value;
    this._coordinatesProxy = this._generateNewProxy(this._coordinates);
    this.onUpdate();
  }

  protected _setObjectProperty(
    object: any,
    prop: string | number | symbol,
    value: any
  ): boolean {
    object[prop] = value;
    if (object[prop] === value) {
      this.onUpdate();
      return true;
    }
    return false;
  }

  protected _generateNewProxy<T extends object>(target: T): T {
    return new Proxy(target, { set: this._setObjectProperty.bind(this) });
  }

}

class Rectangle extends Shape implements IRectangle {
  private _dimensions: {
    height: number;
    width: number;
  };
  private _dimensionsProxy: Rectangle["_dimensions"];

  constructor(
    color: Rectangle["_color"],
    opacity: Rectangle["_opacity"],
    coordinates: Rectangle["_coordinates"],
    dimensions: Rectangle["_dimensions"]
  ) {
    super(color, opacity, coordinates);
    this._dimensions = dimensions;
    this._dimensionsProxy = super._generateNewProxy(this._dimensions);
  }

  public get dimensions(): Rectangle["_dimensions"] {
    return this._dimensionsProxy;
  }

  public set dimensions(value: Rectangle["_dimensions"]) {
    this._dimensions = value;
    this._dimensionsProxy = super._generateNewProxy(this._dimensions);
    this.onUpdate();
  }
}

class Canvas implements ICanvas {
  private _shapes: Shape[];
  private _shapesProxy: Canvas["_shapes"];
  private _redraw = () => {
    console.log("Canvas was redrawn.");
  };

  constructor(shapes: Canvas["_shapes"]) {
    // Apply the callback to the initial shapes
    shapes.forEach(shape => (shape.onUpdate = this._redraw));
    this._shapes = shapes;
    this._shapesProxy = this.generateNewShapesProxy();
    this._redraw();
  }
  private generateNewShapesProxy(): Canvas["_shapes"] {
    return new Proxy(this._shapes, {
      set: (shapes, prop, shape: Shape): boolean => {
        (shapes as any)[prop] = shape;
        if ((shapes as any)[prop] === shape) {
          if (!Number.isNaN(Number(prop))) {
            shape.onUpdate = this._redraw;
            this._redraw();
          }
          return true;
        }
        return false;
      },
      // Also must provide a delete trap, for example when array.pop() is called
      deleteProperty: (shapes, prop): boolean => {
        delete (shapes as any)[prop];
        if (!(prop in shapes)) {
          this._redraw();
          return true;
        }
        return false;
      }
    });
  }

  public get shapes(): Canvas["_shapes"] {
    return this._shapesProxy;
  }

  public set shapes(value: Canvas["_shapes"]) {
    this._shapes = value;
    this._shapesProxy = this.generateNewShapesProxy();
    // Don't forget to add the event listeners to the shapes!
    value.forEach(shape => (shape.onUpdate = this._redraw));
    this._redraw();
  }
}
```

## Demo

Here's how we can use the above classes. Notice how the API consumer treats the
class properties just like normal properties. All the magic happens behind
scenes.

```ts
const rectA = new Rectangle(
  "red",
  100,
  { x: 100, y: 100 },
  { height: 500, width: 400 }
);
const canvas = new Canvas([rectA]);
// => Canvas was redrawn.

rectA.color = "green";
// => Canvas was redrawn.
rectA.coordinates.x = 50;
// => Canvas was redrawn.

const rectB = new Rectangle(
  "blue",
  80,
  { x: 50, y: 200 },
  { height: 100, width: 200 }
);
canvas.shapes.push(rectB);
// => Canvas was redrawn.
rectB.opacity = 45;
// => Canvas was redrawn.
canvas.shapes = [];
// => Canvas was redrawn.
canvas.shapes = [rectB];
// => Canvas was redrawn.
canvas.shapes.pop();
// => Canvas was redrawn.
canvas.shapes[0] = rectA;
// => Canvas was redrawn.
canvas.shapes[0].coordinates.y = 45;
// => Canvas was redrawn.
```
