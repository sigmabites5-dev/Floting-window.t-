# Floating Multitask App (Bubble + Mini Window)

Game ke upar floating bubble. Bubble tap karo to mini window khulti hai: Google, YouTube, WhatsApp Web, Notes. Drag, size +/-, transparency aur close button included.

## Step 1: Project banao
1. PC pe **Android Studio** install karo (free).
2. New Project -> **Empty Views Activity** -> Language: **Kotlin**.
3. Package name: `com.sky.floatmulti` (agar alag rakha to neeche `package` line badal dena).
4. Min SDK: 26.

## Step 2: AndroidManifest.xml
Poori file isse replace karo:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE"/>
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
    <uses-permission android:name="android.permission.INTERNET"/>

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="Float Multi"
        android:usesCleartextTraffic="true"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar">

        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <service
            android:name=".FloatService"
            android:exported="false"
            android:foregroundServiceType="specialUse">
            <property
                android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                android:value="floating_overlay_tools"/>
        </service>
    </application>
</manifest>
```

Agar theme error aaye to `android:theme` ki line hata do ya apne project ki default theme rakho.

## Step 3: MainActivity.kt

```kotlin
package com.sky.floatmulti

import android.Manifest
import android.content.Intent
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.provider.Settings
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        if (Build.VERSION.SDK_INT >= 33) {
            requestPermissions(arrayOf(Manifest.permission.POST_NOTIFICATIONS), 1)
        }

        if (!Settings.canDrawOverlays(this)) {
            // "Display over other apps" permission screen
            startActivity(
                Intent(
                    Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                    Uri.parse("package:$packageName")
                )
            )
        }
    }

    override fun onResume() {
        super.onResume()
        if (Settings.canDrawOverlays(this)) {
            startForegroundService(Intent(this, FloatService::class.java))
            finish() // app band, bubble chalta rahega
        }
    }
}
```

## Step 4: FloatService.kt (main code)

```kotlin
package com.sky.floatmulti

import android.annotation.SuppressLint
import android.app.*
import android.content.Context
import android.content.Intent
import android.content.pm.ServiceInfo
import android.graphics.Color
import android.graphics.PixelFormat
import android.graphics.drawable.GradientDrawable
import android.os.Build
import android.os.IBinder
import android.view.*
import android.webkit.WebChromeClient
import android.webkit.WebView
import android.webkit.WebViewClient
import android.widget.*

class FloatService : Service() {

    private lateinit var wm: WindowManager
    private var bubble: ImageView? = null
    private var panel: LinearLayout? = null
    private var web: WebView? = null
    private var panelLp: WindowManager.LayoutParams? = null
    private var alphaStep = 0

    private val desktopUA =
        "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 " +
        "(KHTML, like Gecko) Chrome/120.0 Safari/537.36"

    override fun onBind(i: Intent?): IBinder? = null

    override fun onCreate() {
        super.onCreate()
        startAsForeground()
        wm = getSystemService(WINDOW_SERVICE) as WindowManager
        addBubble()
    }

    // ---------- Foreground notification ----------
    private fun startAsForeground() {
        val chId = "float_ch"
        val nm = getSystemService(NotificationManager::class.java)
        nm.createNotificationChannel(
            NotificationChannel(chId, "Floating tools", NotificationManager.IMPORTANCE_LOW)
        )
        val n = Notification.Builder(this, chId)
            .setContentTitle("Floating tools chal raha hai")
            .setContentText("Band karne ke liye app stop karo")
            .setSmallIcon(android.R.drawable.ic_menu_more)
            .build()
        if (Build.VERSION.SDK_INT >= 34) {
            startForeground(1, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE)
        } else {
            startForeground(1, n)
        }
    }

    private fun overlayParams(w: Int, h: Int, focusable: Boolean): WindowManager.LayoutParams {
        val flags = if (focusable)
            WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
        else
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
        return WindowManager.LayoutParams(
            w, h,
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
            flags,
            PixelFormat.TRANSLUCENT
        ).apply { gravity = Gravity.TOP or Gravity.START }
    }

    // ---------- Floating bubble ----------
    @SuppressLint("ClickableViewAccessibility")
    private fun addBubble() {
        val size = 150
        val img = ImageView(this).apply {
            setImageResource(android.R.drawable.ic_menu_more)
            setColorFilter(Color.WHITE)
            setPadding(30, 30, 30, 30)
            background = GradientDrawable().apply {
                shape = GradientDrawable.OVAL
                setColor(0xCC6200EE.toInt())
            }
        }
        val lp = overlayParams(size, size, false).apply { x = 0; y = 400 }

        var startX = 0; var startY = 0
        var touchX = 0f; var touchY = 0f
        var moved = false

        img.setOnTouchListener { _, e ->
            when (e.action) {
                MotionEvent.ACTION_DOWN -> {
                    startX = lp.x; startY = lp.y
                    touchX = e.rawX; touchY = e.rawY
                    moved = false
                }
                MotionEvent.ACTION_MOVE -> {
                    val dx = (e.rawX - touchX).toInt()
                    val dy = (e.rawY - touchY).toInt()
                    if (Math.abs(dx) > 10 || Math.abs(dy) > 10) moved = true
                    lp.x = startX + dx
                    lp.y = startY + dy
                    wm.updateViewLayout(img, lp)
                }
                MotionEvent.ACTION_UP -> if (!moved) togglePanel()
            }
            true
        }
        bubble = img
        wm.addView(img, lp)
    }

