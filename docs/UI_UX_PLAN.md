# Math Defender – UI/UX Technical Implementation Plan (Unity / Android Portrait)

Target platform: **Android phones, portrait-only**, built with **Unity 2021+** (URP or Built‑in).

This document specifies **how to implement** the UI/UX of *Math Defender* so that any Unity developer can build it consistently.

---
## 1. UI Architecture

### 1.1 Canvas Types & Overall Structure

Use **three main Canvas layers**:

1. **UI_Overlay_Canvas (Screen Space – Overlay)**
   - For: HUD, score, timer, lives, combo, buttons, global overlays (settings, pause, end-game panels), basic toasts.
   - Render Mode: **Screen Space – Overlay**.
   - Sorting Order: **0**.
   - Pixel Perfect: **On** (if using pixel-art or to avoid blurriness; otherwise optional, test on device).
   - Canvas Scaler: see section **6. Responsive Design**.

2. **UI_Camera_Canvas (Screen Space – Camera)**
   - For: UI that must align with camera FOV or depth (e.g., reticles, screen‑space VFX that should be post‑processed, “hit flash” overlays tied to camera, vignette, chromatic flashes).
   - Render Mode: **Screen Space – Camera**.
   - Render Camera: **Main gameplay camera**.
   - Plane Distance: **1–2** (tuned to be in front of 3D world but behind Overlay UI if layered via sorting orders/layers).
   - Sorting Layer: `UI`.
   - Sorting Order: **1** (above world, below Overlay Canvas where necessary; or vice versa depending on chosen stack; keep consistent).

3. **UI_World_Canvas (World Space)**
   - For: world‑anchored labels/indicators (enemy math problems, damage numbers, 3D indicators, world hints).
   - Render Mode: **World Space**.
   - Separate canvases per **logical cluster** (e.g., enemies, tutorial markers) to avoid large invalidation.
   - Each world canvas is **non‑interaction** (no raycasting necessary except where needed).
   - Sorting Layer: `WorldUI`.
   - Ensure world-space canvases are on their own layer (e.g., `Layer: WorldUI`) for selective rendering/culling.

**Rule of thumb**:
- **Game HUD / menus** → Overlay Canvas.
- **Camera-aligned FX / post‑style UI** → Camera Canvas.
- **Stuff attached to 3D objects** → World Canvas with billboard behavior.

### 1.2 Canvas Hierarchy (Suggested)

```text
Canvas_UI_Overlay (Screen Space – Overlay)
  - EventSystem
  - SafeAreaRoot (handles Android notch/safe areas)
    - HUD
      - TopBar
        - ScoreText (TMP)
        - TimerText (TMP)
        - LivesContainer
      - ComboBar
        - ComboValueText (TMP)
        - ComboFill (Image)
    - BottomBar
      - MainActionButton (solve / fire / confirm)
      - PauseButton
    - ToastLayer
      - ToastContainer (for generic messages)
    - ModalLayer
      - PausePanel
      - SettingsPanel
      - GameOverPanel

Canvas_UI_Camera (Screen Space – Camera)
  - HitFlashOverlay (full-screen Image with additive/soft light shader)
  - FocusVignette
  - ScreenShakeAnchor (optional)

WorldUI_Root
  - Canvas_EnemyLabels (World Space)
    - (runtime) EnemyLabel_# (TMP + billboard)
  - Canvas_DamageNumbers (World Space)
    - (runtime) DamageNumber_# (TMP + billboard)
```

Use a single **EventSystem** in the scene, associated with the Overlay Canvas.

### 1.3 Navigation Flow

States (managed via a simple state machine or Scene‑based approach):
- **MainMenu** → **Gameplay** → **PauseOverlay** → **GameOver**.

Implementation detail:
- Prefer **one main scene** with additive loading for heavy 3D content, but keep **UI prefabs** as separate, reusable prefabs.
- All overlays (Pause, Settings, GameOver) are **child panels** under `ModalLayer`, enabled/disabled via a common `UIStateController`.

---
## 2. Rendering System

### 2.1 Text Rendering – TextMeshPro

All text in the project must use **TextMeshPro (TMP)**.

Configuration:
- Import TMP Essentials.
- Create at least two TMP font assets:
  - `Font_NeonPrimary_SDF` – main UI font.
  - `Font_MonoDigits_SDF` – numeric font for scores/timers if needed.
- Enable **Fallback Font Assets** for languages as required.
- Set `Extra Padding` where glow/outline is used.

