# Complete BiSindo Sign Language Detection App

## Project Structure
```
BiSindoDetector/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/yourname/bisindodetector/
│   │   │   │   ├── MainActivity.java
│   │   │   │   ├── YoloDetector.java
│   │   │   │   └── OverlayView.java
│   │   │   ├── res/
│   │   │   │   ├── layout/
│   │   │   │   │   └── activity_main.xml
│   │   │   │   ├── values/
│   │   │   │   │   ├── colors.xml
│   │   │   │   │   └── strings.xml
│   │   │   │   └── drawable/
│   │   │   │       └── ic_launcher_background.xml
│   │   │   ├── assets/
│   │   │   │   └── bisindo_model_416.torchscript  ← PUT YOUR MODEL HERE
│   │   │   └── AndroidManifest.xml
│   │   └── build.gradle
│   └── build.gradle (Project level)
└── settings.gradle
```

---

## File 1: build.gradle (Project level)

**File Path:** `build.gradle.kts`

```gradle
// Top-level build file where you can add configuration options common to all sub-projects/modules.
plugins {
    id("com.android.application") version "8.12.3" apply false
}

tasks.register("clean", Delete::class){
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
}

android {
    namespace = "com.example.bisindodetector"
    compileSdk = 36

    defaultConfig {
        applicationId = "com.example.bisindodetector"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"

        // Don't compress model file
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

    // Prevent compression of torchscript files
    androidResources {
        noCompress += "torchscript"
    }
}

dependencies {
    implementation(libs.appcompat)
    implementation(libs.material)
    implementation(libs.constraintlayout)

    // CameraX dependencies
    implementation("androidx.camera:camera-camera2:1.5.1")
    implementation("androidx.camera:camera-lifecycle:1.5.1")
    implementation("androidx.camera:camera-view:1.5.1")

    // PyTorch Mobile
    implementation("org.pytorch:pytorch_android_lite:2.1.0")
    implementation("org.pytorch:pytorch_android_torchvision_lite:2.1.0")

    // Testing
    testImplementation(libs.junit)
    androidTestImplementation(libs.ext.junit)
    androidTestImplementation(libs.espresso.core)
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

    <!-- Camera feature -->
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

## File 5: MainActivity.java

**File Path:** `app/src/main/java/com/yourname/bisindodetector/MainActivity.java`

```java
package com.example.bisindodetector;

import androidx.annotation.NonNull;
import androidx.annotation.OptIn;
import androidx.appcompat.app.AppCompatActivity;
import androidx.camera.core.*;
import androidx.camera.lifecycle.ProcessCameraProvider;
import androidx.camera.view.PreviewView;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import android.Manifest;
import android.content.pm.PackageManager;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.graphics.Matrix;
import android.graphics.Rect;
import android.graphics.YuvImage;
import android.media.Image;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;

