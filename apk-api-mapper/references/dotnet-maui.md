# .NET MAUI Assembly Analysis

Read this when the APK contains .NET managed code. Look for `Microsoft.Maui.*` assemblies or type
references to confirm MAUI; Mono/.NET runtime libraries alone also occur in other .NET Android apps.
Follow the same managed analysis path for those apps without labeling them MAUI without evidence.

## Recover the assemblies

Inspect the APK ZIP entries for loose DLLs under `assemblies/`, older `assemblies*.blob` stores,
or stores under `lib/<abi>/` such as `libassemblies.<abi>.blob.so` and `libassembly-store.so`.
Packaging varies by runtime/version; see the [.NET Android assembly-store format](https://github.com/dotnet/android/blob/main/Documentation/project-docs/AssemblyStores.md).

Extract into a new `artifacts/apk/{timestamp}/dotnet/` directory, preserving the APK and original
payloads. For packed, compressed, or ELF-wrapped assemblies, use an extractor compatible with that
store version to recover managed DLLs first. Renaming a `.blob` or `.so` to `.dll` is insufficient.
Keep dependencies from the same ABI/build together and record the APK entry, extraction tool, and
assembly names. If recovery is unsupported, document the blocked API areas and the needed extractor
or unpacked assemblies; Java wrappers cannot substitute for the managed code.

## Decompile on the analysis host

Use **dnSpy CLI (`dnSpy.Console.exe`) on Windows** and **ILSpy CLI (`ilspycmd`) on macOS/Linux**.
Check that the CLI and its required runtime are available; the desktop GUI alone is insufficient.
Inspect local CLI usage before running commands and follow the skill's missing-tool policy.

Examples below use placeholder paths; replace `{timestamp}` and `App.dll` with the actual values.
Start with app-owned assemblies and supply the recovered dependency directory for resolution.

Windows PowerShell, using [dnSpy's console options](https://github.com/dnSpyEx/dnSpy/blob/master/dnSpy/dnSpy.Console/Program.cs):

```powershell
& "C:\tools\dnSpy\dnSpy.Console.exe" --asm-path "artifacts/apk/{timestamp}/dotnet/assemblies" -o "artifacts/apk/{timestamp}/dotnet/csharp/App" "artifacts/apk/{timestamp}/dotnet/assemblies/App.dll"
```

macOS/Linux, using [ILSpy's CLI options](https://github.com/icsharpcode/ILSpy/tree/master/ICSharpCode.ILSpyCmd):

```bash
ilspycmd -p -r "artifacts/apk/{timestamp}/dotnet/assemblies" -o "artifacts/apk/{timestamp}/dotnet/csharp/App" "artifacts/apk/{timestamp}/dotnet/assemblies/App.dll"
```

Use a separate output directory per assembly. Inspect the generated source without building or
executing it. Trace `HttpClient`, `HttpRequestMessage`, `SendAsync`, `DelegatingHandler`, Refit,
RestSharp, serializer settings, dependency-injection registrations, token storage, and signing.
Follow callers and shared handlers through app-owned services and relevant dependencies.

Check ambiguous decompiled C# against managed IL, including async state-machine `MoveNext` methods.
Record the assembly, fully qualified type, method/signature, and metadata token or IL offset where
needed. Follow Java/JNI bridges in JADX only when a request crosses that boundary.

Mono AOT images can coexist with managed IL; their presence alone does not block decompilation.
If the required method bodies are absent, stripped, or only available as native code, record that
limit rather than inferring them from wrappers. See [MAUI runtime and APK layouts](https://learn.microsoft.com/en-us/dotnet/maui/deployment/runtimes-compilation).
