---
title: "Unit testing with Xamarin.Forms' DependencyService"
date: 2016-07-20
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

Xamarin.Forms ships with a built-in [Service Locator](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ff648968(v=pandp.10)?redirectedfrom=MSDN), called [DependencyService](https://docs.microsoft.com/en-gb/xamarin/xamarin-forms/app-fundamentals/dependency-service/), which allows us to register and resolve dependencies. Typically, we use this to enable accessing and invoking platform specific logic in our shared code. The DependencyService is very convenient and easy to use, but it does come with a limitation. Because the `DependencyService` is part of the Xamarin.Forms framework, it expects that the framework has been initialized prior to using it. This presents a problem when we're trying to unit test some code that uses the `DependencyService`.

Consider a typical cross platform view model class :

```csharp
public class MainViewModel
{
    public string Data { get; set; }
    public void LoadData()
    {
        var database = DependencyService.Get<ISQLite>();
        // Use the database
    }
}
```

If we wanted to write a unit test for this, it might look like this :

```csharp
[TestFixture]
public class TestMainViewModel
{
    [OneTimeSetUp]
    public void Setup()
    {
        // Use the testable stub for unit tests
        DependencyService.Register<ISQLite>(new MyFakeSqliteImplementation());
    }

    [Test]
    public void ViewModelShouldLoadData()
    {
        var vm = new MainViewModel();
        vm.LoadData();
        Assert.AreNotEqual(string.Empty, vm.Data);
    }
}
```

Unfortunately, if we try to access the `DependencyService` outside of an iOS/Android/Windows host project, we will receive an exception

> You MUST call Xamarin.Forms.Init()

In our unit test, we haven't called `Xamarin.Forms.Init()` anywhere, and we can't call that method outside of a host. In order to properly unit test the application, we are going to have to make some architectural changes.

### Solution #1

The first possible solution would be to move away from the Service Locator pattern completely, and instead architect our app using [Dependency Injection](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/dn178469(v=pandp.30)?redirectedfrom=MSDN). There are many benefits to Dependency Injection over Service Locators, but this usually comes down to developer preference. There are many [DI Containers](2015-01-09-IoC-Containers-with-Xamarin) that work with Xamarin and I encourgage you to try them out.

### Solution #2

If we really want to continue using the Service Locator pattern, but we also want to unit test our code, we will have to abstract Xamarin.Forms' implementation of the pattern.

To begin with, we need to create a Service Locator interface in our shared code project. Notice that this simply provides a way for our shared code to request an object based on a generic key. There is no reference or dependency to Xamarin.Forms at all.

```csharp
public interface IDependencyService
{
    T Get<T>() where T : class;
}
```

Next, we will change our `MainViewModel` to take in an implementation of the `IDependencyService` as a parameter. There are now two constructors for the `MainViewModel`. In the normal case of running our app, we will instantiate and use a new `DependencyServiceWrapper()` object _(see below)_, which will continue to use Xamarin.Forms' `DependencyService` object internally. The second constructor allows us to pass in any object that implements the `IDependencyService` interface, including mocking or stubbing it out for unit testing purposes.

```csharp
public class MainViewModel
{
    private readonly IDependencyService _dependencyService;

    public MainViewModel() : this(new DependencyServiceWrapper())
    {
    }

    public MainViewModel(IDependencyService dependencyService)
    {
        _dependencyService = dependencyService;
    }

    public string Data { get; set; }
    public void LoadData()
    {
        var database = _dependencyService.Get<ISQLite>();
        // Use the database
    }
}
```

The `DependencyServiceWrapper` class will simply delegate its calls to Xamarin.Forms' built-in `DependencyService`, giving us the same behavior as before while running the app on a host.

```csharp
public class DependencyServiceWrapper : IDependencyService
{
    public T Get<T> () where T : class
    {
        // The wrapper will simply pass everything through to the real Xamarin.Forms DependencyService class when not unit testing
        return DependencyService.Get<T> ();
    }
}
```

While unit testing, we can create a simple implementation of the interface to use. In this case, we're just going to use a `Dictionary<Type, object>` to store the implementations. Even though the `IDependencyService` interface only defined a `Get()` method, the stub implementation also provides a `Register()` method. Using the stub means that we never use the `DependencyService` class, and therefore never have to call `Xamarin.Forms.Init()` during a unit test.

```csharp
public class DependencyServiceStub : IDependencyService
{
    privet readonly Dictionary<Type, object> registeredServices = new Dictionary<Type, object>();

    public void Register<T>(object impl)
    {
        this.registeredServices[typeof(T)] = impl;
    }

    public T Get<T> () where T:class
    {
        return (T)registeredServices[typeof(T)];
    }
}
```

Finally, we can update our unit test to use the new `DependencyServiceStub`.

```csharp
[TestFixture]
public class TestMainViewModel
{
    IDependencyService _dependencyService;
    [OneTimeSetUp]
    public void Setup()
    {
        _dependencyService = new DependencyServiceStub ();
        // Use the testable stub for unit tests
        _dependencyService.Register<ISQLite>(new MyFakeSqliteImplementation());
    }

    [Test]
    public void ViewModelShouldLoadData()
    {
        var vm = new MainViewModel(_dependencyService);
        vm.LoadData();
        Assert.AreNotEqual(string.Empty, vm.Data);
    }
}
```
