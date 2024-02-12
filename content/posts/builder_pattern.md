---
title: "Builder Pattern in Go and Rust"
date: 2024-02-12
draft: false
categories: ["Rust"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

The Builder Pattern is a design pattern that provides a way to construct a complex object step by step, focusing on breaking down the construction of the object into smaller, more manageable steps. It separates the construction of a object from its representation, allowing the same construction process to create different variations or representations of the final object. This helps improve code readability, especially when dealing with objects that have a large number of attributes or configurations.

## Go Builder Pattern

The final code with extra bits and pieces that may not be included in the below explanation can be found [HERE](https://github.com/danrusei/dev-state_blog_code/tree/master/builder_pattern/builder_Go).

Let's start with the Car Struct that defines the instance we would like to construct. It may contains fields (firstPrivateField and secondPrivateField) that are not known by the users of the library, maybe for library internal stuff.

{{< code language="go" isCollapsed="false" >}}
// A car with specific attributes.
type Car struct {
 equip              Equiped
 color              string
 carEngine          Engine
 firstPrivateField  string
 secondPrivatefield string
}

// Engine represents the type of car engine
type Engine struct {
 petrol   Petrol
 diesel   Diesel
 electric Electric
}
{{< /code >}}

`Engine` is a struct that represents the type of a car engine. There are declared a bunch variants of the `Petrol`, `Diesel`, `Electric`, and `Equiped` as custom types to represent various aspects of a car's engine and equipment.

`CarBuilder` is a struct that acts as a builder for creating instances of the `Car` struct. It contains a `car` field of type `Car` that will be gradually constructed.

{{< code language="go" isCollapsed="false" >}}
type CarBuilder struct {
 car Car
}
{{< /code >}}

`NewCarBuilder` is a function that initializes a new `CarBuilder` instance with the specified equipment.
It sets the `equip` field, as mandatory field, in the underlying `Car` struct.

{{< code language="go" isCollapsed="false" >}}
func NewCarBuilder(equip Equiped) *CarBuilder {
 return &CarBuilder{
  car: Car{
   equip: equip,
  },
 }
}
{{< /code >}}

These below methods are setter functions to different attributes of the Car struct. They take a parameter, set the corresponding field in the underlying `Car` struct, and return a pointer to the `CarBuilder` for **method chaining**.

{{< code language="go" isCollapsed="false" >}}
// SetColor sets the color of the car
func (cb *CarBuilder) SetColor(color string)*CarBuilder {
 cb.car.color = color
 return cb
}

// SetPetrolEngine sets the petrol engine type
func (cb *CarBuilder) SetPetrolEngine(power Petrol)*CarBuilder {
 cb.car.carEngine.petrol = power
 return cb
}

// SetDieselEngine sets the diesel engine type
func (cb *CarBuilder) SetDieselEngine(power Diesel)*CarBuilder {
 cb.car.carEngine.diesel = power
 return cb
}

// SetElectricEngine sets the electric engine type
func (cb *CarBuilder) SetElectricEngine(power Electric)*CarBuilder {
 cb.car.carEngine.electric = power
 return cb
}
{{< /code >}}

The `Build` method finalizes the construction process and returns the fully constructed `Car` instance.

{{< code language="go" isCollapsed="false" >}}
// Build constructs and returns the final Car instance
func (cb *CarBuilder) Build() Car {
 return cb.car
}
{{< /code >}}

The user of the library can chain the methods to create the `Car` instance:

{{< code language="go" isCollapsed="false" >}}
// Example usage of Builder Pattern:
 builder := car1.NewCarBuilder(car1.EquipedGold).
  SetColor("Brown").
  SetPetrolEngine(car1.Petrol225HP)

 car_ex1 := builder.Build()

    fmt.Printf("Builder Pattern: %v\n", car_ex1)
{{< /code >}}

and now if we run the code:

{{< code language="bash" isCollapsed="false" >}}
$ go run main.go

Builder Pattern: You selected the Car equiped with Gold Package, having a Petrol 225HP engine and Brown color
{{< /code >}}

## Go Builder Pattern with Functional Options

The Functional Options Pattern is a flexible way to provide optional parameters to a function or constructor. In this pattern, **each option is represented as a function** that takes a pointer to the target struct and modifies its fields accordingly.

{{< code language="go" isCollapsed="false" >}}
// Option is a functional option for configuring Car instances

type Option func(*Car)

{{< /code >}}

The `Car` struct is the same asabove, but instead of creating of an additional builder struct, we modify the object in place. The below functions create options for configuring different aspects of the Car. Each function returns an Option, which is essentially a closure that modifies the Car instance.

{{< code language="go" isCollapsed="false" >}}
/ WithColor sets the color of the car
func WithColor(color string) Option {
 return func(c *Car) {
  c.color = color
 }
}

// WithPetrolEngine sets the petrol engine type
func WithPetrolEngine(power Petrol) Option {
 return func(c *Car) {
  c.carEngine.petrol = power
 }
}

// WithDieselEngine sets the diesel engine type
func WithDieselEngine(power Diesel) Option {
 return func(c *Car) {
  c.carEngine.diesel = power
 }
}

// WithElectricEngine sets the electric engine type
func WithElectricEngine(power Electric) Option {
 return func(c *Car) {
  c.carEngine.electric = power
 }
}

// WithEquipment sets the equipment type
func WithEquipment(equip Equiped) Option {
 return func(c *Car) {
  c.equip = equip
 }
}
{{< /code >}}

The magic happens on the code below, where the `NewCar` function takes a variadic number of Option functions. It initializes a new `Car` instance and applies each option to configure the Car, returning the instance of the newly created Car.

{{< code language="go" isCollapsed="false" >}}
// NewCar creates a new Car instance with specified options

func NewCar(options ...Option) *Car {
 car := &Car{}
 for _, option := range options {
  option(car)
 }
 return car
}
{{< /code >}}

Now the user of the library can construct the Car instance using the Functional Option Pattern:

{{< code language="go" isCollapsed="false" >}}
 // Example usage of Functional Options Pattern
 car_ex2 := car2.NewCar(
  car2.WithEquipment(car2.EquipedSilver),
  car2.WithColor("White"),
  car2.WithElectricEngine(car2.Electric420kW),
 )

 fmt.Printf("Functional Options Pattern: %s\n", car_ex2)
{{< /code >}}

if we run the code:

{{< code language="bash" isCollapsed="false" >}}
$ go run main.go

Functional Options Pattern: You selected the Car equiped with Silver Package, having a Electric 420kW engine and White color
{{< /code >}}

## Rust Consuming Builder Pattern

The intent is to replicate the above functionality, of constructing the `Car` instance in Rust. It is well known that Rust is not a garbage collector language, and relies on ownership and borrowing as the pillars of memory safety and concurency.

The final code with extra bits and pieces that may not be included in the below explanation can be found [HERE](https://github.com/danrusei/dev-state_blog_code/tree/master/builder_pattern/builder_Rust).

The `Car` struct represents a car with several optional fields. We derive Default trait in order to be able to construct the struct using the fields default values. It will be usefull below while we initialize the `CarBuilder` struct.

{{< code language="rust" isCollapsed="false" >}}
 #[derive(Debug, Default)]
pub struct Car {
    equip: Equiped,
    color: Option<String>,
    car_engine: Option<Engine>,
    first_private_field: Option<String>,
    second_private_field: Option<String>,
}
{{< /code >}}

The Rust **Enums** makes easier to represent the data, the example below is the Engine enum that represents various types of engines and has three variants: `Petrol`, `Diesel`, and `Electric`. Each variant have an associated Enum.

{{< code language="rust" isCollapsed="false" >}}

# [derive(Debug, Clone)]

pub enum Engine {
    Petrol(Petrol),
    Diesel(Diesel),
    Electric(Electric),
}
{{< /code >}}

Define the `CarBuilder` struct, used to construct the final instance of the Car.

{{< code language="rust" isCollapsed="false" >}}
pub struct CarBuilder {
    car: Car,
}
{{< /code >}}

Below we initialize the `CarBuilder` and define a set of methods that modifies the builder.

* The `new` method is a static method associated with `CarBuilder` that initializes a new `CarBuilder` with a specified equipment `equip`. It sets the initial state of the car field using the provided equipment and the default values for other fields.
* The `set_color` &  `set_engine` methods takes a mutable reference to self and a value, sets the coresponding fields and returns the modified builder.
* The `build` method **consumes the builder (self)** and constructs a `Result<Car>`. It checks whether a car engine has been set, returning an error if not. It then constructs a new `Car` instance using the builder's fields, providing default values where necessary.

{{< code language="rust" isCollapsed="false" >}}
impl CarBuilder {
    pub fn new(equip: Equiped) -> Self {
        CarBuilder {
            car: Car {
                equip: equip,
                ..Default::default()
            },
        }
    }
    pub fn set_color(mut self, color: impl Into<String>) -> Self {
        self.car.color = Some(color.into());
        self
    }
    pub fn set_engine(mut self, engine: Engine) -> Self {
        self.car.car_engine = Some(engine);
        self
    }
    pub fn build(self) -> Result<Car> {
        let Some(car_engine) = self.car.car_engine else {
            return Err(anyhow!("You need to select an Engine type"));
        };

        Ok(Car {
            equip: self.car.equip,
            color: Some(self.car.color).unwrap_or(Some("White".to_owned())),
            car_engine: Some(car_engine),
            ..Default::default()
        })
    }
}
{{< /code >}}

## Rust Non-Consuming Builder Pattern

Non-consuming builder pattern is usefull if we need to construct multiple instances of the `Car` struct. The main difference between the two implementations lies in how ownership and mutation are handled in the builder pattern. Notice the return of `&mut Self`, instead of the `Self` above.

In the first example the `CarBuilder` takes ownership of the `Car` it is constructing and the methods like `set_color` and `set_engine` consume and return a modified `CarBuilder`, transferring ownership. This enforces immutability within the builder after each method call.

In the below example the `Car2Builder` holds ownership of the `Car2` it is constructing and the methods like `set_color` and `set_engine` take a mutable reference `(&mut self)`, allowing in-place modification of the `Car2` without transferring ownership.
Notice the `.clone()` while constructing the final object, it requires all types to implement the Clone trait. I derived for each type.

{{< code language="rust" isCollapsed="false" >}}
impl Car2Builder {
    pub fn new(equip: Equiped) -> Self {
        Car2Builder {
            car: Car2 {
                equip,
                ..Default::default()
            },
        }
    }

    pub fn set_color(&mut self, color: impl Into<String>) -> &mut Self {
        self.car.color = Some(color.into());
        self
    }

    pub fn set_engine(&mut self, engine: Engine) -> &mut Self {
        self.car.car_engine = Some(engine);
        self
    }

    pub fn build(&mut self) -> Result<Car2> {
        let car_engine = self
            .car
            .car_engine
            .clone()
            .ok_or_else(|| anyhow!("You need to select an Engine type"))?;

        Ok(Car2 {
            equip: self.car.equip.clone(),
            color: Some(self.car.color.clone().unwrap_or("White".to_owned())),
            car_engine: Some(car_engine),
            ..Default::default()
        })
    }
}
{{< /code >}}

The user of the library can utilize the builder patterns to construct the `Car` instance as the examples below.

{{< code language="rust" isCollapsed="false" >}}
fn main() -> Result<()> {
    // Create the Car using consuming Pattern
    let equip = Equiped::EquipedGold;

    let car = CarBuilder::new(equip)
        .set_color("Blue")
        .set_engine(Engine::Electric(Electric::Electric350kW))
        .build()?;

    println!("Consuming Builder Pattern:");
    println!("You selected a {}", car);

    // Create the Car using non-consuming Pattern

    let equip = Equiped::EquipedPlatinum;

    let mut car_builder = Car2Builder::new(equip);

    let car1 = car_builder
        .set_color("Black")
        .set_engine(Engine::Petrol(Petrol::Petrol150HP))
        .build()?;

    let car2 = car_builder
        .set_color("Red")
        .set_engine(Engine::Diesel(Diesel::Diesel250HP))
        .build()?;

    println!("Non-Consuming Builder Pattern:");
    println!("You selected a {}", car1);
    println!("You selected a {}", car2);

    Ok(())
}
{{< /code >}}

and running the code

{{< code language="bash" isCollapsed="false" >}}
$ cargo run

Consuming Builder Pattern:
You selected a Car equiped with EquipedGold, having a Electric(Electric350kW) engine ,and Blue color.

Non-Consuming Builder Pattern:
You selected a Car equiped with EquipedPlatinum, having a Petrol(Petrol150HP) engine ,and Black color.
You selected a Car equiped with EquipedPlatinum, having a Diesel(Diesel250HP) engine ,and Red color.
{{< /code >}}

Choose between the two patterns based on your preference and the specific requirements of your code. The original builder pattern is more aligned with traditional builder patterns, while the modified pattern provides a more mutable and in-place approach.

The complete code for both Go & Rust implementation is available [Here](https://github.com/danrusei/dev-state_blog_code/tree/master/builder_pattern).
