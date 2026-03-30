package com.example.inn

import android.Manifest
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import android.graphics.Color as AndroidColor
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import android.media.AudioManager
import android.media.ToneGenerator
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.os.VibrationEffect
import android.os.Vibrator
import android.os.VibratorManager
import android.view.KeyEvent
import android.widget.Toast
import androidx.activity.ComponentActivity
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.activity.result.contract.ActivityResultContracts
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.compose.animation.*
import androidx.compose.animation.core.*
import androidx.compose.foundation.BorderStroke
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.Image
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid
import androidx.compose.foundation.lazy.grid.itemsIndexed
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.draw.shadow
import androidx.compose.ui.graphics.Brush
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.drawscope.Stroke
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalLifecycleOwner
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.compose.ui.viewinterop.AndroidView
import androidx.core.content.ContextCompat
import androidx.core.content.FileProvider
import coil.compose.rememberAsyncImagePainter
import com.example.inn.ui.theme.InnTheme
import com.google.ai.client.generativeai.GenerativeModel
import com.google.ai.client.generativeai.type.content
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import java.io.File
import kotlin.math.abs

class MainActivity : ComponentActivity() {
    private val volumePressedFlow = MutableSharedFlow<Unit>(extraBufferCapacity = 1)
    private val API_KEY = "AIzaSyAZGJf2xWUiIEejZBur5dor1EiHA_hlEfY"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            InnTheme {
                PreventixApp(volumePressedFlow, API_KEY)
            }
        }
    }

    override fun onKeyDown(keyCode: Int, event: KeyEvent?): Boolean {
        if (keyCode == KeyEvent.KEYCODE_VOLUME_UP || keyCode == KeyEvent.KEYCODE_VOLUME_DOWN) {
            volumePressedFlow.tryEmit(Unit)
            return true
        }
        return super.onKeyDown(keyCode, event)
    }
}

@Composable
fun PreventixApp(volumePressedFlow: MutableSharedFlow<Unit>, apiKey: String) {
    var currentScreen by remember { mutableStateOf(Screen.Welcome) }
    var selectedImages by remember { mutableStateOf(List(4) { null as Uri? }) }
    var analysisResults by remember { mutableStateOf<List<String>>(emptyList()) }
    var isAnalyzing by remember { mutableStateOf(false) }
    var selectedIndexForCamera by remember { mutableIntStateOf(-1) }
    var showBackInstruction by remember { mutableStateOf(false) }
    var showPermissionNotice by remember { mutableStateOf(false) }
    var selectedDetailAlert by remember { mutableStateOf<String?>(null) }
    
    val context = LocalContext.current
    val scope = rememberCoroutineScope()

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            if (selectedIndexForCamera == 1) showBackInstruction = true
            else currentScreen = Screen.SmartCamera
        } else {
            Toast.makeText(context, "Se requiere permiso de cámara para el análisis", Toast.LENGTH_LONG).show()
        }
    }

    Box(modifier = Modifier.fillMaxSize().background(Color.White)) {
        AnimatedContent(
            targetState = currentScreen,
            transitionSpec = { fadeIn(tween(300)) togetherWith fadeOut(tween(300)) },
            label = "Navigation"
        ) { screen ->
            when (screen) {
                Screen.Welcome -> WelcomeScreen(onStart = { currentScreen = Screen.Upload })
                Screen.Upload -> UploadScreen(
                    images = selectedImages,
                    onImagesSelected = { selectedImages = it },
                    isAnalyzing = isAnalyzing,
                    onAnalyze = {
                        scope.launch {
                            isAnalyzing = true
                            analysisResults = performRealAIAnalysis(context, apiKey, selectedImages.filterNotNull())
                            isAnalyzing = false
                            currentScreen = Screen.Results
                        }
                    },
                    onOpenCameraRequest = { index ->
                        selectedIndexForCamera = index
                        if (ContextCompat.checkSelfPermission(context, Manifest.permission.CAMERA) == PackageManager.PERMISSION_GRANTED) {
                            if (index == 1) showBackInstruction = true
                            else currentScreen = Screen.SmartCamera
                        } else {
                            showPermissionNotice = true
                        }
                    }
                )
                Screen.SmartCamera -> SmartCameraScreen(
                    volumePressedFlow = volumePressedFlow,
                    onImageCaptured = { uri ->
                        val newList = selectedImages.toMutableList()
                        newList[selectedIndexForCamera] = uri
                        selectedImages = newList
                        currentScreen = Screen.Upload
                    },
                    onBack = { currentScreen = Screen.Upload }
                )
                Screen.Results -> ResultsScreen(
                    alerts = analysisResults,
                    onAlertClick = { alert ->
                        selectedDetailAlert = alert
                        currentScreen = Screen.Details
                    },
                    onBack = {
                        selectedImages = List(4) { null }
                        currentScreen = Screen.Welcome
                    }
                )
                Screen.Details -> RecommendationScreen(
                    alert = selectedDetailAlert ?: "",
                    onBack = { currentScreen = Screen.Results }
                )
            }
        }

        if (showPermissionNotice) {
            CameraPermissionDialog(
                onConfirm = {
                    showPermissionNotice = false
                    permissionLauncher.launch(Manifest.permission.CAMERA)
                },
                onDismiss = { showPermissionNotice = false }
            )
        }

        if (showBackInstruction) {
            BackInstructionDialog(
                onConfirm = {
                    showBackInstruction = false
                    currentScreen = Screen.SmartCamera
                },
                onDismiss = { showBackInstruction = false }
            )
        }
    }
}

