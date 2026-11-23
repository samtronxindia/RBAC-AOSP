# RBAC-AOSP: role-based access control in the Android Open Source Project

This repository documents the changes made to the Android Open Source Project (AOSP) to implement role-based access control (RBAC) for Android apps, as described in the dissertation *Role Based Access Control Models for Android* (Samir Talegaon, The University of Texas at San Antonio, December 2020).

In stock Android, the user grants individual permissions to apps. In this implementation, permissions are assigned to roles, roles are assigned to apps, and an app activates a role to receive its permissions. A privileged system app, the Role Manager, holds the role data and performs the grants; a new system service lets ordinary apps request and activate roles.

**Status:** this repository currently holds the list of changes only. The modified source files and patches are not published here yet (see [Source files](#source-files)).

## Related repositories

- [SamirRoleManager](https://github.com/samtronxindia/SamirRoleManager): the Role Manager system app (source, screenshots, demo video).
- [Role-Mining-Algorithms](https://github.com/samtronxindia/Role-Mining-Algorithms): the role mining algorithms evaluated for deriving roles from app permission data.
- [AOSP-download-script](https://github.com/samtronxindia/AOSP-download-script): a script for downloading the AOSP source.

## Platform

- Android 10 (API level 29), AOSP source build
- Run on the Android emulator
- Models implemented: RiAa (the RBAC model) and ARiA0 (the base administrative model) from the dissertation

All paths below are relative to the AOSP source root.

## How the pieces fit together

1. An app asks for a role, or asks to activate one, by calling the **Role Manager Service** through its AIDL interface. The service is obtained with `getSystemService(Context.ROLE_MANAGER_SERVICE)`.
2. The Role Manager Service forwards the request to the **Role Manager** app with a protected system broadcast.
3. The Role Manager prompts the user where needed, updates its tables (role-to-app assignment, role-to-permission assignment, active roles, running apps), and grants or revokes the role's permissions through `PackageManager.grantRuntimePermission` and `revokeRuntimePermission`.
4. When an app is uninstalled, a broadcast from `PackageManagerService` tells the Role Manager to revoke its roles. When an app stops running, a periodic alarm check lets the Role Manager de-activate its roles.

## Changes to existing AOSP files

### Permission framework

| File | Change | Why |
| --- | --- | --- |
| `frameworks/base/core/java/android/content/pm/PackageManager.java` | Removed `@SystemApi` and `@hide` from `grantRuntimePermission` and `revokeRuntimePermission` | So the Role Manager can be compiled against the SDK in Android Studio |
| `frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java` | `grantRuntimePermission` and `revokeRuntimePermission` also accept normal permissions | Roles contain normal permissions as well as dangerous ones |
| `frameworks/base/services/core/java/com/android/server/pm/permission/BasePermission.java` | `enforceDeclaredUsedAndRuntimeOrDevelopment` allows normal permissions to be granted and revoked | Same reason. Normal permissions backed by a GID take effect only in a new process, so the app process is restarted after such a grant |
| `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java` | Sends the protected broadcast `Intent.ACTION_PACKAGE_FULLY_REMOVED` on uninstall | So the Role Manager can revoke the removed app's roles and permissions |
| `frameworks/base/services/core/java/com/android/server/AlarmManagerService.java` | Overrides the 60-second minimum alarm interval | So the Role Manager can check running apps more often and de-activate roles soon after an app stops |
| `frameworks/base/core/res/AndroidManifest.xml` | Removed the permission-group association from every dangerous permission | So permissions are governed by the Role Manager's role-to-permission table instead of permission groups |
| `frameworks/base/data/etc/privapp-permissions-platform.xml` | Whitelists `com.samir.samirrolemanager` for `GRANT_RUNTIME_PERMISSIONS` and `REVOKE_RUNTIME_PERMISSIONS` | Gives the Role Manager the system-only permissions it needs |
| `frameworks/base/data/etc/platform.xml` | Modified (listed in the dissertation's Appendix B; the change is not described in the text) | |

### Registering the Role Manager Service

| File | Change | Why |
| --- | --- | --- |
| `frameworks/base/core/java/android/content/Context.java` | Defines `Context.ROLE_MANAGER_SERVICE` | So apps can obtain the service with `getSystemService` |
| `frameworks/base/core/java/android/app/SystemServiceRegistry.java` | Registers the service with `registerService` | Makes the service known to the framework |
| `frameworks/base/services/java/com/android/server/SystemServer.java` | Adds the service with `ServiceManager.addService()` in `startOtherServices` | Starts the service at boot |
| `frameworks/base/Android.mk` | Adds `frameworks/base/core/java/android/app/RMServiceManager.aidl` to `LOCAL_AIDL_INCLUDES` | So the service's AIDL interface is compiled |
| `frameworks/base/services/core/Android.bp` | Adds the AndroidX annotations library as a dependency | The service uses `@Nullable` on its parameters |

### SELinux policy

| File | Change | Why |
| --- | --- | --- |
| `system/sepolicy/private/service_contexts` | Assigns a label to the new service | So processes can add and find its binder reference |
| `system/sepolicy/public/service.te` and `system/sepolicy/prebuilts/api/29.0/public/service.te` | Modified for the new service | The prebuilt copy must match the public policy for the build to pass |
| `system/sepolicy/private/compat/29.0/29.0.cil`, `28.0/28.0.cil`, `27.0/27.0.cil`, `26.0/26.0.cil` | Adds the new service | Required for it to run as a background system service |
| `system/sepolicy/private/untrusted_app.te` | Adds a `find` rule for the service | So third-party apps can locate it |

### Build and SDK

| File | Change | Why |
| --- | --- | --- |
| `build/make/target/product/base_system.mk` | Adds the Role Manager package to `PRODUCT_PACKAGES` | Includes the app in the system image |
| `prebuilts/sdk/29/public/api/android.txt` | Adds `grantRuntimePermission` and `revokeRuntimePermission` | So the custom SDK used to compile the Role Manager exposes the two methods |
| `prebuilts/sdk/29/system/api/android.test.mock.txt` | The same two methods commented out | Same reason |

## New files

### Role Manager Service (in the framework)

| File | Purpose |
| --- | --- |
| `RoleManagerService.aidl` | AIDL interface with two methods: request a role, activate a role |
| `RoleManagerService.java` | The system service; forwards app requests to the Role Manager through protected broadcasts |
| `RMServiceManager.java` | Manager class that apps call; passes requests to the service |

### Role Manager (privileged system app)

| File | Purpose |
| --- | --- |
| `packages/RoleManager/Android.mk` | Prebuilt package: `LOCAL_MODULE := samirrolemanager`, `LOCAL_CERTIFICATE := platform`, `LOCAL_PRIVILEGED_MODULE := true`, `LOCAL_SRC_FILES := samirrolemanager.apk` |
| `packages/RoleManager/samirrolemanager.apk` | The compiled Role Manager, built in Android Studio against the modified SDK |

The app's source is in [SamirRoleManager](https://github.com/samtronxindia/SamirRoleManager). Its main parts:

| File | Purpose |
| --- | --- |
| `RoleManagerDB.java`, `RoleManagerDAO.java` | Room database and data access object |
| `RoleApp.java`, `RolePermission.java` | Role-to-app and role-to-permission assignments |
| `RoleActiveApp.java`, `RunningApps.java` | Active roles and running apps (the session data) |
| `RoleRequestReceiver.java`, `Prompt.java` | Receive a role request and prompt the user |
| `RoleActivationReceiver.java` | Receive a role activation, grant the role's permissions, show a confirmation |
| `AppUninstallReceiver.java` | Revoke roles and permissions when an app is uninstalled |

## Notes on the file list

The list is compiled from Chapter 6 and Appendix B of the dissertation. Three points where those two sources differ:

- `untrusted_app.te` is described in Section 6.5 but is not in Appendix B; its directory above is the usual location and is not stated in the dissertation.
- Appendix B names `service_contexts.te`; the AOSP file is `service_contexts`.
- The dissertation does not give the directory of the three Role Manager Service files. The `Android.mk` entry places `RMServiceManager.aidl` in `frameworks/base/core/java/android/app/`.

## Source files

The modified AOSP files are not in this repository yet. AOSP is licensed under the Apache License 2.0, so modified files or patches against the Android 10 sources can be published here with their licence headers kept.

## References

- Samir Talegaon. *Role Based Access Control Models for Android*. PhD dissertation, The University of Texas at San Antonio, December 2020. [PDF](https://ics.utsa.edu/dissertations/2020_SAMIR_TALEGAON_Role_Based_Access_Control_Mode.pdf)
- Samir Talegaon and Ram Krishnan. Role-Based Access Control Models for Android. IEEE TPS-ISA 2020. DOI [10.1109/TPS-ISA50397.2020.00033](https://doi.org/10.1109/TPS-ISA50397.2020.00033)
- Samir Talegaon and Ram Krishnan. Administrative Models for Role Based Access Control in Android. 2020.
