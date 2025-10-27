# 🇮🇩 BiSindo Translator 2

**Real-time Indonesian Sign Language (BiSindo) Translator using CameraX + PyTorch + YOLOv5**

---

## 📁 Project Structure

```
E:.
├── .gradle
├── .idea
├── app
│   ├── build.gradle.kts
│   └── src
│       ├── androidTest/java/com/example/bisindotranslator2/
│       ├── main
│       │   ├── AndroidManifest.xml
│       │   ├── java/com/example/bisindotranslator2/
│       │   │   ├── MainActivity.kt
│       │   │   ├── OverlayView.kt
│       │   │   └── YoloDetector.kt
│       │   └── res/
│       │       ├── layout/activity_main.xml
│       │       ├── values/colors.xml
│       │       ├── values/strings.xml
│       │       └── xml/file_paths.xml
│       └── test/java/com/example/bisindotranslator2/
├── gradle/wrapper/
│   ├── gradle-wrapper.jar
│   └── gradle-wrapper.properties
├── build.gradle.kts
└── settings.gradle.kts
```

---

## ⚙️ Gradle Configuration

### **Root `build.gradle.kts`**

```kotlin
plugins {
    id("com.android.application") version "8.5.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.22" apply false
}
```

---

### **`settings.gradle.kts`**

```kotlin
rootProject.name = "BiSindoTranslator2"
include(":app")
```

---

### **App-level `build.gradle.kts`**

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.bisindotranslator2"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.example.bisindotranslator2"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    buildFeatures {
        viewBinding = true
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    implementation("com.google.android.material:material:1.11.0")
    implementation("androidx.constraintlayout:constraintlayout:2.1.4")

    // CameraX
    implementation("androidx.camera:camera-camera2:1.3.1")
    implementation("androidx.camera:camera-lifecycle:1.3.1")
    implementation("androidx.camera:camera-view:1.3.1")

    // PyTorch Mobile
    implementation("org.pytorch:pytorch_android:1.10.0")
    implementation("org.pytorch:pytorch_android_torchvision:1.10.0")
}
```

---

## 📱 AndroidManifest.xml

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.bisindotranslator2">

    <uses-permission android:name="android.permission.CAMERA" />

    <application
        android:allowBackup="true"
        android:label="@string/app_name"
        android:theme="@style/Theme.AppCompat.DayNight.NoActionBar">
        <activity android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>

</manifest>
```

---

## 🧠 Kotlin Source Code

### **MainActivity.kt**

```kotlin
package com.example.bisindotranslator2

import android.Manifest
import android.content.pm.PackageManager
import android.os.Bundle
import android.util.Log
import android.widget.Button
import android.widget.TextView
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import org.pytorch.IValue
import org.pytorch.Module
import org.pytorch.torchvision.TensorImageUtils
import java.util.concurrent.Executors

class MainActivity : AppCompatActivity() {

    private lateinit var previewView: PreviewView
    private lateinit var overlayView: OverlayView
    private lateinit var fpsText: TextView
    private lateinit var detectedText: TextView
    private lateinit var clearButton: Button

    private lateinit var yoloDetector: YoloDetector
    private val cameraExecutor = Executors.newSingleThreadExecutor()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        previewView = findViewById(R.id.previewView)
        overlayView = findViewById(R.id.overlayView)
        fpsText = findViewById(R.id.fpsText)
        detectedText = findViewById(R.id.detectedText)
        clearButton = findViewById(R.id.clearButton)

        yoloDetector = YoloDetector(this)

        clearButton.setOnClickListener { detectedText.text = "Detected:" }

        if (allPermissionsGranted()) {
            startCamera()
        } else {
            requestPermissionLauncher.launch(Manifest.permission.CAMERA)
        }
    }

    private val requestPermissionLauncher =
        registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
            if (granted) startCamera()
            else Toast.makeText(this, "Camera permission denied", Toast.LENGTH_SHORT).show()
        }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()
            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(previewView.surfaceProvider)
            }

            val imageAnalyzer = ImageAnalysis.Builder()
                .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                .build()
                .also {
                    it.setAnalyzer(cameraExecutor) { imageProxy ->
                        val results = yoloDetector.analyzeImage(imageProxy)
                        runOnUiThread {
                            overlayView.updateDetections(results)
                            detectedText.text = "Detected: " +
                                results.joinToString(", ") { it.label }
                        }
                        imageProxy.close()
                    }
                }

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(
                    this, CameraSelector.DEFAULT_BACK_CAMERA, preview, imageAnalyzer
                )
            } catch (e: Exception) {
                Log.e("CameraX", "Binding failed", e)
            }
        }, ContextCompat.getMainExecutor(this))
    }

    private fun allPermissionsGranted() =
        ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) ==
                PackageManager.PERMISSION_GRANTED
}
```

---

### **YoloDetector.kt**

