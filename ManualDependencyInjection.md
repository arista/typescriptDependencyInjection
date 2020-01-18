# TypeScript Patterns for Manual Dependency Injection

Dependency Injection is a technique in which everything that a "component" (a class instance) needs to do its work is explicitly passed to it, often in its constructor.  Components following this technique avoid accessing global functions or variables, or static members of other classes.  Components may go so far as to avoid using "new" to directly instantiate instances of other clases, and may even avoid referencing other classes altogether.  The benefit of applying this technique is that the resulting components are simpler, decoupled, more flexible, and easier to unit test.

Of course, some part of the application does have to worry about creating the dependencies and supplying them to the components, "injecting" those dependencies as needed.  This "injection" could be coded manually, but in a language like Java, where Dependency Injection was first popularized, doing so is onerous and burdensome.  So this functionality was typically offloaded to frameworks like ATG's Nucleus, or the Spring Framework.  These frameworks provide a lot of power and functionality, but also require a substantial commitment since they are effectively platforms with their own configuration formats, component libraries, idioms, etc.

However, the benefits of Dependency Injection don't require any particular framework or language.  An application written in JavaScript, for example, can benefit from applying these techniques.  And with some of the features provided by ES6, manually coding the injection is much less burdensome, which in many cases eliminates the need for a separate framework.  Using TypeScript further increases the benefits of this technique, since the compiler can verify that dependencies are being fulfilled properly.

The rest of this article describes a set of coding patterns for developing a TypeScript application using dependency injection techniques without the use of a framework.  In theory, these techniques can be applied to applications of nearly any complexity, from a simple command-line tool to a full back-end service.

Note that there is quite a bit of boilerplate involved, which is the main detriment to using these patterns.  Developers will need to decide if the benefits justify the cost.

# Pattern for Classes With Dependencies

The following set of patterns are used by classes that wish to declare their dependencies, and demonstrate how those dependencies can be passed in.  They also demonstrate how the class can indicate which dependencies it expects to be passed in by users and which it expects to be "injected".

## Basic boilerplate:

```
export namespace Class1 {
  export interface Props {
    // Properties expected to be supplied by code that wants a new instance of Class1
    prop1:string
    optionalProp1:string|null
  }
  export interface Injected {
    // Properties expected to be supplied by "injection"
    service1:Service1
  }
  export type Context = Props & Injected
  export type Factory = (props:Props)=>Class1
}

export class Class1 {
  ctx: Class1.Context
  constructor(ctx:Class1.Context) {
    this.ctx = ctx
  }

  // Create getters to extract values from the Context for convenience
  get prop1() {return this.ctx.prop1}
  get optionalProp1() {return this.ctx.optionalProp1}
  get service1() {return this.ctx.service1}
  ...
}
```

* All of the class's dependencies are gathered into a single "Context" object that is passed to the constructor which stores it as a member variable.  Dependencies include parameters, configuration, pointers to other services, event handlers, etc.
* Classes take their best guess as to which dependencies will typically be provided by a user of that class, and which will be injected.  The former is gathered into a `Props` interface, the latter into `Injected`, and the two are combined into `Context`.
* A `Factory` declaration formalizes the idea that a user of the class need only supply the Props in order to obtain a new instance of the class.
* Getter methods expose each of the Context's properties.  This is both for convenience (to avoid having to type "this.ctx..." everywhere), and to keep the rest of the class from having to be aware of this Context pattern.
* The class's executable code does not reference any external variable or class name.  This means no global variables or functions, no referencing static members of other classes, no using "new" to instantiate an external class name, and no using "instanceof" directly against the name of another class.  In general, the class's code does not reference any external variable or class name.  All such external functionality is provided through the Context.
* On the other hand, type declarations within the class are free to reference external classes and types.
* Creating a `namespace` with the same name as the class provides others with convenient access to the type declarations.  A user of the class that does an `import {Class1} from "..."` can access not only `Class1`, but also type declarations `Class1.Factory`, `Class1.Props`, etc.

