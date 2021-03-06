---
layout: post
title: Fuzzing interlude
tags: [hacking]
---


## Fuzzing Interlude

As I was doing all of the above I realized I was ready to start some vulnerability hunting. We'll start with the basics and work our way into more and more complicated stuff. Kernel-land, despite having a lot of stuff to learn this is kinda random (64 byte aligned shit? wut?) is actually pretty simple once you've got the hang of it. The trick is to just look for some fairly simple patterns in assembly or a decompiler. But before we get into that, we want to maximize our hunting. If you've never seen my office I probably have circa 60-70 cores, most of them sitting there cold. So let's start fuzzing some stuff! The one trick I have for fuzzing is to google if something has ever been fuzzed. If it has, hit another part of the application. If it hasn't, hit the simplest possible part. One thing I haven't seen fuzzed that seems pretty complicated is explorer.exe's ability to open a variety of different files. Go ahead and try it, it's sorta like gnome-open or xdg-open in Linux, you point it at a file and if the extension is registered with a program it pops it open in whatever is most appropriate. This functionality seems like it'd need to know how to parse a lot of different things, find functionality, and then open the file in some manner. That's a few steps, and anything with more than a step or two IMHO is a decent target. So let's get down and dirty and fuzz it. I'll be using Binary Ninja, for one reason: Ilfak. Ilfak has never been particularly kind or nice whenever I buy an IDA license (I've bought like 3, totaling about 10k), one time he was downright rude when I asked when my download would come in, and third Ilfak your wallet pretty hard if we use IDA (wah wahh). So let's go with Binary Ninja and Ghidra, I'll be using the former for the wonderful lighthouse plugin and the latter for its decompiler if I feel like I need some extra help. As a debugger I'll be using x64 debug and kind of a new technique to me - I'm going to pop open explorer with a few files that I know it's going to recognize, step through the code with x64dbg, and see what lighthouse lights up. I'll know that this chain of functions (and their arguments!) are what make it do its thing, and I'll stop after everything is parsed but before explorer.exe pops some window open. I have no idea if this is actually going to work, but wtf why not, it's easier than trying to figure out what's going on with just a debugger and a decompiler. I've used lighthouse before for code coverage and fuzzing, but never for this, so we'll see if it's up to the task. I want to make sure I document my failures here and don't just leave you all with the impression that hacking is just things that work, failing hard is always an option.

Ok let's get cracking, I've fired open Binary Ninja, cmder, and I have lighthouse installed as a plugin to Binary Ninja along with bncov (never used it but seems like a nice alternative to lighthouse if I for some reason have issues), and x64dbg. Right off the bat x64dbg comes in very very handy with downloading all the pdbs I need. Simply go to the "Symbols" tab, check out the imports, highlight all of them, right click, and click download all pdbs. Then I watched We Were Soldiers with Mel Gibson. This is an important step because symbols will take a while to download. Man, Vietnam was fucking stupid, fucking Lyndon Johnson am I right?? Anyway, back to hacking. I'm  going to need DynamoRio and a few iterations of running explorer.exe, then I'll see which functions pop out at me. It should be pretty easy to find the parsing function, and since I have symbols, things get even easier. So let's try it out. I ~~put on my robe and wizards hat~~ download DynamoRIO and get ready to use drcov. It's pretty simple to use, here's an example:

`drrun.exe -t drcov -- explorer.exe folder/`

This will pop open a folder, once again explorer has done some magic I don't yet understand, determined that I've passed it a folder, and opened the file explorer with that folder as the target. Again just complicated enough that there could be something juicy there. Let's try it out. But first things first, we need to know if explorer.exe being used by the system is 32 or 64 bit. I suspect it's 64 but what do I know? So I pop open process monitor and see which explorer.exe is being run (there's a few locations for it on the system). Looks like by default it uses `C:\windows\explorer.exe` so let's sigcheck it out:

```>>> sigcheck.exe C:\Windows\explorer.exe

Sigcheck v2.80 - File version and signature viewer
Copyright (C) 2004-2020 Mark Russinovich
Sysinternals - www.sysinternals.com

c:\windows\explorer.exe:
        Verified:       Signed
        Signing date:   6:00 PM 10/22/2020
        Publisher:      Microsoft Windows
        Company:        Microsoft Corporation
        Description:    Windows Explorer
        Product:        Microsoft« Windows« Operating System
        Prod version:   10.0.19041.610
        File version:   10.0.19041.610 (WinBuild.160101.0800)
        MachineType:    64-bit
C:\Users\punkpc\Desktop\cmder
```
OMG I WAS RIGHT ABOUT SOMETHING. Anyway, now we know that we need to use the 64-bit DR client. So let's do that. OK done, it wasn't that exciting:

```
C:\Users\punkpc\Desktop\DynamoRIO-Windows-8.0.0-1\bin64
Î»  .\drrun.exe -t drcov -- "C:\Windows\explorer.exe" "C:\Users\punkpc\Desktop\DynamoRIO-Windows-8.0.0-1\bin64"
C:\Users\punkpc\Desktop\DynamoRIO-Windows-8.0.0-1\bin64
Î»  ls


    Directory: C:\Users\punkpc\Desktop\DynamoRIO-Windows-8.0.0-1\bin64


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         12/7/2020   7:18 PM         207360 balloon.exe
-a----         12/7/2020   7:18 PM        4689920 balloon.pdb
-a----         12/7/2020   7:18 PM         120320 closewnd.exe
-a----         12/7/2020   7:18 PM        3829760 closewnd.pdb
-a----         12/7/2020   7:18 PM         123904 create_process.exe
-a----         12/7/2020   7:18 PM        3870720 create_process.pdb
-a----         12/7/2020   7:18 PM         236544 drconfig.exe
-a----         12/7/2020   7:18 PM        5058560 drconfig.pdb
-a----         12/7/2020   7:18 PM         246272 drconfiglib.dll
-a----         12/7/2020   7:18 PM           6920 drconfiglib.lib
-a----         12/7/2020   7:18 PM         273920 DRcontrol.exe
-a----         12/7/2020   7:18 PM        4878336 DRcontrol.pdb
-a----         12/7/2020   7:33 PM         510871 drcov.explorer.exe.19900.0000.proc.log
```

You see the log file at the bottom there gives me my code coverage. Well, while that was happening I remembered a talk by Alex Ionescu called "reversing without reversing" and I realized I fucking did it again - I jumped straight into a reversing tool and forgot to do the most basic thing - look some shit up. I Googled something like "How does Windows Explorer work MSDN" and found a pretty solid resource: https://docs.microsoft.com/en-us/windows/win32/shell/developing-with-windows-explorer.

Looks like Windows explorer has some interface definitions that are well documented. In fact it has a lot of them, they all seem worth fuzzing to be honest. But let's try to stick to the original intent. This all looks worth a read, so let's do that first and get a bit of an understanding of how we can programmatically do the stuff that explorer.exe does. In particular, how does it make the decision of what to do when I, in the command line type `explorer.exe <object>`. Let's see if that's in there somewhere.

OK so from continued reading it looks like the explorer is really just an implementation of the Shell in windows. This makes sense, I've always told people that the way to think of the shell is just like an explorer window but in text. Looking back this is obvious, so a lot of the functionality I'm looking for is actually `Shell` functionality here. Cool. Another interesting tidbit, reading Common Explorer Concepts is that stuff in Windows explorer is divided into a few different things. Specifically, files, folders, and virtual folders. Virtual folders are stuff you'd see in explorer that aren't really folders, a good example is printers, you click on a printer just as if it was a folder but it's not a folder. This brings us to the most basic unit within a folder, the SHITEMID, which simply identifies some object within an Explorer folder, it looks like this:

```
typedef struct _SHITEMID { 
    USHORT cb; 
    BYTE   abID[1]; 
} SHITEMID, * LPSHITEMID;
```

From MSDN: `The abID member is the object's identifier. The length of abID is not defined, and its value is determined by the folder that contains the object.` The details on how exactly a folder's objects are enumerated are here: https://docs.microsoft.com/en-us/windows/win32/shell/folder-info. As it turns out MSDN doesn't just call them "objects" by accident, they are implementations of that dirty bastard COM objects. Anyway, I can feel us getting closer and closer to the functionality we want (and if not some additional stuff that might be worth a fuzz) when we list objects' attributes:

```
If you have an object's fully qualified path or PIDL, SHGetFileInfo provides a simple way to retrieve information about an object that is sufficient for many purposes. SHGetFileInfo takes a fully qualified path or PIDL, and returns a variety of information about the object including:

    The object's display name
    The object's attributes
    Handles to the object's icons
    A handle to the system image list
    The path of the file containing the object's icon
```

Even not having found exactly what I want yet, already I'm interested by all of this functionality. From previous fuzzing and 0-day hunting I know that anything involving a structure that reports its own size is reasonably interesting. Why? Because oftentimes you'll find that internals of whatever will trust this value, meaning we could report a USHORT of 1 and have a name for it that is 10 bytes long, the hope would be that 1 byte is allocated and Windows attempts to shove 10 bytes into the buffer, creating overflow conditions. But that's a bit specific, let's keep going and see if we can fuzz some additional functionality along with that. I also still really want to get at that code that parses stuff and determines how explorer.exe will handle the various types of objects. 

Ah and there it is: https://docs.microsoft.com/en-us/windows/win32/shell/app-registration. As it turns out there's really not any magic happening in Explorer.exe other than looking shit up in the registry. If an executable is in the registry with something like the following:

```
HKEY_CLASSES_ROOT
   Applications
      mspaint.exe
         SupportedTypes
            .bmp
            .dib
            .rle
            .jpg
            .jpeg
            .jpe
            .jfif
            .gif
            .emf
            .wmf
            .tif
            .tiff
            .png
            .ico
```

anytime we go to open a .bmp file, mspaint is going to be used. Really, what's happening is this, say the user changes the default for .mp3 files to App2ID, the registry would undergo the following changes:

```
HKEY_CLASSES_ROOT
   .mp3
      (Default) = App1ProgID
```


```
HKEY_CLASSES_ROOT
   App1ProgID
      shell
         Verb1
```


```
HKEY_CLASSES_ROOT
   App2ProgID
      shell
         Verb2
```

And that pretty much takes care of it. There's no real magic happening there much to my dismay, and of course I should have known this going in - changing the default program that something is opened with is a fairly common operation in windows. The real vulnerabilities are just the friends we made along the way.

Just kidding. OK so it doesn't work how I thought it would, but is it still fuzzable? Fuck yeah it is, let's try something, and again I write this as stream of consciousness because I think the thought process is important - I have no idea if this is actually going to work. Why don't we take the function that reads the registry keys when I open up a file with explorer.exe, and start mutating the various registry keys. All I need is to write some functions, and take in file with registry information - these files will be mutated and the registry updated with weird, mutated information. Then explorer.exe will read it, and do its thing. At this point a couple of things are going to be helpful - let's validate that the stuff that MSDN is saying is true, and let's see if there's additional, undocumented attack surface. This shouldn't be difficult to do with procmon - another windows sysinternals tool that is pretty awesome. Let's also go ahead and continue with our coverage experiment with Binary Ninja and lighthouse. OK, let's do the procmon thing first.

Let's also introduce another simple, but extremely useful tool, called Search Everything (shortened to Everything often). It allows for very fast Windows search and can be found here: https://www.voidtools.com/. OK, I open up procmon and add a filter for explorer.exe:

![procmon registry stuff](/assets/img/procmon_explorer.PNG)

First thing I notice is holy shit that's a lot of registry interaction. This isn't really rare, but normally you don't have to scroll down a fuckton of times to just find something that isn't registry related. This is cool and interesting, seems like the documentation is on the right track, and I can use all of this information in my fuzzing session - perhaps I'll make a list of registry stuff that this is interacting with and fuzz it all. But maybe not, let's keep at it and see where we land. I should note, this is a stylistic thing as a hacker, I often just go into stuff completely blind and have no idea if it'll work or not. Some people like to be more deliberate and planned. That's fine, but I find that even when I fail, I end up learning a ton of stuff (like the internals of explorer). Going off on tangents is not a bad thing. Keep doing it and eventually you'll be familiar with a lot of weird Windows internal stuff that isn't even covered in any literature. OK so let's keep moving. The next thing I do is note that the explorer.exe being opened is at `C:\Windows\explorer.exe`. Great, now I know which of that 5 explorer.exe's is actually being used. To recap a bit here is my basic strategy right now: I want to fuzz explorer.exe by opening a few files and a few folders and seeing what is read in the registry. I don't want a window to pop up so I'm going to reverse engineer some functions that closely emulate what explorer.exe is doing, and then write a fuzzing harness using these functions. Now, I don't want to get my small test of explorer.exe confused with the always-open explorer.exe against a file and then a folder. Closing explorer.exe just makes it restart (side note: might be interesting to see if there's an exploitable race condition there at some point, we could check exactly HOW this restart happens by killing explorer.exe, potential replace a dll or executable and have it start up some code. I doubt this would be fruitful, but Windows does things in a funny way sometimes and it's always worth a look. We'll come back to this later if I remember). So I go ahead and copy explorer.exe to my `C:\` drive and call it explorer2.exe, that way I can filter procmon against that name instead of explorer.exe which is doing a ton of stuff by just having programs open. A lot of reverse engineering is all about **data reduction**, you get bombarded with shit with all of these tools, binary ninja gives you a massive amount of instructions, etc. Don't think that a good reverse engineer is able to understand absolutely all of it - a good reverse engineer is able to sparingly use these tools after exhausting all other simpler avenues, like what I did by simply looking up how explorer.exe works. Alex Ionescu's talk Reversing Without Reversing goes so far as to even mention that you could email Microsoft employees and kindly ask how something works. I'm not doing that here, but if it comes to it and I want to more deeply understand what is happening in some system internals it may be worth it to start making some connections at MS that may be able to help answer some questions. I'm not talking about social engineering here, I'm talking about legitimately asking someone there to help you understand the system. I have a few friends at MS, and perhaps someday I'll engage them and see if they can point me to someone willing to help. But anyway, let's keep going.

I not have a copy of explorer.exe named explorer2.exe, so I filter procmon to only give me info on images that are named explorer2.exe. OK, so blank slate! Let's try opening an exe file with explorer2.exe. Output of procmon when I do this:

![procmon explorere2.exe](/assets/img/explorer2.PNG)


Here you can see why Windows is an interesting attack surface. If you run something like strace on an ELF you see a decent amount of output, but it's reasonably readable. Windows on the other hand, does a fuckton of stuff that seems weirdly unnecessary. Half of it fails, some of it succeeds, and all of it can be analyzed for attack surface. I'm going to try to stay focused here, but one thing that is *always* worth doing, is opening up procmon, seeing which dlls a process is loading (sometimes it loads ones that don't even exist) and checking their permissions. One day I'll write a script that does this automatically, it shouldn't take long, I just haven't done it. DLL hijacking/replacement is a very common vulnerability, so any process that elevates its privileges should be checked to see if the dlls are world (or user) writable. If they are, you've got yourself a 0-day. Even these ridiculously simple vulns can be worth a few thousand dollars. But anyway, let's keep moving with our original intent.  

Scrolling down about halfway we start to see some familiar looking stuff. Rememebr I mentioned that explorer.exe really uses Shell functions to do its thing?

```
8:47:45.3273338 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Query Value
8:47:45.3273424 AM	explorer2.exe	22164	RegOpenKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Desired Access: Query Value
8:47:45.3273556 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: Name
8:47:45.3273631 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: HandleTags, HandleTags: 0x0
8:47:45.3273712 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Maximum Allowed
8:47:45.3273790 AM	explorer2.exe	22164	RegQueryValue	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder\Attributes	SUCCESS	Type: REG_DWORD, Length: 4, Data: 538443776
8:47:45.3273860 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: Name
8:47:45.3273936 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: HandleTags, HandleTags: 0x0
8:47:45.3274014 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Maximum Allowed
8:47:45.3274090 AM	explorer2.exe	22164	RegQueryValue	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder\CallForAttributes	NAME NOT FOUND	Length: 16
8:47:45.3274158 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: Name
8:47:45.3274235 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: HandleTags, HandleTags: 0x0
8:47:45.3274313 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Maximum Allowed
8:47:45.3274393 AM	explorer2.exe	22164	RegQueryValue	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder\RestrictedAttributes	NAME NOT FOUND	Length: 16
8:47:45.3274467 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: Name
8:47:45.3274547 AM	explorer2.exe	22164	RegQueryKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	Query: HandleTags, HandleTags: 0x0
8:47:45.3274627 AM	explorer2.exe	22164	RegOpenKey	HKCU\Software\Classes\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	NAME NOT FOUND	Desired Access: Maximum Allowed
8:47:45.3274705 AM	explorer2.exe	22164	RegQueryValue	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder\FolderValueFlags	SUCCESS	Type: REG_DWORD, Length: 4, Data: 0
8:47:45.3274782 AM	explorer2.exe	22164	RegCloseKey	HKCR\CLSID\{98F275B4-4FFF-11E0-89E2-7B86DFD72085}\ShellFolder	SUCCESS	
```

There we start to see a bunch of shell operations querying the registry for stuff. Let's check it for the SupportedTypes attribute that I've been talking about.

It's not worth a screenshot because what I found: nothing. This confused me for a minute as the documentation was pretty clear "Supported types", according to my understanding, is what TELLS explorer2.exe how to open a file right?? Well as it turns out this is only somewhat correct. It looks like instead of querying this itself, it delegates the SupportedTypes check to the actual process. So it seems the workflow is something like `I type explorer2.exe firefox.exe -> firefox.exe queries to see if the file I'm trying to open is in its SupportedTypes in the registry -> if yes, cool, report back and open, if no ask the user what to open it with`. In other words it just YOLOs it out and lets the process handle it for itself. Makes sense I suppose, opening a folder isn't going to have this because it's always going to be opened with explorer.exe natively.

So now I started thinking - perhaps there's a better thing to fuzz here. If I were to find, for example a buffer overflow from reading a registry key how would I weaponize this? One way it could help would be for persistence purposes, a simple registry edit could provide persistence - but that's already do-able without a memory corruption exploit. My original intent was to be able to send a file that corrupts explorer.exe when opened. It looks like this is going to be somewhat difficult as it delegates everything to the process. So perhaps my original intent is now stupid. What do we do now? I have a saying, "when you fail, admit it early and pivot" so let's pivot. We've learned about some neat potential attack surface with windows explorer. In particular it is well documented and seems quite easy to get and set attributes of an "object" in a folder. So there's two possible scenarios I could see - one would be that we create an object that doesn't make sense, is invalid in some way, and causes some kind of corruption and/or memory disclosure via its API. Seems like a good place to fuzz. The other would be to mutate objects within a folder and list them, perhaps if we make a fucked up kind of object something bad will happen. Maybe some other fucked up shit will happen, what do I know? Seems like a decent place to start. So let's start playing around with the explorer API for fun and... yeah you get it.

OK so here's a good starting point, MSDN has been kind enough to provide us with an example program!

```
#include <shlobj.h>
#include <shlwapi.h>
#include <iostream.h>
#include <objbase.h>

int main()
{
    IShellFolder *psfParent = NULL;
    LPITEMIDLIST pidlSystem = NULL;
    LPCITEMIDLIST pidlRelative = NULL;
    STRRET strDispName;
    TCHAR szDisplayName[MAX_PATH];
    HRESULT hr;

    hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);

    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);

    if(SUCCEEDED(hr))
    {
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));
        cout << "SHGDN_NORMAL - " <<szDisplayName << '\n';
    }

    psfParent->Release();
    CoTaskMemFree(pidlSystem);

    return 0;
}
```

Along with a description of the above program:

```
The application first uses SHGetFolderLocation to retrieve the System folder's PIDL. It then calls SHBindToParent, which returns a pointer to the parent folder's IShellFolder interface, and the System folder's PIDL relative to its parent. It then uses the parent folder's IShellFolder::GetDisplayNameOf method to retrieve the display name of the System folder. Because GetDisplayNameOf returns a STRRET structure, StrRetToBuf is used to convert the display name into a normal string. After displaying the display name, the interface pointers are released and the System PIDL freed. Note that you must not free the relative PIDL returned by SHBindToParent.
```

I like this description. Not only is it very clear what's going on in the code (meaning I can edit it to create a fuzzing harness), the various objects and structures look generally interesting. That conversion into a display name just *sounds* to me like could be a bit tricky. But who knows, maybe somewhere else in there there is a gotcha for Windows developers. Fuzzing can cause some crazy shit to happen, which is exactly what we want. Looking at the `SHGetFolderLocation` API:

```
SHSTDAPI SHGetFolderLocation(
  HWND             hwnd,
  int              csidl,
  HANDLE           hToken,
  DWORD            dwFlags,
  PIDLIST_ABSOLUTE *ppidl
);
```

Ha, ok, so one of these (`HANDLE htoken`) is a process access token. I'm not totally clear here on how the API protects itself, what if I set this to an Administrator's or System's access token using something like Mimikatz? Even metasploit's meterpreter has a get_token functionality. I could get arbitrary privileged reads for stuff that I'm not even supposed to see. Let's put a pin in that for now and come back to it so we can focus on getting a fuzzer up and running. Let's throw this code into VSCode and compile it, just for a quick sanity check. Just as a quick note that second argument to SHGetFolderLocation which is set to CSIDL_SYSTEM which means we're going to get the System folder's default display name with that code. Pretty cool, things are starting to take shape here, and I think I know how I'm going to fuzz this, but I won't spoil it just yet. Let's get that compilation done:

```
C:\Users\punkpc\Desktop\0daze
λ clang-cl.exe explorer_api_get_system_display_name.cpp ole32.lib shell32.lib shlwapi.lib
explorer_api_get_system_display_name.cpp(1,17): warning: using directive refers to implicitly-defined namespace 'std'
using namespace std;
                ^
1 warning generated.

C:\Users\punkpc\Desktop\0daze
λ ls
explorer_api_get_system_display_name.cpp  explorer_api_get_system_display_name.exe*  explorer_api_get_system_display_name.obj

C:\Users\punkpc\Desktop\0daze
λ .\explorer_api_get_system_display_name.exe
SHGDN_NORMAL - System32
```

This step took me longer than I'd like to admit. But there's a good reason for it! I'm going to give libfuzzer on Windows a try. I've used it on Linux before and it's ridiculously fast (I remember getting about 22k execs/sec on a decent laptop). It seems like this is a great option right now as everything is in memory. When using libfuzzer it's a good idea to have as much loaded into memory as possible (i.e. build a static executable). So let's get cracking on defining the functions we need to define and adding libfuzzer to this code.

First let's see what else we can do with this code, in particular we're looking for what the "interesting" variables are. So let's get cracking, edit some of the code, write some hacky shit, and then fuzz away. Actually, let's start by simply commenting our code to see what each line is doing:

```
using namespace std;
#include <shlobj.h>
#include <shlwapi.h>
#include <iostream>
#include <objbase.h>

int fuzzMeDrZaus()
{

    //This is the main "folder" interface. It is the representation of the folder in the form a COM object interface.
    IShellFolder *psfParent = NULL; 

    //Each file/folder/virtual object is represented by a series of SHITEMIDs (SH item IDs, not shite MIDs as I read it)
    //Think of SHITEMIDs as filename or folder name and the LPITEMIDLIST as a fully qualified path or rather a list of SHITEMID structures
    LPITEMIDLIST pidlSystem = NULL;
    LPCITEMIDLIST pidlRelative = NULL;

    //this is where we're gong to shove the display name when we call GetDisplayNameOf (a human readable representation of an LPCITEMIDLIST)
    //The STRRET type is specific to the IShellFolder interface and is meant to hold its string. It's really a struct but it's fucking
    //complicated to let's just pretend it's a string.
    STRRET strDispName;
 
    //This is going to hold the actual string in the above struct, so it's an array of chars
    TCHAR szDisplayName[MAX_PATH];

    //The fucking result, for error checking we're not going to do because fuck that noise (and it slows down the fuzzer)
    HRESULT hr;


    /*
    SHSTDAPI SHGetFolderLocation(
      HWND             hwnd,
      int              csidl,
      HANDLE           hToken,
      DWORD            dwFlags,
      PIDLIST_ABSOLUTE *ppidl
    );
    */
   //This is really just being used as a vessel for that second arg for which CSIDL_SYSTEM will be specified
   //A CSIDL == Constant Special Item ID List
   //This function returns the path of a folder but as an ITEMIDLIST. So this is saying gimme an ITEMIDLIST
   //that points to the special System folder (system32). Neat.
   //The last argument is where the actual ITEMIDLIST is going to be stored, i.e. what we give a fuck about
    hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);

    /*
    SHSTDAPI SHBindToParent(
      PCIDLIST_ABSOLUTE pidl,
      REFIID            riid,
      void              **ppv,
      PCUITEMID_CHILD   *ppidlLast
    );
    */
   //This function will return an interface pointer to the parent folder of pidlSystem (the ITEMIDLIST for system)
   //In other words it takes the first arg and shoves the parent of the ItemID into psfParent
   //pidRelative us the items PIDL relative to the parent folder. In other words this is saying
   //"give me the fucking parent of the thing I give you (System folder here). Give it to me as an IShellFolder item,
   //and while you're add it why don't you gimme the relative path to the parent folder too." Note we seem to be 
   //getting the System folder (i.e. "C:\Windows\System32"), grabbing the parent Windows\ and then getting the relative
   //path from there to System32 (i.e. "System32"). Seems kinda weird but OK.
    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);


    //Assuming all the above shit checks out...
    if(SUCCEEDED(hr))
    {
        //use the IShellInterface COM object interface GetDisplayNameOf for the relative path of the parent folder to the system folder, 
        //give me a normal fucking string, and shove it in strDispName, note this is going to shove System32 into *strDispName.
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);

        //This function literally only exists to get the display name and store it into the third arg, szDisplayName
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));

        //Output the display name
        cout << "SHGDN_NORMAL - " << szDisplayName << '\n';
    }


    //I..... release you.
    psfParent->Release();

    //Gimme my fucking memory back.
    CoTaskMemFree(pidlSystem);

    //Yay all is well. Or is it????? (it is)
    return 0;
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {



}
```

The above gives you an idea of how folders and shit work. To be honest it's pretty insane. There's really not a good visual representation of things and you're highly
dependent on the IShellFolder methods to do absolutely anything. This sucks for you coders out there, but it's pretty great for us hackers. This is needlessly complicated
and where things are needlessly complicated there are bugs when they're implemented. I don't know what chucklehead thought this was a good idea, but ok, let's roll with
it.

You can see at the bottom I'm getting ready to use LibFuzzer against this. I just need to decide which input I'm going to fuzz. I think I'm going to fuzz the GetDisplayNameOf
function at it seems to be one of the most appropriate ones as the global state is fairly well established at that point and I'm asking it to perform a weird and (for some ungodly reason) difficult transformationp into a human readable string. I'm going to just roll with this code with a slight modification, but a few of these APIs seem fuzzable. Let's just get something running and iterate over it afterward.

Anyway I'm getting tired, I thought I should probably document how long this stuff is taking me, as it has been a couple of days. So from now on I'll just do a little signing off message and mention the date, which is 12/8/2020 at 1:15 AM.

I was hoping to pick this back up yesterday, but business and a power outage at night (when I do most of my hacking) didn't let me, so starting up 12/10/2020! Let's keep going with this. First of all, at this point, before starting any fuzzing it's worth it to take a step back and think about it. Taking 24 hours to just work on some easy stuff (like website edits) gives you time to think about what you've done and what you are about to do. It's at that time that I realized that libFuzzer is a generational fuzzer, meaning it generates input. This isn't the worst thing in the world in this case, as the number of executions/sec is expected to be very very high, so just by blind luck I may get something that passes the validation code that I assume exists for ITEMIDLISTs that I'm about to fuzz. However, is there a way that I can turn generational fuzzing into some kind of mutational fuzzing? As it turns out I think there is. I haven't seen this technique used out there in the wild, so no idea if it'll work, but what if I take the normal input, use LLVMFuzzerTestOneInput's `Data` argument and perform some kind of operation on the valid data with the fuzz data `Data` and MAKE this is a mutational fuzzer. The natural question arises - what kind of operation would be useful? I'm thinking a few may be! First, appending and prepending to the data might be nice. Second, actually hold up. Like I mention in the title I like to leave in when and how I screw up. libFuzzer is NOT a generational fuzzer, it in fact is a mutational fuzzer. I realized my mistake when I forgot (it's been a while since I used libFuzzer tbh) that it takes in a corpus. So now we have another problem, let's generate a corpus.

