---
layout: post
title: "W/L TechNotes 1: making RoboVM cross-platform"
tags: ["linux windows", "technote"]
---
This post start series of technical notes that would help developers to understand changes were done to RoboVM. Table of content:

1. making RoboVM cross-platform
2. Codesign
3. Linker and tools
4. actool and
5. xib2nib
6. Pain and tears :)

<!-- more -->
# making RoboVM cross-platform
Between hit on 'Run' button and deploy binary to device happens many operations:
- Java sources are compiled into class files by Java compiler;
- RoboVM translates Java byte code into LLVM IR code and compiles it into native code with `librobovm-llvm`;
- All object files being linked into executable with Apple LD64 linker;
- Binary being code signed;
- All assets are being processed with Apple toolchain (.xib, .xcassets);
- .dsym folder is generated with Apple toolchain utility;
- also lot of Apple utilities being called during the build (lipo, ottol, strip)
- once complete it is being deployed to device using `librobovm-libimobiledevice`

Originally RoboVM designed to be run exclusively MacOS. In the list above all lines that include `Apple` has dependency to XCode tools. Most of these calls are implemented in [ToolchainUtil.java](https://github.com/MobiVM/robovm/blob/d3dd77507d25e7106c3b62adf2f18d507c5308b1/compiler/compiler/src/main/java/org/robovm/compiler/util/ToolchainUtil.java). This class is widely used in compiler code and this has own benefits in turning it into facade.

The plan:
- turn `ToolchainUtil` into platform independent facade
- define interface OS dependent implementation
- move existing code from `ToolchainUtil` to [DarwinToolchainUtil.java](https://github.com/dkimitsa/robovm/blob/linuxwindows/compiler/compiler/src/main/java/org/robovm/compiler/util/platforms/darwin/DarwinToolchainUtil.java) and implement Darwin toolchain interface in [DarwinToolchain.java](https://github.com/dkimitsa/robovm/blob/linuxwindows/compiler/compiler/src/main/java/org/robovm/compiler/util/platforms/darwin/DarwinToolchain.java):
- linker invocation is platform depended and to be moved behind `ToolchainUtil` facade;
- also there is bunch of call for Apple tools from compilation code, all these has to be moved behind facade;

Implementing the plan -- following facade api was defined for platform dependent toolchain implementation. Facade `ToolchainUtil` just implements save static api wrappers:
```java
class ToolchainUtil.Contract {
    protected String findXcodePath() throws IOException;
    protected boolean isXcodeInstalled();
    protected boolean isToolchainInstalled();
    protected void pngcrush(Config config, File inFile, File outFile) throws IOException;
    protected void textureatlas(Config config, File inDir, File outDir) throws IOException;
    protected void actool(Config config, File partialInfoPlist, File inDir, File outDir) throws IOException;
    protected void ibtool(Config config, File partialInfoPlist, File inFile, File outFile) throws IOException;
    protected void compileStrings(Config config, File inFile, File outFile) throws IOException;
    protected String otool(File file) throws IOException;
    protected void lipo(Config config, File outFile, List<File> inFiles) throws IOException;
    protected void lipoRemoveArchs(Config config, File file, File inFile, Arch ...archs) throws IOException;
    protected String lipoInfo(Config config, File inFile) throws IOException;
    protected String file(File file) throws IOException;
    protected void packageApplication(Config config, File appDir, File outFile) throws IOException;
    protected void link(Config config, List<String> args, List<File> objectFiles, List<String> libs, File outFile) throws IOException;
    protected void codesign(Config config, SigningIdentity identity, File entitlementsPList, boolean preserveMetadata, boolean verbose,
                         boolean allocate, File target) throws IOException;
    protected File getProvisioningProfileDir();
    protected List<DeviceType> listSimulatorDeviceTypes();
    protected List<SigningIdentity> listSigningIdentity();
    protected void dsymutil(Logger logger, File dsymDir, File exePath) throws IOException;
    protected void strip(Config config, File exePath) throws IOException;
}
```

Once `ToolchainUtil` loaded it creates concrete implementation for particular host OS:
```java
private final static Contract impl;
static {
    SystemInfo systemInfo = getSystemInfo();
    if (systemInfo.os == SystemInfo.OSInfo.macosx)
        impl = new DarwinToolchain();
    else if (systemInfo.os == SystemInfo.OSInfo.macosxlinux)
        impl = ExternalCommonToolchain.DarwinLinux();
    else if (systemInfo.os == SystemInfo.OSInfo.windows)
        impl = ExternalCommonToolchain.Windows();
    else if (systemInfo.os == SystemInfo.OSInfo.linux)
        impl = ExternalCommonToolchain.Linux();
    else
        impl = new Contract("Unsupported OS - " + systemInfo.osName);
}
```

To support new OS it would require to add corresponding concrete implementation;
In case of Linux/Windows there is common implementation [ExternalCommonToolchain.java](https://github.com/dkimitsa/robovm/blob/linuxwindows/compiler/compiler/src/main/java/org/robovm/compiler/util/platforms/external/ExternalCommonToolchain.java) as they share same tools just compiled for different platforms. These platforms have a little bit different set of tools and they are not distributed with RoboVM package and has to be manually downloaded by user. Also `librobovm-llvm` and `librobovm-libimobiledevice` are being embedded into RoboVM distributable package only for Darwin platform (linux one removed) and for Linux/Windows case these are expected to be found in same location as toolset. Location defined as following: `~/.robovm/platform/${platform_name}`, e.g. for Linux64 `~/.robovm/platform/linux-x86_64`. refer [setup guide]({{ site.baseurl }}{% post_url 2017-12-12-robovm-now-linux-windows %}) for details.

This changes the way how `librobovm-llvm` and `librobovm-libimobiledevice` loads native library. If for `Darwin` it stays as before (from resources) Linux/Windows have to provide path for these. There is no direct control of initialization of these libraries and it being triggered lazily once NativeLibrary class is loaded once any API is invoked. This was worked around by introducing provider interface. Libraries will call it to resolve path. Implementation of this provider is done in corresponding concrete ToolChain implementation. Sample interface:
```java
package org.robovm.libimobiledevice;
public class NativeLibrary {
    private static LibMobDevicePlatformLibraryProvider platformLibraryProvider;
    public interface LibMobDevicePlatformLibraryProvider {
        File getLibMobDeviceLibrary();
        default void registerLibMobDeviceProvider() {
            platformLibraryProvider = this;
        }
    }
    ...
}
```

And implementation:
```java
public class ExternalCommonToolchain extends ToolchainUtil.Contract{
    ...
    @Override
    public File getLlvmLibrary() {
        validateToolchain();
        return new File(toolChainPath, "librobovm-llvm" + libExt);
    }
    ...
}
```

Tools invocation in `ExternalCommonToolchain` very similar to `DarwinToolchain` as mosts of tools are same as on MacOSX. The difference will be covered in corresponding tech notes posts.
