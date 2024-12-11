# **背景**

系统内置应用是预装在 `/system` 分区中的应用，它们通常被标记为只读且版本由系统更新决定。但在运行过程中，用户可能会安装新版本的系统内置应用（位于 `/data` 分区），甚至将其降级到比 `/system` 中版本更低的状态。然而，当系统重启后，如果 `/data` 分区的版本低于 `/system` 分区中的版本，系统会自动还原到 `/system` 分区的版本。

------

### **代码解读**

可以直接找到 frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java Line 12162 ~ 12186的源码 

```java
// 比较 VersionCode，当/data下的VersionCode更低时进入此分支
if (isSystemPkgBetter) {
    // The version of the application on /system is greater than the version on
    // /data. Switch back to the application on /system.
    // It's safe to assume the application on /system will correctly scan. If not,
    // there won't be a working copy of the application.
    synchronized (mLock) {
        // just remove the loaded entries from package lists
        mPackages.remove(pkgSetting.name);
    }
 
    logCriticalInfo(Log.WARN,
            "System package updated;"
            + " name: " + pkgSetting.name
            + "; " + pkgSetting.versionCode + " --> " + parsedPackage.getLongVersionCode()
            + "; " + pkgSetting.getPathString()
                    + " --> " + parsedPackage.getPath());
 
    final InstallArgs args = createInstallArgsForExisting(
            pkgSetting.getPathString(), getAppDexInstructionSets(
                    pkgSetting.primaryCpuAbiString, pkgSetting.secondaryCpuAbiString));
    args.cleanUpResourcesLI();
    // 还原并启用 `/system` 分区版本
    synchronized (mLock) {
        mSettings.enableSystemPackageLPw(pkgSetting.name);
    }
}
```