Were this python I'd know exactly what to do, I'd pickle the file, store it in the corpus directory and set my fuzzer to run. A natural question comes - wtf is the equivalent of pickling but in C++ land? Hell if I know, so I pop open Google and figure out that pickling is also known as object serialization. So I need to serialize the variable I want to test, store it in a file, and pop it in the corpus directory. I also need to make sure I can deserialize it so that the file can be mutated, deserialized, and input. One concern here is that if I mutate the file, am I just testing the deserialization library for safety? So fuck it, yeah, I'm not doing that. So what now?

Well, I've managed to convince myself that libFuzzer's mutators aren't that great anyway and it's really not worth the hassle of interacting with the filesystem unnecessarily and that yes, I would really be testing the deserialization library itself (not necessarily a bad thing, just not my goal). So I'm bringing in some help from my good friend `libradamsa`. But radamsa is only part of the bigger whole of the fuzzing story. How am I gonna make sure I discover crashes? There's a few ways I can do this, but my favorite method is static binary rewriting. In other words, what I'm going to do is compile all my shit with the ASAN (address sanitizer) and MSAN (memory sanitizer) and run it with libradamsa mutating the input. You can't compile a binary with both, they are similar, so it will be two different fuzzing sessions running concurrently. The other thing that happened as I was writing this is that I looked up libFuzzer more (I googled *harder*) and read that yes it can be used as a generational fuzzer. That's probably not going to net me amazing results, but it's something I can get up and running fast while I get this other stuff going. I also realized that object serialization isn't really necessary, what I could do is simply store a path like `C:\Windows\System32` and turn it into an ITEMIDLIST using whatever function it is that turns paths into this weird tomfuckery that Microsoft has chosen to represent its folder/file/virtual object stuff. OK so let's get the easiest going first. libFuzzer generating some random ass inputs. I put on my robe and wizards hat (did I make that joke already? better get used to it I guess...). My code now looks like this:


