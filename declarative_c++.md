# Declarative C++

## Motivation
With the introduction of designated initializers, C++ seemed quite close to be used also in a declarative way,sadly, there are a few things missing.
[QML](https://en.wikipedia.org/wiki/QML#Syntax,_semantics) is one of the best example in this area, so, let's take a widgets example code for better understanding:

```c++
class Widget {
public:
    Widget(/*some args*/) { /*do some work here*/}
    Property<int> x;
    Property<int> y;
    Property<int> width;
    Property<int> height;
    std::vector<std::shared_ptr<Widget>> children;
};

class Label : public Widget {
public:
    Label(/*some args*/) : Widget(/*args*/) { /* do some work here */ }
    Property<std::string> text;
};

class Button : Label {
public:
    Signal<void()> clicked;
};
```

## "Imperative" construction of a window

```c++
auto window = std::make_shared<Widget>(/*args*/);
window->height = 10;
window->width = 100;
{
    auto label = std::make_shared<Label>(/*args*/);
    label->y = 10;
    label->text = "Some text";
    window->children.push_back(label);
}
{
    auto button = std::make_shared<Button>(/*args*/);
    button->text = "Quit";
    button->clicked = [window = window.get()]{
        window->closed();
    };
}
```

What are the problems with the previous code:

 - not very nice nor very intuitive.
 - hard to spot errors (e.g. the `button` object is not added to the `window`'s children list).
 - `window->`, `label->` and `button->` are redundant and slow as the `->` is a custom operator which might require some CPU time, yes we can take a pointer/reference of the underlining object to save the CPU time but it will not make the code cleaner, also almost nobody is doing it.

Now, let's try to rewrite the above code in a *declarative* way, similar to https://en.wikipedia.org/wiki/QML#Syntax,_semantics.


## V1: Using designated initializers

```c++
auto window = new Widget {
    .(/*Widget's constructor args goes here*/),
    .height = 10,
    .width{100}, // mind the out-of-order
    .children {
        new Label {
            .y = 10,
            .text{"Some label"}
        },
        new Button {
            .text{"Quit},
            .clicked = [/*how do I use window ?/*]{
                // window->close(); ?!?!?!
            };
        }
    }
};
```
### Designated initializers will still need some changes:
 - allow out-of-order fields **designated** initialization (as in C), even though the designated doesn't match the declared order, the declared order will be used to initialize the object.
 - allow designated constructors initialization. E.g. `.(/*args*/)` will call the object constructor (after the fields are initialized).

Unsolved problems:

 - can't use std::**smart_ptr**s ?
 - no way to pass parent info ?


## V2: using object lambdas
### vanilla C++
```c++
/// add a new constructor to Widget
class Widget {
public:
    template<class F> requires std::invocable<F, Widget&>
    explicit Widget(const F& setup) {
        setup(*this);
    }
///...
};

auto window = std::make_shared<Widget>([](auto& window) {
    window.width = 100;
    window.height = 10;
    window.children = {
        std::make_shared<Label>([](auto& label) {
            label.y = 10;
            label.text = "Some label";
        }),
        std::make_shared<Button>([&](auto& button) {
            button.text = "Quit";
            button.clicked = [window = &window]{ window->close(); };
        }),
    };
});
```

### using *object lambdas* proposal
```c++
auto window = std::make_shared<Widget>(/*constructor args*/)::[this = Widget]{
    width = 100;
    height = 10;
    children = {
        std::make_shared<Label>(/*constructor args*/)::[this = Label]{
            y = 10;
            text = "Some label";
        },                                              // window is a Widget*
        std::make_shared<Button>(/*constructor args*/)::[window = this, this = Button] {
            text = "Quit";
            clicked = [window]{window->close()};
        }
    };
};
```
As you can see bot are quite similar, the main difference is that using **object lambdas** you don't need to specify the methods object as `this` already specifies it, which makes the code cleaner and closer to declarative languages.

Object lambdas are similar to lambdas, with the following differences:

 - explicit `this` type specifier in capture list i.e. `[=, this = Type]`
 - allow object lambdas to be used as a member function pointer.
 - the object lambda block will behave as a member function, the only difference is that it can access **only** public class members.
 - introduce a new overload-able operator **::** `template <typename ThisType> auto & operator::(void (ThisType::*f)());`

Possible default **::** operator for classes:

 ```c++
template<typename ThisType>
auto &operator::(void (ThisType::*func)()) {
    std::invoke(func, *this);
    return *this;
}
```
Possible default **::** operator for `shared|unique_ptr` smart_ptrs:

 ```c++
template<typename ThisType>
auto &operator::(void (ThisType::*func)()) {
    static_assert(std::is_same_v<T, ThisType>, "Invalid type");
    std::invoke(func, *get());
    return *this; // we're returning a std::shared_ptr<T> & not a ThisType&
}
```

Unsolved problems:
 - there is no easy way to pass the `window` object to the lambda capture list e.g. instead to write:

  ```c++
auto window = std::make_shared<Widget>(/*constructor args*/);
window::[&window, this = Widget]{};
  ```

  will be nicer to have:

  ```c++
auto window = std::make_shared<Widget>(/*constructor args*/)::[&window, this = Widget]{};
  ```
alternatively we can have a *smarter* `operator ::` for smart_ptrs:
```c++
// shared_ptr operator::
template<typename ThisType>
auto &operator::(void (ThisType::*func)(const std::shared<ptr> &)) {
    static_assert(std::is_same_v<T, ThisType>, "Invalid type");
    std::invoke(func, *get(), *this);
    return *this; // we're returning a std::shared_ptr<T> & not a ThisType&
}
//...

std::make_shared<Class>(/*Class contructor args*/)::[this = Class](const std::shared_ptr<Class> &self) {
    // make use of self here
    // ...
};
```

 - there is no easy way to name the `children` and references them, instead of:

```c++
auto label = std::make_shared<Label>(/*constructor args*/)::[this = Label]{
    y = 10;
    text = "Some label";
};
auto button = std::make_shared<Button>(/*constructor args*/)::[label, this = Button] {
    text = "Quit";
    clicked = [label, window] {
        // use label here
        window->close()
    };
};
children = { label, button };
```

will be nicer to have:

```c++
children = {
    auto label = std::make_shared<Label>(/*constructor args*/)::[this = Label]{
        y = 10;
        text = "Some label";
    }, // label will be visible below, same as if (auto foo = ) { // use foo}
    std::make_shared<Button>(/*constructor args*/)::[label = label, this = Button] {
        text = "Quit";
        clicked = [label, window] {
            // use label here
            window->close()
        };
    }
}; // label scope ends here
```
