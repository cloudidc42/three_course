# Part 110: Production Build & Deployment

## บทนำ: จาก Development สู่ Production

การพัฒนา Three.js app ใน development นั้นง่ายและสะดวก แต่เมื่อต้องการ deploy ขึ้น production มีหลายสิ่งที่ต้องคำนึงถึง:

1. **Bundle size** - Three.js เต็ม bundle มีขนาด ~600KB (minified)
2. **Asset optimization** - 3D models, textures ต้องถูก compress
3. **Caching** - static assets ควร cache ได้นาน
4. **CDN delivery** - assets ควร serve จาก CDN ใกล้ผู้ใช้
5. **Error monitoring** - ต้องรู้เมื่อเกิด WebGL error ใน production
6. **Performance monitoring** - ต้องรู้ FPS ของผู้ใช้จริง

---

## 1. Vite Production Build Configuration

```javascript
// vite.config.js - Complete production config
import { defineConfig, loadEnv } from 'vite';
import { resolve } from 'path';

export default defineConfig(({ mode }) => {
    const env = loadEnv(mode, process.cwd(), '');
    const isProd = mode === 'production';
    
    return {
        // Base URL for CDN deployment
        base: env.VITE_CDN_URL || '/',
        
        build: {
            // Output directory
            outDir: 'dist',
            
            // Asset inline threshold (< 4KB = inline as base64)
            assetsInlineLimit: 4096,
            
            // Enable CSS code splitting
            cssCodeSplit: true,
            
            // Source maps for error tracking
            sourcemap: isProd ? 'hidden' : true,
            
            // Minification
            minify: isProd ? 'terser' : false,
            terserOptions: {
                compress: {
                    drop_console: isProd,      // ลบ console.log ใน production
                    drop_debugger: true,
                    pure_funcs: isProd ? ['console.log', 'console.info'] : []
                }
            },
            
            rollupOptions: {
                input: {
                    main: resolve(__dirname, 'index.html'),
                    // Multiple entry points ถ้ามี
                    // viewer: resolve(__dirname, 'viewer.html'),
                },
                
                output: {
                    // Manual chunk splitting strategy
                    manualChunks(id) {
                        // Three.js core → separate chunk
                        if (id.includes('three/build')) {
                            return 'three-core';
                        }
                        
                        // Three.js addons → separate chunk
                        if (id.includes('three/examples')) {
                            return 'three-addons';
                        }
                        
                        // 3D Models (large files) → separate chunk
                        if (id.includes('.glb') || id.includes('.gltf')) {
                            return 'models';
                        }
                        
                        // Heavy vendor libs → separate chunks
                        if (id.includes('node_modules/cannon-es')) {
                            return 'physics';
                        }
                        
                        if (id.includes('node_modules')) {
                            return 'vendor';
                        }
                    },
                    
                    // Output file names with content hash
                    entryFileNames: 'assets/[name]-[hash].js',
                    chunkFileNames: 'assets/[name]-[hash].js',
                    assetFileNames: (assetInfo) => {
                        // Group assets by type
                        const ext = assetInfo.name?.split('.').pop() || '';
                        
                        if (/\.(png|jpg|jpeg|gif|svg|webp)$/.test(assetInfo.name || '')) {
                            return 'assets/textures/[name]-[hash][extname]';
                        }
                        if (/\.(glb|gltf|fbx|obj)$/.test(assetInfo.name || '')) {
                            return 'assets/models/[name]-[hash][extname]';
                        }
                        if (/\.(woff|woff2|ttf|eot)$/.test(assetInfo.name || '')) {
                            return 'assets/fonts/[name]-[hash][extname]';
                        }
                        if (ext === 'css') {
                            return 'assets/styles/[name]-[hash][extname]';
                        }
                        
                        return 'assets/[name]-[hash][extname]';
                    }
                },
                
                // External dependencies (ใช้ CDN แทน)
                // external: ['three'] // ถ้าต้องการใช้ Three.js CDN
            }
        },
        
        // Development server
        server: {
            port: 3000,
            host: true, // Access from network (สำหรับ test บน mobile)
            
            // Headers สำหรับ SharedArrayBuffer (COOP/COEP)
            headers: {
                'Cross-Origin-Opener-Policy': 'same-origin',
                'Cross-Origin-Embedder-Policy': 'require-corp'
            }
        },
        
        // Preview server (สำหรับ test production build)
        preview: {
            port: 4173,
            headers: {
                'Cross-Origin-Opener-Policy': 'same-origin',
                'Cross-Origin-Embedder-Policy': 'require-corp',
                'Cache-Control': 'public, max-age=31536000, immutable'
            }
        },
        
        // Asset handling
        assetsInclude: ['**/*.glb', '**/*.gltf', '**/*.hdr', '**/*.ktx2'],
        
        // Optimization
        optimizeDeps: {
            include: ['three', 'three/addons/controls/OrbitControls.js'],
            // Force optimize these (ป้องกัน dynamic import issues)
        },
        
        // Environment variables
        define: {
            __APP_VERSION__: JSON.stringify(process.env.npm_package_version),
            __BUILD_DATE__: JSON.stringify(new Date().toISOString()),
            __IS_PROD__: isProd
        },
        
        // Plugin: Bundle analysis
        plugins: [
            // visualizer({ filename: 'stats.html', open: true, gzipSize: true })
        ]
    };
});
```