Usage guidelines:
- Prefer **SDF** fonts for sharp neon outlines and glows.
- Keep **max material variants** low (e.g., 2–3) to minimize draw calls.
- Reuse **shared material presets** for common states:
  - `NeonPrimary_Default`
  - `NeonPrimary_Highlight`
  - `NeonPrimary_Disabled`
  - `NeonImpact_Popup` (for Math Impact FX)

### 2.2 World-Space Text & Billboard Shaders

World-space text is used for:
- **Enemy equations** floating above enemies.
- **Damage/score popups** (“+10”, “Perfect!”, etc.).

Implementation:
- Each world-space text is **TMP Text** in a **small World Space Canvas**.
- Attach a `Billboard` script:

```csharp
public class Billboard : MonoBehaviour
{
    public Camera targetCamera;

    void LateUpdate()
    {
        if (!targetCamera) targetCamera = Camera.main;
        var camTransform = targetCamera.transform;
        // Face camera, keep upright
        transform.LookAt(transform.position + camTransform.rotation * Vector3.forward,
                         camTransform.rotation * Vector3.up);
    }
}
```

- Optionally, use a **billboard shader** (URP/HDRP) that rotates in vertex stage, but for simplicity the script above is fine.

Shader requirements (if custom):
- Unlit, alpha‑blended.
- Supports emissive color for neon glow.
- Optionally uses **soft particles** to avoid hard intersections with geometry.

### 2.3 Canvas Splitting & Optimization

Goals: minimize **Canvas rebuilds** and **overdraw**.

Guidelines:
1. **Static UI vs Dynamic UI**
   - Put highly dynamic elements (scores, timers, rapidly updating bars) in a **separate child Canvas** under the Overlay Canvas.
   - Example:

```text
Canvas_UI_Overlay
  - StaticLayer (logo, frame, decorations)
  - DynamicLayer (text values / bars that update each frame)
  - ModalLayer
```

   - Configure `DynamicLayer` as **override sorting** with a higher order, so its rebuilds don’t force rebuilds of `StaticLayer`.

2. **World UI Canvases**
   - Use **one world canvas per logical group** (EnemyLabels, DamageNumbers) rather than per element, but avoid one giant canvas covering the whole level if the game area is big.
   - Set `Additional Shader Channels` minimally (e.g., `TexCoord1` off unless needed).

3. **Batching & Materials**
   - Reuse **single shared material** for all HUD texts.
   - Use a limited set of **UI sprite atlases** for icons and frames.

4. **Update Frequency**
   - For text that doesn’t need per‑frame updates (e.g., score), update only when changed, not every frame.

---
## 3. Visual Style – “Neon Cyber‑Clean”

### 3.1 Color Palette

Base palette (define as ScriptableObject or central constants):
- **Background / Dark Base**: `#050811` – near‑black, slightly blue.
- **Primary Neon Cyan**: `#35F2FF` – main accents, primary buttons, main highlights.
- **Secondary Neon Magenta**: `#FF3BB8` – combo / special state, warnings.
- **Energy Lime**: `#C8FF3D` – success / “Correct answer” color.
- **Warning Amber**: `#FFC73D` – timer low, caution.
- **Error Red**: `#FF4D4D` – wrong answer, damage.
- **Neutral Grey**: `#A5B2C2` – secondary text, disabled states.

Gradients (for bars and big elements):
- **HP/Shield Bar**: Cyan → Lime.
- **Combo Bar**: Magenta → Cyan.

### 3.2 Fonts

Style requirements:
- Primary font: **geometric sans** or **techno sans** (e.g., Eurostile-like, Nexa, Rajdhani). Must be legible on small screens.
- For numbers: optional **monospaced** or condensed numeric font to make score/timer stable.

Rules:
- No serif fonts.
- Use max **two font families** across the game.
- Consistent **hierarchy**:
  - H1 (large title) – 32–40 pt (Reference Resolution 1080×1920), bold, neon cyan with subtle outer glow.
  - H2 (section title) – 26–30 pt, bold.
  - Body – 20–24 pt, regular.
  - Captions / labels – 18–20 pt.

### 3.3 UI Elements Design

Buttons:
- Base: rounded rectangle (8–12 px radius at ref resolution) with **neon border** and subtle inner gradient.
- Idle state:
  - Fill: dark blue‑grey (`#081221` → `#050811`).
  - Border: primary neon cyan, 2–3 px.
  - Text: cyan or white with 1 px outer glow.
- Hover (for testing / generic state) / Focused (active):
  - Increase border brightness.
  - Slight scale up (1.05) and glow bloom.
- Pressed:
  - Darken fill.
  - Scale down (0.95) quickly.

Panels:
- Semi‑transparent dark background (`alpha 0.8`), mild gradient.
- Thin neon border or corner accents.
- Minimal noise, no heavy textures; keep it “clean”.

