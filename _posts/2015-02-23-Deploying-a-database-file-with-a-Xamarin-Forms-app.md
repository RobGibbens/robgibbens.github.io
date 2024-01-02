---
title: "Deploying a database file with a Xamarin.Forms app"
date: 2015-02-23
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

# Deploying a database file with a Xamarin.Forms app

It's very common for a mobile application to utilize a local sqlite database. The combination of [Sqlite.net](http://developer.xamarin.com/guides/cross-platform/application_fundamentals/data/part_3_using_sqlite_orm/) and Xamarin.Forms makes this [very easy](http://developer.xamarin.com/guides/cross-platform/xamarin-forms/working-with/databases/). One of the reasons that we choose Sqlite as our mobile database is that it's a single file and easily works cross platform. The same file can be created and opened on iOS, Android, Windows, Mac, Windows Phone, and WinRT.

Sometimes we may want to create our database from scratch the first time our application runs. Sqlite.net will automatically create the file if it doesn't exist the first time we attempt to connect to it. We can then use the **CreateTable<TModel>()** method on the **SQLiteConnection** class to create our tables from our C# models.

Other times though, we will want to ship a prepopulated database with the application. We may have some lookup tables that need have records in them, or default data that the application needs in order to function. Fortunately, this is easy to accomplish as well.