---

## 2. Tree Shaking Three.js

```javascript
// ❌ Import ทั้งหมด - ไม่ tree-shake ได้
import * as THREE from 'three';

// ✅ Import เฉพาะที่ต้องการ - tree-shakeable
import { 
    WebGLRenderer, 
    Scene, 
    PerspectiveCamera,
    BoxGeometry,
    MeshStandardMaterial,
    Mesh,
    DirectionalLight,
    AmbientLight,
    Color,
    Vector3,
    Matrix4
} from 'three';

// Addons - import จาก specific path
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

// ขนาดที่ประมาณได้:
// three (full) = ~600KB minified
// Tree-shaked สำหรับ basic scene = ~180KB
// ประหยัดได้ ~70%!

// ตรวจสอบ bundle size ด้วย:
// npx vite-bundle-visualizer
// หรือ rollup-plugin-visualizer
```

---

## 3. GZIP/Brotli Compression

```javascript
// vite.config.js - เพิ่ม compression plugin
import viteCompression from 'vite-plugin-compression';

export default defineConfig({
    plugins: [
        // Brotli compression (ดีกว่า gzip ~20%)
        viteCompression({
            algorithm: 'brotliCompress',
            ext: '.br',
            threshold: 1024, // compress files > 1KB
            filter: /\.(js|css|html|svg|json|glb)$/
        }),
        
        // Gzip compression (fallback)
        viteCompression({
            algorithm: 'gzip',
            ext: '.gz',
            threshold: 1024
        })
    ]
});
```

```nginx
# Nginx config สำหรับ serve brotli/gzip
server {
    listen 80;
    server_name your-app.com;
    
    root /var/www/html;
    
    # Brotli
    brotli on;
    brotli_comp_level 6;
    brotli_static on;
    brotli_types text/plain text/css application/javascript application/json;
    
    # Gzip fallback
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css application/javascript application/json;
    
    # Cache Control สำหรับ hashed assets (1 year)
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2|glb|gltf)$ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
    
    # COOP/COEP headers สำหรับ SharedArrayBuffer
    add_header Cross-Origin-Opener-Policy "same-origin" always;
    add_header Cross-Origin-Embedder-Policy "require-corp" always;
    
    # CORS สำหรับ 3D models จาก CDN
    add_header Access-Control-Allow-Origin "*";
    add_header Access-Control-Allow-Methods "GET, OPTIONS";
}
```

---

