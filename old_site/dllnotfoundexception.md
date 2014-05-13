---
layout: obsolete
title: "DllNotFoundException"
permalink: /old_site/DllNotFoundException/
redirect_from:
  - /DllNotFoundException/
---

DllNotFoundException
====================

<table>
<col width="100%" />
<tbody>
<tr class="odd">
<td align="left"><h2>Table of contents</h2>
<ul>
<li><a href="#Overview">1 Overview</a></li>
<li><a href="#Background_Information_.26_Possible_Questions">2 Background Information &amp; Possible Questions</a></li>
<li><a href="#Troubleshooting_DllNotFoundExceptions">3 Troubleshooting DllNotFoundExceptions</a></li>
<li><a href="#Additional_Information">4 Additional Information</a></li>
</ul></td>
</tr>
</tbody>
</table>

Overview
--------

So you are enjoying your day developing with mono when all of a sudden you run into a problem:

    $ mono GdiExample.exe

    Unhandled Exception: System.TypeInitializationException: An exception was thrown by the type initializer for  System.Drawing.GDIPlus ---> System.DllNotFoundException: gdiplus.dll
    in (wrapper managed-to-native) System.Drawing.GDIPlus:GdiplusStartup  (ulong&,System.Drawing.GdiplusStartupInput&,System.Drawing.GdiplusStartupOutput&)
    in <0x000bb> System.Drawing.GDIPlus:.cctor ()--- End of inner exception stack trace ---

    in <0x00000> <unknown method>
    in <0x0004d> System.Drawing.SolidBrush:.ctor (Color color)
    in (wrapper remoting-invoke-with-check) System.Drawing.SolidBrush:.ctor (System.Drawing.Color)
    in <0x0004b> GdiExample:Main ()

Panic. Confusion. Chaos. These are generally the common reactions to such an Exception.

Fortunetly there is hope. This error means that mono was unable to locate a library that one of the classes you are trying to use needs. This guide is here to help.

Background Information & Possible Questions
-------------------------------------------

**I am using Linux or OSX, why is mono looking for libraries with "win32" in their name and ending with "dll" instead of "so"/"dylib"?**

To preserve compatibility with the .NET Framework, mono uses the same library names as windows. These names are mapped to linux library names using [DllMaps]({{site.github.url}}/old_site/Config_DllMap).

    <dllmap dll="libgtk-win32-2.0-0.dll" target="libgtk-x11-2.0.so.0"/>

(Substitute ".so" with "dylib" for MacOS X)

Troubleshooting DllNotFoundExceptions
-------------------------------------

A good first step is to set up the log level so you can see what file names mono is attempting to look for:

    $ MONO_LOG_LEVEL=debug mono GdiExample.exe
    <snip>
    Mono-INFO: DllImport attempting to load: 'gdiplus.dll'.
    Mono-INFO: DllImport loading location: 'libgdiplus.dll.so'.
    Mono-INFO: DllImport error loading library: 'libgdiplus.dll.so: cannot open shared object file: No such file or directory'.
    Mono-INFO: DllImport loading library: './libgdiplus.dll.so'.
    Mono-INFO: DllImport error loading library './libgdiplus.dll.so: cannot open shared object file: No such file or directory'.
    Mono-INFO: DllImport loading: 'gdiplus.dll'.
    Mono-INFO: DllImport error loading library 'gdiplus.dll: cannot open shared object file: No such file or directory'.
    Mono-INFO: DllImport loading location: 'libgdiplus.so'.
    Mono-INFO: DllImport error loading library: 'libgdiplus.so: cannot open shared object file: No such file or directory'.
    Mono-INFO: DllImport loading library: './libgdiplus.so'.
    Mono-INFO: DllImport error loading library './libgdiplus.so: cannot open shared object file: No such file or directory'.
    Mono-INFO: DllImport loading: 'gdiplus'.
    Mono-INFO: DllImport error loading library 'gdiplus.so: cannot open shared object file: No such file or directory'.
    Mono-INFO: DllImport loading location: 'libgdiplus.dll'.
    Mono-INFO: DllImport error loading library: 'libgdiplus.dll: cannot open shared object file: No such file or directory'.
    Mono-INFO: DllImport loading library: './libgdiplus.dll'.
    Mono-INFO: DllImport error loading library './libgdiplus.dll: cannot open shared object file: No such file or directory'.
    Mono-INFO: DllImport loading: 'libgdiplus.dll'.
    Mono-INFO: DllImport error loading library 'libgdiplus.dll: cannot open shared object file: No such file or directory'.

    (GdiExample.exe:18793): Mono-WARNING **: DllImport unable to load library 'libgdiplus.dll: cannot open shared object file: No such file or directory'.
    <snip>

What we see here is that mono is making several attemps to locate the missing library, by prepending "lib", replacing the ".dll" extension with ".so", etc, but it was still unable to find the library.

If a library location has not been explicitly specified in a [DllMap]({{site.github.url}}/old_site/Config_DllMap) entry in an application or assembly .config file, Mono will search for a library in a few places:

-   The directory where the referencing image was loaded from.
-   In any place the system's dynamic loader is configured to look for shared libraries. For example on Linux this is specified in the \$LD\_LIBRARY\_PATH environment variable and the /etc/ld.so.conf file. On windows the \$PATH environment variable is used, instead.

The next step in resolving the issue is locating the file on your system.

    $ find /usr -name libgdiplus.so
    /usr/local/lib/libgdiplus.so

Aha! There it is. So why can't mono find it?

By default in basically all distributions of linux the dynamic linker only creates a cache for files in /lib and /usr/lib. Since you have placed your library in /usr/local/lib, it does not know about it. You can verify this using the following command:

    $ ldconfig -p |grep libgdiplus

The command should produce no output, because it does not know about the file.

The correct way to fix this problem is to add /usr/local/lib as one of the paths that ldconfig indexes. To do this, add the path to /etc/ld.so.conf and run "ldconfig" as root, which will force the cache to be rebuilt.

The dynamic linker should now know about all the libraries in this path. You can verify this by typing the above command once again:

    $ ldconfig -p |grep libgdiplus
          libgdiplus.so (libc6) => /usr/local/lib/libgdiplus.so

Yay! Your application should now run properly.

NOTE: As mentioned above you can also set the \$LD\_LIBRARY\_PATH environment variable to the path containing the library, but this is not as recomended because it can cause other problems and is more difficult to maintain.

Additional Information
----------------------

-   [Assemblies\_and\_the\_GAC]({{site.github.url}}/old_site/Assemblies_and_the_GAC "Assemblies and the GAC")
-   [DllMap]({{site.github.url}}/old_site/Config_DllMap)
-   [How the Runtime Locates Assemblies (MSDN)](http://msdn.microsoft.com/library/default.asp?url=/library/en-us/cpguide/html/cpconhowruntimelocatesassemblies.asp)

