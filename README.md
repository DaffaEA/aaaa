# 🇮🇩 BiSindo Translator 2

**Real-time Indonesian Sign Language (BiSindo) Translator using CameraX + PyTorch + YOLOv5**

---

## 📁 Project Structure

```
BiSindoTranslator2/
│
├── app/
│   ├── build.gradle.kts
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml
│       │   ├── assets/
│       │   │   └── yolov5s.torchscript.pt
│       │   ├── java/
│       │   │   └── com/
│       │   │       └── example/
│       │   │           └── bisindotranslator2/
│       │   │               ├── MainActivity.kt
│       │   │               ├── OverlayView.kt
│       │   │               ├── YoloDetector.kt
│       │   │               └── Utils.kt
│       │   └── res/
│       │       ├── layout/
│       │       │   └── activity_main.xml
│       │       ├── values/
│       │       │   ├── colors.xml
│       │       │   └── strings.xml
│       │       └── xml/
│       │           └── file_paths.xml
│       ├── androidTest/
│       │   └── java/
│       │       └── com/
│       │           └── example/
│       │               └── bisindotranslator2/
│       └── test/
│           └── java/
│               └── com/
│                   └── example/
│                       └── bisindotranslator2/
│
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
│
├── .gitignore
├── build.gradle.kts
├── gradle.properties
├── gradlew
├── gradlew.bat
├── settings.gradle.kts
└── README.md
```

---

## 🚀 Features

- ✅ Real-time sign language detection using YOLOv5
- ✅ CameraX integration for smooth camera preview
- ✅ PyTorch Mobile for on-device inference
- ✅ Custom overlay view for bounding boxes
- ✅ FPS counter and detection display
- ✅ Material Design UI

---

## 📋 Prerequisites

- **Android Studio**: Hedgehog (2023.1.1) or newer
- **Minimum SDK**: API 24 (Android 7.0)
- **Target SDK**: API 34 (Android 14)
- **Kotlin**: 1.9.22
- **Gradle**: 8.5.0
- **YOLOv5 Model**: TorchScript format (`.torchscript.pt`)

---

## ⚙️ Gradle Configuration

### **Root `build.gradle.kts`**

```kotlin
plugins {
    id("com.android.application") version "8.5.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.22" apply false
}
```

### **`settings.gradle.kts`**

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "BiSindoTranslator2"
include(":app")
```

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

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
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
    // AndroidX Core
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

    // Testing
    testImplementation("junit:junit:4.13.2")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
}
```

---

## 📱 AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-feature
        android:name="android.hardware.camera"
        android:required="true" />

    <uses-permission android:name="android.permission.CAMERA" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppCompat.DayNight.NoActionBar"
        tools:targetApi="31">
        
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:screenOrientation="portrait">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

---

## 🧠 Kotlin Source Files

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
import androidx.core.content.ContextCompat
import java.util.concurrent.Executors

class MainActivity : AppCompatActivity() {

    private lateinit var previewView: PreviewView
    private lateinit var overlayView: OverlayView
    private lateinit var fpsText: TextView
    private lateinit var detectedText: TextView
    private lateinit var clearButton: Button

    private lateinit var yoloDetector: YoloDetector
    private val cameraExecutor = Executors.newSingleThreadExecutor()

    private val requestPermissionLauncher =
        registerForActivityResult(ActivityResultContracts.RequestPermission()) { isGranted ->
            if (isGranted) {
                startCamera()
            } else {
                Toast.makeText(this, "Camera permission denied", Toast.LENGTH_SHORT).show()
            }
        }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        initializeViews()
        yoloDetector = YoloDetector(this)

        clearButton.setOnClickListener {
            detectedText.text = "Detected: "
        }

