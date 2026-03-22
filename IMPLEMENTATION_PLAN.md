# llmfit-desktop Tauri + React GUI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** llmfit-desktop Tauri 2 uygulamasına React UI entegrasyonu — mevcut llmfit-web React kodunu adapte ederek native masaüstü GUI oluşturmak.

**Architecture:** Tauri 2 IPC (invoke) üzerinden Rust backend'e bağlanan React + Vite frontend. Tüm veri local'den alınır, internet bağlantısı gerekmez.

**Tech Stack:** Tauri 2, React 18, Vite 6, @tauri-apps/api, Vitest

---

## File Structure

```
llmfit-desktop/
├── Cargo.toml                    ← llmfit-core dependency + tauri 2 (mevcut)
├── build.rs                      ← basitleştirilecek
├── tauri.conf.json               ← frontendDist, devUrl, window ayarları [DEĞİŞTİRİLECEK]
├── src/main.rs                  ← Tauri commands (genişletilecek)
├── frontend/                    ← YENİ: llmfit-web'den taşınan React
│   ├── package.json
│   ├── vite.config.js
│   ├── index.html
│   ├── src/
│   │   ├── main.jsx
│   │   ├── App.jsx
│   │   ├── api.js
│   │   ├── styles.css
│   │   ├── App.test.jsx
│   │   ├── api.test.js
│   │   └── test-setup.js
│   └── vitest.config.js
├── ui/                           ← ESKİ: minimal vanilla JS (SİLİNEBİLİR)
└── icons/                        ← Mevcut kalır
```

---

## Task 1: Rust Backend Genişletme (ModelFitInfo)

**Files:**
- Modify: `llmfit-desktop/src/main.rs:11-55`

- [ ] **Step 1: ModelFitInfo struct'ı genişlet**

```rust
#[derive(Serialize, Clone)]
struct ScoreComponents {
    quality: f64,
    speed: f64,
    fit: f64,
    context: f64,
}

#[derive(Serialize, Clone)]
struct ModelFitInfo {
    // Temel bilgiler
    name: String,
    provider: String,
    parameter_count: String,
    params_b: f64,
    context_length: u32,
    use_case: String,
    category: String,
    release_date: Option<String>,
    is_moe: bool,

    // Fit sonuçları
    fit_level: String,       // "perfect", "good", "marginal", "too_tight"
    fit_label: String,       // "Perfect", "Good", "Marginal", "Too Tight"
    run_mode: String,        // "gpu", "cpu_offload", "moe_offload", "cpu_only"
    run_mode_label: String,  // "GPU", "CPU Offload", "MoE Offload", "CPU"

    // Puanlama
    score: f64,
    score_components: ScoreComponents,
    estimated_tps: f64,

    // Çalışma bilgileri
    runtime: String,         // "llamacpp", "mlx", "vllm"
    runtime_label: String,   // "llama.cpp", "MLX", "vLLM"
    best_quant: String,

    // Bellek
    memory_required_gb: f64,
    memory_available_gb: f64,
    utilization_pct: f64,

    // Ek bilgiler
    installed: bool,
    notes: Vec<String>,
    gguf_sources: Vec<String>,
}
```

- [ ] **Step 2: get_model_fits fonksiyonunu güncelle (satır 86-132)**

Mevcut `ModelFitInfo` dönüşü struct'ı yeni alanlarla değiştir:

```rust
Ok(fits
    .into_iter()
    .map(|f| ModelFitInfo {
        // Temel
        name: f.model.name.clone(),
        provider: f.model.provider.clone(),
        parameter_count: f.model.parameter_count.clone(),
        params_b: f.model.parameters_raw.unwrap_or(0) as f64 / 1e9,
        context_length: f.model.context_length,
        use_case: format!("{:?}", f.use_case),
        category: f.use_case.label().to_string(),
        release_date: f.model.release_date.clone(),
        is_moe: f.model.is_moe,

        // Fit
        fit_level: match f.fit_level {
            FitLevel::Perfect => "perfect".to_string(),
            FitLevel::Good => "good".to_string(),
            FitLevel::Marginal => "marginal".to_string(),
            FitLevel::TooTight => "too_tight".to_string(),
        },
        fit_label: f.fit_text().to_string(),
        run_mode: match f.run_mode {
            RunMode::Gpu => "gpu".to_string(),
            RunMode::CpuOffload => "cpu_offload".to_string(),
            RunMode::CpuOnly => "cpu_only".to_string(),
            RunMode::MoeOffload => "moe_offload".to_string(),
        },
        run_mode_label: f.run_mode_text().to_string(),

        // Score
        score: f.score,
        score_components: ScoreComponents {
            quality: f.score_components.quality,
            speed: f.score_components.speed,
            fit: f.score_components.fit,
            context: f.score_components.context,
        },
        estimated_tps: f.estimated_tps,

        // Runtime
        runtime: match f.runtime {
            InferenceRuntime::LlamaCpp => "llamacpp".to_string(),
            InferenceRuntime::Mlx => "mlx".to_string(),
            InferenceRuntime::Vllm => "vllm".to_string(),
        },
        runtime_label: f.runtime_text().to_string(),
        best_quant: f.best_quant.clone(),

        // Memory
        memory_required_gb: f.memory_required_gb,
        memory_available_gb: f.memory_available_gb,
        utilization_pct: f.utilization_pct,

        // Extra
        installed: f.installed,
        notes: f.notes.clone(),
        gguf_sources: f.model.gguf_sources.clone(),
    })
    .collect())
```

- [ ] **Step 3: InferenceRuntime import ekle (satır 3)**

```rust
use llmfit_core::fit::{FitLevel, InferenceRuntime, ModelFit, RunMode};
```

- [ ] **Step 4: Test: cargo check -p llmfit-desktop**

Run: `cargo check -p llmfit-desktop`
Expected: No errors

- [ ] **Step 5: Commit**

```bash
git add llmfit-desktop/src/main.rs
git commit -m "feat(desktop): extend ModelFitInfo with full API format fields"
```

---

## Task 2: Rust Backend Genişletme (SystemInfo)

**Files:**
- Modify: `llmfit-desktop/src/main.rs:11-28`

- [ ] **Step 1: GpuInfoJs ve SystemInfo struct'larını güncelle**

```rust
#[derive(Serialize)]
struct GpuInfoJs {
    name: String,
    vram_gb: Option<f64>,
    backend: String,
    count: u32,
    unified_memory: bool,
}

#[derive(Serialize)]
struct SystemInfo {
    total_ram_gb: f64,
    available_ram_gb: f64,
    cpu_name: String,
    cpu_cores: usize,
    gpus: Vec<GpuInfoJs>,
    unified_memory: bool,
    backend: String,
    has_gpu: bool,
    gpu_vram_gb: Option<f64>,
}
```

- [ ] **Step 2: get_system_specs fonksiyonunu güncelle (satır 62-84)**

```rust
#[tauri::command]
fn get_system_specs() -> Result<SystemInfo, String> {
    let specs = SystemSpecs::detect();
    let gpus = specs
        .gpus
        .iter()
        .map(|g| GpuInfoJs {
            name: g.name.clone(),
            vram_gb: g.vram_gb,
            backend: g.backend.label().to_string(),
            count: g.count,
            unified_memory: g.unified_memory,
        })
        .collect();
    Ok(SystemInfo {
        total_ram_gb: specs.total_ram_gb,
        available_ram_gb: specs.available_ram_gb,
        cpu_name: specs.cpu_name.clone(),
        cpu_cores: specs.total_cpu_cores,
        gpus,
        unified_memory: specs.unified_memory,
        backend: specs.backend.label().to_string(),
        has_gpu: specs.has_gpu,
        gpu_vram_gb: specs.gpu_vram_gb,
    })
}
```

- [ ] **Step 3: Import ekle**

```rust
use llmfit_core::hardware::SystemSpecs;
```

- [ ] **Step 4: Test: cargo check -p llmfit-desktop**

Run: `cargo check -p llmfit-desktop`
Expected: No errors

- [ ] **Step 5: Commit**

```bash
git add llmfit-desktop/src/main.rs
git commit -m "feat(desktop): extend SystemInfo with backend, has_gpu, gpu_vram_gb"
```

