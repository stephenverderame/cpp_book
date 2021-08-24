# Design Patterns (Quickly)

Most of these come from Gang of Four's *Design Patterns: Elements of Reusable Object-Oriented Software*.
I will pretty much exclusively talk about OOP implementations here, but *Modern C++ Design* does a great
job going over some GoF patterns using a generic programming paradigm.

Feel free to do some more searching online; I'm not kidding when I put "quickly" in the title.

### Factory Method

In its simplest form, a factory method provides indirection between creation of an object and its usage.
Construction of an object cannot be done polymorphically. So if a client were to directly construct
an object, it would have to know about the concrete implementation since you cannot create an instance of
an interface.

```C++
std::unique_ptr<Client> makeClient() {
    return std::make_unique<HttpClient>();
}
```

Instead of the client directly constructing an `HttpClient`, it just knows its using some client
by calling `makeClient()`. What if we made a new HttpClient implementation called  `HttpClient2`
which has support for Http 2, and deprecated the old `HttpClient`? Well, with the factory method, our client
would not care or know about this change. Since they only get a client from the `makeClient()`
function, they don't have to change a single line of their code!

We can expand on the factory function by passing it arguments to control what
concrete implementation it creates and how it creates it.

```C++
enum class Clients {
    Http1,
    Http2,
    Websocket,
}
std::unique_ptr<Client> makeClient(Clients clientType) {
    switch (clientType) {
        case Clients::Http1:
            return std::make_unique<HttpClient>();
        case Clients::Http2:
            return std::make_unique<HttpClient2>();
        case Clients::Websocket:
            return std::make_unique<WsClient>();
        default:
            throw std::invalid_argument("Unknown client type");
    }
}
```

Typically, a factory method would be implemented in a source file so that the client would not depend
upon the headers for the concrete implementations.

### Abstract Factory

An abstract factory is similar to a factory method. But, in this case we create a hierarchy of factory objects.
Each factory object contains a virtual factory method. This allows us to polymorphically
pass a factory to a function or constructor, and let the receiving object handle the construction.

```C++
struct ClientFactory {
    virtual ~ClientFactory() = default;
    virtual std::unique_ptr<Client> operator()() const = 0;
};

struct HttpClientFactory : public ClientFactory {
    std::unique_ptr<Client> operator()() const override {
        return std::make_unique<HttpClient>();
    }
};

struct Http2ClientFactory : public ClientFactory {
    std::unique_ptr<Client> operator()() const override {
        return std::make_unique<HttpClient2>();
    }
};

struct WebsocketClientFactory : public ClientFactory {
    std::unique_ptr<Client> operator()() const override {
        return std::make_unique<WsClient>();
    }
};

void foo(const ClientFactory & maker) {
    auto client = maker();
    // ...
}
```

Typically, the implementation of the factory method would be in a source file.

A generic programming approach to this could be to simply pass a factory function as a callable object
instead of creating a class hierarchy. The downside with this is that a user could pass any
callable object with the correct signature, there is no guarantee that it's an actual factory function.

### Facade

Imagine that you're creating a computer vision algorithm to get text from an image. You might decide that your approach will be to
first filter the image, then apply the Hough Transform to detect lines in the image, pick out characters using those lines,
and finally use a neural network to recognize the character. To promote code reuse and decoupling, you decide to separate each step into
its own class to allow users to use each operation on its own and avoid possible issues where changes in one module affect another, unrelated module.

Now however, the user must manually execute each step on their own. We can create a Facade that provides a simplified interface.
The goal of a facade is to allow us to create modules with narrow scope, without sacrificing the simplicity of having a giant module with
an end-to-end interface.

```C++
class HoughTransform {
    // ..
};

class ImageFilter {
    // ...
};

class NeuralNetwork {
    // ...
}; 

struct OCRFacade {
    std::string textFromImage(const cv::Mat& img) const {
        // ...
        // Put all the individual modules together
    }
};
```

A facade doesn't need to simplify usage of multiple modules, it can just be used to simplify a single module's interface as well.

### Decorator

A decorator adds functionality to a polymorphic object by wrapping itself around the object. The decorator itself also implements the same
interface as the object it wrapped. To a client, they have no idea if an object is decorated or not since they would interact with the concrete class
and decorated class through the same abstraction.

```C++
class Port {
public:
    virtul ~Port() = default;
    virtual void write(const std::vector<char> & data)  = 0;
    virtual std::vector<char> read() = 0;
};

class GzipDecorator : public Port {
    std::unique_ptr<Port> port
public:
    explicit GzipDecorator(std::unique_ptr<Port> && port) : port(std::move(port)) {}

    void write(const std::vector<char> & data) {
        auto newData = gzipCompress(data);
        port->write(newData);
    }

    std::vector<char> read() {
        auto data = port->read();
        return gzipDecompress(data);
    }    

};
```

