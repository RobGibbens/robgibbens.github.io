---
title: "Backgrounding with Xamarin.Forms"
date: 2015-09-12
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

# Backgrounding with Xamarin.Forms

In [Xamarin University](http://xamarin.com/university), we have a few in depth courses dedicated to learning the concepts of backgrounding in iOS and Android [iOS210, iOS211, AND210]. If you’re not familiar with the concept, I highly suggest checking out those courses if you're a Xamarin University subscriber, or reading our excellent documentation on the [Xamarin developer portal](https://developer.xamarin.com/).

**Backgrounding** is the term we use for the process of allowing some of the code in our app to continue to execute while another app is in the foreground. On iOS, prior to iOS 9, only a single app is allowed to execute code at a time. This is referred to as the foreground app. If you don’t change your code to tell iOS that you plan on running code in the background, your app will be forcefully terminated and removed from memory if your code attempts to execute in the background. Android actually does allow code to run in a background activity, but background activities are one of the first things to be terminated if the operating system needs more memory. Instead, on Android, we should use another special class called a [Service](https://developer.xamarin.com/guides/android/application_fundamentals/services/).

One of the most common questions we hear is “How can we run these background tasks in a Xamarin.Forms app?” One of the best features of the Xamarin platform, and Xamarin.Forms specifically, is the ability to share large chunks of our code across platforms. This is often true in the case of background tasks as well. The function that we want to execute, such as saving data or calling a web service, is going to be the same on both iOS and Android. Unfortunately, the way that these two platforms have evolved to handle executing background code is completely different. As such, there is no way that we can abstract the backgrounding feature into the Xamarin.Forms library. Instead, we going to continue to rely on the native APIs to execute our shared background task.

![Not the same](/blog/docs/assets/NotTheSame.png)

We could try to abstract this functionality with interfaces and dependency injection, but the platforms are so different that this won't work well. Instead, we'll turn to some features built into Xamarin.Forms to help us achieve these goals and create robust apps. The Xamarin.Forms framework has two feature built in to it that are going to help us here. The first is a simple way to quickly persist some data, called the [Properties Dictionary](https://developer.xamarin.com/guides/cross-platform/xamarin-forms/working-with/app-lifecycle/#Properties_Dictionary). The second feature, called [MessagingCenter](https://developer.xamarin.com/guides/cross-platform/xamarin-forms/messaging-center/) will allow our shared code to communicate with each platform in a loosely coupled way. We’ll use this feature to build our UI in Xamarin.Forms, and then send a message to the iOS and Android apps to kick off a background task in each platform’s specific way, using its platform specific API.

![Messaging Center](/blog/docs/assets/MessagingCenter.gif)

The platform project will be able to execute its background apis, and then call into our shared code to actually execute the shared task.

As the background task is running, we will also want to send messages from the platform projects BACK to the shared Xamarin.Forms code in order to provide status notification and update the Xamarin.Forms UI.

![Messaging Center](/blog/docs/assets/MessagingCenter2.gif)

> Sample code is available at my [Github repo](https://github.com/RobGibbens/XamarinFormsBackgrounding)

## Scenarios

There are three common backgrounding scenarios that we are going to look at.

- Persisting some state quickly when the user sends our app to the background
- Long running, or finite length, tasks that are a normal part of our app’s logic. These are important tasks that we don’t want to be interrupted when the app is sent to the background, but we want to continue to run our code without terminating our app.
- Downloading a file, which is handled as a Background Necessary Application in iOS, and a Service in Android

### Persist Data

#### Save data in Xamarin.Forms

```csharp
public class App : Application  
{
  protected override void OnSleep ()
  {
    Application.Current.Properties ["SleepDate"] = DateTime.Now.ToString("O");
    Application.Current.Properties ["FirstName"] = _backgroundPage.FirstName;
  }

  protected override void OnStart ()
  {
    LoadPersistedValues ();
  }

  protected override void OnResume ()
  {
    LoadPersistedValues ();
  }

  private void LoadPersistedValues()
  {
    if (Application.Current.Properties.ContainsKey("SleepDate")) {
      var value = (string)Application.Current.Properties ["SleepDate"];
      DateTime sleepDate;
      if (DateTime.TryParse (value, out sleepDate)) {
        _backgroundPage.SleepDate = sleepDate;
      }
    }

    if (Application.Current.Properties.ContainsKey("FirstName")) {
      var firstName = (string)Application.Current.Properties ["FirstName"];
      _backgroundPage.FirstName = firstName;
    }
  }

}
```

### Finite Length Tasks

#### Messages

The key to communicating back and forth between the Xamarin.Forms shared code in the PCL and the platform specific projects is going to be the messages that we send with the Messaging Center. The first two classes, _StartLongRunningTaskMessage_ and _StopLongRunningTaskMessage_ are simply marker classes. They have no properties, and simply receiving one of these messages is enough to kick of some code. The third message, _TickedMessage_, will be used to send progress messages from the finite length task in order to update the Xamarin.Forms UI.

```csharp
public class StartLongRunningTaskMessage {}

public class StopLongRunningTaskMessage {}

public class TickedMessage  
{
  public string Message { get; set; }
}
```

#### Define the finite length task

Finite length tasks are usually something useful. Maybe it's saving some data to a local sqlite database, running some cpu intensive algorithm like encryption, or calling a remote web service. This is some bit of code that is important enough that we don't want it to be interrupted if the app is sent to the background. Or, it could be something useless like this code. For this sample, I will just run a loop, and send a message to the UI with the counter. This is just an easy example, replace it with something productive in your app.

```csharp
public async Task RunCounter(CancellationToken token)  
{
  await Task.Run (async () => {

    for (long i = 0; i < long.MaxValue; i++) {
      token.ThrowIfCancellationRequested ();

      await Task.Delay(250);
      var message = new TickedMessage { 
        Message = i.ToString()
      };

      Device.BeginInvokeOnMainThread(() => {
        MessagingCenter.Send<TickedMessage>(message, "TickedMessage");
      });
    }
  }, token);
}
```

#### Start task from Xamarin.Forms UI

In the shared Xamarin.Forms UI, we can wire up the Start and Stop buttons. Each button will simply send a message using the Messaging Center api. Each platform project is listening for these messages, and will use its platform specific apis to start an long running, finite length task.

```csharp
startLongRunningTask.Clicked += (s, e) => {  
  var message = new StartLongRunningTaskMessage ();
  MessagingCenter.Send (message, "StartLongRunningTaskMessage");
};

stopLongRunningTask.Clicked += (s, e) => {  
  var message = new StopLongRunningTaskMessage ();
  MessagingCenter.Send (message, "StopLongRunningTaskMessage");
};
```

#### Start task in iOS

In the _AppDelegate.cs_ file in the iOS project, we will use Messaging Center to subscribe to the Start and Stop messages. For convenience, I've wrapped the iOS apis in another class named _iOSLongRunningTaskExample_. The important methods here for iOS are **UIApplication.SharedApplication.BeginBackgroundTask ("LongRunningTask", OnExpiration)** and **UIApplication.SharedApplication.EndBackgroundTask (taskId)**. These are the methods that tell iOS that we'll be running code in the background and to not terminate our app. As you look at the code in the _iOSLongRunningTaskExample_ class, pay attention to the fact that it's focused on the code needed to run code in the background on iOS. All of the actual logic, our wonderful infinite loop in this case, is still shared between platforms.

```csharp
[Register ("AppDelegate")]
public partial class AppDelegate : global::Xamarin.Forms.Platform.iOS.FormsApplicationDelegate  
{
  public override bool FinishedLaunching (UIApplication app, NSDictionary options)
  {
    MessagingCenter.Subscribe<StartLongRunningTaskMessage> (this, "StartLongRunningTaskMessage", async message => {
      longRunningTaskExample = new iOSLongRunningTaskExample ();
      await longRunningTaskExample.Start ();
    });

    MessagingCenter.Subscribe<StopLongRunningTaskMessage> (this, "StopLongRunningTaskMessage", message => {
      longRunningTaskExample.Stop ();
    });
  }
}
```

```csharp
public class iOSLongRunningTaskExample  
{
  nint _taskId;
  CancellationTokenSource _cts;

  public async Task Start ()
  {
    _cts = new CancellationTokenSource ();

    _taskId = UIApplication.SharedApplication.BeginBackgroundTask ("LongRunningTask", OnExpiration);

    try {
      //INVOKE THE SHARED CODE
      var counter = new TaskCounter();
      await counter.RunCounter(_cts.Token);

    } catch (OperationCanceledException) {
    } finally {
      if (_cts.IsCancellationRequested) {
        var message = new CancelledMessage();
        Device.BeginInvokeOnMainThread (
          () => MessagingCenter.Send(message, "CancelledMessage")
        );
      }
    }

    UIApplication.SharedApplication.EndBackgroundTask (_taskId);
  }

  public void Stop ()
  {
    _cts.Cancel ();
  }

  void OnExpiration ()
  {
    _cts.Cancel ();
  }
}
```

#### Start task in Android

In Android, we use Services to run code outside of the normal Activity lifecycle. In the _MainActivity.cs_ we can subscribe to the same Start and Stop messages that we did in iOS, but here we will start a new _LongRunningTaskService_.

```csharp
[Activity (Label = "FormsBackgrounding.Droid", Icon = "@drawable/icon", MainLauncher = true, ConfigurationChanges = ConfigChanges.ScreenSize | ConfigChanges.Orientation)]
public class MainActivity : global::Xamarin.Forms.Platform.Android.FormsApplicationActivity  
{
  protected override void OnCreate (Bundle bundle)
  {
    MessagingCenter.Subscribe<StartLongRunningTaskMessage> (this, "StartLongRunningTaskMessage", message => {
      var intent = new Intent (this, typeof(LongRunningTaskService));
      StartService (intent);
    });

    MessagingCenter.Subscribe<StopLongRunningTaskMessage> (this, "StopLongRunningTaskMessage", message => {
      var intent = new Intent (this, typeof(LongRunningTaskService));
      StopService (intent);
    });
  }
}
```

#### Run task in Android Service

The service will simply execute the same shared code that iOS did in the _TaskCounter_ class. Again, notice that all of the code in the Service is specific to running a task on Android. Our actual app logic is still shared between platforms.

```csharp
[Service]
public class LongRunningTaskService : Service  
{
  CancellationTokenSource _cts;

  public override IBinder OnBind (Intent intent)
  {
    return null;
  }

  public override StartCommandResult OnStartCommand (Intent intent, StartCommandFlags flags, int startId)
  {
    _cts = new CancellationTokenSource ();

    Task.Run (() => {
      try {
        //INVOKE THE SHARED CODE
        var counter = new TaskCounter();
        counter.RunCounter(_cts.Token).Wait();
      }
      catch (OperationCanceledException) {
      }
      finally {
        if (_cts.IsCancellationRequested) {
          var message = new CancelledMessage();
          Device.BeginInvokeOnMainThread (
            () => MessagingCenter.Send(message, "CancelledMessage")
          );
        }
      }

    }, _cts.Token);

    return StartCommandResult.Sticky;
  }

  public override void OnDestroy ()
  {
    if (_cts != null) {
      _cts.Token.ThrowIfCancellationRequested ();

      _cts.Cancel ();
    }
    base.OnDestroy ();
  }
}
```

#### Update Xamarin.Forms UI

As we saw in the _TaskCounter_ class, as we iterate through our loop we'll be calling **MessagingCenter.Send(message, "TickedMessage")** repeatedly. The Xamarin.Forms _LongRunningPage_ will subscribe to these messages, and update the **Label.Text** on the screen.

```csharp
void HandleReceivedMessages ()  
{
  MessagingCenter.Subscribe<TickedMessage> (this, "TickedMessage", message => {
    Device.BeginInvokeOnMainThread (() => {
      ticker.Text = message.Message;
    });
  });

  MessagingCenter.Subscribe<CancelledMessage> (this, "CancelledMessage", message => {
    Device.BeginInvokeOnMainThread (() => {
      ticker.Text = "Cancelled";
    });
  });
}
```

### Downloading Files

The basic pattern that we used to communicate when executing a finite length task will also be used here. The Xamarin.Forms code will send messages back and forth to the platform specific code.

#### Start download from Xamarin.Forms UI

When the button is clicked in the Xamarin.Forms code, we'll send a message with the url of the download.

```csharp
downloadButton.Clicked += (sender, e) => {  
  var message = new DownloadMessage {
    Url = "http://xamarinuniversity.blob.core.windows.net/ios210/huge_monkey.png"
  };

  MessagingCenter.Send (message, "Download");
}
```

#### Start task in iOS

The iOS _AppDelegate_ will subscribe to this message and invoke our custom _Downloader_ class. Again, this is just a wrapper class around iOS' apis.

```csharp
MessagingCenter.Subscribe<DownloadMessage> (this, "Download", async message => {  
  var downloader = new Downloader (message.Url);
  await downloader.DownloadFile ();
});
```

#### Send download progress from iOS

In the custom _NSUrlSessionDownloadDelegate_ subclass, we will send a _DownloadProgressMessage_ every time we get a chunk of data downloaded, and a _DownloadFinishedMessage_ when the entire file is downloaded and saved locally.

```csharp
public class CustomSessionDownloadDelegate : NSUrlSessionDownloadDelegate  
{
  public override void DidWriteData (NSUrlSession session, NSUrlSessionDownloadTask downloadTask, long bytesWritten, long totalBytesWritten, long totalBytesExpectedToWrite)
  {
    float percentage = (float)totalBytesWritten / (float)totalBytesExpectedToWrite;

    var message = new DownloadProgressMessage () {
      BytesWritten = bytesWritten,
      TotalBytesExpectedToWrite = totalBytesExpectedToWrite,
      TotalBytesWritten = totalBytesWritten,
      Percentage = percentage
    };

    MessagingCenter.Send<DownloadProgressMessage> (message, "DownloadProgressMessage");
  }

  public override void DidFinishDownloading (NSUrlSession session, NSUrlSessionDownloadTask downloadTask, NSUrl location)
  {
    CopyDownloadedImage (location);

    var message = new DownloadFinishedMessage () {
      FilePath = targetFileName,
      Url = downloadTask.OriginalRequest.Url.AbsoluteString
    };

    MessagingCenter.Send<DownloadFinishedMessage> (message, "DownloadFinishedMessage");
  }

  /// More
}
```

#### Start task in Android

In Android, we will once again create a new Service when the _DownloadMessage_ is received.

```csharp
MessagingCenter.Subscribe<DownloadMessage> (this, "Download", message =>  {  
  var intent = new Intent (this, typeof(DownloaderService));
  intent.PutExtra ("url", message.Url);
  StartService (intent);
});
```

#### Download file in Android Service

The service will then send the _DownloadFinishedMessage_ when the file is saved.

```csharp
[Service]
public class DownloaderService : Service  
{
  public override IBinder OnBind (Intent intent)
  {
    return null;
  }

  public  override StartCommandResult OnStartCommand (Intent intent, StartCommandFlags flags, int startId)
  {
    var url = intent.GetStringExtra ("url");

    Task.Run (() => {
      var imageHelper = new ImageHelper ();
      imageHelper.DownloadImageAsync (url)
          .ContinueWith (filePath => {
                    var message = new DownloadFinishedMessage {
                      FilePath = filePath.Result
                    };
                    MessagingCenter.Send (message, "DownloadFinishedMessage");
                  });
    });

    return StartCommandResult.Sticky;
  }
}
```

#### Update Xamarin.Forms UI

Finally, the Xamarin.Forms UI will subscribe to the messages, and update its UI when it receives the appropriate message.

```csharp
MessagingCenter.Subscribe<DownloadProgressMessage> (this, "DownloadProgressMessage", message => {  
  Device.BeginInvokeOnMainThread(() => {
    this.downloadStatus.Text = message.Percentage.ToString("P2");
  });
});

MessagingCenter.Subscribe<DownloadFinishedMessage> (this, "DownloadFinishedMessage", message => {  
  Device.BeginInvokeOnMainThread(() =>
    {
      catImage.Source = FileImageSource.FromFile(message.FilePath);
    });
});
```

## Stay loose my friends

Xamarin.Forms is a wonderful cross platform UI framework. The Xamarin.Forms team has spent **a lot** of time abstracting the major mobile platforms for developers, so that we can focus on building great apps. Unfortunately, not _everything_ can be abstracted, including running code in the background. The key to mixing the multiple platform paradigms lies in staying loosely coupled using the _MessagingCenter_ framework in Xamarin.Forms and allowing the individual platforms to wrap the code in its specific apis.

> _I recorded this post for a Xamarin University Lightning Lecture. Lightning Lectures are free to everyone._

Check it out here : [https://university.xamarin.com/lightninglectures](https://university.xamarin.com/lightninglectures).