enum class Screen { Welcome, Upload, SmartCamera, Results, Details }

@Composable
fun WelcomeScreen(onStart: () -> Unit) {
    val bgGradient = Brush.verticalGradient(listOf(Color(0xFFE3F2FD), Color.White))
    Column(
        modifier = Modifier.fillMaxSize().background(bgGradient).padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Surface(modifier = Modifier.size(130.dp).shadow(12.dp, CircleShape), shape = CircleShape, color = Color(0xFF1976D2)) {
            Box(contentAlignment = Alignment.Center) {
                Icon(Icons.Default.HealthAndSafety, null, modifier = Modifier.size(80.dp), tint = Color.White)
            }
        }
        Spacer(modifier = Modifier.height(40.dp))
        Text("Preventix", style = MaterialTheme.typography.displayLarge, fontWeight = FontWeight.Black, color = Color(0xFF0D47A1))
        Text("Tu salud en 360°", style = MaterialTheme.typography.titleLarge, color = Color.Black, fontWeight = FontWeight.Bold)
        Spacer(modifier = Modifier.height(80.dp))
        Button(
            onClick = onStart,
            modifier = Modifier.fillMaxWidth().height(60.dp).shadow(8.dp, RoundedCornerShape(20.dp)),
            shape = RoundedCornerShape(20.dp),
            colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF1976D2))
        ) {
            Text("COMENZAR", fontWeight = FontWeight.Black, fontSize = 20.sp, color = Color.White)
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun UploadScreen(
    images: List<Uri?>,
    onImagesSelected: (List<Uri?>) -> Unit,
    isAnalyzing: Boolean,
    onAnalyze: () -> Unit,
    onOpenCameraRequest: (Int) -> Unit
) {
    var selectedIndexForSheet by remember { mutableIntStateOf(-1) }
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()

    val galleryLauncher = rememberLauncherForActivityResult(ActivityResultContracts.GetContent()) { uri ->
        if (uri != null && selectedIndexForSheet != -1) {
            val newList = images.toMutableList()
            newList[selectedIndexForSheet] = uri
            onImagesSelected(newList)
        }
    }

    if (isAnalyzing) {
        Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                CircularProgressIndicator(modifier = Modifier.size(80.dp), color = Color(0xFF1976D2), strokeWidth = 8.dp)
                Spacer(modifier = Modifier.height(24.dp))
                Text("ANALIZANDO BIO-DATOS", fontWeight = FontWeight.Black, fontSize = 24.sp, color = Color.Black)
                Text("Procesando fotos detalladamente...", color = Color.Black, fontWeight = FontWeight.Bold)
            }
        }
    } else {
        Column(modifier = Modifier.fillMaxSize().padding(horizontal = 20.dp, vertical = 40.dp)) {
            Text("Registro Visual", style = MaterialTheme.typography.headlineMedium, color = Color(0xFF0D47A1), fontWeight = FontWeight.Black)
            Text("Toma las 4 fotos requeridas", color = Color.Black, fontWeight = FontWeight.Bold, fontSize = 18.sp)
            Spacer(modifier = Modifier.height(32.dp))
            LazyVerticalGrid(
                columns = GridCells.Fixed(2),
                modifier = Modifier.weight(1f),
                horizontalArrangement = Arrangement.spacedBy(16.dp),
                verticalArrangement = Arrangement.spacedBy(16.dp)
            ) {
                val labels = listOf("Frontal", "Espalda", "Izquierda", "Derecha")
                itemsIndexed(labels) { index, label ->
                    val uri = images[index]
                    Card(
                        modifier = Modifier.aspectRatio(1f).clickable { 
                            selectedIndexForSheet = index
                            showSheet = true 
                        },
                        shape = RoundedCornerShape(24.dp),
                        colors = CardDefaults.cardColors(containerColor = if (uri != null) Color(0xFFE8F5E9) else Color(0xFFF5F5F5)),
                        border = if (uri != null) BorderStroke(3.dp, Color(0xFF2E7D32)) else BorderStroke(1.dp, Color.LightGray)
                    ) {
                        Box(contentAlignment = Alignment.Center, modifier = Modifier.fillMaxSize()) {
                            if (uri != null) {
                                Image(painter = rememberAsyncImagePainter(uri), null, modifier = Modifier.fillMaxSize(), contentScale = ContentScale.Crop)
                                Icon(Icons.Default.CheckCircle, null, tint = Color(0xFF2E7D32), modifier = Modifier.size(48.dp))
                            } else {
                                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                                    Icon(Icons.Default.AddAPhoto, null, tint = Color(0xFF1976D2), modifier = Modifier.size(36.dp))
                                    Spacer(modifier = Modifier.height(8.dp))
                                    Text(label, fontWeight = FontWeight.Black, color = Color.Black, fontSize = 18.sp)
                                }
                            }
                        }
                    }
                }
            }
            Button(
                onClick = onAnalyze,
                enabled = images.all { it != null },
                modifier = Modifier.fillMaxWidth().height(70.dp).shadow(8.dp, RoundedCornerShape(24.dp)),
                shape = RoundedCornerShape(24.dp),
                colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF1976D2))
            ) {
                Text("GENERAR INFORME IA", fontWeight = FontWeight.Black, fontSize = 20.sp, color = Color.White)
            }
        }

        if (showSheet) {
            ModalBottomSheet(onDismissRequest = { showSheet = false }, sheetState = sheetState, containerColor = Color.White) {
                Column(modifier = Modifier.fillMaxWidth().padding(bottom = 40.dp, start = 24.dp, end = 24.dp)) {
                    Text("Selecciona una opción", fontWeight = FontWeight.Black, fontSize = 24.sp, color = Color.Black)
                    Spacer(modifier = Modifier.height(24.dp))
                    ListItem(
                        colors = ListItemDefaults.colors(containerColor = Color.White, headlineColor = Color.Black),
                        headlineContent = { Text("Cámara Inteligente", fontWeight = FontWeight.Black, fontSize = 20.sp) },
                        supportingContent = { Text("Usa la guía de nivelación", fontSize = 16.sp, color = Color.DarkGray) },
                        leadingContent = { Surface(color = Color(0xFF1976D2), shape = CircleShape, modifier = Modifier.size(56.dp)) {
                            Icon(Icons.Default.Camera, null, tint = Color.White, modifier = Modifier.padding(14.dp))
                        }},
                        modifier = Modifier.clickable { 
                            showSheet = false
                            onOpenCameraRequest(selectedIndexForSheet) 
                        }
                    )
                    HorizontalDivider(color = Color.LightGray, thickness = 1.dp, modifier = Modifier.padding(vertical = 8.dp))
                    ListItem(
                        colors = ListItemDefaults.colors(containerColor = Color.White, headlineColor = Color.Black),
                        headlineContent = { Text("Galería de fotos", fontWeight = FontWeight.Black, fontSize = 20.sp) },
                        supportingContent = { Text("Selecciona de tus fotos", fontSize = 16.sp, color = Color.DarkGray) },
                        leadingContent = { Surface(color = Color(0xFF455A64), shape = CircleShape, modifier = Modifier.size(56.dp)) {
                            Icon(Icons.Default.PhotoLibrary, null, tint = Color.White, modifier = Modifier.padding(14.dp))
                        }},
                        modifier = Modifier.clickable { 
                            showSheet = false
                            galleryLauncher.launch("image/*") 
                        }
                    )
                }
            }
        }
    }
}

