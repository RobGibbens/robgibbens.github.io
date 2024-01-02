---
title: "Fody.PropertyChanged + Xamarin Studio = Easy Mvvm"
date: 2015-01-05
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

[Model-View-ViewModel (Mvvm)](http://en.wikipedia.org/wiki/Model_View_ViewModel) is a fantastic pattern when designing cross platform mobile applications. With Mvvm, we can write our ViewModels in shared code and target Xamarin.iOS, Xamarin.Android, Windows Phone, and Windows Store apps. Libraries such as [Xamarin.Forms](http://developer.xamarin.com/guides/cross-platform/xamarin-forms/), [MvvmCross](https://github.com/MvvmCross/MvvmCross), and [ReactiveUI](http://www.reactiveui.net/) provide excellent databinding and commanding functionality.

One thing that bothers me with Mvvm is the amount of boilerplate code that is required to implement the [INotifyPropertyChanged interface](http://msdn.microsoft.com/en-us/library/system.componentmodel.inotifypropertychanged.aspx), which powers our databinding engine.

#### INotifyPropertyChanged Implementation

```csharp
  using System.Runtime.CompilerServices;
  using System.ComponentModel;
  
  public class UserViewModel : INotifyPropertyChanged
  {
    public event PropertyChangedEventHandler PropertyChanged;

    private string _firstName;
    public string FirstName {
      get { 
        return _firstName; 
      }
      set {
        if (value != _firstName) {
          _firstName = value;
          NotifyPropertyChanged ();
        }
      }
    }

    private string _lastName;
    public string LastName {
      get { 
        return _lastName; 
      }
      set {
        if (value != _lastName) {
          _lastName = value;
          NotifyPropertyChanged ();
        }
      }
    }

    private void NotifyPropertyChanged ([CallerMemberName] string propertyName = "")
    {
      if (PropertyChanged != null) {
        PropertyChanged (this, new PropertyChangedEventArgs (propertyName));
      }
    }
  }
```

That's a lot of boilerplate cluttering up our code, making it harder to read and understand what this class is really trying to accomplish. The code required by the notification/databinding engine isn't adding any value to our view model and it isn't related to our application or our business logic. Ideally, we'd like to remove that boilerplate code and focus on our app.  

#### Clean ViewModel with Auto Properties

```csharp
  public class UserViewModel
  {
    public string FirstName { get; set; }
    public string LastName { get; set; }
  }
```

Luckily, there is an amazing library named [Fody.PropertyChanged](https://github.com/Fody/PropertyChanged) that we can leverage that will allow us to write only the code that we need, but rewrite the resulting assembly to implement INotifyPropertyChanged for us. All we need to do is add the [PropertyChanged.Fody Nuget Package](http://www.nuget.org/packages/PropertyChanged.Fody/) to our project, and add an attribute to our ViewModel.

### ViewModel + Fody.PropertyChanged

```csharp
  using PropertyChanged;

  [ImplementPropertyChanged]
  public class UserViewModel
  {
    public string FirstName { get; set; }
    public string LastName { get; set; }
  }
```

Out of the box, this all works just fine using Visual Studio on Windows. There are a few problems currently with Xamarin Studio though. Adding the Nuget package is successful, but the project fails to compile. Fixing these issues is easy though.


- Add the PropertyChanged.Fody package via Nuget

    Install-Package PropertyChanged.Fody

- The Nuget package adds a FodyWeavers.xml file to the project, but leaves it empty
  - Add &lt;PropertyChanged /&gt; to FodyWeavers.xml
  
```xml
<?xml version="1.0" encoding="utf-8" ?>
<Weavers>
  <PropertyChanged />
</Weavers>
```

- Set FodyWeavers.xml to Build Action -> Content
- Set FodyWeavers.xml to Quick Properties -> Copy to Output Directory
- Build

Viewing the resulting assembly in Xamarin Studio's Show Disassembly window gives the follwing result

### Disassembled Code with Fody

```csharp
  using System;
  using System.ComponentModel;
  using System.Runtime.CompilerServices;

  namespace TestFody
  {
    public class UserViewModelWithFody : INotifyPropertyChanged
    {
      //
      // Properties
      //
      public string FirstName {
        [CompilerGenerated]
        get {
          return this.<FirstName>k__BackingField;
        }
        [CompilerGenerated]
        set {
          if (string.Equals (this.<FirstName>k__BackingField, value, StringComparison.Ordinal)) {
            return;
          }
          this.<FirstName>k__BackingField = value;
          this.OnPropertyChanged ("FirstName");
        }
      }

      public string LastName {
        [CompilerGenerated]
        get {
          return this.<LastName>k__BackingField;
        }
        [CompilerGenerated]
        set {
          if (string.Equals (this.<LastName>k__BackingField, value, StringComparison.Ordinal)) {
            return;
          }
          this.<LastName>k__BackingField = value;
          this.OnPropertyChanged ("LastName");
        }
      }

      //
      // Methods
      //
      public virtual void OnPropertyChanged (string propertyName)
      {
        PropertyChangedEventHandler propertyChanged = this.PropertyChanged;
        if (propertyChanged != null) {
          propertyChanged (this, new PropertyChangedEventArgs (propertyName));
        }
      }

      //
      // Events
      //
      [field: NonSerialized]
      public event PropertyChangedEventHandler PropertyChanged;
    }
  }
```