## 4. GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy Three.js App

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  
jobs:
  # ============== Build & Test ==============
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      # Performance Budget Check
      - name: Check bundle size
        run: |
          npm run build
          
          # เช็คขนาด chunks
          THREE_CHUNK=$(find dist/assets -name "three-core*" -type f | head -1)
          THREE_SIZE=$(stat -c%s "$THREE_CHUNK" 2>/dev/null || echo "0")
          MAIN_CHUNK=$(find dist/assets -name "main*" -type f | head -1)
          MAIN_SIZE=$(stat -c%s "$MAIN_CHUNK" 2>/dev/null || echo "0")
          
          echo "Three.js chunk size: $THREE_SIZE bytes"
          echo "Main chunk size: $MAIN_SIZE bytes"
          
          # Fail ถ้า Three.js chunk > 700KB
          MAX_THREE_SIZE=716800  # 700KB
          if [ "$THREE_SIZE" -gt "$MAX_THREE_SIZE" ]; then
            echo "❌ FAILED: Three.js chunk is too large! $THREE_SIZE > $MAX_THREE_SIZE bytes"
            exit 1
          fi
          
          echo "✅ Bundle size check passed"
      
      # Lighthouse Performance Test
      - name: Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          configPath: './lighthouserc.json'
          uploadArtifacts: true
          temporaryPublicStorage: true
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7
  
  # ============== Deploy to Staging ==============
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      
      - name: Deploy to Vercel (Staging)
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          working-directory: ./dist
  
  # ============== Deploy to Production ==============
  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    environment:
      name: production
      url: https://your-app.com
    
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      
      - name: Deploy to Cloudflare Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: three-app
          directory: dist
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Purge Cloudflare Cache
        run: |
          curl -X POST "https://api.cloudflare.com/client/v4/zones/${{ secrets.CF_ZONE_ID }}/purge_cache" \
            -H "Authorization: Bearer ${{ secrets.CLOUDFLARE_API_TOKEN }}" \
            -H "Content-Type: application/json" \
            --data '{"purge_everything":true}'
      
      - name: Create GitHub Release
        if: startsWith(github.ref, 'refs/tags/')
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          draft: false
          prerelease: false

  # ============== Notify ==============
  notify:
    needs: [deploy-production]
    runs-on: ubuntu-latest
    if: always()
    
    steps:
      - name: Slack Notification
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_TITLE: 'Three.js App Deploy'
          SLACK_MESSAGE: '${{ needs.deploy-production.result == "success" && "✅ Deploy สำเร็จ!" || "❌ Deploy ล้มเหลว!" }}'
          SLACK_COLOR: '${{ needs.deploy-production.result == "success" && "good" || "danger" }}'
```

---

## 5. Production Monitoring (Sentry for WebGL Errors)

```javascript
// sentry.js - Error monitoring setup
import * as Sentry from "@sentry/browser";
import * as THREE from 'three';

// Initialize Sentry
Sentry.init({
    dsn: import.meta.env.VITE_SENTRY_DSN,
    environment: import.meta.env.MODE,
    release: import.meta.env.VITE_APP_VERSION,
    
    // Performance monitoring
    tracesSampleRate: import.meta.env.PROD ? 0.1 : 1.0,
    
    // Filter WebGL errors
    beforeSend(event, hint) {
        const error = hint.originalException;
        
        // Capture WebGL context lost errors
        if (error instanceof Error && error.message.includes('WebGL')) {
            event.tags = { ...event.tags, webgl: true };
        }
        
        return event;
    },
    
    // Custom integrations
    integrations: [
        Sentry.browserTracingIntegration()
    ]
});

// Wrap Three.js in error monitoring
class MonitoredRenderer {
    constructor(canvas, options = {}) {
        try {
            this.renderer = new THREE.WebGLRenderer({ canvas, ...options });
            this.isLost = false;
            
            // Monitor WebGL context loss
            canvas.addEventListener('webglcontextlost', (e) => {
                this.isLost = true;
                e.preventDefault();
                
                Sentry.captureException(new Error('WebGL Context Lost'), {
                    level: 'error',
                    extra: {
                        statusMessage: e.statusMessage,
                        gpuVendor: this._getGPUVendor()
                    }
                });
                
                console.error('WebGL context lost!');
                this.onContextLost?.();
            });
            
            canvas.addEventListener('webglcontextrestored', () => {
                this.isLost = false;
                console.log('WebGL context restored');
                this.onContextRestored?.();
            });
            
        } catch (e) {
            Sentry.captureException(e, {
                level: 'fatal',
                tags: { phase: 'init' }
            });
            throw e;
        }
    }
    
    _getGPUVendor() {
        const gl = this.renderer.getContext();
        const ext = gl.getExtension('WEBGL_debug_renderer_info');
        return ext ? gl.getParameter(ext.UNMASKED_RENDERER_WEBGL) : 'unknown';
    }
    
    // Monitored render with performance tracking
    render(scene, camera) {
        if (this.isLost) return;
        
        const span = Sentry.startInactiveSpan({ name: 'three.render', op: 'ui' });
        
        try {
            this.renderer.render(scene, camera);
        } catch (e) {
            Sentry.captureException(e, { tags: { phase: 'render' } });
            throw e;
        } finally {
            span?.end();
        }
    }
}

// Performance metrics reporting
class PerformanceReporter {
    constructor() {
        this.metrics = [];
        this.reportInterval = 60000; // รายงานทุก 1 นาที
        
        setInterval(() => this.report(), this.reportInterval);
    }
    