import com.google.common.util.concurrent.ListenableFuture;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.util.List;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "BiSindoDetector";
    private static final int PERMISSION_REQUEST_CODE = 100;

    // UI Components
    private PreviewView previewView;
    private OverlayView overlayView;
    private TextView detectedTextView;
    private TextView fpsTextView;
    private Button clearButton;

    // Detection
    private YoloDetector detector;
    private ExecutorService cameraExecutor;

    // Word building logic
    private String lastDetectedLetter = "";
    private long lastDetectionTime = 0;
    private long letterHoldTime = 1500; // Hold letter for 1.5 seconds to add
    private StringBuilder detectedWord = new StringBuilder();

    // FPS calculation
    private long frameCount = 0;
    private long lastFpsTime = System.currentTimeMillis();
    private int currentFps = 0;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Initialize UI components
        previewView = findViewById(R.id.previewView);
        overlayView = findViewById(R.id.overlayView);
        detectedTextView = findViewById(R.id.detectedText);
        fpsTextView = findViewById(R.id.fpsText);
        clearButton = findViewById(R.id.clearButton);

        // Clear button click listener
        clearButton.setOnClickListener(v -> {
            detectedWord.setLength(0);
            detectedTextView.setText("Detected: ");
            lastDetectedLetter = "";
            Toast.makeText(this, "Cleared!", Toast.LENGTH_SHORT).show();
        });

        // Initialize camera executor
        cameraExecutor = Executors.newSingleThreadExecutor();

        // Load YOLO detector
        try {
            detector = new YoloDetector(this, "bisindo_model_416.torchscript", 416);
            Log.d(TAG, "Model loaded successfully");
            Toast.makeText(this, "Model loaded successfully", Toast.LENGTH_SHORT).show();
        } catch (IOException e) {
            Log.e(TAG, "Error loading model", e);
            Toast.makeText(this, "Error loading model: " + e.getMessage(),
                    Toast.LENGTH_LONG).show();
            finish();
            return;
        }

        // Check and request camera permission
        if (checkPermissions()) {
            startCamera();
        } else {
            requestPermissions();
        }
    }

    private boolean checkPermissions() {
        return ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
                == PackageManager.PERMISSION_GRANTED;
    }

    private void requestPermissions() {
        ActivityCompat.requestPermissions(this,
                new String[]{Manifest.permission.CAMERA},
                PERMISSION_REQUEST_CODE);
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
                                           @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == PERMISSION_REQUEST_CODE) {
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                startCamera();
            } else {
                Toast.makeText(this, "Camera permission is required",
                        Toast.LENGTH_LONG).show();
                finish();
            }
        }
    }

    private void startCamera() {
        ListenableFuture<ProcessCameraProvider> cameraProviderFuture = ProcessCameraProvider.getInstance(this); // Specify the generic type here

        cameraProviderFuture.addListener(() -> {
            try {
                // Now the compiler knows .get() will return a ProcessCameraProvider
                ProcessCameraProvider cameraProvider = cameraProviderFuture.get();
                bindCameraUseCases(cameraProvider);
            } catch (ExecutionException | InterruptedException e) {
                Log.e(TAG, "Error starting camera", e);
                Toast.makeText(this, "Error starting camera", Toast.LENGTH_SHORT).show();
            }
        }, ContextCompat.getMainExecutor(this));
        }


    @OptIn(markerClass = ExperimentalGetImage.class) private void bindCameraUseCases(ProcessCameraProvider cameraProvider) {
        // Preview use case
        Preview preview = new Preview.Builder()
                .build();
        preview.setSurfaceProvider(previewView.getSurfaceProvider());

        // Image analysis use case
        ImageAnalysis imageAnalysis = new ImageAnalysis.Builder()
                .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                .setOutputImageFormat(ImageAnalysis.OUTPUT_IMAGE_FORMAT_YUV_420_888)
                .build();

        imageAnalysis.setAnalyzer(cameraExecutor, this::analyzeImage);

        // Select front camera
        CameraSelector cameraSelector = new CameraSelector.Builder()
                .requireLensFacing(CameraSelector.LENS_FACING_FRONT)
                .build();

        try {
            // Unbind all use cases before rebinding
            cameraProvider.unbindAll();

            // Bind use cases to camera
            cameraProvider.bindToLifecycle(
                    this,
                    cameraSelector,
                    preview,
                    imageAnalysis
            );

            Log.d(TAG, "Camera use cases bound successfully");
        } catch (Exception e) {
            Log.e(TAG, "Use case binding failed", e);
            Toast.makeText(this, "Camera binding failed", Toast.LENGTH_SHORT).show();
        }
    }

    @androidx.camera.core.ExperimentalGetImage
    private void analyzeImage(ImageProxy imageProxy) {
        // Calculate FPS
        frameCount++;
        long currentTime = System.currentTimeMillis();
        if (currentTime - lastFpsTime >= 1000) {
            currentFps = (int) frameCount;
            frameCount = 0;
            lastFpsTime = currentTime;

            runOnUiThread(() -> fpsTextView.setText("FPS: " + currentFps));
        }

        // Convert ImageProxy to Bitmap
        Bitmap bitmap = imageProxyToBitmap(imageProxy);
        if (bitmap == null) {
            imageProxy.close();
            return;
        }
// ...
        // Run detection
        List<YoloDetector.Detection> detections = detector.detect(bitmap); // <- Tambahkan tipe generik

        // Update UI on main thread
        runOnUiThread(() -> {
            // Update overlay with bounding boxes
            overlayView.setDetections(detections);

            // Process detections for word building
            if (!detections.isEmpty()) {
                // Get the detection with highest confidence
                // Kode ini sekarang akan berfungsi dengan baik
                YoloDetector.Detection bestDetection = detections.get(0);
                processDetection(bestDetection);
            } else {
                // No detection - reset
                lastDetectedLetter = "";
            }
        });

        // Clean up
        imageProxy.close();
    }

    private void processDetection(YoloDetector.Detection detection) {
        long currentTime = System.currentTimeMillis();
        String detectedLetter = detection.label;

        if (detectedLetter.equals(lastDetectedLetter)) {
            // Same letter detected - check if held long enough
            long holdDuration = currentTime - lastDetectionTime;

            if (holdDuration >= letterHoldTime) {
                // Add letter to word
                detectedWord.append(detectedLetter);
                detectedTextView.setText("Detected: " + detectedWord.toString());

                // Reset detection to avoid repeated additions
                lastDetectionTime = currentTime;

                // Visual feedback
                overlayView.flashGreen();

                Log.d(TAG, "Added letter: " + detectedLetter + " | Word: " + detectedWord);
            }
        } else {
            // New letter detected
            lastDetectedLetter = detectedLetter;
            lastDetectionTime = currentTime;
        }
    }

    @androidx.camera.core.ExperimentalGetImage
    private Bitmap imageProxyToBitmap(ImageProxy imageProxy) {
        Image image = imageProxy.getImage();
        if (image == null) return null;

        try {
            // Get image planes
            ByteBuffer yBuffer = image.getPlanes()[0].getBuffer();
            ByteBuffer uBuffer = image.getPlanes()[1].getBuffer();
            ByteBuffer vBuffer = image.getPlanes()[2].getBuffer();

            int ySize = yBuffer.remaining();
            int uSize = uBuffer.remaining();
            int vSize = vBuffer.remaining();

            // Convert to NV21 format
            byte[] nv21 = new byte[ySize + uSize + vSize];
            yBuffer.get(nv21, 0, ySize);
            vBuffer.get(nv21, ySize, vSize);
            uBuffer.get(nv21, ySize + vSize, uSize);

            // Convert NV21 to JPEG
            YuvImage yuvImage = new YuvImage(nv21, android.graphics.ImageFormat.NV21,
                    imageProxy.getWidth(), imageProxy.getHeight(), null);
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            yuvImage.compressToJpeg(new Rect(0, 0, imageProxy.getWidth(),
                    imageProxy.getHeight()), 100, out);

            byte[] imageBytes = out.toByteArray();
            Bitmap bitmap = BitmapFactory.decodeByteArray(imageBytes, 0, imageBytes.length);

            // Mirror the image for front camera (so it looks natural)
            Matrix matrix = new Matrix();
            matrix.preScale(-1.0f, 1.0f);
            return Bitmap.createBitmap(bitmap, 0, 0, bitmap.getWidth(),
                    bitmap.getHeight(), matrix, false);
        } catch (Exception e) {
            Log.e(TAG, "Error converting ImageProxy to Bitmap", e);
            return null;
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        cameraExecutor.shutdown();
        if (detector != null) {
            detector.close();
        }
    }

    @Override
    protected void onPause() {
        super.onPause();
        // You can pause detection here if needed
    }

    @Override
    protected void onResume() {
        super.onResume();
        // Resume detection if paused
    }
}
```

---

## File 6: YoloDetector.java

**File Path:** `app/src/main/java/com/yourname/bisindodetector/YoloDetector.java`

```java
package com.yourname.bisindodetector;

