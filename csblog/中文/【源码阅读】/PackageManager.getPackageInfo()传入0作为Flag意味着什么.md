# 环境

Android SDK 31（Android 12）



# 背景

起因是项目源码中有这一条代码

```java
PackageManager packageManager = context.getPackageManager();
PackageInfo info = packageManager.getPackageInfo(context.getPackageName(), 0);
```

其中0作为flag参数传递，被PM Chanllenge应该使用Android常量而不是int值，于是查阅SDK说明与文档。

SDK说明：

![1](PackageManager.getPackageInfo()传入0作为Flag意味着什么_imgs\1.png)

Android文档：

![2](PackageManager.getPackageInfo()传入0作为Flag意味着什么_imgs\2.png)

可见0并没有对应的常量值

也没有直接说明传入0代表什么，但可以猜测传入其他值及其他值的结合就可以获得更多的信息

# 对于Flag的处理

从前面的SDK说明中可以看到，getPackageInfo是abstract方法，所以实现并不在PackageManager中。

打印context.getPackageManager().getClass().getName()看到，context.getPackageManger()方法获得的是ApplicationPackageManager对象。

这里还无法看到Android系统是怎样获得PackageInfo的，但可以看到对flags的处理。

ApplicationPackageManager.getPackageInfo(String packageName, int flags) -->

```
210      @Override
211      public PackageInfo getPackageInfo(String packageName, int flags)
212              throws NameNotFoundException {
213          return getPackageInfoAsUser(packageName, flags, getUserId());
214      }
```

ApplicationPackageManager.getPackageInfoAsUser(String packageName, int flags, int userId) --> 

```java
232      @Override
233      public PackageInfo getPackageInfoAsUser(String packageName, int flags, int userId)
234              throws NameNotFoundException {
235          PackageInfo pi =
236                  getPackageInfoAsUserCached(
237                          packageName,
238                          updateFlagsForPackage(flags, userId),
239                          userId);
240          if (pi == null) {
241              throw new NameNotFoundException(packageName);
242          }
243          return pi;
244      }
```

**ApplicationPackageManager.updateFlagsForPackage(int flags, int userId)**

```java
1829      /**
1830       * Update given flags when being used to request {@link PackageInfo}.
1831       */
1832      private int updateFlagsForPackage(int flags, int userId) {
1833          if ((flags & (GET_ACTIVITIES | GET_RECEIVERS | GET_SERVICES | GET_PROVIDERS)) != 0) {
1834              // Caller is asking for component details, so they'd better be
1835              // asking for specific Direct Boot matching behavior
1836              if ((flags & (MATCH_DIRECT_BOOT_UNAWARE
1837                      | MATCH_DIRECT_BOOT_AWARE
1838                      | MATCH_DIRECT_BOOT_AUTO)) == 0) {
1839                  onImplicitDirectBoot(userId);
1840              }
1841          }
1842          return flags;
1843      }
```

从对上面方面的源码阅读中，也可以猜测到flag为0时，是不会额外获得四大组件信息的。

对于Flag的处理也存在在Android系统中的更多位置，但都脱离不开&位运算，可以得到结论说只要flags传入为0，在最终获取PackageInfo时使用的flags就为0

# 获取PackageInfo

关于获取PackageInfo，Android11引入了缓存机制，关于这一变动，有篇文章写得已经很好了：https://juejin.cn/post/7004708493634568200

除开缓存机制之外，获取PackageInfo最后可以定位到PackageManager的getPackageInfoAsUserUncached()方法

