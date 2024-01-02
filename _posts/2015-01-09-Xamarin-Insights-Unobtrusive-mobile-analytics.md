---
title: "Xamarin Insights : Unobtrusive mobile analytics"
date: 2015-01-09
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

At the [Xamarin Evolve 2014](https://evolve.xamarin.com) conference, Xamarin announced their new mobile analytics solution, [Xamarin Insights](https://insights.xamarin.com). When you add Insights to your mobile application, you can start tracking exceptions, crashes, user identities, and application events. 

One of the best features of Insights is the ability to track the user's actions through the app to be able to trace the conditions that led to an exception. All too often, users will experience an error or, even worse, an app crash. Rarely, these users will email the developer and tell them about the crash, but don't remember any useful information that would help to recreate the exception. By adding calls **Xamarin.Insights.Track()** to our methods, we can track the events leading up to a particular crash.

### Typical Usage

```csharp
public async Task GetData ()
{
    Xamarin.Insights.Track ("Enter GetData");
    
    /* Implement Method */

    Xamarin.Insights.Track ("Exit GetData");
}

private async Task GetLocalData ()
{
    Xamarin.Insights.Track ("Enter GetLocalData");
    
    /* Implement Method */

    Xamarin.Insights.Track ("Exit GetLocalData");
}

private async Task GetRemoteData ()
{
    Xamarin.Insights.Track ("Enter GetRemoteData");
    
    /* Implement Method */

    Xamarin.Insights.Track ("Exit GetRemoteData");
}
```

While this _does_ work, it adds a lot of unnecessary noise to the code.  Instead of adding line after line of analytics tracking to every method, I prefer to get that boilerplate code out of the way, and let the code focus on the problem at hand.

As [I've](/blog/2015/01/09/Clean-ViewModels-with-Xamarin.Forms) [shown](/blog/2015/01/09/End-to-End-Mvvm-with-Xamarin) [before](/blog/2015/01/05/Fody-PropertyChanged-Xamarin-Studio-Easy-Mvvm), I get a lot of use out of [Fody](https://github.com/Fody/Fody) in my mobile apps. Fody allows us to control the build time compilation and change the outputted assembly.  In this case, we can use Fody's MethodDecorator package to move the Insights tracking logic into a method attribute.

### Adding Fody

We'll base our custom attribute on Fody's MethodDecorator attribute, which we can add to our project from [Nuget](http://www.nuget.org/packages/MethodDecoratorEx.Fody).

> Install-Package MethodDecoratorEx.Fody

> NOTE : There are two Fody MethodDecorators on Nuget. I'm using MethodDecoratorEx

You'll need to add the declaration to the FodyWeavers.xml file as well.

```xml
<?xml version="1.0" encoding="utf-8" ?>  
<Weavers>
    <MethodDecoratorEx />
</Weavers>
```

[I wrote up instructions on using Fody with Xamarin Studio](/blog/2015/01/05/Fody-PropertyChanged-Xamarin-Studio-Easy-Mvvm)

### Create Attribute

Next, we'll create a custom attribute to wrap up the Insights code. Each time we enter or leave a method, we'll make a call to **Xamarin.Insights.Track()**.  

```csharp
using System;
using System.Reflection;
using MethodDecoratorInterfaces;
using ArtekSoftware.Demos;
using Xamarin;

[module: Insights]

namespace ArtekSoftware.Demos
{
    [AttributeUsage (
            AttributeTargets.Method 
            | AttributeTargets.Constructor 
            | AttributeTargets.Assembly 
            | AttributeTargets.Module)]
    public class InsightsAttribute : Attribute, IMethodDecorator
    {
        private string _methodName;

        public void Init (object instance, MethodBase method, object[] args)
        {
            _methodName = method.DeclaringType.FullName + "." + method.Name;
        }

        public void OnEntry ()
        {
            var message = string.Format ("OnEntry: {0}", _methodName);
            Insights.Track (message);
        }

        public void OnExit ()
        {
            var message = string.Format ("OnExit: {0}", _methodName);
            Insights.Track (message);
        }

        public void OnException (Exception exception)
        {
            Insights.Report (exception);
        }
    }
}
```

### Add Attribute to Methods

Once the attribute is defined, all we need to do is decorate the methods that we want to track.  Notice that the implementation of each method is focused simply on the method's logic and not Insights tracking.

```csharp
[Insights]
public async Task GetData ()
{
    /* Implement Method */
}

[Insights]
private async Task GetLocalData ()
{
    /* Implement Method */
}

[Insights]
private async Task GetRemoteData ()
{
    /* Implement Method */
}
```

### Performance

This may seem like a lot of extra overhead, especially since we're calling out to a remote server. I asked the Xamarin Insights team about this, and got the following answers.

- When does Insights send its data?
  - If Insights detects a wifi connection then generally we feel free to send data as often as we like, if we are on a Cellular connection we wait a very long time before sending data
- Does Insights.Track immediately call the server, or are the calls batched up?
  - All data is batched up for a few seconds before sending out to the server
- Do Insights.Track and Insights.Report call the server asynchronously, or are these blocking calls?
  - All API calls are essentially async, any Insights activity happens in a background thread
- If queued, does the queue persist across restarts of the app? Restarts of the device?
  - All insights data is journaled to disk, this means track/identify/report/crashes/everything is persistent across restarts. We send out the old data whenever we have a good opportunity to do so, usually after we send out some new data successfully.

### Results

The end result of this is that we get really detailed tracking of the events that lead to an exception. This will make finding and fixing errors in our apps faster and more efficient.  We get the details of constant tracking without littering our code with tracking calls.

![Insights Events](/blog/docs/assets/Insights-Events.png)
