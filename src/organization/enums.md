# Enumerations

Suppose you have a class representing vehicles, and wanted a way to store whether the vehicle was electric, gas, or a hybrid. 
You might consider using a `char` and creating constants `gas = 'g', electric = 'e', hybrid = 'h'`. 
This isn't too bad of a system, but operations on this information would be operations on `char`s, so what's stopping a user from accidentally using `'f'`? 
As far as the compiler is concerned, nothing.

This is the motivation for enums, a type to represent enumerations of possible values. Here's how our original problem might look:

```C++
enum class VehicleType {
    gas, electric, hybrid
};

double getBaseRefuelingCost(VehicleType typ) {
    switch (typ) {
        case VehicleType::gas:
            return 2.49;
        case VehicleType::hybrid:
            return 1.00;
        case VehicleType::electric:
            return 0.50;
    }
}

getBaseRefuelingCost(VehicleType::gas);
getBaseRefuelingCost(0); //error
```

Internally, an enum is represented as some integral type (`int` by default). 
However, as you can see the conversion is not implicit. This provides type checking to ensure that you can only use valid values of the enumeration.

Still, you can explicitly initialize an enum with an integer using similar syntax as initializing a `struct`.

`getBaseRefuelingCost(VehicleType {0}) // be default, 0 is the value of the first enumeration value defined (so gas)`

If you need to, you can actually specify the underlying integral type of the enumeration and can also specify the underlying value of the enumerated values as well. 
Each enumeration will have an underlying value of 1 more than the previous one, and by default the first underlying value is 0

```C++
enum class Colors : char { //underlying type is char
    black, //0
    red = 10, 
    green, //11
    blue, //12
    orange = 20,
    yellow // 21
}
```

The above is known as a *scoped enum*. 
If you remove the `class` keyword after `enum`, you can create a "regular" `enum` which has implicit conversion to int and does not require qualifying the enum name.