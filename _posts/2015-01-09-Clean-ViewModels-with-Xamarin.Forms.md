---
title: "Clean ViewModels with Xamarin.Forms"
date: 2015-01-09
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

I like to keep my view models focused on just the code needed for a particular view, and to keep as much plumbing and infrastructure code out of the view model as possible. I've already covered how to [use Fody to implement INotifyPropertyChanged](/blog/2015/01/05/Fody-PropertyChanged-Xamarin-Studio-Easy-Mvvm) and use auto properties in our view models. Now I want to cover how to connect the **ViewModel** to the **View** automatically.

The most straight forward way to access our view model would be to simply instantiate it in the constructor of our view.

<pre><code class="csharp">using Xamarin.Forms;

namespace CleanViewModels
{  
  public partial class UserPage : ContentPage
  {
    UserViewModel _viewModel;
    public UserPage ()
    {
      InitializeComponent ();
      _viewModel = new UserViewModel ();
            BindingContext = _viewModel;
    }
  }
}
</code></pre>

This technique requires us to add this bit of code to every view that uses a view model though. I'd prefer to have that handled automatically for each view.

### View Model

Let's begin by creating a marker interface for our view model. At its simplest, this interface is just used to constrain the generic type later.

```csharp
namespace CleanViewModels
{
    public interface IViewModel {}
}
```

Each of our view models will need to implement our marker interface.

```csharp
using PropertyChanged;
using System.Windows.Input;
using Xamarin.Forms;

namespace CleanViewModels
{
  [ImplementPropertyChanged]
  public class UserViewModel : IViewModel
  {
    public string FirstName { get; set; }
    public string LastName { get; set; }

    public ICommand LoadUser {
      get {
        return new Command (async () => {
          this.FirstName = "John";
          this.LastName = "Doe";
        });
      }
    }
  }
}
```


### View Page

Now that we have the **ViewModel** defined, we can wire it up to the View. We'll create a new class, named *ViewPage.cs*. This will be our base class for each view.  We will derive **ViewPage** from **ContentPage** and pass in the type of our view model. In the constructor, we will create a new instance of the requested **ViewModel** and set the view's BindingContext to the current **ViewModel**. We also provide a read only property to be able to access the view model in the page, if needed.

```csharp
using Xamarin.Forms;

namespace CleanViewModels
{
  public class ViewPage&lt;T&gt; : ContentPage where T:IViewModel, new()
  {
    readonly T _viewModel; 

    public T ViewModel
    {
      get {
        return _viewModel;
      }
    }

    public ViewPage ()
    {
      _viewModel = new T ();
      BindingContext = _viewModel;
    }
  }
}
```

### View

When using XAML in Xamarin.Forms, I was not able to set the root to use **ViewPage&lt;T&gt;** directly. Instead, we will create a wrapper class that defines the <T> parameter for us, so that we can use the wrapper class in XAML.

```csharp
namespace CleanViewModels
{  
  public class UserPageBase :  ViewPage&lt;UserViewModel&gt; {}

  public partial class UserPage : UserPageBase
  {  
    public UserPage ()
    {
      InitializeComponent ();
    }
  }
}
```

Once the code behind is defined, we can add our xml namespace **local:** and create our view in XAML as **UserPageBase**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<local:UserPageBase 
  xmlns="http://xamarin.com/schemas/2014/forms" 
  xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml" 
  x:Class="CleanViewModels.UserPage" 
  xmlns:local="clr-namespace:CleanViewModels;assembly=CleanViewModels">

  <ContentPage.Padding>
    <OnPlatform 
      x:TypeArguments="Thickness" 
      iOS="5,20,5,5" 
      Android="5,0,5,5" 
      WinPhone="5,0,5,5" />
  </ContentPage.Padding>

  <StackLayout>
    <Entry Text="{ Binding FirstName, Mode=TwoWay }" />
    <Entry Text="{ Binding LastName, Mode=TwoWay }" />

    <Button 
        Text="Load User"
        Command="{Binding LoadUser}"></Button>
    </StackLayout>
</local:UserPageBase>
```

By using a base view page, along with Fody, we are able to keep our view models clean and focused only on the properties and commands that are needed for the view, thereby increasing the readability of the code.

Check out the sample project on [my Github repo](https://github.com/RobGibbens/CleanViewModels).
