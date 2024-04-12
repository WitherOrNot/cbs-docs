# Image Deployment

## Introduction

Beginning with Windows Vista, all Windows client and server SKUs were built from a unified codebase to prevent feature lag. This poses a problem as to how to construct installable Windows images for every edition that contain the correct files and licenses, as well as allowing offline edition upgrades. Luckily, CBS is well-equipped to solve this problem, which is why it plays an essential role in the construction of Windows installation images.

## Foundation Image

The first step in constructing a Windows image is the creation of a foundation image. Prior to Windows 8, these foundation images appear to have been generated during postbuild and populated with all the files that are common between SKUs. Although it remains unclear how foundation images were created for Windows 8 and 8.1, the construction of foundation images for Windows 10 and 11 is well understood. Additionally, testing has found that foundation images generated for Windows 10 can be used to deploy Windows 8 and 8.1 images.

To create a foundation image, the exported functions `CreateNewWindows` and `CreateNewOfflineStore` from `wcp.dll` are used. These have the following declarations:

```c++
HRESULT WINAPI CreateNewWindows(
    DWORD dwFlags,
    LPCWSTR pszSystemDrive,
    POFFLINE_STORE_CREATION_PARAMETERS pParameters,
    PVOID* ppvKeys,
    DWORD* pdwDisposition
);

HRESULT WINAPI CreateNewOfflineStore(
    DWORD dwFlags,
    PCOFFLINE_STORE_CREATION_PARAMETERS pParameters,
    REFIID riid,
    IUnknown** ppStore,
    DWORD* pdwDisposition
);
```

First, `CreateNewWindows` is called like so:

```c++
OFFLINE_STORE_CREATION_PARAMETERS pParameters = { sizeof(OFFLINE_STORE_CREATION_PARAMETERS) };
pParameters.pszHostSystemDrivePath = L"<Path to Offline Image Root>";
pParameters.ulProcessorArchitecture = ARCHITECTURE;
void* regKeys;
DWORD disposition;
CreateNewWindows(0, L"X:\\", &pParameters, &regKeys, &disposition);
```

