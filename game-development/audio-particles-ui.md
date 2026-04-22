# Audio, Particles & UI

## Sound Effects

### Phaser 3
```javascript
// Preload
this.load.audio('jump', ['jump.ogg', 'jump.mp3']); // fallback formats
this.load.audio('hit', 'hit.wav');

// Play (fire-and-forget)
this.sound.play('jump');
this.sound.play('jump', { volume: 0.7, rate: 1.0 });

// With pitch variation (avoids repetitive feel)
this.sound.play('hit', {
    volume: 0.8,
    rate: Phaser.Math.FloatBetween(0.9, 1.1) // slight random pitch
});

// Create reusable audio object
const jumpSfx = this.sound.add('jump', { volume: 0.7 });
jumpSfx.play();
```

### Unity
```csharp
[SerializeField] private AudioClip jumpClip;
[SerializeField] private AudioClip hitClip;
private AudioSource audioSource;

void Awake() { audioSource = GetComponent<AudioSource>(); }

void PlayJump() {
    audioSource.PlayOneShot(jumpClip, 0.8f); // clip, volume
}

void PlayHit() {
    audioSource.pitch = Random.Range(0.9f, 1.1f);
    audioSource.PlayOneShot(hitClip);
    audioSource.pitch = 1f;
}

// Play at position (3D spatial, but works in 2D with AudioListener on camera)
AudioSource.PlayClipAtPoint(explosionClip, transform.position, 0.9f);
```

### Audio Pooling for Rapid-Fire Sounds
When multiple sounds play simultaneously (e.g., rapid gunfire), use multiple AudioSources:
```csharp
// Unity
[SerializeField] private int poolSize = 5;
private AudioSource[] pool;

void Awake() {
    pool = new AudioSource[poolSize];
    for (int i = 0; i < poolSize; i++) {
        pool[i] = gameObject.AddComponent<AudioSource>();
        pool[i].playOnAwake = false;
    }
}

void PlayPooled(AudioClip clip) {
    foreach (var src in pool) {
        if (!src.isPlaying) { src.PlayOneShot(clip); return; }
    }
    pool[0].PlayOneShot(clip); // fallback: override oldest
}
```

---

## Music

### Phaser 3
```javascript
// Preload
this.load.audio('bgm', 'music.mp3');

// Play looping music
const music = this.sound.add('bgm', { loop: true, volume: 0.4 });
music.play();

// Crossfade between tracks
this.tweens.add({ targets: currentMusic, volume: 0, duration: 1000,
    onComplete: () => { currentMusic.stop(); }
});
newMusic.setVolume(0);
newMusic.play();
this.tweens.add({ targets: newMusic, volume: 0.4, duration: 1000 });

// Pause/resume all audio
this.sound.pauseAll();
this.sound.resumeAll();
```

### Unity
```csharp
// Separate AudioSource for music (Loop = true in Inspector)
[SerializeField] private AudioSource musicSource;

void PlayMusic(AudioClip clip) {
    musicSource.clip = clip;
    musicSource.loop = true;
    musicSource.volume = 0.4f;
    musicSource.Play();
}

// Crossfade with coroutine
IEnumerator CrossfadeMusic(AudioClip newClip, float duration) {
    float startVol = musicSource.volume;
    for (float t = 0; t < duration; t += Time.unscaledDeltaTime) {
        musicSource.volume = Mathf.Lerp(startVol, 0, t / duration);
        yield return null;
    }
    musicSource.clip = newClip;
    musicSource.Play();
    for (float t = 0; t < duration; t += Time.unscaledDeltaTime) {
        musicSource.volume = Mathf.Lerp(0, startVol, t / duration);
        yield return null;
    }
}
```

### Volume Ducking
Lower music volume during SFX or dialogue:
```csharp
// Unity AudioMixer approach (recommended):
// 1. Create AudioMixer with groups: Master > Music, SFX
// 2. Expose "MusicVolume" parameter
// 3. When important SFX plays, duck music:
audioMixer.SetFloat("MusicVolume", -10f); // lower by 10 dB
// Restore: audioMixer.SetFloat("MusicVolume", 0f);
```

---

## Particle Systems

