﻿# Nerdbank.GitVersioning

[![Build status](https://ci.appveyor.com/api/projects/status/94wwito7ifg57d65?svg=true)](https://ci.appveyor.com/project/AArnott/nerdbank-gitversioning)
[![NuGet package](https://img.shields.io/nuget/v/Nerdbank.GitVersioning.svg)](https://nuget.org/packages/Nerdbank.GitVersioning)
[![Join the chat at https://gitter.im/AArnott/Nerdbank.GitVersioning](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/AArnott/Nerdbank.GitVersioning?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

## Overview

This package adds precise, semver-compatible git commit information
to every assembly, VSIX, and NuGet package.
It implicitly supports all cloud build services and CI server software
because it simply uses git itself and integrates naturally in MSBuild. 

What sets this package apart from other git-based versioning projects is:

1. Prioritize absolute build reproducibility. Every single commit can be built and produce a unique version.
2. No dependency on tags. Tags can be added to existing commits at any time. Clones may not fetch tags. No dependency on tags means better build reproducibility.
3. No dependency on branch names. Branches come and go, and a commit may belong to any number of branches. Regardless of the branch HEAD may be attached to, the build should be identical.
4. The computed version information is based on an author-defined major.minor version and an optional unstable tag, plus a shortened git commit ID.

## Installation and Configuration

After installing this NuGet package, you may need to configure the version generation logic
in order for it to work properly.

With NuGet 2.x, the configuration is handled automatically via the tools\Install.ps1 script.
For NuGet 3.x, you can run the script tools\Create-VersionFile.ps1 to help you create the
version.json file and remove the old assembly attributes.

The scripts will look for the presence of a version.json or version.txt file.
If one already exists, nothing happens. If the version file does not exist,
the script looks in your project for the Properties\AssemblyInfo.cs file and attempts
to read the Major.Minor version number from
the AssemblyVersion attribute. It then generates a version.json file using the Major.Minor
that was parsed so that your assembly will build with the same AssemblyVersion as before,
which preserves backwards compatibility. Finally, it will remove the various version-related
assembly attributes from AssemblyInfo.cs.

If you did not use the scripts to configure the package, you may find that you get a
compilation failure because of multiple definitions of certain attributes such as
`AssemblyVersionAttribute`.
You should resolve these compilation errors by removing these attributes from your own
source code, as commonly found in your `Properties\AssemblyInfo.cs` file:

```csharp
[assembly: AssemblyVersion("1.0.0.0")]
[assembly: AssemblyFileVersion("1.0.0.0")]
[assembly: AssemblyInformationalVersion("1.0.0-dev")]
```

This NuGet package creates these attributes at build time based on version information
found in your `version.json` or `version.txt` file and your git repo's HEAD position.

Note: After first installing the package, you need to commit the version file so that
it will be picked up during the build's version generation. If you build prior to committing,
the version number produced will be 0.0.x.

### version.json or version.txt file

You must define a version.json or version.txt file in your project directory or some ancestor of it.

When the package is installed, a version.json file is created in your project directory
(for NuGet 2.x clients). This ensures backwards compatibility where the installation of
this package will not cause the assembly version of the project to change. If you would
like the same version number to be applied to all projects in the repo, then you may move
the file to the root directory of your git repo.

Here is the content of a sample version.json file you may start with (do not indent):

    { "version": "1.0.0-beta" }

Or the (deprecated) version.txt file you may start with (do not indent):

    1.0.0
    -beta

### version.json file format

The content of the version.json file is a JSON serialized object with these properties:

```js
{
  "version": "x.y.z-prerelease", // required
  "assemblyVersion": "x.y", // optional. Use when x.y for AssemblyVersionAttribute differs from the default version property.
  "buildNumberOffset": "zOffset", // optional. Use when you need to add/subtract a fixed value from the computed build number.
  "publicReleaseRefSpec": [
    "^refs/heads/master$", // we release out of master
    "^refs/tags/v\\d\\.\\d" // we also release tags starting with vN.N
  ]
}
```

The `x` and `y` variables are for your use to specify a version that is meaningful
to your customers. Consider using [semantic versioning][semver] for guidance.
The `z` variable should be 0.

The optional -prerelease tag allows you to indicate that you are building prerelease software.

The `publicReleaseRefSpec` field causes builds out of certain branches or tags to automatically default the `PublicRelease` property to `true`, making it convenient to build releases out of these refs without the `-gCOMMITID` suffix in your version specs.

When editing the version.json file in an editor that supports JSON schema files, you may
point the editor to the `tools\version.schema.json` file in this package for added
assistance and validation or add this as the first field in your version.json file:

```json
"$schema": "https://raw.githubusercontent.com/AArnott/Nerdbank.GitVersioning/master/src/NerdBank.GitVersioning/version.schema.json",
```

### version.txt file format (obsolete)

The content of the version.txt file is a semver-compliant version in this format:

    x.y.z-prerelease

The `x` and `y` variables are for your use to specify a version that is meaningful
to your customers. Consider using [semantic versioning][semver] for guidance.
The `z` variable should be 0.

The optional -prerelease tag may appear on the second line. This tag allows you
to indicate that you are building prerelease software. 

### Apply to VSIX versions

Besides simply installing the NuGet package into your VSIX-generating project,
you should also open the `source.extension.vsixmanifest` file in a code editor
and set the `PackageManifest/Metadata/Identity/@Version` attribute to this
value: `|YourProjectName;GetBuildVersion|` where `YourProjectName` is
obviously replaced with the actual project name (without extension) of your
VSIX project.

### Apply to NuProj built NuPkg versions

To automatically apply versioning to your NuProj projects, simply install the Nerdbank.GitVersioning package to your project. 
You are encouraged to *remove* any definition of a Version property:

```xml
<Version>1.0.0-removeThisWholeLine</Version>
```

Installing a NuGet package into a NuProj must be done via project.json, since it doesn't support the deprecated packages.config format.

## Build

By default, each build of a Nuget package will include the git commit ID.
When you are preparing a release (whether a stable or unstable prerelease),
you may build setting the `PublicRelease` global property to `true`
in order to avoid the git commit ID being included in the NuGet package version.

From the command line, building a release version might look like this:

    msbuild /p:PublicRelease=true

Note you may consider passing this switch to any build that occurs in the
branch that you publish released NuGet packages from. 
You should only build with this property set from one release branch per
major.minor version to avoid the risk of producing multiple unique NuGet
packages with a colliding version spec.

### Shallow clones

Since git history is a crucial part of calculating a git height based version, it is important
that you *not* build using a shallow clone or the computed version may be incorrect.
Some cloud build systems such as AppVeyor or TeamCity allow you to use shallow cloning or
suppress the .git folder entirely. You should not use these options when using git-based versioning.

## Where and how versions are calculated and applied

This package calculates the version based on a combination of the version file,
the git 'height' of the version, and the git commit ID.

### Assembly version generation

During the build it adds source code such as this to your compilation:

```csharp
[assembly: System.Reflection.AssemblyVersion("1.0")]
[assembly: System.Reflection.AssemblyFileVersion("1.0.24.15136")]
[assembly: System.Reflection.AssemblyInformationalVersion("1.0.24.15136-alpha+g9a7eb6c819")]
```

* The first and second integer components of the versions above come from the 
version file.
* The third integer component of the version here is the height of your git history up to
that point, such that it reliably increases with each release.
* The fourth component (when present) is the first two bytes of the git commit ID, encoded as an integer. This number will appear essentially random, and is not useful in sorting versions. It is useful when you have two branches in git history that have exactly the same major.minor.height version information in order to distinguish which commit it is.
* The -alpha tag also comes from the version file and indicates this is an
unstable version.
* The -g9a7eb6c819 tag is the concatenation of -g and the git commit ID that was built.

This class is also injected into your project at build time:

```csharp
internal sealed partial class ThisAssembly {
    internal const string AssemblyVersion = "1.0";
    internal const string AssemblyFileVersion = "1.0.24.15136";
    internal const string AssemblyInformationalVersion = "1.0.24.15136-alpha+g9a7eb6c819";
}
```

This allows you to actually write source code that can refer to the exact build
number your assembly will be assigned.

### NuGet package version generation

Given the same settings as used in the discussion above, a NuGet package may be
assigned this version: 

    1.0.24-alpha-g9a7eb6c819

When built with the `/p:PublicRelease=true` switch, the NuGet version becomes:

    1.0.24-alpha

### Visual Studio Team Services builds

When building online with visualstudio.com, two VSTS variables are set during the build:

| VSTS variable | MSBuild property | Sample value
| --- | --- | --- |
| GitAssemblyInformationalVersion | AssemblyInformationalVersion | 1.3.1+g15e1898f47
| GitBuildVersion | BuildVersion | 1.3.1.57621

This means you can use these variables in subsequent steps in your cloud build such as publishing artifacts, so that your richer version information can be expressed in the publish location or artifact name.

## Frequently asked questions

### What is 'git height'?

Git 'height' is the count of commits in the longest path between HEAD (the code you're building)
and some origin point. In this case the origin is the commit that set the major.minor version number
to the values found in HEAD.

For example, if the version specified at HEAD is 3.4 and the longest path in git history from HEAD
to where the version file was changed to 3.4 is 15 steps, then the git height is "15".

### Why is the git height used for the PATCH version component for public releases?

The git commit ID does not represent an alphanumerically sortable identifier
in semver, and thus delivers a poor package update experience for NuGet package
consumers. Incrementing the PATCH with each public release ensures that users
who want to update to your latest NuGet package will reliably get the latest
version. 

The git height is guaranteed to always increase with each release within a given major.minor version,
assuming that each release builds on a previous release. And the height automatically resets when
the major or minor version numbers are incremented, which is also typically what you want.

### Why isn't the git commit ID included for public releases?

It could be, but the git height serves as a pseudo-identifier already and the
git commit id would just make it harder for users to type in the version
number if they ever had to.

Note that the git commit ID is *always* included in the 
`AssemblyInformationVersionAttribute` so one can always match a binary to the
exact version of source code that produced it.

### How do I translate from a version to a git commit and vice versa?

A pair of Powershell scripts are included in the Nerdbank.GitVersioning NuGet package
that can help you to translate between the two representations. 

    tools\Get-CommitId.ps1
    tools\Get-Version.ps1

`Get-CommitId.ps1` takes a version and print out the matching commit (or possible commits, in the exceptionally rare event of a collision).
`Get-Version.ps1` prints out the version information for the git commit current at HEAD.

 [semver]: http://semver.org
