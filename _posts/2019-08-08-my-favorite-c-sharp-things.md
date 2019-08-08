---
layout: post
permalink: /my-favorite-c-sharp-things.html
title:  "My Favorite C# Things"
date:   2019-08-08 8:30:07 -0500
categories: work .net c# code
---
C# and .NET have become my very favorite tools for developing applications.  The Visual Studio environment brings with it a sense of completeness I have rarely felt in other IDEs.

Working with [Xamarin](https://en.wikipedia.org/wiki/Xamarin){:target="_blank"} has given me an additional perspective as I've had to port Java, Swift and Objective C code into C# on many occasions.  Porting all that code has taught me many things about C# naming conventions.

## Getters, Setters and Automatic Properties

{% highlight csharp %}
class MyThing {
	
	public string Name { get; set; }

}
{% endhighlight %}

What's going on here?  This is our friend the automatic property.  What we have here defines a _Name_ on a _MyThing_ instance with a public getter and a private setter.  Why do we have to set them at all?  By adding `{ get; set; }` we are denoting whether or not this is a property or a field.  A property we cannot pass into any functions using a `ref` or `out` reference.

We can also improve information hiding with access modifiers, for instance: `{ get; private set; }`.  Now our property is only modifiable internally to the class instance.

The fun doesn't stop there though, we can also explicitly define these setters and getters.

{% highlight csharp %}
class MyThing {
	
	private int timesViewed;

	private string name;
	public string Name {
		get {
			timesViewed++;
			return name;
		}
		set {
			name = value;
		}
	}

	private IRuler ruler;
	public Size Size {
		get {
			if (ruler == null) {
				ruler = new Ruler();
			}
			return ruler.Measure();
		}
	}

}
{% endhighlight %}

With the name property we are now tracking the amount of times the property is being accessed automatically.  We only need another private field to store the actual value.

Below that we have a Size property which is also named in the same way as its class, _Size_.  This is a valid naming convention if you can can't think of anything better and the compiler will respond in kind.  Size is a volatile property and relies on an instance of the _Ruler_ class.  Our handy getter now performs the job of ensuring existence of our ruler before we go and measure our _MyThing_.

## Null-safe Operations

### Safe Properties

We've all done this.

{% highlight csharp %}
if (person.Friend != null) {
	person.Friend.Visit();
}
{% endhighlight %}

Great, but what if _person_ is null as well?  We end up with something like this.

{% highlight csharp %}
if (person != null) {	
	if (person.Friend != null) {
		person.Friend.Visit();
	}
}
{% endhighlight %}

We can simplify this, as long as we're not relying on an exception to be handled.

{% highlight csharp %}
if (person?.Friend != null) {
	person.Friend.Visit();
}
{% endhighlight %}

Now, if either is null we'll get the desired result without an additional level of null checking.  There's plenty of other great use cases for this too.  I've chopped down control statements from 30 lines to 4 with this logic.

### Safe Casting

Sometimes you don't want to end execution is something fails to cast.  This will throw an _InvalidCastException_ if _obj_ isn't the appropriate type.

{% highlight csharp %}
var teacher = (Teacher)person;
{% endhighlight %}

This, on the other hand, will simply result in null if the cast is invalid.

{% highlight csharp %}
var teacher = person as Teacher;
{% endhighlight %}

While there are better ways to identify if something can be casted as a certain type, like via [Reflection](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/reflection){:target="_blank"}, this serves as a handy operation for validating objects in an inheritance heirarchy.

## Dependency Injection

Here's a fun concept that's more inherent to .NET than C# itself, but it's very applicable to Xamarin, so let's discuss.  I like to think of Dependency Injection as a super interface.  We create an interface that defines behavior as usual and apply it across platforms and objects.  Let's look at an example of drawing a rounded rectangle on a canvas in iOS and Android.  Normally we'd use custom renderers, but this will help demonstrate the concept.

### Shared Project

{% highlight csharp %}
interface IDraw {

	void DrawRect(int left, int top, int width, int height);

}

class SharedDraw : IDraw {

	public void DrawRoundRect(int left, int top, int width, int height) {
		DependencyService.Get<IDraw>().DrawRoundRect(left, top, width, height)
	}

}
{% endhighlight %}

### Android

{% highlight csharp %}
class CustomView : View, IDraw {

	Canvas canvas;

	public void DrawRoundRect(int left, int top, int width, int height) {
		var shape = new Paint(PaintFlags.AntiAlias);
		shape.Color = Android.Graphics.Color.Black;

		// Version specific drawing API
		if (Android.OS.Build.VERSION.SdkInt >= Android.OS.BuildVersionCodes.Lollipop)
            canvas.DrawRect(left, top, left + width, top + height, shape);
        else
            canvas.DrawRect(new RectF(left, top, left + width, top + height), shape);
	}

}
{% endhighlight %}

### iOS

{% highlight csharp %}
class CustomView : UIView, IDraw {

	public void DrawRoundRect(int left, int top, int width, int height) {
		var rect = new CGRect(left, top, width, height);

		using (var g = UIGraphics.GetCurrentContext())
		{
			var layer = CGLayer.Create(g, Bounds.Size);
            layer.Context.SetFillColor(UIColor.Black.CGColor);
            layer.Context.FillRect(rect);
			g.DrawLayer(layer, new CGPoint(0, 0));
		}
	}

}
{% endhighlight %}

## LINQ

Saving the best for last, Language Integrated Queries are nothing new, but boy do they deserve a spotlight.  With `System.Linq`, you gain an interface that permits select, order and even iterate operations for `IEnumerables` all with function chaining.  Consider the following.

{% highlight csharp %}
var staffEmails = allUsers.Where(u => u.Roles.HasFlag(UserRole.Staff))
                          .OrderBy(u => u.LastName)
						  .Select(u => u.Email)
						  .ToList();
{% endhighlight %}

Nothing crazy there, but we managed to filter on a bitwise field, order and pick out a specific property from our data set and reform it back into a usable list of strings all in one go.

{% highlight csharp %}
var staffDtos = new List<StaffDto>();
allUsers.Where(u => u.Roles.HasFlag(UserRole.Staff))
        .ForEach(u => staffDtos.Add(new StaffDto(u));
{% endhighlight %}

Here we transformed our user list into a list of staff dto objects.  That's just a beautiful looking loop structure.  Filter, dto conversion and list population in three lines.  Where does that go bad.

## Easy String Tokenization

The goto method for building atomic strings for a long time was `string.Format`.  Don't get me wrong, it still has its uses, but it reeks of age and reminds me of your basic `sprint` functions.  Here's how I prefer to do it nowadays.

{% highlight csharp %}
var messageToStaff = $"Hello All, I wanted to let every {UserRole.Staff.ToString()} member know that we have a new feature called {newFeature.Name}.  This was released on {newFeatureDate.ToString("M/dd/yyyy")} and we'd like to get your feedback.";
{% endhighlight %}

It's similar, but now we don't have to create tokens as indexed numbers and pass in umteen arguments to populate those tokens.  We have context now and it's just so much more readable.

## Wrapping Up

Sometimes it's really nice to sit down and review the language you've been using for years.  Go back and look at the fundamentals and then the new hotness.  You'll gain a new found appreciation for syntax you've used all along or have only just discovered.  Don't be afraid to audit what you know because in the end it's only going to pay dividends.