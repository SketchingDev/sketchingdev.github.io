---
layout: post
title: "C# ASP.NET MVC 3 Hyperlink Helper"
date: 2012-04-19 00:00:00
categories: mvc csharp dotnet
---

The other day whilst tidying the views in my personal [ASP.NET MVC 3](http://www.asp.net/mvc) (C#) project I ended up developing a HTML Helper that creates external hyperlinks, in an attempt to purge my views of hard-coded anchor tags. I've provided an example of it being used, along with a link to the source-code for anyone who is interested.

The HyperlinkExtensions class consists of two extension methods for the HtmlHelper class; allowing the Helpers to work just like the standard HTML Helpers.

## Example usage

```csharp
@Html.Hyperlink("https://eff.org/", "EFF",
    new Dictionary<string, object>()
    {
        {"title", "Electronic Frontier Foundation"}
    }
)
// Output: <a href="https://eff.org/" title="Electronic Frontier Foundation">EFF</a>
```

## Download

[View the Hyperlink Extensions class](https://gist.github.com/2407487).

### Configuration

If you're unsure how to configure your project to use the class then follow the steps below.

1. Add the class to your project
2. Add its namespace to the [Pages](http://msdn.microsoft.com/en-us/library/950xf363%28v=vs.100%29.aspx) element in your project's `web.config` file - below.
3. You should now be able to use the Hyperlink methods in your views.

```xml
<configuration>
  ...
  <system.web.webPages.razor>
	...
	<pages pageBaseType="System.Web.Mvc.WebViewPage">
	  <namespaces>
		...
		<add namespace="ExampleProject.Helpers" />
	  </namespaces>
	</pages>
  </system.web.webPages.razor>
</configuration>
```