> Sample code is available at my [Github repo](https://github.com/RobGibbens/DbPublish)

## The steps

- Create the database file
- Link the database file to each platform specific project
- Copy the database file from the application bundle to a writable location on the file system
- Open the database file for reading and writing

## Create SQLite database

![Create SQL Schema](/blog/docs/assets/SqlSchema.png)

We can create a Sqlite database file using a variety of tools on both Mac and Windows. I use [DB Browser for SQLite](http://sqlitebrowser.org/), a cross platform tool which will allow us to create the database, define the schema, and add records to the database. For this example, we'll be creating a file named "**people.db3**".

## Create the Model

When using Sqlite.net, it's important that the C# models match the table definitions. If there is a discrepancy between the model and the table, we will have unexpected results. We may accidently create new tables, new columns in existing tables, or get no results from queries.

```csharp
[Table ("people")]
public class Person  
{
  [PrimaryKey, AutoIncrement, Column("Id")]
  public int Id { get; set; }

  [MaxLength (250), Unique, Column("Name")]
  public string Name { get; set; }
}
```

## Link database file

Once we have created the database on our desktop, we need to include it with each platform specific project (iOS/Android/WP8). We can keep the file anywhere on the desktop's file system, although including it within our solution in source control is recommended. We're going to use [File Linking](http://blogs.msdn.com/b/jjameson/archive/2009/04/02/linked-files-in-visual-studio-solutions.aspx) to include the exact same file in each project.

### iOS

For iOS, we will link the db3 file into the Resources folder. Be sure to set the Build Action to **BundleResource**.

![Link database file to iOS](/blog/docs/assets/IncludeIOSDb-1.png)

### Android

On Android, we will link the db3 file into the Assets folder, and set the Build Action to **AndroidAsset**.

![Link database file to Android](/blog/docs/assets/IncludeAndroidDb.png)

### Windows Phone 8

Windows Phone will link the database file into the root of the project, and set the Build Action as **Content**.

![Link database file to Windows Phone 8](/blog/docs/assets/IncludeWP8Db.png)

## Copy the database file

Although we've now included the database file with our application, and it will be part of the bundle that gets installed on the user's device, the app bundle and its included files are read-only. In order to actually change the database schema or add new records we'll need to copy the file to a writable location. Each device contains a special folder for each app, known as the app sandbox, to store files. We'll use each platform's file APIs to copy the file from the app bundle to the app sandbox.

### iOS

```csharp
[Register ("AppDelegate")]
public partial class AppDelegate : FormsApplicationDelegate  
{
  public override bool FinishedLaunching (UIApplication app, NSDictionary options)
  {
    Forms.Init ();

    string dbPath = FileAccessHelper.GetLocalFilePath ("people.db3");

    LoadApplication (new App (dbPath, new SQLitePlatformIOS ()));

    return base.FinishedLaunching (app, options);
  }
}
```

```csharp
public class FileAccessHelper  
{
  public static string GetLocalFilePath (string filename)
  {
    string docFolder = Environment.GetFolderPath (Environment.SpecialFolder.Personal);
    string libFolder = Path.Combine (docFolder, "..", "Library", "Databases");

    if (!Directory.Exists (libFolder)) {
      Directory.CreateDirectory (libFolder);
    }

    string dbPath = Path.Combine (libFolder, filename);

    CopyDatabaseIfNotExists (dbPath);

    return dbPath;
  }

  private static void CopyDatabaseIfNotExists (string dbPath)
  {
    if (!File.Exists (dbPath)) {
      var existingDb = NSBundle.MainBundle.PathForResource ("people", "db3");
      File.Copy (existingDb, dbPath);
    }
  }
}
```

### Android

```csharp
[Activity (Label = "People", Icon = "@drawable/icon", MainLauncher = true, ConfigurationChanges = ConfigChanges.ScreenSize | ConfigChanges.Orientation)]
public class MainActivity : global::Xamarin.Forms.Platform.Android.FormsApplicationActivity  
{
  protected override void OnCreate (Bundle bundle)
  {
    base.OnCreate (bundle);
    global::Xamarin.Forms.Forms.Init (this, bundle);

    string dbPath = FileAccessHelper.GetLocalFilePath ("people.db3");

    LoadApplication (new People.App (dbPath, new SQLitePlatformAndroid ()));
  }
}
```

```csharp
public class FileAccessHelper  
{
  public static string GetLocalFilePath (string filename)
  {
    string path = Environment.GetFolderPath (Environment.SpecialFolder.Personal);
    string dbPath = Path.Combine (path, filename);

    CopyDatabaseIfNotExists (dbPath);

    return dbPath;
  }

  private static void CopyDatabaseIfNotExists (string dbPath)
  {
    if (!File.Exists (dbPath)) {
      using (var br = new BinaryReader (Application.Context.Assets.Open ("people.db3"))) {
        using (var bw = new BinaryWriter (new FileStream (dbPath, FileMode.Create))) {
          byte[] buffer = new byte[2048];
          int length = 0;
          while ((length = br.Read (buffer, 0, buffer.Length)) > 0) {
            bw.Write (buffer, 0, length);
          }
        }
      }
    }
  }
}
```

### Windows Phone 8

```csharp
public partial class MainPage : global::Xamarin.Forms.Platform.WinPhone.FormsApplicationPage  
{
  public MainPage ()
  {
    InitializeComponent ();

    SupportedOrientations = SupportedPageOrientation.PortraitOrLandscape;

    global::Xamarin.Forms.Forms.Init ();

    string dbPath = FileAccessHelper.GetLocalFilePath ("people.db3");

    LoadApplication (new People.App (dbPath, new SQLitePlatformWP8 ()));
  }
}
```

```csharp
public class FileAccessHelper  
{
  public static string GetLocalFilePath (string filename)
  {
    string path = Windows.Storage.ApplicationData.Current.LocalFolder.Path;
    string dbPath = Path.Combine (path, filename);

    CopyDatabaseIfNotExists (dbPath);

    return dbPath;
  }

  public static void CopyDatabaseIfNotExists (string dbPath)
  {
    var storageFile = IsolatedStorageFile.GetUserStoreForApplication ();

    if (!storageFile.FileExists (dbPath)) {
      using (var resourceStream = Application.GetResourceStream (new Uri ("people.db3", UriKind.Relative)).Stream) {
        using (var fileStream = storageFile.CreateFile (dbPath)) {
          byte[] readBuffer = new byte[4096];
          int bytes = -1;

          while ((bytes = resourceStream.Read (readBuffer, 0, readBuffer.Length)) > 0) {
            fileStream.Write (readBuffer, 0, bytes);
          }
        }
      }
    }
  }
}
```

## Running the app

By including our prepopulated database with our app, we have the ability to give our users a better first run experience and minimize the amount of work that the app needs to do to be able to run for the first time.

![Included data](/blog/docs/assets/iOSDB.png)

## Source Code

> _You can find a complete sample on my [Github repo](https://github.com/RobGibbens/DbPublish)_