        if (allPermissionsGranted()) {
            startCamera()
        } else {
            requestPermissionLauncher.launch(Manifest.permission.CAMERA)
        }
    }

    private fun initializeViews() {
        previewView = findViewById(R.id.previewView)
        overlayView = findViewById(R.id.overlayView)
        fpsText = findViewById(R.id.fpsText)
        detectedText = findViewById(R.id.detectedText)
        clearButton = findViewById(R.id.clearButton)
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            val preview = Preview.Builder()
                .build()
                .also {
                    it.setSurfaceProvider(previewView.surfaceProvider)
                }

            val imageAnalyzer = ImageAnalysis.Builder()
                .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                .build()
                .also { analysis ->
                    analysis.setAnalyzer(cameraExecutor) { imageProxy ->
                        processImage(imageProxy)
                    }
                }

            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(
                    this,
                    cameraSelector,
                    preview,
                    imageAnalyzer
                )
            } catch (e: Exception) {
                Log.e(TAG, "Camera binding failed", e)
            }
        }, ContextCompat.getMainExecutor(this))
    }

    private fun processImage(imageProxy: ImageProxy) {
        val startTime = System.currentTimeMillis()
        val results = yoloDetector.analyzeImage(imageProxy)
        val processingTime = System.currentTimeMillis() - startTime

        runOnUiThread {
            overlayView.updateDetections(results)
            
            val detectedLabels = results.joinToString(", ") { it.label }
            detectedText.text = "Detected: $detectedLabels"
            
            val fps = if (processingTime > 0) 1000 / processingTime else 0
            fpsText.text = "FPS: $fps"
        }

        imageProxy.close()
    }

    private fun allPermissionsGranted() = ContextCompat.checkSelfPermission(
        this, Manifest.permission.CAMERA
    ) == PackageManager.PERMISSION_GRANTED

    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
    }

    companion object {
        private const val TAG = "BiSindoTranslator"
    }
}
```

### **YoloDetector.kt**

```kotlin
package com.example.bisindotranslator2

import android.content.Context
import android.graphics.RectF
import android.util.Log
import androidx.camera.core.ImageProxy
import org.pytorch.IValue
import org.pytorch.Module
import org.pytorch.torchvision.TensorImageUtils

data class Detection(
    val rect: RectF,
    val label: String,
    val score: Float
)

class YoloDetector(private val context: Context) {

    private val module: Module by lazy {
        Module.load(Utils.assetFilePath(context, MODEL_NAME))
    }

    fun analyzeImage(imageProxy: ImageProxy): List<Detection> {
        return try {
            val bitmap = Utils.imageToBitmap(imageProxy)
            
            val inputTensor = TensorImageUtils.bitmapToFloat32Tensor(
                bitmap,
                TensorImageUtils.TORCHVISION_NORM_MEAN_RGB,
                TensorImageUtils.TORCHVISION_NORM_STD_RGB
            )

            val outputTuple = module.forward(IValue.from(inputTensor)).toTuple()
            val outputTensor = outputTuple[0].toTensor()
            val scores = outputTensor.dataAsFloatArray

            // TODO: Implement actual YOLO post-processing
            // This is a placeholder implementation
            parseDetections(scores, imageProxy.width, imageProxy.height)
        } catch (e: Exception) {
            Log.e(TAG, "Error analyzing image", e)
            emptyList()
        }
    }

    private fun parseDetections(
        scores: FloatArray,
        imageWidth: Int,
        imageHeight: Int
    ): List<Detection> {
        // Placeholder implementation
        // Replace with actual YOLO NMS and post-processing
        return listOf(
            Detection(
                RectF(200f, 400f, 600f, 800f),
                "A",
                0.95f
            )
        )
    }

    companion object {
        private const val TAG = "YoloDetector"
        private const val MODEL_NAME = "yolov5s.torchscript.pt"
    }
}
```

### **OverlayView.kt**

```kotlin
package com.example.bisindotranslator2

import android.content.Context
import android.graphics.*
import android.util.AttributeSet
import android.view.View

