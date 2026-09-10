---
title: "Interpreters on iOS and Mac Catalyst"
description: Learn how .NET MAUI apps on iOS and Mac Catalyst use the Mono and CoreCLR interpreters.
ms.date: 09/11/2026
---

# Interpreters on iOS and Mac Catalyst

.NET Multi-Platform App UI (.NET MAUI) uses different interpreter models for iOS and Mac Catalyst depending on the target .NET version. .NET 10 and earlier apps use the Mono interpreter with Mono AOT when needed. .NET 11+ apps use CoreCLR with ReadyToRun and the CoreCLR interpreter because iOS and Mac Catalyst don't permit JIT-generated code.

NativeAOT is a separate publish model. It has no JIT compiler or interpreter, and the `UseInterpreter` and `MtouchInterpreter` properties have no effect when NativeAOT is used. For more information, see [Native AOT deployment](~/deployment/nativeaot.md).

::: moniker range="<=net-maui-10.0"

## Mono interpreter

When you compile a .NET MAUI app for iOS or Mac Catalyst with Mono, Mono AOT compiles managed code into native code. iOS devices, ARM64 iOS simulators, and ARM64 Mac Catalyst apps restrict JIT-generated code, so Mono uses AOT in those scenarios. An iOS simulator or Mac Catalyst app can use JIT compilation when its architecture permits it.

Mono AOT provides faster startup and other performance optimizations, but it restricts some features:

- Generic support is limited. Not every possible generic instantiation can be determined at compile time.
- Dynamic code generation is restricted in Mono AOT builds. `System.Reflection.Emit` and `System.Runtime.Remoting` aren't supported, and some uses of the C# [dynamic](/dotnet/csharp/advanced-topics/interop/using-type-dynamic) type aren't permitted.

When a Mono AOT restriction occurs, a `System.ExecutionEngineException` is thrown with a message of "Attempting to JIT compile method while running in aot-only mode".

The Mono interpreter can execute selected IL at runtime without generating native code dynamically, while Mono AOT compiles the rest. It can accommodate some patterns that Mono full AOT can't compile, but it doesn't make arbitrary runtime code generation or `System.Reflection.Emit` generally supported on iOS or Mac Catalyst.

The interpreter is enabled by default for .NET MAUI iOS and Mac Catalyst `Debug` builds that use Mono, and can be enabled for `Release` builds. However, there are potential drawbacks to using the interpreter in a production app:

- While the app size usually shrinks significantly when the interpreter is enabled, in certain cases the app size can increase.
- App execution speed is slower because interpreted code runs more slowly than AOT-compiled code. The reduction can range from unmeasurable to unacceptable, so performance testing should be performed.
- Native stack traces in crash reports become less useful because they contain generic frames from the interpreter that don't identify the executing code. Managed stack traces don't change.

> [!TIP]
> If your .NET MAUI iOS app or ARM64-based Mac Catalyst app works correctly as a `Debug` build but then crashes as a `Release` build, try enabling the Mono interpreter for the `Release` build. Your app, or one of its libraries, might use a feature that requires the interpreter.

## Enable the Mono interpreter

`UseInterpreter` is a Mono control. It affects only apps that use Mono, including .NET MAUI iOS and Mac Catalyst apps targeting .NET 10 or earlier. The Mono interpreter can be enabled in iOS `Release` builds by setting the `$(UseInterpreter)` MSBuild property to `true` in your project file:

```xml
<PropertyGroup Condition="$(TargetFramework.Contains('-ios')) and '$(Configuration)' == 'Release'">
    <UseInterpreter>true</UseInterpreter>
</PropertyGroup>
```

The Mono interpreter can also be enabled for Mac Catalyst `Release` builds on ARM64:

```xml
<PropertyGroup Condition="'$(RuntimeIdentifier)' == 'maccatalyst-arm64' and '$(Configuration)' == 'Release'">
    <UseInterpreter>true</UseInterpreter>
</PropertyGroup>
```

