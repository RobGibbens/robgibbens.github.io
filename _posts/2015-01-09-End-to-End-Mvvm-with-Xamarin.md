---
title: "End to End Mvvm with Xamarin"
date: 2015-01-09
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

As a [Xamarin University](https://xamarin.com/university) instructor, I am occasionally asked how I would put together a typical MVVM based app while connecting to a remote service. This is not the only way to architect our apps, and may not be the best for **your** app. Your mileage may vary.

Our goal is to connect to a remote service api, download the data, convert into a model that we control, and bind it to our ui. We'll connect to a remote service to download technical conference information, and we'll utilize a handful of useful open source .net libraries to speed things along and make our code cleaner.

We'll be using Xamarin Studio 5 (although Visual Studio will work just fine), and we'll use [Xamarin.Forms](https://xamarin.com/forms) for our user interface. All of the code is available from my [Github Repo](https://github.com/RobGibbens/DtoToVM)

### Create Solution

Start by creating a new solution.  We'll be using the "Blank App (Xamarin.Forms Portable)" template under the "Mobile Apps" node. I prefer PCLs over Shared Projects, but that's just me. Shared Projects will work also.  I'll name the solution "DtoToVm" This template will create a core Portable Class Library as well as an Android and an iOS project (and a Windows Phone app if you're using Visual Studio on Windows). We'll be putting almost all of our code in the shared PCL.

> File -> New Solution -> Blank App (Xamarin.Forms Portable)

![Solution structure](/blog/docs/assets/SolutionStructure.png)

### Remote Third Party API (JSON)

Let's take a look at the api that we'll be working with. For this example, I'll be using [api.tekconf.com/v1/conferences](http://api.tekconf.com/v1/conferences?format=json) an open source conference management system (written by me). When performing a GET request on the conferences uri, we're given a list of all of the active conferences in the system.

```javascript
[
  {
  slug: "codestock-2014",
  name: "CodeStock 2014",
  start: "/Date(1405036800000)/",
  end: "/Date(1405123200000)/",
  callForSpeakersOpens: "/Date(1391619909000)/",
  callForSpeakersCloses: "/Date(1391619909000)/",
  registrationOpens: "/Date(1391619909000)/",
  registrationCloses: "/Date(1405036800000)/",
  description: "CodeStock is a two day event (July 11th & July 12th of 2014) for technology and information exchange. Created by the community, for the community â€“ this is not an industry trade show pushing the latest in marketing as technology, but a gathering of working professionals sharing knowledge and experience. Join us at CodeStock 2014 and move into the future.",
  lastUpdated: "/Date(1392149694455)/",
  address: {
    streetNumber: 0,
    city: "Knoxville",
    state: "TN",
    postalArea: "US"
  },
  imageUrl: "http://tekconf.blob.core.windows.net/images/conferences/codestock-2014.png",
  imageUrlSquare: "http://tekconf.blob.core.windows.net/images/conferences/codestock-2014-square.jpg",
  isLive: true,
  homepageUrl: "http://www.codestock.org/",
  position: [
    -83.9207392,
    35.9606384
  ],
  defaultTalkLength: 60,
  rooms: [ ],
  sessionTypes: [ ],
  subjects: [ ],
  tags: [ ],
  sessions: [ ],
  numberOfSessions: 0,
  isAddedToSchedule: false,
  isOnline: false,
  dateRange: "July 11 - 12, 2014",
  formattedAddress: "Knoxville, TN"
  }
]
```

### Service Client

In order to connect to this service, we'll use [Microsoft's HttpClient](http://www.nuget.org/packages/Microsoft.Net.Http/2.2.27-beta) library. This is a cross platform nuget package that works well with PCLs and provides a nice async interface for accessing remote web endpoints. TekConf may not be the only service that we connect to though, so we'll wrap the access to the site in a client. This segregates the TekConf service to a small corner of our apps and allows the app code to focus on the app.

> Add Class to PCL -> Services/TekConfClient.cs

```csharp
namespace DtoToVM.Services
{
  using System;
  using System.Collections.Generic;
  using System.Linq;
  using System.Net.Http;
  using System.Net.Http.Headers;
  using System.Threading.Tasks;
  using AutoMapper;
  using Newtonsoft.Json;
  using DtoToVM.Dtos;
  using DtoToVM.Models;

  public class TekConfClient
  {
    public async Task<List<Conference>> GetConferences ()
    {
      IEnumerable<ConferenceDto> conferenceDtos = Enumerable.Empty<ConferenceDto>();
      IEnumerable<Conference> conferences = Enumerable.Empty<Conference> ();

      using (var httpClient = CreateClient ()) {
        var response = await httpClient.GetAsync ("conferences").ConfigureAwait(false);
        if (response.IsSuccessStatusCode) {
          var json = await response.Content.ReadAsStringAsync ().ConfigureAwait(false);
          if (!string.IsNullOrWhiteSpace (json)) {
            conferenceDtos = await Task.Run (() => 
              JsonConvert.DeserializeObject<IEnumerable<ConferenceDto>>(json)
            ).ConfigureAwait(false);

            conferences = await Task.Run(() => 
              Mapper.Map<IEnumerable<Conference>> (conferenceDtos)
            ).ConfigureAwait(false);
          }
        }
      }

      return conferences.ToList();
    }

    private const string ApiBaseAddress = "http://api.tekconf.com/v1/";
    private HttpClient CreateClient ()
    {
      var httpClient = new HttpClient 
      { 
        BaseAddress = new Uri(ApiBaseAddress)
      };

      httpClient.DefaultRequestHeaders.Accept.Clear();
      httpClient.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

      return httpClient;
    }
  }
}
```

### Data Transfer Object (DTO)

Notice that we're introducing two new classes here, the DTO and the Model. We're creating a DTO so that we can deserialize the JSON string from the api into strongly typed C# objects. We'll use the [Newtonsoft JSON.net nuget package](http://www.nuget.org/packages/Newtonsoft.Json/) for the deserialization. This DTO will match the remote json, and therefore be in the format and naming dictated by the remote, possibly third party service.

> Add Class to PCL -> Dtos/ConferenceDto.cs

```csharp
namespace DtoToVM.Dtos
{
  using System;
  public class ConferenceDto
  {
    public string Slug { get; set; }
    public string Name { get; set; }
    public DateTime Start { get; set; }
    public double[] Position { get; set; }
  }
}
```

### Model

The model class will be in the shape and structure that we want to deal with in our app. The DTO and the model very likely will be different. In our case, the DTO has an array of doubles named _position_ to hold the geographic location of the conference, while our Conference model has two individual properties named _Latitude_ and _Longitude_, which will be easier to store in our local SQLite database. 

> Add Class to PCL -> Models/Conference.cs

```csharp
namespace DtoToVM.Models
{
  using System;
  using SQLite.Net.Attributes;
  public class Conference
  {
    [PrimaryKey, AutoIncrement, Column ("_id")]
    public int Id { get; set; }
    [Unique]
    public string Slug { get; set; }
    public string Name { get; set; }
    public DateTime Start { get; set; }
    public double Latitude { get; set; }
    public double Longitude { get; set; }
  }
}
```

### Map DTO to Model

To convert from the externally defined DTO to our model, we'll use [AutoMapper](http://www.nuget.org/packages/AutoMapper), which provides us an easy way to map two different objects and copy the data from one to another.

> Add Class to PCL -> Bootstrapper.cs

```csharp
namespace DtoToVM
{
  using AutoMapper;
  using DtoToVM.Dtos;
  using DtoToVM.Models;

  public class Bootstrapper
  {
    public void Automapper()
    {
      Mapper.CreateMap<ConferenceDto, Conference> ()
        .ForMember(dest => dest.Latitude, opt => opt.ResolveUsing<LatitudeResolver>())
        .ForMember(dest => dest.Longitude, opt => opt.ResolveUsing<LongitudeResolver>());
    }
  }

  public class LatitudeResolver : ValueResolver<ConferenceDto, double>
  {
    protected override double ResolveCore(ConferenceDto source)
    {
      return source.Position[0];
    }
  }

  public class LongitudeResolver : ValueResolver<ConferenceDto, double>
  {
    protected override double ResolveCore(ConferenceDto source)
    {
      return source.Position[1];
    }
  }
}
```

### View Model

At this point, we're downloading the conference data from the remote site, and transforming it into our C# model. Next, we want to create some user interface to display that data. [Xamarin.Forms](https://xamarin.com/forms) shines for an app like this, where we want the same UI across three platforms, and we're not going to be customizing the UI very much. In order to simplify databinding, we'll create a View Model class for the list of conferences. The View Model will implement the [INotifyPropertyChanged interface](http://msdn.microsoft.com/en-us/library/system.componentmodel.inotifypropertychanged.aspx) to relay change events to our UI. We'll using [Fody.PropertyChanged](https://github.com/Fody/PropertyChanged) to implement the interface for us, removing the boilerplate code. See my previous post [Fody.PropertyChanged + Xamarin Studio = Easy Mvvm](http://artekblog.azurewebsites.net/fody-propertychanged-xamarin-studio/) for more information about this library.

The View Model will use the TekConfClient class to download the Conference information. It will then save the list of Conference models to a local SQLite database. Lastly, it will set the Conferences property, which our UI will bind to.

> Add Class to PCL -> ViewModels/ConferencesViewModel.cs

```csharp
namespace DtoToVM.ViewModels
{
  using System.Collections.Generic;
  using System.Linq;
  using System.Threading.Tasks;
  using PropertyChanged;
  using DtoToVM.Data;
  using DtoToVM.Models;
  using DtoToVM.Services;

  [ImplementPropertyChanged]
  public class ConferencesViewModel
  {
    readonly SQLiteClient _db;

    public ConferencesViewModel ()
    {
      _db = new SQLiteClient ();
    }

    public List<Conference> Conferences { get; set; }

    public async Task GetConferences ()
    {
      await GetLocalConferences ();
      await GetRemoteConferences ();
      await GetLocalConferences ();
    }

    private async Task GetLocalConferences()
    {
      var conferences = await _db.GetConferencesAsync ();
      this.Conferences = conferences.OrderBy(x => x.Name).ToList();
    }

    private async Task GetRemoteConferences()
    {
      var remoteClient = new TekConfClient ();
      var conferences = await remoteClient.GetConferences ().ConfigureAwait(false);
      await _db.SaveAll (conferences).ConfigureAwait(false);
    }
  }
}
```

### Save to Database

Once we download the remote conference information, we'll cache it in a local SQLite database. By using the [SQLite.Net PCL](http://www.nuget.org/packages/SQLite.Net-PCL/), the [SQLite.Net Async PCL](http://www.nuget.org/packages/SQLite.Net.Async-PCL/), and the [SQLite Android](http://www.nuget.org/packages/SQLite.Net.Platform.XamarinAndroid/) and [SQLite iOS](http://www.nuget.org/packages/SQLite.Net.Platform.XamarinIOS/) libraries, we are able to easily save our Conference model to a local database. The SQLite PCL enables us to write almost all of our database code in our PCL and share it across all the different platforms that we're supporting. The one piece that is not shared though is the database connection. For that, we'll create an ISQLite interface, and we'll use Xamarin.Forms DependencyService to resolve the platform specific implementation of the SQLite connection.

> Add Class to PCL -> Data/SQLiteClient.cs

```csharp
namespace DtoToVM.Data
{
  using SQLite.Net.Async;

  public interface ISQLite {
    SQLiteAsyncConnection GetConnection();
  }
  
  using System.Collections.Generic;
  using System.Threading.Tasks;
  using SQLite.Net.Async;
  using Xamarin.Forms;
  using DtoToVM.Data;
  using DtoToVM.Models;

  public class SQLiteClient
  {
    private static readonly AsyncLock Mutex = new AsyncLock ();
    private readonly SQLiteAsyncConnection _connection;

    public SQLiteClient ()
    {
      _connection = DependencyService.Get<ISQLite> ().GetConnection ();
      CreateDatabaseAsync ();
    }

    public async Task CreateDatabaseAsync ()
    {
      using (await Mutex.LockAsync ().ConfigureAwait (false)) {
        await _connection.CreateTableAsync<Conference> ().ConfigureAwait (false);
      }
    }

    public async Task<List<Conference>> GetConferencesAsync ()
    {
      List<Conference> conferences = new List<Conference> ();
      using (await Mutex.LockAsync ().ConfigureAwait (false)) {
        conferences = await _connection.Table<Conference> ().ToListAsync ().ConfigureAwait (false);
      }

      return conferences;
    }

    public async Task Save (Conference conference)
    {
      using (await Mutex.LockAsync ().ConfigureAwait (false)) {
        // Because our conference model is being mapped from the dto,
        // we need to check the database by name, not id
        var existingConference = await _connection.Table<Conference> ()
            .Where (x => x.Slug == conference.Slug)
            .FirstOrDefaultAsync ();

        if (existingConference == null) {
          await _connection.InsertAsync (conference).ConfigureAwait (false);
        } else {
          conference.Id = existingConference.Id;
          await _connection.UpdateAsync (conference).ConfigureAwait (false);
        }
      }
    }

    public async Task SaveAll (IEnumerable<Conference> conferences)
    {
      foreach (var conference in conferences) {
        await Save (conference);
      }
    }
  }
}
```

> Note : Find the AsyncLock class on [Scott Hanselman's Blog](http://www.hanselman.com/blog/ComparingTwoTechniquesInNETAsynchronousCoordinationPrimitives.aspx)

### SQLite Implementations

> Add Class to iOS Project -> Data/SQLiteClient.cs

```csharp
using Xamarin.Forms;
using DtoToVM.iOS.Data;

[assembly: Dependency (typeof(SQLiteClient))]
namespace DtoToVM.iOS.Data
{
  using System;
  using DtoToVM.Data;
  using SQLite.Net.Async;
  using System.IO;
  using SQLite.Net.Platform.XamarinIOS;
  using SQLite.Net;

  public class SQLiteClient : ISQLite
  {
    public SQLiteAsyncConnection GetConnection ()
    {
      var sqliteFilename = "Conferences.db3";
      var documentsPath = Environment.GetFolderPath (Environment.SpecialFolder.Personal);
      var libraryPath = Path.Combine (documentsPath, "..", "Library");
      var path = Path.Combine (libraryPath, sqliteFilename);
    
      var platform = new SQLitePlatformIOS ();

      var connectionWithLock = new SQLiteConnectionWithLock (
                      platform,
                      new SQLiteConnectionString (path, true));

      var connection = new SQLiteAsyncConnection (() => connectionWithLock);

      return connection;
    }
  }
}
```

> Add Class to Android Project -> Data/SQLiteClient.cs

```csharp
using Xamarin.Forms;
using DtoToVM.Android.Data;

[assembly: Dependency (typeof(SQLiteClient))]
namespace DtoToVM.Android.Data
{
  using System;
  using DtoToVM.Data;
  using SQLite.Net.Async;
  using System.IO;
  using SQLite.Net.Platform.XamarinAndroid;
  using SQLite.Net;

  public class SQLiteClient : ISQLite
  {
    public SQLiteAsyncConnection GetConnection ()
    {
      var sqliteFilename = "Conferences.db3";
      var documentsPath = Environment.GetFolderPath (Environment.SpecialFolder.Personal);

      var path = Path.Combine (documentsPath, sqliteFilename);

      var platform = new SQLitePlatformAndroid ();

      var connectionWithLock = new SQLiteConnectionWithLock (
                     platform,
                     new SQLiteConnectionString (path, true));

      var connection = new SQLiteAsyncConnection (() => connectionWithLock);

      return connection;
    }
  }
}
```

Notice the use of 

```csharp
[assembly: Dependency (typeof(SQLiteClient))]
```

on each class. This will register the class in the DependencyService, and allow our PCL to resolve the dependency.  Also note that on iOS, we are specifying the database path differently, so that the db3 database file is not backed up into iCloud.

### Xamarin.Forms UI

The last part of the project is to create the user interface and wire everything up. We can use Xamarin.Forms to quickly create a cross platform UI and bind our list of conferences to a ListView control. Here, we'll create an instance of our ConferencesViewModel class, set the databinding to the ListView's cell template, and load the data. The key is setting the ListView's ItemsSource property to our ViewModel's Conferences property, which is implmenting the INotifyPropertyChanged events. 

```csharp
_conferencesListView.ItemsSource = viewModel.Conferences;
```

> Add Class to PCL -> Pages/ConferencesPage.cs

```csharp
namespace DtoToVM.Pages
{
  using System.Threading.Tasks;
  using Xamarin.Forms;
  using DtoToVM.ViewModels;

  public class ConferencesPage : ContentPage
  {
    ListView _conferencesListView;

    public ConferencesPage ()
    {
      this.Content = new Label { 
        HorizontalOptions = LayoutOptions.CenterAndExpand,
        VerticalOptions = LayoutOptions.CenterAndExpand
      };

      Init ();
    }

    private async Task Init ()
    {
      _conferencesListView = new ListView { 
        HorizontalOptions = LayoutOptions.FillAndExpand,
        VerticalOptions = LayoutOptions.FillAndExpand,
      };

      var cell = new DataTemplate (typeof(TextCell));
      cell.SetBinding (TextCell.TextProperty, "Name");
      cell.SetBinding (TextCell.DetailProperty, new Binding (path: "Start", stringFormat: "{0:MM/dd/yyyy}"));

      _conferencesListView.ItemTemplate = cell;

      var viewModel = new ConferencesViewModel ();
      await viewModel.GetConferences ();
      _conferencesListView.ItemsSource = viewModel.Conferences;

      this.Content = new StackLayout {
        VerticalOptions = LayoutOptions.FillAndExpand,
        Padding = new Thickness (
          left: 0, 
          right: 0, 
          bottom: 0, 
          top: Device.OnPlatform (iOS: 20, Android: 0, WinPhone: 0)),
        Children = { 
          _conferencesListView 
        }
      };
    }
  }
}
```

![Apps](/blog/docs/assets/Apps.png)

In just a few lines of code we were able to create a fully native app for three different platforms. We leveraged the wealth of fantastic open source PCL libraries available on Nuget to quickly connect to a remote service, download json data, convert it to a model for our app, and save it to a local database. We then wired everything up and bound the data to our UI.

This is not the only solution available to you, but it's my preferred approach and is just one of the reasons I find working with C# and Xamarin such a joy.

[View the code on Github](https://github.com/RobGibbens/DtoToVM)