```
#include <shlobj.h>
#include <shlwapi.h>
#include <iostream.h>
#include <objbase.h>

int fuzzMeDrZaus()
{
    IShellFolder *psfParent = NULL;
    LPITEMIDLIST pidlSystem = NULL;
    LPCITEMIDLIST pidlRelative = NULL;
    STRRET strDispName;
    TCHAR szDisplayName[MAX_PATH];
    HRESULT hr;

    hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);

    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);

    if(SUCCEEDED(hr))
    {
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));
        cout << "SHGDN_NORMAL - " <<szDisplayName << '\n';
    }

    psfParent->Release();
    CoTaskMemFree(pidlSystem);

    return 0;
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {



}

```

and i've turned it into this:

```
using namespace std;
#include <shlobj.h>
#include <shlwapi.h>
#include <iostream>
#include <objbase.h>

int fuzzMeDrZaus(const STRRET *strDispName_fuzzy, size_t size)
{

    //This is the main "folder" interface. It is the representation of the folder in the form a COM object interface.
    IShellFolder *psfParent = NULL; 

    //Each file/folder/virtual object is represented by a series of SHITEMIDs (SH item IDs, not shite MIDs as I read it)
    //Think of SHITEMIDs as filename or folder name and the LPITEMIDLIST as a fully qualified path or rather a list of 
    //SHITEMID structures
    LPITEMIDLIST pidlSystem = NULL;
    LPCITEMIDLIST pidlRelative = NULL;

    //this is where we're gong to shove the display name when we call GetDisplayNameOf (a human readable 
    //representation of an LPCITEMIDLIST)
    //The STRRET type is specific to the IShellFolder interface and is meant to hold its string. It's really a struct 
    //but it's fucking
    //complicated so let's just pretend it's a string.
    //STRRET strDispName;
 
    //This is going to hold the actual string in the above struct, so it's an array of chars
    TCHAR szDisplayName[MAX_PATH];

    //The fucking result, for error checking we're not going to do because fuck that noise (and it slows down the fuzzer)
    HRESULT hr;


    /*
    SHSTDAPI SHGetFolderLocation(
      HWND             hwnd,
      int              csidl,
      HANDLE           hToken,
      DWORD            dwFlags,
      PIDLIST_ABSOLUTE *ppidl
    );
    */
   //This is really just being used as a vessel for that second arg for which CSIDL_SYSTEM will be specified
   //A CSIDL == Constant Special Item ID List
   //This function returns the path of a folder but as an ITEMIDLIST. So this is saying gimme an ITEMIDLIST
   //that points to the special System folder (system32). Neat.
   //The last argument is where the actual ITEMIDLIST is going to be stored, i.e. what we give a fuck about
    hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);

    /*
    SHSTDAPI SHBindToParent(
      PCIDLIST_ABSOLUTE pidl,
      REFIID            riid,
      void              **ppv,
      PCUITEMID_CHILD   *ppidlLast
    );
    */
   //This function will return an interface pointer to the parent folder of pidlSystem (the ITEMIDLIST for system)
   //In other words it takes the first arg and shoves the parent of the ItemID into psfParent
   //pidRelative us the items PIDL relative to the parent folder. In other words this is saying
   //"give me the fucking parent of the thing I give you (System folder here). Give it to me as an IShellFolder item,
   //and while you're add it why don't you gimme the relative path to the parent folder too." Note we seem to be 
   //getting the System folder (i.e. "C:\Windows\System32"), grabbing the parent Windows\ and then getting the relative
   //path from there to System32 (i.e. "System32"). Seems kinda weird but OK.
    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);


    //Assuming all the above shit checks out...
    if(SUCCEEDED(hr))
    {
        //use the IShellInterface COM object interface GetDisplayNameOf for the relative path of the parent folder to the system folder, 
        //give me a normal fucking string, and shove it in strDispName, note this is going to shove System32 into *strDispName.
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName_fuzzy);

        //This function literally only exists to get the display name and store it into the third arg, szDisplayName
        hr = StrRetToBuf(&strDispName_fuzzy, pidlSystem, szDisplayName, sizeof(szDisplayName));

        //Output the display name
        cout << "SHGDN_NORMAL - " <<szDisplayName << '\n';
    }


    //I..... release you.
    psfParent->Release();

    //Gimme my fucking memory back.
    CoTaskMemFree(pidlSystem);

    //Yay all is well. Or is it????? (it is)
    return 0;
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  fuzzMeDrZaus(Data, size);
}
```