import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.RectF;
import android.util.Log;

import org.pytorch.IValue;
import org.pytorch.Module;
import org.pytorch.Tensor;
import org.pytorch.torchvision.TensorImageUtils;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.util.ArrayList;
import java.util.List;

public class YoloDetector {
    private static final String TAG = "YoloDetector";
    
    private Module model;
    private int inputSize;
    private float confidenceThreshold = 0.45f;
    private float iouThreshold = 0.4f;
    
    // BiSindo alphabet labels - UPDATE THESE to match your model's classes
    private String[] labels = {
        "A", "B", "C", "D", "E", "F", "G", "H", "I", "J", 
        "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", 
        "U", "V", "W", "X", "Y", "Z"
    };

    // ImageNet normalization (standard for YOLO)
    private static final float[] MEAN = {0.485f, 0.456f, 0.406f};
    private static final float[] STD = {0.229f, 0.224f, 0.225f};

    public static class Detection {
        public RectF box;
        public String label;
        public float confidence;

        public Detection(RectF box, String label, float confidence) {
            this.box = box;
            this.label = label;
            this.confidence = confidence;
        }
    }

    public YoloDetector(Context context, String modelName, int inputSize) throws IOException {
        this.inputSize = inputSize;
        
        // Load model from assets
        String modelPath = assetFilePath(context, modelName);
        model = Module.load(modelPath);
        
        Log.d(TAG, "Model loaded: " + modelPath);
        Log.d(TAG, "Input size: " + inputSize + "x" + inputSize);
        Log.d(TAG, "Number of classes: " + labels.length);
    }