Possible values of `ARCHITECTURE` can be found [here](https://learn.microsoft.com/en-us/uwp/api/windows.system.processorarchitecture). The architecture value reflects the processor architecture of the offline image.

Once the image is created, the `COMPONENTS` hive is then populated by `CreateNewOfflineStore` like so:

```c++
ICSIStore* store;
CreateNewOfflineStore(0, &pParameters, __uuidof(ICSIStore), (IUnknown**)&store, &disposition);
```

Upon successful completion, the image is ready to be serviced offline by CBS. An implementation of this foundation image generation can be found in [SxSFounder](https://github.com/WitherOrNot/sxsfounder).

## Package Types

Generally, Windows images are composed of the following types of primary packages:
 - Foundation Packages
 - Edition Packages
 - Language Packs
 - Features on Demand

### Foundation Packages

Foundation packages contain the minimum possible set of components needed for an image to be serviceable using its own servicing stack. This typically includes the core servicing stack, advanced installer DLLs, components for DISM, and core windows APIs. Once these components are present, CBS will prefer using the offline servicing stack for offline operations.

### Edition Packages

Edition packages contain all non-localized components for a particular Windows edition, including the foundation package as a dependency.

### Language Packs

Language packs (LPs) contain all localized components for all Windows editions of either the Client or Server variety. If an edition package is installed without a language pack, the system will crash with the stop code `MUI_NO_VALID_SYSTEM_LANGUAGE`. This may also occur if the installed language packs do not match the allowed system languages specified in the system product policy.

### Features on Demand

Features on Demand (FoD), a system introduced in Windows 8, allow for system features to be added after image deployment. Modern Windows versions usually come preinstalled with a few FoDs and allow others to be installable over Windows Update.

## Staged Image Deployment

Staged images are the most common form of Windows installation image, where each edition is installed directly to a separate sub-image.

Prior to Windows 10 1903, this installation was done with DISM. From late Windows 10 onwards, this installation was done with CBSS, an internal fork of Vista's PkgMgr. Thanks to leaks from internal test updates, CBSS is [publicly available](https://archive.org/download/windows-10.0-kb-9999999-x-64).

Since it is unclear from available data whether the DISM API or command-line tool was used, DISM commands will be used to represent operations done with DISM for convenience.

### Pre-Servicing

For Windows 7 up through early Windows 10, certain modifications to the foundation image need to be made prior to servicing.

First, non-package legacy catalogs must be copied to `\Windows\System32\CatRoot\{F750E6C3-38EE-11D1-85E5-00C04FC295EE}`, so that they are included in the catalog database. Without these catalogs present, older Windows versions may fail to boot due to system files lacking an available signature.

Then, for Windows 8 through Windows 10 1511, [fingerprinting data](https://betawiki.net/wiki/Leak_prevention_in_Microsoft_Windows) must be added to the registry at `HKLM\SYSTEM\WPA\478C035F-04BC-48C7-B324-2462D786DAD7-5P-9` and to the filesystem at `\Windows\System32\config\FP`. Without this data present, Windows may fail to boot or crash during usage with the stop code `KERNEL_SECURITY_CHECK_FAILURE` or `CRITICAL_STRUCTURE_CORRUPTION`.

### DISM Installation

To install with DISM, the environment variable `WINDOWS_WCP_INSKUASSEMBLY` must be set to `1`, and the foundation package must be deployed to the image without being committed as an installed package. Then, the edition packages are added using a DISM unattend file:

```
dism /image:<Offline Image Path> /apply-unattend:<Path to Unattend file>
```

The contents of the unattend file specify to install the target edition and stage all editions that it can upgrade to. Example:

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
    <servicing>
        <package action="install">
            <assemblyIdentity name="Microsoft-Windows-CoreEdition" version="10.0.22000.1" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" />
            <source location="packages\Microsoft-Windows-CoreEdition~31bf3856ad364e35~amd64~~10.0.22000.1.mum" />
        </package>
        <package action="stage">
            <assemblyIdentity name="Microsoft-Windows-ProfessionalEdition" version="10.0.22000.1" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" />
            <source location="packages\Microsoft-Windows-ProfessionalEdition~31bf3856ad364e35~amd64~~10.0.22000.1.mum" />
        </package>
        ...
    </servicing>
</unattend>
```

### CBSS Installation

To install with publicly available CBSS, the environment variable `PKGMGR_RUNNING_IN_OFFLINE_STACK_ALREADY` must be set to `1`. Modern CBSS is run with an undocumented `/ls` flag, which is presumed to have an equivalent effect. Once set, installation is done using a CBSS batch file:

```
cbss [/ls] /o:<Offline Image Windows Directory> /b:<Path to Batch File>
```

The contents of the batch file are similar to a DISM unattend file, also specifying installation of the target edition and staging all available upgrades. Example:

```
/ip /m:packages\Microsoft-Windows-CoreEdition~31bf3856ad364e35~amd64~~10.0.22000.1.mum
/sp /m:packages\Microsoft-Windows-ProfessionalEdition~31bf3856ad364e35~amd64~~10.0.22000.1.mum
...
```

### Language Pack Installation

Once the edition package is installed, all future installations are done with DISM. Language pack installation is done using DISM package installation:

```
dism /image:<Offline Image Path> /add-package /packagepath:<Language Pack Path> 
```

### Edition-Specific Unattend

Edition-specific settings are then applied from an unattend file. This unattend file is installed by default to `\Windows\servicing\Editions\<Edition Shortname>.xml` and copied after Edition Package installation to `\Windows\<Edition Shortname>.xml`. It is then installed using DISM unattend installation:

```
dism /image:<Offline Image Path> /apply-unattend:<Path to Offline Image>\Windows\<Edition Shortname>.xml
```

### Locale Settings

The image's system language is then set:

```
dism /image:<Offline Image Path> /set-allintl:<Language Code>
```

This updates the offline image's registry values in the following locations:
 - `SYSTEM\ControlSet001\Control\Nls\CodePage`
 - `SYSTEM\ControlSet001\Control\Nls\Locale`
 - `SYSTEM\ControlSet001\Control\Nls\Language`
 - `SYSTEM\ControlSet001\Control\CommonGlobUserSettings\Control Panel\International`

### Sysprep

In newer Windows versions, the Sysprep generalization state must be added after servicing. This is done by creating the file `\Windows\Setup\State\State.ini` with the following contents:

```
[State]
ImageState=IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE
```

The following registry modifications must also be made:
 - `SOFTWARE\Microsoft\Windows\CurrentVersion\Setup\State!ImageState` set to `IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE` (`REG_SZ`)
 - `SOFTWARE\Microsoft\Windows\CurrentVersion\Setup\ImageServicingData!ImageState` set to `IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE` (`REG_SZ`)
 - `SYSTEM\Setup\Status\SysprepStatus!GeneralizationState` set to `4` (`REG_DWORD`)

The open-source project [explodeSxS](https://github.com/asdcorp/explodeSxS) provides an implementation of the staged image deployment process.

## Unstaged Image Deployment

Rarely, some versions of Windows are shipped unstaged, meaning that no particular edition package is installed in the installation image and the foundation package is deployed without being marked as installed. Instead, the full list of editions built at compile time are available to choose from during initial setup, either in an interactive menu or by entering an appropriate product key. Then, setup will manually install the edition package and language pack during offline image deployment. Some examples of known unstaged builds are checked builds of Windows 7, [Windows 10 build 14357.1000](https://betawiki.net/wiki/Windows_10_build_14357.1000), and [Windows Server 2016 build 10586 (th2_srv1_nano_dev2)](https://betawiki.net/wiki/Windows_Server_2016_build_10586_(th2_srv1_nano_dev2)).

### Packages Directory

The packages directory, located at `\packages` is structured similarly to the `WinSxS` directory, except that the only sub-directories present are for component folders, and all manifests for packages, components, and deployments are stored at the root of the folder. Additionally, the following files must be present at the root of the folder:
 - The package manifests and associated catalogs for all available editions must be named after their editions' shortname (ex. `Professional.mum`, `Professional.cat`, etc.)
 - The edition unattend file must also be named after the edition shortname (ex. `Professional.xml`)
 - The appropriate edition matrix and upgrade matrix, named `EditionMatrix.xml` and `UpgradeMatrix.xml` respectively
 - `pkeyconfig.xrm-ms` containing product key information for the available editions
 - `pidgenx.dll`

Note that only files related to the dependencies of edition packages are present in the packages directory.

### WIM Metadata

For setup to be configured for installing an unstaged image, the appropriate metadata must be set in `install.wim`:
 - `FLAGS` must be set to `Windows Foundation`
 - `DESCRIPTION` must end with the string `EDITIONS:` followed by a comma-separated list of available editions (ex. `EDITIONS:CORE,PROFESSIONAL,ENTERPRISE`)

These values can be set using [wimlib](https://wimlib.net).

### Language Packs

Since language packs are not installed in the image, they are instead packaged into cabinet files and stored in the setup ISO. Prior to cabinet compression, the package manifest for the language pack must be named `update.mum`, and the associated catalog file must be named `update.cat`. Then, the language pack is saved in `langpacks/<language code>/lp.cab` in the setup ISO.

To support installation of language packs from the ISO, the file `sources\lang.ini` must be updated in both the ISO and in `sources\boot.wim` within the ISO. In this file, under the section `[Available Languages]` the flag for each language must be set to `1`. Example contents:

```
[Available UI Languages] 
en-US = 1 
qps-ploc = 1 
 
[Fallback Languages] 
en-US = en-us 
qps-ploc = en-us 
```

Note that if an appropriate EULA file is not available for a chosen edition and language, it cannot be installed with Windows Setup.

## Sources

 - [stageSxS - asdcorp](https://github.com/asdcorp/stageSxS)
 - [Leak prevention in Microsoft Windows - BetaWiki](https://betawiki.net/wiki/Leak_prevention_in_Microsoft_Windows)
 - [Windows 10 build 14357.1000 - BetaWiki](https://betawiki.net/wiki/Windows_10_build_14357.1000)
 - [Windows Server 2016 build 10586 (th2_srv1_nano_dev2) - BetaWiki](https://betawiki.net/wiki/Windows_Server_2016_build_10586_(th2_srv1_nano_dev2))
