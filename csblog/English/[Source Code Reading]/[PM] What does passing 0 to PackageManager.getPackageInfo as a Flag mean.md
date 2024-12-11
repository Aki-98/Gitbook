# Environment

Android SDK 31（Android 12）



# Background

The issue arose from the following line of code in the project :

```java
PackageManager packageManager = context.getPackageManager();
PackageInfo info = packageManager.getPackageInfo(context.getPackageName(), 0);
```

The `0` used as a flag parameter is flagged by PM Challenge, suggesting that Android constants should be used instead of integer values. I consulted the SDK documentation for clarification.

SDK Documentation:

![1]([PM] What does passing 0 to PackageManager.getPackageInfo as a Flag mean_imgs\SpcSRFlX0uM.png)

Android Documentation：

![2]([PM] What does passing 0 to PackageManager.getPackageInfo as a Flag mean_imgs\lNMCyg4Jzz7.png)




It's evident that there is no corresponding constant value for 0.

There's no direct explanation of what passing 0 signifies, but it can be inferred that using other values and combinations of values could yield more information.

# Handling of Flags

From the previous SDK documentation, it's clear that `getPackageInfo` is an abstract method, so its implementation is not in `PackageManager`.

Printing `context.getPackageManager().getClass().getName()` reveals that the `context.getPackageManager()` method returns an `ApplicationPackageManager` object.

Here, we cannot yet see how the Android system retrieves `PackageInfo`, but we can observe the handling of flags.

**ApplicationPackageManager.getPackageInfo(String packageName, int flags) -->**

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

From the source code reading of the above aspects, it can also be inferred that when the flag is 0, no additional information about the four major components is obtained.

The handling of flags exists in various places within the Android system, but all are based on bitwise AND operations. We can conclude that as long as the flags passed in are 0, the flags used when finally obtaining `PackageInfo` will also be 0.

# Obtaining PackageInfo

Regarding obtaining `PackageInfo`, Android 11 introduced a caching mechanism. There's an article that explains this change well: https://juejin.cn/post/7004708493634568200

Apart from the caching mechanism, obtaining `PackageInfo` can ultimately be traced to the `getPackageInfoAsUserUncached()` method in `PackageManager`.

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

By reading the code of `ActivityThread`, it can be determined that `PackageInfo` is obtained through `IPackageManager` and is acquired by `PackageManagerService`.

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

And in `PackageManagerService`, the method responsible for obtaining `PackageInfo` can ultimately be traced to:

```java
3400          protected PackageInfo getPackageInfoInternalBody(String packageName, long versionCode,
3401                  int flags, int filterCallingUid, int userId) {
3402              // reader
3403              // Normalize package name to handle renamed packages and static libs
3404              packageName = resolveInternalPackageNameLPr(packageName, versionCode);
3405  
3406              final boolean matchFactoryOnly = (flags & MATCH_FACTORY_ONLY) != 0;
3407              if (matchFactoryOnly) { //[When flags=0, it doesn't enter here.]
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
    				// [the flags will be passed in this block]
3433                  if (filterSharedLibPackageLPr(ps, filterCallingUid, userId, flags)) {  
3434                      return null;
3435                  }
3436                  if (ps != null && shouldFilterApplicationLocked(ps, filterCallingUid, userId)) {
3437                      return null;
3438                  }
3439  
    				// [the flags will be passed in this block]
3440                  return generatePackageInfo(ps, flags, userId);
3441              }
    				// [This part is not entered when the package is found, only consider the method mentioned above for locating.]
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

`filterSharedLibPackageLPr` is merely a judgment method, and its method comment is: "Callers can access only the libs they depend on, otherwise they need to explicitly ask for the shared libraries given the caller is allowed to access all static libs."

The understanding of this is an access control mechanism. Without permission, access to shared libraries is not allowed, and the `getPackageInfoInternalBody` method will ultimately return a null value.

When this permission control mechanism is passed by default, it will ultimately enter the `generatePackageInfo(ps, flags, userId)` method.

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
		
        // [When flags are 0, it won't enter the following section.]
        if ((flags & MATCH_UNINSTALLED_PACKAGES) != 0
                && ps.isSystem()) {
            flags |= MATCH_ANY_USER;
        }

        final PackageUserState state = ps.readUserState(userId);
        AndroidPackage p = ps.pkg;
        if (p != null) {
            final PermissionsState permissionsState = ps.getPermissionsState();

            // [When flags are 0, the GID information will not be retrieved.]
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

    		// [the flags will be passed in this block]
            PackageInfo packageInfo = PackageInfoUtils.generate(p, gids, flags,
                    ps.firstInstallTime, ps.lastUpdateTime, permissions, state, userId, ps);

            if (packageInfo == null) {
                return null;
            }

            packageInfo.packageName = packageInfo.applicationInfo.packageName =
                    resolveExternalPackageNameLPr(p);

            return packageInfo;
            
            // [When PackageInfo is found, it won't enter the following section.]
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

Reading `PackageInfoUtils` will provide the final conclusion.

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
    		// [Here, the flags will be passed on.]
116          PackageInfo info = PackageInfoWithoutStateUtils.generateWithoutComponentsUnchecked(pkg,
117                  gids, flags, firstInstallTime, lastUpdateTime, grantedPermissions, state, userId,
118                  apexInfo, applicationInfo);
119  
120          info.isStub = pkg.isStub();
121          info.coreApp = pkg.isCoreApp();
122  
    		// [When flags are 0, ACTIVITIES information will not be retrieved.]
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
    		// [When flags are 0, RECEIVERS information will not be retrieved.]
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
    		// [When flags are 0, SERVICES information will not be retrieved.]
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
    		// [When flags are 0, PROVIDERS information will not be retrieved.]
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
    		// [When flags are 0, INSTRUMENTATION information will not be retrieved.]
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

Here's the source code of PackageInfoWithoutStateUtils.generateWithoutComponentsUnchecked() 

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
    		// [When flags are 0, GIDS information will not be retrieved.]
240          if ((flags & PackageManager.GET_GIDS) != 0) {
241              pi.gids = gids;
242          }
    		// [When flags are 0, CONFIGURATIONS information will not be retrieved.]
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
    		// [When flags are 0, PERMISSIONS information will not be retrieved.]
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
    		// [When flags are 0, ATTRIBUTIONS information will not be retrieved.]
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
    		// [When flags are 0, SIGNATURES information will not be retrieved.]
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
    		// [When flags are 0, SIGNING_CERTIFICATES information will not be retrieved.]
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



# Comments in the PackageInfo Source Code

The comments within the PackageInfo source code also indicate that when flags=0, certain information will not be retrieved.

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



# Summary

When using `PackageManager.getPackageInfo(String packageName, int flags)`, passing in flags as 0 will not retrieve information about the four major components, permissions, gids, etc.

You can refer to all flags starting with `PackageManager.GET_` to understand that excluding these flags will not fetch the corresponding information.

To include multiple flags, you can use the following format:

```java
PackageInfo info = packageManager.getPackageInfo(context.getPackageName(), PackageManager.GET_ACTIVITIES | PackageManager.GET_PERMISSIONS);
```

When passing flags=0, the basic information included in PackageInfo is:

- packageName
- versionCode
- applicationInfo
- firstInstallTime
- lastUpdateTime
- compileSdkVersion
- ...