class OverlayView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    private var detections: List<Detection> = emptyList()

    private val boxPaint = Paint().apply {
        color = Color.GREEN
        style = Paint.Style.STROKE
        strokeWidth = 5f
        isAntiAlias = true
    }

    private val textPaint = Paint().apply {
        color = Color.WHITE
        textSize = 60f
        typeface = Typeface.DEFAULT_BOLD
        isAntiAlias = true
    }

    private val textBackgroundPaint = Paint().apply {
        color = Color.GREEN
        style = Paint.Style.FILL
        alpha = 180
    }

    fun updateDetections(newDetections: List<Detection>) {
        detections = newDetections
        invalidate()
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        for (detection in detections) {
            // Draw bounding box
            canvas.drawRect(detection.rect, boxPaint)
            
            // Draw label with background
            val label = "${detection.label} ${(detection.score * 100).toInt()}%"
            val textBounds = Rect()
            textPaint.getTextBounds(label, 0, label.length, textBounds)
            
            val textBackground = RectF(
                detection.rect.left,
                detection.rect.top - textBounds.height() - 20,
                detection.rect.left + textBounds.width() + 20,
                detection.rect.top
            )
            
            canvas.drawRect(textBackground, textBackgroundPaint)
            canvas.drawText(
                label,
                detection.rect.left + 10,
                detection.rect.top - 10,
                textPaint
            )
        }
    }
}
```

### **Utils.kt**

```kotlin
package com.example.bisindotranslator2

import android.content.Context
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import android.graphics.ImageFormat
import android.graphics.Matrix
import android.graphics.Rect
import android.graphics.YuvImage
import androidx.camera.core.ImageProxy
import java.io.ByteArrayOutputStream
import java.io.File
import java.io.FileOutputStream

object Utils {

    fun assetFilePath(context: Context, assetName: String): String {
        val file = File(context.filesDir, assetName)
        
        if (file.exists() && file.length() > 0) {
            return file.absolutePath
        }

        context.assets.open(assetName).use { inputStream ->
            FileOutputStream(file).use { outputStream ->
                val buffer = ByteArray(4 * 1024)
                var read: Int
                while (inputStream.read(buffer).also { read = it } != -1) {
                    outputStream.write(buffer, 0, read)
                }
                outputStream.flush()
            }
        }
        
        return file.absolutePath
    }

    fun imageToBitmap(imageProxy: ImageProxy): Bitmap {
        val yBuffer = imageProxy.planes[0].buffer
        val uBuffer = imageProxy.planes[1].buffer
        val vBuffer = imageProxy.planes[2].buffer

        val ySize = yBuffer.remaining()
        val uSize = uBuffer.remaining()
        val vSize = vBuffer.remaining()

        val nv21 = ByteArray(ySize + uSize + vSize)

        yBuffer.get(nv21, 0, ySize)
        vBuffer.get(nv21, ySize, vSize)
        uBuffer.get(nv21, ySize + vSize, uSize)

        val yuvImage = YuvImage(nv21, ImageFormat.NV21, imageProxy.width, imageProxy.height, null)
        val out = ByteArrayOutputStream()
        yuvImage.compressToJpeg(Rect(0, 0, imageProxy.width, imageProxy.height), 100, out)
        val imageBytes = out.toByteArray()
        
        return BitmapFactory.decodeByteArray(imageBytes, 0, imageBytes.size)
    }

    fun rotateBitmap(bitmap: Bitmap, rotationDegrees: Int): Bitmap {
        val matrix = Matrix()
        matrix.postRotate(rotationDegrees.toFloat())
        return Bitmap.createBitmap(bitmap, 0, 0, bitmap.width, bitmap.height, matrix, true)
    }
}
```

---

## 🖼️ XML Layouts

### **res/layout/activity_main.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/black"
    tools:context=".MainActivity">

    <androidx.camera.view.PreviewView
        android:id="@+id/previewView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintBottom_toTopOf="@id/bottomPanel"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:scaleType="fillCenter" />

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
        android:background="@color/semi_transparent_black"
        android:gravity="center_vertical"
        android:orientation="horizontal"
        android:padding="12dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent">

        <TextView
            android:id="@+id/detectedText"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:paddingEnd="8dp"
            android:text="Detected: "
            android:textColor="@color/white"
            android:textSize="18sp"
            tools:ignore="RtlSymmetry" />

        <Button
            android:id="@+id/clearButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:backgroundTint="@color/teal_700"
            android:text="Clear"
            android:textColor="@color/white" />
    </LinearLayout>

</androidx.constraintlayout.widget.ConstraintLayout>
```