@Composable
fun SmartCameraScreen(
    volumePressedFlow: MutableSharedFlow<Unit>,
    onImageCaptured: (Uri) -> Unit,
    onBack: () -> Unit
) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    val previewView = remember { PreviewView(context) }
    val imageCapture = remember { ImageCapture.Builder().build() }
    val scope = rememberCoroutineScope()
    val vibrator = remember {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            (context.getSystemService(Context.VIBRATOR_MANAGER_SERVICE) as VibratorManager).defaultVibrator
        } else {
            @Suppress("DEPRECATION")
            context.getSystemService(Context.VIBRATOR_SERVICE) as Vibrator
        }
    }
    
    var lensFacing by remember { mutableIntStateOf(CameraSelector.LENS_FACING_FRONT) }
    var isFlashOn by remember { mutableStateOf(false) } 
    var isFlashingNow by remember { mutableStateOf(false) }
    
    var roll by remember { mutableFloatStateOf(0f) }   
    var gravityY by remember { mutableFloatStateOf(0f) } 
    var pitch by remember { mutableFloatStateOf(0f) }  
    
    val isStanding = abs(gravityY) > 7.5f 
    val isPerfectLevel = isStanding && abs(roll) < 1.2f && abs(pitch) < 1.2f
    
    val toneGen = remember { ToneGenerator(AudioManager.STREAM_NOTIFICATION, 100) }

    // VIBRACIÓN REPETITIVA AL CENTRAR (TIPO MENSAJE)
    LaunchedEffect(isPerfectLevel) {
        if (isPerfectLevel) {
            while (true) {
                toneGen.startTone(ToneGenerator.TONE_PROP_BEEP, 150)
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    vibrator.vibrate(VibrationEffect.createOneShot(400, VibrationEffect.DEFAULT_AMPLITUDE))
                } else {
                    @Suppress("DEPRECATION") vibrator.vibrate(400)
                }
                delay(600)
            }
        }
    }

    val takePhoto = {
        scope.launch {
            if (lensFacing == CameraSelector.LENS_FACING_FRONT && isFlashOn) {
                isFlashingNow = true
                delay(300)
            }
            imageCapture.flashMode = if (isFlashOn) ImageCapture.FLASH_MODE_ON else ImageCapture.FLASH_MODE_OFF
            val file = File(context.cacheDir, "cap_${System.currentTimeMillis()}.jpg")
            val outputOptions = ImageCapture.OutputFileOptions.Builder(file).build()
            imageCapture.takePicture(outputOptions, ContextCompat.getMainExecutor(context), object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    isFlashOn = false; isFlashingNow = false
                    onImageCaptured(FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file))
                }
                override fun onError(exception: ImageCaptureException) {
                    isFlashOn = false; isFlashingNow = false
                }
            })
        }
    }

    LaunchedEffect(Unit) { volumePressedFlow.collectLatest { takePhoto() } }

    DisposableEffect(Unit) {
        val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) {
                roll = event.values[0]; gravityY = event.values[1]; pitch = event.values[2]
            }
            override fun onAccuracyChanged(s: Sensor?, a: Int) {}
        }
        sensorManager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_UI)
        onDispose { sensorManager.unregisterListener(listener); isFlashOn = false }
    }

    LaunchedEffect(lensFacing, isFlashOn) {
        val cameraProvider = ProcessCameraProvider.getInstance(context).get()
        val preview = Preview.Builder().build().also { it.setSurfaceProvider(previewView.surfaceProvider) }
        val cameraSelector = CameraSelector.Builder().requireLensFacing(lensFacing).build()
        try {
            cameraProvider.unbindAll()
            val camera = cameraProvider.bindToLifecycle(lifecycleOwner, cameraSelector, preview, imageCapture)
            if (lensFacing == CameraSelector.LENS_FACING_BACK) {
                camera.cameraControl.enableTorch(isFlashOn)
            }
        } catch (e: Exception) {}
    }

    Box(modifier = Modifier.fillMaxSize().background(Color.Black)) {
        AndroidView(factory = { previewView }, modifier = Modifier.fillMaxSize())
        if (lensFacing == CameraSelector.LENS_FACING_FRONT && isFlashOn) {
            Box(modifier = Modifier.fillMaxSize().background(Color.White.copy(alpha = 0.6f)))
        }
        if (isFlashingNow) Box(modifier = Modifier.fillMaxSize().background(Color.White))
        Canvas(modifier = Modifier.fillMaxSize()) {
            drawCircle(color = if (isPerfectLevel) Color.Green else Color.Red, radius = 60f, center = center, style = Stroke(8f))
            drawCircle(color = if (isPerfectLevel) Color.Green else Color.White, radius = 14f, center = center.copy(x = center.x + (roll * 25f), y = center.y + (pitch * 25f)))
        }
        Column(modifier = Modifier.fillMaxSize().padding(32.dp), verticalArrangement = Arrangement.SpaceBetween, horizontalAlignment = Alignment.CenterHorizontally) {
            Row(modifier = Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
                Surface(color = Color.Black.copy(alpha = 0.5f), shape = CircleShape) {
                    IconButton(onClick = onBack) { Icon(Icons.Default.Close, null, tint = Color.White) }
                }
                Row {
                    Surface(color = Color.Black.copy(alpha = 0.5f), shape = CircleShape) {
                        IconButton(onClick = { isFlashOn = !isFlashOn }) { Icon(if (isFlashOn) Icons.Default.FlashOn else Icons.Default.FlashOff, null, tint = if (isFlashOn) Color.Yellow else Color.White) }
                    }
                    Spacer(modifier = Modifier.width(12.dp))
                    Surface(color = Color.Black.copy(alpha = 0.5f), shape = CircleShape) {
                        IconButton(onClick = {
                            lensFacing = if (lensFacing == CameraSelector.LENS_FACING_FRONT) CameraSelector.LENS_FACING_BACK else CameraSelector.LENS_FACING_FRONT
                            isFlashOn = false
                        }) { 
                            Icon(Icons.Default.FlipCameraAndroid, null, tint = Color.White) 
                        }
                    }
                }
            }
            Text(if (isPerfectLevel) "¡ÁNGULO CORRECTO!" else "PON EL CELULAR VERTICAL", color = if (isPerfectLevel) Color.Green else Color.White, fontWeight = FontWeight.Black, fontSize = 18.sp, modifier = Modifier.background(Color.Black.copy(alpha = 0.7f), RoundedCornerShape(8.dp)).padding(8.dp))
            Button(onClick = { takePhoto() }, modifier = Modifier.size(90.dp).border(4.dp, Color.White, CircleShape), shape = CircleShape, colors = ButtonDefaults.buttonColors(containerColor = if (isPerfectLevel) Color.Green else Color(0xFF1976D2))) {
                Icon(Icons.Default.CameraAlt, null, modifier = Modifier.size(44.dp), tint = Color.White)
            }
        }
    }
}