    record(fps, drawCalls, triangles) {
        this.metrics.push({ fps, drawCalls, triangles, ts: Date.now() });
        if (this.metrics.length > 300) this.metrics.shift(); // keep 5 min
    }
    
    report() {
        if (this.metrics.length === 0) return;
        
        const avgFPS = this.metrics.reduce((sum, m) => sum + m.fps, 0) / this.metrics.length;
        const minFPS = Math.min(...this.metrics.map(m => m.fps));
        
        Sentry.addBreadcrumb({
            category: 'performance',
            message: `Avg FPS: ${avgFPS.toFixed(1)}, Min FPS: ${minFPS}`,
            level: avgFPS < 25 ? 'warning' : 'info',
            data: { avgFPS, minFPS }
        });
        
        // Alert ถ้า FPS ต่ำมาก
        if (avgFPS < 20) {
            Sentry.captureMessage('Low FPS detected', {
                level: 'warning',
                extra: { avgFPS, minFPS, userAgent: navigator.userAgent }
            });
        }
        
        this.metrics = [];
    }
}
```

---

## 6. Complete Deployment Checklist Page

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Production Deployment Checklist</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            background: #050510;
            color: #eee;
            font-family: system-ui, -apple-system, sans-serif;
            max-width: 900px;
            margin: 0 auto;
            padding: 30px 20px;
            min-height: 100vh;
        }
        h1 { color: #0af; font-size: 24px; margin-bottom: 8px; }
        .subtitle { color: #888; margin-bottom: 30px; font-size: 14px; }
        
        .section {
            margin: 20px 0;
            border: 1px solid #224;
            border-radius: 8px;
            overflow: hidden;
        }
        .section-header {
            padding: 12px 16px;
            background: rgba(0,170,255,0.1);
            display: flex;
            align-items: center;
            gap: 10px;
            font-weight: 700;
            font-size: 15px;
        }
        .section-header.done { background: rgba(0,255,0,0.1); }
        .section-header.partial { background: rgba(255,165,0,0.1); }
        
        .check-item {
            display: flex;
            align-items: flex-start;
            gap: 12px;
            padding: 10px 16px;
            border-top: 1px solid #112;
            cursor: pointer;
        }
        .check-item:hover { background: rgba(255,255,255,0.03); }
        .check-item input[type=checkbox] {
            margin-top: 2px;
            accent-color: #0af;
            cursor: pointer;
            width: 16px;
            height: 16px;
            flex-shrink: 0;
        }
        .item-content { flex: 1; }
        .item-title { font-size: 14px; margin-bottom: 3px; }
        .item-detail { font-size: 12px; color: #666; }
        .item-detail code {
            background: rgba(0,170,255,0.1);
            padding: 1px 6px;
            border-radius: 3px;
            color: #8cf;
            font-family: monospace;
        }
        
        .progress-bar {
            height: 4px;
            background: #112;
            margin: 0 16px 16px;
            border-radius: 2px;
        }
        .progress-fill {
            height: 100%;
            border-radius: 2px;
            background: linear-gradient(90deg, #0af, #0f8);
            transition: width 0.3s;
        }
        
        .total-progress {
            position: sticky;
            top: 0;
            background: rgba(5,5,16,0.95);
            padding: 12px 0;
            margin-bottom: 20px;
            border-bottom: 1px solid #224;
            backdrop-filter: blur(10px);
            z-index: 10;
        }
        .total-bar { height: 6px; background: #112; border-radius: 3px; }
        .total-fill { height: 100%; border-radius: 3px; background: linear-gradient(90deg, #0af, #0f8); transition: width 0.3s; }
        .total-label { display: flex; justify-content: space-between; margin-top: 6px; font-size: 13px; color: #888; }
        
        .tag {
            display: inline-block;
            padding: 1px 6px;
            border-radius: 10px;
            font-size: 10px;
            font-weight: 700;
            margin-left: 6px;
        }
        .tag-critical { background: rgba(255,0,0,0.2); color: #f44; }
        .tag-important { background: rgba(255,165,0,0.2); color: #fa0; }
        .tag-optional { background: rgba(100,100,100,0.2); color: #888; }
    </style>
</head>
<body>
    <h1>🚀 Three.js Production Deployment Checklist</h1>
    <p class="subtitle">ตรวจสอบทุกรายการก่อน deploy Three.js app ขึ้น production</p>
    
    <div class="total-progress">
        <div class="total-bar"><div class="total-fill" id="total-fill" style="width:0%"></div></div>
        <div class="total-label">
            <span id="progress-text">0 / 0 รายการ</span>
            <span id="progress-pct">0%</span>
        </div>
    </div>
    
    <div id="checklist"></div>

<script>
const sections = [
    {
        title: '📦 Bundle & Build',
        items: [
            { 
                title: 'Tree-shake Three.js imports', 
                detail: 'ใช้ <code>import { Mesh } from "three"</code> แทน <code>import * as THREE</code>',
                critical: true
            },
            { 
                title: 'กำหนด manual chunks ใน vite.config.js', 
                detail: 'แยก <code>three-core</code>, <code>three-addons</code>, <code>vendor</code> ออกจากกัน',
                critical: true
            },
            { 
                title: 'ตั้ง assetsInlineLimit', 
                detail: 'ตั้งเป็น 4096 bytes เพื่อ inline ไฟล์เล็กเป็น base64',
                important: true
            },
            { 
                title: 'เปิด Terser minification สำหรับ production', 
                detail: 'ลบ <code>console.log</code> และ <code>debugger</code> statements'
            },
            { 
                title: 'ตรวจสอบ bundle size หลัง build', 
                detail: 'Three.js chunk ควรน้อยกว่า 700KB, main chunk น้อยกว่า 200KB',
                critical: true
            },
            { 
                title: 'ตั้ง VITE_APP_VERSION ใน .env.production', 
                detail: 'ใช้สำหรับ Sentry release tracking'
            }
        ]
    },
    {
        title: '🖼️ Assets (3D Models & Textures)',
        items: [
            { 
                title: 'Compress GLB/GLTF ด้วย draco/meshoptimizer', 
                detail: 'ใช้ <code>gltf-transform optimize</code> ลด file size ได้ 60-80%',
                critical: true
            },
            { 
                title: 'Convert textures เป็น KTX2/WebP', 
                detail: 'KTX2 สำหรับ GPU compressed textures, WebP สำหรับ albedo/UI',
                important: true
            },
            { 
                title: 'Generate LOD levels สำหรับ large models', 
                detail: 'สร้าง 3 LOD levels: high/medium/low',
                important: true
            },
            { 
                title: 'Resize textures ตาม max usage', 
                detail: 'ตรวจสอบว่าไม่มี 4K texture ที่แสดงเป็นแค่ 256px'
            },
            { 
                title: 'Pack textures เป็น Atlas เพื่อลด draw calls', 
                detail: 'รวม textures ที่ใช้ด้วยกันเข้า atlas เดียว'
            }
        ]
    },
    {
        title: '⚡ Performance',
        items: [
            { 
                title: 'วัด FPS บน mobile devices จริง', 
                detail: 'test บน iOS Safari และ Android Chrome ดู FPS และ GPU usage',
                critical: true
            },
            { 
                title: 'Implement LOD สำหรับ complex objects', 
                detail: 'ลด polygon count ของ objects ที่อยู่ไกล camera',
                important: true
            },
            { 
                title: 'ปิด Shadows หรือลด shadow map size บน mobile', 
                detail: 'ตรวจสอบ device tier ก่อน enable shadows',
                important: true
            },
            { 
                title: 'ตั้ง maxPixelRatio = min(devicePixelRatio, 2)', 
                detail: 'ป้องกัน retina display render ที่ 3x ซึ่งหนักเกินไป'
            },
            { 
                title: 'ตรวจสอบ draw calls < 200 สำหรับ target scenes', 
                detail: 'ใช้ InstancedMesh และ merged geometry'
            },
            { 
                title: 'Profile และลบ unused shader code', 
                detail: 'ตรวจสอบ MaterialShader ที่ไม่ได้ใช้'
            }
        ]
    },
    {
        title: '🌐 CDN & Caching',
        items: [
            { 
                title: 'Deploy บน CDN (Cloudflare/Vercel/Netlify)', 
                detail: 'Assets ควร serve จาก edge locations',
                critical: true
            },
            { 
                title: 'ตั้ง Cache-Control headers สำหรับ hashed assets', 
                detail: '<code>Cache-Control: public, max-age=31536000, immutable</code>',
                critical: true
            },
            { 
                title: 'ตั้ง CORS headers สำหรับ 3D models', 
                detail: '<code>Access-Control-Allow-Origin: *</code> สำหรับ .glb, .gltf'
            },
            { 
                title: 'Enable Brotli/GZIP compression บน server', 
                detail: 'ลด transfer size ได้ 60-80%',
                important: true
            },
            { 
                title: 'Preload critical 3D assets', 
                detail: '<code>&lt;link rel="preload" as="fetch" href="scene.glb"&gt;</code>'
            }
        ]
    },
    {
        title: '🔒 Security & Headers',
        items: [
            { 
                title: 'ตั้ง Content Security Policy', 
                detail: 'กำหนด source ที่อนุญาตสำหรับ scripts, images, connect-src',
                important: true
            },
            { 
                title: 'ตั้ง COOP/COEP headers (ถ้าใช้ SharedArrayBuffer)', 
                detail: '<code>Cross-Origin-Opener-Policy: same-origin</code>',
                critical: true
            },
            { 
                title: 'ตรวจสอบไม่มี API keys ใน bundle', 
                detail: 'ตรวจสอบ .env.production ไม่ commit ลง git'
            }
        ]
    },
    {
        title: '📊 Monitoring & Error Tracking',
        items: [
            { 
                title: 'Setup Sentry สำหรับ JavaScript errors', 
                detail: 'ตั้ง DSN, release tracking, และ source maps',
                important: true
            },
            { 
                title: 'Track WebGL context loss events', 
                detail: 'เพิ่ม event listener สำหรับ <code>webglcontextlost</code>'
            },
            { 
                title: 'Implement FPS monitoring', 
                detail: 'รายงาน avg/min FPS ไปยัง analytics'
            },
            { 
                title: 'ตั้ง Lighthouse CI ใน GitHub Actions', 
                detail: 'fail build ถ้า performance score < 70',
                important: true
            },
            { 
                title: 'Uptime monitoring สำหรับ health endpoint', 
                detail: 'ตั้ง pingdom/uptime robot ตรวจสอบทุก 5 นาที'
            }
        ]
    },
    {
        title: '♿ Accessibility & UX',
        items: [
            { 
                title: 'ตั้ง meta viewport ที่ถูกต้อง', 
                detail: '<code>width=device-width, initial-scale=1.0</code>'
            },
            { 
                title: 'เพิ่ม fallback สำหรับ no-WebGL users', 
                detail: 'แสดง static image หรือข้อความอธิบายแทน'
            },
            { 
                title: 'รองรับ prefers-reduced-motion', 
                detail: 'ปิด animations เมื่อ user ต้องการ'
            },
            { 
                title: 'เพิ่ม aria-label สำหรับ canvas', 
                detail: '<code>role="application" aria-label="..."</code>'
            },
            { 
                title: 'รองรับ prefers-color-scheme', 
                detail: 'ปรับ scene สำหรับ light/dark mode'
            }
        ]
    }
];

// Render checklist
const container = document.getElementById('checklist');
let totalItems = 0;
let checkedItems = 0;

sections.forEach((section, si) => {
    const sectionEl = document.createElement('div');
    sectionEl.className = 'section';
    
    const header = document.createElement('div');
    header.className = 'section-header';
    header.innerHTML = `<span>${section.title}</span>`;
    
    const progressBar = document.createElement('div');
    progressBar.className = 'progress-bar';
    const fill = document.createElement('div');
    fill.className = 'progress-fill';
    fill.style.width = '0%';
    fill.id = `fill-${si}`;
    progressBar.appendChild(fill);
    
    sectionEl.appendChild(header);
    
    let sectionChecked = 0;
    
    section.items.forEach((item, ii) => {
        totalItems++;
        const id = `check-${si}-${ii}`;
        
        const itemEl = document.createElement('label');
        itemEl.className = 'check-item';
        itemEl.htmlFor = id;
        
        const checkbox = document.createElement('input');
        checkbox.type = 'checkbox';
        checkbox.id = id;
        
        // Load saved state
        const saved = localStorage.getItem(id);
        if (saved === 'true') {
            checkbox.checked = true;
            checkedItems++;
            sectionChecked++;
        }
        
        checkbox.addEventListener('change', () => {
            localStorage.setItem(id, checkbox.checked);
            updateProgress();
        });
        
        const content = document.createElement('div');
        content.className = 'item-content';
        
        const title = document.createElement('div');
        title.className = 'item-title';
        title.innerHTML = item.title;
        if (item.critical) title.innerHTML += '<span class="tag tag-critical">Critical</span>';
        else if (item.important) title.innerHTML += '<span class="tag tag-important">Important</span>';
        else title.innerHTML += '<span class="tag tag-optional">Optional</span>';
        
        const detail = document.createElement('div');
        detail.className = 'item-detail';
        detail.innerHTML = item.detail;
        
        content.appendChild(title);
        content.appendChild(detail);
        itemEl.appendChild(checkbox);
        itemEl.appendChild(content);
        sectionEl.appendChild(itemEl);
    });
    
    sectionEl.appendChild(progressBar);
    container.appendChild(sectionEl);
    
    function updateSectionProgress() {
        let sChecked = 0;
        section.items.forEach((_, ii) => {
            const cb = document.getElementById(`check-${si}-${ii}`);
            if (cb?.checked) sChecked++;
        });
        const pct = section.items.length > 0 ? (sChecked / section.items.length * 100) : 0;
        document.getElementById(`fill-${si}`).style.width = pct + '%';
        header.className = `section-header ${pct === 100 ? 'done' : pct > 0 ? 'partial' : ''}`;
    }
    
    // Observe changes
    sectionEl.addEventListener('change', () => {
        updateProgress();
        updateSectionProgress();
    });
    
    updateSectionProgress();
});

function updateProgress() {
    let total = 0, checked = 0;
    document.querySelectorAll('input[type=checkbox]').forEach(cb => {
        total++;
        if (cb.checked) checked++;
    });
    
    const pct = total > 0 ? Math.round(checked / total * 100) : 0;
    document.getElementById('total-fill').style.width = pct + '%';
    document.getElementById('progress-text').textContent = `${checked} / ${total} รายการ`;
    document.getElementById('progress-pct').textContent = `${pct}%`;
}

updateProgress();
</script>
</body>
</html>
```