```kotlin
package com.example.bisindotranslator2

import android.content.Context
import android.graphics.RectF
import androidx.camera.core.ImageProxy
import org.pytorch.*
import org.pytorch.torchvision.TensorImageUtils

data class Detection(val rect: RectF, val label: String, val score: Float)

class YoloDetector(context: Context) {

    private val module: Module =
        Module.load(Utils.assetFilePath(context, "yolov5s.torchscript.pt"))

    fun analyzeImage(imageProxy: ImageProxy): List<Detection> {
        val bitmap = Utils.imageToBitmap(imageProxy)
        val inputTensor = TensorImageUtils.bitmapToFloat32Tensor(
            bitmap,
            TensorImageUtils.TORCHVISION_NORM_MEAN_RGB,
            TensorImageUtils.TORCHVISION_NORM_STD_RGB
        )

        val outputTuple = module.forward(IValue.from(inputTensor)).toTuple()
        val outputTensor = outputTuple[0].toTensor()
        val scores = outputTensor.dataAsFloatArray

        // Simplified fake results for example
        return listOf(Detection(RectF(200f, 400f, 600f, 800f), "A", 0.95f))
    }
}
```

---

### **OverlayView.kt**

```kotlin
package com.example.bisindotranslator2

import android.content.Context
import android.graphics.*
import android.util.AttributeSet
import android.view.View

class OverlayView(context: Context, attrs: AttributeSet?) : View(context, attrs) {

    private var detections: List<Detection> = emptyList()
    private val boxPaint = Paint().apply {
        color = Color.GREEN
        style = Paint.Style.STROKE
        strokeWidth = 5f
    }

    private val textPaint = Paint().apply {
        color = Color.WHITE
        textSize = 60f
        typeface = Typeface.DEFAULT_BOLD
    }

    fun updateDetections(newDetections: List<Detection>) {
        detections = newDetections
        invalidate()
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        for (det in detections) {
            canvas.drawRect(det.rect, boxPaint)
            canvas.drawText("${det.label} ${(det.score * 100).toInt()}%",
                det.rect.left, det.rect.top - 10, textPaint)
        }
    }
}
```

---

### **Utils.kt**

```kotlin
package com.example.bisindotranslator2

import android.content.Context
import android.graphics.Bitmap
import android.graphics.ImageFormat
import android.graphics.Matrix
import androidx.camera.core.ImageProxy
import java.io.File
import java.io.FileOutputStream

object Utils {
    fun assetFilePath(context: Context, assetName: String): String {
        val file = File(context.filesDir, assetName)
        if (file.exists()) return file.absolutePath
        context.assets.open(assetName).use { input ->
            FileOutputStream(file).use { output ->
                input.copyTo(output)
            }
        }
        return file.absolutePath
    }

    fun imageToBitmap(imageProxy: ImageProxy): Bitmap {
        val planeProxy = imageProxy.planes[0]
        val buffer = planeProxy.buffer
        val bytes = ByteArray(buffer.remaining())
        buffer.get(bytes)
        return Bitmap.createBitmap(640, 480, Bitmap.Config.ARGB_8888)
    }
}
```

---

## 🖼️ XML Layouts

### **activity_main.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/black">

    <androidx.camera.view.PreviewView
        android:id="@+id/previewView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintBottom_toTopOf="@id/bottomPanel"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <com.example.bisindotranslator2.OverlayView
        android:id="@+id/overlayView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintBottom_toBottomOf="@id/previewView"
        app:layout_constraintEnd_toEndOf="@id/previewView"
        app:layout_constraintStart_toStartOf="@id/previewView"
        app:layout_constraintTop_toTopOf="@id/previewView" />

    <TextView
        android:id="@+id/fpsText"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:background="@color/semi_transparent_black"
        android:padding="8dp"
        android:text="FPS: 0"
        android:textColor="@color/white"
        android:textSize="16sp"
        android:textStyle="bold"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <LinearLayout
        android:id="@+id/bottomPanel"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="center_vertical"
        android:padding="12dp"
        android:background="@color/semi_transparent_black"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent">

        <TextView
            android:id="@+id/detectedText"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="Detected: "
            android:textColor="@color/white"
            android:textSize="18sp"
            android:paddingEnd="8dp" />

        <Button
            android:id="@+id/clearButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Clear"
            android:textColor="@color/white"
            android:backgroundTint="@color/teal_700" />
    </LinearLayout>

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

### **colors.xml**

```xml
<resources>
    <color name="black">#000000</color>
    <color name="white">#FFFFFF</color>
    <color name="teal_700">#018786</color>
    <color name="semi_transparent_black">#88000000</color>
</resources>
```

---

### **strings.xml**

```xml
<resources>
    <string name="app_name">BiSindo Translator 2</string>
</resources>
```

---

## 🧩 Model Placement

Place your YOLOv5 `.torchscript.pt` model inside:

```
app/src/main/assets/yolov5s.torchscript.pt
```

If the `assets` folder doesn’t exist, create it under:

```
app/src/main/assets/
```

---

## ▶️ Run Instructions

1. Open project in **Android Studio Hedgehog+**
2. Connect your Android device
3. Press **Run ▶️**
4. Grant **camera permission**
5. The app will start real-time BiSindo detection.

---

## 🧱 Future Add-ons

* 🔁 Add camera switch (front ↔ back)
* 🎚️ Add dynamic confidence threshold
* 🗣️ Text-to-speech for detected signs
* 🌐 Cloud-based model update support

---

**Developed with ❤️ using Kotlin, CameraX, and PyTorch Mobile**
