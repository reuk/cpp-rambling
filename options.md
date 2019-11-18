# Avoiding `#if`

Or, "hey, can we go back to using `#if` please?"

## The case for `#if`

Sometimes in C/C++ we want some code to change its behaviour depending on options which are selected
at compile-time.

The canonical way to do this is to use the preprocessor to selectively remove bits of our program
before the compiler proper gets to see it.

Imagine we have a widget of some kind which retains most of its behaviour regardless of mode, but
has a few mode-specific characteristics. Such a class might look a bit like this:

```cpp
// Don't do this!
class Widget
{
public:
    int getWidgetHeight() const
    {
        const auto baseHeight = title.getHeight() + subtitle.getHeight();

        // Do something slightly different depending on the data members that are present
#if COOL_MODE_ENABLED
        const auto additionalHeight = radMeterDisplay.getHeight();
#else
        const auto additionalHeight = 0;
#endif
        return baseHeight + additionalHeight;
    }

private:
    Title title;
    Subtitle subtitle;

    // If Cool Mode is on, that means we can show some particle effects I guess
#if COOL_MODE_ENABLED
    RadMeterDisplay radMeterDisplay;
    NeatParticleEffect particles;
#else
    BeigeBackgroundImage normalBoringBackground;
#endif
};
```

Then, in our CMakeLists.txt (other build systems are available) we might use
`target_compile_definitions(widgets PRIVATE COOL_MODE_ENABLED=1)` to select the current build mode.

Code such as this is fairly common, but it's very ugly, and not particularly easy to work with.
We only build one mode at a time, and it's very easy to break the modes we're *not* building without
realising. Also, your IDE won't be able to help you with refactoring code like this, because it
will completely ignore code in `#if` branches that aren't being built.

There must be a better way.

## Virtual Inheritance

The OOP 'solution' to this problem would be to turn our widget into a virtual base class:

```cpp
class WidgetBase
{
public:
    virtual ~WidgetBase() noexcept = default;

    int getWidgetHeight() const
    {
        const auto baseHeight = title.getHeight() + subtitle.getHeight();
        return baseHeight + getAdditionalHeight();
    }

private:
    virtual int getAdditionalHeight() const = 0;

    Title title;
    Subtitle subtitle;
};

class CoolWidget : public WidgetBase
{
    int getAdditionalHeight() const override { return radMeterDisplay.getHeight(); }

    RadMeterDisplay radMeterDisplay;
    NeatParticleEffect particles;
};

class LameWidget : public WidgetBase
{
    int getAdditionalHeight() const override { return 0; }

    BeigeBackgroundImage normalBoringBackground;
};
```

The one redeeming quality of this code is that it all builds, all the time. That means we're not in
danger of accidentally making changes that break certain modes.

Unfortunately, we've introduced some virtual dispatch which will slow our program down. Also, all
our logic is spread across lots of classes now, so it's not that readable. Tracking down control
flow through virtual functions can be quite awkward.

A bigger problem, though, is that now we need some kind of factory which will create the right
derived type of `Widget` for us depending on the current build mode. The factory can't encode
the real derived type in its interface, so it will have to use a base-class pointer instead:

```cpp
std::unique_ptr<WidgetBase> makeWidget()
{
#if COOL_MODE_ENABLED
    return std::make_unique<CoolWidget>();
#else
    return std::make_unique<LameWidget>();
#endif
}
```

Yuck. Considering that often these additional build modes are added into an existing program as
requirements change ("hey, we need a build of this app without feature XYZ for that demo later")
it's not going to be feasible to rewrite all our classes which interact with Widget to convert
value semantics into pointer semantics. Not that we would want to anyway, because of the virtual
dispatch thing.

## Surely Nothing Can Be Better Than OOP?

The ideal solution here would be some mechanism that avoided overhead from virtual dispatch, and let
us carry on using our classes with value semantics (i.e. not via pointers). It should also build as
much of the program as possible in all modes, as a precaution against accidental breakage.

Let's start, in the tradition of all great C++ solutions, by adding some nice new strong shiny
types.

```cpp
struct CoolDataMembers
{
    RadMeterDisplay radMeterDisplay;
    NeatParticleEffect particles;
};

int getAdditionalHeight (const CoolDataMembers& m) { return m.radMeterDisplay.getHeight(); }

struct LameDataMembers
{
    BeigeBackgroundImage normalBoringBackground;
};

int getAdditionalHeight (const LameDataMembers& m) { return 0; }
```

Here, we're using normal function overloading to accomplish the same thing as the virtual methods
from the previous solution. This time, there's no virtual dispatch overhead, which is already an
improvement. Unfortunately we're still going to have to scatter around lots of `#if`s if we try
to use this directly. Let's try turning our structs into template struct specialisations instead.
That normally helps.

```cpp
enum class AppMode
{
    coolOff,
    coolOn,
};

```

Now, instead of explicitly using `CoolDataMembers` or `LameDataMembers`, we can do something like
this in a header somewhere:

```cpp
#if COOL_MODE_ENABLED
constexpr auto currentAppMode = AppMode::coolOn;
#else
constexpr auto currentAppMode = AppMode::coolOff;
#endif
```

And then, we can just declare a member of `DataMembers<currentAppMode>`. This way, we just have a
single `#if`, like the OOP solution. The whole thing would look like this:

```cpp
enum class AppMode
{
    coolOff,
    coolOn,
};

#if COOL_MODE_ENABLED
constexpr auto currentAppMode = AppMode::coolOn;
#else
constexpr auto currentAppMode = AppMode::coolOff;
#endif

class Widget
{
public:
    int getWidgetHeight() const
    {
        const auto baseHeight = title.getHeight() + subtitle.getHeight();
        return baseHeight + getAdditionalHeight (modeMembers);
    }

private:
    template <AppMode = currentAppMode> struct DataMembers;

    template <>
    struct DataMembers<AppMode::coolOn>
    {
        RadMeterDisplay radMeterDisplay;
        NeatParticleEffect particles;
    };

    static int getAdditionalHeight (const DataMembers<AppMode::coolOn>& m) { return m.radMeterDisplay.getHeight(); }

    template <>
    struct DataMembers<AppMode::coolOff>
    {
        BeigeBackgroundImage normalBoringBackground;
    };

    static int getAdditionalHeight (const DataMembers<AppMode::coolOff>& m) { return 0; }

    Title title;
    Subtitle subtitle;
    DataMembers<> modeMembers;
};
```

This solves most of our problems. There's no virtual dispatch. We still have value semantics. We
build everything in every mode, so it's difficult to accidentally introduce errors.

The main problem that remains is that our `getAdditionalHeight` function sits quite a long way
away from where it's used, which doesn't make that much sense because its purpose is intrinsically
linked with `getWidgetHeight()`. In more complex cases, we might want to pass some local variables
from the outer function (`getWidgetHeight`) into the specialised function (`getAdditionalHeight`)
which can lead to nasty long parameter lists. The real pain starts when only one mode needs to
know about the local variables, because we need all overloads of `getAdditionalHeight` to share
the same parameter list. This means that one of our specialised functions will have a bunch of
unused unnecessary parameters.

## Coolness Overload

The solution comes in the form of this neat C++17 class, which is often discussed in conjunction
with another C++17 feature, `std::visit`:

```cpp
template <class... Ts> struct Overloaded : Ts... { using Ts::operator()...; };
template <class... Ts> Overloaded (Ts...) -> Overloaded<Ts...>;
```

This class allows us to build a mini overload set on demand. You can think of it as combining
several lambdas or function objects into one super-function with overloads for all the `operator()`
functions on the original objects.

It can be used like this:

```cpp
const auto overloadSet = Overloaded
{
    [] (const char* str) { std::cout << "the string reads: " << str << '\n'; },
    [] (int i) { std::cout << "the int has value: " << i << '\n'; },
};

overloadSet ("hello world"); // prints "the string reads: hello world"
overloadSet (42);            // prints "the int has value: 42"
```

If we were to use this to implement `getWidgetHeight()`, it might look like this:

```cpp
int getWidgetHeight() const
{
    const auto baseHeight = title.getHeight() + subtitle.getHeight();
    return baseHeight + Overloaded
    {
        [] (const DataMembers<AppMode::coolOn>& m) { return m.radMeterDisplay.getHeight(); },
        [] (const DataMembers<AppMode::coolOff>&) { return 0; },
    } (modeMembers);
}
```

We did it! In this version, each 'overload' can capture whichever variables from local scope it
requires, and we don't have to jump/context-switch to `getAdditionalHeight` to read this code. This
is even better than the last version, in fact, because the unused lambda branches will get pruned
away by the compiler (although they'll still be parsed and checked), so we don't need to worry about
unused functions hanging around in our final binary. Truly, this is the holy grail of modern C++:
fast, typesafe, maintainable, and utterly unreadable.

Maybe the preprocessor isn't so bad after all.

## Postscript: `switch constexpr`

C++17 gave us `if constexpr` which is fine unless you really like SFINAE. However, there's no
equivalent to a compile-time switch, and instead we have to be content to do lots of ugly `if
constexpr (...) else if constexpr (...) else ...` branches. Except `switch constexpr` exists. It's
just spelled `Overloaded`.

```cpp
// We need some helpers...

// This is a compile-time type-level constant with automatic type deduction
template <auto T>
using IntegralConstant = std::integral_constant<decltype (T), T>;

// This is a value of IntegralConstant type
template <auto T>
constexpr auto integralConstant = IntegralConstant<T>{};

// Now we can put it all together
template <typename T>
inline auto dispatchOnSize (T t)
{
    // beautiful
    Overloaded
    {
        [&] (IntegralConstant<1>) { doTinyDispatch (t); },
        [&] (IntegralConstant<2>) { doMediumDispatch (t); },
        [&] (IntegralConstant<4>) { doLargeDispatch (t); },
        [&] (auto) { doDefaultDispatch (t); },
    } (integralConstant<sizeof (T)>);
}

// This is what that would look like without `switch constexpr`
template <typename T>
inline auto dispatchOnSizeButYucky (T t)
{
    constexpr auto s = sizeof (T);

    // hideous
    if constexpr (s == 1)
        doTinyDispatch (t);
    else if constexpr (s == 2)
        doMediumDispatch (t);
    else if constexpr (s == 4)
        doLargeDispatch (t);
    else
        doDefaultDispatch (t);
}
```