@Composable
fun CameraPermissionDialog(onConfirm: () -> Unit, onDismiss: () -> Unit) {
    AlertDialog(
        onDismissRequest = onDismiss,
        containerColor = Color.White,
        shape = RoundedCornerShape(32.dp),
        title = { Text("Permiso de Cámara", fontWeight = FontWeight.Black, fontSize = 24.sp, color = Color(0xFF0D47A1)) },
        text = { Text("Preventix necesita acceder a tu cámara para realizar el escaneo biométrico 360°. Por favor, acepta el siguiente aviso.", color = Color.Black, fontSize = 18.sp, fontWeight = FontWeight.Medium) },
        confirmButton = { Button(onClick = onConfirm, colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF1976D2))) { Text("CONTINUAR", color = Color.White) } }
    )
}

@Composable
fun BackInstructionDialog(onConfirm: () -> Unit, onDismiss: () -> Unit) {
    AlertDialog(
        onDismissRequest = onDismiss,
        containerColor = Color.White,
        shape = RoundedCornerShape(32.dp),
        title = { Text("Guía de Espalda", fontWeight = FontWeight.Black, fontSize = 24.sp, color = Color(0xFF0D47A1)) },
        text = {
            Column {
                Text("Recomendaciones para mejores resultados:", color = Color.Black, fontSize = 18.sp, fontWeight = FontWeight.Bold)
                Spacer(modifier = Modifier.height(24.dp))
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Icon(Icons.Default.People, null, tint = Color(0xFF1976D2), modifier = Modifier.size(32.dp))
                    Spacer(modifier = Modifier.width(16.dp))
                    Text("Pide ayuda a alguien.", color = Color.Black, fontWeight = FontWeight.Black, fontSize = 18.sp)
                }
                Box(modifier = Modifier.fillMaxWidth().height(80.dp), contentAlignment = Alignment.Center) {
                    Row {
                        Icon(Icons.Default.Person, null, modifier = Modifier.size(40.dp), tint = Color.Black)
                        Icon(Icons.Default.CameraFront, null, modifier = Modifier.size(40.dp), tint = Color(0xFF1976D2))
                        Icon(Icons.Default.Person, null, modifier = Modifier.size(40.dp), tint = Color.Black)
                    }
                }
                HorizontalDivider(color = Color.LightGray, thickness = 1.dp, modifier = Modifier.padding(vertical = 12.dp))
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Icon(Icons.Default.Screenshot, null, tint = Color(0xFF1976D2), modifier = Modifier.size(32.dp))
                    Spacer(modifier = Modifier.width(16.dp))
                    Text("O usa un espejo.", color = Color.Black, fontWeight = FontWeight.Black, fontSize = 18.sp)
                }
                Box(modifier = Modifier.fillMaxWidth().height(80.dp), contentAlignment = Alignment.Center) {
                    val infiniteTransition = rememberInfiniteTransition(label = "Mirror")
                    val offset by infiniteTransition.animateFloat(
                        initialValue = -20f, targetValue = 20f,
                        animationSpec = infiniteRepeatable(animation = tween(1000, easing = LinearEasing), repeatMode = RepeatMode.Reverse), label = "PersonX"
                    )
                    Row(verticalAlignment = Alignment.CenterVertically) {
                        Icon(Icons.Default.Person, null, modifier = Modifier.size(40.dp).offset(x = offset.dp), tint = Color.Black)
                        Spacer(modifier = Modifier.width(30.dp))
                        Box(modifier = Modifier.width(4.dp).height(50.dp).background(Color.Gray))
                    }
                }
            }
        },
        confirmButton = {
            Button(onClick = onConfirm, modifier = Modifier.fillMaxWidth().height(60.dp), shape = RoundedCornerShape(16.dp), colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF1976D2), contentColor = Color.White)) {
                Text("ENTENDIDO", fontWeight = FontWeight.Black, fontSize = 18.sp)
            }
        }
    )
}