---

## Task 3: Frontend Kurulumu (package.json + vite.config.js)

**Files:**
- Create: `llmfit-desktop/frontend/package.json`
- Create: `llmfit-desktop/frontend/vite.config.js`
- Create: `llmfit-desktop/frontend/index.html`

- [ ] **Step 1: package.json oluştur**

```json
{
  "name": "llmfit-desktop-frontend",
  "private": true,
  "version": "0.8.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest run"
  },
  "dependencies": {
    "@tauri-apps/api": "^2.2.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.4",
    "vite": "^6.0.7",
    "vitest": "^2.1.8",
    "@testing-library/react": "^16.2.0",
    "jsdom": "^25.0.1"
  }
}
```

- [ ] **Step 2: vite.config.js oluştur**

```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  clearScreen: false,
  server: {
    port: 1420,
    strictPort: true,
    watch: {
      ignored: ['**/src-tauri/**'],
    },
  },
  build: {
    outDir: 'dist',
    target: 'esnext',
    minify: 'esbuild',
  },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test-setup.js'],
  },
});
```

- [ ] **Step 3: index.html oluştur**

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>llmfit — LLM Hardware Fitting</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

- [ ] **Step 4: npm install çalıştır**

Run: `cd llmfit-desktop/frontend && npm install`
Expected: packages installed

- [ ] **Step 5: Commit**

```bash
git add llmfit-desktop/frontend/package.json llmfit-desktop/frontend/vite.config.js llmfit-desktop/frontend/index.html
git commit -m "feat(desktop): scaffold React + Vite frontend structure"
```

---

## Task 4: React Dosyalarını Taşıma

**Files:**
- Create: `llmfit-desktop/frontend/src/main.jsx`
- Create: `llmfit-desktop/frontend/src/App.jsx`
- Create: `llmfit-desktop/frontend/src/styles.css`
- Create: `llmfit-desktop/frontend/src/test-setup.js`
- Create: `llmfit-desktop/frontend/src/App.test.jsx`
- Create: `llmfit-desktop/frontend/src/api.test.js`
- Create: `llmfit-desktop/frontend/vitest.config.js`

- [ ] **Step 1: main.jsx kopyala** (llmfit-web/src/main.jsx aynı)

```jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './styles.css';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

- [ ] **Step 2: styles.css kopyala** (llmfit-web/src/styles.css — 588 satır, aynen kopyala)

Kopyala: `llmfit-web/src/styles.css` → `llmfit-desktop/frontend/src/styles.css`

- [ ] **Step 3: App.jsx kopyala** (llmfit-web/src/App.jsx — 569 satır)

Kopyala: `llmfit-web/src/App.jsx` → `llmfit-desktop/frontend/src/App.jsx`

- [ ] **Step 4: test-setup.js kopyala**

Kopyala: `llmfit-web/src/test-setup.js` → `llmfit-desktop/frontend/src/test-setup.js`

- [ ] **Step 5: App.test.jsx kopyala**

Kopyala: `llmfit-web/src/App.test.jsx` → `llmfit-desktop/frontend/src/App.test.jsx`

- [ ] **Step 6: api.test.js kopyala**

Kopyala: `llmfit-web/src/api.test.js` → `llmfit-desktop/frontend/src/api.test.js`

- [ ] **Step 7: vitest.config.js oluştur**

```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test-setup.js'],
  },
});
```

- [ ] **Step 8: npm run test çalıştır (frontend/)**

Run: `cd llmfit-desktop/frontend && npm run test`
Expected: Testler çalışır (fetch mock'lama olmadığı için bazıları fail olabilir, önemli değil)

- [ ] **Step 9: npm run build çalıştır**

Run: `cd llmfit-desktop/frontend && npm run build`
Expected: dist/ klasörü oluşur

- [ ] **Step 10: Commit**

```bash
git add llmfit-desktop/frontend/src/
git commit -m "feat(desktop): copy React UI from llmfit-web"
```

---

## Task 5: API Katmanı (api.js) Yazma

**Files:**
- Create: `llmfit-desktop/frontend/src/api.js`

- [ ] **Step 1: api.js yaz — tauri invoke wrapper**

```javascript
import { invoke } from '@tauri-apps/api/core';

