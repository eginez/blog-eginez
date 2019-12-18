+++
title = "Building the Q# compiler in mac os"
description = ""
tags = [ "q#", "quantum computing" ]
date = "2019-12-16"
categories = [ "Development", "q#", "quantum computing" ]
menu = "main"
+++
 Earlier this year [Microsoft open sourced](https://cloudblogs.microsoft.com/quantum/2019/07/11/microsoft-quantum-oss-available-github/) a number of their quantum programming tools including the compiler for the qsharp(q#) quantum programming language. I really enjoy reading and learning from open source software, so the news were really exciting for me since it allowed to look at the internals of qsharp.

 The following post summarizes some of my experiences working and building the qsharp compiler code base as member of the open source comunity.

 My main dev environment is macOS so lots of the instructions here will be targeted for it, however if your are running a *nix environment is pretty much same.

## Prerequisites 
Here is the list of things you are going to need to build the qsharp compiler.

- [Dotnet](https://dotnet.microsoft.com/download/dotnet-core/thank-you/sdk-3.1.100-macos-x64-installer)

    ```console
    $> dotnet --version
    3.0.101
    ```
- [Power shell](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-macos?view=powershell-6)

    ```bash
    $> brew cask install powershell
    ...
    $> pwsh --version
    PowerShell 6.2.3
    ```

- [Qsharp's source code](https://github.com/microsoft/qsharp-compiler)

    ```console
    $> git clone https://github.com/microsoft/qsharp-compiler.git
    ```
- [bsondump](https://github.com/mongodb/mongo-tools) and [jq](https://stedolan.github.io/jq/download/) To visualize the compiler output. I found it surprinsing that there was no easy way to install a bson->json converter so I decided to build a mongodb tool from the source above (FYI you'll need the go tool chain for that)

- A text editor of your choice. 
    Here life starts getting a little complicated for macOS users. Vim, usually my editor of choice, sadly struggled to support symbolic navigation, specially cross-project. Visual Studio Code, fine choice for most projects, and an excellent choice for C# projects, did not have a good F# plugin.  I settled for [Visual Studio for macOS](https://visualstudio.microsoft.com/vs/community/), even though it lacks some pretty key text editor plugins (vim style emulation, for example). On the plus side I found that Visual Studio offered the best experience when navigating a mix C#, F# code base.



##  Building

- Move to your the qsharpcompiler repository and execute the boostraping script

```console
$> cd qsharp-compiler
$> pwsh bootstrap.ps1
...
Enable telemetry: false
Adding Newtonsoft.Json.Bson
Adding Microsoft.VisualStudio.LanguageServer.Protocol
Adding FSharp.Core
Adding System.ValueTuple
Adding Newtonsoft.Json
Adding System.Collections.Immutable
Adding Markdig
Adding YamlDotNet
Adding FParsec
Adding Microsoft.CodeAnalysis.CSharp

dependency
----------
{dependency, dependency, dependency, dependency…}
```

- Build the compiler project

```console
$> dotnet build QsCompiler.sln
Microsoft (R) Build Engine version 16.3.2+e481bbf88 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.
....

```
Make sure you have sync'd your repo with the lastest qsharp repository. I fixed [a few minor issues](https://github.com/microsoft/qsharp-compiler/commit/1ca68a9c43a9d6c05f7fb4cb1ce46295e70d6091) with the build script in macOS and they relatively new commits.


If everything worked fine, you now should be able to call the qsharp compiler from its command line tool.
```console
$> src/QsCompiler/CommandLineTool/bin/Debug/netcoreapp3.0/qsc
Microsoft Q# compiler command line tool. 1.0.0
© Microsoft Corporation. All rights reserved.

ERROR(S):
  No verb selected.

  build       Builds a compilation unit to run on the Q# quantum simulation framework.

  diagnose    Generates intermediate representations of the code to help diagnose issues.

  format      Generates formatted Q# code.

  help        Display more information on a specific command.

  version     Display version information.

$> src/QsCompiler/CommandLineTool/bin/Debug/netcoreapp3.0/qsc version
Microsoft Q# compiler command line tool. 1.0.0
```


## Running it
With a built compiler we are ready to take for a spin. The command line tool of the qsharp compiler, provides a few options to specify input. For example we can pass a snippet of q# code like so:

```console
$> src/QsCompiler/CommandLineTool/bin/Debug/netcoreapp3.0/qsc build -s "var uf = 2;"

Error QS3034:
File: /Users/eginez/repos/qsc/__CODE_SNIPPET__.qs
Position: [ln 1, cn 5]
Unexpected code fragment.

Error QS3035:
File: /Users/eginez/repos/qsc/__CODE_SNIPPET__.qs
Position: [ln 1, cn 1]
An expression used as a statement must be a call expression.

____________________________________________

Q# compilation failed: 2 errors, 0 warnings


$> src/QsCompiler/CommandLineTool/bin/Debug/netcoreapp3.0/qsc build -s "mutable uf = 2;"
____________________________________________

Q#: Success! (0 errors, 0 warnings)
```

Of course you probably want to save the source code to a file and pass that file to the compiler, for that you can do
```console
$> cat sample.qs
namespace blank {
        operation OneQ () : Unit {
        using(q = Qubit()) {
        }
    }
}
$> src/QsCompiler/CommandLineTool/bin/Debug/netcoreapp3.0/qsc build -i sample.qs -o out/
____________________________________________

Q#: Success! (0 errors, 0 warnings)

$>ls out
twnlm5ty.bson twnlm5ty.dll
```

The output of the compiler is both a bson file and dll with the bson in it. Finally you can look at the output of the compiler with the `bsondump` tools
```json
$> $GOPATH/mongodb/mongo-tools/bin/bsondump twnlm5ty.bson|jq
...
"Statements": [
                          {
                            "Statement": {
                              "Case": "QsQubitScope",
                              "Fields": [
                                {
                                  "Kind": {
                                    "Case": "Allocate"
                                  },
                                  "Binding": {
                                    "Kind": {
                                      "Case": "ImmutableBinding"
                                    },
                                    "Lhs": {
                                      "Case": "VariableName",
                                      "Fields": [
                                        "q"
                                      ]
                                    },
                                    "Rhs": {
                                      "Case": "SingleQubitAllocation"
                                    }
                                  },
                                  "Body": {
                                    "Statements": [],
                                    "KnownSymbols": {
                                      "Variables": [
                                        {
                                          "VariableName": "q",
                                          "Type": {
                                            "Case": "Qubit"
                                          },
                                          "InferredInformation": {
                                            "IsMutable": false,
                                            "HasLocalQuantumDependency": false
                                          },
                                          "Position": {
                                            "Case": "Value",
                                            "Fields": [
                                              {
                                                "Item1": {
                                                  "$numberInt": "1"
                                                },
                                                "Item2": {
                                                  "$numberInt": "8"
                                                }
                                              }
                                            ]
                                          },
                                          "Range": {
                                            "Item1": {
                                              "Line": {
                                                "$numberInt": "1"
                                              },
                                              "Column": {
                                                "$numberInt": "7"
                                              }
                                            },
                                            "Item2": {
                                              "Line": {
                                                "$numberInt": "1"
                                              },
                                              "Column": {
                                                "$numberInt": "8"
                                              }
                                            }
                                          }
                                        }
                                      ],
....
```
As you can see the compiler outputs the whole AST and metadata in json format.

The compiler code itself is split into a number of packages, some interesting packages to look at are: `Optimizations`, `SyntaxProcessor`, `Compilation Manager`. In addition the compiler project has a number of targets, for example: `CommandLineTool`, `LanguageServer` and the `Tests` targets. The `CommandLineTool` and the `Test` targets are the most relevent and as we saw built with no problems, the `Test` target still has a build problem on macOS I haven't tracked down yet.