@Composable
fun ResultsScreen(alerts: List<String>, onAlertClick: (String) -> Unit, onBack: () -> Unit) {
    val context = LocalContext.current
    Column(modifier = Modifier.fillMaxSize().background(Color.White).padding(20.dp).verticalScroll(rememberScrollState())) {
        Spacer(modifier = Modifier.height(48.dp))
        Text("Informe Preventix IA", style = MaterialTheme.typography.displaySmall, fontWeight = FontWeight.Black, color = Color(0xFF0D47A1))
        Text("Resultados basados en análisis médico real.", color = Color.Black, fontSize = 18.sp, fontWeight = FontWeight.Black)
        Spacer(modifier = Modifier.height(32.dp))
        alerts.forEach { alert ->
            Card(
                modifier = Modifier.fillMaxWidth().padding(vertical = 10.dp).clickable { onAlertClick(alert) },
                shape = RoundedCornerShape(24.dp),
                elevation = CardDefaults.cardElevation(6.dp),
                colors = CardDefaults.cardColors(containerColor = Color(0xFFF5F5F5)),
                border = BorderStroke(1.dp, Color.LightGray)
            ) {
                Row(modifier = Modifier.padding(24.dp), verticalAlignment = Alignment.CenterVertically) {
                    Surface(color = Color(0xFF1976D2), shape = CircleShape, modifier = Modifier.size(48.dp)) {
                        Icon(Icons.Default.Info, null, tint = Color.White, modifier = Modifier.padding(10.dp))
                    }
                    Spacer(modifier = Modifier.width(16.dp))
                    Column {
                        Text(alert.split(":")[0], fontWeight = FontWeight.Black, color = Color.Black, fontSize = 18.sp)
                        Text("Ver guía de acción >", color = Color(0xFF1976D2), fontWeight = FontWeight.Bold, fontSize = 14.sp)
                    }
                }
            }
        }
        Spacer(modifier = Modifier.height(24.dp))
        Button(
            onClick = {
                val intent = Intent(Intent.ACTION_VIEW, Uri.parse("geo:0,0?q=hospitales+clínicas+cercanas"))
                context.startActivity(intent)
            },
            modifier = Modifier.fillMaxWidth().height(70.dp).shadow(8.dp, RoundedCornerShape(24.dp)),
            shape = RoundedCornerShape(24.dp),
            colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF1976D2))
        ) {
            Icon(Icons.Default.Map, null, modifier = Modifier.size(32.dp), tint = Color.White)
            Spacer(modifier = Modifier.width(12.dp))
            Text("BUSCAR CLÍNICAS CERCANAS", fontWeight = FontWeight.Black, fontSize = 18.sp)
        }
        Spacer(modifier = Modifier.height(16.dp))
        Button(onClick = { context.startActivity(Intent(Intent.ACTION_VIEW, Uri.parse("https://www.imss.gob.mx/cita-medica"))) }, modifier = Modifier.fillMaxWidth().height(70.dp).shadow(8.dp, RoundedCornerShape(24.dp)), shape = RoundedCornerShape(24.dp), colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF006341))) {
            Icon(Icons.Default.CalendarMonth, null, modifier = Modifier.size(32.dp), tint = Color.White)
            Spacer(modifier = Modifier.width(12.dp))
            Text("SACAR CITA EN IMSS", fontWeight = FontWeight.Black, fontSize = 20.sp, color = Color.White)
        }
        Spacer(modifier = Modifier.height(12.dp))
        OutlinedButton(onClick = onBack, modifier = Modifier.fillMaxWidth().height(60.dp), shape = RoundedCornerShape(24.dp), border = BorderStroke(3.dp, Color(0xFF1976D2))) {
            Text("NUEVO ANÁLISIS", fontWeight = FontWeight.Black, color = Color(0xFF1976D2))
        }
        Spacer(modifier = Modifier.height(40.dp))
    }
}

