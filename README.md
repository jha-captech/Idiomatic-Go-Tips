# Tips For Writing Idiomatic Go
## Table of content
- [Introduction](#intro)
- [Naming Things](#naming)
- [Errors](#errors)
- [Functions](#functions)
- [Structs and Methods](#structs-and-methods)
- [Interfaces](#interfaces)
- [Concurrency](#concurrency)

<a id="intro"></a>
## Introduction
Disclaimer: these ideas are not my own but rather a collection of general advice from the Go community. Below are some of the resources I have drawn on.

- [100 Go Mistakes and How to Avoid Them](https://100go.co/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Idiomatic Go](https://dmitri.shuralyov.com/idiomatic-go)
- [What's in a name?](https://go.dev/talks/2014/names.slide#1)
- [Go Style Decisions](https://google.github.io/styleguide/go/decisions.html)

<a id="naming"></a>
## 1. Naming Things

1. In Go, objects that have the first letter of their name capitalized are exported and can be accessed from outside of the package in which they are defined. The same goes for struct fields. This is Called MixedCaps naming or pascal case. In contrast, objects that are internal only start with a lowercase letter and use camel case.

    ```go
    var aVar string // not exported
    var AVar string // exported

    func aFunc() { ... } // not exported
    func AFunc() { ... } // exported

    type AStruct struct {
        aVar string // not exported
        AVar string // exported
    }
    ```

1. In Go, acronyms and words that are all capitalized should have constant case. For example `API` should be called `API` if exported and `api` if not. If two acronyms are referenced together such as `XML API`, the exported name would be `XMLAPI` and the un-exported name would be `xmlAPI`. In addition, if an acronym uses mixed case such as `gRPC`, the use of mixed case can be preserved except when it conflicts with exporting. As such, the exported name would be `GRPC` and the un-exported name would be `gRPC`.

1. In go constants are not named any different than other variables.

    ```go
    const aConstantValue string // correct
    const A_CONSTANT_VALUE string // incorrect
    ```

1. Go favors short variable named where ever possible. This also goes for method receivers where a one two letter abbreviation of the given struct is preferred. In order to preserve clarity of the code, longer names should be used for function parameter, especially when the value is a simple type. 

    The general rule of thumb is that the length of a name should be proportional to the size of its scope and inversely proportional to the number of times that it is used within that scope. WHile shorter variable names are preferred, clarity is the most important factor.  

1. It is not encouraged to use the `IInterface` naming convention for structs that is found in other languages. 

<a id="errors"></a>
## 2. Errors

1. Return errors as values, don't use `Panic()` (often). Arguably Go's most noticeable feature is that their is no real `try/catch` functionality (Yes, you can recover from a panic, but that's as common).Because of this, errors are just values returned by functions and should be treated as such. Use `error` to signal that a function can fail. By convention, `error` is the last result parameter.

1. Handle errors by check if an `error` variable is `nil`. Use early returns or guard statements to do this check before continuing with you logic.
   
   Good method:
    ```go
    value, err := doSomething()
    if err != nil {
        return err
    }
    doSomethingElse(value)
    ```
   
   Bad methods:
   ```go
    value, err := doSomething()
    if err != nil {
        return err
    } else {
        doSomethingElse(value)
    }

    // Also bad
    value, err := doSomething()
    if err == nil {
        doSomethingElse(value)
    } 
    return err
    ```

1. Wrap errors before returning them to create a stack trace. This can be done by using the `%w` flag with `fmt.Errorf()`. This makes finding where an error occurred much easier. By convention, use the name of the given function inside of the wrapped error along with a colon.

    ```go
    func foo() error {
        err := doSomething()
        if err != nil {
            return fmt.Errorf("in foo: %w", err) 
        }
        return nil
    }
    ```

1. If you have a situation where you have multiple error checking statements in a function, you can use `defer` with an anonymous function to only wrap an error on exit. Note that 
   
   ```go
   func foo() (err error) {
        defer func() {
            if err != nil {
                err = fmt.Errorf("in foo: %w", err)
            }
        }()
   
        err = doSomething()
        if err != nil {
            return err
        }
		
        err := doSomethingElse()
        if err != nil {
            return err
        }
   
        return nil
    }
   ```

1. There are situations where a panic is needed. In situations like this,  panic is used to indicate that execution  of the current task cannot proceed as something has gone very wrong. In situations like this, prefix functions and methods that can panic with `Must` to indicate that they can panic. 
   
   Good method:
    ```go
    // Good
    func mustDoSomething() {
        panic("Panic!")
    }
    ```
   Bad Method:
   ```go
    // Bad
    func doSomething() {
        panic("Panic!")
    }
    ```

1. Errors(and other variables) can be declared within an `if` statement and also used in that `if` statement. This improves readability and scopes that variable to that block.

   ```go
   if err := doSomething(); err != nil {
        // Handle error and return it
   }
   ```

<a id="functions"></a>
## 3. Functions

1. In Go, functions are documented by placing a comment above their definition. It is recommended to use full sentences with proper capitalization and punctuation

    ```go
    // This function does some stuff.
    func foo() { ... }
    ```

1. In go, utilize early returns and guard statements to reduce complexity nested logic. If you find a situation where you are using an `if/else` statement, try and change the logic to check the else condition first and exit the function early. This leads to a situation where the 'happy path' for the function exits at the bottom of the function.

   Good method:
    ```go
    func greaterThanTen(value int) string {
        if value < 5 {
            return "less than 5"
        }

        if value < 10 {
            return "greater than 5"
        }

        return "greater than 10"
    }
   ````
   
   Bad method:
    ```go
    func greaterThanTen(value int) string {
        if value > 10 {
            return "greater than 10"
        } else if value >= 5 {
            return "greater than 5"
        } else {
            return "less than 5"
        }
    }
    ```

1. Named return values should only be used for documentation purposes and should not be relied on within code. Furthermore, if a named return value is defined, you should not use a naked `return` as this can cause unexpected side effects as the named value will be returned in its current state. Also note that a named return value can be overridden by explicitly returning a different value. 

    Bad use of named return value with a naked return:
    ```go
    func foo(input int) (importantReturnValue int) {
        importantReturnValue = -1
        if input == 0 {
            return // this will return -1 as the value for importantReturnValue
        }
        importantReturnValue += input
        return
    }
    ```
    Good use of a named return value:
      ```go
    func foo(input int) (importantReturnValue int) {
        importantReturnValue = -1
        if input == 0 {
            return 0
        }
        importantReturnValue += input
        return importantReturnValue
    }
    ```

<a id="structs-and-methods"></a>
## 4. Structs and Methods

1. Use struct constructors, especially when validation is needed. This give more control over creating of the object and offers a layer of abstraction. By convention, constructors are named `New<Struct_Name>()` where `Struct_Name` is the name of the given struct.

    ```go
    type myStruct struct {
        value int
    }
    
    // Good
    func newMyStruct(value int) myStruct {
        if value < 0 {
            value = 0
        }
        return myStruct{
            value: value
        }
    }

    myStructInstance := new(10)
    }
    ```

1. Method reivers should have constant naming and type, i.e., don't have one method accept a pointer to a given struct and another accept a value.
    
    ```go
    // Good
    type myStruct struct {
        value int
    }

    func (ms myStruct) methodOne() { ... }

    func (ms myStruct) methodTwo() { ... }
    ```

    ```go
    // Bad
    type myStruct struct {
        value int
    }

    func (ms *myStruct) methodOne() { ... }

    func (ms myStruct) methodTwo() { ... }
    ```

<a id="interfaces"></a>
## 5. Interfaces
Stuff about interfaces

<a id="concurrency"></a>
## 6. Concurrency
Stuff about concurrency

<a id="general"></a>
## 7. General Stuff

1. In Go, to parse a string as a time, `time.Parse()` is used. This function takes a time format as its first argument, and and a string to parse as its second. Layouts must use the reference time `Mon Jan 2 15:04:05 MST 2006` to show the pattern with which to parse a given string.