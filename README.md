<div align="center">

<img src="assets/icon.png" alt="drawing" width="700px"/></br>

[![NuGet](https://img.shields.io/nuget/v/erroror.svg)](https://www.nuget.org/packages/erroror)

[![Build](https://github.com/amantinband/error-or/actions/workflows/build.yml/badge.svg)](https://github.com/amantinband/error-or/actions/workflows/build.yml) [![publish ErrorOr to nuget](https://github.com/amantinband/error-or/actions/workflows/publish.yml/badge.svg)](https://github.com/amantinband/error-or/actions/workflows/publish.yml)

[![GitHub contributors](https://img.shields.io/github/contributors/amantinband/error-or)](https://GitHub.com/amantinband/error-or/graphs/contributors/) [![GitHub Stars](https://img.shields.io/github/stars/amantinband/error-or.svg)](https://github.com/amantinband/error-or/stargazers) [![GitHub license](https://img.shields.io/github/license/amantinband/error-or)](https://github.com/amantinband/error-or/blob/main/LICENSE)
[![codecov](https://codecov.io/gh/amantinband/error-or/branch/main/graph/badge.svg?token=DR2EBIWK7B)](https://codecov.io/gh/amantinband/error-or)
---

### A simple, fluent discriminated union of an error or a result.

`dotnet add package ErrorOr`

</div>

- [Give it a star ⭐!](#give-it-a-star-)
- [Getting Started](#getting-started)
  - [Single Error](#single-error)
    - [This 👇🏽](#this-)
    - [Turns into this 👇🏽](#turns-into-this-)
  - [Multiple Errors](#multiple-errors)
- [A more practical example](#a-more-practical-example)
  - [Dropping the exceptions throwing logic](#dropping-the-exceptions-throwing-logic)
- [Usage](#usage)
  - [Creating an `ErrorOr<result>`](#creating-an-errororresult)
    - [From Value](#from-value)
    - [From Single Error](#from-single-error)
    - [From List of Errors](#from-list-of-errors)
  - [Checking if the `ErrorOr<result>` is an error](#checking-if-the-errororresult-is-an-error)
  - [Accessing the `ErrorOr<result>` result](#accessing-the-errororresult-result)
    - [Accessing the Value](#accessing-the-value)
    - [Accessing the List of Errors](#accessing-the-list-of-errors)
    - [Accessing the First Error](#accessing-the-first-error)
  - [Performing actions based on the `ErrorOr<result>` result](#performing-actions-based-on-the-errororresult-result)
    - [`Match`](#match)
    - [`MatchFirst`](#matchfirst)
    - [`Switch`](#switch)
    - [`SwitchFirst`](#switchfirst)
- [How Is This Different From `OneOf<T0, T1>` or `FluentResults`?](#how-is-this-different-from-oneoft0-t1-or-fluentresults)
- [Contribution](#contribution)
- [Credits](#credits)
- [License](#license)

# Give it a star ⭐!

Loving it? Show your support by giving this project a star!

# Getting Started

## Single Error

### This 👇🏽

```csharp
User GetUser(Guid id = default)
{
    if (id == default)
    {
        throw new ValidationException("Id is required");
    }

    return new User(Name: "Amichai");
}
```

```csharp
try
{
    var user = GetUser();
    Console.WriteLine(user.Name);
}
catch (Exception e)
{
    Console.WriteLine(e.Message);
}
```

### Turns into this 👇🏽

```csharp
ErrorOr<User> GetUser(Guid id = default)
{
    if (id == default)
    {
        return Error.Validation("Id is required");
    }

    return new User(Name: "Amichai");
}
```

```csharp
errorOrUser.SwitchFirst(
    user => Console.WriteLine(user.Name),
    error => Console.WriteLine(error.Description));
```

## Multiple Errors

Internally, the `ErrorOr` object has a list of `Error`s, so if you have multiple errors, you don't need to compromise and have only the first one.

```csharp
public class User
{
    public string Name { get; }

    private User(string name)
    {
        Name = name;
    }

    public static ErrorOr<User> Create(string name)
    {
        List<Error> errors = new();

        if (name.Length < 2)
        {
            errors.Add(Error.Validation(description: "Name is too short"));
        }

        if (name.Length > 100)
        {
            errors.Add(Error.Validation(description: "Name is too long"));
        }

        if (string.IsNullOrWhiteSpace(name))
        {
            errors.Add(Error.Validation(description: "Name cannot be empty or whitespace only"));
        }

        if (errors.Count > 0)
        {
            return errors;
        }

        return new User(firstName, lastName);
    }
}
```

```csharp
public async Task<ErrorOr<User>> CreateUser(string name)
{
    if (await _userRepository.GetAsync(name) is User user)
    {
        return Error.Conflict("User already exists");
    }

    var errorOrUser = User.Create("Amichai");

    if (errorOrUser.IsError)
    {
        return errorOrUser.Errors;
    }

    await _userRepository.AddAsync(errorOrUser.Value);
    return errorOrUser.Value;
}
```

# A more practical example

```csharp
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetUser(Guid Id)
{
    var getUserQuery = new GetUserQuery(Id);

    ErrorOr<User> getUserResponse = await _mediator.Send(getUserQuery);

    return getUserResponse.Match(
        user => Ok(_mapper.Map<UserResponse>(user)),
        errors => ValidationProblem(errors.ToModelStateDictionary()));
}
```

## Dropping the exceptions throwing logic

You have validation logic such as `MediatR` behaviors, you can drop the exceptions throwing logic and simply return a list of errors from the pipeline behavior

```csharp
public class ValidationBehavior<TRequest, TResult> : IPipelineBehavior<TRequest, ErrorOr<TResult>>
    where TRequest : IRequest<ErrorOr<TResult>>
{
    private readonly IValidator<TRequest>? _validator;

    public ValidationBehavior(IValidator<TRequest>? validator = null)
    {
        _validator = validator;
    }

    public async Task<ErrorOr<TResult>> Handle(
        TRequest request,
        CancellationToken cancellationToken,
        RequestHandlerDelegate<ErrorOr<TResult>> next)
    {
        if (_validator == null)
        {
            return await next();
        }

        var validationResult = _validator.Validate(request);

        if (validationResult.IsError)
        {
            return validationResult.Errors
               .ConvertAll(validationFailure => Error.Validation(
                   code: validationFailure.PropertyName,
                   description: validationFailure.ErrorMessage));
        }

        return await next();
    }
}
```

# Usage

## Creating an `ErrorOr<result>`

There are implicit converters from `TResult`, `Error`, `List<Error>` to `ErrorOr<TResult>`

### From Value

```csharp
ErrorOr<int> result = 5;
```

```csharp
public ErrorOr<int> GetValue()
{
    return 5;
}
```

### From Single Error

```csharp
ErrorOr<int> result = Error.Unexpected();
```

```csharp
public ErrorOr<int> GetValue()
{
    return Error.Unexpected();
}
```

### From List of Errors

```csharp
ErrorOr<int> result = new List<Error> { Error.Unexpected(), Error.Validation() };
```

```csharp
public ErrorOr<int> GetValue()
{
    return new List<Error>
    {
        Error.Unexpected(),
        Error.Validation()
    };
}
```

## Checking if the `ErrorOr<result>` is an error

```csharp
if (errorOrResult.IsError)
{
    // errorOrResult is an error
}
```

## Accessing the `ErrorOr<result>` result

### Accessing the Value

```csharp
ErrorOr<int> result = 5;

var value = result.Value;
```

### Accessing the List of Errors

```csharp
ErrorOr<int> result = new List<Error> { Error.Unexpected(), Error.Validation() };

List<Error> value = result.Errors; // List<Error> { Error.Unexpected(), Error.Validation() }
```

```csharp
ErrorOr<int> result = Error.Unexpected();

List<Error> value = result.Errors; // List<Error> { Error.Unexpected() }
```

### Accessing the First Error

```csharp
ErrorOr<int> result = new List<Error> { Error.Unexpected(), Error.Validation() };

Error value = result.FirstError; // Error.Unexpected()
```

```csharp
ErrorOr<int> result = Error.Unexpected();

Error value = result.FirstError; // Error.Unexpected()
```

## Performing actions based on the `ErrorOr<result>` result

### `Match`

Actions that return a value on the value or list of errors

```csharp
string foo = errorOrString.Match(
    value => value,
    errors => $"{errors.Count} errors occurred.");
```

### `MatchFirst`

Actions that return a value on the value or first error

```csharp
string foo = errorOrString.MatchFirst(
    value => value,
    firstError => firstError.Description);
```

### `Switch`

Actions that don't return a value on the value or list of errors

```csharp
errorOrString.Switch(
    value => Console.WriteLine(value),
    errors => Console.WriteLine($"{errors.Count} errors occurred."));
```

### `SwitchFirst`

Actions that don't return a value on the value or first error

```csharp
errorOrString.SwitchFirst(
    value => Console.WriteLine(value),
    firstError => Console.WriteLine(firstError.Description));
```

# How Is This Different From `OneOf<T0, T1>` or `FluentResults`?

It's similar to the others, just aims to be more intuitive and fluent.
If you find yourself typing `OneOf<User, DomainError>` or `Result.Fail<User>("failure")` again and again, you might enjoy the fluent API of `ErrorOr<User>` (and it's also faster).

# Contribution

If you have any questions, comments, or suggestions, please open an issue or create a pull request 🙂

# Credits

- [OneOf](https://github.com/mcintyre321/OneOf/tree/master/OneOf) - An awesome library which provides F# style discriminated unions behavior for C#

# License

This project is licensed under the terms of the [MIT](https://github.com/mantinband/error-or/blob/main/LICENSE) license.