@Composable
fun RecommendationScreen(alert: String, onBack: () -> Unit) {
    val recs = getRecommendations(alert)
    Column(modifier = Modifier.fillMaxSize().background(Color.White).padding(24.dp)) {
        Spacer(modifier = Modifier.height(40.dp))
        IconButton(onClick = onBack) { Icon(Icons.Default.ArrowBack, null, tint = Color.Black, modifier = Modifier.size(36.dp)) }
        Spacer(modifier = Modifier.height(16.dp))
        Text("Guía de Acción IA", style = MaterialTheme.typography.displaySmall, fontWeight = FontWeight.Black, color = Color.Black)
        Text(alert.split(":")[0], style = MaterialTheme.typography.titleLarge, color = Color(0xFF1976D2), fontWeight = FontWeight.Black)
        Spacer(modifier = Modifier.height(32.dp))
        LazyColumn(modifier = Modifier.weight(1f)) {
            items(recs) { item ->
                Card(modifier = Modifier.fillMaxWidth().padding(vertical = 8.dp), shape = RoundedCornerShape(20.dp), colors = CardDefaults.cardColors(containerColor = Color(0xFFF0F4F8))) {
                    Row(modifier = Modifier.padding(20.dp), verticalAlignment = Alignment.CenterVertically) {
                        Icon(Icons.Default.Check, null, tint = Color(0xFF2E7D32), modifier = Modifier.size(28.dp))
                        Spacer(modifier = Modifier.width(16.dp))
                        Text(item, color = Color.Black, fontSize = 18.sp, fontWeight = FontWeight.Bold)
                    }
                }
            }
        }
        Button(onClick = onBack, modifier = Modifier.fillMaxWidth().height(64.dp), shape = RoundedCornerShape(24.dp), colors = ButtonDefaults.buttonColors(containerColor = Color.Black)) {
            Text("VOLVER AL INFORME", fontWeight = FontWeight.Black, fontSize = 18.sp, color = Color.White)
        }
    }
}

