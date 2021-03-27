# Declarative C++

## Motivation
With the introduction of designated initializers, C++ seemed quite close to be used also in a declarative way,sadly, there are a few things missing.
Let's take an example code for better understanding:

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
auto window = std::make_shared<window>(/*args*/);
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

Now, let's try to rewrite the above code in a *declarative* way


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

```c++
auto window = std::make_shared<Widget>(/*constructor args*/)::[this = Window]{
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

Object lambdas are similar to lambdas, with the following differences:

 - explicit `this` type specifier in capture list i.e. `[=, this = Type]`
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
window::[&window, this = Window]{};
  ```

  will be nicer to have:

  ```c++
auto window = std::make_shared<Widget>(/*constructor args*/)::[&window, this = Window]{};
  ```

  - there is no easy way to name the `children` and references them, instead to write:

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

will be nice to write:

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
};
```