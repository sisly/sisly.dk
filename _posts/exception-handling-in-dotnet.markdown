http://ericlippert.com/2014/03/03/living-with-unchecked-exceptions/
http://ericlippert.com/2014/03/06/living-with-unchecked-exceptions-part-two/

>  think the whole notion of "handling" exceptions is a bit of a fool's game. I can probably count on the fingers of one hand the times where I've been able to catch a specific exception and then do something intelligent with it. 99% of the time you should either catch everything or catch nothing. When an exception of any type occurs, rewind to a stable state and then either abort or continue.

I tend to catch nothing when it clearly represents programmer error. For example, a `NullReferenceException`. Compare that to a `FileNotFoundException` which you might expect if a file you expect happens to be deleted. While you could argue that's a programmer error in that the code should check for file existence, it's still possible for the file to be deleted between the check and the access. Thus `FileNotfoundException` is a legit runtime exception in my book and definitely catchable. You can recover from that. Some exceptions are not programmer errors and not recoverable. For example, an `OutOfMemoryException` could happen because of memory fragmentation that's no falt of the developer but a result of a particular user's environment. There's not much you can do with that one.

But not everything fits into these nice groupings. For example, I firmly believe `ArgumentException` should represent programmer error and not, for example, user input error. And thus I think an application should crash immediately when it encounters such an error so it's noticeable and gets fixed faster than if it's hidden and potentially corrupting data. `ArgumentException` should never be used to represent user input error. For example, if you ask a user for her phone number, and she types in something obviously not a phone number, your code should not throw an `ArgumentException`. Your code should validate the data and expect that user input might not be correctly formatted.

Unfortunately, the .NET framework doesn't always follow this line of thinking. `Process.GetProcessById` with an ID that doesn't map to a process throws an `ArgumentException`. Boo! I might expect that for `-2`. But not for `42`. The way I think about it is this, is `42` a correctly formatted argument for `GetProcessById`? If an argument is valid argument for one call of a method, I expect it to be for every call. If the argument doesn't happen to match a running process, I'd expect some other exception to be thrown. The argument was valid for the method, but it just happens to not match a running process. So throw a `ProcessNotFoundException` or something for Pete's sake!

## Executing 3rd party extensions

Say you have a plug-in model. These are bits of code written by others that are dynamically loaded and executed. Your code ostensibly has no clue what they will do. Ideally, plugins are independent of each other and if one fails, it's fine to continue running the application. It would be insanity to run these and not try to recover from their exceptions. Typically you might do the following:

```csharp
foreach (var plugin in plugins.ToList()) {
    try {
        plugin.DoSomething();
    }
    catch (Exception e) {
        // Report the exception to some log.
        plugins.UnRegister(plugin);
    }
}
```

But hey! FxCop will admonish you. Catching `Exception` is a bad practice. And in way, I agree. What you really want to do is catch all exceptions _except_ those you shouldn't. Huh? Well there's a whole class of exceptions you shouldn't ever catch. And there's a whole class of exceptions you _can't_ catch (like `StackOverflowException`).



```csharp
/// <summary>
/// Represents exceptions we should never attempt to catch and ignore.
/// </summary>
/// <param name="exception">The exception being thrown.</param>
/// <returns></returns>
public static bool IsCriticalException(this Exception exception)
{
    if (exception == null)
    {
        throw new ArgumentNullException("exception");
    }

    return exception.IsFatalException()
        || exception is AppDomainUnloadedException
        || exception is BadImageFormatException
        || exception is CannotUnloadAppDomainException
        || exception is InvalidProgramException
        || exception is NullReferenceException
        || exception is ArgumentException;
}

/// <summary>
/// Represents exceptions we should never attempt to catch and ignore when executing third party plugin code. 
/// This is not as extensive as IsCriticalException.
/// </summary>
/// <param name="exception">The exception being thrown.</param>
/// <returns></returns>
public static bool IsFatalException(this Exception exception)
{
    if (exception == null)
    {
        throw new ArgumentNullException("exception");
    }

    return exception is StackOverflowException
        || exception is OutOfMemoryException
        || exception is ThreadAbortException
        || exception is AccessViolationException;
}
```

So my plugin code becomes:

```csharp
foreach (var plugin in plugins.ToList()) {
    try {
        plugin.DoSomething();
    }
    catch (Exception e) {
        if (e.IsFatalException()) throw; // Crash the app.
        
        // Report the exception to some log.
        plugins.UnRegister(plugin);
    }
}
```


* [Examine an exception in a Visual Studio debugger session](http://haacked.com/archive/2006/06/07/ExamineAnExceptionInACatchBlock.aspx/)
* [Exception Handling Mistakes](http://haacked.com/archive/2006/01/09/FinallyBlockDoesNotRequireExceptionClause.aspx/)