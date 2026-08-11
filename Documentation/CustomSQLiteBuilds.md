Custom SQLite Builds
====================

By default, GRDB uses the version of SQLite that ships with the target operating system.

**You can have GRDB use another SQLite.** Two techniques are available:

- [Swift Package Manager](#swift-package-manager): your package graph provides SQLite, and GRDB links against it. This technique does not enable extra GRDB APIs.

- [Xcode and SQLiteLib](#xcode-and-sqlitelib): Xcode builds SQLite from source with the compilation options of your choice, and GRDB gets extra APIs, such as [SQLite Pre-Update Hooks](https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/transactionobserver). This technique is not compatible with the Swift Package Manager.


## Swift Package Manager

GRDB defines a package trait named `SystemSQLite`, enabled by default. It links GRDB against the SQLite library that ships with the target operating system.

**Disable this trait when your package graph provides SQLite.** GRDB then declares no SQLite link of its own, and uses the SQLite that your binary already contains:

```swift
dependencies: [
    .package(url: "https://github.com/groue/GRDB.swift.git", from: "x.y.z", traits: []),
],
targets: [
    // Contains Sources/CSQLite/sqlite3.c
    // and Sources/CSQLite/include/sqlite3.h
    .target(
        name: "CSQLite",
        cSettings: [
            .define("SQLITE_ENABLE_FTS5"),
            .define("SQLITE_ENABLE_SNAPSHOT"),
        ]),
    .executableTarget(
        name: "MyApp",
        dependencies: [
            "CSQLite",
            .product(name: "GRDB", package: "GRDB.swift"),
        ]),
]
```

The `CSQLite` target above contains the [SQLite amalgamation](https://www.sqlite.org/amalgamation.html). Any other target or package that exports the SQLite symbols works as well. GRDB does not import it, and does not need to know about it. GRDB imports `GRDBSQLite`, a system library target that declares no symbol of its own, and any SQLite in the final link satisfies it.

`traits: []` disables all the default traits of GRDB, not only `SystemSQLite`. SPM has no syntax for disabling a single trait, so if GRDB ever ships another default trait, you will have to list it yourself.

Unlike the Xcode technique below, this one is a plain SPM dependency. Companion libraries such as [GRDBQuery](https://github.com/groue/GRDBQuery) and [GRDBSnapshotTesting](https://github.com/groue/GRDBSnapshotTesting) keep working.

> Warning: The `sqlite3` link is declared on the `GRDB` target. A target that depends on the `GRDBSQLite` product alone does not get it. Depend on `GRDB`.

### The header search path

`GRDBSQLite` includes `<sqlite3.h>`, and the compiler must find the header that matches the library you link. Apple platforms, and Linux systems that have `libsqlite3-dev` installed, already have one on the default search path. Other platforms have none.

SPM applies build settings to the target that declares them, and a root package can not add compiler flags to a target of one of its dependencies. Pass the header search path to the build itself:

```sh
swift build -Xcc -I -Xcc Sources/CSQLite/include
```

Or check a `toolset.json` file into your repository, so that everybody builds the same way:

```json
{
  "schemaVersion": "1.0",
  "swiftCompiler": {
    "extraCLIOptions": ["-Xcc", "-I", "-Xcc", "Sources/CSQLite/include"]
  }
}
```

```sh
swift build --toolset toolset.json
```

The `swift test` and `swift run` commands accept the same option.

> Warning: On Apple platforms, disabling the trait is not enough. The compiler finds the `sqlite3.h` of the SDK, and the SDK header emits an automatic link directive for `/usr/lib/libsqlite3.dylib`. A package that disables the trait, but does not redirect the header search path, silently keeps the system SQLite, and builds without any error.
>
> Run `otool -L` on the built binary, and check for `/usr/lib/libsqlite3.dylib`.

### What GRDB expects from your SQLite

- **SQLite 3.20.0 or later**, as with any other installation technique.

- **WAL snapshots**, on all platforms but Linux. Build your SQLite with `SQLITE_ENABLE_SNAPSHOT`, so that it exports `sqlite3_snapshot_get`, `sqlite3_snapshot_open`, `sqlite3_snapshot_free` and `sqlite3_snapshot_cmp`. Otherwise the link fails with undefined symbols.

- **FTS5**, if you use full-text search. GRDB looks the FTS5 API up at runtime, so a SQLite without FTS5 links, and then `FTS5.api(_:)` crashes. Build your SQLite with `SQLITE_ENABLE_FTS5`.

> Note: The `SystemSQLite` trait selects the SQLite that GRDB links against. It does not change the flags that GRDB is compiled with, so it does not enable extra GRDB APIs. Pre-update hooks still need the Xcode technique below.


## Xcode and SQLiteLib

**You can build GRDB with a custom build of [SQLite 3.47.2](https://www.sqlite.org/changes.html).**

A custom SQLite build can activate extra SQLite features, and extra GRDB features as well, such as support for the [FTS5 full-text search engine](../../../#full-text-search), and [SQLite Pre-Update Hooks](https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/transactionobserver).

GRDB builds SQLite with [swiftlyfalling/SQLiteLib](https://github.com/swiftlyfalling/SQLiteLib), which uses the same SQLite configuration as the one used by Apple in its operating systems, and lets you add extra compilation options that leverage the features you need.

> Warning: This technique is not compatible with the Swift Package Manager (SPM). It will create [build issues](https://github.com/groue/GRDB.swift/issues/1709) with SPM companion libraries such as [GRDBQuery](https://github.com/groue/GRDBQuery) or [GRDBSnapshotTesting](https://github.com/groue/GRDBSnapshotTesting). SPM users should read [Swift Package Manager](#swift-package-manager) above.

**To install GRDB with a custom SQLite build:**

1. Clone the GRDB git repository, checkout the latest tagged version:

    ```sh
    cd [GRDB directory]
    git checkout [latest tag]
    git submodule update --init SQLiteCustom/src
    ```

2. Choose your [extra compilation options](https://www.sqlite.org/compile.html). For example, `SQLITE_ENABLE_FTS5`, `SQLITE_ENABLE_PREUPDATE_HOOK`.

    It is recommended that you enable the `SQLITE_ENABLE_SNAPSHOT` option. It allows GRDB to optimize [ValueObservation](https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/valueobservation) when you use a [Database Pool](https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/databasepool).

3. Create a folder named `GRDBCustomSQLite` somewhere in your project directory.

4. Create four files in the `GRDBCustomSQLite` folder:

    - `SQLiteLib-USER.xcconfig`: this file sets the extra SQLite compilation flags.

        ```xcconfig
        // As many -D options as there are custom SQLite compilation options
        // Note: there is no space between -D and the option name.
        CUSTOM_SQLLIBRARY_CFLAGS = -DSQLITE_ENABLE_SNAPSHOT -DSQLITE_ENABLE_FTS5
        ```

    - `GRDBCustomSQLite-USER.xcconfig`: this file lets GRDB know about extra compilation flags, and enables extra GRDB APIs.

        ```xcconfig
        // As many -D options as there are custom SQLite compilation options
        // Note: there is one space between -D and the option name.
        CUSTOM_OTHER_SWIFT_FLAGS = -D SQLITE_ENABLE_SNAPSHOT -D SQLITE_ENABLE_FTS5
        ```

    - `GRDBCustomSQLite-USER.h`: this file lets your application know about extra compilation flags.

        ```c
        // As many #define as there are custom SQLite compilation options
        #define SQLITE_ENABLE_SNAPSHOT
        #define SQLITE_ENABLE_FTS5
        ```

    - `GRDBCustomSQLite-INSTALL.sh`: this file installs the three other files.

        ```sh
        # License: MIT License
        # https://github.com/swiftlyfalling/SQLiteLib/blob/master/LICENSE
        #
        #######################################################
        #                   PROJECT PATHS
        #  !! MODIFY THESE TO MATCH YOUR PROJECT HIERARCHY !!
        #######################################################

        # The path to the folder containing GRDBCustom.xcodeproj:
        GRDB_SOURCE_PATH="${PROJECT_DIR}/GRDB"

        # The path to your custom "SQLiteLib-USER.xcconfig":
        SQLITELIB_XCCONFIG_USER_PATH="${PROJECT_DIR}/GRDBCustomSQLite/SQLiteLib-USER.xcconfig"

        # The path to your custom "GRDBCustomSQLite-USER.xcconfig":
        CUSTOMSQLITE_XCCONFIG_USER_PATH="${PROJECT_DIR}/GRDBCustomSQLite/GRDBCustomSQLite-USER.xcconfig"

        # The path to your custom "GRDBCustomSQLite-USER.h":
        CUSTOMSQLITE_H_USER_PATH="${PROJECT_DIR}/GRDBCustomSQLite/GRDBCustomSQLite-USER.h"

        #######################################################
        #
        #######################################################


        if [ ! -d "$GRDB_SOURCE_PATH" ];
        then
        echo "error: Path to GRDB source (GRDB_SOURCE_PATH) missing/incorrect: $GRDB_SOURCE_PATH"
        exit 1
        fi

        SyncFileChanges () {
            SOURCE=$1
            DESTINATIONPATH=$2
            DESTINATIONFILENAME=$3
            DESTINATION="${DESTINATIONPATH}/${DESTINATIONFILENAME}"

            if [ ! -f "$SOURCE" ];
            then
            echo "error: Source file missing: $SOURCE"
            exit 1
            fi

            rsync -a "$SOURCE" "$DESTINATION"
        }

        SyncFileChanges $SQLITELIB_XCCONFIG_USER_PATH "${GRDB_SOURCE_PATH}/SQLiteCustom/src" "SQLiteLib-USER.xcconfig"
        SyncFileChanges $CUSTOMSQLITE_XCCONFIG_USER_PATH "${GRDB_SOURCE_PATH}/SQLiteCustom" "GRDBCustomSQLite-USER.xcconfig"
        SyncFileChanges $CUSTOMSQLITE_H_USER_PATH "${GRDB_SOURCE_PATH}/SQLiteCustom" "GRDBCustomSQLite-USER.h"

        echo "Finished syncing"
        ```

        Modify the top of `GRDBCustomSQLite-INSTALL.sh` file so that it contains correct paths.

5. Embed the `GRDBCustom.xcodeproj` project in your own project.

6. Add the `GRDBCustom` target in the **Target Dependencies** section of the **Build Phases** tab of your **application target**.

7. Add the `GRDBCustom.framework` from the targeted platform to the **Embedded Binaries** section of the **General**  tab of your **application target**.

8. Add a Run Script phase for your target in the **Pre-actions** section of the **Build** tab of your **application scheme**:

    ```sh
    source "${PROJECT_DIR}/GRDBCustomSQLite/GRDBCustomSQLite-INSTALL.sh"
    ```

    The path should be the path to your `GRDBCustomSQLite-INSTALL.sh` file.

    Select your application target in the "Provide build settings from" menu.

9. Check the "Shared" checkbox of your application scheme (this lets you commit the pre-action in your Version Control System).

10. If you have enabled "Hardened Runtime" for your target (**Build Settings**/**Signing**) then you may need to check **Disable Library Validation** under the **Hardened Runtime** section of the **Signing & Capabilities** tab.

    (The build error without this exception is "Library not loaded ... different Team IDs")

Now you can use GRDB with your custom SQLite build:

```swift
import GRDB

let dbQueue = try DatabaseQueue(...)
```