```java
private static PackageInfo getPackageInfoAsUserUncached(
    String packageName, int flags, int userId) {
    try {
        return ActivityThread.getPackageManager().getPackageInfo(packageName, flags, userId);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

通过阅读ActivityThread的代码可以知道，PackageInfo是通过IPackageManager，由PackageManagerService获得的

```java
2345      @UnsupportedAppUsage
2346      public static IPackageManager getPackageManager() {
2347          if (sPackageManager != null) {
2348              return sPackageManager;
2349          }
2350          final IBinder b = ServiceManager.getService("package");
2351          sPackageManager = IPackageManager.Stub.asInterface(b);
2352          return sPackageManager;
2353      }
```

而在PackageManagerService中，对于PackageInfo的获取最后可以定位到这个方法：

```java
3400          protected PackageInfo getPackageInfoInternalBody(String packageName, long versionCode,
3401                  int flags, int filterCallingUid, int userId) {
3402              // reader
3403              // Normalize package name to handle renamed packages and static libs
3404              packageName = resolveInternalPackageNameLPr(packageName, versionCode);
3405  
3406              final boolean matchFactoryOnly = (flags & MATCH_FACTORY_ONLY) != 0;
3407              if (matchFactoryOnly) { // flags=0时不走入这里
3408                  // Instant app filtering for APEX modules is ignored
3409                  if ((flags & MATCH_APEX) != 0) {
3410                      return mApexManager.getPackageInfo(packageName,
3411                              ApexManager.MATCH_FACTORY_PACKAGE);
3412                  }
3413                  final PackageSetting ps = mSettings.getDisabledSystemPkgLPr(packageName);
3414                  if (ps != null) {
3415                      if (filterSharedLibPackageLPr(ps, filterCallingUid, userId, flags)) {
3416                          return null;
3417                      }
3418                      if (shouldFilterApplicationLocked(ps, filterCallingUid, userId)) {
3419                          return null;
3420                      }
3421                      return generatePackageInfo(ps, flags, userId);
3422                  }
3423              }
3424  
3425              AndroidPackage p = mPackages.get(packageName);
3426              if (matchFactoryOnly && p != null && !p.isSystem()) {
3427                  return null;
3428              }
3429              if (DEBUG_PACKAGE_INFO)
3430                  Log.v(TAG, "getPackageInfo " + packageName + ": " + p);
3431              if (p != null) { 
3432                  final PackageSetting ps = getPackageSetting(p.getPackageName());
    				//这里会将flags继续传入
3433                  if (filterSharedLibPackageLPr(ps, filterCallingUid, userId, flags)) {  
3434                      return null;
3435                  }
3436                  if (ps != null && shouldFilterApplicationLocked(ps, filterCallingUid, userId)) {
3437                      return null;
3438                  }
3439  
    				//这里会将flags继续传入
3440                  return generatePackageInfo(ps, flags, userId);
3441              }
    				//找到Package时不走入这里了，只考虑上面的方法定位
3442              if (!matchFactoryOnly && (flags & MATCH_KNOWN_PACKAGES) != 0) {
3443                  final PackageSetting ps = mSettings.getPackageLPr(packageName);
3444                  if (ps == null) return null;
3445                  if (filterSharedLibPackageLPr(ps, filterCallingUid, userId, flags)) {
3446                      return null;
3447                  }
3448                  if (shouldFilterApplicationLocked(ps, filterCallingUid, userId)) {
3449                      return null;
3450                  }
3451                  return generatePackageInfo(ps, flags, userId);
3452              }
3453              if ((flags & MATCH_APEX) != 0) {
3454                  return mApexManager.getPackageInfo(packageName, ApexManager.MATCH_ACTIVE_PACKAGE);
3455              }
3456              return null;
3457          }
```

filterSharedLibPackageLPr也只是一个判断方法，方法注释是：Callers can access only the libs they depend on, otherwise they need to explicitly ask for the shared libraries given the caller is allowed to access all static libs.（调用者只能访问它所依赖的库，除非它明确请求访问共享库。另外，给定调用者可以访问所有静态库的情况下，它仍然需要显式请求访问共享库）

对此的理解是一个访问权限控制，没有权限的情况下不允许访问共享库，最终getPackageInfoInternalBody方法就会返回null值。

默认已通过这一权限控制机制时，最后会走到generatePackageInfo(ps, flags, userId)这一方法内

```java
    private PackageInfo generatePackageInfo(PackageSetting ps, int flags, int userId) {
        if (!mUserManager.exists(userId)) return null;
        if (ps == null) {
            return null;
        }
        final int callingUid = Binder.getCallingUid();
        // Filter out ephemeral app metadata:
        //   * The system/shell/root can see metadata for any app
        //   * An installed app can see metadata for 1) other installed apps
        //     and 2) ephemeral apps that have explicitly interacted with it
        //   * Ephemeral apps can only see their own data and exposed installed apps
        //   * Holding a signature permission allows seeing instant apps
        if (shouldFilterApplicationLocked(ps, callingUid, userId)) {
            return null;
        }
		
        // flags为0不会走入下面的
        if ((flags & MATCH_UNINSTALLED_PACKAGES) != 0
                && ps.isSystem()) {
            flags |= MATCH_ANY_USER;
        }

        final PackageUserState state = ps.readUserState(userId);
        AndroidPackage p = ps.pkg;
        if (p != null) {
            final PermissionsState permissionsState = ps.getPermissionsState();

            // flags为0不会获取GID信息
            // Compute GIDs only if requested
            final int[] gids = (flags & PackageManager.GET_GIDS) == 0
                    ? EMPTY_INT_ARRAY : permissionsState.computeGids(userId);
            // Compute granted permissions only if package has requested permissions
            Set<String> permissions = ArrayUtils.isEmpty(p.getRequestedPermissions())
                    ? Collections.emptySet() : permissionsState.getPermissions(userId);
            if (state.instantApp) {
                permissions = new ArraySet<>(permissions);
                permissions.removeIf(permissionName -> {
                    BasePermission permission = mPermissionManager.getPermissionTEMP(
                            permissionName);
                    if (permission == null) {
                        return true;
                    }
                    if (!permission.isInstant()) {
                        EventLog.writeEvent(0x534e4554, "140256621", UserHandle.getUid(userId,
                                ps.appId), permissionName);
                        return true;
                    }
                    return false;
                });
            }

    		// 这里会将flags继续传入
            PackageInfo packageInfo = PackageInfoUtils.generate(p, gids, flags,
                    ps.firstInstallTime, ps.lastUpdateTime, permissions, state, userId, ps);

            if (packageInfo == null) {
                return null;
            }

            packageInfo.packageName = packageInfo.applicationInfo.packageName =
                    resolveExternalPackageNameLPr(p);

            return packageInfo;
            
            // 找到了PackageInfo不会走入下面的
        } else if ((flags & MATCH_UNINSTALLED_PACKAGES) != 0 && state.isAvailable(flags)) {
            PackageInfo pi = new PackageInfo();
            pi.packageName = ps.name;
            pi.setLongVersionCode(ps.versionCode);
            pi.sharedUserId = (ps.sharedUser != null) ? ps.sharedUser.name : null;
            pi.firstInstallTime = ps.firstInstallTime;
            pi.lastUpdateTime = ps.lastUpdateTime;

            ApplicationInfo ai = new ApplicationInfo();
            ai.packageName = ps.name;
            ai.uid = UserHandle.getUid(userId, ps.appId);
            ai.primaryCpuAbi = ps.primaryCpuAbiString;
            ai.secondaryCpuAbi = ps.secondaryCpuAbiString;
            ai.setVersionCode(ps.versionCode);
            ai.flags = ps.pkgFlags;
            ai.privateFlags = ps.pkgPrivateFlags;
            pi.applicationInfo = PackageParser.generateApplicationInfo(ai, flags, state, userId);

            if (DEBUG_PACKAGE_INFO) Log.v(TAG, "ps.pkg is n/a for ["
                    + ps.name + "]. Provides a minimum info.");
            return pi;
        } else {
            return null;
        }
    }