## Pattern for using Factories

If a class wants to create an instance of another class, it declares a Factory as a dependency (typically an injected dependency) and uses that Factory to create the instance.  It avoids using `new` to create the instance directly.

```
export namespace Class1 {
  ...
  export interface Injected {
    class2Factory:Class2.Factory
  }
  ...
}

export class Class1 {
  ...
  get class2Factory() {return this.ctx.class2Factory}
  ...
  function f() {
    const newInstance = this.class2Factory({
      prop1:10,
      prop2:20,
    })
    ...
  }
}
```
As described above, Class2.Factory takes in the Props that are meant to be provided by users of Class2.  If Class2 has other `Injected` dependencies, Class1 doesn't need to know about them.  Class1 can assume that whoever provided the Class2.Factory will take care of any injected dependencies.

## Pattern for emitting events

If a Class wishes to emit events without direct coupling to a receiver, it can declare a listener for the events as a dependency, typically under `Props`.  The event types can be declared within the same namespace.

```
export namespace Class1 {
  export interface Props {
    ...
    onEvent:(e:Event)=>void
  }

  export type Event =
    Event.MessageReceived |
    Event.Closed

  export namespace Event {
    export type Message = {
      type: "Class1.Event.MessageReceived",
      subject:string
      message:string
    }
    export type Closed = {
      type: "Class1.Event.ChannelClosed",
    }
  }
}

export class Class1 {
  ...
  get onEvent() {return this.ctx.onEvent}
  ...
  function f() {
    this.onEvent({
      type: "Class1.Event.MessageReceived",
      subject: "hello",
      message: "a message",
    })
  }
}
```
TypeScript works well to discriminate Event types based on a string field (like `type`), so there is no need to declare a separate enum or set of constants to represent the Event type indicators.

Declaring a single event listener dependency is typically sufficient.  If the application requires more dynamic or flexible event routing, that can be handled outside of the class by providing the class with a listener that handles that flexibility.

## Pattern for subclassing

A subclass will typically extend the Props and Injected interfaces of its superclass.  Aside from that, subclassing works as usual.

```
export namespace Subclass1 {
  export interface Props extends Superclass1.Props {
    ...
  }
  export interface Injected extends Superclass1.Injected {
    ...
  }
  export type Context = Props & Injected
  export type Factory = (props:Props)=>Class1
}

export class Subclass1 extends Superclass1 {
  ctx: Subclass1.Context
  constructor(ctx:Subclass1.Context) {
    super(ctx)
    this.ctx = ctx
  }
}
```

## Pattern for coding to interfaces

Some development practices advocate for classes that reference interfaces for their dependencies, rather than concrete implementations.  For example, rather than Class1 depending directly on Service1, it should instead depend on an `IService1` interface (for example) that is implemented by Service1.  This prevents Class1 from accessing parts of Service1 that were not intended to be exposed, and also allows alternate implementations of Service1 to be provided to Class1.

To use this approach classes should declare dependencies on the interfaces:

```
export namespace Class1 {
  ...
  export interface Injected {
    // Properties expected to be supplied by "injection"
    service1:IService1
  }
  ...
}
```

If the interface is expected to be instantiated multiple times, as opposed to acting as a singleton, then where the interface is defined it should also define a `Factory` and `Props`:

```
export namespace IService1 {
  export interface Props {
    ...
  }
  export type Factory = (props:Props)=>IService1
}

export interface IService1 {
  function1(...)
  function2(...)
}
```

The actual implementation of the interface should then use those props:

```
export namespace Service1 {
  export interface Injected {
    ...
  }
  export type Context = IService1.Props & Injected
  export type Factory = (props:Props)=>Service1
}

export class Service1 {
  ctx: Service1.Context
  constructor(ctx:Service1.Context) {
}
```

# Patterns for Injectors