Icons:
- Simple geometric shapes with neon outlines.
- Use **single‑color vector style** with glow added by shader/material, not baked into texture.

HUD Layout (portrait):
- **Top bar**: score (left), timer (center), lives/hearts (right).
- **Middle**: gameplay area, largely free of static UI.
- **Bottom**: main answer/interaction controls, as large and thumb‑reachable.

---
## 4. Feedback & FX – “Math Impact”

“Math Impact” is the visual + audio feedback when the player interacts with math elements (solves correctly/incorrectly, streaks, combos).

### 4.1 Correct Answer Feedback

Elements:
1. **World-space popup** above the target:
   - Text: e.g., `+10`, `Perfect!`, `Streak x3`.
   - Implementation:
     - Spawn from **Object Pool** (see section 5).
     - TMP world-space text with `NeonImpact_Popup` material (bright cyan/lime gradient, strong glow).
     - Animation (via DOTween, LeanTween, or custom coroutine):
       - Start scale ~0.7 → punch to 1.1 in 0.1 s.
       - Move upward 0.5–1 unit over 0.4–0.6 s.
       - Fade alpha to 0.
     - Return to pool when finished.

2. **Screen-space HUD feedback**:
   - Brief **glow pulse** around the score and/or combo bar.
   - Implementation: animate outline thickness/emission or scale using an animation curve.

3. **Subtle camera / vignette FX** (optional):
   - Short (0.1–0.2 s) soft vignette color pulse (cyan or lime) on Camera Canvas.

### 4.2 Wrong Answer / Damage Feedback

Elements:
1. **Screen flash** (Camera Canvas):
   - Red semi‑transparent overlay image covering screen.
   - Animation: quick in (alpha 0 → 0.3 in 0.05 s) and out (0.3 → 0 in 0.15 s).

2. **World-space popup**:
   - Text: `-5`, `Oops`, etc., in **Error Red**.
   - Slight shake (random jitter) before fading.

3. **HUD Shake** (optional, light):
   - Shake lives icon / HP bar horizontally 2–3 px for 0.1–0.15 s.

### 4.3 Combo / Streak FX

When the player reaches thresholds (x3, x5, x10):
- **Combo text** in HUD grows and changes color:
  - x3 → cyan.
  - x5 → cyan + magenta pulse.
  - x10 → animated gradient + glow.
- Trigger **center-screen text** like `COMBO x5!` in Overlay layer:
  - Anim: scale in (0 → 1) + rotation wiggle + fade out.

Implementation note:
- Use a **central `UIFxController`** that exposes methods:

```csharp
public void PlayCorrectAnswerFX(Vector3 worldPosition, int scoreDelta);
public void PlayWrongAnswerFX(Vector3 worldPosition, int penalty);
public void PlayComboFX(int comboLevel);
```

Gameplay systems call this without knowing internal FX details.

### 4.4 Animation Implementation Details

- Use **Animator** or tween library (DOTween recommended) for most UI animations.
- All configurable parameters (durations, scales, colors) exposed as **ScriptableObjects** or `public` fields.
- Ensure animations are **frame-rate independent** (use `Time.deltaTime`).

---
## 5. Performance for Android

### 5.1 Object Pooling for UI Elements

Pooled elements:
- World-space text popups (damage, score, combo).
- Repeating HUD elements such as floating hints.

Implementation:
- Create a generic **ObjectPool<T>** or use Unity’s built-in pooling (if available in the chosen version).

Example structure:

```csharp
public class UIPopupPool : MonoBehaviour
{
    [SerializeField] private TextMeshProUGUI popupPrefab;
    [SerializeField] private int initialSize = 20;

    private readonly Queue<TextMeshProUGUI> pool = new();

    void Awake()
    {
        for (int i = 0; i < initialSize; i++)
        {
            var inst = Instantiate(popupPrefab, transform);
            inst.gameObject.SetActive(false);
            pool.Enqueue(inst);
        }
    }

    public TextMeshProUGUI Get()
    {
        if (pool.Count == 0)
        {
            var inst = Instantiate(popupPrefab, transform);
            inst.gameObject.SetActive(false);
            pool.Enqueue(inst);
        }
        var popup = pool.Dequeue();
        popup.gameObject.SetActive(true);
        return popup;
    }

    public void Release(TextMeshProUGUI popup)
    {
        popup.gameObject.SetActive(false);
        pool.Enqueue(popup);
    }
}
```

- For **world-space** popups, use `TextMeshPro` (3D) or TMP in World Space Canvas instead of `TextMeshProUGUI` if more appropriate.
- Integrate pooling with FX scripts (on animation end, call `Release`).

### 5.2 Zero-Allocation Text Updates