And I think I'm ready to yeet (right kids??) this on out. I put on my robe a.... ok I'll stop. Let's get this fuzzer running:

Nope. Just kidding, this didn't work. Why not? The STRRET structure (remember I sort of glazed over it before as basically just a string) is something that cannot be cast to. This whole time I was treating it as if it were a simple `type` but it is in fact a `struct` with a `union` in it. So I had to get a little deeper into it. This took a lot longer than I'd like to admit (though I will admit it took ~2hrs), but I think overall this will be much better than really dumb fuzzing. Check out the code below, which this time actually compiles:

```
#include <shlobj.h>
#include <shlwapi.h>
#include <iostream>
#include <objbase.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

int fuzzMeDrZaus(const uint8_t *Data, size_t size)
{

    //This is the main "folder" interface. It is the representation of the folder in the form a COM object interface.
    IShellFolder *psfParent = NULL; 

    //Each file/folder/virtual object is represented by a series of SHITEMIDs (SH item IDs, not shite MIDs as I read it)
    //Think of SHITEMIDs as filename or folder name and the LPITEMIDLIST as a fully qualified path or rather a list of 
    //SHITEMID structures
    LPITEMIDLIST pidlSystem;
    pidlSystem->mkid.cb = (USHORT) size;
    std::memcpy(pidlSystem->mkid.abID, (const void *) Data, size);

    LPCITEMIDLIST pidlRelative = NULL;

    //this is where we're gong to shove the display name when we call GetDisplayNameOf (a human readable 
    //representation of an LPCITEMIDLIST)
    //The STRRET type is specific to the IShellFolder interface and is meant to hold its string. It's really a struct 
    //but it's fucking
    //complicated so let's just pretend it's a string.
    STRRET strDispName;
 
    //This is going to hold the actual string in the above struct, so it's an array of chars
    TCHAR szDisplayName[MAX_PATH];

    //The fucking result, for error checking we're not going to do because fuck that noise (and it slows down the fuzzer)
    HRESULT hr;

    /*
    SHSTDAPI SHGetFolderLocation(
      HWND             hwnd,
      int              csidl,
      HANDLE           hToken,
      DWORD            dwFlags,
      PIDLIST_ABSOLUTE *ppidl
    );
    */
   //This is really just being used as a vessel for that second arg for which CSIDL_SYSTEM will be specified
   //A CSIDL == Constant Special Item ID List
   //This function returns the path of a folder but as an ITEMIDLIST. So this is saying gimme an ITEMIDLIST
   //that points to the special System folder (system32). Neat.
   //The last argument is where the actual ITEMIDLIST is going to be stored, i.e. what we give a fuck about

    //Replace this folder location with fuzz data
    //hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);



    /*
    SHSTDAPI SHBindToParent(
      PCIDLIST_ABSOLUTE pidl,
      REFIID            riid,
      void              **ppv,
      PCUITEMID_CHILD   *ppidlLast
    );
    */
   //This function will return an interface pointer to the parent folder of pidlSystem (the ITEMIDLIST for system)
   //In other words it takes the first arg and shoves the parent of the ItemID into psfParent
   //pidRelative us the items PIDL relative to the parent folder. In other words this is saying
   //"give me the fucking parent of the thing I give you (System folder here). Give it to me as an IShellFolder item,
   //and while you're add it why don't you gimme the relative path to the parent folder too." Note we seem to be 
   //getting the System folder (i.e. "C:\Windows\System32"), grabbing the parent Windows\ and then getting the relative
   //path from there to System32 (i.e. "System32"). Seems kinda weird but OK.
    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);


    //Assuming all the above shit checks out...
    if(SUCCEEDED(hr))
    {
        //use the IShellInterface COM object interface GetDisplayNameOf for the relative path of the parent folder to the system folder, 
        //give me a normal fucking string, and shove it in strDispName, note this is going to shove System32 into *strDispName.
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);

        //This function literally only exists to get the display name and store it into the third arg, szDisplayName
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));

        //Output the display name
        std::cout << "SHGDN_NORMAL - " <<szDisplayName << '\n';
    }


    //I..... release you.
    psfParent->Release();

    //Gimme my fucking memory back.
    CoTaskMemFree(pidlSystem);

    //Yay all is well. Or is it????? (it is)
    return 0;
};


extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {

  fuzzMeDrZaus(Data, Size);
  return 0;
}
```

OK now we're doing something that I feel may actually have some value. This is the big change: The PIDL that contained System called pidlSystem has now been explicitly defined as opposed to retrieved. 2 functions that depend on it are called afterward. These two functions, along with any internal processing that happens when you define a PIDL will now be fuzzed. Let's get yeeting. First we want to execute the program and make sure it doesn't crash under normal conditions:

```
C:\Users\punkpc\Desktop\0daze
λ .\explorer_api_get_system_display_name.exe
INFO: Seed: 705234228
INFO: Loaded 1 modules   (293 inline 8-bit counters): 293 [00007FF79313B148, 00007FF79313B26D),
INFO: Loaded 1 PC tables (293 PCs): 293 [00007FF7930E6798,00007FF7930E79E8),
INFO: -max_len is not provided; libFuzzer will not generate inputs larger than 4096 bytes
SHGDN_NORMAL - Desktop
INFO: A corpus is not provided, starting from an empty corpus
=================================================================
==9528==ERROR: AddressSanitizer: access-violation on unknown address 0x000000000000 (pc 0x7ff792f816e1 bp 0x00000014f490 sp 0x00000014f0e0 T0)
==9528==The signal is caused by a READ memory access.
==9528==Hint: address points to the zero page.
    #0 0x7ff792f816e0 in fuzzMeDrZaus(unsigned char const *,unsigned __int64) C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:91
    #1 0x7ff792f83d6a in LLVMFuzzerTestOneInput C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:103
    #2 0x7ff792ff5fca in fuzzer::Fuzzer::ExecuteCallback(unsigned char const *,unsigned __int64) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:559
    #3 0x7ff792ff5446 in fuzzer::Fuzzer::RunOne(unsigned char const *,unsigned __int64,bool,struct fuzzer::InputInfo *,bool *) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:471
    #4 0x7ff792ff7d39 in fuzzer::Fuzzer::ReadAndExecuteSeedCorpora(class std::vector<struct fuzzer::SizedFile,class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile> > &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:754
    #5 0x7ff792ff7f0a in fuzzer::Fuzzer::Loop(class std::vector<struct fuzzer::SizedFile,class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile> > &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:800
    #6 0x7ff79300e920 in fuzzer::FuzzerDriver(int *,char * * *,int (*)(unsigned char const *,unsigned __int64)) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerDriver.cpp:847
    #7 0x7ff792fd12d2 in main C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerMain.cpp:20
    #8 0x7ff7930166ef in __scrt_common_main_seh d:\agent\_work\63\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288
    #9 0x7ff9ba787033  (C:\Windows\System32\KERNEL32.DLL+0x180017033)
    #10 0x7ff9bbe1d0d0  (C:\Windows\SYSTEM32\ntdll.dll+0x18004d0d0)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: access-violation C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:91 in fuzzMeDrZaus(unsigned char const *,unsigned __int64)
==9528==ABORTING
MS: 0 ; base unit: 0000000000000000000000000000000000000000
0xa,
\x0a
artifact_prefix='./'; Test unit written to ./crash-adc83b19e793491b1c6ea0fd8b46cd9f32e592fc
Base64: Cg==
```