The injector is the class responsible for satisfying the `Injected` dependencies declared by the classes.  It does so by implementing all of the `Factory` interfaces, supplying the injected properties through, for example, configuration values, pointers to other factories, or pointers to singleton "service" instances.  Because the injector needs to have access to all of the values it will be injecting, the injector often doubles as the "top-level" class of an application, creating and maintaining all of the "singleton" services for the application, many of which themselves require injected values.

While many applications will have a single top-level injector, more complex applications could have multiple injectors managing different areas of the application, with a single top-level injector coordinating those individual injectors.

The point of "Manual Dependency Injection" is for these injector classes to be written in code, without the use of a framework.  These injectors need not be complex, and can themselves be coded to some of the same patterns described before (declaring a Context, Props, and Injected) so as to be included in a larger context.

Manual coding does require the use of "boilerplate" code, and the amount of such code will determine how robust the coding pattern is.  For example, a "quick and dirty" injector can get multiple application components up and running quickly, while a more robust injector can handle mutual dependencies (aka "circular" dependencies), injector subclassing, factory substitution, etc.

## A "Quick and Dirty" Injector

The fastest path to an injector is to create all of the singletons and factories in the constructor, satisfying injected dependencies as needed.  Consider this scenario:

* `Service1` will have a singleton instance with a single injected numerical property `param1`
* `Service2` is another singleton instance with several injected properties:
  * service1 - a pointer to an instance of `Service1`
  * handlerFactory - a pointer to a `Handler.Factory` instance that it will use to create instances of `Handler`
  * onEvent - an event handler for messages that it will be emitting
* `Handler` will have multiple instances, each with an injected `service1` property pointing to an instance of `Service1`.  A `Handler.Factory` must therefore be defined, which will inject the `service1` property in addition to whatever `Handler.Props` it defines.

Here is what that might look like:

```
export class Injector1 {
  constructor() {
    const service1 = new Service1({
      param1: 10
    })

    const handlerFactory = (props:Handler.Props)=>return new Handler({
      ...props,
      service1: service1
    })

    const service2 = new Service2({
      handlerFactory: handlerFactory
      onEvent: (e)=>service1.handleMessage(e.message)
    })
  }
}
```
That's the minimum needed to get the job done.  And even this minimal approach has significant benefits.  The services are still written without direct knowledge of each other or the injector, and if the dependencies change, TypeScript will let you know what needs to be filled in.

### Handling Mutual Dependencies

The problem with the simple approach described above is that it does not handle mutual dependencies (aka "circular dependencies").  Two services might depend on each other, in which case the above approach will not work.  Consider what will happen if `Service1` requires a pointer to a `Service2`, and vice versa:

```
    const service1 = new Service1({
      ...
      // FAIL - `service` doesn't point to anything yet
      service2: service2
    })
    ...
    const service2 = new Service2({
      ...
      service1: service1
    })
```
Mutual dependencies are common and acceptable, so the injector needs to be able to handle them.  Using es6's "get" syntax can address this:

```
    const service1 = new Service1({
      get service2() {return service2}
    })
```
Instead of defining service2 as a concrete property value, it is now defined as a getter function placed directly on the object.  This means that `service2` doesn't have to exist until that getter is called.  Service1, meanwhile, is none the wiser.  It can still call `this.ctx.service2` and not realize that it is invoking a getter rather than getting an previously-determined value.  Now both services effectively have pointers to each other, satisfying the mutual dependency.

This approach will fail if `Service1` and `Service2` reference each other in their constructors.  In fact, no approach can handle that situation - the components will have to be restructured in such a case.  To avoid this situation, components may want to avoid doing significant work in their constructors, instead presenting a `start()` or `initialize()` method that kicks off the component's function.

### Caveats to the get() Approach

There are a couple caveats to using get():

* If a component removes one of its dependencies, TypeScript will warn you about all the places where you are still setting it, which lets you go back and clean up that code.  However, TypeScript will not do that for the `get()` syntax, so you'll lose that benefit.