    /**
     * Copy model from assets to internal storage
     */
    private String assetFilePath(Context context, String assetName) throws IOException {
        File file = new File(context.getFilesDir(), assetName);
        
        // If file already exists and has content, return it
        if (file.exists() && file.length() > 0) {
            Log.d(TAG, "Model already exists in internal storage");
            return file.getAbsolutePath();
        }

        // Copy from assets to internal storage
        Log.d(TAG, "Copying model from assets to internal storage...");
        try (InputStream is = context.getAssets().open(assetName);
             OutputStream os = new FileOutputStream(file)) {
            
            byte[] buffer = new byte[4 * 1024];
            int read;
            while ((read = is.read(buffer)) != -1) {
                os.write(buffer, 0, read);
            }
            os.flush();
        }
        
        Log.d(TAG, "Model copied successfully");
        return file.getAbsolutePath();
    }

    /**
     * Detect BiSindo signs in the image
     */
    public List detect(Bitmap bitmap) {
        // Resize bitmap to model input size
        Bitmap resizedBitmap = Bitmap.createScaledBitmap(bitmap, inputSize, inputSize, true);
        
        // Convert to tensor with normalization
        Tensor inputTensor = TensorImageUtils.bitmapToFloat32Tensor(
            resizedBitmap,
            MEAN,
            STD
        );
        
        // Run inference
        Tensor outputTensor = model.forward(IValue.from(inputTensor)).toTensor();
        
        // Get output array
        float[] outputs = outputTensor.getDataAsFloatArray();
        long[] shape = outputTensor.shape();
        
        // Parse output shape
        int batchSize = (int) shape[0];
        int numPredictions = (int) shape[1];
        int numValues = (int) shape[2]; // 5 + num_classes (x, y, w, h, conf, class_scores...)
        
        Log.d(TAG, "Output shape: [" + batchSize + ", " + numPredictions + ", " + numValues + "]");
        
        // Process predictions
        List detections = processOutputs(outputs, numPredictions, numValues, 
                                                    bitmap.getWidth(), bitmap.getHeight());
        
        // Apply Non-Maximum Suppression
        return nonMaxSuppression(detections);
    }

