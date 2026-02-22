# Convention Plugins Patterns (build-logic)

build-logic included build pattern for consistent multi-module Gradle configuration. Eliminates duplicated build config across feature modules.

---

## Directory Structure

```
project-root/
├── build-logic/
│   ├── convention/
│   │   ├── build.gradle.kts
│   │   └── src/main/kotlin/
│   │       ├── KmpLibraryConventionPlugin.kt
│   │       ├── ComposeFeatureConventionPlugin.kt
│   │       ├── AndroidAppConventionPlugin.kt
│   │       └── AppConfig.kt
│   ├── settings.gradle.kts
│   └── gradle.properties
├── gradle/
│   └── libs.versions.toml
├── settings.gradle.kts          # includeBuild("build-logic")
└── ...modules
```

---

## Version Catalog Entries (for classpath)

```toml
[libraries]
kotlin-gradle-plugin = { group = "org.jetbrains.kotlin", name = "kotlin-gradle-plugin", version.ref = "kotlin" }
compose-gradle-plugin = { group = "org.jetbrains.compose", name = "compose-gradle-plugin", version.ref = "compose-multiplatform" }
android-gradle-plugin = { group = "com.android.tools.build", name = "gradle", version.ref = "agp" }
```

---

## build-logic/settings.gradle.kts

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}

rootProject.name = "build-logic"
include(":convention")
```

---

## build-logic/gradle.properties

```properties
org.gradle.parallel=true
```

---

## build-logic/convention/build.gradle.kts

```kotlin
plugins {
    `kotlin-dsl`
}

dependencies {
    compileOnly(libs.kotlin.gradle.plugin)
    compileOnly(libs.compose.gradle.plugin)
    compileOnly(libs.android.gradle.plugin)
}

// Replace {package_base} below with your actual root package (e.g., com.example.myapp)
gradlePlugin {
    plugins {
        register("kmpLibrary") {
            id = "{package_base}.kmp.library"
            implementationClass = "KmpLibraryConventionPlugin"
        }
        register("composeFeature") {
            id = "{package_base}.compose.feature"
            implementationClass = "ComposeFeatureConventionPlugin"
        }
        register("androidApp") {
            id = "{package_base}.android.app"
            implementationClass = "AndroidAppConventionPlugin"
        }
    }
}
```

---

## Shared Constants (AppConfig.kt)

```kotlin
object AppConfig {
    const val COMPILE_SDK = 36
    const val MIN_SDK = 24
    const val TARGET_SDK = 36
    const val JVM_TARGET = "17"
}
```

---

## KmpLibraryConventionPlugin

Shared KMP + Android library configuration:

```kotlin
import com.android.build.gradle.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.jetbrains.kotlin.gradle.dsl.KotlinMultiplatformExtension
import org.jetbrains.kotlin.gradle.dsl.JvmTarget

class KmpLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        with(pluginManager) {
            apply("org.jetbrains.kotlin.multiplatform")
            apply("com.android.library")
        }

        extensions.configure<KotlinMultiplatformExtension> {
            androidTarget {
                compilations.all {
                    compileTaskProvider.configure {
                        compilerOptions {
                            jvmTarget.set(JvmTarget.fromTarget(AppConfig.JVM_TARGET))
                        }
                    }
                }
            }
            iosX64()
            iosArm64()
            iosSimulatorArm64()
        }

        extensions.configure<LibraryExtension> {
            compileSdk = AppConfig.COMPILE_SDK
            defaultConfig.minSdk = AppConfig.MIN_SDK
            compileOptions {
                sourceCompatibility = JavaVersion.VERSION_17
                targetCompatibility = JavaVersion.VERSION_17
            }
        }
    }
}
```

---

## ComposeFeatureConventionPlugin

KMP library + Compose support:

```kotlin
import com.android.build.gradle.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.jetbrains.kotlin.gradle.dsl.KotlinMultiplatformExtension
import org.jetbrains.kotlin.gradle.dsl.JvmTarget

class ComposeFeatureConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        with(pluginManager) {
            apply("org.jetbrains.kotlin.multiplatform")
            apply("org.jetbrains.kotlin.plugin.compose")
            apply("org.jetbrains.compose")
            apply("com.android.library")
        }

        extensions.configure<KotlinMultiplatformExtension> {
            androidTarget {
                compilations.all {
                    compileTaskProvider.configure {
                        compilerOptions {
                            jvmTarget.set(JvmTarget.fromTarget(AppConfig.JVM_TARGET))
                        }
                    }
                }
            }
            iosX64()
            iosArm64()
            iosSimulatorArm64()
        }

        extensions.configure<LibraryExtension> {
            compileSdk = AppConfig.COMPILE_SDK
            defaultConfig.minSdk = AppConfig.MIN_SDK
            compileOptions {
                sourceCompatibility = JavaVersion.VERSION_17
                targetCompatibility = JavaVersion.VERSION_17
            }
        }
    }
}
```

---

## AndroidAppConventionPlugin

Android app module configuration:

```kotlin
import com.android.build.gradle.internal.dsl.BaseAppModuleExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure

class AndroidAppConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        with(pluginManager) {
            apply("com.android.application")
            apply("org.jetbrains.kotlin.multiplatform")
            apply("org.jetbrains.kotlin.plugin.compose")
            apply("org.jetbrains.compose")
        }

        extensions.configure<BaseAppModuleExtension> {
            compileSdk = AppConfig.COMPILE_SDK
            defaultConfig {
                minSdk = AppConfig.MIN_SDK
                targetSdk = AppConfig.TARGET_SDK
            }
            compileOptions {
                sourceCompatibility = JavaVersion.VERSION_17
                targetCompatibility = JavaVersion.VERSION_17
            }
        }
    }
}
```

---

## Root settings.gradle.kts Integration

```kotlin
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
```

---

## Usage in Feature Modules

Before (verbose, duplicated):
```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.composeMultiplatform)
    alias(libs.plugins.composeCompiler)
    alias(libs.plugins.androidLibrary)
}

kotlin {
    androidTarget {
        compilations.all { /* JVM target config */ }
    }
    iosX64(); iosArm64(); iosSimulatorArm64()
    sourceSets {
        commonMain.dependencies {
            // compose deps + feature-specific deps
        }
    }
}

android {
    namespace = "..."
    compileSdk = 35
    defaultConfig.minSdk = 24
    // ...
}
```

After (convention plugin):
```kotlin
plugins {
    id("{package_base}.compose.feature")
    alias(libs.plugins.kotlinx.serialization)  // only feature-specific plugins
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            // Only feature-specific deps — KMP targets and compose deps come from convention
            implementation(project(":core:feature"))
            implementation(project(":<feature>:data:api"))
            implementation(libs.koin.compose.viewmodel)
            implementation(libs.navigation.compose)
        }
    }
}

android {
    namespace = "{package_base}.<feature>.feature"
}
```

---

## Migration Strategy

1. Create `build-logic/` structure
2. Extract shared config values from existing `build.gradle.kts` files
3. Create convention plugins
4. Migrate one module as proof-of-concept
5. Gradually migrate remaining modules
6. Keep old `build.gradle.kts` patterns working during transition