---

## 7. Environment Variables Setup

```bash
# .env (Development defaults)
VITE_APP_NAME=My Three.js App
VITE_API_URL=http://localhost:8080

# .env.production (Production overrides)
VITE_CDN_URL=https://cdn.yourdomain.com
VITE_API_URL=https://api.yourdomain.com
VITE_SENTRY_DSN=https://xxx@sentry.io/xxx
VITE_APP_VERSION=$npm_package_version
VITE_ANALYTICS_ID=UA-XXXXX-X

# .env.staging (Staging overrides)
VITE_CDN_URL=https://cdn-staging.yourdomain.com
VITE_API_URL=https://api-staging.yourdomain.com
VITE_SENTRY_DSN=https://xxx@sentry.io/yyy
```

```javascript
// ใช้ env variables ใน app
const APP_NAME = import.meta.env.VITE_APP_NAME;
const API_URL = import.meta.env.VITE_API_URL;
const IS_PROD = import.meta.env.PROD;
const IS_DEV = import.meta.env.DEV;

// ตรวจสอบ required variables
function validateEnv() {
    const required = ['VITE_API_URL'];
    const missing = required.filter(key => !import.meta.env[key]);
    
    if (missing.length > 0) {
        throw new Error(`Missing env variables: ${missing.join(', ')}`);
    }
}

if (IS_DEV) validateEnv();
```