* Within the body of a `get()` function (e.g., `{return service2}`), the `this` property no longer points to the `Injector1` instance.  It instead points to the object containing the `get()`, which is not particularly useful.  This means that the body of the `get()` cannot use `this` to access properties of `Injector1`.  While this isn't an issue for the example above, but it will be in the upcoming sections.

The second point is an unfortunate consequence of the `get()` syntax, and getting around it requires going back to pre-ES6 techniques for binding expressions to `this`.  For example, this code will not get `currentCount` from `Injector1`.  In fact, it will result in a circular function call:

```
    const handlerFactory = (props:Handler.Props)=>return new Handler({
      ...
      get currentCount() {return this.currentCount}
    })
```
Instead of referencing `this`, the expression will need to bind `this` to some other variable, like this:
```
    const handlerFactory = (i=>(props:Handler.Props)=>return new Handler({
      ...
      get currentCount() {return i.currentCount}
    }))(this)
```

### Drawbacks to the "Quick and Dirty" Approach

The simplistic approach can get quite far, and in many cases it might be enough.  It does, however, have some drawbacks that could cause problems as an application grows more complex:

* The order in which singletons are created is significant, and hand-coding that order can be fragile in an application with many singletons and factories.
* All of the singletons and factories are variables only visible within the constructor.  This makes it more difficult to use the injector in multiple contexts.  For example, if multiple applications want to make use of the same basic set of components, they will have difficulty subclassing or composing injectors written this way.
* Users of the injector are not able to substitute in their own classes for the services and factories.  For example, it would be difficult to substitute in a "mock" version of `Service1` for testing purposes, or to create a customized subclass of `Service1`.

## A Robust Injector Pattern

The following describes a coding pattern for building robust injectors that can scale to handle complex applications, or themselves be composed into larger applications.  The pattern has these features:

* All singletons are created "lazily", so components and factories can be declared in any order
* All properties are injected using the `get()` syntax, handling mutual dependencies
* All factories and singletons are exposed as member variables, allowing the injector to be extended through subclassing, or composed into larger components
* All factories can be customized, allowing "mocks" to be sent in for testing, or for customized components to be used in place of those speciifed by the injector
* None of this requires any changes to the underlying components - they are still authored to be decoupled from each other and the injector.

The main drawback of the pattern is that it involves a significant amount of boilerplate code.

The pattern follows this general approach:

* Provide an implementation for *every* `Factory` interface
* Allow substitute `Factory` instances to be passed in to the Injector that override the `Factory` implementations
* Define "lazy" singleton constructors that use the Factories

### Step 1: Define a Context for the Injector

Like all other classes, the Injector should be set up with Props, Injected, etc.:

```
export namespace Injector1 {
  export interface Props {
  }
  export interface Injected {
  }
  export type Context = Props & Injected
  export type Factory = (props:Props)=>Injector1
}

export class Injector1 {
  ctx: Injector1.Context
  constructor(ctx: Injector1.Context) {
    this.ctx = ctx
  }
}
```

This provides a standard way for the injector to declare configuration parameters or services that it needs.

### Step 2: Allow Factory "overrides" to be passed to the Injector

Declare the Factories as optional properties in `Props`:

```
  export interface Props {
    ...
    service1Factory?: Service1.Factory
    handlerFactory?: Handler.factory
    service2Factory?: Service2.Factory
  }
}
```
These properties will be used in later steps, and will allow users of `Injector1` to effectively substitute in their own implementations of those classes.

### Step 3: Implement all Factories

Create implementations of all the `Factory` classes, even those that will only be used to create singletons.  These implementations are placed outside of the constructor, and can appear in any order.

All of the Factory implementations follow this pattern:

```
  // Handler
  get handlerFactory() {return this.ctx.handlerFactory || this._handlerFactory}
  private _handlerFactory = (i=>(props:Handler.Props)=>new Handler({
    ...props,
    // injected properties
    get service1() {return i.service1}
  }))(this)
```