export const DEFAULT_FILTERS = {
  search: '',
  minFit: 'marginal',
  runtime: 'any',
  useCase: 'all',
  provider: '',
  sort: 'score',
  limit: '50'
};

export async function fetchSystemInfo(signal) {
  const specs = await invoke('get_system_specs');
  return {
    node: { name: 'desktop', os: getOs() },
    system: specs
  };
}

export async function fetchModels(filters, signal) {
  const allFits = await invoke('get_model_fits');
  const fetchedModels = Array.isArray(allFits) ? allFits : [];
  const fitFiltered = applyClientFitFilter(fetchedModels, filters.minFit);
  const limit = Number.parseInt(filters.limit, 10);
  const models = Number.isFinite(limit) && limit > 0 ? fitFiltered.slice(0, limit) : fitFiltered;
  const filtered = applyFilters(models, filters);

  return {
    node: { name: 'desktop', os: getOs() },
    system: await invoke('get_system_specs'),
    total_models: fitFiltered.length,
    returned_models: filtered.length,
    filters: filters,
    models: filtered
  };
}

function getOs() {
  if (typeof navigator !== 'undefined') {
    const ua = navigator.userAgent;
    if (ua.includes('Mac')) return 'macOS';
    if (ua.includes('Win')) return 'Windows';
    if (ua.includes('Linux')) return 'Linux';
    return ua;
  }
  return 'unknown';
}

function fitRank(level) {
  switch (level) {
    case 'perfect': return 3;
    case 'good': return 2;
    case 'marginal': return 1;
    case 'too_tight': return 0;
    default: return -1;
  }
}

function applyClientFitFilter(models, minFit) {
  const list = Array.isArray(models) ? models : [];
  if (minFit === 'all') return list;
  if (minFit === 'too_tight') return list.filter(m => m.fit_level === 'too_tight');
  const threshold = fitRank(minFit);
  return list.filter(model => fitRank(model.fit_level) >= threshold);
}