```

对于PackageInfoUtils的阅读就可以得到最终结论了

```java
81      /**
82       * @param pkgSetting See {@link PackageInfoUtils} for description of pkgSetting usage.
83       */
84      @Nullable
85      public static PackageInfo generate(AndroidPackage pkg, int[] gids,
86              @PackageManager.PackageInfoFlags int flags, long firstInstallTime, long lastUpdateTime,
87              Set<String> grantedPermissions, PackageUserState state, int userId,
88              @Nullable PackageSetting pkgSetting) {
89          return generateWithComponents(pkg, gids, flags, firstInstallTime, lastUpdateTime,
90                  grantedPermissions, state, userId, null, pkgSetting);
91      }
92  
93      /**
94       * @param pkgSetting See {@link PackageInfoUtils} for description of pkgSetting usage.
95       */
96      @Nullable
97      public static PackageInfo generate(AndroidPackage pkg, ApexInfo apexInfo, int flags,
98              @Nullable PackageSetting pkgSetting) {
99          return generateWithComponents(pkg, EmptyArray.INT, flags, 0, 0, Collections.emptySet(),
100                  new PackageUserState(), UserHandle.getCallingUserId(), apexInfo, pkgSetting);
101      }
102  
103      /**
104       * @param pkgSetting See {@link PackageInfoUtils} for description of pkgSetting usage.
105       */
106      private static PackageInfo generateWithComponents(AndroidPackage pkg, int[] gids,
107              @PackageManager.PackageInfoFlags int flags, long firstInstallTime, long lastUpdateTime,
108              Set<String> grantedPermissions, PackageUserState state, int userId,
109              @Nullable ApexInfo apexInfo, @Nullable PackageSetting pkgSetting) {
110          ApplicationInfo applicationInfo = generateApplicationInfo(pkg, flags, state, userId,
111                  pkgSetting);
112          if (applicationInfo == null) {
113              return null;
114          }
115  
    		// 这里会将flags继续传入
116          PackageInfo info = PackageInfoWithoutStateUtils.generateWithoutComponentsUnchecked(pkg,
117                  gids, flags, firstInstallTime, lastUpdateTime, grantedPermissions, state, userId,
118                  apexInfo, applicationInfo);
119  
120          info.isStub = pkg.isStub();
121          info.coreApp = pkg.isCoreApp();
122  
    		// flags为0时不会获得ACTIVITIES信息
123          if ((flags & PackageManager.GET_ACTIVITIES) != 0) {
124              final int N = pkg.getActivities().size();
125              if (N > 0) {
126                  int num = 0;
127                  final ActivityInfo[] res = new ActivityInfo[N];
128                  for (int i = 0; i < N; i++) {
129                      final ParsedActivity a = pkg.getActivities().get(i);
130                      if (ComponentParseUtils.isMatch(state, pkg.isSystem(), pkg.isEnabled(), a,
131                              flags)) {
132                          if (PackageManager.APP_DETAILS_ACTIVITY_CLASS_NAME.equals(
133                                  a.getName())) {
134                              continue;
135                          }
136                          res[num++] = generateActivityInfo(pkg, a, flags, state,
137                                  applicationInfo, userId, pkgSetting);
138                      }
139                  }
140                  info.activities = ArrayUtils.trimToSize(res, num);
141              }
142          }
    		// flags为0时不会获得RECEIVERS信息
143          if ((flags & PackageManager.GET_RECEIVERS) != 0) {
144              final int size = pkg.getReceivers().size();
145              if (size > 0) {
146                  int num = 0;
147                  final ActivityInfo[] res = new ActivityInfo[size];
148                  for (int i = 0; i < size; i++) {
149                      final ParsedActivity a = pkg.getReceivers().get(i);
150                      if (ComponentParseUtils.isMatch(state, pkg.isSystem(), pkg.isEnabled(), a,
151                              flags)) {
152                          res[num++] = generateActivityInfo(pkg, a, flags, state, applicationInfo,
153                                  userId, pkgSetting);
154                      }
155                  }
156                  info.receivers = ArrayUtils.trimToSize(res, num);
157              }
158          }
    		// flags为0时不会获得SERVICES信息
159          if ((flags & PackageManager.GET_SERVICES) != 0) {
160              final int size = pkg.getServices().size();
161              if (size > 0) {
162                  int num = 0;
163                  final ServiceInfo[] res = new ServiceInfo[size];
164                  for (int i = 0; i < size; i++) {
165                      final ParsedService s = pkg.getServices().get(i);
166                      if (ComponentParseUtils.isMatch(state, pkg.isSystem(), pkg.isEnabled(), s,
167                              flags)) {
168                          res[num++] = generateServiceInfo(pkg, s, flags, state, applicationInfo,
169                                  userId, pkgSetting);
170                      }
171                  }
172                  info.services = ArrayUtils.trimToSize(res, num);
173              }
174          }
    		// flags为0时不会获得PROVIDERS信息
175          if ((flags & PackageManager.GET_PROVIDERS) != 0) {
176              final int size = pkg.getProviders().size();
177              if (size > 0) {
178                  int num = 0;
179                  final ProviderInfo[] res = new ProviderInfo[size];
180                  for (int i = 0; i < size; i++) {
181                      final ParsedProvider pr = pkg.getProviders()
182                              .get(i);
183                      if (ComponentParseUtils.isMatch(state, pkg.isSystem(), pkg.isEnabled(), pr,
184                              flags)) {
185                          res[num++] = generateProviderInfo(pkg, pr, flags, state, applicationInfo,
186                                  userId, pkgSetting);
187                      }
188                  }
189                  info.providers = ArrayUtils.trimToSize(res, num);
190              }
191          }
    		// flags为0时不会获得INSTRUMENTATION信息
192          if ((flags & PackageManager.GET_INSTRUMENTATION) != 0) {
193              int N = pkg.getInstrumentations().size();
194              if (N > 0) {
195                  info.instrumentation = new InstrumentationInfo[N];
196                  for (int i = 0; i < N; i++) {
197                      info.instrumentation[i] = generateInstrumentationInfo(
198                              pkg.getInstrumentations().get(i), pkg, flags, userId, pkgSetting);
199                  }
200              }
201          }
202  
203          return info;
204      }
205  
```

PackageInfoWithoutStateUtils.generateWithoutComponentsUnchecked() 源码如下

```java