* The `get handlerFactory()` line is where we provide the Factory override if it was supplied to the injector.  Otherwise we return the factory that we're about to create.
* The `private _handlerFactory` line is where we actually create the Factory.  The `(i=>...)(this)` syntax is used to address the issue with `get()` properties described above.
* The Factory itself is a function that takes in `props` and returns a new instance of the implementation class (`Handler` in this case).  The class is passed a single object, which contains a combination of the supplied props (`...props`) and the injected properties.
* Each injected property is specified using the `get()` syntax.  References to other singletons, factories, or values in the injector go through `i.` (instead of `this.`).
* The injected properties are free to reference factories or singletons that haven't yet been defined.  The "lazy" `get()` approach insulates us from ordering issues.  They can even reference abstract values that will be supplied by subclasses.

### Step 4: Declare Singletons

For each instance that is expected to be a singleton, create a declaration for that singleton.  These are also placed outside of the constructor, and can appear in any order.

```
  // service1
  get service1() {return this._service1.get()}
  private _service1:Lazy<Service1> = new Lazy(()=>this.service1Factory({
    // props
    param1: 10
  }))
```
Here we presume the existence of a `Lazy` class, which encapsulates the "get-or-create" functionality for lazy creation.  It is passed a function used to create the instance if it hasn't already been created, and calling `get()` on it will get or create the value.  A sample `Lazy` implementation is supplied later, that also has the ability to detect unresolvable circular dependencies (such as classes referencing each other in their constructors).

Note that the pattern doesn't depend on a `Lazy` class, but it does require some sort of "get-or-create" functionality be supplied.

The singleton should be created using the Factory instance defined earlier, passing in any required `props` (but not injected properties).  This is often where event handlers are used to "wire together" decoupled components.

Because all of this relies on lazy evaluation, these singleton declarations can appear in any order, intermixed or separate from the Factory implementations.

### Step 5: Bootstrap the Singletons

At this point, the injector has declared all of its factories and singletons, and is prepared to lazily instantiate any of those objects and automatically create their dependencies.  However, the constructor is still empty and nothing is actually being instantiated yet.

If this really is the top-level class for an application, then something needs to at least reference the singletons that will perform the application's functions.  Just referencing a singleton will be enough to instantiate it and all of its dependencies.  For example:

```
  constructor(ctx: Injector1.Context) {
    this.ctx = ctx

    this.service2
  }
```

Note, however, that the injector doesn't necessarily have to be the one to instantiate the classes.  In fact, it may be preferable to let the injector simply define the factories and singletons, and let a separate top-level application class decide which singletons it wants to use.  There is no penalty if the injector defines services and factories that don't get used, so an injector could be designed to expose a superset of services used by any single application, thereby allowing it to be applied in multiple contexts.

## A Sample Lazy implementation

The following is a sample implementation of the `Lazy` class described earlier.  It has "get-or-create" functionality, and can detect circular initialization.

```
export namespace Lazy {
  export type Factory<T> = ()=>T
}

export class Lazy<T> {
  factory:Lazy.Factory<T>
  value:T|null = null
  valueInitialized = false
  valueInitializing = false
  
  constructor(factory:Lazy.Factory<T>) {
    this.factory = factory
  }

  get():T {
    // If a value hasn't yet been created, use the specified factory
    // to create and return it
    if (!this.valueInitialized) {
      // Use the valueInitializing flag to detect circular
      // initialization
      if (this.valueInitializing) {
        throw new Error(`Circular dependency while initializing`)
      }
      this.valueInitializing = true
      try {
        this.value = this.factory()
        this.valueInitialized = true
        return this.value
      }
      finally {
        this.valueInitializing = false
      }
    }
    // This should never happen, but TypeScript must be satisfied
    else if (this.value == null) {
      throw new Error(`Assertion failed: value should not be null`)
    }
    else {
      return this.value
    }
  }
}
```
