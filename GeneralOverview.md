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

As shown, manifests are able to describe assembly dependencies, file integrity information, and installation information in a readable format. To prevent corruption or malicious modification of these manifests, they are stored alongside signed catalog files containing their hashes.

Additionally, to provide the SxS functionality, manifests specify an `assemblyIdentity` tag that uniquely identifies a particular assembly, specifying its name, version, architecture, signing key, and runtime. This information is then specified by programs in their own manifests (usually located within the binary) to load a particular version of a library. The information in this tag is also used in the manifest naming system, which will be covered in a later chapter.

## Windows Componentization

The fundamental idea behind CBS is to extend the SxS system to not only store program libraries, but the parts that make up Windows itself.

Storing the OS this way provides immediate benefits, such as providing simpler methods for system updates and system file corruption detection. It also centralizes and organizes the storage of system files, making them easier to manage for Microsoft and system administrators. Due to these benefits, Microsoft introduced CBS with Windows Vista, and it has served as the primary method of managing system files ever since.

CBS, like SxS, works in terms of assemblies, with only difference being that assemblies are now used to hold system data rather than program libraries. The 3 primary kinds of CBS assemblies are:
 - Components, which group a set of related files, registry keys, and other items
 - Deployments, which group a set of related components
 - Packages, which group a set of related deployments and sub-packages

### Components

Components are the most fundamental unit of Windows servicing. They tend to house only a small set of files, and they can also contain extended metadata related to registry changes, permissions, and other installation information that is needed to correctly set up these files. Components may also specify other components as dependencies.

Example manifest:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v3" manifestVersion="1.0" copyright="Copyright (c) Microsoft Corporation. All Rights Reserved.">
  <assemblyIdentity name="Microsoft-Windows-VolumeActivation-EventQuery" version="6.2.8250.0" processorArchitecture="amd64" language="neutral" buildType="release" publicKeyToken="31bf3856ad364e35" versionScope="nonSxS" />
  <file name="VolumeActivation.Events.xml" destinationPath="$(runtime.programData)\Microsoft\Event Viewer\Views\ServerRoles\" sourceName="VolumeActivation.Events.xml" sourcePath=".\" importPath="$(build.nttree)\">
    <securityDescriptor name="WRP_FILE_DEFAULT_SDDL" />
    <asmv2:hash xmlns:asmv2="urn:schemas-microsoft-com:asm.v2">
      <dsig:Transforms xmlns:dsig="http://www.w3.org/2000/09/xmldsig#">
        <dsig:Transform Algorithm="urn:schemas-microsoft-com:HashTransforms.Identity" />
      </dsig:Transforms>
      <dsig:DigestMethod xmlns:dsig="http://www.w3.org/2000/09/xmldsig#" Algorithm="http://www.w3.org/2000/09/xmldsig#sha256" />
      <dsig:DigestValue xmlns:dsig="http://www.w3.org/2000/09/xmldsig#">UZq+KgtzhkSi0GBqhcE16sHbVjSqibRrRdWaG/KUsmc=</dsig:DigestValue>
    </asmv2:hash>
  </file>
  <trustInfo>
    <security>
      <accessControl>
        <securityDescriptorDefinitions>
          <securityDescriptorDefinition name="WRP_FILE_DEFAULT_SDDL" sddl="O:S-1-5-80-956008885-3418522649-1831038044-1853292631-2271478464G:S-1-5-80-956008885-3418522649-1831038044-1853292631-2271478464D:P(A;;FA;;;S-1-5-80-956008885-3418522649-1831038044-1853292631-2271478464)(A;;GRGX;;;BA)(A;;GRGX;;;SY)(A;;GRGX;;;BU)(A;;GRGX;;;S-1-15-2-1)S:(AU;FASA;0x000D0116;;;WD)" operationHint="replace" description="Default SDDL for Windows Resource Protected file" />
        </securityDescriptorDefinitions>
      </accessControl>
    </security>
  </trustInfo>