---

## 8. Bundle Analyzer Setup

```javascript
// vite.config.js - เพิ่ม analyzer
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
    plugins: [
        // เรียกใช้: npm run build -- --mode analyze
        process.env.ANALYZE && visualizer({
            filename: 'dist/stats.html',
            open: true,
            gzipSize: true,
            brotliSize: true,
            template: 'treemap' // 'treemap' | 'sunburst' | 'network'
        })
    ].filter(Boolean)
});
```

```json
// package.json scripts
{
    "scripts": {
        "dev": "vite",
        "build": "vite build",
        "preview": "vite preview",
        "analyze": "ANALYZE=true vite build",
        "build:staging": "vite build --mode staging",
        "build:production": "vite build --mode production",
        "lint": "eslint src --ext .js,.ts",
        "size-check": "bundlesize"
    },
    "bundlesize": [
        {
            "path": "dist/assets/three-core-*.js",
            "maxSize": "700kB"
        },
        {
            "path": "dist/assets/main-*.js",
            "maxSize": "200kB"
        }
    ]
}
```

---

## 9. Cloudflare Pages / Vercel Configuration

```toml
# _headers (Cloudflare Pages / Netlify)
/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin

/assets/*
  Cache-Control: public, max-age=31536000, immutable
  
/*.html
  Cache-Control: no-cache, no-store, must-revalidate
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp

/models/*
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET, OPTIONS
  Cache-Control: public, max-age=86400
```

