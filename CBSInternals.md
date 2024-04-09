# CBS Internals

## Introduction

The last chapter introduced the servicing stack, which is the collection of parts that carry out Windows servicing. In this chapter, we will explore the servicing stack in greater depth, covering what exactly it is, how it works, and how a developer can interact with it.

## WinSxS

The servicing stack is located in and operates on the WinSxS folder, located in `%SystemRoot%\WinSxS`. This folder has the following structure:

 - `Backups`, contains copies of manifests and files for critical system components
 - `Catalogs`, contains signed catalogs containing the hash of each manifest
 - `FileMaps`, contains `cdf-ms` files which hold data on the location of system files
 - `Manifests`, contains manifests for components and deployments, with the `.manifest` extension
 - `InstallTemp` and `Temp`, used for temporary files during servicing
 - `Fusion` and `FusionDiff`, (Windows 11) used to store SxS components for app libraries, similar to legacy SxS
 - Component folders

### Component Naming Scheme

Component folders in WinSxS use names, known as key forms, based on the `assemblyIdentity` tag in the component manifest. This scheme is as follows:

`<processorArchitecture>_<name>_<publicKeyToken>_<version>_<language>_<pseudokey>`

All values except `pseudokey` correspond to an attribute of the `assemblyIdentity` tag with the same name. The `pseudokey` value is a hash based on all available attributes of this tag, an implementation of its calculation can be found in the open-source project [haveSxS](https://github.com/asdcorp/haveSxS). To generate the final name, each attribute value is converted to lowercase, then certain attributes are truncated to fit an appropriate length. The table of lengths for each truncatable attribute is given below:

|Attribute|Length|
|-|-|
|`culture`|16|
|`name`|40|
|`language`|8|

If an attribute value exceeds the prescribed length, truncation is done by computing `k = (length / 2) - 1`, then setting the value to `<first k characters>..<last k characters>`. As an example, the `name` value `Microsoft-Windows-NetworkDiagnosticsFrameworkCore` exceeds the length limit of `40`, so the first and last `(40 / 2) - 1 = 19` characters would be used, truncating to `microsoft-windows-n..osticsframeworkcore`.

## Servicing Stack

The servicing stack is composed of a series of binaries located in the `Microsoft-Windows-ServicingStack` component folder. These binaries by themselves do not constitute any processes or services, instead services or tools that want to manage packages use the APIs exposed by these binaries.

The most important binaries that are part of the servicing stack are:
 - `CbsCore.dll`, implements the CBS/Trusted Installer (TI) API, responsible for package installation and dependency management
 - `wcp.dll`, implements the Component Servicing Infrastructure (CSI), responsible for component/deployment installation
 - `smiengine.dll` and `smipi.dll`, implement the Service Management Infrastructure (SMI), responsible for simple file and registry modifications
 - `poqexec.exe`, implements the Kernel Transaction Manager (KTM), responsible for carrying out individual file and registry operations during online setup
 - `wdscore.dll`, implements the Panther Engine, responsible for Windows image deployment and CBS logging

In practice, a typical program interacting with CBS only needs to load `CbsCore.dll` for the CBS API and `wdscore.dll` for logging.

## Package Installation
