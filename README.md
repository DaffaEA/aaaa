# 🇮🇩 BiSindo Translator 2

**Real-time Indonesian Sign Language (BiSindo) Translator using CameraX + PyTorch + YOLOv8**

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
│       │   │   └── best.torchscript
│       │   ├── java/
│       │   │   └── com/
│       │   │       └── example/
│       │   │           └── bisindotranslator2/
│       │   │               ├── MainActivity.kt
│       │   │               ├── OverlayView.kt
│       │   │               ├── YoloV8Detector.kt
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

- ✅ Real-time BiSindo sign language detection using YOLOv8
- ✅ CameraX integration for smooth camera preview
- ✅ PyTorch Mobile for on-device inference
- ✅ Custom overlay view for bounding boxes
- ✅ FPS counter and detection display
- ✅ NMS (Non-Maximum Suppression) for accurate detections
- ✅ Support for all 26 BiSindo alphabet letters (A-Z)
- ✅ Material Design UI

---

## 📋 Prerequisites

- **Android Studio**: Hedgehog (2023.1.1) or newer
- **Minimum SDK**: API 24 (Android 7.0)
- **Target SDK**: API 34 (Android 14)
- **Kotlin**: 1.9.22
- **Gradle**: 8.5.0
- **YOLOv8 Model**: TorchScript format (`best.torchscript`)

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

    private lateinit var yoloDetector: YoloV8Detector
    private val cameraExecutor = Executors.newSingleThreadExecutor()

    private var frameCount = 0
    private var lastFpsTime = System.currentTimeMillis()

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
        
        try {
            yoloDetector = YoloV8Detector(this)
        } catch (e: Exception) {
            Log.e(TAG, "Failed to load model", e)
            Toast.makeText(this, "Failed to load AI model", Toast.LENGTH_LONG).show()
            return
        }

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
                .setTargetResolution(android.util.Size(416, 416))
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
        try {
            val results = yoloDetector.detect(imageProxy)
            
            // Calculate FPS
            frameCount++
            val currentTime = System.currentTimeMillis()
            if (currentTime - lastFpsTime >= 1000) {
                val fps = frameCount
                frameCount = 0
                lastFpsTime = currentTime
                
                runOnUiThread {
                    fpsText.text = "FPS: $fps"
                }
            }

            runOnUiThread {
                overlayView.updateDetections(results, imageProxy.width, imageProxy.height)
                
                val detectedLabels = results
                    .sortedByDescending { it.score }
                    .take(3)
                    .joinToString(", ") { "${it.label} (${(it.score * 100).toInt()}%)" }
                
                detectedText.text = if (detectedLabels.isEmpty()) {
                    "Detected: -"
                } else {
                    "Detected: $detectedLabels"
                }
            }
        } catch (e: Exception) {
            Log.e(TAG, "Error processing image", e)
        } finally {
            imageProxy.close()
        }
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

### **YoloV8Detector.kt**