    /**
     * Process model outputs to extract detections
     */
    private List processOutputs(float[] outputs, int numPredictions, 
                                          int numValues, int originalWidth, int originalHeight) {
        List detections = new ArrayList<>();
        
        for (int i = 0; i < numPredictions; i++) {
            int offset = i * numValues;
            
            // YOLO output format: [x_center, y_center, width, height, objectness, class_scores...]
            float xCenter = outputs[offset];
            float yCenter = outputs[offset + 1];
            float width = outputs[offset + 2];
            float height = outputs[offset + 3];
            float objectness = outputs[offset + 4];
            
            // Skip low confidence predictions
            if (objectness < confidenceThreshold) {
                continue;
            }
            
            // Find class with highest score
            int classIndex = 0;
            float maxClassScore = outputs[offset + 5];
            
            for (int j = 1; j < labels.length && (offset + 5 + j) < outputs.length; j++) {
                float score = outputs[offset + 5 + j];
                if (score > maxClassScore) {
                    maxClassScore = score;
                    classIndex = j;
                }
            }
            
            // Calculate final confidence
            float confidence = objectness * maxClassScore;
            
            if (confidence < confidenceThreshold) {
                continue;
            }
            
            // Convert from normalized coordinates to pixel coordinates
            float scaleX = (float) originalWidth / inputSize;
            float scaleY = (float) originalHeight / inputSize;
            
            float pixelXCenter = xCenter * scaleX;
            float pixelYCenter = yCenter * scaleY;
            float pixelWidth = width * scaleX;
            float pixelHeight = height * scaleY;
            
            float left = pixelXCenter - pixelWidth / 2;
            float top = pixelYCenter - pixelHeight / 2;
            float right = pixelXCenter + pixelWidth / 2;
            float bottom = pixelYCenter + pixelHeight / 2;
            
            // Clamp to image boundaries
            left = Math.max(0, Math.min(left, originalWidth));
            top = Math.max(0, Math.min(top, originalHeight));
            right = Math.max(0, Math.min(right, originalWidth));
            bottom = Math.max(0, Math.min(bottom, originalHeight));
            
            RectF box = new RectF(left, top, right, bottom);
            String label = classIndex < labels.length ? labels[classIndex] : "Unknown";
            
            detections.add(new Detection(box, label, confidence));
        }
        
        Log.d(TAG, "Found " + detections.size() + " detections before NMS");
        return detections;
    }

    /**
     * Apply Non-Maximum Suppression to remove overlapping boxes
     */
    private List nonMaxSuppression(List detections) {
        if (detections.isEmpty()) {
            return detections;
        }
        
        List result = new ArrayList<>();
        
        // Sort by confidence (descending)
        detections.sort((a, b) -> Float.compare(b.confidence, a.confidence));
        
        while (!detections.isEmpty()) {
            // Take the detection with highest confidence
            Detection best = detections.remove(0);
            result.add(best);
            
            // Remove detections that overlap significantly with this one
            detections.removeIf(detection -> 
                calculateIoU(best.box, detection.box) > iouThreshold
            );
        }
        
        Log.d(TAG, "After NMS: " + result.size() + " detections");
        return result;
    }

    /**
     * Calculate Intersection over Union between two boxes
     */
    private float calculateIoU(RectF box1, RectF box2) {
        float intersectLeft = Math.max(box1.left, box2.left);
        float intersectTop = Math.max(box1.top, box2.top);
        float intersectRight = Math.min(box1.right, box2.right);
        float intersectBottom = Math.min(box1.bottom, box2.bottom);
        
        float intersectWidth = Math.max(0, intersectRight - intersectLeft);
        float intersectHeight = Math.max(0, intersectBottom - intersectTop);
        float intersectArea = intersectWidth * intersectHeight;
        
        float box1Area = (box1.right - box1.left) * (box1.bottom - box1.top);
        float box2Area = (box2.right - box2.left) * (box2.bottom - box2.top);
        float unionArea = box1Area + box2Area - intersectArea;
        
        return intersectArea / unionArea;
    }

    public void setConfidenceThreshold(float threshold) {
        this.confidenceThreshold = threshold;
        Log.d(TAG, "Confidence threshold set to: " + threshold);
    }

    public void setIouThreshold(float threshold) {
        this.iouThreshold = threshold;
        Log.d(TAG, "IoU threshold set to: " + threshold);
    }

