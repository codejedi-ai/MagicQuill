# Mobile Canvas Integration Guide for MagicQuill

This document provides a comprehensive blueprint for building a mobile canvas interface (Kotlin/Android and iOS/Swift) that connects to the MagicQuill backend. The goal is to replicate the powerful drawing and editing capabilities of the Gradio web interface in native mobile apps.

---

## Table of Contents

1. [Understanding the Brush System](#understanding-the-brush-system)
2. [Canvas Architecture Overview](#canvas-architecture-overview)
3. [Layer Management](#layer-management)
4. [Mobile Implementation Details](#mobile-implementation-details)
5. [Data Flow & API Integration](#data-flow--api-integration)
6. [UX Considerations](#ux-considerations)
7. [Performance Optimization](#performance-optimization)
8. [Testing Strategy](#testing-strategy)

---

## Understanding the Brush System

MagicQuill uses **4 distinct brushes**, each writing to a **separate image layer**. This separation is critical—the backend processes each layer independently to control different aspects of image generation.

### The 4 Brushes

| Brush | Layer Name | Purpose | Backend Usage |
|-------|------------|---------|---------------|
| **Add Edge** | `add_edge_image` | Draw outlines/edges to add | Creates `add_edge_mask` - tells model where to add new edges |
| **Remove Edge** | `remove_edge_image` | Draw outlines/edges to remove | Creates `remove_edge_mask` - tells model where to erase edges |
| **RGB/Color** | `add_color_image` | Apply color tints/overlays | Full RGB image - provides color hints for the model |
| **Eraser** | `total_mask` | Define inpainting region | Creates `total_mask` - defines the area to be regenerated |

### Key Insights

1. **Each brush writes to its own transparent PNG layer** - not a single combined image
2. **The eraser (`total_mask`) defines the "work area"** - everything outside this mask is preserved
3. **Edge brushes use alpha channels** - the backend extracts the alpha to create binary masks
4. **Color brush is a full RGB image** - it can overlay colors on top of the original image
5. **All layers must match the original image dimensions** exactly

---

## Canvas Architecture Overview

### Multi-Layer Canvas System

```
┌─────────────────────────────────────┐
│      Mobile Canvas View              │
├─────────────────────────────────────┤
│  Layer 4: Total Mask (Eraser)       │  ← Transparent PNG, alpha = erase area
│  Layer 3: Remove Edge Mask          │  ← Transparent PNG, alpha = remove edges
│  Layer 2: Add Edge Mask             │  ← Transparent PNG, alpha = add edges
│  Layer 1: Color Overlay             │  ← RGB image with color tints
│  Layer 0: Original Image (Base)     │  ← User's uploaded photo
└─────────────────────────────────────┘
```

### State Management

Each layer is stored as:
- **In-memory**: Native bitmap/canvas object for real-time drawing
- **Serialized**: Base64-encoded PNG string for API transmission
- **Metadata**: Dimensions, brush size, color, opacity

---

## Layer Management

### Layer Data Structure

```kotlin
// Kotlin/Android Example
data class CanvasLayer(
    val type: LayerType,
    val bitmap: Bitmap,           // For drawing operations
    val base64Png: String? = null // For API transmission
)

enum class LayerType {
    ORIGINAL_IMAGE,    // Base photo
    ADD_COLOR,         // RGB color brush
    ADD_EDGE,          // Edge add brush
    REMOVE_EDGE,       // Edge remove brush
    TOTAL_MASK         // Eraser mask
}
```

```swift
// iOS/Swift Example
enum LayerType {
    case originalImage
    case addColor
    case addEdge
    case removeEdge
    case totalMask
}

struct CanvasLayer {
    let type: LayerType
    var cgImage: CGImage          // For drawing operations
    var base64Png: String? = nil       // For API transmission
}
```

### Layer Rendering Strategy

**Option 1: Composite on-the-fly (Recommended for mobile)**
- Store each layer as a separate bitmap/canvas
- Composite layers when rendering to screen
- Only serialize to PNG when sending to API

**Option 2: Single composite canvas**
- Draw all layers onto one canvas for display
- Maintain separate layers in memory for updates
- Re-composite when any layer changes

**Recommendation**: Use Option 1 for better performance and easier undo/redo.

---

## Mobile Implementation Details

### Android (Kotlin) Implementation

#### 1. Canvas Setup

```kotlin
class MagicQuillCanvasView(context: Context) : View(context) {
    private val layers = mutableMapOf<LayerType, CanvasLayer>()
    private var currentBrush: LayerType = LayerType.ADD_EDGE
    private var brushSize: Float = 20f
    private var brushColor: Int = Color.BLACK
    
    private val paint = Paint().apply {
        isAntiAlias = true
        style = Paint.Style.STROKE
        strokeCap = Paint.Cap.ROUND
        strokeJoin = Paint.Join.ROUND
    }
    
    init {
        // Initialize empty layers matching original image dimensions
        initializeLayers()
    }
    
    private fun initializeLayers(imageWidth: Int, imageHeight: Int) {
        layers[LayerType.ADD_EDGE] = createTransparentLayer(imageWidth, imageHeight)
        layers[LayerType.REMOVE_EDGE] = createTransparentLayer(imageWidth, imageHeight)
        layers[LayerType.ADD_COLOR] = createTransparentLayer(imageWidth, imageHeight)
        layers[LayerType.TOTAL_MASK] = createTransparentLayer(imageWidth, imageHeight)
    }
    
    private fun createTransparentLayer(width: Int, height: Int): CanvasLayer {
        val bitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888)
        val canvas = Canvas(bitmap)
        canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR)
        return CanvasLayer(currentBrush, bitmap)
    }
}
```

#### 2. Touch Handling

```kotlin
private var path = Path()
private var lastX = 0f
private var lastY = 0f

override fun onTouchEvent(event: MotionEvent): Boolean {
    val x = event.x
    val y = event.y
    
    when (event.action) {
        MotionEvent.ACTION_DOWN -> {
            path.moveTo(x, y)
            lastX = x
            lastY = y
            return true
        }
        MotionEvent.ACTION_MOVE -> {
            path.quadTo(lastX, lastY, (x + lastX) / 2, (y + lastY) / 2)
            lastX = x
            lastY = y
            drawStroke()
        }
        MotionEvent.ACTION_UP -> {
            path.lineTo(x, y)
            drawStroke()
            path.reset()
        }
    }
    return super.onTouchEvent(event)
}

private fun drawStroke() {
    val layer = layers[currentBrush] ?: return
    val canvas = Canvas(layer.bitmap)
    
    paint.strokeWidth = brushSize
    paint.color = when (currentBrush) {
        LayerType.ADD_COLOR -> brushColor
        LayerType.ADD_EDGE, LayerType.REMOVE_EDGE -> Color.BLACK
        LayerType.TOTAL_MASK -> Color.WHITE  // White = erase area
        else -> Color.TRANSPARENT
    }
    
    // For masks, use alpha channel
    if (currentBrush == LayerType.ADD_EDGE || 
        currentBrush == LayerType.REMOVE_EDGE ||
        currentBrush == LayerType.TOTAL_MASK) {
        paint.alpha = 255  // Fully opaque for mask areas
    }
    
    canvas.drawPath(path, paint)
    invalidate()  // Redraw composite view
}
```

#### 3. Layer Serialization to Base64 PNG

```kotlin
fun serializeLayer(layerType: LayerType): String? {
    val layer = layers[layerType] ?: return null
    val bitmap = layer.bitmap
    
    val outputStream = ByteArrayOutputStream()
    bitmap.compress(Bitmap.CompressFormat.PNG, 100, outputStream)
    val byteArray = outputStream.toByteArray()
    val base64 = Base64.encodeToString(byteArray, Base64.NO_WRAP)
    
    return "data:image/png;base64,$base64"
}

fun serializeAllLayers(): Map<String, String?> {
    return mapOf(
        "original_image" to serializeLayer(LayerType.ORIGINAL_IMAGE),
        "add_color_image" to serializeLayer(LayerType.ADD_COLOR),
        "add_edge_image" to serializeLayer(LayerType.ADD_EDGE),
        "remove_edge_image" to serializeLayer(LayerType.REMOVE_EDGE),
        "total_mask" to serializeLayer(LayerType.TOTAL_MASK)
    )
}
```

#### 4. Composite Rendering

```kotlin
override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
    
    // Draw base image
    layers[LayerType.ORIGINAL_IMAGE]?.bitmap?.let {
        canvas.drawBitmap(it, 0f, 0f, null)
    }
    
    // Draw color overlay (blend mode)
    layers[LayerType.ADD_COLOR]?.bitmap?.let {
        paint.xfermode = PorterDuffXfermode(PorterDuff.Mode.MULTIPLY)
        canvas.drawBitmap(it, 0f, 0f, paint)
        paint.xfermode = null
    }
    
    // Draw edge masks (for preview only - backend uses alpha)
    layers[LayerType.ADD_EDGE]?.bitmap?.let {
        paint.alpha = 128  // Semi-transparent preview
        canvas.drawBitmap(it, 0f, 0f, paint)
        paint.alpha = 255
    }
    
    // Draw erase mask preview
    layers[LayerType.TOTAL_MASK]?.bitmap?.let {
        paint.colorFilter = ColorMatrixColorFilter(
            ColorMatrix(floatArrayOf(
                0f, 0f, 0f, 0f, 0f,  // R
                0f, 0f, 0f, 0f, 0f,  // G
                0f, 0f, 0f, 0f, 0f,  // B
                0.3f, 0.3f, 0.3f, 0f, 255f  // A (red tint for preview)
            ))
        )
        canvas.drawBitmap(it, 0f, 0f, paint)
        paint.colorFilter = null
    }
}
```

### iOS (Swift) Implementation

#### 1. Canvas Setup

```swift
import UIKit
import CoreGraphics

class MagicQuillCanvasView: UIView {
    private var layers: [LayerType: CanvasLayer] = [:]
    private var currentBrush: LayerType = .addEdge
    private var brushSize: CGFloat = 20.0
    private var brushColor: UIColor = .black
    
    private var currentPath: UIBezierPath?
    private var lastPoint: CGPoint = .zero
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        setupLayers()
    }
    
    required init?(coder: NSCoder) {
        super.init(coder: coder)
        setupLayers()
    }
    
    private func setupLayers() {
        let size = bounds.size
        layers[.addEdge] = createTransparentLayer(size: size)
        layers[.removeEdge] = createTransparentLayer(size: size)
        layers[.addColor] = createTransparentLayer(size: size)
        layers[.totalMask] = createTransparentLayer(size: size)
    }
    
    private func createTransparentLayer(size: CGSize) -> CanvasLayer {
        let format = UIGraphicsImageRendererFormat()
        format.opaque = false
        let renderer = UIGraphicsImageRenderer(size: size, format: format)
        
        let image = renderer.image { context in
            // Transparent background
        }
        
        return CanvasLayer(type: currentBrush, cgImage: image.cgImage!)
    }
}
```

#### 2. Touch Handling

```swift
override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
    guard let touch = touches.first else { return }
    let point = touch.location(in: self)
    
    currentPath = UIBezierPath()
    currentPath?.move(to: point)
    lastPoint = point
}

override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
    guard let touch = touches.first,
          let path = currentPath else { return }
    let point = touch.location(in: self)
    
    path.addQuadCurve(to: point, controlPoint: lastPoint)
    lastPoint = point
    drawStroke()
}

override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
    guard let path = currentPath else { return }
    path.addLine(to: lastPoint)
    drawStroke()
    currentPath = nil
}

private func drawStroke() {
    guard let layer = layers[currentBrush],
          let path = currentPath else { return }
    
    let format = UIGraphicsImageRendererFormat()
    format.opaque = false
    let renderer = UIGraphicsImageRenderer(
        size: CGSize(width: layer.cgImage.width, 
                     height: layer.cgImage.height),
        format: format
    )
    
    let newImage = renderer.image { context in
        // Draw existing layer
        if let cgImage = layer.cgImage {
            context.cgContext.draw(cgImage, in: 
                CGRect(origin: .zero, 
                       size: CGSize(width: cgImage.width, 
                                   height: cgImage.height)))
        }
        
        // Draw new stroke
        context.cgContext.setStrokeColor(getBrushColor().cgColor)
        context.cgContext.setLineWidth(brushSize)
        context.cgContext.setLineCap(.round)
        context.cgContext.setLineJoin(.round)
        context.cgContext.addPath(path.cgPath)
        context.cgContext.strokePath()
    }
    
    layers[currentBrush] = CanvasLayer(
        type: currentBrush, 
        cgImage: newImage.cgImage!
    )
    
    setNeedsDisplay()
}

private func getBrushColor() -> UIColor {
    switch currentBrush {
    case .addColor:
        return brushColor
    case .addEdge, .removeEdge:
        return .black
    case .totalMask:
        return .white  // White = erase area
    default:
        return .clear
    }
}
```

#### 3. Layer Serialization to Base64 PNG

```swift
func serializeLayer(_ layerType: LayerType) -> String? {
    guard let layer = layers[layerType],
          let cgImage = layer.cgImage else { return nil }
    
    let uiImage = UIImage(cgImage: cgImage)
    guard let pngData = uiImage.pngData() else { return nil }
    
    let base64 = pngData.base64EncodedString()
    return "data:image/png;base64,\(base64)"
}

func serializeAllLayers() -> [String: String?] {
    return [
        "original_image": serializeLayer(.originalImage),
        "add_color_image": serializeLayer(.addColor),
        "add_edge_image": serializeLayer(.addEdge),
        "remove_edge_image": serializeLayer(.removeEdge),
        "total_mask": serializeLayer(.totalMask)
    ]
}
```

#### 4. Composite Rendering

```swift
override func draw(_ rect: CGRect) {
    guard let context = UIGraphicsGetCurrentContext() else { return }
    
    // Draw base image
    if let originalLayer = layers[.originalImage],
       let cgImage = originalLayer.cgImage {
        context.draw(cgImage, in: bounds)
    }
    
    // Draw color overlay
    if let colorLayer = layers[.addColor],
       let cgImage = colorLayer.cgImage {
        context.setBlendMode(.multiply)
        context.draw(cgImage, in: bounds)
        context.setBlendMode(.normal)
    }
    
    // Draw edge previews (semi-transparent)
    if let edgeLayer = layers[.addEdge],
       let cgImage = edgeLayer.cgImage {
        context.setAlpha(0.5)
        context.draw(cgImage, in: bounds)
        context.setAlpha(1.0)
    }
    
    // Draw erase mask preview (red tint)
    if let maskLayer = layers[.totalMask],
       let cgImage = maskLayer.cgImage {
        context.setFillColor(UIColor.red.withAlphaComponent(0.3).cgColor)
        context.setBlendMode(.sourceAtop)
        context.draw(cgImage, in: bounds)
        context.setBlendMode(.normal)
    }
}
```

---

## Data Flow & API Integration

### Complete Workflow

```
┌─────────────────┐
│  User Uploads   │
│  Base Image     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Initialize     │
│  5 Canvas       │
│  Layers         │
└────────┬────────┘
         │
         ▼
┌─────────────────┐      ┌──────────────────┐
│  User Draws     │─────▶│  Update Layer     │
│  with Brushes   │      │  Bitmap in Memory │
└────────┬────────┘      └──────────────────┘
         │
         ▼
┌─────────────────┐
│  User Taps      │
│  "Generate"     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Serialize All  │
│  Layers to      │
│  Base64 PNGs    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐      ┌──────────────────┐
│  POST to        │─────▶│  FastAPI Backend  │
│  /magic_quill/  │      │  Processes &      │
│  generate       │      │  Generates Image  │
└────────┬────────┘      └────────┬─────────┘
         │                        │
         │                        ▼
         │              ┌──────────────────┐
         │              │  Returns Base64  │
         │              │  Generated Image  │
         │              └────────┬─────────┘
         │                       │
         └───────────────────────┘
                    │
                    ▼
         ┌──────────────────┐
         │  Display Result  │
         │  in Mobile App   │
         └──────────────────┘
```

### API Request Construction

#### Android (Kotlin) Example

```kotlin
class MagicQuillApiClient(private val baseUrl: String) {
    private val client = OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(120, TimeUnit.SECONDS)  // Long timeout for generation
        .build()
    
    suspend fun generateImage(
        canvasView: MagicQuillCanvasView,
        prompt: String,
        params: GenerationParams
    ): Result<String> = withContext(Dispatchers.IO) {
        try {
            val layers = canvasView.serializeAllLayers()
            
            val requestBody = JSONObject().apply {
                put("from_frontend", JSONObject().apply {
                    put("original_image", layers["original_image"])
                    put("add_color_image", layers["add_color_image"])
                    put("add_edge_image", layers["add_edge_image"])
                    put("remove_edge_image", layers["remove_edge_image"])
                    put("total_mask", layers["total_mask"])
                })
                put("from_backend", JSONObject().apply {
                    put("prompt", prompt)
                })
                put("params", JSONObject().apply {
                    put("ckpt_name", params.ckptName)
                    put("negative_prompt", params.negativePrompt)
                    put("fine_edge", params.fineEdge)
                    put("grow_size", params.growSize)
                    put("edge_strength", params.edgeStrength)
                    put("color_strength", params.colorStrength)
                    put("inpaint_strength", params.inpaintStrength)
                    put("seed", params.seed)
                    put("steps", params.steps)
                    put("cfg", params.cfg)
                    put("sampler_name", params.samplerName)
                    put("scheduler", params.scheduler)
                })
            }
            
            val request = Request.Builder()
                .url("$baseUrl/magic_quill/generate")
                .post(requestBody.toString().toRequestBody("application/json".toMediaType()))
                .build()
            
            val response = client.newCall(request).execute()
            if (response.isSuccessful) {
                val jsonResponse = JSONObject(response.body?.string() ?: "")
                Result.success(jsonResponse.getString("generated_image"))
            } else {
                Result.failure(Exception("API Error: ${response.code}"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    suspend fun guessPrompt(
        originalImage: String,
        addColorImage: String?,
        addEdgeImage: String?
    ): Result<String> = withContext(Dispatchers.IO) {
        // Implementation similar to above
    }
}
```

#### iOS (Swift) Example

```swift
class MagicQuillApiClient {
    private let baseURL: String
    private let session: URLSession
    
    init(baseURL: String) {
        self.baseURL = baseURL
        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 30
        config.timeoutIntervalForResource = 120
        self.session = URLSession(configuration: config)
    }
    
    func generateImage(
        canvasView: MagicQuillCanvasView,
        prompt: String,
        params: GenerationParams
    ) async throws -> String {
        let layers = canvasView.serializeAllLayers()
        
        let requestBody: [String: Any] = [
            "from_frontend": [
                "original_image": layers["original_image"] as Any,
                "add_color_image": layers["add_color_image"] as Any,
                "add_edge_image": layers["add_edge_image"] as Any,
                "remove_edge_image": layers["remove_edge_image"] as Any,
                "total_mask": layers["total_mask"] as Any
            ],
            "from_backend": [
                "prompt": prompt
            ],
            "params": [
                "ckpt_name": params.ckptName,
                "negative_prompt": params.negativePrompt,
                "fine_edge": params.fineEdge,
                "grow_size": params.growSize,
                "edge_strength": params.edgeStrength,
                "color_strength": params.colorStrength,
                "inpaint_strength": params.inpaintStrength,
                "seed": params.seed,
                "steps": params.steps,
                "cfg": params.cfg,
                "sampler_name": params.samplerName,
                "scheduler": params.scheduler
            ]
        ]
        
        guard let url = URL(string: "\(baseURL)/magic_quill/generate") else {
            throw APIError.invalidURL
        }
        
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONSerialization.data(withJSONObject: requestBody)
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.serverError
        }
        
        let json = try JSONSerialization.jsonObject(with: data) as? [String: Any]
        guard let generatedImage = json?["generated_image"] as? String else {
            throw APIError.invalidResponse
        }
        
        return generatedImage
    }
}
```

---

## UX Considerations

### Brush Selection UI

**Recommended Layout:**
```
┌─────────────────────────────────┐
│  [Original Image Preview]       │
│                                 │
│                                 │
├─────────────────────────────────┤
│  Brush Tools:                   │
│  [➕Add Edge] [➖Remove Edge]    │
│  [🎨Color] [🧹Eraser]            │
├─────────────────────────────────┤
│  Brush Size: [━━━━━━━━●━━]      │
│  Color Picker: [🔴] [🟢] [🔵]   │
├─────────────────────────────────┤
│  [✨Guess Prompt] [🚀Generate]  │
└─────────────────────────────────┘
```

### Visual Feedback

1. **Active Brush Indicator**: Highlight the selected brush tool
2. **Layer Visibility Toggles**: Allow users to show/hide individual layers
3. **Brush Preview**: Show brush size circle at touch point
4. **Progress Indicator**: Display loading state during API calls
5. **Undo/Redo**: Maintain history stack for each layer

### Touch Gestures

- **Single Touch**: Draw with current brush
- **Pinch**: Zoom canvas
- **Pan**: Move canvas when zoomed
- **Long Press**: Show brush options menu
- **Two-Finger Tap**: Reset zoom

---

## Performance Optimization

### Memory Management

1. **Layer Bitmap Caching**: Only keep active layers in memory, serialize others
2. **Resolution Scaling**: Allow users to work at lower resolution, upscale for final generation
3. **Progressive Rendering**: Render composite view incrementally
4. **Background Serialization**: Convert to base64 on background thread

### Network Optimization

1. **Compression**: Consider compressing base64 strings before transmission
2. **Chunked Uploads**: For very large images, implement multipart upload
3. **Request Queuing**: Prevent multiple simultaneous generation requests
4. **Caching**: Cache generated images locally

### Android-Specific

```kotlin
// Use hardware acceleration
canvasView.setLayerType(View.LAYER_TYPE_HARDWARE, null)

// Offload serialization to background thread
lifecycleScope.launch(Dispatchers.Default) {
    val base64 = serializeLayer(layerType)
    withContext(Dispatchers.Main) {
        // Update UI
    }
}
```

### iOS-Specific

```swift
// Use Metal for rendering if available
if let metalDevice = MTLCreateSystemDefaultDevice() {
    // Use Metal renderer for better performance
}

// Background serialization
Task.detached(priority: .userInitiated) {
    let base64 = await serializeLayer(layerType)
    await MainActor.run {
        // Update UI
    }
}
```

---

## Testing Strategy

### Unit Tests

1. **Layer Serialization**: Verify base64 output format matches backend expectations
2. **Brush Drawing**: Test each brush type creates correct alpha channels
3. **Dimension Matching**: Ensure all layers maintain consistent dimensions
4. **API Request Format**: Validate JSON structure matches backend schema

### Integration Tests

1. **End-to-End Flow**: Upload image → draw → generate → display result
2. **Error Handling**: Test network failures, invalid responses, timeout scenarios
3. **Concurrent Operations**: Test multiple users/requests (if applicable)

### Manual Testing Checklist

- [ ] All 4 brushes draw correctly on their respective layers
- [ ] Layer visibility toggles work
- [ ] Undo/redo functions properly
- [ ] Base64 serialization produces valid PNGs
- [ ] API requests match expected format
- [ ] Generated images display correctly
- [ ] Performance is acceptable on low-end devices
- [ ] Memory usage remains stable during extended use

---

## Next Steps

1. **Implement Core Canvas**: Start with single layer, add multi-layer support
2. **Add Brush Tools**: Implement each of the 4 brushes
3. **API Integration**: Connect to existing FastAPI endpoints
4. **Polish UX**: Add visual feedback, gestures, and animations
5. **Optimize**: Profile and optimize performance bottlenecks
6. **Test**: Comprehensive testing on real devices

---

## Additional Resources

- See `MOBILE_COMMUNICATION.md` for API endpoint details
- Backend code: `gradio_run.py` lines 79-147 show how layers are processed
- Gradio component: `gradio_magicquill/templates/component/index.js` shows web implementation reference

---

**Note**: This guide assumes the backend will expose a `POST /magic_quill/generate` endpoint. Currently, this endpoint needs to be added to `gradio_run.py` (see `MOBILE_COMMUNICATION.md` Section 3 for the suggested implementation).