```kotlin
package com.example.bisindotranslator2

import android.content.Context
import android.graphics.Bitmap
import android.graphics.RectF
import android.util.Log
import androidx.camera.core.ImageProxy
import org.pytorch.IValue
import org.pytorch.Module
import org.pytorch.Tensor
import org.pytorch.torchvision.TensorImageUtils
import kotlin.math.max
import kotlin.math.min

data class Detection(
    val rect: RectF,
    val label: String,
    val score: Float,
    val classId: Int
)

class YoloV8Detector(context: Context) {

    private val module: Module
    private val inputSize = 416
    
    // BiSindo alphabet labels (A-Z)
    private val labels = arrayOf(
        "A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M",
        "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z"
    )

    init {
        try {
            val modelPath = Utils.assetFilePath(context, MODEL_NAME)
            module = Module.load(modelPath)
            Log.d(TAG, "✅ Model loaded successfully from: $modelPath")
        } catch (e: Exception) {
            Log.e(TAG, "❌ Failed to load model", e)
            throw RuntimeException("Failed to load YOLOv8 model", e)
        }
    }

    fun detect(imageProxy: ImageProxy): List<Detection> {
        return try {
            // Convert ImageProxy to Bitmap
            val bitmap = Utils.imageToBitmap(imageProxy)
            
            // Resize to model input size
            val resizedBitmap = Bitmap.createScaledBitmap(bitmap, inputSize, inputSize, true)
            
            // Convert to tensor with normalization
            val inputTensor = TensorImageUtils.bitmapToFloat32Tensor(
                resizedBitmap,
                floatArrayOf(0f, 0f, 0f),  // No mean subtraction
                floatArrayOf(255f, 255f, 255f)  // Normalize to [0, 1]
            )

            // Run inference
            val outputTensor = module.forward(IValue.from(inputTensor)).toTensor()
            
            // Parse YOLOv8 output
            val detections = parseYoloV8Output(
                outputTensor,
                imageProxy.width,
                imageProxy.height,
                CONF_THRESHOLD,
                IOU_THRESHOLD
            )
            
            Log.d(TAG, "Detected ${detections.size} objects")
            detections
            
        } catch (e: Exception) {
            Log.e(TAG, "Error during detection", e)
            emptyList()
        }
    }

    private fun parseYoloV8Output(
        output: Tensor,
        originalWidth: Int,
        originalHeight: Int,
        confThreshold: Float,
        iouThreshold: Float
    ): List<Detection> {
        
        val outputArray = output.dataAsFloatArray
        val shape = output.shape()
        
        // YOLOv8 output shape: [1, 84, 8400] or [1, num_classes+4, num_predictions]
        // where 84 = 4 (bbox) + 80 (classes) for COCO, or 4 + 26 for BiSindo
        
        val numPredictions = shape[2].toInt()
        val numClasses = shape[1].toInt() - 4
        
        Log.d(TAG, "Output shape: ${shape.contentToString()}, predictions: $numPredictions, classes: $numClasses")
        
        val detectionList = mutableListOf<Detection>()
        
        // Parse predictions
        for (i in 0 until numPredictions) {
            // Get bbox coordinates (center_x, center_y, width, height)
            val cx = outputArray[i]
            val cy = outputArray[numPredictions + i]
            val w = outputArray[2 * numPredictions + i]
            val h = outputArray[3 * numPredictions + i]
            
            // Get class scores
            var maxScore = 0f
            var maxClassId = 0
            
            for (c in 0 until numClasses) {
                val score = outputArray[(4 + c) * numPredictions + i]
                if (score > maxScore) {
                    maxScore = score
                    maxClassId = c
                }
            }
            
            // Filter by confidence threshold
            if (maxScore > confThreshold) {
                // Convert from center format to corner format
                val x1 = (cx - w / 2) * originalWidth / inputSize
                val y1 = (cy - h / 2) * originalHeight / inputSize
                val x2 = (cx + w / 2) * originalWidth / inputSize
                val y2 = (cy + h / 2) * originalHeight / inputSize
                
                // Clamp coordinates
                val rect = RectF(
                    max(0f, x1),
                    max(0f, y1),
                    min(originalWidth.toFloat(), x2),
                    min(originalHeight.toFloat(), y2)
                )
                
                val label = if (maxClassId < labels.size) labels[maxClassId] else "Unknown"
                
                detectionList.add(Detection(rect, label, maxScore, maxClassId))
            }
        }
        
        // Apply Non-Maximum Suppression
        return applyNMS(detectionList, iouThreshold)
    }

    private fun applyNMS(detections: List<Detection>, iouThreshold: Float): List<Detection> {
        if (detections.isEmpty()) return emptyList()
        
        // Sort by confidence score (descending)
        val sortedDetections = detections.sortedByDescending { it.score }.toMutableList()
        val keepDetections = mutableListOf<Detection>()
        
        while (sortedDetections.isNotEmpty()) {
            val best = sortedDetections.removeAt(0)
            keepDetections.add(best)
            
            sortedDetections.removeAll { detection ->
                calculateIoU(best.rect, detection.rect) > iouThreshold
            }
        }
        
        return keepDetections
    }

    private fun calculateIoU(box1: RectF, box2: RectF): Float {
        val intersectionLeft = max(box1.left, box2.left)
        val intersectionTop = max(box1.top, box2.top)
        val intersectionRight = min(box1.right, box2.right)
        val intersectionBottom = min(box1.bottom, box2.bottom)
        
        if (intersectionRight < intersectionLeft || intersectionBottom < intersectionTop) {
            return 0f
        }
        
        val intersectionArea = (intersectionRight - intersectionLeft) * (intersectionBottom - intersectionTop)
        val box1Area = (box1.right - box1.left) * (box1.bottom - box1.top)
        val box2Area = (box2.right - box2.left) * (box2.bottom - box2.top)
        val unionArea = box1Area + box2Area - intersectionArea
        
        return intersectionArea / unionArea
    }

    companion object {
        private const val TAG = "YoloV8Detector"
        private const val MODEL_NAME = "best.torchscript"
        private const val CONF_THRESHOLD = 0.5f  // Confidence threshold
        private const val IOU_THRESHOLD = 0.45f  // NMS IoU threshold
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
    private var imageWidth = 1
    private var imageHeight = 1
    private var viewWidth = 1
    private var viewHeight = 1

    private val boxPaint = Paint().apply {
        color = Color.GREEN
        style = Paint.Style.STROKE
        strokeWidth = 5f
        isAntiAlias = true
    }

    private val textPaint = Paint().apply {
        color = Color.WHITE
        textSize = 48f
        typeface = Typeface.DEFAULT_BOLD
        isAntiAlias = true
    }

    private val textBackgroundPaint = Paint().apply {
        color = Color.GREEN
        style = Paint.Style.FILL
        alpha = 200
    }

    fun updateDetections(newDetections: List<Detection>, imgWidth: Int, imgHeight: Int) {
        detections = newDetections
        imageWidth = imgWidth
        imageHeight = imgHeight
        invalidate()
    }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        viewWidth = w
        viewHeight = h
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        if (viewWidth == 0 || viewHeight == 0 || imageWidth == 0 || imageHeight == 0) return
        
        // Calculate scale factors to map image coordinates to view coordinates
        val scaleX = viewWidth.toFloat() / imageWidth
        val scaleY = viewHeight.toFloat() / imageHeight
        
        for (detection in detections) {
            // Scale bounding box to view coordinates
            val scaledRect = RectF(
                detection.rect.left * scaleX,
                detection.rect.top * scaleY,
                detection.rect.right * scaleX,
                detection.rect.bottom * scaleY
            )
            
            // Draw bounding box
            boxPaint.color = getColorForClass(detection.classId)
            canvas.drawRect(scaledRect, boxPaint)
            
            // Draw label with background
            val label = "${detection.label} ${(detection.score * 100).toInt()}%"
            val textBounds = Rect()
            textPaint.getTextBounds(label, 0, label.length, textBounds)
            
            val textX = scaledRect.left + 10
            val textY = scaledRect.top - 10
            
            val textBackground = RectF(
                textX - 5,
                textY - textBounds.height() - 5,
                textX + textBounds.width() + 10,
                textY + 5
            )
            
            textBackgroundPaint.color = getColorForClass(detection.classId)
            canvas.drawRect(textBackground, textBackgroundPaint)
            canvas.drawText(label, textX, textY, textPaint)
        }
    }

    private fun getColorForClass(classId: Int): Int {
        // Generate different colors for different classes
        val colors = arrayOf(
            Color.GREEN, Color.BLUE, Color.CYAN, Color.MAGENTA,
            Color.YELLOW, Color.RED, Color.rgb(255, 165, 0), // Orange
            Color.rgb(128, 0, 128) // Purple
        )
        return colors[classId % colors.size]
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
import android.util.Log
import androidx.camera.core.ImageProxy
import java.io.ByteArrayOutputStream
import java.io.File
import java.io.FileOutputStream
import java.nio.ByteBuffer

object Utils {

    private const val TAG = "Utils"

    fun assetFilePath(context: Context, assetName: String): String {
        val file = File(context.filesDir, assetName)
        
        if (file.exists() && file.length() > 0) {
            Log.d(TAG, "Model file already exists: ${file.absolutePath}")
            return file.absolutePath
        }

        try {
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
            Log.d(TAG, "Model file copied to: ${file.absolutePath}, size: ${file.length()} bytes")
        } catch (e: Exception) {
            Log.e(TAG, "Error copying model file", e)
            throw RuntimeException("Failed to copy model file", e)
        }
        
        return file.absolutePath
    }

    fun imageToBitmap(imageProxy: ImageProxy): Bitmap {
        return when (imageProxy.format) {
            ImageFormat.YUV_420_888 -> {
                yuvToBitmap(imageProxy)
            }
            ImageFormat.JPEG -> {
                val buffer = imageProxy.planes[0].buffer
                val bytes = ByteArray(buffer.remaining())
                buffer.get(bytes)
                BitmapFactory.decodeByteArray(bytes, 0, bytes.size)
            }
            else -> {
                throw IllegalArgumentException("Unsupported image format: ${imageProxy.format}")
            }
        }
    }

    private fun yuvToBitmap(imageProxy: ImageProxy): Bitmap {
        val yBuffer = imageProxy.planes[0].buffer
        val uBuffer = imageProxy.planes[1].buffer
        val vBuffer = imageProxy.planes[2].buffer

        val ySize = yBuffer.remaining()
        val uSize = uBuffer.remaining()
        val vSize = vBuffer.remaining()

        val nv21 = ByteArray(ySize + uSize + vSize)

        // U and V are swapped
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
        if (rotationDegrees == 0) return bitmap
        
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
            android:text="Detected: -"
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

## 🧩 Model Setup & Export

### **Step 1: Train Your YOLOv8 Model (Kaggle/Colab)**

Use your training script to train the BiSindo YOLOv8 model:

```python
from ultralytics import YOLO

