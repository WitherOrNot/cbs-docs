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

Component folders in WinSxS use names, known as key forms, based on the `assemblyIdentity` tag in the component manifest. The naming scheme is as follows:

`<processorArchitecture>_<name>_<publicKeyToken>_<version>_<language>_<pseudokey>`

All values except `pseudokey` correspond to an attribute of the `assemblyIdentity` tag with the same name. The `pseudokey` value is a hash based on all available attributes of this tag, an implementation of its calculation can be found in the open-source project [haveSxS](https://github.com/asdcorp/haveSxS). To generate the final name, each attribute value is converted to lowercase, then certain attributes are truncated to fit an appropriate length. The table of lengths for each truncatable attribute is given below:

|Attribute|Length|
|-|-|
|`language`|16|
|`name`|40|
|`version`|8|

If an attribute value exceeds the prescribed length, truncation is done by computing `k = (length / 2) - 1`, then setting the value to `<first k characters>..<last k characters>`. As an example, the `name` value `Microsoft-Windows-NetworkDiagnosticsFrameworkCore` exceeds the length limit of `40`, so the first and last `(40 / 2) - 1 = 19` characters would be used, truncating to `microsoft-windows-n..osticsframeworkcore`.

## Servicing Stack

The servicing stack is composed of a series of binaries located in the `Microsoft-Windows-ServicingStack` component folder. These binaries by themselves do not constitute any processes or services, instead services or tools that want to manage packages use the APIs exposed by these binaries.

The most important binaries that are part of the servicing stack are:
 - `CbsCore.dll`, implements the CBS/Trusted Installer (TI) API, responsible for package installation and dependency management
 - `wcp.dll`, implements the Component Servicing Infrastructure (CSI), responsible for component/deployment installation
 - `smiengine.dll` and `smipi.dll`, implement the Service Management Infrastructure (SMI), responsible for simple file and registry modifications
 - `poqexec.exe`, responsible for carrying out individual file and registry operations during online setup
 - `wdscore.dll`, implements the Panther Engine, responsible for Windows image deployment and CBS logging
 - `TurboStack.dll`, (introduced in Windows 10 1809), implements multithreaded servicing operations, required in Windows 11
 - `mspatcha.dll`, implements PA19 delta compression
 - `msdelta.dll`, implements PA30 delta compression

In practice, a typical program interacting with CBS only needs to load `CbsCore.dll` for the CBS API and `wdscore.dll` for logging.

## CBS API

Although most of the servicing stack is comprised of DLLs which export useful APIs, the vast majority of them are wrapped by the CBS API, found in `CbsCore.dll`. The CBS API primarily uses the Component Object Model (COM). Although this API is mostly undocumented, methods to interact with it have been figured out through reverse engineering.

To begin interacting with the API, the function `CbsCoreInitialize` must be initialized. Its function definition is as follows:

```c++
typedef int (__stdcall FUNC_INT*)(int);
typedef void (FUNC_VOID*)(void);

HRESULT WINAPI CbsCoreInitialize(
    IMalloc* pMlloc,
    FUNC_INT* TiLockProcess,
    FUNC_VOID* TiUnlockProcess,
    FUNC_VOID* InstanceCreated,
    FUNC_VOID* InstanceDestroyed,
    FUNC_VOID* Unknown,
    FUNC_VOID* RequireShutdownProcessing,
    IClassFactory** ppClassFactory
);
```

Functions of type `FUNC_INT` must return `1` on success. It is enough to stub these functions for successful interaction with the CBS API. Please note that successful initialization requires administrator privileges, though it is recommended that a CBS API client run as TrustedInstaller. A reference implementation of code to obtain TI privileges can be found in the [superUser](https://github.com/mspaintmsi/superUser) project.

Once the API is initialized, a session must be created to contain package operations. This is done like so:

```c++
pClassFactory->CreateInstance(NULL, __uuidof(ICbsSession), &pSession);
oSession->Initialize(CbsSessionOption::OPTION, L"Client Name", bootDrive, winDir);
```

The list of `CbsSessionOption` values to replace `OPTION` can be found [here](https://github.com/WitherOrNot/cbsexploder/blob/319344c0641e6d280e6a592e6f61e07a22d8ff47/cbsexploder/CbsApi.h#L79). For offline servicing of a disk image or mounted disk, `bootDrive` must specify the path of the Windows installation, and `winDir` must specify the associated `Windows` directory. For online servicing of the local system, both of these values must be `NULL` instead.

To load a package, the `CreatePackage` function must be used, like so:

```c++
pSession->CreatePackage(0, CbsPackageType::PACKAGE_TYPE, packagePath, sandboxPath, &pPackage);
```

The list of available values for `PACKAGE_TYPE` can be found [here](https://github.com/WitherOrNot/cbsexploder/blob/319344c0641e6d280e6a592e6f61e07a22d8ff47/cbsexploder/CbsApi.h#L40). If specifying a cabinet file as a package, `sandboxPath` must point to a valid directory to allow for decompression. For expanded packages, the value `ExpandedWithMum` should be used, and `sandboxPath` can be left as `NULL`.

The package's current and applicable state can be checked before operations with `EvaluateApplicability`:

```c++
pPackage->EvaluateApplicability(0, &applicableState, &currentState);
```

The possible values for the current and applicable states can be found [here](https://github.com/WitherOrNot/cbsexploder/blob/319344c0641e6d280e6a592e6f61e07a22d8ff47/cbsexploder/CbsApi.h#L11). If the applicable state is not `Installed`, then a dependency for the package is missing and must be installed in a prior session.

Finally, to queue and execute package operations, `InitiateChanges` and `FinalizeEx` are used:

```c++
pPackage1->InitiateChanges(0, CbsState::STATE1, pUIHandler);
pPackage2->InitiateChanges(0, CbsState::STATE2, pUIHandler);
...
pSession->FinalizeEx(0, &requiredAction);
```

The possible values for package states are the same as those for the current and applicable states. `pUIHandler` references an implementation of `ICbsUIHandler`, whose reference implementation can be found [here](https://github.com/WitherOrNot/cbsexploder/blob/319344c0641e6d280e6a592e6f61e07a22d8ff47/cbsexploder/uihandler.cpp). As shown, multiple operations for multiple packages can be queued at once before being executed by `FinalizeEx`. The value of `requiredAction` is `1` if a reboot is required and `0` otherwise.

## Package Management

