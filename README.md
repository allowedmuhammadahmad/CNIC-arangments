# CNIC A4 Print Maker

Android app (Kotlin + Jetpack Compose + OpenCV) that turns a photo of an ID card's
front and back into a two-page, print-ready A4 PDF: 8 copies of the front on page 1,
8 copies of the back on page 2, laid out in a 2×4 grid with equal spacing.

Everything runs on-device. There is no `INTERNET` permission, no analytics, and no
network calls anywhere in the code.

## How to build

This project needs the Android SDK and Gradle, which this environment doesn't have
(no Android SDK, no network access to download Gradle/dependencies), so the APK
couldn't be compiled here. To build it yourself:

1. Install **Android Studio** (Koala/2024.1 or newer).
2. `File → Open` this folder.
3. Let Gradle sync (it will download the Kotlin/AGP/Compose/CameraX/OpenCV
   dependencies — the OpenCV Android artifact comes straight from Maven Central,
   no separate OpenCV SDK download needed).
4. `Build → Generate Signed App Bundle / APK → APK`, or just run
   `./gradlew assembleRelease` from a terminal. The debug signing config is wired
   up so `assembleRelease` produces an installable APK out of the box — swap in
   your own keystore before distributing it beyond your own devices.
5. Install the APK from `app/build/outputs/apk/release/app-release.apk`.

Minimum SDK 26 (Android 8.0), target/compile SDK 34.

## What's implemented

- **Capture**: CameraX preview with a card-shaped guide outline, flash toggle,
  manual shutter, plus "pick from gallery" for front and back.
- **Auto detection**: classical OpenCV pipeline (Canny edges → contours →
  polygon approximation → aspect-ratio/area filtering) finds the card's 4
  corners without any ML model or network call. Falls back to manual crop with
  the exact message from the spec when it isn't confident.
- **Manual crop**: draggable 4-corner overlay on the photo; "Apply" runs
  perspective correction (`getPerspectiveTransform` + `warpPerspective`) to
  produce an upright, cropped card image.
- **Enhancement**: brightness/contrast (linear transform), sharpness (unsharp
  mask), and mild denoise (bilateral filter) — all *global* filters, so no
  pixel of the name, ID number, DOB, photo, signature, or QR code is ever
  redrawn or fabricated. Auto Enhance picks values from the image's own
  brightness/contrast statistics; Reset zeroes them; manual sliders are always
  available.
- **A4 layout math** (`A4LayoutCalculator`): computes equal spacing *per axis*
  — left margin = right margin = every horizontal gap, and separately top
  margin = bottom margin = every vertical gap — and centers the whole 2×4
  grid. See the note below on why it's per-axis rather than one single value.
- **A4 live preview**: front/back tabs, pinch-zoom/pan, live margin/gap
  readout, "Auto Equal Spacing" button.
- **PDF**: exactly two pages via `android.graphics.pdf.PdfDocument`, front-only
  page 1 / back-only page 2, cards placed at their true physical size in PDF
  points so "Actual Size / 100%" printing is accurate; card artwork rasterized
  at the chosen DPI (150/300/600, default 300).
- **Print / Share / Save**: share sheet (PDF to any app) plus a direct call
  into Android's `PrintManager` with A4/portrait attributes.
- **Projects**: save/open/delete, stored as JSON + JPEGs under the app's
  private storage only (`context.filesDir`), never shared storage.
- **Delete Project / temp cleanup**: wipes the project's saved files and all
  cache-held working images.
- **Error handling**: the exact messages requested for missing front/back,
  failed detection, and blurry photos (variance-of-Laplacian blur check).

## One honest gap

"Auto capture when card is stable" in the camera screen is not wired up — the
capture button and card guide work, and auto-crop still runs automatically on
whatever photo you take, but true frame-by-frame stability detection would
need a CameraX `ImageAnalysis` analyzer running the corner detector on every
preview frame with throttling. The hook point is documented in
`camera/CameraCaptureScreen.kt`; it's a bounded addition on top of the existing
`CardDetector`, not a redesign.

## About "all spacing must be mathematically equal"

With real ID-1 card size (85.6 × 54 mm) in a 2×4 grid on A4 (210 × 297 mm),
the same spacing value can't apply to *both* axes without shrinking the cards
well below true size — the horizontal margins/gap come out to ~12.9 mm and the
vertical ones to ~16.2 mm at full size. `A4LayoutCalculator` keeps the card at
true size and makes spacing equal *within* each axis (all horizontal values
equal to each other, all vertical values equal to each other), which is what
"Auto Equal Spacing" computes. If you'd rather have one single spacing value
across both axes at the cost of shrinking the printed card, that's a one-line
change in `A4LayoutCalculator.compute()` — say the word and I'll wire in a
toggle for it.