LOL. So actually, I did a few tests, and if the size parameter is not correct, this causes the above exception. This is a pretty clear example of a Null Pointer Dereference (I could probably get a CVE out of it but idgaf, it'd be a super low issue and it doesn't really matter, it's just bad, or no, data validation handling). So let's try to get something that works better. Looking at the traceback it seems like this is the line that crashes with the Null Pointer Deref: `psfParent->Release();` so let's comment that out. OK now I'm getting some meat, and the shark smells blood. Now I am getting some really funky outputs like these:

```
SHGDN_NORMAL -
        NEW_FUNC[1/25]: 0x7ff67e0e1b40 in std::operator<<<struct std::char_traits<char> >(class std::basic_ostream<char,struct std::char_traits<char> > &,char) C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\VC\Tools\MSVC\14.28.29333\include\ostream:779
        NEW_FUNC[2/25]: 0x7ff67e0e2e30 in std::operator<<<struct std::char_traits<char> >(class std::basic_ostream<char,struct std::char_traits<char> > &,char const *) C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\VC\Tools\MSVC\14.28.29333\include\ostream:734
#91     NEW    cov: 53 ft: 53 corp: 2/5b lim: 4 exec/s: 0 rss: 78Mb L: 4/4 MS: 4 ChangeBit-CrossOver-ChangeBit-ChangeBinInt-
==================2==================
```

and 

```
SHGDN_NORMAL -
#114    REDUCE cov: 53 ft: 53 corp: 2/4b lim: 4 exec/s: 0 rss: 78Mb L: 3/3 MS: 3 ShuffleBytes-CopyPart-EraseBytes-
```

Looks like I found a memory disclosure bug in the API. Let's put a pin in that and come back to it. Now the program has crashed (after about 1 second of fuzzing):

```

=================================================================
==22524==ERROR: AddressSanitizer: access-violation on unknown address 0x000000622bf7 (pc 0x7ff9bb3b9e18 bp 0x000000000000 sp 0x00000014f050 T0)
==22524==The signal is caused by a READ memory access.
    #0 0x7ff9bb3b9e17  (C:\Windows\System32\SHELL32.dll+0x1800b9e17)
    #1 0x7ff9bb3b9d25  (C:\Windows\System32\SHELL32.dll+0x1800b9d25)
    #2 0x7ff67e0e15e9 in fuzzMeDrZaus(unsigned char const *,unsigned __int64) C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:75
    #3 0x7ff67e0e403a in LLVMFuzzerTestOneInput C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:106
    #4 0x7ff67e15635a in fuzzer::Fuzzer::ExecuteCallback(unsigned char const *,unsigned __int64) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:559
    #5 0x7ff67e1557d6 in fuzzer::Fuzzer::RunOne(unsigned char const *,unsigned __int64,bool,struct fuzzer::InputInfo *,bool *) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:471
    #6 0x7ff67e157a61 in fuzzer::Fuzzer::MutateAndTestOne(void) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:702
    #7 0x7ff67e158665 in fuzzer::Fuzzer::Loop(class std::vector<struct fuzzer::SizedFile,class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile> > &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:838
    #8 0x7ff67e16ecb0 in fuzzer::FuzzerDriver(int *,char * * *,int (*)(unsigned char const *,unsigned __int64)) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerDriver.cpp:847
    #9 0x7ff67e131662 in main C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerMain.cpp:20
    #10 0x7ff67e176a7f in __scrt_common_main_seh d:\agent\_work\63\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288
    #11 0x7ff9ba787033  (C:\Windows\System32\KERNEL32.DLL+0x180017033)
    #12 0x7ff9bbe1d0d0  (C:\Windows\SYSTEM32\ntdll.dll+0x18004d0d0)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: access-violation (C:\Windows\System32\SHELL32.dll+0x1800b9e17)
==22524==ABORTING
MS: 5 CopyPart-EraseBytes-InsertByte-CopyPart-ChangeBit-; base unit: adc83b19e793491b1c6ea0fd8b46cd9f32e592fc
0xa,0x1b,
\x0a\x1b
artifact_prefix='./'; Test unit written to ./crash-9f6a9759b162b9e20b94cc65271484408867811a
Base64: Chs=
```

iiiiiinteresting. Looks like it's tried to read the memory address at 0x000000622bf7. Something has fucked up the flow of the program, and it doesn't look like another Null Pointer Dereference. Let's look at the data that caused that:

`0x0A 0x1B`

Hm, ok. Whatever. And kicking off the fuzzer yet again we get a classic:

```

=================================================================
==21160==ERROR: AddressSanitizer: access-violation on unknown address 0x000000000024 (pc 0x7ff9bbde57db bp 0x000000000000 sp 0x00000014ec20 T0)
==21160==The signal is caused by a READ memory access.
==21160==Hint: address points to the zero page.
    #0 0x7ff9bbde57da  (C:\Windows\SYSTEM32\ntdll.dll+0x1800157da)
    #1 0x7ff9b772e1be  (C:\Windows\SYSTEM32\windows.storage.dll+0x1800be1be)
    #2 0x7ff9b772dd66  (C:\Windows\SYSTEM32\windows.storage.dll+0x1800bdd66)
    #3 0x7ff9b772dc7d  (C:\Windows\SYSTEM32\windows.storage.dll+0x1800bdc7d)
    #4 0x7ff9b77fe6b3  (C:\Windows\SYSTEM32\windows.storage.dll+0x18018e6b3)
    #5 0x7ff67e0e1326 in fuzzMeDrZaus(unsigned char const *,unsigned __int64) C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:51
    #6 0x7ff67e0e403a in LLVMFuzzerTestOneInput C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:106
    #7 0x7ff67e15635a in fuzzer::Fuzzer::ExecuteCallback(unsigned char const *,unsigned __int64) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:559
    #8 0x7ff67e1557d6 in fuzzer::Fuzzer::RunOne(unsigned char const *,unsigned __int64,bool,struct fuzzer::InputInfo *,bool *) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:471
    #9 0x7ff67e157a61 in fuzzer::Fuzzer::MutateAndTestOne(void) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:702
    #10 0x7ff67e158665 in fuzzer::Fuzzer::Loop(class std::vector<struct fuzzer::SizedFile,class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile> > &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:838
    #11 0x7ff67e16ecb0 in fuzzer::FuzzerDriver(int *,char * * *,int (*)(unsigned char const *,unsigned __int64)) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerDriver.cpp:847
    #12 0x7ff67e131662 in main C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerMain.cpp:20
    #13 0x7ff67e176a7f in __scrt_common_main_seh d:\agent\_work\63\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288
    #14 0x7ff9ba787033  (C:\Windows\System32\KERNEL32.DLL+0x180017033)
    #15 0x7ff9bbe1d0d0  (C:\Windows\SYSTEM32\ntdll.dll+0x18004d0d0)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: access-violation (C:\Windows\SYSTEM32\ntdll.dll+0x1800157da)
==21160==ABORTING
MS: 3 ShuffleBytes-InsertByte-InsertRepeatedBytes-; base unit: 5ba93c9db0cff93f52b521d7420e43f6eda2784f
0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xb8,0x0,
\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xb8\x00
artifact_prefix='./'; Test unit written to ./crash-dab1d57bcb778c55496eb3b65d4fcc74be8f871e
Base64: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAuAA=
```

Seriously? A bunch of A's crashes the API. I could've just guessed that! Let's see if we can get libFuzzer to continue running even after ASAN has found a crash. OK so I could probably have guessed most of these crashes, but let's keep running... ha, every time I run it it goes for a few seconds (one time it almost made it to 10!) and then crashes with something like:

```
==20556==ERROR: AddressSanitizer: access-violation on unknown address 0x000000000024 (pc 0x7ff9bbde57db bp 0x000000000000 sp 0x00000014ec20 T0)
==20556==The signal is caused by a READ memory access.
==20556==Hint: address points to the zero page.
    #0 0x7ff9bbde57da  (C:\Windows\SYSTEM32\ntdll.dll+0x1800157da)
    #1 0x7ff9b772e1be  (C:\Windows\SYSTEM32\windows.storage.dll+0x1800be1be)
    #2 0x7ff9b772dd66  (C:\Windows\SYSTEM32\windows.storage.dll+0x1800bdd66)
    #3 0x7ff9b772dc7d  (C:\Windows\SYSTEM32\windows.storage.dll+0x1800bdc7d)
    #4 0x7ff9b77fe6b3  (C:\Windows\SYSTEM32\windows.storage.dll+0x18018e6b3)
    #5 0x7ff695dc132e in fuzzMeDrZaus(unsigned char const *,unsigned __int64) C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:51
    #6 0x7ff695dc41ea in LLVMFuzzerTestOneInput C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:106
    #7 0x7ff695e367da in fuzzer::Fuzzer::ExecuteCallback(unsigned char const *,unsigned __int64) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:559
    #8 0x7ff695e35c56 in fuzzer::Fuzzer::RunOne(unsigned char const *,unsigned __int64,bool,struct fuzzer::InputInfo *,bool *) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:471
    #9 0x7ff695e37ee1 in fuzzer::Fuzzer::MutateAndTestOne(void) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:702
    #10 0x7ff695e38ae5 in fuzzer::Fuzzer::Loop(class std::vector<struct fuzzer::SizedFile,class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile> > &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:838
    #11 0x7ff695e4f130 in fuzzer::FuzzerDriver(int *,char * * *,int (*)(unsigned char const *,unsigned __int64)) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerDriver.cpp:847
    #12 0x7ff695e11ae2 in main C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerMain.cpp:20
    #13 0x7ff695e56eff in __scrt_common_main_seh d:\agent\_work\63\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288
    #14 0x7ff9ba787033  (C:\Windows\System32\KERNEL32.DLL+0x180017033)
    #15 0x7ff9bbe1d0d0  (C:\Windows\SYSTEM32\ntdll.dll+0x18004d0d0)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: access-violation (C:\Windows\SYSTEM32\ntdll.dll+0x1800157da)
==20556==ABORTING
MS: 3 CrossOver-EraseBytes-ChangeBinInt-; base unit: 5ba93c9db0cff93f52b521d7420e43f6eda2784f
0x0,0x0,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x2d,0x0,0x2d,0x2d,0x2d,0x2d,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xcc,0x0,0xa,0x0,0x0,0xff,0x0,0x0,0xff,0x0,0xf8,0xff,0xa,0x0,0xff,0xa,0xa,0xff,0x0,0xa,0x0,0xa,0xff,0xff,0x0,0x0,0xff,0xff,0x0,0xff,0xff,0xff,0xff,0xa,0x86,0xa,0x0,0x86,0x0,0x0,0xa,0xa,0xa,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0xa,0xa,0xa,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xb,0x0,0xa,0x0,0xa,0xa,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,
\x00\x00------------------------\x00----\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xcc\x00\x0a\x00\x00\xff\x00\x00\xff\x00\xf8\xff\x0a\x00\xff\x0a\x0a\xff\x00\x0a\x00\x0a\xff\xff\x00\x00\xff\xff\x00\xff\xff\xff\xff\x0a\x86\x0a\x00\x86\x00\x00\x0a\x0a\x0a\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x0a\x0a\x0a\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0b\x00\x0a\x00\x0a\x0a\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x0a\x00\x00\x00\x00\x00\x00\x0a\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00
artifact_prefix='./'; Test unit written to ./crash-18f71fc38fc4779055d6d5ae0c08ace3c5490eba
Base64: AAAtLS0tLS0tLS0tLS0tLS0tLS0tLS0tLS0ALS0tLQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAADMAAoAAP8AAP8A+P8KAP8KCv8ACgAK//8AAP//AP////8KhgoAhgAACgoKAAAKAAAAAAAAAAAACgoKCgoAAAAAAAAAAAAAAAAAAAALAAoACgoKAAAAAAAAAAAACgoAAAAAAAAAAAoAAAAAAAAAAAoKAAAAAAAACgAACgAAAAAAAAoAAAAAAAAAAAA=
```

Some of these crashes are "close enough" to 0x0 that I'd call them likely to be Null Page dereferences. What's interesting is that even after commenting out that "free memory" code I am still getting those dereferences. So that's at the very least 2 Null pointer derefs, 1 potential overflow/RIP corruption, and at least one but most likely a couple of memory disclosure vulns. That's pretty cool, and it's nice to see I fucking finally did something right. Let's keep going with this though, and later we'll analyze the crashes. Here's one more crash to drive the point home:

```
==17980==ERROR: AddressSanitizer: access-violation on unknown address 0x000000000024 (pc 0x7ff9bbde57db bp 0x000000000000 sp 0x00000014ec20 T0)
==17980==The signal is caused by a READ memory access.
==17980==Hint: address points to the zero page.
    #0 0x7ff9bbde57da  (C:\Windows\SYSTEM32\ntdll.dll+0x1800157da)
    #1 0x7ff9b772e1be  (C:\Windows\SYSTEM32\windows.storage.dll+0x1800be1be)
    #2 0x7ff9b772dd66  (C:\Windows\SYSTEM32\windows.storage.dll+0x1800bdd66)
    #3 0x7ff9b772dc7d  (C:\Windows\SYSTEM32\windows.storage.dll+0x1800bdc7d)
    #4 0x7ff9b77fe6b3  (C:\Windows\SYSTEM32\windows.storage.dll+0x18018e6b3)
    #5 0x7ff6ca7a132e in fuzzMeDrZaus(unsigned char const *,unsigned __int64) C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:51
    #6 0x7ff6ca7a41ea in LLVMFuzzerTestOneInput C:\Users\punkpc\Desktop\0daze\explorer_api_get_system_display_name.cpp:106
    #7 0x7ff6ca8167da in fuzzer::Fuzzer::ExecuteCallback(unsigned char const *,unsigned __int64) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:559
    #8 0x7ff6ca815c56 in fuzzer::Fuzzer::RunOne(unsigned char const *,unsigned __int64,bool,struct fuzzer::InputInfo *,bool *) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:471
    #9 0x7ff6ca817ee1 in fuzzer::Fuzzer::MutateAndTestOne(void) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:702
    #10 0x7ff6ca818ae5 in fuzzer::Fuzzer::Loop(class std::vector<struct fuzzer::SizedFile,class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile> > &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:838
    #11 0x7ff6ca82f130 in fuzzer::FuzzerDriver(int *,char * * *,int (*)(unsigned char const *,unsigned __int64)) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerDriver.cpp:847
    #12 0x7ff6ca7f1ae2 in main C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerMain.cpp:20
    #13 0x7ff6ca836eff in __scrt_common_main_seh d:\agent\_work\63\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288
    #14 0x7ff9ba787033  (C:\Windows\System32\KERNEL32.DLL+0x180017033)
    #15 0x7ff9bbe1d0d0  (C:\Windows\SYSTEM32\ntdll.dll+0x18004d0d0)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: access-violation (C:\Windows\SYSTEM32\ntdll.dll+0x1800157da)
==17980==ABORTING
MS: 4 InsertRepeatedBytes-CrossOver-ChangeBinInt-ShuffleBytes-; base unit: adc83b19e793491b1c6ea0fd8b46cd9f32e592fc
0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0x47,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0x0,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0xff,0x0,0x0,0xa,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0x0,0xa,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0xa,0x0,0xee,0xff,0xff,0xff,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xa,0x0,0x0,0x0,0x0,0xa,0xa,0x0,0x0,0x0,0xa,0x0,0xa,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,
\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xffG\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xff\x00\x00\x0a\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x00\x00\x00\x0a\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x00\x00\x00\x00\x00\x0a\x00\x00\x0a\x00\x00\x00\x00\x0a\x00\xee\xff\xff\xff\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x0a\x00\x00\x00\x00\x0a\x0a\x00\x00\x00\x0a\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x00
artifact_prefix='./'; Test unit written to ./crash-dd3525f54a3b23e1776114158cec8d11bd19e6c6
Base64: /////////////////////0f/////////////////////////////////////////////////////////////////////////////////////////AP///////////wAACgAKAAAAAAAAAAAAAAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoAAAAAAAAAAAAAAAAAAAAAAAoAAAAKAAAKAAAAAAAAAAAAAAAAAAAAAAAACgAAAAAACgAACgAAAAAKAO7///8AAAAAAAAAAAAAAAAAAAAKAAAAAAoKAAAACgAKAAAAAAAAAAA=
```

So we can deduce pretty clearly that long input causes a null page deref, and some inputs appear to corrupt RIP. Let's keep rolling with this theme. Annoyingly, the option to keep my fuzzer running even after ASAN crashes is not working on Windows. If anyone knows how to do this tweet at me. For now I'm just going to use a simple powershell script that restarts a process after it crashes. Why I do dis you might ask? I want to collect plenty of examples of crashes, triage them for ones that are the same or similar (like null pointer derefs) and analyze them later. Then the real trick comes - see if we can weaponize any of these. Currently it's crashing whenever a call to `SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);`. Here's the script I ended up using to kick it off:

```
while($true) {
  If (!(Get-Process -Name explorer_api_get_system_display_name -ErrorAction SilentlyContinue))
    {
      ./explorer_api_get_system_display_name.exe *>&1 > fuzzing.txt
    }
}
```

And I ran it with:

```PS C:\Users\punkpc\Desktop\0daze> powershell.exe -windowstyle hidden -ExecutionPolicy Bypass .\run_forever.ps1```

So everything from the fuzzing process gets redirected to fuzzing.txt.

OK so that's kicked off and redirected to a file. I'm going to have to brush up on my command-line fu to crunch some of that data (actually I'll just use Python). So with that running in the background let's see what else we can do. Well, all of the crashes seen have been for the function `SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);` let's see what happens when this is normal, but we then fuck up the data. Here's the new code:


```
#include <shlobj.h>
#include <shlwapi.h>
#include <iostream>
#include <objbase.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

int fuzzMeDrZaus(const uint8_t *Data, size_t size)
{

    //This is the main "folder" interface. It is the representation of the folder in the form a COM object interface.
    IShellFolder *psfParent = NULL; 

    //Each file/folder/virtual object is represented by a series of SHITEMIDs (SH item IDs, not shite MIDs as I read it)
    //Think of SHITEMIDs as filename or folder name and the LPITEMIDLIST as a fully qualified path or rather a list of 
    //SHITEMID structures
    LPITEMIDLIST pidlSystem;

    LPCITEMIDLIST pidlRelative = NULL;

    //this is where we're gong to shove the display name when we call GetDisplayNameOf (a human readable 
    //representation of an LPCITEMIDLIST)
    //The STRRET type is specific to the IShellFolder interface and is meant to hold its string. It's really a struct 
    //but it's fucking
    //complicated so let's just pretend it's a string.
    STRRET strDispName;
 
    //This is going to hold the actual string in the above struct, so it's an array of chars
    TCHAR szDisplayName[MAX_PATH];

    //The fucking result, for error checking we're not going to do because fuck that noise (and it slows down the fuzzer)
    HRESULT hr;

    /*
    SHSTDAPI SHGetFolderLocation(
      HWND             hwnd,
      int              csidl,
      HANDLE           hToken,
      DWORD            dwFlags,
      PIDLIST_ABSOLUTE *ppidl
    );
    */
   //This is really just being used as a vessel for that second arg for which CSIDL_SYSTEM will be specified
   //A CSIDL == Constant Special Item ID List
   //This function returns the path of a folder but as an ITEMIDLIST. So this is saying gimme an ITEMIDLIST
   //that points to the special System folder (system32). Neat.
   //The last argument is where the actual ITEMIDLIST is going to be stored, i.e. what we give a fuck about

    //Replace this folder location with fuzz data
    hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);

    /*
    SHSTDAPI SHBindToParent(
      PCIDLIST_ABSOLUTE pidl,
      REFIID            riid,
      void              **ppv,
      PCUITEMID_CHILD   *ppidlLast
    );
    */
   //This function will return an interface pointer to the parent folder of pidlSystem (the ITEMIDLIST for system)
   //In other words it takes the first arg and shoves the parent of the ItemID into psfParent
   //pidRelative us the items PIDL relative to the parent folder. In other words this is saying
   //"give me the fucking parent of the thing I give you (System folder here). Give it to me as an IShellFolder item,
   //and while you're add it why don't you gimme the relative path to the parent folder too." Note we seem to be 
   //getting the System folder (i.e. "C:\Windows\System32"), grabbing the parent Windows\ and then getting the relative
   //path from there to System32 (i.e. "System32"). Seems kinda weird but OK.
    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);


    pidlSystem->mkid.cb = (USHORT) size;
    std::memcpy(pidlSystem->mkid.abID, (const void *) Data, size);

    printf("==================%zu==================\n\n", pidlSystem->mkid.cb);
    printf("----------------------%02x---------------------\n\n", pidlSystem->mkid.abID);


    //Assuming all the above shit checks out...
    if(SUCCEEDED(hr))
    {
        //use the IShellInterface COM object interface GetDisplayNameOf for the relative path of the parent folder to the system folder, 
        //give me a normal fucking string, and shove it in strDispName, note this is going to shove System32 into *strDispName.
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);

        //This function literally only exists to get the display name and store it into the third arg, szDisplayName
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));

        //Output the display name
        std::cout << "SHGDN_NORMAL - " << szDisplayName << '\n';
    }


    //I..... release you.
    psfParent->Release();

    //Gimme my fucking memory back.
    CoTaskMemFree(pidlSystem);

    //Yay all is well. Or is it????? (it is)
    return 0;
};


extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {

  fuzzMeDrZaus(Data, Size);
  return 0;
}
```

And another one bytes the dust (see what I did there? I'm so fucking clever). So compiling this and running using the same script, OK, it ran for a little bit longer and THEN crashed. Instead of the castle just sinking into the swamp, it burned down, fell over, and then sank into the swamp. Let's take a look at some of those crashes. What do you notice in the below image?

![crashes](/assets/img/crashes.PNG)


What popped out to me is that there are small files and there are bigger files. The small files go down to just 1 byte! So most likely there's a character, or a few characters, that crash this API (note: I don't know which function is crashing yet). As it turns out every small file I opened has a 0x0A character in it, which is a line feed. So that brings me to the conclusion: line feeds crash this API. Since this is the explorer.exe API, shouldn't creating a file with a line feed crash it? Let's find out! Well, try as I might, using python and trying to copy and paste the char into a filename both fail. Line feeds are simply invalid in Windows filenames. So it makes sense that their code can't handle it. BUT, null pointer dereferences can lead to unexpected behavior and they are never good. They are squarely 0-days whether we like it or not and they should be planned for (by for example stripping all LF chars, throwing an exception, etc.) and not allowed to fail ungracefully. Signing off tonight with a few more 0-days under my belt and a ton of analysis to do tomorrow. Later peeps, it's 2:38 AM here on 12/10/2020. 

Fuck I just lost a few pages of work. long story short, some stuff that has made my life easier:

- Search Everything as shown in screeny
- cmder as shown in screeny
- adding vcvars64.bat to my path as seen in screeny and running it (this is what "open dev command prompt" does when you do that)

The above allowed me to create a friendly dev environment that I hadn't really been able to get down in Windows. Anyway, here's the screeny

![easylife.PNG]. OK and now we're back to hunting. So far we've been able to make pretty good work of the Shell32 API, not super surprising - and this is where target selection becomes important - the windows explorer.exe has been around since the dawn of fucking civilization. It is old. The API makes little to no sense. It should be taken out back behind the barn and shot out of mercy. Yet of course it's still functional, all the features WORK right?? This is why people used to criticize Microsoft, none of this was built with security in mind, and adding security after the fact is only marginally effective. IMHO Windows 10 is still highly attackable, there's so many driver-based LPEs that last I checked, Zerodium wasn't even buying them. Think about that for a minute, the most popular place to sell your sploits doesn't want your sploit because there's way too fucking many of them. And this isn't analysis that can be done by just anyone, you have to somewhat know your OS shit to do this. That's not meant as discouragement to any budding 0-day hunters, I believe everyone can learn this if they put the work in, I'm just saying it does require a bit of work and knowledge. Anyway, back to out code eh? The last function to fuzz is `strRetToBuf` and, just... oh dear god, I haven't even done anything yet and I already know it's going to be a fucking disaster. It's a function that's specifically meant to shove an STRRET object into a memory buffer. I almost feel bad analyzing it because if there isn't a 0-day in there I'll eat my fucking hat.

OK let's get this overwith, apologize to the function later, and buy it dinner later (we're fucking it, it's the least we can do) for its trouble. Let's try a couple of things, easiest first (this is almost like a mantra to me - always go for the easiest thing you can do first) - meaning I'm just going to reuse the exact same code but move it a few lines down. In particular, I'm going to let pidlSystem act normally until it gets to the latter functions in the code. That way we'll see if we can get crashes 

Let's take a look at the code I'm gonna try:

```
#include <shlobj.h>
#include <shlwapi.h>
#include <iostream>
#include <objbase.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

int fuzzMeDrZaus(const uint8_t *Data, size_t size)
{

    //This is the main "folder" interface. It is the representation of the folder in the form a COM object interface.
    IShellFolder *psfParent = NULL; 

    //Each file/folder/virtual object is represented by a series of SHITEMIDs (SH item IDs, not shite MIDs as I read it)
    //Think of SHITEMIDs as filename or folder name and the LPITEMIDLIST as a fully qualified path or rather a list of 
    //SHITEMID structures
    LPITEMIDLIST pidlSystem;

    LPCITEMIDLIST pidlRelative = NULL;

    //this is where we're gong to shove the display name when we call GetDisplayNameOf (a human readable 
    //representation of an LPCITEMIDLIST)
    //The STRRET type is specific to the IShellFolder interface and is meant to hold its string. It's really a struct 
    //but it's fucking
    //complicated so let's just pretend it's a string.
    STRRET strDispName;
 
    //This is going to hold the actual string in the above struct, so it's an array of chars
    TCHAR szDisplayName[MAX_PATH];

    //The fucking result, for error checking we're not going to do because fuck that noise (and it slows down the fuzzer)
    HRESULT hr;

    /*
    SHSTDAPI SHGetFolderLocation(
      HWND             hwnd,
      int              csidl,
      HANDLE           hToken,
      DWORD            dwFlags,
      PIDLIST_ABSOLUTE *ppidl
    );
    */
   //This is really just being used as a vessel for that second arg for which CSIDL_SYSTEM will be specified
   //A CSIDL == Constant Special Item ID List
   //This function returns the path of a folder but as an ITEMIDLIST. So this is saying gimme an ITEMIDLIST
   //that points to the special System folder (system32). Neat.
   //The last argument is where the actual ITEMIDLIST is going to be stored, i.e. what we give a fuck about

    //Replace this folder location with fuzz data
    //hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);



    /*
    SHSTDAPI SHBindToParent(
      PCIDLIST_ABSOLUTE pidl,
      REFIID            riid,
      void              **ppv,
      PCUITEMID_CHILD   *ppidlLast
    );
    */
   //This function will return an interface pointer to the parent folder of pidlSystem (the ITEMIDLIST for system)
   //In other words it takes the first arg and shoves the parent of the ItemID into psfParent
   //pidRelative us the items PIDL relative to the parent folder. In other words this is saying
   //"give me the fucking parent of the thing I give you (System folder here). Give it to me as an IShellFolder item,
   //and while you're add it why don't you gimme the relative path to the parent folder too." Note we seem to be 
   //getting the System folder (i.e. "C:\Windows\System32"), grabbing the parent Windows\ and then getting the relative
   //path from there to System32 (i.e. "System32"). Seems kinda weird but OK.
    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);



    //redefine pidlSystem to be all fucked up
    pidlSystem->mkid.cb = (USHORT) size;
    std::memcpy(pidlSystem->mkid.abID, (const void *) Data, size);   

    //Assuming all the above shit checks out...
    if(SUCCEEDED(hr))
    {
        //use the IShellInterface COM object interface GetDisplayNameOf for the relative path of the parent folder to the system folder, 
        //give me a normal fucking string, and shove it in strDispName, note this is going to shove System32 into *strDispName.
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);

        //This function literally only exists to get the display name and store it into the third arg, szDisplayName
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));

        //Output the display name
        std::cout << "SHGDN_NORMAL - " <<szDisplayName << '\n';
    }


    //I..... release you.
    psfParent->Release();

    //Gimme my fucking memory back.
    CoTaskMemFree(pidlSystem);

    //Yay all is well. Or is it????? (it is)
    return 0;
};


extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {

  fuzzMeDrZaus(Data, Size);
  return 0;
}
```

So what exactly am I fuzzing here? You'll note the redefinition of pidlSystem has been moved down several lines so that the program flows as normal with no errors. Then when I get to the bit where it goes:

```
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);

        //This function literally only exists to get the display name and store it into the third arg, szDisplayName
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));
```

pidlSystem will have my fuzz data in it. Cool, so really what I'm fuzzing is two functions, one where I'm going to see if I've somehow effected the global state and am able to crash it (`GetDisplayNameOf` - I will be fuzzing this directly later) and my likely crash point of `StrRetToBuf`. Why do I call it my likely crash point? Because it directly uses `pidlSystem` and good god what an abomination of a function.

Let's give this a shot and see what happens. Oh bother, well I just realized I never gave you guys (metaphorical guys ffs) the string to compile all of this stuff. 

aaaaaaaaaaand on my first run, surprising no one, I get a WRITE access violation on a non-null page:

```
C:\Users\Garrett McParrot\Desktop\0day\shell32an
λ .\shell32_pwn.exe
INFO: Seed: 3812252186
INFO: Loaded 1 modules   (291 inline 8-bit counters): 291 [00007FF60BACBF08, 00007FF60BACC02B),
INFO: Loaded 1 PC tables (291 PCs): 291 [00007FF60BA71750,00007FF60BA72980),
INFO: -max_len is not provided; libFuzzer will not generate inputs larger than 4096 bytes
=================================================================
==20308==ERROR: AddressSanitizer: access-violation on unknown address 0x7ff8762b8443 (pc 0x7ff60b911402 bp 0x00000014f5f0 sp 0x00000014f260 T0)
==20308==The signal is caused by a WRITE memory access.
    #0 0x7ff60b911401 in fuzzMeDrZaus(unsigned char const *, unsigned __int64) C:\Users\Garrett McParrot\Desktop\0day\shell32an\shell32_pwn.cpp:76
    #1 0x7ff60b913e4a in LLVMFuzzerTestOneInput C:\Users\Garrett McParrot\Desktop\0day\shell32an\shell32_pwn.cpp:107
    #2 0x7ff60b9864ea in fuzzer::Fuzzer::ExecuteCallback(unsigned char const *, unsigned __int64) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:559
    #3 0x7ff60b987f76 in fuzzer::Fuzzer::ReadAndExecuteSeedCorpora(class std::vector<struct fuzzer::SizedFile, class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile>> &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:749
    #4 0x7ff60b98842a in fuzzer::Fuzzer::Loop(class std::vector<struct fuzzer::SizedFile, class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile>> &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:800
    #5 0x7ff60b99ee40 in fuzzer::FuzzerDriver(int *, char ***, int (__cdecl *)(unsigned char const *, unsigned __int64)) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerDriver.cpp:847
    #6 0x7ff60b961602 in main C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerMain.cpp:20
    #7 0x7ff60b9a68af in __scrt_common_main_seh d:\agent\_work\2\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288
    #8 0x7ff874dd7c23  (C:\Windows\System32\KERNEL32.DLL+0x180017c23)
    #9 0x7ff87630d4d0  (C:\Windows\SYSTEM32\ntdll.dll+0x18006d4d0)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: access-violation C:\Users\Garrett McParrot\Desktop\0day\shell32an\shell32_pwn.cpp:76 in fuzzMeDrZaus(unsigned char const *, unsigned __int64)
==20308==ABORTING
MS: 0 ; base unit: 0000000000000000000000000000000000000000

                                                                                                  
artifact_prefix='./'; Test unit written to ./crash-da39a3ee5e6b4b0d3255bfef95601890afd80709       
```

More analysis will be needed to see exactly what's going on here, but that could be a juicy one. It's always nice to know you can redirect flow somewhere unintended, I'll come back to that one for sure. Subsequent runs of the fuzzer seem to show null derefs as well:

```
C:\Users\Garrett McParrot\Desktop\0day\shell32an
λ .\shell32_pwn.exe
INFO: Seed: 145174053
INFO: Loaded 1 modules   (291 inline 8-bit counters): 291 [00007FF60BACBF08, 00007FF60BACC02B),
INFO: Loaded 1 PC tables (291 PCs): 291 [00007FF60BA71750,00007FF60BA72980),
INFO: -max_len is not provided; libFuzzer will not generate inputs larger than 4096 bytes
=================================================================
==10992==ERROR: AddressSanitizer: access-violation on unknown address 0x000000000010 (pc 0x7ff874f5a7e8 bp 0x000000000000 sp 0x00000014f1b0 T0)
==10992==The signal is caused by a READ memory access.
==10992==Hint: address points to the zero page.
    #0 0x7ff874f5a7e7  (C:\Windows\System32\SHELL32.dll+0x18005a7e7)
    #1 0x7ff874f5a716  (C:\Windows\System32\SHELL32.dll+0x18005a716)
    #2 0x7ff60b911317 in fuzzMeDrZaus(unsigned char const *, unsigned __int64) C:\Users\Garrett McParrot\Desktop\0day\shell32an\shell32_pwn.cpp:71
    #3 0x7ff60b913e4a in LLVMFuzzerTestOneInput C:\Users\Garrett McParrot\Desktop\0day\shell32an\shell32_pwn.cpp:107
    #4 0x7ff60b9864ea in fuzzer::Fuzzer::ExecuteCallback(unsigned char const *, unsigned __int64) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:559
    #5 0x7ff60b987f76 in fuzzer::Fuzzer::ReadAndExecuteSeedCorpora(class std::vector<struct fuzzer::SizedFile, class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile>> &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:749
    #6 0x7ff60b98842a in fuzzer::Fuzzer::Loop(class std::vector<struct fuzzer::SizedFile, class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile>> &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:800
    #7 0x7ff60b99ee40 in fuzzer::FuzzerDriver(int *, char ***, int (__cdecl *)(unsigned char const *, unsigned __int64)) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerDriver.cpp:847
    #8 0x7ff60b961602 in main C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerMain.cpp:20
    #9 0x7ff60b9a68af in __scrt_common_main_seh d:\agent\_work\2\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288
    #10 0x7ff874dd7c23  (C:\Windows\System32\KERNEL32.DLL+0x180017c23)
    #11 0x7ff87630d4d0  (C:\Windows\SYSTEM32\ntdll.dll+0x18006d4d0)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: access-violation (C:\Windows\System32\SHELL32.dll+0x18005a7e7)
==10992==ABORTING
MS: 0 ; base unit: 0000000000000000000000000000000000000000
```

But these are generally less fun, so let's keep moving. Now if you're looking carefully  you may notice something a bit fucked up. The last several crashes (even when I moved the code to fuck pidlSystem up later in the program) are all from either line 76 or line 71. The WRITE access violation is from line 76 which is:

`pidlSystem->mkid.cb = (USHORT) size;`

and the NULL deref is from line 71:

`hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);`

Note that my code that changes pidlSystem is AFTER line 71 (76,77). That tells me that there's some global state being kept there that gets messed up when I mess with pidlSystem. Let's put a pin in that and come back later should we find it interesting. I went ahead and moved that redefinition further down:

```

#include <shlobj.h>
#include <shlwapi.h>
#include <iostream>
#include <objbase.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

int fuzzMeDrZaus(const uint8_t *Data, size_t size)
{

    //This is the main "folder" interface. It is the representation of the folder in the form a COM object interface.
    IShellFolder *psfParent = NULL; 

    //Each file/folder/virtual object is represented by a series of SHITEMIDs (SH item IDs, not shite MIDs as I read it)
    //Think of SHITEMIDs as filename or folder name and the LPITEMIDLIST as a fully qualified path or rather a list of 
    //SHITEMID structures
    LPITEMIDLIST pidlSystem;

    LPCITEMIDLIST pidlRelative = NULL;

    //this is where we're gong to shove the display name when we call GetDisplayNameOf (a human readable 
    //representation of an LPCITEMIDLIST)
    //The STRRET type is specific to the IShellFolder interface and is meant to hold its string. It's really a struct 
    //but it's fucking
    //complicated so let's just pretend it's a string.
    STRRET strDispName;
 
    //This is going to hold the actual string in the above struct, so it's an array of chars
    TCHAR szDisplayName[MAX_PATH];

    //The fucking result, for error checking we're not going to do because fuck that noise (and it slows down the fuzzer)
    HRESULT hr;

    /*
    SHSTDAPI SHGetFolderLocation(
      HWND             hwnd,
      int              csidl,
      HANDLE           hToken,
      DWORD            dwFlags,
      PIDLIST_ABSOLUTE *ppidl
    );
    */
   //This is really just being used as a vessel for that second arg for which CSIDL_SYSTEM will be specified
   //A CSIDL == Constant Special Item ID List
   //This function returns the path of a folder but as an ITEMIDLIST. So this is saying gimme an ITEMIDLIST
   //that points to the special System folder (system32). Neat.
   //The last argument is where the actual ITEMIDLIST is going to be stored, i.e. what we give a fuck about

    //Replace this folder location with fuzz data
    //hr = SHGetFolderLocation(NULL, CSIDL_SYSTEM, NULL, NULL, &pidlSystem);



    /*
    SHSTDAPI SHBindToParent(
      PCIDLIST_ABSOLUTE pidl,
      REFIID            riid,
      void              **ppv,
      PCUITEMID_CHILD   *ppidlLast
    );
    */
   //This function will return an interface pointer to the parent folder of pidlSystem (the ITEMIDLIST for system)
   //In other words it takes the first arg and shoves the parent of the ItemID into psfParent
   //pidRelative us the items PIDL relative to the parent folder. In other words this is saying
   //"give me the fucking parent of the thing I give you (System folder here). Give it to me as an IShellFolder item,
   //and while you're add it why don't you gimme the relative path to the parent folder too." Note we seem to be 
   //getting the System folder (i.e. "C:\Windows\System32"), grabbing the parent Windows\ and then getting the relative
   //path from there to System32 (i.e. "System32"). Seems kinda weird but OK.
    hr = SHBindToParent(pidlSystem, IID_IShellFolder, (void **) &psfParent, &pidlRelative);




    //Assuming all the above shit checks out...
    if(SUCCEEDED(hr))
    {
        //use the IShellInterface COM object interface GetDisplayNameOf for the relative path of the parent folder to the system folder, 
        //give me a normal fucking string, and shove it in strDispName, note this is going to shove System32 into *strDispName.
        hr = psfParent->GetDisplayNameOf(pidlRelative, SHGDN_NORMAL, &strDispName);


        //redefine pidlSystem to be all fucked up
        pidlSystem->mkid.cb = (USHORT) size;
        std::memcpy(pidlSystem->mkid.abID, (const void *) Data, size);   

        //This function literally only exists to get the display name and store it into the third arg, szDisplayName
        hr = StrRetToBuf(&strDispName, pidlSystem, szDisplayName, sizeof(szDisplayName));

        //Output the display name
        std::cout << "SHGDN_NORMAL - " <<szDisplayName << '\n';
    }


    //I..... release you.
    psfParent->Release();

    //Gimme my fucking memory back.
    CoTaskMemFree(pidlSystem);

    //Yay all is well. Or is it????? (it is)
    return 0;
};


extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {

  fuzzMeDrZaus(Data, Size);
  return 0;
}
```

And now it crashes always at line 76, most of the time with a null ptr deref and some of the time with a WRITE access violation near here:

```
λ .\shell32_pwn.exe
INFO: Seed: 219112310
INFO: Loaded 1 modules   (291 inline 8-bit counters): 291 [00007FF683F8BF08, 00007FF683F8C02B),
INFO: Loaded 1 PC tables (291 PCs): 291 [00007FF683F31750,00007FF683F32980),
INFO: -max_len is not provided; libFuzzer will not generate inputs larger than 4096 bytes
=================================================================
==16588==ERROR: AddressSanitizer: access-violation on unknown address 0x7ff8762b8443 (pc 0x7ff683dd1402 bp 0x00000014f5f0 sp 0x00000014f260 T0)
==16588==The signal is caused by a WRITE memory access.
    #0 0x7ff683dd1401 in fuzzMeDrZaus(unsigned char const *, unsigned __int64) C:\Users\Garrett McParrot\Desktop\0day\shell32an\shell32_pwn.cpp:76
    #1 0x7ff683dd3e4a in LLVMFuzzerTestOneInput C:\Users\Garrett McParrot\Desktop\0day\shell32an\shell32_pwn.cpp:107
    #2 0x7ff683e464ea in fuzzer::Fuzzer::ExecuteCallback(unsigned char const *, unsigned __int64) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:559
    #3 0x7ff683e47f76 in fuzzer::Fuzzer::ReadAndExecuteSeedCorpora(class std::vector<struct fuzzer::SizedFile, class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile>> &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:749
    #4 0x7ff683e4842a in fuzzer::Fuzzer::Loop(class std::vector<struct fuzzer::SizedFile, class fuzzer::fuzzer_allocator<struct fuzzer::SizedFile>> &) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerLoop.cpp:800
    #5 0x7ff683e5ee40 in fuzzer::FuzzerDriver(int *, char ***, int (__cdecl *)(unsigned char const *, unsigned __int64)) C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerDriver.cpp:847
    #6 0x7ff683e21602 in main C:\src\llvm_package_1100-final\llvm-project\compiler-rt\lib\fuzzer\FuzzerMain.cpp:20
    #7 0x7ff683e668af in __scrt_common_main_seh d:\agent\_work\2\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288
    #8 0x7ff874dd7c23  (C:\Windows\System32\KERNEL32.DLL+0x180017c23)
    #9 0x7ff87630d4d0  (C:\Windows\SYSTEM32\ntdll.dll+0x18006d4d0)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: access-violation C:\Users\Garrett McParrot\Desktop\0day\shell32an\shell32_pwn.cpp:76 in fuzzMeDrZaus(unsigned char const *, unsigned __int64)
==16588==ABORTING
MS: 0 ; base unit: 0000000000000000000000000000000000000000


artifact_prefix='./'; Test unit written to ./crash-da39a3ee5e6b4b0d3255bfef95601890afd80709
```

OK, strangely nonsensical, let's keep fucking with shit though. I think we've exhausted what we can do with just copy and pasting that redefinition of pidlSystem. So where do we go from here? Couple of things, I know there's an API that takes in a string and converts it into an SHITEMIDLIST, I'd like to fuzz that as I'm sure there's a bug in that conversion. As I've said before, there's blood in the water here, and it's feeding time. The next thing I'd like to do is my original intent, which will get me something that I would really like: a weaponizable exploit. How am I gonna do it? I'm going to run explorer.exe a few times under DynamoRIO and use drcov to get coverage information. The functions that I hit when I just do a normal browse to a location or file properties or some user action are going to get fuzzed. Why? Because these will be guaranteed to be weaponizable. Even if it's just denial of service that's OK, I'll have learned something, but of course I'm hoping there's much much more. A browse to location and pwn exploit would be not only a ton of fun, but potentially worth something. The lsat thing I can do is start messing around with some of the structures, like the STRRET struct would be a nice one to build from scratch, then fuzz the appropriate fields of when feeding into a function. There of course may be some overlap in all of these, but so far so good. I've lost count of the 0-day vulns in this and that's a win in itself.

Oh right and I also realized I haven't mentioned the string I'm using to compile this stuff:

`clang-cl.exe /Zi -fsanitize=fuzzer,address -fsanitize-recover=address shell32_pwn.cpp ole32.lib shell32.lib shlwapi.lib`

will link to libFuzzer and then just executing the executable will kick off fuzzing. libFuzzer, you donE good. Later, when I start taking in names of strings I may try using AFL and grabbing names from files on the filesystem, we'll see! If you enjoyed this, follow me on twitter, i'm @hyp3ri0n, and don't forget to donate to your favorite open source project (I don't care if it's not mine)! Signing out for now :).





