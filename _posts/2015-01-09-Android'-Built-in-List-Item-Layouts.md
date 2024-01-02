---
title: "Android's Built-in List Item Layouts"
date: 2015-01-09
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

Android's [List View](http://developer.xamarin.com/guides/android/user_interface/working_with_listviews_and_adapters/) allows us to iterate over an enumerable collection and display each piece of data in a list item. The list view works in conjunction with an adapter to loop over the data and display the data in a layout. We could, and often do, create our own layouts for this purpose. Customizing the layout allows us to match the list view's look and feel to the rest of our app, and to tailor the fields and controls that are shown.

For the times where we just want to display some simple data on the screen though, Android does include some [built-in list item layouts](http://developer.android.com/reference/android/R.layout.html).  I was having a hard time finding any good documentation of these built-in layouts, so I have created a [sample app](https://github.com/RobGibbens/ListViewDemo) displaying as many of the layouts as I could figure out.  The app is written using [Xamarin.Android](http://android.xamarin.com).

All of the code is available from my [Github repo](https://github.com/RobGibbens/ListViewDemo)


### ArrayAdapter with SimpleListItem1

The simplest adapter to use is the built-in ArrayAdapter, using the built-in Android.Resource.Layout.SimpleListItem1 layout. This layout contains a single TextView, allowing display of a single piece of text.

```csharp
using System;
using Android.App;
using Android.OS;
using Android.Widget;

namespace ListViewDemo
{
  [Activity (Label = "ExampleActivity")]
  public class ExampleActivity : ListActivity
  {
    protected override void OnCreate (Bundle savedInstanceState)
    {
      base.OnCreate (savedInstanceState);

      var kittens = new [] { "Fluffy", "Muffy", "Tuffy" };

      var adapter = new ArrayAdapter (
                this, //Context, typically the Activity
                Android.Resource.Layout.SimpleListItem1, //The layout. How the data will be presented 
                kittens //The enumerable data
                );

      this.ListAdapter = adapter;
    }
  }
}
```

The other layouts include (with the controls available)

### Android.Resource.Layout.ActivityListItem

- 1 ImageView (Android.Resource.Id.Icon)
- 1 TextView (Android.Resource.Id.Text1)

![ActivityListItem](/blog/docs/assets/ActivityListItem.png)
  
### Android.Resource.Layout.SimpleListItem1

- 1 TextView (Android.Resource.Id.Text1)

![SimpleListItem1](/blog/docs/assets/SimpleListItem1.png)

### Android.Resource.Layout.SimpleListItem2

- 1 TextView/Title (Android.Resource.Id.Text1)
- 1 TextView/Subtitle (Android.Resource.Id.Text2)

![SimpleListItem2](/blog/docs/assets/SimpleListItem2.png)

### Android.Resource.Layout.SimpleListItemActivated1

- 1 TextView (Android.Resource.Id.Text1)
- Note : Set choice mode to multiple or single
  
```csharp
  this.ListView.ChoiceMode = ChoiceMode.Multiple;
```

![SimpleListItemActivated1](/blog/docs/assets/SimpleListItemActivated1.png)

### Android.Resource.Layout.SimpleListItemActivated2

- 1 TextView (Android.Resource.Id.Text1)
- 1 TextView/Subtitle (Android.Resource.Id.Text2)
- Note : Set choice mode to multiple or single
  
```csharp
  this.ListView.ChoiceMode = ChoiceMode.Multiple;
```

![SimpleListItemActivated2](/blog/docs/assets/SimpleListItemActivated2.png)

### Android.Resource.Layout.SimpleListItemChecked

- 1 TextView (Android.Resource.Id.Text1)
- Note : Set choice mode to multiple or single
  
```csharp
  this.ListView.ChoiceMode = ChoiceMode.Multiple;
```

![SimpleListItemChecked](/blog/docs/assets/SimpleListItemChecked.png)

### Android.Resource.Layout.SimpleListItemMultipleChoice

- 1 TextView (Android.Resource.Id.Text1)
- Note : Set choice mode to multiple or single
  
```csharp
  this.ListView.ChoiceMode = ChoiceMode.Multiple;
```

![SimpleListItemMultipleChoice](/blog/docs/assets/SimpleListItemMultipleChoice.png)

### Android.Resource.Layout.SimpleListItemSingleChoice

- 1 TextView (Android.Resource.Id.Text1)
- Note : Set choice mode to single
  
```csharp
  this.ListView.ChoiceMode = ChoiceMode.Single;
```

![SimpleListItemSingleChoice](/blog/docs/assets/SimpleListItemSingleChoice.png)

### Android.Resource.Layout.TestListItem

- 1 TextView (Android.Resource.Id.Text1)

![TestListItem](/content/images/2014/Jul/TestListItem.png)
 
### Android.Resource.Layout.TwoLineListItem

- 1 TextView/Title (Android.Resource.Id.Text1)
- 1 TextView/Subtitle (Android.Resource.Id.Text2)

![TwoLineListItem](/blog/docs/assets/TwoLineListItem.png)

Again, checkout the Xamarin.Android sample app on my [Github repo](http://github.com/RobGibbens/ListViewDemo) to see these list view layouts in action.