# Train model
model = YOLO("yolov8m.pt")
results = model.train(
    data="bisindo.yaml",
    epochs=50,
    imgsz=416,  # Important: match Android input size
    batch=16,
    optimizer="AdamW",
    lr0=0.000333,
    name="bisindo_yolov8m",
    augment=True
)
```

### **Step 2: Export to TorchScript**

After training, export your best model to TorchScript format:

```python
import torch
from ultralytics import YOLO

# Load your trained model
model = YOLO("/path/to/best.pt")

# Export to TorchScript format
model.export(
    format='torchscript',
    imgsz=416,  # Must match your training size
    optimize=True,
    half=False  # Use False for better compatibility
)

print("✅ Model exported to TorchScript format!")
print("📁 Find: best.torchscript in the weights folder")
```

### **Step 3: Add Model to Android Assets**

1. After export, you'll find `best.torchscript` in your weights folder
2. Copy it to your Android project:
   ```
   app/src/main/assets/best.torchscript
   ```
3. Create the `assets` folder if it doesn't exist:
   ```
   app/src/main/assets/
   ```

---

## 🔧 Key Changes for YOLOv8 Integration

### **What's Different from YOLOv5?**

1. **Output Format**: YOLOv8 outputs `[1, num_classes+4, num_predictions]` instead of YOLOv5's format
2. **No Objectness Score**: YOLOv8 directly outputs class probabilities
3. **Center-based Bbox**: Uses (cx, cy, w, h) format
4. **Built-in NMS**: We implement custom NMS for mobile

### **Model Configuration**

```kotlin
// In YoloV8Detector.kt
private val inputSize = 416  // Match your export size
private const val CONF_THRESHOLD = 0.5f  // Adjust based on your needs
private const val IOU_THRESHOLD = 0.45f  // NMS threshold
```

### **Normalization**

```kotlin
// Input normalization: divide by 255 to get [0, 1] range
val inputTensor = TensorImageUtils.bitmapToFloat32Tensor(
    resizedBitmap,
    floatArrayOf(0f, 0f, 0f),      // No mean subtraction
    floatArrayOf(255f, 255f, 255f)  // Normalize to [0, 1]
)
```

---

## ▶️ Installation & Running

### **Prerequisites**

1. Install Android Studio Hedgehog or newer
2. Enable USB Debugging on your Android device
3. Train and export your YOLOv8 model to TorchScript

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
   - Place `best.torchscript` in `app/src/main/assets/`
   - Verify the file size (should be several MB)

4. **Sync Gradle**
   - Click "Sync Project with Gradle Files" in Android Studio
   - Wait for dependencies to download

5. **Connect your device**
   - Enable Developer Options and USB Debugging
   - Connect via USB cable
   - Authorize the computer if prompted

6. **Run the app**
   - Click the Run button ▶️ in Android Studio
   - Select your device
   - Grant camera permissions when prompted

---

## 🔧 Troubleshooting

### **Common Issues**

**1. Model not found error**
```
Error: Model file not found in assets
```
**Solution**: 
- Ensure `best.torchscript` is in `app/src/main/assets/`
- Check file name matches exactly (case-sensitive)
- Verify file is not corrupted (check file size)

**2. Out of memory error**
```
OutOfMemoryError during inference
```
**Solution**: 
- Reduce input size in `YoloV8Detector.kt`: `private val inputSize = 320`
- Re-export model with smaller size: `model.export(imgsz=320)`

**3. Low FPS (< 5 FPS)**
**Solution**: 
- Use a smaller model (yolov8n instead of yolov8m)
- Reduce input resolution to 320x320
- Enable GPU acceleration if available

**4. No detections appearing**
**Solution**: 
- Lower confidence threshold: `private const val CONF_THRESHOLD = 0.3f`
- Check if model was trained properly
- Verify lighting conditions are adequate

**5. Wrong detections or labels**
**Solution**: 
- Verify label order in `YoloV8Detector.kt` matches your training data
- Check your `bisindo.yaml` class order
- Ensure model was exported correctly

---

## 📊 Performance Optimization

### **Model Selection**

| Model | Size | Speed | Accuracy |
|-------|------|-------|----------|
| YOLOv8n | ~6 MB | Fast (20-30 FPS) | Good |
| YOLOv8s | ~22 MB | Medium (15-25 FPS) | Better |
| YOLOv8m | ~52 MB | Slow (10-15 FPS) | Best |

### **Recommended Settings**

```kotlin
// For real-time performance on mid-range devices
private val inputSize = 320
private const val CONF_THRESHOLD = 0.5f
private const val IOU_THRESHOLD = 0.45f

