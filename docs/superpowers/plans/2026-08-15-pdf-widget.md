# PDF Widget Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** An Android home-screen widget that renders a page of a chosen PDF and acts as the entire reading surface, with button-driven pan, zoom and page turns, per-document position memory, and night mode.

**Architecture:** All navigation maths lives in a pure Kotlin `Viewport` value type with no Android imports, so it carries the test suite. A rendering layer wraps the platform `PdfRenderer` and turns a viewport into a bitmap sized to the widget. A Glance widget composes the two and dispatches button actions. State is persisted as JSON in a Preferences DataStore, keyed by widget ID for widget settings and by document URI for reading positions.

**Tech Stack:** Kotlin, Android Gradle Plugin 8.7.2, Glance for App Widgets 1.1.1, AndroidX DataStore Preferences 1.1.1, kotlinx.serialization 1.7.3, JUnit 4, Robolectric 4.13.

**Spec:** `docs/superpowers/specs/2026-08-15-pdf-widget-design.md`

## Global Constraints

- Package: `com.samoed.pdfwidget`. Application ID identical.
- `minSdk = 31`, `compileSdk = 36`, `targetSdk = 36`, JVM target 17.
- Widget receives clicks only. Never add touch, drag, or gesture handling.
- No comments and no KDoc in any source file.
- No string comparison for known value sets — use enums (`WidgetAction`, `PanDirection`, `ColorMode`).
- No literal numbers at call sites. Every tunable lives in `ViewportConfig` or `RenderConfig`.
- Functions stay under ~20 lines; classes stay between 50 and 250 lines.
- Zoom ladder: `1.0, 1.5, 2.0, 3.0, 4.0` as multipliers of fit-to-widget scale.
- Pan step: `0.85` of the visible fraction along the panned axis.
- Offsets are normalised `0..1` of page dimensions, never pixels.
- `PdfRenderer` access is serialised through `RenderScheduler`; never open two pages at once.

---

### Task 1: Project scaffold and an empty widget on the home screen

**Files:**
- Create: `settings.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml`, `gradle.properties`
- Create: `app/build.gradle.kts`
- Create: `app/src/main/AndroidManifest.xml`
- Create: `app/src/main/res/xml/pdf_widget_info.xml`
- Create: `app/src/main/res/values/strings.xml`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidget.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidgetReceiver.kt`

**Interfaces:**
- Consumes: nothing.
- Produces: `PdfWidget : GlanceAppWidget` and `PdfWidgetReceiver : GlanceAppWidgetReceiver`, both in package `com.samoed.pdfwidget.widget`. Later tasks replace the body of `PdfWidget.provideGlance`.

- [ ] **Step 1: Create the Gradle version catalog**

`gradle/libs.versions.toml`:

```toml
[versions]
agp = "8.7.2"
kotlin = "2.0.21"
glance = "1.1.1"
datastore = "1.1.1"
serialization = "1.7.3"
coroutines = "1.9.0"
core-ktx = "1.15.0"
junit = "4.13.2"
robolectric = "4.13"

[libraries]
androidx-core-ktx = { module = "androidx.core:core-ktx", version.ref = "core-ktx" }
androidx-glance-appwidget = { module = "androidx.glance:glance-appwidget", version.ref = "glance" }
androidx-glance-material3 = { module = "androidx.glance:glance-material3", version.ref = "glance" }
androidx-datastore-preferences = { module = "androidx.datastore:datastore-preferences", version.ref = "datastore" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "serialization" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutines" }
junit = { module = "junit:junit", version.ref = "junit" }
robolectric = { module = "org.robolectric:robolectric", version.ref = "robolectric" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

- [ ] **Step 2: Create the root Gradle files**

`settings.gradle.kts`:

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "pdf-widget"
include(":app")
```

`build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.compose) apply false
    alias(libs.plugins.kotlin.serialization) apply false
}
```

`gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx2048m
android.useAndroidX=true
kotlin.code.style=official
```

- [ ] **Step 3: Create the app module build file**

`app/build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.kotlin.serialization)
}

android {
    namespace = "com.samoed.pdfwidget"
    compileSdk = 36

    defaultConfig {
        applicationId = "com.samoed.pdfwidget"
        minSdk = 31
        targetSdk = 36
        versionCode = 1
        versionName = "0.1"
    }

    buildFeatures {
        compose = true
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    sourceSets["main"].kotlin.srcDir("src/main/kotlin")
    sourceSets["test"].kotlin.srcDir("src/test/kotlin")

    testOptions {
        unitTests.isIncludeAndroidResources = true
    }
}

dependencies {
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.glance.appwidget)
    implementation(libs.androidx.glance.material3)
    implementation(libs.androidx.datastore.preferences)
    implementation(libs.kotlinx.serialization.json)
    testImplementation(libs.junit)
    testImplementation(libs.robolectric)
    testImplementation(libs.kotlinx.coroutines.test)
}
```

- [ ] **Step 4: Create the widget metadata and strings**

`app/src/main/res/xml/pdf_widget_info.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="250dp"
    android:minHeight="250dp"
    android:targetCellWidth="4"
    android:targetCellHeight="5"
    android:resizeMode="horizontal|vertical"
    android:updatePeriodMillis="0"
    android:widgetCategory="home_screen"
    android:description="@string/widget_description" />
```

`app/src/main/res/values/strings.xml`:

```xml
<resources>
    <string name="app_name">PDF Widget</string>
    <string name="widget_description">Read a PDF on your home screen</string>
    <string name="no_document">Tap to choose a PDF</string>
    <string name="document_unavailable">Document unavailable. Tap to reselect.</string>
</resources>
```

- [ ] **Step 5: Create the widget and receiver**

`app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidget.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import android.content.Context
import androidx.glance.GlanceId
import androidx.glance.appwidget.GlanceAppWidget
import androidx.glance.appwidget.provideContent
import androidx.glance.text.Text

class PdfWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent { Text("PDF Widget") }
    }
}
```

`app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidgetReceiver.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import androidx.glance.appwidget.GlanceAppWidget
import androidx.glance.appwidget.GlanceAppWidgetReceiver

class PdfWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = PdfWidget()
}
```

- [ ] **Step 6: Create the manifest**

`app/src/main/AndroidManifest.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application
        android:allowBackup="true"
        android:label="@string/app_name">
        <receiver
            android:name=".widget.PdfWidgetReceiver"
            android:exported="true">
            <intent-filter>
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
            </intent-filter>
            <meta-data
                android:name="android.appwidget.provider"
                android:resource="@xml/pdf_widget_info" />
        </receiver>
    </application>
</manifest>
```

- [ ] **Step 7: Verify the build**

Run: `./gradlew assembleDebug`
Expected: BUILD SUCCESSFUL.

- [ ] **Step 8: Verify manually on a device**

Run: `./gradlew installDebug`
Then long-press the home screen, add the "PDF Widget" widget.
Expected: a widget appears showing the text "PDF Widget".

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat: scaffold project with an empty Glance widget"
```

---

### Task 2: Viewport navigation maths

This is the heart of the app and the only module with exhaustive tests. It has no Android imports and runs on the plain JVM.

**Files:**
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/core/ViewportConfig.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/core/PanDirection.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/core/NormalizedRegion.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/core/Viewport.kt`
- Test: `app/src/test/kotlin/com/samoed/pdfwidget/core/ViewportTest.kt`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `enum class PanDirection { UP, DOWN, LEFT, RIGHT }`
  - `data class NormalizedRegion(val left: Float, val top: Float, val right: Float, val bottom: Float)`
  - `@Serializable data class Viewport(val pageIndex: Int = 0, val zoomStopIndex: Int = 0, val offsetX: Float = 0f, val offsetY: Float = 0f)`
  - Members: `fun visibleFraction(): Float`, `fun maxOffset(): Float`, `fun zoomedIn(): Viewport`, `fun zoomedOut(): Viewport`, `fun panned(direction: PanDirection, pageCount: Int): Viewport`, `fun atNextPage(pageCount: Int): Viewport`, `fun atPreviousPage(): Viewport`, `fun sourceRegion(): NormalizedRegion`
  - `object ViewportConfig` with `ZOOM_STOPS: List<Float>`, `PAN_STEP_FRACTION: Float`, `FIRST_PAGE_INDEX: Int`

- [ ] **Step 1: Write the failing tests**

`app/src/test/kotlin/com/samoed/pdfwidget/core/ViewportTest.kt`:

```kotlin
package com.samoed.pdfwidget.core

import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Test

private const val PAGE_COUNT = 10
private const val TOLERANCE = 0.0001f

class ViewportTest {

    @Test
    fun `fit zoom shows the whole page`() {
        val viewport = Viewport()
        assertEquals(1f, viewport.visibleFraction(), TOLERANCE)
        assertEquals(0f, viewport.maxOffset(), TOLERANCE)
    }

    @Test
    fun `zooming in halves the visible fraction at the second stop`() {
        val viewport = Viewport().zoomedIn()
        assertEquals(1f / ViewportConfig.ZOOM_STOPS[1], viewport.visibleFraction(), TOLERANCE)
    }

    @Test
    fun `zooming in stops at the top of the ladder`() {
        var viewport = Viewport()
        repeat(ViewportConfig.ZOOM_STOPS.size + 3) { viewport = viewport.zoomedIn() }
        assertEquals(ViewportConfig.ZOOM_STOPS.lastIndex, viewport.zoomStopIndex)
    }

    @Test
    fun `zooming out stops at fit`() {
        var viewport = Viewport(zoomStopIndex = 2)
        repeat(5) { viewport = viewport.zoomedOut() }
        assertEquals(0, viewport.zoomStopIndex)
    }

    @Test
    fun `zooming preserves the centre of the view`() {
        val zoomed = Viewport(zoomStopIndex = 3, offsetX = 0.4f, offsetY = 0.4f)
        val centreBefore = zoomed.offsetY + zoomed.visibleFraction() / 2
        val after = zoomed.zoomedOut()
        val centreAfter = after.offsetY + after.visibleFraction() / 2
        assertEquals(centreBefore, centreAfter, TOLERANCE)
    }

    @Test
    fun `zooming clamps the offset back inside the page`() {
        val viewport = Viewport(zoomStopIndex = 4, offsetX = 0.7f, offsetY = 0.7f).zoomedOut()
        assertTrue(viewport.offsetX <= viewport.maxOffset() + TOLERANCE)
        assertTrue(viewport.offsetY <= viewport.maxOffset() + TOLERANCE)
    }

