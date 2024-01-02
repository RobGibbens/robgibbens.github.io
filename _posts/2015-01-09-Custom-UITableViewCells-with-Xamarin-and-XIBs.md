---
title: "Custom UITableViewCells with Xamarin and XIBs"
date: 2015-01-09
---
> <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path d="M13 17.5a1 1 0 11-2 0 1 1 0 012 0zm-.25-8.25a.75.75 0 00-1.5 0v4.5a.75.75 0 001.5 0v-4.5z"></path><path fill-rule="evenodd" d="M9.836 3.244c.963-1.665 3.365-1.665 4.328 0l8.967 15.504c.963 1.667-.24 3.752-2.165 3.752H3.034c-1.926 0-3.128-2.085-2.165-3.752L9.836 3.244zm3.03.751a1 1 0 00-1.732 0L2.168 19.499A1 1 0 003.034 21h17.932a1 1 0 00.866-1.5L12.866 3.994z"></path></svg> **Note**
> This blog is _woefully_ out of date, and is here simply as an archive

Sometimes when we are creating an iOS app with Xamarin, we choose to forego using the Storyboard designer, and simply create the user interface in C#. This works fine, but when using a **UITableView**, there are times when we want to design the **UITableViewCell** with the designer. Luckily, this is easy enough to do using a single .xib file in Xamarin Studio. 

I have created a sample project in my [Github repo](https://github.com/RobGibbens/XibTableCellDesign) to demonstrate using a *.xib* file to design our cell.

#### Create the project

For this sample, we will start off with the Empty Project template.

> File -> New Solution -> iOS -> iPhone -> Empty Project

#### Create the table view

Next, add a new class for the **UITableViewController**, cleverly named _MyTableViewController.cs_.  In the _AppDelegate.cs_, set the RootViewController to our new class. In **MyTableViewController**, we'll load our data and wire the TableView.Source to **MyTableViewSource**. 

```csharp
public override void ViewDidLoad ()
{
    base.ViewDidLoad ();

    var conferences = ConferenceRepository.GetConferences ();

    TableView.Source = new MyTableViewSource (conferences);
}
```

#### Create the cell

At this point, we have not used a designer for any of our UI. In fact, we don't even have a _.storyboard_ or _.xib_ in our project. In order to design our table view cell, we're going to add one now.  We'll add a new file using the _iPhone Table View Cell_ template, and name it _MyCustomCell.xib_.

> Add -> New File -> iOS -> iPhone Table View Cell

![iPhoneTableViewCell](/blog/docs/assets/iPhoneTableViewCell.png)

Xamarin Studio does not support designing _.xib_ files, but double clicking on the file will open it in Xcode and allow us to design the cell.

As we drag our controls on to the design surface, we have to make sure that we wire up the **Outlet** in the header file. In Xcode, we do this by ctrl-click-dragging the control into the header file. Without the Outlets, we wouldn't be able to access the controls in our C# code.

![xcode](/blog/docs/assets/xcode.png)

Save the _.xib_ and quit Xcode. Xamarin Studio will detect the change, and update the _MyCustomCell.designer.cs_ file with the new Outlets.

```csharp
using MonoTouch.Foundation;
using System.CodeDom.Compiler;

namespace XibTableCellDesign
{
    [Register ("MyCustomCell")]
    partial class MyCustomCell
    {
        [Outlet]
        MonoTouch.UIKit.UILabel ConferenceDescription { get; set; }

        [Outlet]
        MonoTouch.UIKit.UILabel ConferenceName { get; set; }

        [Outlet]
        MonoTouch.UIKit.UILabel ConferenceStart { get; set; }

        void ReleaseDesignerOutlets ()
        {
            if (ConferenceDescription != null) {
                ConferenceDescription.Dispose ();
                ConferenceDescription = null;
            }

            if (ConferenceName != null) {
                ConferenceName.Dispose ();
                ConferenceName = null;
            }

            if (ConferenceStart != null) {
                ConferenceStart.Dispose ();
                ConferenceStart = null;
            }
        }
    }
}
```

#### Add the model

Notice that these Outlets are private. We will not be able to access the controls from our **UITableViewSource** GetCell method, which is where we normally set our control properties, such as label.Text = value. We can get around this by adding a public property in the custom cell to set with our model data.

```csharp
public partial class MyCustomCell : UITableViewCell
{
    public Conference Model { get; set; }
}
```

Then, in the LayoutSubviews method of our custom cell, we can set the control properties to the model object's properties.

```csharp
public override void LayoutSubviews ()
{
    base.LayoutSubviews ();

    this.ConferenceName.Text = Model.Name;
    this.ConferenceStart.Text = Model.StartDate.ToShortDateString ();
    this.ConferenceDescription.Text = Model.Description;
}
```

#### Use the cell

The last step is to create and use the cell in our table view source's GetCell method.

```csharp
public override UITableViewCell GetCell (UITableView tableView, NSIndexPath indexPath)
{
    var conference = _conferences [indexPath.Row];
    var cell = (MyCustomCell)tableView.DequeueReusableCell (MyCustomCell.Key);
    if (cell == null) {
        cell = MyCustomCell.Create ();
    }
    cell.Model = conference;

  return cell;
}
```

![CustomTableViewCells](/blog/docs/assets/CustomTableViewCells-1.png)

Checkout the sample app on my [Github repo](https://github.com/RobGibbens/XibTableCellDesign) for an example of designing a table view cell with Xamarin Studio and *.xib* files.