### Phaser 3 (3.60+)
```javascript
// Explosion burst
const explosion = this.add.particles(x, y, 'particle', {
    speed: { min: 80, max: 200 },
    angle: { min: 0, max: 360 },
    scale: { start: 1, end: 0 },
    alpha: { start: 1, end: 0 },
    tint: [0xff0000, 0xff6600, 0xffff00], // red-orange-yellow
    lifespan: 600,
    quantity: 20,
    frequency: -1,   // -1 = explode mode (one burst, then stop)
    gravityY: 100
});
explosion.explode(20);
// Auto-destroy after particles die:
this.time.delayedCall(1000, () => explosion.destroy());

// Continuous emitter (dust trail, fire, rain)
const trail = this.add.particles(0, 0, 'particle', {
    speed: { min: 10, max: 30 },
    angle: { min: 260, max: 280 },
    scale: { start: 0.5, end: 0 },
    alpha: { start: 0.5, end: 0 },
    lifespan: 400,
    frequency: 50,   // emit every 50ms
    blendMode: 'ADD'  // additive blending for glow effects
});
trail.startFollow(player); // follow sprite
```

### Unity Particle System (2D)
1. Create > Effects > Particle System
2. Set **Renderer > Material** to a 2D sprite material (e.g., Sprites-Default)
3. Set **Renderer > Sorting Layer** appropriately
4. Key modules:

| Module | Settings for 2D |
|--------|----------------|
| Main | Duration, Looping, Start Lifetime, Start Speed, Start Size, Start Color, Simulation Space (World) |
| Emission | Rate over Time, or Bursts for explosions |
| Shape | Circle (edge or filled), Rectangle |
| Color over Lifetime | Gradient for fade effects |
| Size over Lifetime | Curve, typically shrinking |
| Renderer | Sprite material, Sorting Layer |

```csharp
// Trigger burst from code
[SerializeField] private ParticleSystem explosionPS;

void Explode(Vector3 position) {
    var ps = Instantiate(explosionPS, position, Quaternion.identity);
    ps.Play();
    Destroy(ps.gameObject, ps.main.duration + ps.main.startLifetime.constantMax);
}
```

### Common Particle Recipes
| Effect | Settings |
|--------|----------|
| **Explosion** | Burst 20-50, speed high, short lifetime, scale start→0, radial direction |
| **Dust on landing** | Burst 5-10, speed low, short lifetime, horizontal spread, grey tint |
| **Trail/smoke** | Continuous, follow object, low speed, medium lifetime, alpha fade, grey |
| **Sparks** | Burst 10, speed high, gravity, tiny particles, yellow/orange |
| **Rain** | Continuous from top edge, downward direction, high speed, thin shape |
| **Fire** | Continuous, upward speed, red→orange→yellow tint, alpha fade, additive blend |
| **Collect sparkle** | Burst on pickup, radial, gold tint, scale down, short lifetime |

---

## UI / HUD

### Health Bar (Fill Bar)

#### Phaser 3
```javascript
// Using graphics
create() {
    this.healthBarBg = this.add.rectangle(100, 30, 200, 20, 0x333333)
        .setOrigin(0, 0.5).setScrollFactor(0);
    this.healthBarFill = this.add.rectangle(100, 30, 200, 20, 0x00ff00)
        .setOrigin(0, 0.5).setScrollFactor(0);
}

updateHealth(current, max) {
    const ratio = current / max;
    this.healthBarFill.setDisplaySize(200 * ratio, 20);
    // Color change: green → yellow → red
    if (ratio > 0.5) this.healthBarFill.setFillStyle(0x00ff00);
    else if (ratio > 0.25) this.healthBarFill.setFillStyle(0xffff00);
    else this.healthBarFill.setFillStyle(0xff0000);
}
```

#### Unity
```csharp
// UI Image with Image Type = Filled, Fill Method = Horizontal
[SerializeField] private Image healthFill;

public void UpdateHealth(float current, float max) {
    float ratio = current / max;
    healthFill.fillAmount = ratio;
    healthFill.color = Color.Lerp(Color.red, Color.green, ratio);
}
```

### Score Display
```javascript
// Phaser
const scoreText = this.add.text(16, 16, 'Score: 0', {
    fontFamily: 'Arial', fontSize: '24px', color: '#ffffff',
    stroke: '#000000', strokeThickness: 3
}).setScrollFactor(0).setDepth(100);

// Update: scoreText.setText('Score: ' + score);
```

