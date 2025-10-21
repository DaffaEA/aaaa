# Complete BiSindo Sign Language Detection App

## Project Structure
```
BiSindoDetector/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── kotlin/com/example/bisindodetector/
│   │   │   │   ├── MainActivity.kt
│   │   │   │   ├── YoloDetector.kt
│   │   │   │   └── OverlayView.kt
│   │   │   ├── res/
│   │   │   │   ├── layout/
│   │   │   │   │   └── activity_main.xml
│   │   │   │   ├── values/
│   │   │   │   │   ├── colors.xml
│   │   │   │   │   └── strings.xml
│   │   │   │   └── drawable/
│   │   │   ├── assets/
│   │   │   │   └── bisindo_model_416.torchscript
│   │   │   └── AndroidManifest.xml
│   │   └── build.gradle.kts
│   └── build.gradle.kts (Project level)
└── settings.gradle.kts
```

---

## File 1: build.gradle (Project level)

**File Path:** `build.gradle.kts`

```gradle
// Top-level build file where you can add configuration options common to all sub-projects/modules.
plugins {
    id("com.android.application") version "8.2.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.20" apply false
}

tasks.register("clean", Delete::class) {
    delete(rootProject.layout.buildDirectory)
}
```

---

## File 2: settings.gradle

**File Path:** `settings.gradle`

```gradle
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

rootProject.name = "BiSindo Detector"
include(":app")
```

---

## File 3: build.gradle (Module: app)

**File Path:** `app/build.gradle.kts(module: app)`

```gradle
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.bisindodetector"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.example.bisindodetector"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"

        packaging {
            resources {
                excludes += listOf("META-INF/NOTICE.md", "META-INF/LICENSE.md")
            }
        }
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
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = "1.8"
    }

    buildFeatures {
        viewBinding = true
    }

    androidResources {
        noCompress += "torchscript"
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    implementation("com.google.android.material:material:1.11.0")
    implementation("androidx.constraintlayout:constraintlayout:2.1.4")

    // CameraX dependencies
    implementation("androidx.camera:camera-camera2:1.3.1")
    implementation("androidx.camera:camera-lifecycle:1.3.1")
    implementation("androidx.camera:camera-view:1.3.1")

    // PyTorch Mobile
    implementation("org.pytorch:pytorch_android_lite:2.1.0")
    implementation("org.pytorch:pytorch_android_torchvision_lite:2.1.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")

    // Testing
    testImplementation("junit:junit:4.13.2")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
}
```

---

## File 4: AndroidManifest.xml

**File Path:** `app/src/main/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- Camera permission -->
    <uses-permission android:name="android.permission.CAMERA" />

    <!-- Camera features -->
    <uses-feature
        android:name="android.hardware.camera"
        android:required="true" />
    <uses-feature
        android:name="android.hardware.camera.front"
        android:required="false" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar"
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

## File 5: MainActivity.kt

**File Path:** `app/src/main/java/com/yourname/bisindodetector/MainActivity.kt`

```kt
package com.example.bisindodetector

import android.Manifest
import android.content.pm.PackageManager
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import android.graphics.Matrix
import android.graphics.Rect
import android.graphics.YuvImage
import android.os.Bundle
import android.util.Log
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.core.content.ContextCompat
import com.example.bisindodetector.databinding.ActivityMainBinding
import kotlinx.coroutines.*
import java.io.ByteArrayOutputStream
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors

class MainActivity : AppCompatActivity() {

    companion object {
        private const val TAG = "BiSindoDetector"
        private const val LETTER_HOLD_TIME = 1500L // 1.5 seconds
    }

    private lateinit var binding: ActivityMainBinding
    private lateinit var cameraExecutor: ExecutorService
    private var detector: YoloDetector? = null

    // Word building logic
    private var lastDetectedLetter = ""
    private var lastDetectionTime = 0L
    private val detectedWord = StringBuilder()

    // FPS calculation
    private var frameCount = 0L
    private var lastFpsTime = System.currentTimeMillis()
    private var currentFps = 0

