### Background

System pre-installed applications are stored in the `/system` partition. These applications are typically marked as read-only, and their versions are determined by system updates. However, during runtime, users may install newer versions of these system pre-installed applications (stored in the `/data` partition) or even downgrade them to versions lower than those in the `/system` partition. However, after a system reboot, if the version in the `/data` partition is lower than the version in the `/system` partition, the system will automatically revert to the version in the `/system` partition.

------

### **Code Explanation**

The relevant source code can be found in `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java` at Line 12162 ~ 12186:

```java
java复制代码// Compare VersionCode, if the VersionCode in /data is lower, execute this branch
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
    // Restore and enable the version in the `/system` partition
    synchronized (mLock) {
        mSettings.enableSystemPackageLPw(pkgSetting.name);
    }
}
```