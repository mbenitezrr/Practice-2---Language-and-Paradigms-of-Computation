# Integrated Itinerary and Fare Engine - Metro System of the Valle de Aburra

Second Practice - SI1002-7 Languages and Paradigms of Computation, 2026-2, Universidad EAFIT.

## Team

- Mateo Benítez Ramírez (C.C. 1021924163)

## How to build and run

Requires g++ with C++17 support. The program reads nothing from standard input.

On Online GDB click on the gear icon and then click Extra Compiler Flags, and type: -Wall -Wextra
The program compiles and runs with zero warnings.


## Design summary

- **Part 1:** `TransportMode` is abstract (`baseFare()` and `serviceType()` are pure virtual), has a virtual destructor,
  three hooks that default to `true`, and a non-virtual `describe()` (template method). `TouristCable` reuses
  `CableLine::baseFare()`.
- **Part 2:** `Passenger` applies the Non-Virtual Interface idiom: the public non-virtual `fareOn()` clamps the result
  with `std::max(0LL, ...)`, and the protected virtual `computeFare()` is the customisation point.
- **Part 3:** `Network` owns its modes with `std::unique_ptr`. `chargeItinerary` sorts a copy of the legs with a lambda and
  applies the integration rule in the only explicit loop of the program.
- **Part 4:** the five reports use `std::accumulate`, `std::count_if`, `std::for_each` and an injected `std::function`,
  with no explicit loops.

## Defence questions

### 1. Why must the destructor of `TransportMode` be virtual? What happens without it?

`Network` stores its modes as `std::unique_ptr<TransportMode>`, so every mode is destroyed through a pointer to the
base class. If the destructor is not virtual, deleting a derived object through a base pointer is undefined behaviour
according to the C++ standard. In practice only `~TransportMode()` runs, so the destructor of the derived class
(`MetroLine`, `TouristCable`, ...) never runs, and anything the derived class owns is leaked or left in an
inconsistent state. With a virtual destructor, the call is dispatched dynamically: the most derived destructor runs
first and then the base destructor, so the object is destroyed completely. Here the derived classes happen to hold
only simple members, so the bug would stay hidden, but it is still a correctness requirement: the base class cannot
know what future subclasses will own.

### 2. Why is an abstract class with pure virtual functions preferable to an enumeration plus a switch?

With an enumeration plus a switch, the knowledge of "what each mode is" is scattered across the program: every place
that needs a mode-dependent value (fare, service type, student policy, integration policy) has its own `switch`.
When a new mode appears, all of those places must be found and modified, and the compiler does not guarantee that none
was forgotten. With an abstract class, each mode is a single class that declares its own behaviour. Adding a mode means
writing one new class and touching no existing code (open/closed principle). The pure virtual functions force each
concrete class to supply `baseFare()` and `serviceType()`, because the compiler refuses to instantiate a class that
does not, and the hooks with default values only need to be overridden where the behaviour differs. For example,
`TouristCable` was added by overriding three hooks, and `StudentPassenger` asks the mode for its policy through
`allowsStudentFlatFare()` instead of asking what class it is. This is also why the assignment forbids `dynamic_cast`,
`typeid` and comparing `serviceType()` strings: they are a `switch` in disguise.