    // Camera permission launcher
    private val requestPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            startCamera()
        } else {
            Toast.makeText(
                this,
                "Camera permission is required",
                Toast.LENGTH_LONG
            ).show()
            finish()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Initialize camera executor
        cameraExecutor = Executors.newSingleThreadExecutor()

        // Setup clear button
        binding.clearButton.setOnClickListener {
            detectedWord.clear()
            binding.detectedText.text = "Detected: "
            lastDetectedLetter = ""
            Toast.makeText(this, "Cleared!", Toast.LENGTH_SHORT).show()
        }

        // Load YOLO detector
        try {
            detector = YoloDetector(this, "bisindo_model_416.torchscript", 416)
            Log.d(TAG, "Model loaded successfully")
            Toast.makeText(this, "Model loaded successfully", Toast.LENGTH_SHORT).show()
        } catch (e: Exception) {
            Log.e(TAG, "Error loading model", e)
            Toast.makeText(
                this,
                "Error loading model: ${e.message}",
                Toast.LENGTH_LONG
            ).show()
            finish()
            return
        }

        // Check and request camera permission
        when {
            ContextCompat.checkSelfPermission(
                this,
                Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED -> {
                startCamera()
            }
            else -> {
                requestPermissionLauncher.launch(Manifest.permission.CAMERA)
            }
        }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            try {
                val cameraProvider = cameraProviderFuture.get()
                bindCameraUseCases(cameraProvider)
            } catch (e: Exception) {
                Log.e(TAG, "Error starting camera", e)
                Toast.makeText(this, "Error starting camera", Toast.LENGTH_SHORT).show()
            }
        }, ContextCompat.getMainExecutor(this))
    }

    @androidx.annotation.OptIn(androidx.camera.core.ExperimentalGetImage::class)
    private fun bindCameraUseCases(cameraProvider: ProcessCameraProvider) {
        // Preview use case
        val preview = Preview.Builder()
            .build()
            .also {
                it.setSurfaceProvider(binding.previewView.surfaceProvider)
            }

        // Image analysis use case
        val imageAnalysis = ImageAnalysis.Builder()
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .setOutputImageFormat(ImageAnalysis.OUTPUT_IMAGE_FORMAT_YUV_420_888)
            .build()

        imageAnalysis.setAnalyzer(cameraExecutor) { imageProxy ->
            analyzeImage(imageProxy)
        }

        // Select front camera
        val cameraSelector = CameraSelector.Builder()
            .requireLensFacing(CameraSelector.LENS_FACING_FRONT)
            .build()

        try {
            // Unbind all use cases before rebinding
            cameraProvider.unbindAll()

            // Bind use cases to camera
            cameraProvider.bindToLifecycle(
                this,
                cameraSelector,
                preview,
                imageAnalysis
            )

            Log.d(TAG, "Camera use cases bound successfully")
        } catch (e: Exception) {
            Log.e(TAG, "Use case binding failed", e)
            Toast.makeText(this, "Camera binding failed", Toast.LENGTH_SHORT).show()
        }
    }

    @androidx.camera.core.ExperimentalGetImage
    private fun analyzeImage(imageProxy: ImageProxy) {
        // Calculate FPS
        frameCount++
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastFpsTime >= 1000) {
            currentFps = frameCount.toInt()
            frameCount = 0
            lastFpsTime = currentTime

            runOnUiThread {
                binding.fpsText.text = "FPS: $currentFps"
            }
        }

        // Convert ImageProxy to Bitmap
        val bitmap = imageProxyToBitmap(imageProxy)
        if (bitmap == null) {
            imageProxy.close()
            return
        }

        // Run detection
        val detections = detector?.detect(bitmap) ?: emptyList()

        // Update UI on main thread
        runOnUiThread {
            // Update overlay with bounding boxes
            binding.overlayView.setDetections(detections)

            // Process detections for word building
            if (detections.isNotEmpty()) {
                // Get the detection with highest confidence (already sorted)
                val bestDetection = detections.first()
                processDetection(bestDetection)
            } else {
                // No detection - reset
                lastDetectedLetter = ""
            }
        }

        // Clean up
        imageProxy.close()
    }

    private fun processDetection(detection: YoloDetector.Detection) {
        val currentTime = System.currentTimeMillis()
        val detectedLetter = detection.label

        if (detectedLetter == lastDetectedLetter) {
            // Same letter detected - check if held long enough
            val holdDuration = currentTime - lastDetectionTime

            if (holdDuration >= LETTER_HOLD_TIME) {
                // Add letter to word
                detectedWord.append(detectedLetter)
                binding.detectedText.text = "Detected: $detectedWord"

                // Reset detection to avoid repeated additions
                lastDetectionTime = currentTime

                // Visual feedback
                binding.overlayView.flashGreen()

                Log.d(TAG, "Added letter: $detectedLetter | Word: $detectedWord")
            }
        } else {
            // New letter detected
            lastDetectedLetter = detectedLetter
            lastDetectionTime = currentTime
        }
    }

    @androidx.camera.core.ExperimentalGetImage
    private fun imageProxyToBitmap(imageProxy: ImageProxy): Bitmap? {
        val image = imageProxy.image ?: return null

        return try {
            // Get image planes
            val yBuffer = image.planes[0].buffer
            val uBuffer = image.planes[1].buffer
            val vBuffer = image.planes[2].buffer

            val ySize = yBuffer.remaining()
            val uSize = uBuffer.remaining()
            val vSize = vBuffer.remaining()

            // Convert to NV21 format
            val nv21 = ByteArray(ySize + uSize + vSize)
            yBuffer.get(nv21, 0, ySize)
            vBuffer.get(nv21, ySize, vSize)
            uBuffer.get(nv21, ySize + vSize, uSize)

            // Convert NV21 to JPEG
            val yuvImage = YuvImage(
                nv21,
                android.graphics.ImageFormat.NV21,
                imageProxy.width,
                imageProxy.height,
                null
            )
            val out = ByteArrayOutputStream()
            yuvImage.compressToJpeg(
                Rect(0, 0, imageProxy.width, imageProxy.height),
                100,
                out
            )

            val imageBytes = out.toByteArray()
            val bitmap = BitmapFactory.decodeByteArray(imageBytes, 0, imageBytes.size)

            // Mirror the image for front camera
            val matrix = Matrix().apply {
                preScale(-1.0f, 1.0f)
            }
            Bitmap.createBitmap(
                bitmap,
                0,
                0,
                bitmap.width,
                bitmap.height,
                matrix,
                false
            )
        } catch (e: Exception) {
            Log.e(TAG, "Error converting ImageProxy to Bitmap", e)
            null
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
        detector?.close()
    }
}
```

---

## File 6: YoloDetector.kt

**File Path:** `app/src/main/java/com/yourname/bisindodetector/YoloDetector.kt`

```kt
package com.example.bisindodetector

