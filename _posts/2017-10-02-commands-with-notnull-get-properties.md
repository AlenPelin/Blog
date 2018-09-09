---
layout: post
title: Commands with NotNull-enforced get-only properties
---
## Problem

In real world there are cases when a class needs to hold many properties that must be initialized before class is ready to use. For example:

```c#
public class WelcomeGuestCommand
{
    public string Title { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }

    public string Execute()
    {
        return $"Welcome {Title.ToUpper()} {FirstName} {LastName}";
    }
}
```

Having the class defined as above it is easy to come to an error with `NullReferenceException` thrown:

```c#
public void WelcomeTonyStark()
{
    var welcome = new WelcomeGuestCommand
    {
        FirstName = "Tony",
        LastName = "Stark"
    };

    var message = welcome.Execute();
    Console.WriteLine(message);
}
```

A `NullReferenceException` will be thrown because we forgot to initialize `Title` property.

Of course, we can resolve this kind of issue by checking for null every time we use our properties, for instance, in the beginning of every method using guard:

```c#
public string Execute()
{
    // Guard.ArgumentNotNull method actually is a full equivalent of simple 
    // checking value for null and throwing new ArgumentNullException otherwise
    Guard.ArgumentNotNull(Title, nameof(Title));
    Guard.ArgumentNotNull(FirstName, nameof(FirstName));
    Guard.ArgumentNotNull(LastName, nameof(LastName));

    return $"Welcome {Title.ToUpper().Trim()} {FirstName.Trim()} {LastName.Trim()}";
}
```

Or, it can be solved via null-checks in properties:

```c#
public class WelcomeGuestCommand
{
    private string _Title;
    
    [NotNull]
    public string Title
    { 
        get => _Title ?? throw new ArgumentNotNullException(nameof(Title));
        set => _Title = value ?? throw new ArgumentNotNullException(nameof(value));
    }
    
    private string _FirstName;
    
    [NotNull]
    public string FirstName
    { 
        get => _FirstName ?? throw new ArgumentNotNullException(nameof(FirstName));
        set => _FirstName = value ?? throw new ArgumentNotNullException(nameof(value));
    }
    
    private string _LastName;
    
    [NotNull]
    public string LastName
    { 
        get => _LastName ?? throw new ArgumentNotNullException(nameof(LastName));
        set => _LastName = value ?? throw new ArgumentNotNullException(nameof(value));
    }
}
```

These are valid solution, but it is not elegant at all, especially when certain properties must always be initialized.

## Proposed Solution

Thanks to R# and C# 7 we now have can use the following construct (well, in previous versions it worked too, but not that elegant):

```c#
// welcome guest interface describes command
public interface IWelcomeGuest
{
    string Title { get; }
    string FirstName { get; }
    string LastName { get; }
    
    [NotNull]
    string Execute();
}

// our command exposes NotNull on get-only properties which is valid
// because they can only be initialized in constructor and we can
// ensure their not-null values using "?? throw ex" C# 7 syntax
public class WelcomeGuestCommand : IWelcomeGuest
{
    // marking with NotNull helps R# to assist when property is not
    // initialized in constructor (and you can configure it to treat
    // that as error) and use R# null-ref analysis.
    [NotNull]
    public string Title { get; }
    
    [NotNull]
    public string FirstName { get; }
    
    [NotNull]
    public string LastName { get; }
    
    public WelcomeGuestCommand([NotNull] IWelcomeGuest args)
    {
        Title = args?.Title 
            ?? throw new ArgumentNullException(nameof(args.Title));
        
        FirstName = args?.FirstName 
            ?? throw new ArgumentNullException(nameof(args.FirstName));
        
        LastName = args?.LastName 
            ?? throw new ArgumentNullException(nameof(args.LastName));
    }

    public string Execute()
    {
        return $"Welcome {Title.ToUpper()} {FirstName} {LastName}";
    }
}

// welcome guest class is used to pass values to constructor
// and to invoke default command implementation
public class WelcomeGuest : IWelcomeGuest
{
    public string Title { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    
    // executes default implementation
    public string Execute()
    {
        var command = new WelcomeGuestCommand(this);
        var result = command.Execute();
        
        return result;
    }
}

public class Tests
{
    [Fact]
    public void WelcomeTonyStark()
    {
        // the ArgumentNullException will be thrown at the moment
        // when the command is constructed: Value cannot be null.
        // Parameter name: Title
        var welcome = new WelcomeGuest
        {
            FirstName = "Tony",
            LastName = "Stark"
        };

        // so the Execute call will fail with human-friendly exception details
        Assert.Equal("Welcome MR Tony Stark", welcome.Execute());
    }
}
```

## Benefits

There are a number of benefits that the approach gives:

### Read-Only Properties

You can use get-only properties to ensure command does not have state.

### Static NotNull analysis

You can rely on R# NotNull static analysis as all necessary properties are marked not-null and cannot change (see above).

### Copying constructor

You can use either arguments or command to create another command with the same parameters.

```c#
var args = new WelcomeGuest
{
    FirstName = "Tony",
    LastName = "Stark"
};

var welcome1 = new WelcomeGuestCommand(args);
var welcome2 = new WelcomeGuestCommand(welcome1);
```