</assembly>
```

### Deployments

Deployments are used to categorize a set of components that must be installed together. Deployments must contain the `deployment` tag, and they reference the contained components as dependencies.

Example manifest:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v3" manifestVersion="1.0" copyright="Copyright (c) Microsoft Corporation. All Rights Reserved.">
  <assemblyIdentity name="Microsoft-Hyper-V-Management-Client-Interop-Deployment" version="6.2.8250.0" processorArchitecture="amd64" language="neutral" buildType="release" publicKeyToken="31bf3856ad364e35" versionScope="nonSxS" />
  <dependency discoverable="no">
    <dependentAssembly dependencyType="install">
      <assemblyIdentity name="Microsoft.Virtualization.Client.RdpClientAxHost" version="6.2.8250.0" processorArchitecture="msil" language="neutral" buildType="release" publicKeyToken="31bf3856ad364e35" versionScope="nonSxS" />
    </dependentAssembly>
  </dependency>
  <dependency discoverable="no">
    <dependentAssembly dependencyType="install">
      <assemblyIdentity name="Microsoft.Virtualization.Client.RdpClientInterop" version="6.2.8250.0" processorArchitecture="msil" language="neutral" buildType="release" publicKeyToken="31bf3856ad364e35" versionScope="nonSxS" />
    </dependentAssembly>
  </dependency>
  <deployment xmlns="urn:schemas-microsoft-com:asm.v3" />
</assembly>
```

### Packages

Packages are used to group a set of deployments and packages to deliver a single feature set. Packages must contain the `package` tag, and each included deployment or sub-package is contained within an `update` tag. During servicing, Windows may either install all the updates in a package or select particular updates as needed. Packages may also specify parent packages with the `parent` tag. If a given package's parent is not available during installation, the given package's installation will fail. Additionally, the package release type is specified by the `releaseType` attribute of the `package` tag. This value affects how CBS chooses to handle requests to install or uninstall packages.

Example manifest:

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v3" copyright="Copyright (c) Microsoft Corporation. All Rights Reserved." manifestVersion="1.0">
  <assemblyIdentity buildType="release" language="en-US" name="Microsoft-Windows-BusinessScanning-Feature-Package-admin" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" version="6.2.8250.0"/>
  <package identifier="Microsoft-Windows-BusinessScanning-Feature-Package-admin LP" releaseType="Language Pack">
    <parent disposition="detect" integrate="separate">
      <assemblyIdentity buildType="release" language="neutral" name="Microsoft-Windows-BusinessScanning-Feature-Package-admin" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" version="6.2.8250.0"/>
    </parent>
    <update description="Wraps admin components depot contributing to Microsoft-Windows-BusinessScanning-Feature-Package-admin" displayName="Microsoft-Windows-BusinessScanning-Feature-Package-admin" name="Microsoft-Windows-BusinessScanning-Feature-Package-admin">
      <component>
        <assemblyIdentity buildType="release" language="en-US" name="Microsoft-Windows-BusinessScanning-Feature-Package-admin-deployment-LanguagePack" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" version="6.2.8250.0" versionScope="nonSxS"/>
      </component>
    </update>
  </package>
</assembly>
```

## Servicing Stack

Each type of assembly is processed by the parts that carry out the CBS process, which are collectively known as the servicing stack. In order of highest to lowest level, the servicing stack consists of the following:

 - Trusted Installer (TI), responsible for managing packages
 - Component Servicing Infrastructure (CSI), responsible for managing deployments and components
 - Component Management Infrastructure (CMI), responsible for high level component installation
 - Driver Management and Install (DMI), responsible for driver installation
 - Systems Management Infrastructure (SMI), responsible for low level component installation
 - Kernel Transaction Manager (KTM), responsible for managing individual operations

The next chapter will explore each of these parts in detail, as well as how a developer can interact with them.

## Sources

 - [Understanding Component-Based Servicing - Microsoft Community Hub](https://techcommunity.microsoft.com/t5/ask-the-performance-team/understanding-component-based-servicing/ba-p/373012)
 - [Side-by-side assembly - Wikipedia](https://en.wikipedia.org/wiki/Side-by-side_assembly)