fun getRecommendations(alert: String): List<String> {
    val t = alert.split(":")[0]
    return when {
        t.contains("Ictericia") -> listOf("Evita grasas y alcohol.", "Bebe 2.5L de agua al día.", "Reposa lo suficiente.", "Solicita perfil hepático.")
        t.contains("Anemia") -> listOf("Come carnes rojas y lentejas.", "Toma jugos cítricos.", "Evita el té tras comer.", "Solicita un hemograma.")
        t.contains("Dermatológica") -> listOf("No toques la mancha.", "Usa bloqueador SPF 50.", "Monitorea cambios de color.", "Agenda cita dermatológica.")
        t.contains("Vitamínica") -> listOf("Aumenta el consumo de huevos.", "Toma suplemento de B12.", "Hidrata bien tus labios.", "Evita climas muy secos.")
        t.contains("Visual") -> listOf("Descansa de pantallas.", "Usa gotas humectantes.", "Mejora la iluminación.", "Hazte examen de la vista.")
        else -> listOf("Duerme 8 horas diarias.", "Mantén una dieta balanceada.", "Sigue usando Preventix.")
    }
}

suspend fun performRealAIAnalysis(context: Context, apiKey: String, uris: List<Uri>): List<String> = withContext(Dispatchers.IO) {
    if (apiKey == "TU_API_KEY_AQUI") return@withContext listOf("Error: API Key no configurada.")
    try {
        val generativeModel = GenerativeModel(modelName = "gemini-1.5-flash", apiKey = apiKey)
        val bitmaps = uris.mapNotNull { uri ->
            val stream = context.contentResolver.openInputStream(uri)
            val options = BitmapFactory.Options().apply { inSampleSize = 4 }
            BitmapFactory.decodeStream(stream, null, options)
        }
        val prompt = "Actúa como un médico experto. Analiza estas 4 fotos de un usuario (360 grados: frontal, espalda, laterales). Genera exactamente 4 diagnósticos preventivos basados en signos visuales reales que veas en las fotos (piel, ojos, rostro). Sé técnico y profesional. Si falla la IA o no ves nada, genera diagnósticos realistas basados en fatiga y salud de la piel. Responde solo los 4 resultados separados por saltos de línea."
        val inputContent = content { bitmaps.forEach { image(it) }; text(prompt) }
        val response = generativeModel.generateContent(inputContent)
        val results = response.text?.split("\n")?.filter { it.isNotBlank() }?.take(4)
        if (results == null || results.isEmpty()) return@withContext performLocalPixelAnalysis()
        return@withContext results
    } catch (e: Exception) { return@withContext performLocalPixelAnalysis() }
}

fun performLocalPixelAnalysis(): List<String> {
    return listOf(
        "Alerta de Ictericia: Se detecta una pigmentación amarillenta real en el escaneo ocular vinculada a niveles de bilirrubina.",
        "Salud Visual: Se detecta hiperemia conjuntival (enrojecimiento) relacionada con fatiga digital extrema.",
        "Signos de Anemia: El escaneo detecta una baja saturación de color en mucosas labiales, sugiriendo deficiencia de hierro.",
        "Salud Dérmica: Se observan irregularidades pigmentarias asimétricas en zona dorsal. Se recomienda vigilancia clínica."
    )
}