function applyFilters(fits, filters) {
  let result = fits;

  if (filters.search) {
    const s = filters.search.toLowerCase();
    result = result.filter(f =>
      (f.name || '').toLowerCase().includes(s) ||
      (f.provider || '').toLowerCase().includes(s) ||
      (f.parameter_count || '').toLowerCase().includes(s) ||
      (f.use_case || '').toLowerCase().includes(s) ||
      (f.category || '').toLowerCase().includes(s)
    );
  }

  if (filters.runtime && filters.runtime !== 'any') {
    result = result.filter(f => f.runtime === filters.runtime);
  }

  if (filters.useCase && filters.useCase !== 'all') {
    const uc = filters.useCase.toLowerCase();
    result = result.filter(f => (f.use_case || '').toLowerCase() === uc);
  }

  if (filters.provider) {
    const p = filters.provider.toLowerCase();
    result = result.filter(f => (f.provider || '').toLowerCase().includes(p));
  }

  return result;
}
```

- [ ] **Step 2: api.test.js güncelle**

Mevcut `llmfit-web/src/api.test.js` dosyası fetch yerine invoke kullanacak şekilde güncellenir. Testler mock'lanır.

- [ ] **Step 3: Test çalıştır**

Run: `cd llmfit-desktop/frontend && npm run test`
Expected: Temel testler geçer

- [ ] **Step 4: npm run build çalıştır**

Run: `cd llmfit-desktop/frontend && npm run build`
Expected: dist/ güncellenir

- [ ] **Step 5: Commit**

```bash
git add llmfit-desktop/frontend/src/api.js llmfit-desktop/frontend/src/api.test.js
git commit -m "feat(desktop): write Tauri invoke API layer replacing fetch calls"
```

---

## Task 6: Tauri Configuration Güncelleme

**Files:**
- Modify: `llmfit-desktop/tauri.conf.json`

- [ ] **Step 1: tauri.conf.json güncelle**

```json
{
  "productName": "llmfit",
  "version": "0.8.0",
  "identifier": "com.llmfit.desktop",
  "build": {
    "frontendDist": "./frontend/dist",
    "devUrl": "http://localhost:1420",
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build",
    "devtools": true
  },
  "app": {
    "withGlobalTauri": true,
    "windows": [
      {
        "title": "llmfit — LLM Hardware Fitting",
        "width": 1280,
        "height": 840,
        "minWidth": 900,
        "minHeight": 600,
        "resizable": true,
        "center": true
      }
    ],
    "security": {
      "csp": "default-src 'self'; style-src 'self' 'unsafe-inline'; script-src 'self'"
    }
  },
  "bundle": {
    "active": true,
    "targets": "all",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/256x256.png"
    ]
  }
}
```

- [ ] **Step 2: build.rs güncelle (basitleştir)**

Mevcut build.rs sadece `tauri_build::build()` çağırıyor — Tauri 2 otomatik frontend build yapıyor, manual hook gerekmez. Dosya aynı kalabilir.

- [ ] **Step 3: Commit**

```bash
git add llmfit-desktop/tauri.conf.json
git commit -m "feat(desktop): update tauri.conf.json for React frontend integration"
```

---

## Task 7: Eski UI Temizleme

**Files:**
- Delete: `llmfit-desktop/ui/` (tüm dosyalar)

- [ ] **Step 1: Eski UI klasörünü sil**

```bash
rm -rf llmfit-desktop/ui
```

- [ ] **Step 2: tauri.conf.json frontendDist kontrol et**

`frontendDist` zaten `./frontend/dist` olarak ayarlandığı için sorun olmamalı.

- [ ] **Step 3: Commit**

```bash
git rm -r llmfit-desktop/ui
git commit -m "chore(desktop): remove legacy vanilla JS UI (replaced by React)"
```

---

## Task 8: Cargo Workspace Güncelleme

**Files:**
- Modify: `Cargo.toml` (root)

- [ ] **Step 1: default-members'a llmfit-desktop ekle**

```toml
default-members = ["llmfit-core", "llmfit-tui", "llmfit-desktop"]
```

- [ ] **Step 2: Test: cargo check**

Run: `cargo check`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add Cargo.toml
git commit -m "chore: add llmfit-desktop to workspace default-members"
```

---

## Task 9: Full Build ve Test

**Files:**
- None (test only)

- [ ] **Step 1: Cargo build (release)**

Run: `cargo build -p llmfit-desktop --release`
Expected: Build başarılı, target/release/llmfit-desktop.exe oluşur

- [ ] **Step 2: Tauri build**

Run: `cargo tauri build -p llmfit-desktop`
Expected: Tauri bundle oluşur (.exe veya platform'a göre)

- [ ] **Step 3: Hata varsa düzelt ve commit**

Her hata düzeltildiğinde commit at.

---

## Task 10: Final Review ve Cleanup

- [ ] **Step 1: Tüm değişiklikleri gözden geçir**

`git diff --stat` ile dosya listesini kontrol et.

- [ ] **Step 2: Final commit**

```bash
git add -A
git commit -m "feat: llmfit-desktop React GUI integration complete"
```

---

## Bağımlılık Sırası

```
Task 1 (Rust ModelFitInfo) 
    ↓
Task 2 (Rust SystemInfo)
    ↓
Task 3 (Frontend setup)
    ↓
Task 4 (React files copy)
    ↓
Task 5 (api.js)
    ↓
Task 6 (tauri.conf.json)
    ↓
Task 7 (Eski UI sil)
    ↓
Task 8 (Workspace)
    ↓
Task 9 (Full build)
    ↓
Task 10 (Final)
```

## Önemli Notlar

1. **Node.js gerekli:** Windows'ta Node.js + npm kurulu olmalı. npm install çalıştırılacak.
2. **Tauri CLI:** `cargo tauri dev` ve `cargo tauri build` komutları çalışmalı. Gerekirse `cargo install tauri-cli` veya `npm install -g @tauri-apps/cli`.
3. **Windows build:** Tauri 2 Windows'ta build için Visual Studio Build Tools gerekebilir.
4. **Test sırası:** Her task'tan sonra commit at, hata olursa geri dönmek kolaylaşır.
