# General Overview

## Introduction

Component-Based Servicing (CBS) is a subsystem of Windows that underlies many important capabilities, including image deployment, Features on Demand, system updates, and system corruption repair.

Despite its overarching reach and importance, even most advanced Windows users are unaware of CBS, how it works, and all the things it is involved in. This is partially by design, as Microsoft has released little official information as to how CBS works, and much of its functionality is locked away behind undocumented APIs. As such, the most comprehensive sources of information on CBS are scattered between various online chatrooms and obscure forums, making them hard to find and vulnerable to being lost. Thus, I have decided to compile these sources, along with my own findings, into a organized series of writeups.

Due to the limited scope and availability of this information, these writeups will cover desktop Windows versions 7 through 11, with a focus on 8 through 10. Additionally, most of these writeups will center around the theme of image deployment, since it is (in my opinion) the most interesting thing that CBS is used for.

## Side-by-Side Assembly

In order to understand CBS, we must begin with its predecessor, the Side-by-Side (SxS) assembly system.

SxS is a subsystem introduced in Windows 98 Second Edition and Windows 2000 which allows multiple versions and variants of software libraries to coexist on a system, providing a way for programs to select a specific version on demand. It also provides security and integrity-checking features that prevent these libraries from being maliciously modified or corrupted.

To accomplish this, libraries, known as assemblies, are accompanied by an associated XML file, known as a manifest, which describes their contents and associations with other assemblies. An example manifest is given below:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
    <assemblyIdentity version="5.2.2.3" processorArchitecture="x86" name="Microsoft.Windows.Networking.RtcDll" type="win32" publicKeyToken="6595b64144ccf1df"/>
    <description>RTC Core DLL</description>
    <dependency>
        <dependentAssembly>
            <assemblyIdentity type="win32" name="Microsoft.Windows.Networking.DxmRtp" version="5.2.2.3" processorArchitecture="x86" publicKeyToken="6595b64144ccf1df" language="*"/>
        </dependentAssembly>
    </dependency>
    <dependency optional="yes">
        <dependentAssembly>
            <assemblyIdentity type="win32" name="Microsoft.Windows.Networking.RtcRes" version="5.2.2.3" processorArchitecture="x86" publicKeyToken="6595b64144ccf1df" language="*"/>
        </dependentAssembly>
    </dependency>
    <file name="rtcdll.dll" hash="e0391579b3315433346a80ba86d0876f4dd669d9" hashalg="SHA1">
        <comClass description="RTCClient Class" clsid="{7a42ea29-a2b7-40c4-b091-f6f024aa89be}" threadingModel="Apartment"/>
        <typelib tlbid="{cd260094-de10-4aee-ac73-ef87f6e12683}" version="1.1" helpdir=""/>
    </file>
</assembly>
```

As shown, manifests are able to describe assembly dependencies, file integrity information, and installation information in a readable format. To prevent corruption or malicious modification of these manifests, they are store alongside signed catalog files containing their hashes.

Additionally, to provide the SxS functionality, manifests specify an `assemblyIdentity` tag that uniquely identifies a particular assembly, specifying its name, version, architecture, signing key, and runtime. This information is then specified by programs in their own manifests (usually located within the binary) to load a particular version of a library. The information in this tag is also used in the manifest naming system, which will be covered in a later chapter.

## Windows Componentization

The fundamental idea behind CBS is to extend the SxS system to not only store program libraries, but the parts that make up Windows itself.

Storing the OS this way provides immediate benefits, such as providing simpler methods for system updates and system file corruption detection. It also centralizes and organizes the storage of system files, making them easier to manage for Microsoft and system adminsistrators.

To accomplish this, CBS uses 3 primary kinds of assemblies:
 - Components, which group a set of related files, registry keys, and other items
 - Deployments, which group a set of related components
 - Packages, which group a set of related deployments

These assemblies are processed by the various parts that make up CBS for operations like installation, upgrading, and removal. 
