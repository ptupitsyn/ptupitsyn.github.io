---
layout: post
title: Cross-Platform Desktop .NET UI with Avalonia
---

Cross-platform .NET is all the rage now. You can do console apps in C# and F#, 
web with ASP.NET Core, mobile with Xamarin. But can you do desktop UI apps?

![UI demo gif](../images/let-it-snow-tree.gif)

# Enter Avalonia

.NET Core by itself does not provide any UI framework, and Microsoft does not seem to bother with this.
They are mostly trying to appeal to web developers, and this is understandable: web apps are much more popular.

With Mono you could at least use Windows Forms, but Mono is old stuff.

Turns out there is [Avalonia UI](https://github.com/AvaloniaUI/Avalonia) project, and it is pretty decent!
* WPF-inspired, so existing XAML experience translates easily (though there are some [differences](http://avaloniaui.net/tutorial/from-wpf.html))
* Supports .NET Core
* Works on Windows, Linux, macOS, and even Android/iOS with Mono

So I've decided to try it out. And to make things interesting, let's do an old-school DOS demoscene inspired falling snow effect!

# Creating the UI

First, install [Visual Studio Extension](https://marketplace.visualstudio.com/items?itemName=AvaloniaTeam.AvaloniaforVisualStudio).
Create a project with the installed project template. It is a bit outdated, so we'll have to change 
`<TargetFrameworks>netcoreapp1.1;net461</TargetFrameworks>` to `<TargetFramework>netcoreapp2.0</TargetFramework>` in the csproj file.

We are presented with regular WPF files: `App.xaml`, `MainWindow.xaml`, and their code-behind.
XAML Designer did not work for me, but who uses it, anyway?

On the screenshot above you can see the animation itself and some controls: button, sliders, and a `ListBox` for colors.
Everything is pretty standard, `ItemsControl` with `DataTemplate` works as you would expect, I did not encounter any difficulties.

MVVM and bindings are also very similar to WPF.
I've encountered a bug with two-way Slider value binding and had to switch to the nightly build.

# Snow Rendering

In XAML there is an `<Image Source="{Binding Bitmap}" />`, and `SnowViewModel.Bitmap` is a
`WriteableBitmap`, a concept well-known from WPF. You can even write to it from any thread, which simplifies rendering.

`InvalidateVisual` has to be called on the `Image` control in order to refresh the contents after
underlying `WriteableBitmap` has been updated.

Drawing is achieved like this:

```cs
using (var buf = Bitmap.Lock())
{
    var ptr = (uint*) buf.Address;
    *ptr = 255;  // Change top-left pixel.
}
```

# How to Run?

Tested on Windows, Linux, and macOS:
* Install [.NET Core SDK](https://www.microsoft.com/net/download)
* `git clone https://github.com/ptupitsyn/let-it-snow.git`
* `cd let-it-snow/AvaloniaCoreSnow`
* `dotnet run`


# Conclusion

Even though Avalonia is still in alpha, it worked pretty well for me right away,
and with prior WPF experience I could easily achieve the desired result. Impressive!

Alternatives?
* Qt (a bit more complicated)
* Java (lacks pointers)
* Electron (oh god)

UI in this demo is quite simple, but it uses a number of essential features:
* Layout (Grid, StackPanel)
* Bindings, MVVM
* ItemsControl
* WriteableBitmap

This is already enough to build an UI of any complexity.

---
* Source code: https://github.com/ptupitsyn/let-it-snow
* This article in Russian: [Кроссплатформенная новогодняя демка на .NET Core и Avalonia](https://habrahabr.ru/post/345708/)