```json
// vercel.json
{
    "headers": [
        {
            "source": "/assets/(.*)",
            "headers": [
                { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
            ]
        },
        {
            "source": "/(.*)",
            "headers": [
                { "key": "Cross-Origin-Opener-Policy", "value": "same-origin" },
                { "key": "Cross-Origin-Embedder-Policy", "value": "require-corp" },
                { "key": "X-Frame-Options", "value": "DENY" },
                { "key": "X-Content-Type-Options", "value": "nosniff" }
            ]
        }
    ],
    "rewrites": [
        { "source": "/(.*)", "destination": "/index.html" }
    ],
    "buildCommand": "npm run build",
    "outputDirectory": "dist",
    "installCommand": "npm ci"
}
```

---

## 10. Health Check Endpoint

```javascript
// health.js - ตรวจสอบ app สถานะ
export function setupHealthCheck(renderer, scene, camera) {
    const ENDPOINT = '/_health';
    
    // สำหรับ server-side rendering หรือ service workers
    // แต่สำหรับ client-side only apps ใช้ window._health แทน
    window._health = {
        status: 'starting',
        version: import.meta.env.VITE_APP_VERSION || '0.0.1',
        startTime: Date.now(),
        renderer: null
    };
    
    // อัปเดตเมื่อ renderer พร้อม
    function setReady() {
        const caps = renderer.capabilities;
        
        window._health = {
            status: 'ok',
            version: import.meta.env.VITE_APP_VERSION || '0.0.1',
            startTime: Date.now(),
            renderer: {
                webgl: caps.isWebGL2 ? 2 : 1,
                maxTextureSize: caps.maxTextureSize,
                precision: caps.precision
            },
            memory: {
                geometries: renderer.info.memory.geometries,
                textures: renderer.info.memory.textures
            }
        };
        
        console.log('Health check ready:', window._health);
    }
    
    // เรียกหลังจาก render ครั้งแรก
    const originalRender = renderer.render.bind(renderer);
    let firstRender = true;
    
    renderer.render = (s, c) => {
        originalRender(s, c);
        if (firstRender) {
            firstRender = false;
            setReady();
        }
    };
}
```