    @Test
    fun `panning down at fit zoom turns the page`() {
        val viewport = Viewport(pageIndex = 2).panned(PanDirection.DOWN, PAGE_COUNT)
        assertEquals(3, viewport.pageIndex)
        assertEquals(0f, viewport.offsetY, TOLERANCE)
    }

    @Test
    fun `panning down while zoomed moves within the page`() {
        val viewport = Viewport(pageIndex = 2, zoomStopIndex = 2).panned(PanDirection.DOWN, PAGE_COUNT)
        assertEquals(2, viewport.pageIndex)
        assertTrue(viewport.offsetY > 0f)
    }

    @Test
    fun `panning down overlaps the previous view`() {
        val viewport = Viewport(zoomStopIndex = 2)
        val panned = viewport.panned(PanDirection.DOWN, PAGE_COUNT)
        val expected = viewport.visibleFraction() * ViewportConfig.PAN_STEP_FRACTION
        assertEquals(expected, panned.offsetY, TOLERANCE)
    }

    @Test
    fun `panning down at the bottom edge rolls onto the next page`() {
        val viewport = Viewport(pageIndex = 1, zoomStopIndex = 2)
        val atBottom = viewport.copy(offsetY = viewport.maxOffset())
        val rolled = atBottom.panned(PanDirection.DOWN, PAGE_COUNT)
        assertEquals(2, rolled.pageIndex)
        assertEquals(0f, rolled.offsetY, TOLERANCE)
    }

    @Test
    fun `panning up at the top edge rolls back to the bottom of the previous page`() {
        val viewport = Viewport(pageIndex = 1, zoomStopIndex = 2, offsetY = 0f)
        val rolled = viewport.panned(PanDirection.UP, PAGE_COUNT)
        assertEquals(0, rolled.pageIndex)
        assertEquals(viewport.maxOffset(), rolled.offsetY, TOLERANCE)
    }

    @Test
    fun `rollover preserves the horizontal offset`() {
        val viewport = Viewport(pageIndex = 1, zoomStopIndex = 2, offsetX = 0.3f)
        val rolled = viewport.copy(offsetY = viewport.maxOffset()).panned(PanDirection.DOWN, PAGE_COUNT)
        assertEquals(0.3f, rolled.offsetX, TOLERANCE)
    }

    @Test
    fun `panning down on the last page is a no-op`() {
        val viewport = Viewport(pageIndex = PAGE_COUNT - 1)
        assertEquals(viewport, viewport.panned(PanDirection.DOWN, PAGE_COUNT))
    }

    @Test
    fun `panning up on the first page is a no-op`() {
        val viewport = Viewport(pageIndex = 0)
        assertEquals(viewport, viewport.panned(PanDirection.UP, PAGE_COUNT))
    }

    @Test
    fun `horizontal panning never changes the page`() {
        val viewport = Viewport(pageIndex = 5, zoomStopIndex = 2, offsetX = 0f)
        val left = viewport.panned(PanDirection.LEFT, PAGE_COUNT)
        val right = viewport.copy(offsetX = viewport.maxOffset()).panned(PanDirection.RIGHT, PAGE_COUNT)
        assertEquals(5, left.pageIndex)
        assertEquals(5, right.pageIndex)
    }

    @Test
    fun `horizontal panning clamps at both edges`() {
        val viewport = Viewport(zoomStopIndex = 2)
        assertEquals(0f, viewport.panned(PanDirection.LEFT, PAGE_COUNT).offsetX, TOLERANCE)
        val atRight = viewport.copy(offsetX = viewport.maxOffset())
        assertEquals(viewport.maxOffset(), atRight.panned(PanDirection.RIGHT, PAGE_COUNT).offsetX, TOLERANCE)
    }

    @Test
    fun `next page resets the vertical offset and keeps zoom and horizontal offset`() {
        val viewport = Viewport(pageIndex = 1, zoomStopIndex = 3, offsetX = 0.2f, offsetY = 0.5f)
        val next = viewport.atNextPage(PAGE_COUNT)
        assertEquals(2, next.pageIndex)
        assertEquals(0f, next.offsetY, TOLERANCE)
        assertEquals(0.2f, next.offsetX, TOLERANCE)
        assertEquals(3, next.zoomStopIndex)
    }

    @Test
    fun `previous page resets the vertical offset`() {
        val viewport = Viewport(pageIndex = 4, zoomStopIndex = 2, offsetY = 0.4f)
        val previous = viewport.atPreviousPage()
        assertEquals(3, previous.pageIndex)
        assertEquals(0f, previous.offsetY, TOLERANCE)
    }

    @Test
    fun `page turns are no-ops at the document boundaries`() {
        val last = Viewport(pageIndex = PAGE_COUNT - 1)
        assertEquals(last, last.atNextPage(PAGE_COUNT))
        val first = Viewport(pageIndex = 0)
        assertEquals(first, first.atPreviousPage())
    }

    @Test
    fun `source region covers the whole page at fit zoom`() {
        val region = Viewport().sourceRegion()
        assertEquals(0f, region.left, TOLERANCE)
        assertEquals(0f, region.top, TOLERANCE)
        assertEquals(1f, region.right, TOLERANCE)
        assertEquals(1f, region.bottom, TOLERANCE)
    }

    @Test
    fun `source region tracks the offset when zoomed`() {
        val viewport = Viewport(zoomStopIndex = 1, offsetX = 0.1f, offsetY = 0.2f)
        val region = viewport.sourceRegion()
        assertEquals(0.1f, region.left, TOLERANCE)
        assertEquals(0.2f, region.top, TOLERANCE)
        assertEquals(0.1f + viewport.visibleFraction(), region.right, TOLERANCE)
    }
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `./gradlew :app:testDebugUnitTest --tests "*ViewportTest*"`
Expected: FAIL — unresolved references `Viewport`, `PanDirection`, `ViewportConfig`.

- [ ] **Step 3: Write the supporting types**

`app/src/main/kotlin/com/samoed/pdfwidget/core/ViewportConfig.kt`:

```kotlin
package com.samoed.pdfwidget.core

object ViewportConfig {
    val ZOOM_STOPS = listOf(1.0f, 1.5f, 2.0f, 3.0f, 4.0f)
    const val PAN_STEP_FRACTION = 0.85f
    const val FIRST_PAGE_INDEX = 0
    const val MIN_OFFSET = 0f
    const val WHOLE_PAGE = 1f
    const val EDGE_TOLERANCE = 0.0001f
}
```

`app/src/main/kotlin/com/samoed/pdfwidget/core/PanDirection.kt`:

```kotlin
package com.samoed.pdfwidget.core

enum class PanDirection { UP, DOWN, LEFT, RIGHT }
```

`app/src/main/kotlin/com/samoed/pdfwidget/core/NormalizedRegion.kt`:

```kotlin
package com.samoed.pdfwidget.core

data class NormalizedRegion(
    val left: Float,
    val top: Float,
    val right: Float,
    val bottom: Float,
) {
    val width: Float get() = right - left
    val height: Float get() = bottom - top
}
```

- [ ] **Step 4: Write the Viewport implementation**

`app/src/main/kotlin/com/samoed/pdfwidget/core/Viewport.kt`:

```kotlin
package com.samoed.pdfwidget.core

import kotlinx.serialization.Serializable

@Serializable
data class Viewport(
    val pageIndex: Int = ViewportConfig.FIRST_PAGE_INDEX,
    val zoomStopIndex: Int = 0,
    val offsetX: Float = ViewportConfig.MIN_OFFSET,
    val offsetY: Float = ViewportConfig.MIN_OFFSET,
) {
    fun visibleFraction(): Float = ViewportConfig.WHOLE_PAGE / ViewportConfig.ZOOM_STOPS[zoomStopIndex]

    fun maxOffset(): Float = ViewportConfig.WHOLE_PAGE - visibleFraction()

    fun zoomedIn(): Viewport = atZoomStop(zoomStopIndex + 1)

    fun zoomedOut(): Viewport = atZoomStop(zoomStopIndex - 1)

    fun atNextPage(pageCount: Int): Viewport =
        if (pageIndex + 1 >= pageCount) this
        else copy(pageIndex = pageIndex + 1, offsetY = ViewportConfig.MIN_OFFSET)

    fun atPreviousPage(): Viewport =
        if (pageIndex <= ViewportConfig.FIRST_PAGE_INDEX) this
        else copy(pageIndex = pageIndex - 1, offsetY = ViewportConfig.MIN_OFFSET)

    fun panned(direction: PanDirection, pageCount: Int): Viewport = when (direction) {
        PanDirection.LEFT -> copy(offsetX = clamp(offsetX - panStep()))
        PanDirection.RIGHT -> copy(offsetX = clamp(offsetX + panStep()))
        PanDirection.UP -> pannedUp()
        PanDirection.DOWN -> pannedDown(pageCount)
    }

    fun sourceRegion(): NormalizedRegion = NormalizedRegion(
        left = offsetX,
        top = offsetY,
        right = offsetX + visibleFraction(),
        bottom = offsetY + visibleFraction(),
    )

    private fun pannedUp(): Viewport =
        if (offsetY > ViewportConfig.MIN_OFFSET + ViewportConfig.EDGE_TOLERANCE) {
            copy(offsetY = clamp(offsetY - panStep()))
        } else {
            atPreviousPage().let { if (it == this) this else it.copy(offsetY = it.maxOffset()) }
        }

    private fun pannedDown(pageCount: Int): Viewport =
        if (offsetY < maxOffset() - ViewportConfig.EDGE_TOLERANCE) {
            copy(offsetY = clamp(offsetY + panStep()))
        } else {
            atNextPage(pageCount)
        }

    private fun atZoomStop(target: Int): Viewport {
        val index = target.coerceIn(0, ViewportConfig.ZOOM_STOPS.lastIndex)
        if (index == zoomStopIndex) return this
        val previousFraction = visibleFraction()
        val zoomed = copy(zoomStopIndex = index)
        return zoomed.copy(
            offsetX = zoomed.recentred(offsetX, previousFraction),
            offsetY = zoomed.recentred(offsetY, previousFraction),
        )
    }

    private fun recentred(previousOffset: Float, previousFraction: Float): Float {
        val centre = previousOffset + previousFraction / 2
        return clamp(centre - visibleFraction() / 2)
    }

    private fun panStep(): Float = visibleFraction() * ViewportConfig.PAN_STEP_FRACTION

    private fun clamp(value: Float): Float = value.coerceIn(ViewportConfig.MIN_OFFSET, maxOffset())
}
```

`recentred` is called on the already-zoomed copy, so `visibleFraction()` is the new fraction and the old one is passed in explicitly.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `./gradlew :app:testDebugUnitTest --tests "*ViewportTest*"`
Expected: all `ViewportTest` tests PASS.

- [ ] **Step 6: Commit**

```bash
git add app/src/main/kotlin/com/samoed/pdfwidget/core app/src/test/kotlin/com/samoed/pdfwidget/core
git commit -m "feat: add pure viewport navigation maths with rollover"
```

---

### Task 3: Persisted widget and document state

**Files:**
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/data/ColorMode.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/data/WidgetState.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/data/WidgetStateStore.kt`
- Test: `app/src/test/kotlin/com/samoed/pdfwidget/data/WidgetStateStoreTest.kt`

**Interfaces:**
- Consumes: `Viewport` from Task 2.
- Produces:
  - `enum class ColorMode { SYSTEM, LIGHT, NIGHT }`
  - `@Serializable data class WidgetState(val documentUri: String? = null, val controlsExpanded: Boolean = false, val colorMode: ColorMode = ColorMode.SYSTEM)`
  - `class WidgetStateStore(context: Context)` with `suspend fun widgetState(widgetId: Int): WidgetState`, `suspend fun updateWidget(widgetId: Int, transform: (WidgetState) -> WidgetState)`, `suspend fun position(documentUri: String): Viewport`, `suspend fun savePosition(documentUri: String, viewport: Viewport)`, `suspend fun removeWidget(widgetId: Int)`

- [ ] **Step 1: Write the failing test**

`app/src/test/kotlin/com/samoed/pdfwidget/data/WidgetStateStoreTest.kt`:

```kotlin
package com.samoed.pdfwidget.data

import androidx.test.core.app.ApplicationProvider
import com.samoed.pdfwidget.core.Viewport
import kotlinx.coroutines.test.runTest
import org.junit.Assert.assertEquals
import org.junit.Assert.assertNull
import org.junit.Test
import org.junit.runner.RunWith
import org.robolectric.RobolectricTestRunner

private const val FIRST_WIDGET = 101
private const val SECOND_WIDGET = 202
private const val DOCUMENT = "content://docs/book.pdf"
private const val OTHER_DOCUMENT = "content://docs/other.pdf"

@RunWith(RobolectricTestRunner::class)
class WidgetStateStoreTest {

