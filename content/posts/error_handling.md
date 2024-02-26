---
title: "Practical guide to Error Handling in Rust"
date: 2024-02-26
draft: false
categories: ["Rust"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

Effective error handling ensures that a program can gracefully handle unexpected situations and errors, making the software more robust and reliable. Well-designed error messages help users understand what went wrong and how to correct it, and contribute to the overall user experience using the library or the API.

In this article I'll gradually go through a number of options of handling errors in Rust and try to explain the benefits of using a method vs the other.

## Using Unwrap method

If you are experimenting or writing a simple program for personal use, you might be mostly interested by the happy path and handling the errors may not be so important at this stage. Rust provides a means for expeditious prototyping using the `Unwrap` method on `Option<T>` and `Result<T, E>` return types.

As a simple example, let's asume that we are building a service, part of an online grocery store that validates the customer inputs and confirm the order. The program expects a file_name as argument, read the content of the file, parse the fields ensuring that each line contain the mandatory fields/collumns. If the parsing of the file succeeds, then the order is confirmed to the customer, otherwise the program panics.

The below code is partial, the entire code, that can be found [HERE](https://github.com/danrusei/dev-state_blog_code/blob/master/error_handling/src/1_order_unwrap.rs)

{{< code language="rust" isCollapsed="false" >}}

.# [derive(Debug)]
struct UserCommand {
    product: String,
    quantity: u32,
    delivery_date: NaiveDate,
}

fn main() {
    let file_name = env::args().nth(1).unwrap();
    let content = fs::read_to_string(file_name).unwrap();

    let mut commands: Vec<UserCommand> = Vec::new();

    for line in content.lines() {
        let mut parts = line.split_whitespace();
        let product = parts.next().unwrap().to_string();

        let quant = parts.next().unwrap();
        let quantity = quant.trim().parse::<u32>().unwrap();

        let d_date = parts.next().unwrap();
        let delivery_date = NaiveDate::parse_from_str(d_date.trim(), "%d.%m.%Y").unwrap();

        let command = UserCommand {
            product,
            quantity,
            delivery_date,
        };

        commands.push(command);
    }

    println!("\nYour command was processed and it is ready for delivery. The ordered items:\n");

    commands.iter().for_each(|cmd| println!(" * {} ", cmd));
}
{{< /code >}}

The `unwrap` method is a simple way to extract the value from a `Result<T, E>` or `Option<T>`. It assumes that the operation was successful and retrieves the value if it is present, or panics if it encounters an error or a None value. There are multiple reasons for the above program to panic, but first let's see the happy path when everything went well.

The program reads the file that contain the customer order:

{{< code language="bash" isCollapsed="false" >}}
$ cat order.txt
carrots 5 12.05.2024
salads 4 12.05.2024
broccoli 3 14.05.2024
spinach 6 14.05.2024
{{< /code >}}

and the result of a succesfull execution of the code:

{{< code language="bash" isCollapsed="false" >}}
$ cargo run --bin order_unwrap order.txt

Your command was processed and it is ready for delivery. The ordered items:

* 5 carrots - to be delivered on 2024-05-12
* 4 salads - to be delivered on 2024-05-12
* 3 broccoli - to be delivered on 2024-05-14
* 6 spinach - to be delivered on 2024-05-14

{{< /code >}}

Now I'll make 2 mistakes, first the command is executed without providing the file as argument. Second the parsing of the quantity to `u32` fails due to erroneous user input.  

{{< code language="bash" isCollapsed="false" >}}
$ cargo run --bin order_unwrap

thread 'main' panicked at src/1_order_unwrap.rs:12:40:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

=============================

$ cargo run --bin order_unwrap order.txt

thread 'main' panicked at src/1_order_unwrap.rs:22:52:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

{{< /code >}}

In the first call, the panic occurred because the `unwrap` method was called on an `Option<T>` that was in a `None` state. In the second call, the panic occurred because the unwrap method was called on a `Result<T, E>` that was in an Err (error) state. The specific error is a `ParseIntError` with the kind `InvalidDigit`. This indicates a failure to parse a string as an integer due to an invalid digit.

### Understand Result and Option enums

Let's investigate the return types to the called functions:

* env::args() .nth(1)             -> returns an `Option<String>`
* fs::read_to_string(&file_name)  -> returns `io::Result<String>`,  which is a type alias of `result::Result<T, Error>`;
* parts.next()                    -> returns `Option<&'a str>`
* quant.trim().parse::<u32>()     -> returns `Result<u32, ParseIntError>`
* NaiveDate::parse_from_str(d_date.trim(), "%d.%m.%Y") -> returns `Result<NaiveDate, chrono::ParseError>`

`Result<T, E>` and `Option<T>` are two important enums in Rust that are used to represent the result of computations that may fail or values that may or may not be present. In practice, you'll often use `Result<T,E>` for functions that can produce an error, and `Option<T>` for functions that may or may not return a value.

#### Result Enum

The Result enum is defined as follows:

{{< code language="rust" isCollapsed="false" >}}
enum Result<T, E> {
 Ok(T),
 Err(E),
}
{{< /code >}}

Result<T, E> has two variants: Ok and Err.

* Ok(T) -> represents the successful result with the associated value of type T.
* Err(E) ->  represents an error with the associated value of type E.
Result is commonly used for operations that may fail, and it ensures that error handling is explicit in Rust. For example, when opening a file, the Result<T, E> type is used to indicate whether the operation was successful (Ok) or resulted in an error (Err). The error type (E) can be any type that describes the error, such as a string or a custom error enum.

#### Option Enum

The Option enum is defined as follows:

{{< code language="rust" isCollapsed="false" >}}
enum Option<T> {
 Some(T),
 None,
}
{{< /code >}}

Option has two variants: `Some` and `None`.

* `Some(T)` represent a value of type T.
* `None` represents the absence of a value.

Option is commonly used when a value might be missing or is optional. For example, when looking up a value in a collection, the result can be wrapped in an Option<T>. If the value is present, it's wrapped in `Some`; otherwise, it's `None`.

## Using Expect method

The expect method is similar to unwrap, but it allows you to provide a custom error message. This can be helpful for debugging or providing more meaningful error information.

The below code is partial, the entire code, that can be found [HERE](https://github.com/danrusei/dev-state_blog_code/blob/master/error_handling/src/2_order_expect.rs)

{{< code language="rust" isCollapsed="false" >}}
let file_name = env::args()
        .nth(1)
        .expect("Please provide a file name as a command-line argument.");
    let content = fs::read_to_string(&file_name).expect("Error reading the file");

    let mut commands: Vec<UserCommand> = Vec::new();

    for line in content.lines() {
        let mut parts = line.split_whitespace();
        let product = parts
            .next()
            .expect("Missing product information")
            .to_string();

        let quant = parts.next().expect("Missing quantity information");
        let quantity = quant
            .trim()
            .parse::<u32>()
            .expect("Invalid quantity format, expecting integer");

        let d_date = parts.next().expect("Missing delivery date information");
        let delivery_date = NaiveDate::parse_from_str(d_date.trim(), "%d.%m.%Y")
            .expect("Invalid date format, should be %d.%m.%Y");

            .......
    }
{{< /code >}}

Running the code without providing the file as argument, and with erroneous user input for the quantity:

{{< code language="bash" isCollapsed="false" >}}
cargo run --bin order_expect

thread 'main' panicked at src/2_order_expect.rs:14:10:
Please provide a file name as a command-line argument.
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

=============================

cargo run --bin order_expect order.txt

thread 'main' panicked at src/2_order_expect.rs:30:14:
Invalid quantity format, expecting integer: ParseIntError { kind: InvalidDigit }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
{{< /code >}}

The `expect` method, as well as the `unwrap` method are provided by the `Option<T>` and `Result<T, E>` types for extracting the value when it is `Some` or `Ok`. `expect` is similar to `unwrap`, in the way that it cause the program to crash if the variant is `None` for `Option<T>` or `Err` for `Result<T, E>`. However `expect` provides the additional benefit of allowing the programmer to provide a custom error message when a panic occurs. This makes it useful when you want to provide more context about the error.

## Using Result and Option combinators and return Error as String

In this version, the main function returns a `Result<(), String>`, where `()` represents a unit type. The `ok_or` and `map_err` methods are used to convert `Option<T>` and `Result<T, E>` to the desired error type and the `?` operator to propagate errors to the caller. This ensures that any error during file reading, parsing, or processing is collected and returned as an `Err` variant with a descriptive error message.

The below code is partial, the entire code can be found [HERE](https://github.com/danrusei/dev-state_blog_code/blob/master/error_handling/src/3_order_combinators.rs)

{{< code language="rust" isCollapsed="false" >}}
 fn main() -> Result<(), String> {
 let file_name = env::args()
        .nth(1)
        .ok_or("Please provide a file name as a command-line argument.".to_string())?;
    let content =
        fs::read_to_string(&file_name).map_err(|e| format!("Error reading the file: {}", e))?;

    let mut commands: Vec<UserCommand> = Vec::new();

    for line in content.lines() {
        let mut parts = line.split_whitespace();
        let product = parts
            .next()
            .ok_or("Missing product information".to_string())?
            .to_string();

        let quant = parts
            .next()
            .ok_or("Missing quantity information".to_string())?;
        let quantity = quant
            .trim()
            .parse::<u32>()
            .map_err(|e| format!("Invalid quantity format: {}", e))?;

        let d_date = parts
            .next()
            .ok_or("Missing delivery date information".to_string())?;
        let delivery_date = NaiveDate::parse_from_str(d_date.trim(), "%d.%m.%Y")
            .map_err(|e| format!("Invalid date format: {}", e))?;
  .............
    }
 }
{{< /code >}}

Returning an `Err` variant with a descriptive error message is generally considered better than using `expect` for few reasons:

* **Error Propagation**  Using `Result<T, E>` allows for more granular control over error handling and propagation. By using combinators like `ok_or` and `map_err`, can be handled different error cases at each step and provide specific error messages. With `expect`, a single failure at any step would cause the entire program to panic.
* **Custom Error Messages** Using `map_err` allows you to customize error messages for each step. This can be crucial for debugging and understanding the nature of the error. `expect` provides a static error message, which might not be as informative or context-specific
* **Avoiding Panics** Panicking is generally discouraged unless you are dealing with unrecoverable errors.
* **Error Handling in Calling Code** By returning a `Result<T, E>`, the calling code has the opportunity to handle errors in a way that makes sense for the application. It can decide whether to log the error, display a user-friendly message, or take other appropriate actions.

Running the code without providing the file as argument, and with erroneous user input for the quantity. Notice this time that the program do not panics anymore:

{{< code language="bash" isCollapsed="false" >}}
$ cargo run --bin order_combinators

Error: "Please provide a file name as a command-line argument."

=============================

$ cargo run --bin order_combinators order.txt

Error: "Invalid quantity format: invalid digit found in string"

{{< /code >}}

## Using [anyhow crate](https://crates.io/crates/anyhow)

In this version, the `anyhow::Result` type is used instead of `Result<(), String>`. With `anyhow::anyhow!` macro you can create error instances with a more concise syntax. The `anyhow` crate provides a more flexible and ergonomic way of handling errors, making it easier to work with different error types and improving the readability of the code.

The `Context` trait is used to provide additional context information for the errors, making it easier to understand where the errors occurred. The `with_context` method is used to attach context information to the errors.

The below code is partial, the entire code can be found [HERE](https://github.com/danrusei/dev-state_blog_code/blob/master/error_handling/src/4_order_anyhow.rs)

{{< code language="rust" isCollapsed="false" >}}
use anyhow::{anyhow, Context};

fn main() -> anyhow::Result<()> {
    let file_name = env::args()
        .nth(1)
        .ok_or_else(|| anyhow!("Please provide a file name as a command-line argument."))?;
    let content = fs::read_to_string(&file_name)
        .with_context(|| format!("Error reading the file: {}", &file_name))?;

    let mut commands: Vec<UserCommand> = Vec::new();

    for (line_num, line) in content.lines().enumerate() {
        let mut parts = line.split_whitespace();
        let product = parts
            .next()
            .ok_or_else(|| anyhow!("Missing product information at line {}", line_num + 1))?
            .to_string();

        let quant = parts
            .next()
            .ok_or_else(|| anyhow!("Missing quantity information at line {}", line_num + 1))?;
        let quantity = quant
            .trim()
            .parse::<u32>()
            .with_context(|| format!("Invalid quantity format at line {}", line_num + 1))?;

        let d_date = parts
            .next()
            .ok_or_else(|| anyhow!("Missing delivery date information at line {}", line_num + 1))?;
        let delivery_date = NaiveDate::parse_from_str(d_date.trim(), "%d.%m.%Y")
            .with_context(|| format!("Invalid date format at line {}", line_num + 1))?;

 .............
    }
 }
{{< /code >}}

Both `ok_or` and `ok_or_else` transforms the `Option<T>` into a `Result<T, E>`, mapping `Some(v)` to `Ok(v)` and `None` to `Err(err)`.
However arguments passed to `ok_or` are eagerly evaluated, instead of `ok_or_else` which is lazily evaluated. If you are passing the result of a function call, it is recommended to use `ok_or_else`.

Running the code without providing the file as argument, and with erroneous user input for quantity. Notice the additional context provided by `anyhow`.

{{< code language="bash" isCollapsed="false" >}}
$ cargo run --bin order_anyhow

Error: Please provide a file name as a command-line argument.

=============================

$ cargo run --bin order_anyhow order.txt

Error: Invalid quantity format at line 3
Caused by:
    invalid digit found in string

{{< /code >}}

## Using My Custom Error type

If you are designing a library or an API you may want to propagate also the application specific error types and the library consumers to see a predictible set of errors.

Some advantages of using Custom Error type vs anyhow:

* **Control and Flexibility** With a custom error enum, you have fine-grained control over the types of errors the application can encounter.
* **Application-Specific Error Types** Define error types that are specific to the application / library domain, providing meaningful and contextual error information.
* **Predictable API**  Consumers of the API or library see a clear and predictable set of error types. This can make it easier for them to understand and handle errors.

In the below example:

* I've created a custom error enum `MyError` that represents different error cases.
* I implemented the `fmt::Display` trait for `MyError` to provide custom error messages.
* Implement `std::error::Error` trait for `MyError`. The `std::error::Error` trait helps us to convert any type to an `Err` type.

The below code is partial, the entire code can be found [HERE](https://github.com/danrusei/dev-state_blog_code/blob/master/error_handling/src/5_order_custom_error.rs)

{{< code language="rust" isCollapsed="false" >}}

.# [derive(Debug)]
enum MyError {
    CommandLineArgs,
    FileReadError(std::io::Error),
    ParsingError(String),
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            MyError::CommandLineArgs => {
                write!(f, "Please provide a file name as a command-line argument.")
            }
            MyError::FileReadError(err) => write!(f, "Error reading the file: {}", err),
            MyError::ParsingError(msg) => write!(f, "Parsing error: {}", msg),
        }
    }
}

impl std::error::Error for MyError {}

{{< /code >}}

And the main program now returns `Result<(), MyError>`:

{{< code language="rust" isCollapsed="false" >}}
fn main() -> Result<(), MyError> {
    let file_name = env::args().nth(1).ok_or(MyError::CommandLineArgs)?;
    let content = fs::read_to_string(&file_name).map_err(MyError::FileReadError)?;

    let mut commands: Vec<UserCommand> = Vec::new();

    for (line_num, line) in content.lines().enumerate() {
        let mut parts = line.split_whitespace();

        let product = parts
            .next()
            .ok_or(MyError::ParsingError(format!(
                "Missing product information in line {}",
                line_num + 1
            )))?
            .to_string();

        let quant = parts.next().ok_or(MyError::ParsingError(format!(
            "Missing quantity information in line {}",
            line_num + 1
        )))?;
        let quantity = quant.trim().parse::<u32>().map_err(|e| {
            MyError::ParsingError(format!(
                "Invalid quantity format in line {}: {}",
                line_num + 1,
                e
            ))
        })?;

        let d_date = parts.next().ok_or(MyError::ParsingError(format!(
            "Missing delivery date information in line {}",
            line_num + 1
        )))?;
        let delivery_date = NaiveDate::parse_from_str(d_date.trim(), "%d.%m.%Y").map_err(|e| {
            MyError::ParsingError(format!(
                "Invalid date format in line {}: {}",
                line_num + 1,
                e
            ))
        })?;

        ........................
    }
}
{{< /code >}}

Running the code without providing the file as argument, and with erroneous user input for quantity.

{{< code language="bash" isCollapsed="false" >}}
$ cargo run --bin order_custom_error

Error: CommandLineArgs

=============================

$ cargo run --bin order_custom_error order.txt

Error: ParsingError("Invalid quantity format in line 3: invalid digit found in string")
{{< /code >}}

If you prefer a more fine-grained control over your error types and want to provide specific information for different error scenarios, a custom error enum might be more suitable. You might also consider a hybrid approach, where you use a custom error enum for domain-specific errors and anyhow for more generic or infrastructure-related errors.

## Using [thiserror crate](https://crates.io/crates/thiserror)

Using `thiserror` crate makes the code more concise and maintains the benefits of providing custom error messages for each error variant.

In the bellow example:

* `thiserror` is added as dependency in Cargo.toml.
* the `Error` trait is derived for `MyError` enum using `#[derive(Error)]`.
* the `#[error("...")]` attribute specify custom error messages for each variant of the enum.
* `#[from]` attribute is used to automatically convert `std::io::Error` to `MyError::FileReadError`.
* the `fmt::Display` trait implementation for `MyError` is automatically derived by `thiserror`.

The below code is partial, the entire code can be found [HERE](https://github.com/danrusei/dev-state_blog_code/blob/master/error_handling/src/6_order_thiserror.rs)

{{< code language="rust" isCollapsed="false" >}}
use thiserror::Error;

.# [derive(Debug, Error)]
enum MyError {
    #[error("Please provide a file name as a command-line argument.")]
    CommandLineArgs,

    #[error("Error reading the file: {0}")]
    FileReadError(#[from] std::io::Error),

    #[error("Parsing error: {0}")]
    ParsingError(String),
}
{{< /code >}}

The main program suffered no changes, as `thiserror` was only used to define my Custom error.

Running the code once again without providing the file as argument, and with erroneous user input for quantity.

{{< code language="bash" isCollapsed="false" >}}
$ cargo run --bin order_thiserror

Error: CommandLineArgs

=============================

$ cargo run --bin order_thiserror order.txt

Error: ParsingError("Invalid quantity format in line 3: invalid digit found in string")
{{< /code >}}

If you are still wondering whether to use `thiserror` or `anyhow`, the author of the crates provided some guidance:

*Use thiserror if you care about designing your own dedicated error type(s) so that the caller receives exactly the information that you choose in the event of failure. This most often applies to library-like code. Use Anyhow if you don't care what error type your functions return, you just want it to be easy. This is common in application-like code*

I hope this article was fun and informative and gave you some alternatives on how to handle errors in Rust.
