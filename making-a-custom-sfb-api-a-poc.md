In this post, we have multiple antagonists, two victims (my sanity and the loss of almost an entire Saturday), a lost voice from screaming in frustration and an outcome. It's a journey (at least from my perspective), but a potentially viable outcome.

# Intro

In working on a project in my day job, I ran into the need to create a Resource Account from a web portal I was developing. I figured that it'd be pretty straightforward because there'd be a REST API endpoint somewhere I could hit, right??

WRONG*

(* - as far as I could tell...)

The Microsoft Graph API, as fantastic as it is, doesn't have any ways to manage and configure Skype for Business on the management side. So I thought I'd use the Skype for Business API, but that's also a deadend. 

The Skype for Business API's are about developing solutions for client use, such as searching users and getting status update changes. So where too next?

# Plan Un
Authentication for Skype for Business (and everything O365), is OAuth2 based, so I thought "can I get an access token easily and then see what endpoints the PowerShell commands hit and see if I can replicate that? The answer to that was "yes, but no". I figured out how to get a token after butchering the `New-CsOnlineSession` command to output the token it gets, but when I ran `Get-CsOnlineUser -Identity name@example.com`, Fiddler didn't show me any HTTP calls. 

## An a-ha moment?
I was about to abandon the plan until I remembered that unless you explicitly set the proxy in your PowerShell session, it'll by pass it. So after setting the PowerShell window to have a proxy, I ran `Get-CsOnlineUser -Identity name@example.com` and then discovered...

Skype for Business is SOAP XML and not JSON. So now we definitely abandon the plan.

## What do the modules say?
I dove into the imported modules that get downloaded when you run `$session = New-CsOnlineSession;Import-PsSession $session` and spotted that it uses `Invoke-Command` against `adminau1.online.lync.com`. I thought that since I know how to get a token, I should be able to create a credential object with the access token and get a response? Should is the operative word there. I got the following error;

```
[adminau1.online.lync.com] Connecting to remote server adminau1.online.lync.com failed with the following error message : WinRM cannot complete the operation. Verify that the specified computer name is valid, that the computer is accessible
over the network, and that a firewall exception for the WinRM service is enabled and allows access from this computer. By default, the WinRM firewall exception for public profiles limits access to remote computers within the same local
subnet
```

Okay, it was worth a try. My theory here is that the imported PowerShell commands leverage the PowerShell session that was established for authentication and it probably wasn't going to get me much further ahead going down this path.

# Plan Duex
Google-fu was failing me on finding me my silver bullet (there's a chance said silver bullet doesn't exist), so I go to plan 2: build a Function App. I know I can upload PowerShell modules into Function Apps as I'd done that before on a couple of Proof of Concepts.

To test my theory, I'm just using a Consumption Plan Function App. Consumption Apps are charged per second of utilisation and you get 1,000,000 free requests a month, so I'm unlikely to ever exceed the free request quota, however after a period of inactivity, the Function App goes to sleep. When it does that, I'll take my connected PowerShell session with it.

Even if I put the Function App in an App Service that will run 24/7, the session expires after an hour, so that needs to be handled. In my experience, the only way to get going again after a session disconnect is to remove the PowerShell session and reconnect again. I can either configure the Function App to re-authenticate every 59 minutes, or just have it re-authenticate at the next attempted usage and accept it'll take a few moments for it to get going again. For now, we'll just authenticate every time we hit the Function App.

PowerShell Function Apps are now a separate Runtime Stack in Azure, so it's a fully fledged, supported platform for writing quick API's and webhook functionality. I practically live in PowerShell in my day job, so I'm a very big fan of this.

First step is to ~~kill all the lawyers~~ create a folder called `Modules` at `/wwwroot`. This is assuming you are using .NET Core Functions, with  Runtime as `v2` and not Runtime as `v1`. Runtime v1 you need to place modules in the root of each function. You can do this in the Kudu management interface under Debug Console, then select either CMD or PowerShell.

## Looking good!
After creating the Modules folder and then the subsequent `SkypeOnlineConnector` folder and uploading all the content files (you can drag and drop from the source folder on Windows into your browser), we try to import the module;

```powershell
try {
    Import-Module "SkypeOnlineConnector" -ErrorAction Stop
    Write-Output "Imported Connector Module"
} catch {
    Write-Output "Failed to import Connector Module: [$($_.Exception.Message)]"
}
```

And we get...`[Information] OUTPUT: Imported Connector Module`. Excellent! Now we try to create an online session

## Computer says nooooooo...