    private fun store() = WidgetStateStore(ApplicationProvider.getApplicationContext())

    @Test
    fun `an unknown widget has default state`() = runTest {
        val state = store().widgetState(FIRST_WIDGET)
        assertNull(state.documentUri)
        assertEquals(ColorMode.SYSTEM, state.colorMode)
        assertEquals(false, state.controlsExpanded)
    }

    @Test
    fun `widget state round-trips`() = runTest {
        val store = store()
        store.updateWidget(FIRST_WIDGET) { it.copy(documentUri = DOCUMENT, controlsExpanded = true) }
        assertEquals(DOCUMENT, store.widgetState(FIRST_WIDGET).documentUri)
        assertEquals(true, store.widgetState(FIRST_WIDGET).controlsExpanded)
    }

    @Test
    fun `widgets are independent of one another`() = runTest {
        val store = store()
        store.updateWidget(FIRST_WIDGET) { it.copy(documentUri = DOCUMENT) }
        store.updateWidget(SECOND_WIDGET) { it.copy(documentUri = OTHER_DOCUMENT) }
        assertEquals(DOCUMENT, store.widgetState(FIRST_WIDGET).documentUri)
        assertEquals(OTHER_DOCUMENT, store.widgetState(SECOND_WIDGET).documentUri)
    }

    @Test
    fun `an unknown document starts at the first page`() = runTest {
        assertEquals(Viewport(), store().position(DOCUMENT))
    }

    @Test
    fun `positions round-trip and are keyed by document`() = runTest {
        val store = store()
        val viewport = Viewport(pageIndex = 7, zoomStopIndex = 2, offsetY = 0.25f)
        store.savePosition(DOCUMENT, viewport)
        assertEquals(viewport, store.position(DOCUMENT))
        assertEquals(Viewport(), store.position(OTHER_DOCUMENT))
    }

    @Test
    fun `removing a widget keeps document positions`() = runTest {
        val store = store()
        store.updateWidget(FIRST_WIDGET) { it.copy(documentUri = DOCUMENT) }
        store.savePosition(DOCUMENT, Viewport(pageIndex = 3))
        store.removeWidget(FIRST_WIDGET)
        assertNull(store.widgetState(FIRST_WIDGET).documentUri)
        assertEquals(3, store.position(DOCUMENT).pageIndex)
    }
}
```

- [ ] **Step 2: Add the Robolectric test dependency and run to verify failure**

Add to `app/build.gradle.kts` dependencies:

```kotlin
    testImplementation("androidx.test:core:1.6.1")
```

Run: `./gradlew :app:testDebugUnitTest --tests "*WidgetStateStoreTest*"`
Expected: FAIL — unresolved references `WidgetStateStore`, `ColorMode`.

- [ ] **Step 3: Write the state types**

`app/src/main/kotlin/com/samoed/pdfwidget/data/ColorMode.kt`:

```kotlin
package com.samoed.pdfwidget.data

enum class ColorMode { SYSTEM, LIGHT, NIGHT }
```

`app/src/main/kotlin/com/samoed/pdfwidget/data/WidgetState.kt`:

```kotlin
package com.samoed.pdfwidget.data

import com.samoed.pdfwidget.core.Viewport
import kotlinx.serialization.Serializable

@Serializable
data class WidgetState(
    val documentUri: String? = null,
    val controlsExpanded: Boolean = false,
    val colorMode: ColorMode = ColorMode.SYSTEM,
)

@Serializable
data class StoreContents(
    val widgets: Map<String, WidgetState> = emptyMap(),
    val positions: Map<String, Viewport> = emptyMap(),
)
```

- [ ] **Step 4: Write the store**

`app/src/main/kotlin/com/samoed/pdfwidget/data/WidgetStateStore.kt`:

```kotlin
package com.samoed.pdfwidget.data

import android.content.Context
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import com.samoed.pdfwidget.core.Viewport
import kotlinx.coroutines.flow.first
import kotlinx.serialization.json.Json

private const val STORE_NAME = "pdf_widget_state"
private val CONTENTS_KEY = stringPreferencesKey("contents")

private val Context.stateDataStore by preferencesDataStore(name = STORE_NAME)

class WidgetStateStore(private val context: Context) {

    private val json = Json { ignoreUnknownKeys = true }

    suspend fun widgetState(widgetId: Int): WidgetState =
        contents().widgets[widgetId.toString()] ?: WidgetState()

    suspend fun updateWidget(widgetId: Int, transform: (WidgetState) -> WidgetState) {
        mutate { current ->
            val existing = current.widgets[widgetId.toString()] ?: WidgetState()
            current.copy(widgets = current.widgets + (widgetId.toString() to transform(existing)))
        }
    }

    suspend fun removeWidget(widgetId: Int) {
        mutate { it.copy(widgets = it.widgets - widgetId.toString()) }
    }

    suspend fun position(documentUri: String): Viewport =
        contents().positions[documentUri] ?: Viewport()

    suspend fun savePosition(documentUri: String, viewport: Viewport) {
        mutate { it.copy(positions = it.positions + (documentUri to viewport)) }
    }

    private suspend fun contents(): StoreContents = decode(context.stateDataStore.data.first())

    private suspend fun mutate(transform: (StoreContents) -> StoreContents) {
        context.stateDataStore.edit { preferences ->
            preferences[CONTENTS_KEY] = json.encodeToString(transform(decode(preferences)))
        }
    }

    private fun decode(preferences: Preferences): StoreContents =
        preferences[CONTENTS_KEY]?.let { json.decodeFromString<StoreContents>(it) } ?: StoreContents()
}
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `./gradlew :app:testDebugUnitTest --tests "*WidgetStateStoreTest*"`
Expected: PASS.

If tests interfere via a shared DataStore file, add to the test class:

```kotlin
    @After
    fun clearStore() {
        ApplicationProvider.getApplicationContext<android.content.Context>()
            .filesDir.parentFile?.resolve("files/datastore")?.deleteRecursively()
    }
```

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: persist widget settings and per-document reading positions"
```

---

### Task 4: PDF rendering

**Files:**
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/render/RenderConfig.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/render/RenderGeometry.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/render/RenderRequest.kt` (also holds `PageRenderer` and `DocumentUnavailableException`)
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/render/NightFilter.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/render/DocumentSource.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/render/PdfPageRenderer.kt`
- Test: `app/src/test/kotlin/com/samoed/pdfwidget/render/RenderGeometryTest.kt`
- Test: `app/src/androidTest/kotlin/com/samoed/pdfwidget/render/PdfPageRendererTest.kt`
- Create: `app/src/androidTest/assets/fixture.pdf`

**Interfaces:**
- Consumes: `Viewport`, `NormalizedRegion` from Task 2; `ColorMode` from Task 3.
- Produces:
  - `data class RenderRequest(val documentUri: String, val viewport: Viewport, val widthPx: Int, val heightPx: Int, val night: Boolean)`
  - `interface PageRenderer { suspend fun pageCount(documentUri: String): Int; suspend fun render(request: RenderRequest): Bitmap }`
  - `class PdfPageRenderer(context: Context) : PageRenderer`
  - `object RenderGeometry { fun transform(region: NormalizedRegion, pageWidth: Float, pageHeight: Float, targetWidth: Int, targetHeight: Int): RenderTransform }`
  - `data class RenderTransform(val scale: Float, val translateX: Float, val translateY: Float)`
  - `class DocumentUnavailableException(cause: Throwable) : Exception(cause)`

The geometry is separated from the Android rendering call so the aspect-ratio maths is testable on the JVM.

- [ ] **Step 1: Write the failing geometry test**

`app/src/test/kotlin/com/samoed/pdfwidget/render/RenderGeometryTest.kt`:

```kotlin
package com.samoed.pdfwidget.render

import com.samoed.pdfwidget.core.NormalizedRegion
import org.junit.Assert.assertEquals
import org.junit.Test

private const val TOLERANCE = 0.001f
private const val PAGE_WIDTH = 600f
private const val PAGE_HEIGHT = 800f
private val WHOLE_PAGE = NormalizedRegion(0f, 0f, 1f, 1f)

class RenderGeometryTest {