import android.content.Context
import android.graphics.Bitmap
import android.graphics.RectF
import android.util.Log
import org.pytorch.IValue
import org.pytorch.Module
import org.pytorch.Tensor
import org.pytorch.torchvision.TensorImageUtils
import java.io.File
import java.io.FileOutputStream
import java.io.IOException

class YoloDetector(
    context: Context,
    modelName: String,
    private val inputSize: Int
) {
    companion object {
        private const val TAG = "YoloDetector"
        
        // ImageNet normalization
        private val MEAN = floatArrayOf(0.485f, 0.456f, 0.406f)
        private val STD = floatArrayOf(0.229f, 0.224f, 0.225f)
    }

    data class Detection(
        val box: RectF,
        val label: String,
        val confidence: Float
    )

    private val model: Module
    private var confidenceThreshold = 0.45f
    private var iouThreshold = 0.4f

    // BiSindo alphabet labels - UPDATE THESE to match your model's classes
    private val labels = arrayOf(
        "A", "B", "C", "D", "E", "F", "G", "H", "I", "J",
        "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T",
        "U", "V", "W", "X", "Y", "Z"
    )

    init {
        val modelPath = assetFilePath(context, modelName)
        model = Module.load(modelPath)

        Log.d(TAG, "Model loaded: $modelPath")
        Log.d(TAG, "Input size: ${inputSize}x$inputSize")
        Log.d(TAG, "Number of classes: ${labels.size}")
    }

    /**
     * Copy model from assets to internal storage
     */
    private fun assetFilePath(context: Context, assetName: String): String {
        val file = File(context.filesDir, assetName)

        // If file already exists and has content, return it
        if (file.exists() && file.length() > 0) {
            Log.d(TAG, "Model already exists in internal storage")
            return file.absolutePath
        }

        // Copy from assets to internal storage
        Log.d(TAG, "Copying model from assets to internal storage...")
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

        Log.d(TAG, "Model copied successfully")
        return file.absolutePath
    }

    /**
     * Detect BiSindo signs in the image
     */
    fun detect(bitmap: Bitmap): List<Detection> {
        // Resize bitmap to model input size
        val resizedBitmap = Bitmap.createScaledBitmap(bitmap, inputSize, inputSize, true)

        // Convert to tensor with normalization
        val inputTensor = TensorImageUtils.bitmapToFloat32Tensor(
            resizedBitmap,
            MEAN,
            STD
        )

        // Run inference
        val outputTensor = model.forward(IValue.from(inputTensor)).toTensor()

        // Get output array
        val outputs = outputTensor.dataAsFloatArray
        val shape = outputTensor.shape()

        // Parse output shape
        val batchSize = shape[0].toInt()
        val numPredictions = shape[1].toInt()
        val numValues = shape[2].toInt() // 5 + num_classes

        Log.d(TAG, "Output shape: [$batchSize, $numPredictions, $numValues]")

        // Process predictions
        val detections = processOutputs(
            outputs,
            numPredictions,
            numValues,
            bitmap.width,
            bitmap.height
        )

        // Apply Non-Maximum Suppression
        return nonMaxSuppression(detections)
    }

    /**
     * Process model outputs to extract detections
     */
    private fun processOutputs(
        outputs: FloatArray,
        numPredictions: Int,
        numValues: Int,
        originalWidth: Int,
        originalHeight: Int
    ): List<Detection> {
        val detections = mutableListOf<Detection>()

        for (i in 0 until numPredictions) {
            val offset = i * numValues

            // YOLO output format: [x_center, y_center, width, height, objectness, class_scores...]
            val xCenter = outputs[offset]
            val yCenter = outputs[offset + 1]
            val width = outputs[offset + 2]
            val height = outputs[offset + 3]
            val objectness = outputs[offset + 4]

            // Skip low confidence predictions
            if (objectness < confidenceThreshold) {
                continue
            }

            // Find class with highest score
            var classIndex = 0
            var maxClassScore = outputs[offset + 5]

            for (j in 1 until labels.size) {
                if (offset + 5 + j < outputs.size) {
                    val score = outputs[offset + 5 + j]
                    if (score > maxClassScore) {
                        maxClassScore = score
                        classIndex = j
                    }
                }
            }

            // Calculate final confidence
            val confidence = objectness * maxClassScore

            if (confidence < confidenceThreshold) {
                continue
            }

            // Convert from normalized coordinates to pixel coordinates
            val scaleX = originalWidth.toFloat() / inputSize
            val scaleY = originalHeight.toFloat() / inputSize

            val pixelXCenter = xCenter * scaleX
            val pixelYCenter = yCenter * scaleY
            val pixelWidth = width * scaleX
            val pixelHeight = height * scaleY

            var left = pixelXCenter - pixelWidth / 2
            var top = pixelYCenter - pixelHeight / 2
            var right = pixelXCenter + pixelWidth / 2
            var bottom = pixelYCenter + pixelHeight / 2

            // Clamp to image boundaries
            left = left.coerceIn(0f, originalWidth.toFloat())
            top = top.coerceIn(0f, originalHeight.toFloat())
            right = right.coerceIn(0f, originalWidth.toFloat())
            bottom = bottom.coerceIn(0f, originalHeight.toFloat())

            val box = RectF(left, top, right, bottom)
            val label = if (classIndex < labels.size) labels[classIndex] else "Unknown"

            detections.add(Detection(box, label, confidence))
        }

        Log.d(TAG, "Found ${detections.size} detections before NMS")
        return detections
    }

    /**
     * Apply Non-Maximum Suppression to remove overlapping boxes
     */
    private fun nonMaxSuppression(detections: List<Detection>): List<Detection> {
        if (detections.isEmpty()) {
            return detections
        }

        val result = mutableListOf<Detection>()
        val sortedDetections = detections.sortedByDescending { it.confidence }.toMutableList()

        while (sortedDetections.isNotEmpty()) {
            // Take the detection with highest confidence
            val best = sortedDetections.removeAt(0)
            result.add(best)

            // Remove detections that overlap significantly with this one
            sortedDetections.removeAll { detection ->
                calculateIoU(best.box, detection.box) > iouThreshold
            }
        }

        Log.d(TAG, "After NMS: ${result.size} detections")
        return result
    }

    /**
     * Calculate Intersection over Union between two boxes
     */
    private fun calculateIoU(box1: RectF, box2: RectF): Float {
        val intersectLeft = maxOf(box1.left, box2.left)
        val intersectTop = maxOf(box1.top, box2.top)
        val intersectRight = minOf(box1.right, box2.right)
        val intersectBottom = minOf(box1.bottom, box2.bottom)

        val intersectWidth = maxOf(0f, intersectRight - intersectLeft)
        val intersectHeight = maxOf(0f, intersectBottom - intersectTop)
        val intersectArea = intersectWidth * intersectHeight

        val box1Area = (box1.right - box1.left) * (box1.bottom - box1.top)
        val box2Area = (box2.right - box2.left) * (box2.bottom - box2.top)
        val unionArea = box1Area + box2Area - intersectArea

        return intersectArea / unionArea
    }

    fun setConfidenceThreshold(threshold: Float) {
        confidenceThreshold = threshold
        Log.d(TAG, "Confidence threshold set to: $threshold")
    }

    fun setIouThreshold(threshold: Float) {
        iouThreshold = threshold
        Log.d(TAG, "IoU threshold set to: $threshold")
    }

    fun close() {
        // Clean up resources if needed
        Log.d(TAG, "Detector closed")
    }
}
```

---

## File 7: OverlayView.kt

**File Path:** `app/src/main/java/com/yourname/bisindodetector/OverlayView.kt`

```kt
package com.example.bisindodetector