```
[Error] EXCEPTION: Get-CsOnlinePowerShellAccessToken : One or more errors occurred. (Could not load type 'System.Security.Cryptography.SHA256Cng' from assembly 'System.Core, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.)
```

Sod it. The issue we have here is `System.Security.Cryptography.SHA256Cng` is not available in .NET Core, which is what PowerShell Function Apps are run off. There's no option to use the full .NET Framework for PowerShell in a Function App.

# Part Trois
Years and years ago when I first starting playing with PowerShell, I built a WinForms project that was a File System ACL generator that executed PowerShell scripts. The reason for that was, manipulating ACL's was far easier for me in PowerShell than in C# and building a GUI in Visual Studio was easier than doing it in PowerShell. I've since learnt about WPF and that certainly would have made my life easier back then.

However, C# code executing PowerShell code can be done. And thy will be done. However at this point, I just bought a Switch Lite and Animal Crossing, so I did take a short break (followed by a nap) and came back to it later in the day.

I've built plenty of .NET Core C# Function Apps, but never a .NET Framework C# Function App but upon the launch of a new Visual Studio Project for one, it looks about the same as a .NET Core one.

To see if this will work the way I think it will, we have a very basic function;

```c#
public static async Task<HttpResponseMessage> Run([HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)]HttpRequestMessage req, TraceWriter log)
{
    log.Info("C# HTTP trigger function processed a request.");

    // parse query parameter
    string name = req.GetQueryNameValuePairs()
        .FirstOrDefault(q => string.Compare(q.Key, "name", true) == 0)
        .Value;

    if (name == null)
    {
        // Get request body
        dynamic data = await req.Content.ReadAsAsync<object>();
        name = data?.name;
    }

    using (PowerShell psInstance = PowerShell.Create())
    {
        psInstance.AddScript("Get-Module");
        Collection<PSObject> psOutput = psInstance.Invoke();

        foreach (PSObject item in psOutput)
        {
            if (item != null)
            {
                var blah = item;
            }
        }
    }

    return name == null
        ? req.CreateResponse(HttpStatusCode.BadRequest, "Please pass a name on the query string or in the request body")
        : req.CreateResponse(HttpStatusCode.OK, "Hello " + name);
}
```

Testing that out and...success!!! I can see all the running processes on my development desktop.

Inside the Visual Studio Project, I created a folder called `Scripts` and a small PowerShell script that will import the module and try to connect.

I fumble together a small piece of code that from what I can understand, should successfully execute PowerShell;

```c#
Process powershell = new Process
{
    StartInfo = new ProcessStartInfo
    {
        FileName = "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
        Arguments = "D:\\home\\data\\test-script.ps1",
        UseShellExecute = true,
        RedirectStandardOutput = true,
        RedirectStandardError = false,
        CreateNoWindow = true
    }
};

powershell.Start();

while (!powershell.StandardOutput.EndOfStream)
{
    log.Info(powershell.StandardOutput.ReadLine());
}
```

The Script just dumps "Hello World" into a text file. Testing this locally worked! I had the file in the directory I specified. So I deploy to my Function App, test it and...a 500 error...

## Son de la @%$#!
I've just realised that there isn't a C:\ on Function Apps, it's a D:\...so we change C:\ to D:\, republish and test and...successful run! But no text file was created :(

Is the issue that we can't save files in a folder where it's a pre-compiled binary? So I change the output path to `D:\local\Temp`, republish and test. Another successful run but no text file.

Is redirecting stdout to the log causing the issue? Didn't appear to be, but I was getting an error and I was struggling to trap it.

I was struggling with testing this locally and deploying, so I reverted back to a .NET Core C# Function App and then discovered what my issue was. I was saving data to a location that is transient. As such, I found that `D:\home\data` is recommended (https://github.com/Azure/azure-functions-host/issues/3626#issuecomment-431485622). If you want it to be transient and only have it necessary for during the functions run time, you can use `%TEMP%`.

The PowerShell script started to execute, then I get met with `The Win32 internal error “The handle is invalid” 0x6 occurred while getting the console mode`. A quick search suggested that this occurs when commands use `Write-Progress` and when there's no interactive window, it doesn't like it. I tried the solution here (https://stackoverflow.com/questions/33401244/the-win32-internal-error-the-handle-is-invalid-0x6-occurred-while-getting-the) and we had success!!

I finally had a file! Executing the actual script I wanted to get a user out of Skype for Business Online was...successful!!!

Now, the script was hardcoded with a username, so that needs to be able to accept parameters that come in from the Function App. After modifying that and a couple of tweaks to the C# Function App, we have the following set of code;

` place working code here `