    @Test
    fun `whole page fits the target without distortion`() {
        val transform = RenderGeometry.transform(WHOLE_PAGE, PAGE_WIDTH, PAGE_HEIGHT, 300, 400)
        assertEquals(0.5f, transform.scale, TOLERANCE)
    }

    @Test
    fun `fit uses the limiting axis`() {
        val transform = RenderGeometry.transform(WHOLE_PAGE, PAGE_WIDTH, PAGE_HEIGHT, 600, 400)
        assertEquals(0.5f, transform.scale, TOLERANCE)
    }

    @Test
    fun `content is centred on the unconstrained axis`() {
        val transform = RenderGeometry.transform(WHOLE_PAGE, PAGE_WIDTH, PAGE_HEIGHT, 600, 400)
        assertEquals(150f, transform.translateX, TOLERANCE)
        assertEquals(0f, transform.translateY, TOLERANCE)
    }

    @Test
    fun `a zoomed region scales up and offsets left and up`() {
        val region = NormalizedRegion(0.5f, 0.5f, 1f, 1f)
        val transform = RenderGeometry.transform(region, PAGE_WIDTH, PAGE_HEIGHT, 300, 400)
        assertEquals(1f, transform.scale, TOLERANCE)
        assertEquals(-300f, transform.translateX, TOLERANCE)
        assertEquals(-400f, transform.translateY, TOLERANCE)
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `./gradlew :app:testDebugUnitTest --tests "*RenderGeometryTest*"`
Expected: FAIL — unresolved reference `RenderGeometry`.

- [ ] **Step 3: Write the geometry**

`app/src/main/kotlin/com/samoed/pdfwidget/render/RenderGeometry.kt`:

```kotlin
package com.samoed.pdfwidget.render

import com.samoed.pdfwidget.core.NormalizedRegion
import kotlin.math.min

data class RenderTransform(val scale: Float, val translateX: Float, val translateY: Float)

object RenderGeometry {
    fun transform(
        region: NormalizedRegion,
        pageWidth: Float,
        pageHeight: Float,
        targetWidth: Int,
        targetHeight: Int,
    ): RenderTransform {
        val sourceWidth = region.width * pageWidth
        val sourceHeight = region.height * pageHeight
        val scale = min(targetWidth / sourceWidth, targetHeight / sourceHeight)
        val marginX = (targetWidth - sourceWidth * scale) / 2
        val marginY = (targetHeight - sourceHeight * scale) / 2
        return RenderTransform(
            scale = scale,
            translateX = -region.left * pageWidth * scale + marginX,
            translateY = -region.top * pageHeight * scale + marginY,
        )
    }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `./gradlew :app:testDebugUnitTest --tests "*RenderGeometryTest*"`
Expected: PASS.

- [ ] **Step 5: Write the rendering configuration and request types**

`app/src/main/kotlin/com/samoed/pdfwidget/render/RenderConfig.kt`:

```kotlin
package com.samoed.pdfwidget.render

object RenderConfig {
    const val PAGE_BACKGROUND = android.graphics.Color.WHITE
    const val CACHE_FILE_PREFIX = "document-"
    const val CACHE_FILE_SUFFIX = ".pdf"
    const val COPY_BUFFER_BYTES = 64 * 1024
}
```

`app/src/main/kotlin/com/samoed/pdfwidget/render/RenderRequest.kt`:

```kotlin
package com.samoed.pdfwidget.render

import android.graphics.Bitmap
import com.samoed.pdfwidget.core.Viewport

data class RenderRequest(
    val documentUri: String,
    val viewport: Viewport,
    val widthPx: Int,
    val heightPx: Int,
    val night: Boolean,
)

class DocumentUnavailableException(cause: Throwable) : Exception(cause)

interface PageRenderer {
    suspend fun pageCount(documentUri: String): Int
    suspend fun render(request: RenderRequest): Bitmap
}
```

- [ ] **Step 6: Write the night filter**

`app/src/main/kotlin/com/samoed/pdfwidget/render/NightFilter.kt`:

```kotlin
package com.samoed.pdfwidget.render

import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.ColorMatrix
import android.graphics.ColorMatrixColorFilter
import android.graphics.Paint

private val INVERT = ColorMatrix(
    floatArrayOf(
        -1f, 0f, 0f, 0f, 255f,
        0f, -1f, 0f, 0f, 255f,
        0f, 0f, -1f, 0f, 255f,
        0f, 0f, 0f, 1f, 0f,
    )
)

object NightFilter {
    fun inverted(source: Bitmap): Bitmap {
        val target = Bitmap.createBitmap(source.width, source.height, Bitmap.Config.ARGB_8888)
        val paint = Paint().apply { colorFilter = ColorMatrixColorFilter(INVERT) }
        Canvas(target).drawBitmap(source, 0f, 0f, paint)
        source.recycle()
        return target
    }
}
```

- [ ] **Step 7: Write the document source**

`app/src/main/kotlin/com/samoed/pdfwidget/render/DocumentSource.kt`:

```kotlin
package com.samoed.pdfwidget.render

import android.content.Context
import android.net.Uri
import android.os.ParcelFileDescriptor
import java.io.File

class DocumentSource(private val context: Context) {

    fun descriptor(documentUri: String): ParcelFileDescriptor =
        try {
            openDirect(documentUri) ?: openCopy(documentUri)
        } catch (error: Exception) {
            throw DocumentUnavailableException(error)
        }

    private fun openDirect(documentUri: String): ParcelFileDescriptor? =
        context.contentResolver.openFileDescriptor(Uri.parse(documentUri), "r")

    private fun openCopy(documentUri: String): ParcelFileDescriptor {
        val cached = cacheFile(documentUri)
        if (!cached.exists()) {
            context.contentResolver.openInputStream(Uri.parse(documentUri))!!.use { input ->
                cached.outputStream().use { output ->
                    input.copyTo(output, RenderConfig.COPY_BUFFER_BYTES)
                }
            }
        }
        return ParcelFileDescriptor.open(cached, ParcelFileDescriptor.MODE_READ_ONLY)
    }

    private fun cacheFile(documentUri: String): File = File(
        context.cacheDir,
        RenderConfig.CACHE_FILE_PREFIX + documentUri.hashCode() + RenderConfig.CACHE_FILE_SUFFIX,
    )
}
```

- [ ] **Step 8: Write the renderer**

`app/src/main/kotlin/com/samoed/pdfwidget/render/PdfPageRenderer.kt`:

```kotlin
package com.samoed.pdfwidget.render

import android.content.Context
import android.graphics.Bitmap
import android.graphics.Matrix
import android.graphics.pdf.PdfRenderer
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock
import kotlinx.coroutines.withContext

class PdfPageRenderer(context: Context) : PageRenderer {

    private val source = DocumentSource(context)
    private val mutex = Mutex()

    override suspend fun pageCount(documentUri: String): Int = locked {
        open(documentUri) { it.pageCount }
    }

    override suspend fun render(request: RenderRequest): Bitmap = locked {
        val bitmap = open(request.documentUri) { renderer ->
            renderer.openPage(request.viewport.pageIndex).use { page ->
                drawPage(page, request)
            }
        }
        if (request.night) NightFilter.inverted(bitmap) else bitmap
    }

    private fun drawPage(page: PdfRenderer.Page, request: RenderRequest): Bitmap {
        val transform = RenderGeometry.transform(
            region = request.viewport.sourceRegion(),
            pageWidth = page.width.toFloat(),
            pageHeight = page.height.toFloat(),
            targetWidth = request.widthPx,
            targetHeight = request.heightPx,
        )
        val bitmap = Bitmap.createBitmap(request.widthPx, request.heightPx, Bitmap.Config.ARGB_8888)
        bitmap.eraseColor(RenderConfig.PAGE_BACKGROUND)
        val matrix = Matrix().apply {
            setScale(transform.scale, transform.scale)
            postTranslate(transform.translateX, transform.translateY)
        }
        page.render(bitmap, null, matrix, PdfRenderer.Page.RENDER_MODE_FOR_DISPLAY)
        return bitmap
    }

    private fun <T> open(documentUri: String, block: (PdfRenderer) -> T): T =
        source.descriptor(documentUri).use { descriptor ->
            PdfRenderer(descriptor).use(block)
        }

    private suspend fun <T> locked(block: suspend () -> T): T =
        withContext(Dispatchers.IO) { mutex.withLock { block() } }
}
```

- [ ] **Step 9: Add an instrumented test against a fixture PDF**

`PdfRenderer` is backed by native code and does not work under Robolectric, so this test runs on a device or emulator.

Add to `app/build.gradle.kts`:

```kotlin
android {
    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
    sourceSets["androidTest"].kotlin.srcDir("src/androidTest/kotlin")
}

dependencies {
    androidTestImplementation("androidx.test.ext:junit:1.2.1")
    androidTestImplementation("androidx.test:runner:1.6.2")
}
```

Generate a two-page fixture and place it at `app/src/androidTest/assets/fixture.pdf`:

```bash
printf 'Page one\n\f\nPage two\n' | libreoffice --headless --convert-to pdf --outdir app/src/androidTest/assets /dev/stdin
```

If LibreOffice is unavailable, use any two-page PDF you already have and rename it `fixture.pdf`.

`app/src/androidTest/kotlin/com/samoed/pdfwidget/render/PdfPageRendererTest.kt`:

```kotlin
package com.samoed.pdfwidget.render

import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.test.platform.app.InstrumentationRegistry
import com.samoed.pdfwidget.core.Viewport
import kotlinx.coroutines.runBlocking
import org.junit.Assert.assertEquals
import org.junit.Assert.assertNotEquals
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Test
import org.junit.runner.RunWith
import java.io.File

private const val FIXTURE = "fixture.pdf"
private const val TARGET_WIDTH = 400
private const val TARGET_HEIGHT = 500

@RunWith(AndroidJUnit4::class)
class PdfPageRendererTest {

    private lateinit var documentUri: String
    private lateinit var renderer: PdfPageRenderer

    @Before
    fun copyFixture() {
        val context = InstrumentationRegistry.getInstrumentation().targetContext
        val target = File(context.cacheDir, FIXTURE)
        InstrumentationRegistry.getInstrumentation().context.assets.open(FIXTURE).use { input ->
            target.outputStream().use { input.copyTo(it) }
        }
        documentUri = android.net.Uri.fromFile(target).toString()
        renderer = PdfPageRenderer(context)
    }

    private fun request(viewport: Viewport, night: Boolean = false) = RenderRequest(
        documentUri = documentUri,
        viewport = viewport,
        widthPx = TARGET_WIDTH,
        heightPx = TARGET_HEIGHT,
        night = night,
    )

    @Test
    fun reportsPageCount() = runBlocking {
        assertTrue(renderer.pageCount(documentUri) >= 1)
    }

    @Test
    fun rendersAtTheRequestedSize() = runBlocking {
        val bitmap = renderer.render(request(Viewport()))
        assertEquals(TARGET_WIDTH, bitmap.width)
        assertEquals(TARGET_HEIGHT, bitmap.height)
    }

    @Test
    fun aZoomedRenderDiffersFromAFitRender() = runBlocking {
        val fit = renderer.render(request(Viewport()))
        val zoomed = renderer.render(request(Viewport(zoomStopIndex = 3)))
        assertNotEquals(fit.getPixel(TARGET_WIDTH / 2, TARGET_HEIGHT / 2), zoomed.getPixel(0, 0))
    }

    @Test
    fun nightModeInvertsThePage() = runBlocking {
        val day = renderer.render(request(Viewport()))
        val night = renderer.render(request(Viewport(), night = true))
        assertNotEquals(day.getPixel(0, 0), night.getPixel(0, 0))
    }
}
```

- [ ] **Step 10: Run the instrumented test**

Run: `./gradlew :app:connectedDebugAndroidTest`
Expected: all four tests PASS. Requires a connected device or running emulator.

- [ ] **Step 11: Verify the build and commit**

Run: `./gradlew :app:assembleDebug :app:testDebugUnitTest`
Expected: BUILD SUCCESSFUL, all tests PASS.

```bash
git add -A
git commit -m "feat: render a viewport of a PDF page to a bitmap"
```

---

### Task 5: Wire the widget to render a hard-coded document

At the end of this task the widget renders a real PDF but has no controls yet. This isolates the rendering integration from the control wiring.

**Files:**
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/widget/WidgetSize.kt`
- Modify: `app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidget.kt`

**Interfaces:**
- Consumes: `WidgetStateStore` (Task 3), `PdfPageRenderer`, `RenderRequest` (Task 4).
- Produces: `object WidgetSize { fun pixels(context: Context, widgetId: Int): Pair<Int, Int> }`; `PdfWidget` gains `private suspend fun renderPage(...)`.

- [ ] **Step 1: Write the widget size helper**

`app/src/main/kotlin/com/samoed/pdfwidget/widget/WidgetSize.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import android.appwidget.AppWidgetManager
import android.content.Context
import android.util.TypedValue

private const val FALLBACK_DP = 250

object WidgetSize {
    fun pixels(context: Context, widgetId: Int): Pair<Int, Int> {
        val options = AppWidgetManager.getInstance(context).getAppWidgetOptions(widgetId)
        val widthDp = options.getInt(AppWidgetManager.OPTION_APPWIDGET_MAX_WIDTH, FALLBACK_DP)
        val heightDp = options.getInt(AppWidgetManager.OPTION_APPWIDGET_MAX_HEIGHT, FALLBACK_DP)
        return toPixels(context, widthDp) to toPixels(context, heightDp)
    }

    private fun toPixels(context: Context, dp: Int): Int = TypedValue.applyDimension(
        TypedValue.COMPLEX_UNIT_DIP,
        dp.toFloat().coerceAtLeast(FALLBACK_DP.toFloat()),
        context.resources.displayMetrics,
    ).toInt()
}
```

- [ ] **Step 2: Replace the widget body**

`app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidget.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import android.content.Context
import android.graphics.Bitmap
import androidx.compose.runtime.Composable
import androidx.glance.GlanceId
import androidx.glance.GlanceModifier
import androidx.glance.Image
import androidx.glance.ImageProvider
import androidx.glance.appwidget.GlanceAppWidget
import androidx.glance.appwidget.GlanceAppWidgetManager
import androidx.glance.appwidget.provideContent
import androidx.glance.layout.Alignment
import androidx.glance.layout.Box
import androidx.glance.layout.ContentScale
import androidx.glance.layout.fillMaxSize
import androidx.glance.text.Text
import com.samoed.pdfwidget.data.ColorMode
import com.samoed.pdfwidget.data.WidgetStateStore
import com.samoed.pdfwidget.render.PdfPageRenderer
import com.samoed.pdfwidget.render.RenderRequest

class PdfWidget : GlanceAppWidget() {

    override suspend fun provideGlance(context: Context, id: GlanceId) {
        val widgetId = GlanceAppWidgetManager(context).getAppWidgetId(id)
        val store = WidgetStateStore(context)
        val state = store.widgetState(widgetId)
        val documentUri = state.documentUri
        val bitmap = documentUri?.let {
            renderPage(context, it, store, widgetId, state.colorMode)
        }
        provideContent { Content(bitmap) }
    }

    @Composable
    private fun Content(bitmap: Bitmap?) {
        Box(modifier = GlanceModifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            if (bitmap == null) {
                Text("Tap to choose a PDF")
            } else {
                Image(
                    provider = ImageProvider(bitmap),
                    contentDescription = null,
                    contentScale = ContentScale.Fit,
                    modifier = GlanceModifier.fillMaxSize(),
                )
            }
        }
    }

    private suspend fun renderPage(
        context: Context,
        documentUri: String,
        store: WidgetStateStore,
        widgetId: Int,
        colorMode: ColorMode,
    ): Bitmap? = runCatching {
        val (width, height) = WidgetSize.pixels(context, widgetId)
        PdfPageRenderer(context).render(
            RenderRequest(
                documentUri = documentUri,
                viewport = store.position(documentUri),
                widthPx = width,
                heightPx = height,
                night = ColorModeResolver.isNight(context, colorMode),
            )
        )
    }.getOrNull()
}
```

- [ ] **Step 3: Resolve SYSTEM against the device theme**

`ColorMode.SYSTEM` must follow the device's dark theme, otherwise the default mode never renders night.

`app/src/main/kotlin/com/samoed/pdfwidget/data/ColorModeResolver.kt`:

```kotlin
package com.samoed.pdfwidget.data

import android.content.Context
import android.content.res.Configuration

object ColorModeResolver {
    fun isNight(context: Context, mode: ColorMode): Boolean = when (mode) {
        ColorMode.NIGHT -> true
        ColorMode.LIGHT -> false
        ColorMode.SYSTEM -> systemIsNight(context)
    }

    private fun systemIsNight(context: Context): Boolean =
        context.resources.configuration.uiMode and Configuration.UI_MODE_NIGHT_MASK ==
            Configuration.UI_MODE_NIGHT_YES
}
```

Add `import com.samoed.pdfwidget.data.ColorModeResolver` to `PdfWidget.kt`.

- [ ] **Step 4: Verify the build**

Run: `./gradlew :app:assembleDebug :app:testDebugUnitTest`
Expected: BUILD SUCCESSFUL, all tests PASS.

The widget still has no way to choose a document, so there is nothing to see on the home screen yet. Visual verification happens in Task 6, which adds the picker.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: render the stored page into the widget"
```

---

### Task 6: Document picking

**Files:**
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/widget/DocumentPickerActivity.kt`
- Create: `app/src/main/res/values/themes.xml`
- Modify: `app/src/main/AndroidManifest.xml`

**Interfaces:**
- Consumes: `WidgetStateStore` (Task 3), `PdfWidget` (Task 5).
- Produces: `DocumentPickerActivity` with `companion object { const val EXTRA_WIDGET_ID = "widget_id"; fun intent(context: Context, widgetId: Int): Intent }`

- [ ] **Step 1: Add a transparent theme**

`app/src/main/res/values/themes.xml`:

```xml
<resources>
    <style name="Theme.Transparent" parent="android:Theme.Material.Light.NoActionBar">
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:windowIsTranslucent">true</item>
        <item name="android:windowNoTitle">true</item>
        <item name="android:windowAnimationStyle">@null</item>
    </style>
</resources>
```

- [ ] **Step 2: Write the picker activity**

`app/src/main/kotlin/com/samoed/pdfwidget/widget/DocumentPickerActivity.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import android.app.Activity
import android.content.Context
import android.content.Intent
import android.net.Uri
import android.os.Bundle
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.glance.appwidget.GlanceAppWidgetManager
import androidx.glance.appwidget.updateAll
import androidx.lifecycle.lifecycleScope
import com.samoed.pdfwidget.data.WidgetStateStore
import kotlinx.coroutines.launch

private val PDF_MIME_TYPES = arrayOf("application/pdf")
private const val NO_WIDGET = -1

class DocumentPickerActivity : AppCompatActivity() {

    private val widgetId: Int
        get() = intent.getIntExtra(EXTRA_WIDGET_ID, NO_WIDGET)

    private val picker = registerForActivityResult(ActivityResultContracts.OpenDocument()) { uri ->
        if (uri == null) finish() else onDocumentPicked(uri)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        if (savedInstanceState == null) picker.launch(PDF_MIME_TYPES)
    }

    private fun onDocumentPicked(uri: Uri) {
        contentResolver.takePersistableUriPermission(uri, Intent.FLAG_GRANT_READ_URI_PERMISSION)
        lifecycleScope.launch {
            WidgetStateStore(applicationContext)
                .updateWidget(widgetId) { it.copy(documentUri = uri.toString()) }
            PdfWidget().updateAll(applicationContext)
            setResult(Activity.RESULT_OK)
            finish()
        }
    }

    companion object {
        const val EXTRA_WIDGET_ID = "widget_id"

        fun intent(context: Context, widgetId: Int): Intent =
            Intent(context, DocumentPickerActivity::class.java)
                .putExtra(EXTRA_WIDGET_ID, widgetId)
                .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK)
    }
}
```

Add `implementation("androidx.appcompat:appcompat:1.7.0")` to `app/build.gradle.kts` dependencies.

- [ ] **Step 3: Register the activity**

Add inside `<application>` in `app/src/main/AndroidManifest.xml`:

```xml
        <activity
            android:name=".widget.DocumentPickerActivity"
            android:theme="@style/Theme.Transparent"
            android:excludeFromRecents="true"
            android:exported="false" />
```

- [ ] **Step 4: Make the empty widget launch the picker**

In `PdfWidget.Content`, wrap the empty-state `Text` with a clickable modifier. Add these imports:

```kotlin
import androidx.glance.action.clickable
import androidx.glance.appwidget.action.actionStartActivity
```

and replace the `Text("Tap to choose a PDF")` call with:

```kotlin
                Text(
                    text = "Tap to choose a PDF",
                    modifier = GlanceModifier.clickable(
                        actionStartActivity(DocumentPickerActivity.intent(context, widgetId))
                    ),
                )
```

This requires passing `context` and `widgetId` into `Content`; change its signature to `Content(context: Context, widgetId: Int, bitmap: Bitmap?)` and update the `provideContent` call accordingly.

- [ ] **Step 5: Verify manually**

Run: `./gradlew installDebug`
Place a fresh widget, tap it, choose a PDF.
Expected: the file picker opens over the home screen; after choosing, the widget shows the first page.

- [ ] **Step 6: Verify persistence across reboot**

Run: `adb reboot`, wait for the device, look at the home screen.
Expected: the widget still renders the page (persistable permission held).

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: pick a document from the widget via the system file picker"
```

---

### Task 7: Controls and action dispatch

**Files:**
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/widget/WidgetAction.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/widget/WidgetActionCallback.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/widget/ActionApplier.kt`
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/widget/ControlOverlay.kt`
- Create: icon drawables under `app/src/main/res/drawable/`
- Modify: `app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidget.kt`
- Test: `app/src/test/kotlin/com/samoed/pdfwidget/widget/ActionApplierTest.kt`

**Interfaces:**
- Consumes: `Viewport`, `PanDirection` (Task 2); `WidgetState`, `ColorMode`, `WidgetStateStore` (Task 3).
- Produces:
  - `enum class WidgetAction { NEXT_PAGE, PREVIOUS_PAGE, ZOOM_IN, ZOOM_OUT, PAN_UP, PAN_DOWN, PAN_LEFT, PAN_RIGHT, TOGGLE_CONTROLS, TOGGLE_NIGHT, PICK_DOCUMENT }`
  - `object ActionApplier { fun applyToViewport(action: WidgetAction, viewport: Viewport, pageCount: Int): Viewport; fun applyToState(action: WidgetAction, state: WidgetState): WidgetState }`
  - `class WidgetActionCallback : ActionCallback` with `companion object { val ACTION_KEY: ActionParameters.Key<String> }`

- [ ] **Step 1: Write the failing action test**

`app/src/test/kotlin/com/samoed/pdfwidget/widget/ActionApplierTest.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import com.samoed.pdfwidget.core.Viewport
import com.samoed.pdfwidget.data.ColorMode
import com.samoed.pdfwidget.data.WidgetState
import org.junit.Assert.assertEquals
import org.junit.Test

private const val PAGE_COUNT = 20

class ActionApplierTest {

    @Test
    fun `next page advances the viewport`() {
        val result = ActionApplier.applyToViewport(WidgetAction.NEXT_PAGE, Viewport(pageIndex = 1), PAGE_COUNT)
        assertEquals(2, result.pageIndex)
    }

    @Test
    fun `previous page rewinds the viewport`() {
        val result = ActionApplier.applyToViewport(WidgetAction.PREVIOUS_PAGE, Viewport(pageIndex = 1), PAGE_COUNT)
        assertEquals(0, result.pageIndex)
    }

    @Test
    fun `zoom in advances the zoom stop`() {
        val result = ActionApplier.applyToViewport(WidgetAction.ZOOM_IN, Viewport(), PAGE_COUNT)
        assertEquals(1, result.zoomStopIndex)
    }

    @Test
    fun `pan down at fit zoom turns the page`() {
        val result = ActionApplier.applyToViewport(WidgetAction.PAN_DOWN, Viewport(pageIndex = 0), PAGE_COUNT)
        assertEquals(1, result.pageIndex)
    }

    @Test
    fun `state-only actions leave the viewport untouched`() {
        val viewport = Viewport(pageIndex = 3, zoomStopIndex = 2)
        assertEquals(viewport, ActionApplier.applyToViewport(WidgetAction.TOGGLE_NIGHT, viewport, PAGE_COUNT))
        assertEquals(viewport, ActionApplier.applyToViewport(WidgetAction.TOGGLE_CONTROLS, viewport, PAGE_COUNT))
    }

    @Test
    fun `toggle controls flips expansion`() {
        val state = WidgetState(controlsExpanded = false)
        assertEquals(true, ActionApplier.applyToState(WidgetAction.TOGGLE_CONTROLS, state).controlsExpanded)
    }

    @Test
    fun `toggle night cycles system to night to light`() {
        var state = WidgetState(colorMode = ColorMode.SYSTEM)
        state = ActionApplier.applyToState(WidgetAction.TOGGLE_NIGHT, state)
        assertEquals(ColorMode.NIGHT, state.colorMode)
        state = ActionApplier.applyToState(WidgetAction.TOGGLE_NIGHT, state)
        assertEquals(ColorMode.LIGHT, state.colorMode)
        state = ActionApplier.applyToState(WidgetAction.TOGGLE_NIGHT, state)
        assertEquals(ColorMode.SYSTEM, state.colorMode)
    }

    @Test
    fun `navigation actions leave widget state untouched`() {
        val state = WidgetState(controlsExpanded = true, colorMode = ColorMode.NIGHT)
        assertEquals(state, ActionApplier.applyToState(WidgetAction.NEXT_PAGE, state))
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `./gradlew :app:testDebugUnitTest --tests "*ActionApplierTest*"`
Expected: FAIL — unresolved references `WidgetAction`, `ActionApplier`.

- [ ] **Step 3: Write the action enum and applier**

`app/src/main/kotlin/com/samoed/pdfwidget/widget/WidgetAction.kt`:

```kotlin
package com.samoed.pdfwidget.widget

enum class WidgetAction {
    NEXT_PAGE,
    PREVIOUS_PAGE,
    ZOOM_IN,
    ZOOM_OUT,
    PAN_UP,
    PAN_DOWN,
    PAN_LEFT,
    PAN_RIGHT,
    TOGGLE_CONTROLS,
    TOGGLE_NIGHT,
    PICK_DOCUMENT,
}
```

`app/src/main/kotlin/com/samoed/pdfwidget/widget/ActionApplier.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import com.samoed.pdfwidget.core.PanDirection
import com.samoed.pdfwidget.core.Viewport
import com.samoed.pdfwidget.data.ColorMode
import com.samoed.pdfwidget.data.WidgetState

object ActionApplier {

    fun applyToViewport(action: WidgetAction, viewport: Viewport, pageCount: Int): Viewport =
        when (action) {
            WidgetAction.NEXT_PAGE -> viewport.atNextPage(pageCount)
            WidgetAction.PREVIOUS_PAGE -> viewport.atPreviousPage()
            WidgetAction.ZOOM_IN -> viewport.zoomedIn()
            WidgetAction.ZOOM_OUT -> viewport.zoomedOut()
            WidgetAction.PAN_UP -> viewport.panned(PanDirection.UP, pageCount)
            WidgetAction.PAN_DOWN -> viewport.panned(PanDirection.DOWN, pageCount)
            WidgetAction.PAN_LEFT -> viewport.panned(PanDirection.LEFT, pageCount)
            WidgetAction.PAN_RIGHT -> viewport.panned(PanDirection.RIGHT, pageCount)
            WidgetAction.TOGGLE_CONTROLS,
            WidgetAction.TOGGLE_NIGHT,
            WidgetAction.PICK_DOCUMENT -> viewport
        }

    fun applyToState(action: WidgetAction, state: WidgetState): WidgetState = when (action) {
        WidgetAction.TOGGLE_CONTROLS -> state.copy(controlsExpanded = !state.controlsExpanded)
        WidgetAction.TOGGLE_NIGHT -> state.copy(colorMode = nextColorMode(state.colorMode))
        else -> state
    }

    private fun nextColorMode(current: ColorMode): ColorMode = when (current) {
        ColorMode.SYSTEM -> ColorMode.NIGHT
        ColorMode.NIGHT -> ColorMode.LIGHT
        ColorMode.LIGHT -> ColorMode.SYSTEM
    }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `./gradlew :app:testDebugUnitTest --tests "*ActionApplierTest*"`
Expected: PASS.

- [ ] **Step 5: Write the action callback**

`app/src/main/kotlin/com/samoed/pdfwidget/widget/WidgetActionCallback.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import android.content.Context
import androidx.glance.GlanceId
import androidx.glance.action.ActionParameters
import androidx.glance.action.actionParametersOf
import androidx.glance.appwidget.GlanceAppWidgetManager
import androidx.glance.appwidget.action.ActionCallback
import androidx.glance.appwidget.action.actionRunCallback
import androidx.glance.appwidget.update
import androidx.glance.action.Action
import com.samoed.pdfwidget.data.WidgetStateStore
import com.samoed.pdfwidget.render.PdfPageRenderer

class WidgetActionCallback : ActionCallback {

    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters,
    ) {
        val action = WidgetAction.valueOf(parameters[ACTION_KEY] ?: return)
        val widgetId = GlanceAppWidgetManager(context).getAppWidgetId(glanceId)
        val store = WidgetStateStore(context)
        val state = store.widgetState(widgetId)
        store.updateWidget(widgetId) { ActionApplier.applyToState(action, it) }
        state.documentUri?.let { uri -> applyNavigation(context, store, uri, action) }
        PdfWidget().update(context, glanceId)
    }

    private suspend fun applyNavigation(
        context: Context,
        store: WidgetStateStore,
        documentUri: String,
        action: WidgetAction,
    ) {
        val pageCount = runCatching { PdfPageRenderer(context).pageCount(documentUri) }.getOrNull() ?: return
        val updated = ActionApplier.applyToViewport(action, store.position(documentUri), pageCount)
        store.savePosition(documentUri, updated)
    }

    companion object {
        val ACTION_KEY = ActionParameters.Key<String>("action")

        fun action(action: WidgetAction): Action = actionRunCallback<WidgetActionCallback>(
            actionParametersOf(ACTION_KEY to action.name)
        )
    }
}
```

- [ ] **Step 6: Add control icons**

Every icon is a 24dp white vector with the same wrapper. Create each file in `app/src/main/res/drawable/` using this template, substituting the path data from the table:

```xml
<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
    <path android:fillColor="#FFFFFFFF" android:pathData="PATH_DATA_HERE" />
</vector>
```

| File | `pathData` |
|---|---|
| `ic_chevron_up.xml` | `M12,7 L20,17 L4,17 Z` |
| `ic_chevron_down.xml` | `M12,17 L4,7 L20,7 Z` |
| `ic_chevron_left.xml` | `M7,12 L17,4 L17,20 Z` |
| `ic_chevron_right.xml` | `M17,12 L7,4 L7,20 Z` |
| `ic_zoom_in.xml` | `M11,4 L13,4 L13,11 L20,11 L20,13 L13,13 L13,20 L11,20 L11,13 L4,13 L4,11 L11,11 Z` |
| `ic_zoom_out.xml` | `M4,11 L20,11 L20,13 L4,13 Z` |
| `ic_page_next.xml` | `M5,4 L15,12 L5,20 Z M17,4 L20,4 L20,20 L17,20 Z` |
| `ic_page_previous.xml` | `M19,4 L9,12 L19,20 Z M4,4 L7,4 L7,20 L4,20 Z` |
| `ic_night.xml` | `M12,3 A9,9 0 0,0 12,21 Z` |
| `ic_document.xml` | `M6,3 L14,3 L18,7 L18,21 L6,21 Z` |
| `ic_controls.xml` | `M6,10 A2,2 0 1,1 6,14 A2,2 0 1,1 6,10 Z M12,10 A2,2 0 1,1 12,14 A2,2 0 1,1 12,10 Z M18,10 A2,2 0 1,1 18,14 A2,2 0 1,1 18,10 Z` |

- [ ] **Step 7: Write the control overlay**

`app/src/main/kotlin/com/samoed/pdfwidget/widget/ControlOverlay.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import androidx.compose.runtime.Composable
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp
import androidx.glance.GlanceModifier
import androidx.glance.Image
import androidx.glance.ImageProvider
import androidx.glance.action.clickable
import androidx.glance.appwidget.cornerRadius
import androidx.glance.background
import androidx.glance.layout.Alignment
import androidx.glance.layout.Column
import androidx.glance.layout.Row
import androidx.glance.layout.padding
import androidx.glance.layout.size
import androidx.glance.unit.ColorProvider
import com.samoed.pdfwidget.R

private val SCRIM = ColorProvider(Color(0x3D000000))
private val BUTTON_SIZE = 40.dp
private val ICON_SIZE = 20.dp
private val CLUSTER_RADIUS = 20.dp
private val CLUSTER_PADDING = 4.dp

@Composable
fun ControlButton(iconRes: Int, action: WidgetAction) {
    Image(
        provider = ImageProvider(iconRes),
        contentDescription = null,
        modifier = GlanceModifier
            .size(BUTTON_SIZE)
            .padding(BUTTON_SIZE.minus(ICON_SIZE).div(2))
            .clickable(WidgetActionCallback.action(action)),
    )
}

@Composable
fun ControlCluster(expanded: Boolean) {
    Column(
        modifier = GlanceModifier
            .background(SCRIM)
            .cornerRadius(CLUSTER_RADIUS)
            .padding(CLUSTER_PADDING),
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        if (expanded) {
            PanPad()
            Row {
                ControlButton(R.drawable.ic_zoom_out, WidgetAction.ZOOM_OUT)
                ControlButton(R.drawable.ic_zoom_in, WidgetAction.ZOOM_IN)
                ControlButton(R.drawable.ic_night, WidgetAction.TOGGLE_NIGHT)
            }
            Row {
                ControlButton(R.drawable.ic_page_previous, WidgetAction.PREVIOUS_PAGE)
                ControlButton(R.drawable.ic_page_next, WidgetAction.NEXT_PAGE)
                ControlButton(R.drawable.ic_document, WidgetAction.PICK_DOCUMENT)
            }
        }
        ControlButton(R.drawable.ic_controls, WidgetAction.TOGGLE_CONTROLS)
    }
}

@Composable
private fun PanPad() {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        ControlButton(R.drawable.ic_chevron_up, WidgetAction.PAN_UP)
        Row {
            ControlButton(R.drawable.ic_chevron_left, WidgetAction.PAN_LEFT)
            ControlButton(R.drawable.ic_chevron_right, WidgetAction.PAN_RIGHT)
        }
        ControlButton(R.drawable.ic_chevron_down, WidgetAction.PAN_DOWN)
    }
}
```

- [ ] **Step 8: Take the pick action as a parameter**

`WidgetAction.PICK_DOCUMENT` cannot go through `WidgetActionCallback` — an `ActionCallback` cannot start an activity and receive its result. The cluster takes the pick action from its caller instead.

In `ControlOverlay.kt`, add these imports:

```kotlin
import androidx.glance.action.Action
```

and replace `ControlCluster` and `ControlButton` with:

```kotlin
@Composable
fun ControlButton(iconRes: Int, action: Action) {
    Image(
        provider = ImageProvider(iconRes),
        contentDescription = null,
        modifier = GlanceModifier
            .size(BUTTON_SIZE)
            .padding(BUTTON_SIZE.minus(ICON_SIZE).div(2))
            .clickable(action),
    )
}

@Composable
fun ControlCluster(expanded: Boolean, pickAction: Action) {
    Column(
        modifier = GlanceModifier
            .background(SCRIM)
            .cornerRadius(CLUSTER_RADIUS)
            .padding(CLUSTER_PADDING),
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        if (expanded) {
            PanPad()
            Row {
                ControlButton(R.drawable.ic_zoom_out, WidgetActionCallback.action(WidgetAction.ZOOM_OUT))
                ControlButton(R.drawable.ic_zoom_in, WidgetActionCallback.action(WidgetAction.ZOOM_IN))
                ControlButton(R.drawable.ic_night, WidgetActionCallback.action(WidgetAction.TOGGLE_NIGHT))
            }
            Row {
                ControlButton(R.drawable.ic_page_previous, WidgetActionCallback.action(WidgetAction.PREVIOUS_PAGE))
                ControlButton(R.drawable.ic_page_next, WidgetActionCallback.action(WidgetAction.NEXT_PAGE))
                ControlButton(R.drawable.ic_document, pickAction)
            }
        }
        ControlButton(R.drawable.ic_controls, WidgetActionCallback.action(WidgetAction.TOGGLE_CONTROLS))
    }
}

@Composable
private fun PanPad() {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        ControlButton(R.drawable.ic_chevron_up, WidgetActionCallback.action(WidgetAction.PAN_UP))
        Row {
            ControlButton(R.drawable.ic_chevron_left, WidgetActionCallback.action(WidgetAction.PAN_LEFT))
            ControlButton(R.drawable.ic_chevron_right, WidgetActionCallback.action(WidgetAction.PAN_RIGHT))
        }
        ControlButton(R.drawable.ic_chevron_down, WidgetActionCallback.action(WidgetAction.PAN_DOWN))
    }
}
```

- [ ] **Step 9: Compose the overlay into the widget**

Replace `app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidget.kt` in full:

```kotlin
package com.samoed.pdfwidget.widget

import android.content.Context
import android.graphics.Bitmap
import androidx.compose.runtime.Composable
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp
import androidx.glance.GlanceId
import androidx.glance.GlanceModifier
import androidx.glance.Image
import androidx.glance.ImageProvider
import androidx.glance.action.clickable
import androidx.glance.appwidget.GlanceAppWidget
import androidx.glance.appwidget.GlanceAppWidgetManager
import androidx.glance.appwidget.action.actionStartActivity
import androidx.glance.appwidget.provideContent
import androidx.glance.layout.Alignment
import androidx.glance.layout.Box
import androidx.glance.layout.ContentScale
import androidx.glance.layout.fillMaxSize
import androidx.glance.layout.padding
import androidx.glance.text.Text
import androidx.glance.text.TextStyle
import androidx.glance.unit.ColorProvider
import com.samoed.pdfwidget.data.ColorMode
import com.samoed.pdfwidget.data.ColorModeResolver
import com.samoed.pdfwidget.data.WidgetStateStore
import com.samoed.pdfwidget.render.PdfPageRenderer
import com.samoed.pdfwidget.render.RenderRequest

private val OVERLAY_PADDING = 8.dp
private val INDICATOR_COLOR = ColorProvider(Color(0x99000000))

data class PageView(val bitmap: Bitmap, val label: String)

class PdfWidget : GlanceAppWidget() {