    // ---------- Mini window ----------
    private fun togglePanel() {
        if (panel != null) { hidePanel(); return }
        showPanel()
    }

    private fun hidePanel() {
        panel?.let { wm.removeView(it) }
        web?.destroy()
        panel = null; web = null
    }

    private fun showPanel() {
        val dm = resources.displayMetrics
        val w = (dm.widthPixels * 0.9).toInt()
        val h = (dm.heightPixels * 0.5).toInt()

        val wv = WebView(this).apply {
            settings.javaScriptEnabled = true
            settings.domStorageEnabled = true
            webViewClient = WebViewClient()
            webChromeClient = WebChromeClient()
            loadUrl("https://www.google.com")
        }
        web = wv

        val bar = HorizontalScrollView(this)
        val row = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL }
        bar.addView(row)

        fun btn(label: String, action: () -> Unit) {
            row.addView(Button(this).apply {
                text = label
                textSize = 11f
                isAllCaps = false
                setOnClickListener { action() }
            })
        }

        btn("Google")   { wv.settings.userAgentString = null; wv.loadUrl("https://www.google.com") }
        btn("YouTube")  { wv.settings.userAgentString = null; wv.loadUrl("https://m.youtube.com") }
        btn("WhatsApp") { wv.settings.userAgentString = desktopUA; wv.loadUrl("https://web.whatsapp.com") }
        btn("Notes") {
            wv.loadDataWithBaseURL(
                null,
                "<body contenteditable style='font-size:20px;padding:12px'>Yahan likho...</body>",
                "text/html", "utf-8", null
            )
        }
        btn("Calc")     { wv.settings.userAgentString = null; wv.loadUrl("https://www.google.com/search?q=calculator") }
        btn("Back")     { if (wv.canGoBack()) wv.goBack() }
        btn("+")        { resizePanel(1.15f) }
        btn("-")        { resizePanel(0.85f) }
        btn("Alpha")    { cycleAlpha() }
        btn("X")        { hidePanel() }

        val root = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            setBackgroundColor(Color.WHITE)
            addView(bar, LinearLayout.LayoutParams(-1, -2))
            addView(wv, LinearLayout.LayoutParams(-1, 0, 1f))
        }

        val lp = overlayParams(w, h, true).apply {
            x = (dm.widthPixels - w) / 2
            y = 150
        }
        panelLp = lp
        panel = root
        alphaStep = 0
        wm.addView(root, lp)

        // bubble ko upar rakhne ke liye dubara add
        bubble?.let {
            val bl = it.layoutParams as WindowManager.LayoutParams
            wm.removeView(it)
            wm.addView(it, bl)
        }
    }

    private fun resizePanel(factor: Float) {
        val p = panel ?: return
        val lp = panelLp ?: return
        val dm = resources.displayMetrics
        lp.width = (lp.width * factor).toInt().coerceIn(500, dm.widthPixels)
        lp.height = (lp.height * factor).toInt().coerceIn(500, dm.heightPixels)
        wm.updateViewLayout(p, lp)
    }

    private fun cycleAlpha() {
        val p = panel ?: return
        alphaStep = (alphaStep + 1) % 3
        p.alpha = when (alphaStep) { 0 -> 1f; 1 -> 0.7f; else -> 0.45f }
    }

    override fun onDestroy() {
        panel?.let { wm.removeView(it) }
        bubble?.let { wm.removeView(it) }
        web?.destroy()
        super.onDestroy()
    }
}
```

## Step 5: Phone pe chalao
1. Phone pe **Developer options -> USB debugging** on karo, cable se PC se jodo.
2. Android Studio mein green **Run** button dabao.
3. App khulega aur "Display over other apps" permission maangega. Allow karo, back dabao.
4. Bubble screen pe aa jayega. Game kholo, bubble tap karo, mini window khulegi.

POCO/HyperOS pe ye bhi karo, warna service band ho sakti hai:
- Settings -> Apps -> Manage apps -> **Float Multi** -> **Autostart** on.
- Battery saver -> **No restrictions**.
- Recent apps mein app card ko **lock** karo.
- Settings -> Apps -> Permissions -> Other permissions -> **Display pop-up windows while running in background** allow karo.

## Step 6: GitHub pe daalo (free)
1. github.com pe free account, naya repo banao.
2. Android Studio: **Git -> Share Project on GitHub**.
3. APK ka free build GitHub Actions se: repo mein `.github/workflows/build.yml` banao:

```yaml
name: Build APK
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - run: chmod +x gradlew && ./gradlew assembleDebug
      - uses: actions/upload-artifact@v4
        with:
          name: app-debug
          path: app/build/outputs/apk/debug/app-debug.apk
```

Push karne ke baad repo ke **Actions** tab se APK download ho jayega.

## GitHub ke alternatives (free)
- **GitLab**: free CI/CD.
- **Codeberg**: open-source projects ke liye.
- **Codemagic**: free build minutes, mobile apps ke liye.
- **GitHub Codespaces**: browser mein hi coding.

## Limits (sach baat)
- Ye asli installed apps ko floating nahi karta. Ye game ke upar mini browser tools deta hai.
- Kuch games overlay detect karke touch block kar sakte hain, aur online games mein anti-cheat flag ho sakta hai. Careful raho.
- YouTube/WhatsApp Web mobile WebView mein kabhi kabhi limited chalte hain.