If you used Java, you may be familiar with the `BufferedWriter` class.
That is a decorator that adds a buffer to a Writer.
It is also a writer itself.

Decorators can be used with factories, so the client doesn't have to know
how the abstraction is decorated.

### Composite + Flyweight

A composite is like a decorator, but instead of adding more functionality, it adds more objects.

Consider you are developing a game and have a class `GameObject` which handles rendering,
and collision with objects in the game. This game might have multiple levels which you handle in the 
`Level` class. Now a level, is essentially a giant game object. It must be rendered, and colliding with a level
would essentially mean colliding with a `GameObject` in that level. Thus, `Level` can be a composite of `GameObjects`
and implement the same interface as `GameObject`.

Let's go back to our `GameObject`. Let's say each one has its own 3D model. Now what if want to have a bunch of the same type of object
(say the same type of House) in the game? Well, naively, we can construct multiple instances of `GameObject`, but then we'd get multiple copies
of the 3D model even though all the houses have the same one. This is where Flyweight comes in. Instead of having one instance of the class per game object,
we can have one instance of `GameObject` for each type of game object. The `GameObject` class can keep track of all the locations it should be,
and simply render itself and do collision detection on itself multiple times when those methods are called.

### Observer + Mediator

Consider you have a UI class that needs to notify the program logic of events on the UI.
You could have the UI directly call methods in different classes that need to be notified of events,
but what if you create a new module that needs to listen for UI events? With the current approach,
the UI will need to be changed.

Instead, let's call the UI a Subject, and the modules that need to notified of changes in the Subject, Observers.

```C++
class Observer {
public:
    virtual ~Observer() = default;
    virtual void notify(const Event & e) = 0;
};

class Subject {
    std::vector<std::reference_wrapper<Observer>> observers;
public:
    virtual ~Subject() = default;
    
    /// Requires `o` outlive this subject
    void add_listener(Observer & o) {
        observers.emplace_back(std::ref(o));
    }

protected:
    void notify_all(const Event & e) {
        for(auto o : observers) {
            o.get().notify(e);
        }
    }
};

class UI : public Subject {}
class Listerner1 : public Observer {}
class Listernr2 : public Observer {}

UI ui;
Listener1 one;
Listener2 two;
ui.add_listener(one);
ui.add_listener(two);
```

The `Event` object is simply an object to encapsulate information about what event was raised.
We can expand on this structure by providing a two-way channel so that an Observer could tell the Subject when it's being deleted so
that an Observer need not outlive the subject.

The Mediator pattern is a special case of the Observer patten when a Subject ony supports one observer listening for events. This observer
is called the mediator.

An expansion on this structure is publisher/subscriber architecture. In this architecture, the subject would be the publisher,
and observers would be the subscribers. Publisher/subscriber architecture typically allows multiple publishers.
Events are typically published on a message channel, which is an intermediary module, process, or program that sends the messages
to the subscribers. Publisher/subscriber architecture is basically the Observer pattern but along a stronger boundary.
While Observer is typically implemented to have different modules communicate, publisher/subscriber architecture typically
facilitates communication of components that are different processes, threads, programs, or even on different machines.
ROS nodes use this structure.

### Command

The command pattern is basically a callable object. In our previous Observer example, we had an `Event` object which encapsulated information
about the event raised. Let's say that, instead of passing state, we passed instructions (code) on the action to perform. This is basically the
Command pattern.

### State

State is a pattern to help implement FSM (finite state machines) by encapsulating each FSM node with an object.
Each FSM state object would implement an abstract state interface. Behavior is then controlled by the FSM
delegating actions to the abstract state interface. The concrete class receiving these delegated function calls
changes depending on the current state of the FSM. State transitions in the FSM would be implemented by changing the concrete
state implementation behind the interface.

Take a Window class for example. A window can be open, minimized, maximized, and closed.
Depending on the current window state, behavior of the minimize, maximize, and close buttons changes.
Each of these three can be a concrete window state class.
The state class will handle behavior of the operations, as well as state transitions to other states.