    override suspend fun provideGlance(context: Context, id: GlanceId) {
        val widgetId = GlanceAppWidgetManager(context).getAppWidgetId(id)
        val store = WidgetStateStore(context)
        val state = store.widgetState(widgetId)
        val page = state.documentUri?.let {
            loadPage(context, store, widgetId, it, state.colorMode)
        }
        provideContent { Content(context, widgetId, page, state.controlsExpanded) }
    }

    @Composable
    private fun Content(context: Context, widgetId: Int, page: PageView?, expanded: Boolean) {
        val pickAction = actionStartActivity(DocumentPickerActivity.intent(context, widgetId))
        Box(modifier = GlanceModifier.fillMaxSize()) {
            if (page == null) EmptyState(pickAction) else PageSurface(page)
            Box(GlanceModifier.fillMaxSize().padding(OVERLAY_PADDING), Alignment.BottomStart) {
                page?.let { Text(it.label, style = TextStyle(color = INDICATOR_COLOR)) }
            }
            Box(GlanceModifier.fillMaxSize().padding(OVERLAY_PADDING), Alignment.BottomEnd) {
                ControlCluster(expanded = expanded, pickAction = pickAction)
            }
        }
    }

    @Composable
    private fun EmptyState(pickAction: androidx.glance.action.Action) {
        Box(GlanceModifier.fillMaxSize().clickable(pickAction), Alignment.Center) {
            Text("Tap to choose a PDF")
        }
    }