201      /**
202       * This bypasses critical checks that are necessary for usage with data passed outside of
203       * system server.
204       *
205       * Prefer {@link #generateWithoutComponents(ParsingPackageRead, int[], int, long, long, Set,
206       * PackageUserState, int, ApexInfo, ApplicationInfo)}.
207       */
208      @NonNull
209      public static PackageInfo generateWithoutComponentsUnchecked(ParsingPackageRead pkg, int[] gids,
210              @PackageManager.PackageInfoFlags int flags, long firstInstallTime, long lastUpdateTime,
211              Set<String> grantedPermissions, PackageUserState state, int userId,
212              @Nullable ApexInfo apexInfo, @NonNull ApplicationInfo applicationInfo) {
213          PackageInfo pi = new PackageInfo();
214          pi.packageName = pkg.getPackageName();
215          pi.splitNames = pkg.getSplitNames();
216          pi.versionCode = pkg.getVersionCode();
217          pi.versionCodeMajor = pkg.getVersionCodeMajor();
218          pi.baseRevisionCode = pkg.getBaseRevisionCode();
219          pi.splitRevisionCodes = pkg.getSplitRevisionCodes();
220          pi.versionName = pkg.getVersionName();
221          pi.sharedUserId = pkg.getSharedUserId();
222          pi.sharedUserLabel = pkg.getSharedUserLabel();
223          pi.applicationInfo = applicationInfo;
224          pi.installLocation = pkg.getInstallLocation();
225          if ((pi.applicationInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0
226                  || (pi.applicationInfo.flags & ApplicationInfo.FLAG_UPDATED_SYSTEM_APP) != 0) {
227              pi.requiredForAllUsers = pkg.isRequiredForAllUsers();
228          }
229          pi.restrictedAccountType = pkg.getRestrictedAccountType();
230          pi.requiredAccountType = pkg.getRequiredAccountType();
231          pi.overlayTarget = pkg.getOverlayTarget();
232          pi.targetOverlayableName = pkg.getOverlayTargetName();
233          pi.overlayCategory = pkg.getOverlayCategory();
234          pi.overlayPriority = pkg.getOverlayPriority();
235          pi.mOverlayIsStatic = pkg.isOverlayIsStatic();
236          pi.compileSdkVersion = pkg.getCompileSdkVersion();
237          pi.compileSdkVersionCodename = pkg.getCompileSdkVersionCodeName();
238          pi.firstInstallTime = firstInstallTime;
239          pi.lastUpdateTime = lastUpdateTime;
    		// flags为0时不会获得GIDS信息
240          if ((flags & PackageManager.GET_GIDS) != 0) {
241              pi.gids = gids;
242          }
    		// flags为0时不会获得CONFIGURATIONS信息
243          if ((flags & PackageManager.GET_CONFIGURATIONS) != 0) {
244              int size = pkg.getConfigPreferences().size();
245              if (size > 0) {
246                  pi.configPreferences = new ConfigurationInfo[size];
247                  pkg.getConfigPreferences().toArray(pi.configPreferences);
248              }
249              size = pkg.getReqFeatures().size();
250              if (size > 0) {
251                  pi.reqFeatures = new FeatureInfo[size];
252                  pkg.getReqFeatures().toArray(pi.reqFeatures);
253              }
254              size = pkg.getFeatureGroups().size();
255              if (size > 0) {
256                  pi.featureGroups = new FeatureGroupInfo[size];
257                  pkg.getFeatureGroups().toArray(pi.featureGroups);
258              }
259          }
    		// flags为0时不会获得PERMISSIONS信息
260          if ((flags & PackageManager.GET_PERMISSIONS) != 0) {
261              int size = ArrayUtils.size(pkg.getPermissions());
262              if (size > 0) {
263                  pi.permissions = new PermissionInfo[size];
264                  for (int i = 0; i < size; i++) {
265                      pi.permissions[i] = generatePermissionInfo(pkg.getPermissions().get(i),
266                              flags);
267                  }
268              }
269              final List<ParsedUsesPermission> usesPermissions = pkg.getUsesPermissions();
270              size = usesPermissions.size();
271              if (size > 0) {
272                  pi.requestedPermissions = new String[size];
273                  pi.requestedPermissionsFlags = new int[size];
274                  for (int i = 0; i < size; i++) {
275                      final ParsedUsesPermission usesPermission = usesPermissions.get(i);
276                      pi.requestedPermissions[i] = usesPermission.name;
277                      // The notion of required permissions is deprecated but for compatibility.
278                      pi.requestedPermissionsFlags[i] |=
279                              PackageInfo.REQUESTED_PERMISSION_REQUIRED;
280                      if (grantedPermissions != null
281                              && grantedPermissions.contains(usesPermission.name)) {
282                          pi.requestedPermissionsFlags[i] |=
283                                  PackageInfo.REQUESTED_PERMISSION_GRANTED;
284                      }
285                      if ((usesPermission.usesPermissionFlags
286                              & ParsedUsesPermission.FLAG_NEVER_FOR_LOCATION) != 0) {
287                          pi.requestedPermissionsFlags[i] |=
288                                  PackageInfo.REQUESTED_PERMISSION_NEVER_FOR_LOCATION;
289                      }
290                  }
291              }
292          }
    		// flags为0时不会获得ATTRIBUTIONS信息
293          if ((flags & PackageManager.GET_ATTRIBUTIONS) != 0) {
294              int size = ArrayUtils.size(pkg.getAttributions());
295              if (size > 0) {
296                  pi.attributions = new Attribution[size];
297                  for (int i = 0; i < size; i++) {
298                      pi.attributions[i] = generateAttribution(pkg.getAttributions().get(i));
299                  }
300              }
301              if (pkg.areAttributionsUserVisible()) {
302                  pi.applicationInfo.privateFlagsExt
303                          |= ApplicationInfo.PRIVATE_FLAG_EXT_ATTRIBUTIONS_ARE_USER_VISIBLE;
304              } else {
305                  pi.applicationInfo.privateFlagsExt
306                          &= ~ApplicationInfo.PRIVATE_FLAG_EXT_ATTRIBUTIONS_ARE_USER_VISIBLE;
307              }
308          } else {
309              pi.applicationInfo.privateFlagsExt
310                      &= ~ApplicationInfo.PRIVATE_FLAG_EXT_ATTRIBUTIONS_ARE_USER_VISIBLE;
311          }
312  
313          if (apexInfo != null) {
314              File apexFile = new File(apexInfo.modulePath);
315  
316              pi.applicationInfo.sourceDir = apexFile.getPath();
317              pi.applicationInfo.publicSourceDir = apexFile.getPath();
318              if (apexInfo.isFactory) {
319                  pi.applicationInfo.flags |= ApplicationInfo.FLAG_SYSTEM;
320                  pi.applicationInfo.flags &= ~ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
321              } else {
322                  pi.applicationInfo.flags &= ~ApplicationInfo.FLAG_SYSTEM;
323                  pi.applicationInfo.flags |= ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
324              }
325              if (apexInfo.isActive) {
326                  pi.applicationInfo.flags |= ApplicationInfo.FLAG_INSTALLED;
327              } else {
328                  pi.applicationInfo.flags &= ~ApplicationInfo.FLAG_INSTALLED;
329              }
330              pi.isApex = true;
331          }
332  
333          PackageParser.SigningDetails signingDetails = pkg.getSigningDetails();
334          // deprecated method of getting signing certificates
    		// flags为0时不会获得SIGNATURES信息
335          if ((flags & PackageManager.GET_SIGNATURES) != 0) {
336              if (signingDetails.hasPastSigningCertificates()) {
337                  // Package has included signing certificate rotation information.  Return the oldest
338                  // cert so that programmatic checks keep working even if unaware of key rotation.
339                  pi.signatures = new Signature[1];
340                  pi.signatures[0] = signingDetails.pastSigningCertificates[0];
341              } else if (signingDetails.hasSignatures()) {
342                  // otherwise keep old behavior
343                  int numberOfSigs = signingDetails.signatures.length;
344                  pi.signatures = new Signature[numberOfSigs];
345                  System.arraycopy(signingDetails.signatures, 0, pi.signatures, 0,
346                          numberOfSigs);
347              }
348          }
349  
    		// flags为0时不会获得SIGNING_CERTIFICATESN信息
350          // replacement for GET_SIGNATURES
351          if ((flags & PackageManager.GET_SIGNING_CERTIFICATES) != 0) {
352              if (signingDetails != PackageParser.SigningDetails.UNKNOWN) {
353                  // only return a valid SigningInfo if there is signing information to report
354                  pi.signingInfo = new SigningInfo(signingDetails);
355              } else {
356                  pi.signingInfo = null;
357              }
358          }
359  
360          return pi;
361      }
362  
```





# PackageInfo本身的注释

PackageInfo源码中本身含有的注释也说明了flags=0时，不会获取的信息

```java
25  /**
26   * Overall information about the contents of a package.  This corresponds
27   * to all of the information collected from AndroidManifest.xml.
28   */
29  public class PackageInfo implements Parcelable {
    ...
146      /**
147       * All kernel group-IDs that have been assigned to this package.
148       * This is only filled in if the flag {@link PackageManager#GET_GIDS} was set.
149       */
150      public int[] gids;
151  
152      /**
153       * Array of all {@link android.R.styleable#AndroidManifestActivity
154       * &lt;activity&gt;} tags included under &lt;application&gt;,
155       * or null if there were none.  This is only filled in if the flag
156       * {@link PackageManager#GET_ACTIVITIES} was set.
157       */
158      public ActivityInfo[] activities;
159  
160      /**
161       * Array of all {@link android.R.styleable#AndroidManifestReceiver
162       * &lt;receiver&gt;} tags included under &lt;application&gt;,
163       * or null if there were none.  This is only filled in if the flag
164       * {@link PackageManager#GET_RECEIVERS} was set.
165       */
166      public ActivityInfo[] receivers;
167  
168      /**
169       * Array of all {@link android.R.styleable#AndroidManifestService
170       * &lt;service&gt;} tags included under &lt;application&gt;,
171       * or null if there were none.  This is only filled in if the flag
172       * {@link PackageManager#GET_SERVICES} was set.
173       */
174      public ServiceInfo[] services;
175  
176      /**
177       * Array of all {@link android.R.styleable#AndroidManifestProvider
178       * &lt;provider&gt;} tags included under &lt;application&gt;,
179       * or null if there were none.  This is only filled in if the flag
180       * {@link PackageManager#GET_PROVIDERS} was set.
181       */
182      public ProviderInfo[] providers;
183  
184      /**
185       * Array of all {@link android.R.styleable#AndroidManifestInstrumentation
186       * &lt;instrumentation&gt;} tags included under &lt;manifest&gt;,
187       * or null if there were none.  This is only filled in if the flag
188       * {@link PackageManager#GET_INSTRUMENTATION} was set.
189       */
190      public InstrumentationInfo[] instrumentation;
191  
192      /**
193       * Array of all {@link android.R.styleable#AndroidManifestPermission
194       * &lt;permission&gt;} tags included under &lt;manifest&gt;,
195       * or null if there were none.  This is only filled in if the flag
196       * {@link PackageManager#GET_PERMISSIONS} was set.
197       */
198      public PermissionInfo[] permissions;
199  
200      /**
201       * Array of all {@link android.R.styleable#AndroidManifestUsesPermission
202       * &lt;uses-permission&gt;} tags included under &lt;manifest&gt;,
203       * or null if there were none.  This is only filled in if the flag
204       * {@link PackageManager#GET_PERMISSIONS} was set.  This list includes
205       * all permissions requested, even those that were not granted or known
206       * by the system at install time.
207       */
208      public String[] requestedPermissions;
209  
210      /**
211       * Array of flags of all {@link android.R.styleable#AndroidManifestUsesPermission
212       * &lt;uses-permission&gt;} tags included under &lt;manifest&gt;,
213       * or null if there were none.  This is only filled in if the flag
214       * {@link PackageManager#GET_PERMISSIONS} was set.  Each value matches
215       * the corresponding entry in {@link #requestedPermissions}, and will have
216       * the flag {@link #REQUESTED_PERMISSION_GRANTED} set as appropriate.
217       */
218      public int[] requestedPermissionsFlags;
219  
220      /**
221       * Array of all {@link android.R.styleable#AndroidManifestAttribution
222       * &lt;attribution&gt;} tags included under &lt;manifest&gt;, or null if there were none. This
223       * is only filled if the flag {@link PackageManager#GET_ATTRIBUTIONS} was set.
224       */
225      @SuppressWarnings({"ArrayReturn", "NullableCollection"})
226      public @Nullable Attribution[] attributions;
227  
253      /**
254       * Array of all signatures read from the package file. This is only filled
255       * in if the flag {@link PackageManager#GET_SIGNATURES} was set. A package
256       * must be signed with at least one certificate which is at position zero.
257       * The package can be signed with additional certificates which appear as
258       * subsequent entries.
259       *
260       * <strong>Note:</strong> Signature ordering is not guaranteed to be
261       * stable which means that a package signed with certificates A and B is
262       * equivalent to being signed with certificates B and A. This means that
263       * in case multiple signatures are reported you cannot assume the one at
264       * the first position to be the same across updates.
265       *
266       * <strong>Deprecated</strong> This has been replaced by the
267       * {@link PackageInfo#signingInfo} field, which takes into
268       * account signing certificate rotation.  For backwards compatibility in
269       * the event of signing certificate rotation, this will return the oldest
270       * reported signing certificate, so that an application will appear to
271       * callers as though no rotation occurred.
272       *
273       * @deprecated use {@code signingInfo} instead
274       */
275      @Deprecated
276      public Signature[] signatures;
277  
278      /**
279       * Signing information read from the package file, potentially
280       * including past signing certificates no longer used after signing
281       * certificate rotation.  This is only filled in if
282       * the flag {@link PackageManager#GET_SIGNING_CERTIFICATES} was set.
283       *
284       * Use this field instead of the deprecated {@code signatures} field.
285       * See {@link SigningInfo} for more information on its contents.
286       */
287      public SigningInfo signingInfo;
288  
289      /**
290       * Application specified preferred configuration
291       * {@link android.R.styleable#AndroidManifestUsesConfiguration
292       * &lt;uses-configuration&gt;} tags included under &lt;manifest&gt;,
293       * or null if there were none. This is only filled in if the flag
294       * {@link PackageManager#GET_CONFIGURATIONS} was set.
295       */
296      public ConfigurationInfo[] configPreferences;
297  
```



# 总结

当使用PackageManager.getPackageInfo(String packageName, int flags)时，传入flags等于0，不会获取四大组件、permissions、gids等信息。

可以参考PackageManager.GET_开头的所有Flag，认为不加入此Flag，并不会获得相应信息。



如果要加入多个Flag，可以参考下面的写法：

```java
PackageInfo info = packageManager.getPackageInfo(context.getPackageName(), PackageManager.GET_ACTIVITIES | PackageManager.GET_PERMISSIONS);
```



传入flags=0时PackageInfo包含的基础信息：

- packageName
- versionCode
- applicationInfo
- firstInstallTime
- lastUpdateTime
- compileSdkVersion
- ...