To avoid GC allocations during gameplay:

1. **Cache TMP references**
   - Don’t use `GetComponent<T>()` every frame; cache in `Awake`.

2. **Use `StringBuilder` or TMP’s `SetText`**
   - Replace `text = score.ToString()` with:

```csharp
scoreText.SetText("{0}", score);
```

   - This avoids intermediate string allocations.

3. **Avoid `string.Format` / concatenation** in Update
   - Prebuild constant parts, only inject numbers.

4. **Update only on change**
   - Use properties with setter side‑effects:

```csharp
private int _score;
public int Score
{
    get => _score;
    set
    {
        if (_score == value) return;
        _score = value;
        scoreText.SetText("{0}", _score);
    }
}
```

5. **Disable unused UI**
   - When panels are hidden, deactivate GameObjects to stop layout/calculation.

### 5.3 General Mobile Optimizations

- Limit number of **active Canvases** (Overlay, Camera, 1–3 World UI groups).
- Keep **overdraw** under control: dark backgrounds, minimal full‑screen transparent images.
- Use **Sprite Atlases** for icons (Unity Sprite Atlas).
- Limit UI resolution where possible (see Canvas Scaler below).

---
## 6. Responsive Design – Canvas Scaler for Portrait Android

### 6.1 Target Resolution & Scaling Mode

Use the **Overlay Canvas** as the master for scaling.

Settings (Canvas Scaler component):
- UI Scale Mode: **Scale With Screen Size**.
- Reference Resolution: **1080 × 1920** (portrait).
- Screen Match Mode: **Match Width Or Height**.
- Match: **0.5** (balanced) or **0.6–0.7** (favor height slightly; test on common devices).
- Reference Pixels Per Unit: **100** (standard; adjust only if needed for art).

Rationale:
- Most Android phones are in the 16:9–20:9 range; a 1080×1920 reference works well.

### 6.2 Safe Area Handling

Android devices with notches / rounded corners require safe area support.

Implementation:
- Create `SafeAreaRoot` under Overlay Canvas.
- Attach a `SafeArea` script that reads `Screen.safeArea` and applies it as padding/margins.

Pseudo-code:

```csharp
[RequireComponent(typeof(RectTransform))]
public class SafeArea : MonoBehaviour
{
    private RectTransform _panel;

    void Awake()
    {
        _panel = GetComponent<RectTransform>();
        ApplySafeArea();
    }

    void ApplySafeArea()
    {
        Rect safeArea = Screen.safeArea;
        Vector2 anchorMin = safeArea.position;
        Vector2 anchorMax = safeArea.position + safeArea.size;

        anchorMin.x /= Screen.width;
        anchorMin.y /= Screen.height;
        anchorMax.x /= Screen.width;
        anchorMax.y /= Screen.height;

        _panel.anchorMin = anchorMin;
        _panel.anchorMax = anchorMax;
    }
}
```

- Place all HUD content as children of `SafeAreaRoot`.

### 6.3 Layout Components & Anchors

- For **top bar**: anchor to **top‑stretch** (left & right filled, pivot at top).
- For **bottom controls**: anchor to **bottom‑stretch**.
- For **center gameplay hints**: anchor to **middle center**, use `LayoutGroup`s carefully.

Use **Layout Groups** only where necessary; avoid deep nesting to reduce layout calculation cost.

### 6.4 Testing Matrix

Test layouts at minimum on:
- 720×1520 (low‑end 18.5:9).
- 1080×1920 (reference).
- 1440×3040 (high‑end tall phone).

Verify:
- Text sizes remain readable (minimum 14–16 pt at smallest screen).
- Critical controls remain within **thumb‑reachable** zones (lower 60% of the screen).

---
## 7. Implementation Checklist

1. **Set up canvases** (Overlay, Camera, World UI groups) and EventSystem.
2. Configure **Canvas Scaler** with reference resolution and match settings.
3. Implement **SafeArea** system and wrap HUD elements.
4. Create **TMP font assets** and shared materials (normal, highlight, impact).
5. Define **color palette constants** or ScriptableObject.
6. Build **HUD prefabs** (TopBar, BottomBar, ComboBar, Toast, Modal panels).
7. Implement **Billboard** behavior for world-space text.
8. Implement **UIFxController** with Math Impact, error, and combo FX.
9. Create **object pools** for world-space popups and frequently used UI items.
10. Refactor score/combo/timer updates to use **zero-allocation** patterns.
11. Profile on Android devices for **overdraw** and **GC allocs**; iterate on canvas splits and FX.

This plan should give developers enough detail to implement the Math Defender UI/UX consistently and efficiently for Android portrait devices.