    @Composable
    private fun PageSurface(page: PageView) {
        Image(
            provider = ImageProvider(page.bitmap),
            contentDescription = null,
            contentScale = ContentScale.Fit,
            modifier = GlanceModifier.fillMaxSize(),
        )
    }

    private suspend fun loadPage(
        context: Context,
        store: WidgetStateStore,
        widgetId: Int,
        documentUri: String,
        colorMode: ColorMode,
    ): PageView? = runCatching {
        val renderer = PdfPageRenderer(context)
        val viewport = store.position(documentUri)
        val (width, height) = WidgetSize.pixels(context, widgetId)
        val bitmap = renderer.render(
            RenderRequest(
                documentUri = documentUri,
                viewport = viewport,
                widthPx = width,
                heightPx = height,
                night = ColorModeResolver.isNight(context, colorMode),
            )
        )
        PageView(bitmap, "${viewport.pageIndex + 1} / ${renderer.pageCount(documentUri)}")
    }.getOrNull()
}
```

- [ ] **Step 10: Verify manually**

Run: `./gradlew installDebug`
Expected: the collapsed widget shows the page, a page indicator, and one toggle button. Tapping the toggle reveals the pan pad, zoom, page and document buttons. Every button changes the page as specified, and the night button cycles system → night → light.

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat: add collapsible controls and action dispatch"
```

---

### Task 8: Resize, cleanup, and the unavailable-document state

**Files:**
- Modify: `app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidgetReceiver.kt`
- Modify: `app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidget.kt`

**Interfaces:**
- Consumes: everything from Tasks 3–7.
- Produces: `PdfWidgetReceiver.onDeleted`, `PdfWidgetReceiver.onAppWidgetOptionsChanged`; `PdfWidget` renders an unavailable state.

- [ ] **Step 1: Handle deletion and resize**

`app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidgetReceiver.kt`:

```kotlin
package com.samoed.pdfwidget.widget

import android.appwidget.AppWidgetManager
import android.content.Context
import android.os.Bundle
import androidx.glance.appwidget.GlanceAppWidget
import androidx.glance.appwidget.GlanceAppWidgetManager
import androidx.glance.appwidget.GlanceAppWidgetReceiver
import androidx.glance.appwidget.update
import com.samoed.pdfwidget.data.WidgetStateStore
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch

class PdfWidgetReceiver : GlanceAppWidgetReceiver() {

    override val glanceAppWidget: GlanceAppWidget = PdfWidget()

    override fun onDeleted(context: Context, appWidgetIds: IntArray) {
        super.onDeleted(context, appWidgetIds)
        CoroutineScope(Dispatchers.IO).launch {
            val store = WidgetStateStore(context)
            appWidgetIds.forEach { store.removeWidget(it) }
        }
    }

    override fun onAppWidgetOptionsChanged(
        context: Context,
        appWidgetManager: AppWidgetManager,
        appWidgetId: Int,
        newOptions: Bundle,
    ) {
        super.onAppWidgetOptionsChanged(context, appWidgetManager, appWidgetId, newOptions)
        CoroutineScope(Dispatchers.IO).launch {
            val glanceId = GlanceAppWidgetManager(context).getGlanceIdBy(appWidgetId)
            PdfWidget().update(context, glanceId)
        }
    }
}
```

- [ ] **Step 2: Render an explicit unavailable state**

In `PdfWidget.provideGlance`, distinguish "no document chosen" from "document unavailable". Change `renderPage` to return a sealed result rather than a nullable bitmap:

```kotlin
sealed interface PageResult {
    data object NoDocument : PageResult
    data object Unavailable : PageResult
    data class Rendered(val bitmap: Bitmap, val label: String) : PageResult
}
```

Place this at the top of `PdfWidget.kt`. `renderPage` returns `PageResult.Unavailable` when `runCatching` fails, and `Content` renders `R.string.document_unavailable` with the same picker click action as the empty state.

- [ ] **Step 3: Verify manually**

Run: `./gradlew installDebug`
Place two widgets on different pages, give each a different PDF, page each forwards independently.
Expected: they do not affect each other. Resizing a widget re-renders at the new size and keeps the reading position. Removing the widget and placing a new one with the same PDF resumes at the same page.

Then revoke access: move or delete the source PDF, tap the widget.
Expected: it shows "Document unavailable. Tap to reselect." and tapping opens the picker.

- [ ] **Step 4: Run the full test suite**

Run: `./gradlew :app:testDebugUnitTest`
Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: handle resize, deletion, and unavailable documents"
```

---

### Task 9: Reduce tap latency

**Files:**
- Create: `app/src/main/kotlin/com/samoed/pdfwidget/render/RenderScheduler.kt`
- Modify: `app/src/main/kotlin/com/samoed/pdfwidget/render/PdfPageRenderer.kt`
- Modify: `app/src/main/kotlin/com/samoed/pdfwidget/widget/PdfWidget.kt`
- Test: `app/src/test/kotlin/com/samoed/pdfwidget/render/RenderSchedulerTest.kt`

**Interfaces:**
- Consumes: `PageRenderer`, `RenderRequest` (Task 4).
- Produces: `class RenderScheduler(private val delegate: PageRenderer)` with `suspend fun render(request: RenderRequest): Bitmap` and `fun prefetch(scope: CoroutineScope, request: RenderRequest)`; a process-wide `object Renderers { fun scheduler(context: Context): RenderScheduler }`.

The scheduler holds a single-entry cache keyed by the full `RenderRequest`, so a prefetched next page is returned immediately on the following tap.

This departs from the spec, which described caching the open `PdfRenderer` between actions. Caching the rendered bitmap achieves the same latency goal without holding a file descriptor open indefinitely or serving stale content when the underlying file changes. If tap latency is still visible after this task, add descriptor caching inside `PdfPageRenderer` as a follow-up — no other module is affected.

- [ ] **Step 1: Write the failing test**

`app/src/test/kotlin/com/samoed/pdfwidget/render/RenderSchedulerTest.kt`:

```kotlin
package com.samoed.pdfwidget.render

import android.graphics.Bitmap
import com.samoed.pdfwidget.core.Viewport
import kotlinx.coroutines.test.runTest
import org.junit.Assert.assertEquals
import org.junit.Test
import org.junit.runner.RunWith
import org.robolectric.RobolectricTestRunner

private const val DOCUMENT = "content://docs/book.pdf"
private const val WIDTH = 10
private const val HEIGHT = 10

private class CountingRenderer : PageRenderer {
    var renderCount = 0
    override suspend fun pageCount(documentUri: String): Int = 5
    override suspend fun render(request: RenderRequest): Bitmap {
        renderCount++
        return Bitmap.createBitmap(WIDTH, HEIGHT, Bitmap.Config.ARGB_8888)
    }
}

private fun request(pageIndex: Int) = RenderRequest(
    documentUri = DOCUMENT,
    viewport = Viewport(pageIndex = pageIndex),
    widthPx = WIDTH,
    heightPx = HEIGHT,
    night = false,
)

@RunWith(RobolectricTestRunner::class)
class RenderSchedulerTest {