```C++
class Window {
    class WindowState {
    public:
        virtual ~WindowState() = default;
        virtual void maximize() = 0;
        virtual void minimize() = 0;
        virtual void close() = 0;
    protected:
        Window * context;

        explicit WindowState(Window * wind) : context(wind) {}

        void transition(std::unique_ptr<WindowState> && newState) {
            context->state = std::move(newState);
        }
    };
    
    std::unique_ptr<WindowState> state;
    
public:
    Window() {
        state = std::make_unique<MaxWindowState>(this);
    }

    void maximize() {
        state->maximize();
    } 

    void minimize() {
        state->minimize();
    } 

    void close() {
        state->close();
    } 
};

class MaxWindowState : public WindowState {
public:
    explicit MaxWindowState(Window * ctx) : WindowState(ctx) {}

    void maximize() override {}
    void minimize() override {
        context->// minimize
        transition(std::make_unique<MinWindowState>(context));
    }
    void close() override {
        // ...
        transition(std::make_unique<CloseWindowState>(context));
    }
};

class MinWindowState : public WindowState {
  public:
      explicit MinWindowState(Window * ctx) : WindowState(ctx) {}
  
      void maximize() override {
          // ...
          transition(std::make_unique<MaxWindowState>(context));
      }
      void minimize() override {}
      void close() override {}
  };

class CloseWindowState : public WindowState {
public:
    explicit CloseWindowState(Window * ctx) : WindowState(ctx) {}

    void maximize() override {}
    void minimize() override {}
    void close() override {}
};
```

A naive approach to implementing a state machine could be to use an `enum` to represent the current state,
and use switch statements to control behavior based on the current state. This can work, however if, like above,
the state transition depends on both the input and current state, then we'd have nested switch statements in every
method! Not only is that a lot of code duplication, it's super ugly and takes a lot of space.

### Visitor

The goal of a visitor is to cut down on switches when behavior depends on 2 or more factors.
When applying the visitor pattern, we typically have two hierarchies: a hierarchy of nodes and visitors.
A node accepts a visitor. Using these two hierarchies, we can have unique behavior for each node-visitor combination.

Consider we have three credit card classes, normal, business, and gold. Now also consider we have three
types of purchases: services, consumer goods, and capital goods. We want each credit card and purchase
type combination to have different discounts. Let's use the Visitor pattern to do this in an OOP way
that avoids nested switches or nested if sequences.

```C++
// Node heirarchy
class CreditCard {
public:
    ~CreditCard() = default;
    
    // visitor accept function
    virtual double getPurchaseDiscount(const PurchaseType & p, int amount) = 0;
};

class RegularCard : public CreditCard {
public:
    double getPurchaseDiscount(const PurchaseType & p, int amount) override {
        return p.visitRegular(this, amount);
    };
};

class BusinessCard : public CreditCard {
public:
    double getPurchaseDiscount(const PurchaseType & p, int amount) override {
        return p.visitBusiness(this, amount);
    };
};

class GoldCard : public CreditCard {
public:
    double getPurchaseDiscount(const PurchaseType & p, int amount) override {
        return p.visitGold(this, amount);
    };
};


// Visitor heirarchy
class PurchaseType {
public:
  virtual ~PurhcaseType() = default;

  virtual double visitRegular(RegularCard * card, int amount) = 0;
  virtual double visitBusiness(BusinessCard * card, int amount) = 0;
  virtual double visitGold(GoldCard * card, int amount) = 0;
};

class ServicePurhace : public PurchaseType {
public:

  double visitRegular(RegularCard * card, int amount) override {
        return 0;
  }

  double visitBusiness(BusinessCard * card, int amount) override {
        return std::min(amount * 0.001, 0.25);
  }

  double visitGold(GoldCard * card, int amount) override {
        return 0.08;
  }
};

class ConsumerPurhace : public PurchaseType {
public:

  double visitRegular(RegularCard * card, int amount) override {
        return 0.001;
  }

  double visitBusiness(BusinessCard * card, int amount) override {
        return 0;
  }

  double visitGold(GoldCard * card, int amount) override {
        return 0.15;
  }
};

class CapitalPurhace : public PurchaseType {
public:

  double visitRegular(RegularCard * card, int amount) override {
        return 0;
  }

  double visitBusiness(BusinessCard * card, int amount) override {
        return std::min(amount * 0.001, 0.8);
  }

  double visitGold(GoldCard * card, int amount) override {
        return 0.05;
  }
};
    
```

A concrete visitor visits a concrete node. The concrete node calls the corresponding visitor method.
The visitor method then performs the computation. This computation can depend on both the Visitor's type,
and the Node's type.

### Iterator

Already discussed

### Template Method

Let's say we had a complicated algorithm that spanned multiple steps like in our computer vision example.
As a reminder, we filtered the image, applied a Hough Transform, then used a neural network.
What if we wanted to run this algorithm on an embedded device, and change our algorithm to 
take less memory?

We can make this algorithm a template method, and make each step of the algorithm a virtual function so
subclasses can customize the behavior.