import android.content.Context
import android.graphics.Canvas
import android.graphics.Color
import android.graphics.Paint
import android.graphics.RectF
import android.util.AttributeSet
import android.view.View

class OverlayView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    companion object {
        private const val FLASH_DURATION = 300L // ms
    }

    private var detections: List<YoloDetector.Detection> = emptyList()
    private val boxPaint: Paint
    private val textPaint: Paint
    private val backgroundPaint: Paint
    private val fillPaint: Paint

    private var flashEffect = false
    private var flashStartTime = 0L

    init {
        // Bounding box paint
        boxPaint = Paint().apply {
            color = Color.GREEN
            style = Paint.Style.STROKE
            strokeWidth = 6f
            isAntiAlias = true
        }

        // Text paint
        textPaint = Paint().apply {
            color = Color.WHITE
            textSize = 48f
            style = Paint.Style.FILL
            isAntiAlias = true
            isFakeBoldText = true
        }

        // Background for text
        backgroundPaint = Paint().apply {
            color = Color.GREEN
            style = Paint.Style.FILL
            isAntiAlias = true
        }

        // Semi-transparent fill for boxes
        fillPaint = Paint().apply {
            color = Color.argb(30, 0, 255, 0)
            style = Paint.Style.FILL
            isAntiAlias = true
        }
    }

    fun setDetections(detections: List<YoloDetector.Detection>) {
        this.detections = detections
        invalidate() // Redraw
    }

    fun flashGreen() {
        flashEffect = true
        flashStartTime = System.currentTimeMillis()
        invalidate()
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        // Draw flash effect if active
        if (flashEffect) {
            val elapsed = System.currentTimeMillis() - flashStartTime
            if (elapsed < FLASH_DURATION) {
                // Fade out effect
                val alpha = (255 * (1 - elapsed.toFloat() / FLASH_DURATION)).toInt()
                val flashPaint = Paint().apply {
                    color = Color.argb(alpha / 2, 0, 255, 0)
                }
                canvas.drawRect(0f, 0f, width.toFloat(), height.toFloat(), flashPaint)
                invalidate() // Continue animation
            } else {
                flashEffect = false
            }
        }

        // Draw all detections
        for (detection in detections) {
            // Draw semi-transparent fill
            canvas.drawRect(detection.box, fillPaint)

            // Draw bounding box
            canvas.drawRect(detection.box, boxPaint)

            // Prepare label text
            val labelText = "${detection.label} ${String.format("%.0f%%", detection.confidence * 100)}"

            // Measure text dimensions
            val textWidth = textPaint.measureText(labelText)
            val textHeight = textPaint.textSize

            // Draw background rectangle for text
            val padding = 10f
            val textBackground = RectF(
                detection.box.left,
                detection.box.top - textHeight - padding * 2,
                detection.box.left + textWidth + padding * 2,
                detection.box.top
            )

            // Make sure text background is within view bounds
            if (textBackground.top < 0) {
                textBackground.offset(0f, -textBackground.top + detection.box.top + 5)
            }

            canvas.drawRect(textBackground, backgroundPaint)

            // Draw text
            canvas.drawText(
                labelText,
                textBackground.left + padding,
                textBackground.bottom - padding,
                textPaint
            )
        }
    }
}
```

---

## File 8: activity_main.xml

**File Path:** `app/src/main/res/layout/activity_main.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/black">

    <!-- Camera Preview -->
    <androidx.camera.view.PreviewView
        android:id="@+id/previewView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintBottom_toTopOf="@id/bottomPanel"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <!-- Overlay for bounding boxes -->
    <com.example.bisindodetector.OverlayView
        android:id="@+id/overlayView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintBottom_toBottomOf="@id/previewView"
        app:layout_constraintEnd_toEndOf="@id/previewView"
        app:layout_constraintStart_toStartOf="@id/previewView"
        app:layout_constraintTop_toTopOf="@id/previewView" />

    <!-- FPS Counter -->
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
        android:orientation="vertical"
        android:padding="16dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent">

        <!-- Detected Text Display -->
        <TextView
            android:id="@+id/detectedText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="12dp"
            android:background="@color/white"
            android:minHeight="60dp"
            android:padding="12dp"
            android:text="Detected: "
            android:textColor="@color/black"
            android:textSize="20sp"
            android:textStyle="bold" />

        <!-- Control Buttons -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:gravity="center"
            android:orientation="horizontal">

            <!-- Clear Button -->
            <Button
                android:id="@+id/clearButton"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:backgroundTint="@color/red"
                android:text="Clear"
                android:textColor="@color/white"
                android:textStyle="bold" />

        </LinearLayout>
    </LinearLayout>

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