    @Test
    fun `an identical request is served from cache`() = runTest {
        val delegate = CountingRenderer()
        val scheduler = RenderScheduler(delegate)
        scheduler.render(request(0))
        scheduler.render(request(0))
        assertEquals(1, delegate.renderCount)
    }

    @Test
    fun `a different request re-renders`() = runTest {
        val delegate = CountingRenderer()
        val scheduler = RenderScheduler(delegate)
        scheduler.render(request(0))
        scheduler.render(request(1))
        assertEquals(2, delegate.renderCount)
    }

    @Test
    fun `a prefetched request is served from cache`() = runTest {
        val delegate = CountingRenderer()
        val scheduler = RenderScheduler(delegate)
        scheduler.prefetch(this, request(3))
        scheduler.render(request(3))
        assertEquals(1, delegate.renderCount)
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `./gradlew :app:testDebugUnitTest --tests "*RenderSchedulerTest*"`
Expected: FAIL — unresolved reference `RenderScheduler`.

- [ ] **Step 3: Write the scheduler**

`app/src/main/kotlin/com/samoed/pdfwidget/render/RenderScheduler.kt`:

```kotlin
package com.samoed.pdfwidget.render

import android.content.Context
import android.graphics.Bitmap
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.launch
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

class RenderScheduler(private val delegate: PageRenderer) {

    private val mutex = Mutex()
    private var cachedRequest: RenderRequest? = null
    private var cachedBitmap: Bitmap? = null

    suspend fun pageCount(documentUri: String): Int = delegate.pageCount(documentUri)

    suspend fun render(request: RenderRequest): Bitmap = mutex.withLock {
        cachedBitmap?.takeIf { cachedRequest == request } ?: renderAndCache(request)
    }

    fun prefetch(scope: CoroutineScope, request: RenderRequest) {
        scope.launch { runCatching { render(request) } }
    }

    private suspend fun renderAndCache(request: RenderRequest): Bitmap {
        val bitmap = delegate.render(request)
        cachedRequest = request
        cachedBitmap = bitmap
        return bitmap
    }
}

object Renderers {
    private var instance: RenderScheduler? = null

    fun scheduler(context: Context): RenderScheduler = instance ?: RenderScheduler(
        PdfPageRenderer(context.applicationContext)
    ).also { instance = it }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `./gradlew :app:testDebugUnitTest --tests "*RenderSchedulerTest*"`
Expected: PASS.

- [ ] **Step 5: Route the widget through the scheduler and prefetch the next page**

In `PdfWidget.provideGlance`, replace `PdfPageRenderer(context)` with `Renderers.scheduler(context)`. After rendering the current page, prefetch the next one:

```kotlin
        Renderers.scheduler(context).prefetch(
            CoroutineScope(Dispatchers.IO),
            currentRequest.copy(viewport = viewport.atNextPage(pageCount)),
        )
```

Guard the prefetch so it is skipped when `viewport.atNextPage(pageCount) == viewport`.

- [ ] **Step 6: Verify manually**

Run: `./gradlew installDebug`
Tap next-page repeatedly.
Expected: page turns feel immediate rather than showing a visible delay.

- [ ] **Step 7: Run the full suite and commit**

Run: `./gradlew :app:testDebugUnitTest :app:assembleDebug`
Expected: all tests PASS, BUILD SUCCESSFUL.

```bash
git add -A
git commit -m "perf: cache and prefetch rendered pages"
```

---

## Fallback: if Glance cannot carry the bitmaps

If the launcher rejects the widget update with a bitmap memory or `TransactionTooLargeException` error, or Glance fails to render `ImageProvider(bitmap)` at these sizes, switch only the widget layer:

1. Write the rendered bitmap to `context.cacheDir/widget-<widgetId>.png`.
2. Expose it through a `FileProvider` declared in the manifest with `android:grantUriPermissions="true"`.
3. Call `grantUriPermission` for the launcher package, then use `RemoteViews.setImageViewUri`.

Tasks 2, 3, 4 and 9 are unaffected — they contain no widget-layer code.