---

## แบบฝึกหัด

1. **ง่าย**: ตั้งค่า Vite project ด้วย tree-shaking Three.js และ manual chunks แล้วเปรียบเทียบ bundle size
2. **กลาง**: สร้าง GitHub Actions workflow ที่ build, check bundle size, และ deploy ไป Vercel
3. **ยาก**: Implement production monitoring ด้วย Sentry ที่ track WebGL errors, FPS metrics, และ context loss events
4. **Challenge**: สร้าง complete production pipeline ที่มี: build optimization, CDN deployment, error monitoring, performance budget enforcement, และ automated lighthouse testing

---

## สรุปบทเรียน Part 101-110

หลังจากผ่านบทเรียนขั้นสูงทั้ง 10 บท คุณได้เรียนรู้:

- **Part 102**: Draw Call Optimization - ลด draw calls ด้วย Instancing, Merging, Atlas, LOD
- **Part 103**: Geometry Merging - เทคนิค merge geometry ขั้นสูงพร้อม multi-material
- **Part 104**: Texture Atlasing - รวม textures ประหยัด GPU bindings
- **Part 105**: Occlusion Culling - ซ่อน objects ที่ไม่ถูกมองเห็น
- **Part 106**: Worker Threads - ย้าย computation ไป background threads
- **Part 107**: Memory Management - ป้องกัน GPU memory leaks
- **Part 108**: Mobile Performance - Adaptive quality สำหรับ mobile
- **Part 109**: Progressive Enhancement - Feature detection และ graceful degradation
- **Part 110**: Production Build - Vite config, CI/CD, monitoring

ทักษะเหล่านี้จะทำให้คุณสร้าง Three.js application ที่:
- **เร็ว** - optimized draw calls และ bundle size
- **เสถียร** - ไม่มี memory leaks
- **รองรับทุก device** - adaptive quality
- **Production-ready** - monitoring และ deployment ครบครัน

---

## อ้างอิง

- [Vite Build Config](https://vitejs.dev/config/build-options.html)
- [Rollup Manual Chunks](https://rollupjs.org/configuration-options/#output-manualchunks)
- [Cloudflare Pages](https://pages.cloudflare.com/)
- [Sentry for JavaScript](https://docs.sentry.io/platforms/javascript/)
- [Web.dev Performance](https://web.dev/performance/)
- [Bundle Size Best Practices](https://web.dev/reduce-javascript-payloads-with-code-splitting/)