// For better accuracy on high-end devices
private val inputSize = 416
private const val CONF_THRESHOLD = 0.6f
private const val IOU_THRESHOLD = 0.45f
```

### **Tips for Better Performance**

1. **Use YOLOv8n or YOLOv8s** for mobile deployment
2. **Lower input resolution** (320x320) for faster inference
3. **Adjust confidence threshold** based on your use case
4. **Good lighting** significantly improves detection accuracy
5. **Stable camera** - avoid shaky movements during detection

---

## 🧪 Testing Your Model

### **Verify Model Output**

Add this to `YoloV8Detector.kt` for debugging:

```kotlin
Log.d(TAG, "Output shape: ${shape.contentToString()}")
Log.d(TAG, "Num predictions: $numPredictions")
Log.d(TAG, "Num classes: $numClasses")
Log.d(TAG, "Detections before NMS: ${detectionList.size}")
Log.d(TAG, "Detections after NMS: ${results.size}")
```

### **Expected Output**

- **Shape**: `[1, 30, 8400]` for 26 classes (4 bbox + 26 classes)
- **FPS**: 10-30 depending on device and model size
- **Detections**: Should see bounding boxes and labels on screen

---

## 🎯 Usage Tips

### **For Best Detection Results**

1. **Lighting**: Use good, even lighting
2. **Distance**: Keep hand 1-2 feet from camera
3. **Background**: Use plain, contrasting backgrounds
4. **Steady Hand**: Hold sign steady for 1-2 seconds
5. **Clear Signs**: Make clear, distinct hand gestures

### **App Controls**

- **Clear Button**: Clears the detected text display
- **FPS Counter**: Shows real-time inference speed (top-right)
- **Detection Display**: Shows detected letters with confidence scores (bottom)

---

## 🧱 Roadmap

- [x] YOLOv8 TorchScript integration
- [x] Real-time detection with NMS
- [x] BiSindo A-Z alphabet support
- [ ] Add front/back camera switch
- [ ] Implement confidence threshold slider in UI
- [ ] Add text-to-speech for detected signs
- [ ] Letter accumulation to form words
- [ ] Save detection history
- [ ] Export detected words/sentences
- [ ] Multi-language support (ID/EN)
- [ ] Dark/Light theme toggle
- [ ] Model update via OTA

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

- [Ultralytics YOLOv8](https://github.com/ultralytics/ultralytics)
- [PyTorch Mobile](https://pytorch.org/mobile/)
- [CameraX](https://developer.android.com/training/camerax)
- BiSindo community
- Indonesian Sign Language Dataset

---

## 📚 References

- **YOLOv8 Paper**: [Ultralytics YOLOv8 Docs](https://docs.ultralytics.com/)
- **PyTorch Mobile**: [PyTorch Mobile Documentation](https://pytorch.org/mobile/home/)
- **BiSindo**: [Indonesian Sign Language Research](https://en.wikipedia.org/wiki/Indonesian_Sign_Language)

---

**Developed with ❤️ using Kotlin, CameraX, PyTorch Mobile, and YOLOv8**

**Note**: This app requires a trained YOLOv8 model in TorchScript format. Train your model using the provided Python script, export to TorchScript, and place it in the assets folder.