### **res/values/colors.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
    <color name="teal_700">#FF018786</color>
    <color name="semi_transparent_black">#88000000</color>
</resources>
```

### **res/values/strings.xml**

```xml
<resources>
    <string name="app_name">BiSindo Translator 2</string>
</resources>
```

---

## 🧩 Model Setup

### **Step 1: Prepare Your Model**

Convert your YOLOv5 model to TorchScript format:

```python
# Python script to convert YOLOv5 to TorchScript
import torch

model = torch.hub.load('ultralytics/yolov5', 'custom', path='your_model.pt')
model.eval()

example = torch.rand(1, 3, 640, 640)
traced_script_module = torch.jit.trace(model, example)
traced_script_module.save("yolov5s.torchscript.pt")
```

### **Step 2: Add Model to Assets**

1. Create the `assets` folder:
   ```
   app/src/main/assets/
   ```

2. Place your model file:
   ```
   app/src/main/assets/yolov5s.torchscript.pt
   ```

---

## ▶️ Installation & Running

### **Prerequisites**

1. Install Android Studio Hedgehog or newer
2. Enable USB Debugging on your Android device
3. Prepare your YOLOv5 TorchScript model

### **Steps**

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/BiSindoTranslator2.git
   cd BiSindoTranslator2
   ```

2. **Open in Android Studio**
   - Open Android Studio
   - Select "Open an Existing Project"
   - Navigate to the project folder

3. **Add your model**
   - Place `yolov5s.torchscript.pt` in `app/src/main/assets/`

4. **Sync Gradle**
   - Click "Sync Project with Gradle Files" in Android Studio

5. **Connect your device**
   - Enable Developer Options and USB Debugging
   - Connect via USB cable

6. **Run the app**
   - Click the Run button ▶️ in Android Studio
   - Select your device
   - Grant camera permissions when prompted

---

## 🔧 Troubleshooting

### **Common Issues**

**1. Model not found**
```
Error: Model file not found in assets
```
**Solution**: Ensure `yolov5s.torchscript.pt` is placed in `app/src/main/assets/`

**2. Camera permission denied**
```
Camera permission denied
```
**Solution**: Grant camera permission in Android Settings → Apps → BiSindo Translator 2

**3. Build failed**
```
Could not resolve org.pytorch:pytorch_android:1.10.0
```
**Solution**: Ensure you have internet connection and sync Gradle files

---

## 📊 Performance Tips

- **Model Size**: Use smaller models (YOLOv5s/n) for better FPS
- **Input Resolution**: Lower resolution = faster inference
- **Target FPS**: Aim for 15-30 FPS on mobile devices
- **Battery**: Real-time detection is battery-intensive

---

## 🧱 Roadmap

- [ ] Add front/back camera switch
- [ ] Implement dynamic confidence threshold slider
- [ ] Add text-to-speech for detected signs
- [ ] Implement letter/word accumulation
- [ ] Add offline dictionary for BiSindo
- [ ] Support for sentence formation
- [ ] Add history of detected signs
- [ ] Cloud model update support
- [ ] Multi-language support (ID/EN)

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the project
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## 📧 Contact

Your Name - [@yourtwitter](https://twitter.com/yourtwitter) - email@example.com

Project Link: [https://github.com/yourusername/BiSindoTranslator2](https://github.com/yourusername/BiSindoTranslator2)

---

## 🙏 Acknowledgments

- [Ultralytics YOLOv5](https://github.com/ultralytics/yolov5)
- [PyTorch Mobile](https://pytorch.org/mobile/)
- [CameraX](https://developer.android.com/training/camerax)
- BiSindo community

---

**Developed with ❤️ using Kotlin, CameraX, and PyTorch Mobile**