```C++
class OCR {
    virtual cv::Mat filter(const cv::Mat & img) = 0;
    virtual std::vector<Lines> findLines(const cv::Mat & img) = 0;
    virtual std::string ocrStep(std::vector<cv::Mat> characterImages) = 0;
public:
    virtual ~OCR() = default;

    // this is the template method
    std::string operator()(const cv::Mat & img) {
        const auto filtered = filter(img);
        const auto lines = findLines(filtered);
        // ...
        // compute line intersections
        return ocrStep(intersectionImages);
    }
};
```

Subclasses can now customize the function. This function is known
as a template method. The parts of the algorithm that don't change are kept in a base class
and hooks for customization are provided to subclasses.
This pattern basically applies the Open-Close Principle
to customize a computation. In C++, it's also quite similar to the NVI idiom.

### Strategy

Strategy is similar to template method, however instead of subclasses overriding customization hooks,
they can override the entire algorithm itself.

```C++
class OCR {
public:
    virtual ~OCR() = default;
    virtual std::string operator()(const cv::Mat & img) = 0;
};
```

In a way, abstract factory is to the factory function in the same way that strategy is to the template method.


### Prototype

Said simply: this pattern is to create a class hierarchy that has a polymorphic `clone()` function
to create a new instance of an object from another.

### Singleton

A singleton prevents multiple instances of itself from being created. This can be used in situations
where it doesn't make sense to have multiple instances of a module such as a `Logger` class. It does this
by making its constructor private, and only allowing access to the instance via a static method.
Singleton's can also have lazy initialization.
That means they don't create an instance until it's actually used during execution of the static `get` method.

This pattern must be used carefully. While a Singleton is slightly better than globals, the Singleton
itself is still a global object of sorts.

In C++, the order of initialization and de-initialized between global and static data is undefined.
Therefore, if we use Singleton `A` in the destructor of singleton `B`, it's possible that when `B` is destroyed,
the static data of `A` has already been deleted leaving us with undefined behavior. 

Here's a simple example (that does not deal with the aforementioned dead reference problem)

```C++
class LoggerSingleton {
    LoggerSingleton() = default; // private constructor
public:
    static const LoggerSingleton & get() {
        static LoggerSingleton singleton; // only created once in first call to get
        return singleton;
    }

    void log(std::string_view str) {
        // ...
    }

};

LoggerSingleton::get().log("Algorithm started!");
``` 

A singleton can also use smart pointers too.

```C++
class LoggerSingleton {
    LoggerSingleton() = default; // private constructor

    static std::unique_ptr<LoggerSingleton> logger;
public:
    static const LoggerSingleton & get() {
        if (!logger) {
            logger = std::unique_ptr(new LoggerSingleton());
            // cannot use make_unique because
            // the constructor is private
        }
        return *logger.get();
    }

    void log(std::string_view str) {
        // ...
    }

};

LoggerSingleton::get().log("Algorithm started!");
``` 

### Chain of Responsibility

Imagine that you have a problem with your pay, and think that there a mistake being made. Your first move might be to talk to your manager.
Your manager understands your point, but can't do anything himself, so he talks to his HR contact. Your manager's HR rep finds that there is a policy
in place that overlooked a corner case that you so happen to be a part of. So the HR rep talks to their manager to see if this policy can be changed.
The HR rep's manager can't change it, but asks their manager to see if they can change it. We continue this process until
we get to the CEO or an HR Executive who is able to change the policy. This is basically a chain of responsibility.

We can raise an event in one module, and keep passing the event along until we get to a module that can handle it.

Let's say we click a `New Page` button. The button itself can't create a new page, so it passes the event to the toolbar.
The toolbar can't create one, so it passes the event to the widget it is a part of. The widget also can't create one, so it passes
the same event to the window. Now the window, creates a new page.

### Repository

This is a pattern for abstracting data access. Truthfully, I think using the term "Repository Design Pattern" just makes us sound smart,
but in reality this pattern boils down to the good old adage: "program to an interface, not an implementation."

Instead of using a database directly, the repository abstracts the data behind an interface. This allows us to have different implementations
of the repository. This is especially important for testing. Abstracting data access being a repository allows us to have a mock database implementation.

Uncle Bob refers to this as a Gateway.

```C++
class DataRepository {
public:
    virtual ~DataRepository() = default;
    virtual void save(std::string key, std::string value) = 0;
    virtual std::optional<std::string> read(std::string_view key) = 0;
};

class SQLiteRepo : public DataRepository {};
class MockRepo : public DataRepository {};
class RemoteSQLRepo : public DataRepository {};
```

The repository now forces us to separate business logic from data access logic. Code that needs a database
doesn't know or care what kind of database it's using. This allows us to change databases,
mock databases, and more without changing a single line of existing code.

This pattern is from Martin Fowler's *Patterns of Enterprise Application Architecture*.