```csharp
// Unity (TextMeshPro)
[SerializeField] private TextMeshProUGUI scoreText;
scoreText.text = $"Score: {score}";
```

---

## Menu Systems

### Main Menu (Phaser 3)
```javascript
class MainMenu extends Phaser.Scene {
    constructor() { super('MainMenu'); }

    create() {
        this.add.text(400, 150, 'MY GAME', {
            fontSize: '64px', fontFamily: 'Arial', color: '#ffffff'
        }).setOrigin(0.5);

        const playBtn = this.add.text(400, 300, 'PLAY', {
            fontSize: '32px', color: '#00ff00'
        }).setOrigin(0.5).setInteractive({ useHandCursor: true });

        playBtn.on('pointerover', () => playBtn.setColor('#ffff00'));
        playBtn.on('pointerout', () => playBtn.setColor('#00ff00'));
        playBtn.on('pointerdown', () => this.scene.start('Game'));
    }
}
```

### Pause Menu (Unity)
```csharp
[SerializeField] private GameObject pausePanel;

void Update() {
    if (Input.GetKeyDown(KeyCode.Escape)) TogglePause();
}

void TogglePause() {
    bool isPaused = Time.timeScale == 0f;
    Time.timeScale = isPaused ? 1f : 0f;
    pausePanel.SetActive(!isPaused);
}

// Button callbacks (set in Inspector)
public void OnResume() { Time.timeScale = 1f; pausePanel.SetActive(false); }
public void OnQuit() { Time.timeScale = 1f; SceneManager.LoadScene("MainMenu"); }
```

---

## Screen Transitions

### Fade (Phaser)
```javascript
// Fade out, then switch scene
this.cameras.main.fadeOut(500, 0, 0, 0); // duration, r, g, b
this.cameras.main.once('camerafadeoutcomplete', () => {
    this.scene.start('NextScene');
});

// Fade in on new scene's create()
this.cameras.main.fadeIn(500);
```

### Fade (Unity)
```csharp
// Canvas with full-screen black Image (raycastTarget = false)
[SerializeField] private CanvasGroup fadeCanvas;

IEnumerator FadeOut(float duration) {
    fadeCanvas.blocksRaycasts = true;
    float t = 0;
    while (t < duration) {
        t += Time.unscaledDeltaTime;
        fadeCanvas.alpha = t / duration;
        yield return null;
    }
    fadeCanvas.alpha = 1;
}

IEnumerator FadeIn(float duration) {
    float t = 0;
    while (t < duration) {
        t += Time.unscaledDeltaTime;
        fadeCanvas.alpha = 1 - (t / duration);
        yield return null;
    }
    fadeCanvas.alpha = 0;
    fadeCanvas.blocksRaycasts = false;
}

// Usage:
IEnumerator TransitionToScene(string sceneName) {
    yield return StartCoroutine(FadeOut(0.5f));
    SceneManager.LoadScene(sceneName);
    yield return StartCoroutine(FadeIn(0.5f));
}
```

---

## Dialog / Text Boxes

### Typewriter Effect (Phaser)
```javascript
showDialog(text, x, y) {
    const dialog = this.add.text(x, y, '', {
        fontSize: '18px', color: '#ffffff', wordWrap: { width: 300 }
    }).setScrollFactor(0).setDepth(200);

    let i = 0;
    this.time.addEvent({
        delay: 30, // ms per character
        repeat: text.length - 1,
        callback: () => { dialog.text += text[i]; i++; }
    });
}
```

### Typewriter Effect (Unity)
```csharp
[SerializeField] private TextMeshProUGUI dialogText;

IEnumerator TypeText(string text, float delay = 0.03f) {
    dialogText.text = "";
    foreach (char c in text) {
        dialogText.text += c;
        yield return new WaitForSecondsRealtime(delay);
    }
}

// Skip: set dialogText.text = fullText immediately on click
```

### Dialog System Pattern
```
DialogManager
├── Show(text, speaker, portrait)
├── Advance() → next line or close
├── IsActive → bool
└── OnDialogComplete event

Usage:
1. Show dialog box with text
2. Typewriter effect plays
3. Click/Space → show full text immediately OR advance to next line
4. After last line → close dialog, resume gameplay
```