    public void close() {
        // Clean up resources if needed
        Log.d(TAG, "Detector closed");
    }
}
```

---

## File 7: OverlayView.java

**File Path:** `app/src/main/java/com/yourname/bisindodetector/OverlayView.java`

```java
package com.example.bisindodetector;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.RectF;
import android.util.AttributeSet;
import android.view.View;

import java.util.ArrayList;
import java.util.List;

public class OverlayView extends View {
    // UBAH BARIS INI
    private List<YoloDetector.Detection> detections = new ArrayList<>();
    private Paint boxPaint;
    private Paint textPaint;
    private Paint backgroundPaint;
    private Paint fillPaint;

    private boolean flashEffect = false;
    private long flashStartTime = 0;
    private static final long FLASH_DURATION = 300; // ms

    public OverlayView(Context context, AttributeSet attrs) {
        super(context, attrs);
        initPaints();
    }

    private void initPaints() {
        // Bounding box paint
        boxPaint = new Paint();
        boxPaint.setColor(Color.GREEN);
        boxPaint.setStyle(Paint.Style.STROKE);
        boxPaint.setStrokeWidth(6f);
        boxPaint.setAntiAlias(true);

        // Text paint
        textPaint = new Paint();
        textPaint.setColor(Color.WHITE);
        textPaint.setTextSize(48f);
        textPaint.setStyle(Paint.Style.FILL);
        textPaint.setAntiAlias(true);
        textPaint.setFakeBoldText(true);

        // Background for text
        backgroundPaint = new Paint();
        backgroundPaint.setColor(Color.GREEN);
        backgroundPaint.setStyle(Paint.Style.FILL);
        backgroundPaint.setAntiAlias(true);

        // Semi-transparent fill for boxes
        fillPaint = new Paint();
        fillPaint.setColor(Color.argb(30, 0, 255, 0)); // Transparent green
        fillPaint.setStyle(Paint.Style.FILL);
        fillPaint.setAntiAlias(true);
    }

    // UBAH BARIS INI JUGA
    public void setDetections(List<YoloDetector.Detection> detections) {
        this.detections = detections;
        invalidate(); // Redraw
    }