## File 9: strings.xml

**File Path:** `app/src/main/res/values/strings.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">BiSindo Detector</string>
    <string name="camera_permission_required">Camera permission is required for sign language detection</string>
    <string name="model_load_failed">Failed to load detection model</string>
    <string name="camera_init_failed">Camera initialization failed</string>
    <string name="detected_prefix">Detected: </string>
    <string name="fps_format">FPS: %d</string>
    <string name="clear_button">Clear</string>
    <string name="space_button">Space</string>
    <string name="backspace_button">⌫</string>
</resources>
```

---

## File 10: colors.xml

**File Path:** `app/src/main/res/values/colors.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
    <color name="green">#FF00FF00</color>
    <color name="red">#FFFF0000</color>
    <color name="semi_transparent_black">#80000000</color>
    <color name="semi_transparent_green">#8000FF00</color>
    
    <!-- Material Design Colors -->
    <color name="purple_200">#FFBB86FC</color>
    <color name="purple_500">#FF6200EE</color>
    <color name="purple_700">#FF3700B3</color>
    <color name="teal_200">#FF03DAC5</color>
    <color name="teal_700">#FF018786</color>
</resources>
```
## gradle/libs.version.toml
```toml
[versions]
agp = "8.2.0"
kotlin = "1.9.20"
coreKtx = "1.12.0"
junit = "4.13.2"
junitVersion = "1.1.5"
espressoCore = "3.5.1"
appcompat = "1.6.1"
material = "1.11.0"
constraintlayout = "2.1.4"
camerax = "1.3.1"
pytorch = "2.1.0"
coroutines = "1.7.3"

[libraries]
core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
junit = { group = "junit", name = "junit", version.ref = "junit" }
ext-junit = { group = "androidx.test.ext", name = "junit", version.ref = "junitVersion" }
espresso-core = { group = "androidx.test.espresso", name = "espresso-core", version.ref = "espressoCore" }
appcompat = { group = "androidx.appcompat", name = "appcompat", version.ref = "appcompat" }
material = { group = "com.google.android.material", name = "material", version.ref = "material" }
constraintlayout = { group = "androidx.constraintlayout", name = "constraintlayout", version.ref = "constraintlayout" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
jetbrains-kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

## Proguard.rules.pro
```pro
# Add project specific ProGuard rules here.
# You can control the set of applied configuration files using the
# proguardFiles setting in build.gradle.

# PyTorch
-keep class org.pytorch.** { *; }
-keep class com.facebook.jni.** { *; }

# Keep model classes
-keep class com.example.bisindodetector.YoloDetector { *; }
-keep class com.example.bisindodetector.YoloDetector$Detection { *; }

# CameraX
-keep class androidx.camera.** { *; }
```

## gradle.properties
```properties
# Project-wide Gradle settings.
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
android.enableJetifier=true
kotlin.code.style=official
```