On iOS and Mac Catalyst, the Mono interpreter can also be configured with the `$(MtouchInterpreter)` MSBuild property. If both properties are set, `MtouchInterpreter` takes precedence over `UseInterpreter`. The property optionally takes a comma-separated list of assemblies to be interpreted. `all` specifies all assemblies, and an assembly prefixed with a minus sign is AOT-compiled. This enables you to:

- Interpret all assemblies by specifying `all` or AOT compile everything by specifying `-all`.
- Interpret individual assemblies by specifying **MyAssembly** or AOT compile individual assemblies by specifying **-MyAssembly**.
- Mix and match to interpret some assemblies and AOT compile other assemblies.

The following example shows how to interpret all assemblies except **System.Xml.dll**:

```xml
<PropertyGroup Condition="$(TargetFramework.Contains('-ios')) and '$(Configuration)' == 'Release'">
    <!-- Interpret everything, except System.Xml.dll -->
    <MtouchInterpreter>all,-System.Xml</MtouchInterpreter>
</PropertyGroup>
```

The following example shows how to AOT compile all assemblies except **System.Numerics.dll**:

```xml
<PropertyGroup Condition="$(TargetFramework.Contains('-ios')) and '$(Configuration)' == 'Release'">
    <!-- AOT everything, except System.Numerics.dll, which will be interpreted -->
    <MtouchInterpreter>-all,System.Numerics</MtouchInterpreter>
</PropertyGroup>
```

Alternatively, use the following example to AOT-compile all assemblies:

```xml
<PropertyGroup Condition="$(TargetFramework.Contains('-ios')) and '$(Configuration)' == 'Release'">
    <!-- AOT-compile all assemblies -->
    <MtouchInterpreter>-all</MtouchInterpreter>
</PropertyGroup>
```

Another common scenario where the Mono interpreter is required is a .NET MAUI Mac Catalyst app running on ARM64 that throws an exception on launch. This launch exception can often be fixed by interpreting the assembly that contains the affected code:

```xml
<PropertyGroup Condition="'$(RuntimeIdentifier)' == 'maccatalyst-arm64' and '$(Configuration)' == 'Release'">
    <!-- AOT-compile all assemblies except MyAssembly, which is interpreted -->
    <MtouchInterpreter>-all,MyAssembly</MtouchInterpreter>
</PropertyGroup>
```

> [!IMPORTANT]
> A stack frame executed by the interpreter won't provide useful information. However, because the interpreter can be disabled on a per-assembly basis, it's possible to have stack frames from some assemblies accurately depicted in crash reports.

::: moniker-end

::: moniker range=">=net-maui-11.0"

## CoreCLR interpreter

In .NET 11+, CoreCLR is the runtime for .NET MAUI apps on iOS and Mac Catalyst. .NET 10 doesn't provide CoreCLR for iOS or Mac Catalyst.

On iOS and Mac Catalyst, composite partial ReadyToRun is used in `Debug` builds and composite full ReadyToRun is used in `Release` builds. The CoreCLR interpreter is always enabled and executes code that isn't precompiled because iOS and Mac Catalyst don't permit JIT compilation.

`Partial` and `full` describe the ReadyToRun compilation mode. Full ReadyToRun doesn't guarantee that every method is precompiled; the CoreCLR interpreter executes methods that aren't precompiled. This behavior is part of the runtime and doesn't require an interpreter MSBuild property.

`UseInterpreter` and `MtouchInterpreter` are Mono controls for .NET 10 and earlier. They don't configure the CoreCLR interpreter and shouldn't be added to .NET 11+ iOS or Mac Catalyst project files to try to enable or disable it.

NativeAOT remains a separate publish model. It has no JIT compiler or interpreter, and `UseInterpreter` and `MtouchInterpreter` have no effect when NativeAOT is used. For more information, see [Native AOT deployment](~/deployment/nativeaot.md).

::: moniker-end