    public void flashGreen() {
        flashEffect = true;
        flashStartTime = System.currentTimeMillis();
        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        // Draw flash effect if active
        if (flashEffect) {
            long elapsed = System.currentTimeMillis() - flashStartTime;
            if (elapsed < FLASH_DURATION) {
                // Fade out effect
                int alpha = (int) (255 * (1 - (float) elapsed / FLASH_DURATION));
                Paint flashPaint = new Paint();
                flashPaint.setColor(Color.argb(alpha / 2, 0, 255, 0));
                canvas.drawRect(0, 0, getWidth(), getHeight(), flashPaint);
                invalidate(); // Continue animation
            } else {
                flashEffect = false;
            }
        }

        // Draw all detections
        // Baris ini sekarang akan berfungsi tanpa error
        for (YoloDetector.Detection detection : detections) {
            // Draw semi-transparent fill
            canvas.drawRect(detection.box, fillPaint);

            // Draw bounding box
            canvas.drawRect(detection.box, boxPaint);

            // Prepare label text
            String labelText = detection.label + " " +
                    String.format("%.0f%%", detection.confidence * 100);

            // Measure text dimensions
            float textWidth = textPaint.measureText(labelText);
            float textHeight = textPaint.getTextSize();

            // Draw background rectangle for text
            float padding = 10f;
            RectF textBackground = new RectF(
                    detection.box.left,
                    detection.box.top - textHeight - padding * 2,
                    detection.box.left + textWidth + padding * 2,
                    detection.box.top
            );

            // Make sure text background is within view bounds
            if (textBackground.top < 0) {
                textBackground.offset(0, -textBackground.top + detection.box.top + 5);
            }

            canvas.drawRect(textBackground, backgroundPaint);

            // Draw text
            canvas.drawText(
                    labelText,
                    textBackground.left + padding,
                    textBackground.bottom - padding,
                    textPaint
            );
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

    <!-- Bottom Panel -->
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

    BiSindo Detector
    Camera permission is required for sign language detection
    Failed to load detection model
    Camera initialization failed

```

---

## File 10: colors.xml

**File Path:** `app/src/main/res/values/colors.xml`

```xml


    #FF000000
    #FFFFFFFF
    #FF00FF00
    #FFFF0000
    #80000000
    #8000FF00

```

---

## Setup Instructions

### Step 1: Create New Project in Android Studio
1. Open Android Studio
2. Click "New Project"
3. Select "Empty Activity"
4. Set:
   - Name: `BiSindo Detector`
   - Package name: `com.yourname.bisindodetector`
   - Language: `Java`
   - Minimum SDK: `API 24: Android 7.0 (Nougat)`

### Step 2: Add Your Model File
1. In Android Studio, switch to **Project** view (top-left dropdown)
2. Navigate to `app/src/main/`
3. Right-click on `main` → New → Directory → Name it `assets`
4. Copy your `bisindo_model_416.torchscript` file into the `assets` folder

### Step 3: Update Package Name
Replace all occurrences of `com.yourname.bisindodetector` with your actual package name throughout all files.

### Step 4: Update Class Labels
In `YoloDetector.java`, update the `labels` array to match your trained model's classes:

```java
private String[] labels = {
    // Replace with your actual BiSindo letter classes in the correct order
    "A", "B", "C", "D", "E", "F", "G", "H", "I", "J", 
    "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", 
    "U", "V", "W", "X", "Y", "Z"
};
```

### Step 5: Sync Gradle
1. Click "Sync Now" when prompted
2. Wait for Gradle to download dependencies

### Step 6: Build and Run
1. Connect your Android device or start an emulator
2. Click the green "Run" button
3. Select your device
4. Grant camera permission when prompted

---

## Troubleshooting

### Model Not Found
**Error:** `FileNotFoundException: bisindo_model_416.torchscript`
**Solution:** Make sure the model file is in `app/src/main/assets/`

### Camera Permission Denied
**Solution:** Go to Settings → Apps → BiSindo Detector → Permissions → Enable Camera

### Low FPS / Lag
**Solutions:**
- Use a physical device instead of emulator
- Reduce model size to 320: Change `416` to `320` in MainActivity
- Lower confidence threshold: `detector.setConfidenceThreshold(0.3f);`

### Wrong Detections
**Solutions:**
- Check if labels array matches your model's training order
- Adjust confidence threshold: Try 0.3 - 0.6
- Ensure good lighting conditions
- Hold hand steady and clear in frame

### App Crashes on Start
**Solutions:**
- Check Logcat for error messages
- Verify model file is not corrupted
- Ensure minimum SDK is 24 or higher
- Check if PyTorch dependencies are correctly added

---

## Features

✅ Real-time BiSindo sign language detection  
✅ Front camera with mirrored view  
✅ FPS counter for performance monitoring  
✅ Hold letter for 1.5 seconds to add to word  
✅ Clear button to reset detected text  
✅ Visual feedback (green flash) when letter is added  
✅ Bounding boxes with confidence scores  
✅ Optimized for 416x416 input size  

---

## Customization Options

### Change Hold Duration
In `MainActivity.java`:
```java
private long letterHoldTime = 1500; // Change to 1000 for 1 second, 2000 for 2 seconds
```

### Change Confidence Threshold
In `MainActivity.onCreate()`:
```java
detector.setConfidenceThreshold(0.5f); // Try 0.3 to 0.7
```

### Add Space Button
Add this to `activity_main.xml` in the bottom panel:
```xml

```

And in `MainActivity.onCreate()`:
```java
Button spaceButton = findViewById(R.id.spaceButton);
spaceButton.setOnClickListener(v -> {
    detectedWord.append(" ");
    detectedTextView.setText("Detected: " + detectedWord.toString());
});
```

---

## Next Steps

1. Test with different lighting conditions
2. Test from various distances
3. Fine-tune confidence and IoU thresholds
4. Add backspace functionality
5. Add text-to-speech for detected words
6. Save detected words to a file

Good luck with your BiSindo detector app! 🚀
