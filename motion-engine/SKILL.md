---
name: "motion-engine"
description: "Служебный скилл: файлы движка для motion-director (рендер, приёмы анимации, звук). Сам не запускается: его использует motion-director."
---

# Motion Engine — файлы движка (MOTION-ENGINE-FILES v3)

Этот скилл не нужно вызывать и читать. Здесь лежат файлы, которые скилл `motion-director` извлекает своей командой в `~/.cache/motion-director/tool/`. Перепечатывать и править их на глаз нельзя: при извлечении каждый файл сверяется с контрольной суммой.

<!-- manifest: {"mo.cjs": "c9cd36ab1d58", "motion.js": "47db10a46d4b", "template.html": "45bf66bdb45c", "faces.py": "5b9b4e6d80d8", "asr.cjs": "2257eb08a019", "copy.md": "a1996d504888", "review.md": "89ac63e0d978", "v": 3} -->

<!-- file: mo.cjs -->
````js
#!/usr/bin/env node
/* mo — движок motion-director: HTML + GSAP → кадры → FFmpeg. Запуск: node mo.cjs <команда> ... */
'use strict';
const fs = require('fs'), path = require('path'), os = require('os'), cp = require('child_process'), crypto = require('crypto');
const CACHE = path.join(os.homedir(), '.cache', 'motion-director'), TOOL = __dirname;
const LIBS = ['gsap', 'CustomEase', 'SplitText', 'DrawSVGPlugin', 'MotionPathPlugin', 'MorphSVGPlugin'];
const out = s => process.stdout.write(s + '\n');
process.stdout.on('error', () => {});
const die = m => { process.stderr.write('ОШИБКА: ' + m + '\n'); process.exit(1); };
const run = (c, a, o) => cp.spawnSync(c, a, Object.assign({ encoding: 'utf8', maxBuffer: 1 << 28 }, o));
const sleep = ms => new Promise(r => setTimeout(r, ms));
const mod = n => path.join(CACHE, 'node_modules', n);
const ex = p => fs.existsSync(p);

function args(argv) {
  const r = { _: [] };
  for (let i = 0; i < argv.length; i++) {
    const a = argv[i];
    if (!a.startsWith('--')) { r._.push(a); continue; }
    const n = argv[i + 1];
    if (n === undefined || n.startsWith('--')) r[a.slice(2)] = true; else { r[a.slice(2)] = n; i++; }
  }
  return r;
}
function npmi(pkgs) {
  fs.mkdirSync(CACHE, { recursive: true });
  const pj = path.join(CACHE, 'package.json');
  if (!ex(pj)) fs.writeFileSync(pj, '{"name":"motion-cache","private":true}');
  const lock = path.join(CACHE, '.lock');             // две сессии не должны ставить пакеты одновременно
  for (let i = 0; i < 900; i++) {
    try { fs.mkdirSync(lock); break; } catch (e) {
      try { if (Date.now() - fs.statSync(lock).mtimeMs > 120000) fs.rmdirSync(lock); } catch (e2) {}
      Atomics.wait(new Int32Array(new SharedArrayBuffer(4)), 0, 0, 200);
    }
  }
  const r = run('npm', ['i', '--no-audit', '--no-fund', '--silent', ...pkgs], { cwd: CACHE });
  try { fs.rmdirSync(lock); } catch (e) {}
  if (r.status !== 0) die('npm не поставил ' + pkgs.join(', ') + ' (нет доступа к реестру или нет такого пакета)\n' + (r.stderr || '').slice(-300));
}
function playwright() {
  for (const p of ['playwright', mod('playwright'), 'playwright-core']) { try { return require(p); } catch (e) {} }
  npmi(['playwright']);
  run('npx', ['playwright', 'install', 'chromium'], { cwd: CACHE, stdio: 'inherit' });
  return require(mod('playwright'));
}
function cfg(dir) {
  const f = path.join(dir, 'index.html');
  if (!ex(f)) die('нет ' + f + ' — сначала mo init');
  const m = fs.readFileSync(f, 'utf8').match(/<[^>]*id=["']stage["'][^>]*>/);
  if (!m) die('в index.html нет элемента id="stage"');
  const g = k => { const x = m[0].match(new RegExp('data-' + k + '=["\']([^"\']+)')); return x && x[1]; };
  const c = { W: +g('width'), H: +g('height'), fps: +(g('fps') || 30), D: +g('duration') };
  if (!(c.W && c.H && c.D)) die('у #stage нужны data-width, data-height, data-duration');
  return c;
}
function probe(file) {
  const r = run('ffprobe', ['-v', 'error', '-print_format', 'json', '-show_streams', '-show_format', file]);
  if (r.status !== 0) die('ffprobe не прочитал ' + file);
  const j = JSON.parse(r.stdout), o = { file, dur: +j.format.duration || 0 };
  const v = j.streams.find(s => s.codec_type === 'video' && !(s.disposition && s.disposition.attached_pic));
  const a = j.streams.find(s => s.codec_type === 'audio');
  o.audio = !!a;
  if (v) {
    let rot = v.tags && v.tags.rotate ? +v.tags.rotate : 0;
    (v.side_data_list || []).forEach(sd => { if (sd.rotation != null) rot = +sd.rotation; });
    const sw = Math.abs(rot) % 180 === 90, fr = (v.avg_frame_rate || '0/1').split('/');
    o.w = sw ? v.height : v.width; o.h = sw ? v.width : v.height;
    o.fps = +(fr[0] / (+fr[1] || 1)).toFixed(3); o.codec = v.codec_name;
    o.hdr = /arib-std-b67|smpte2084/.test(v.color_transfer || '');
  }
  return o;
}

/* ---------- браузер ---------- */
function footage(dir, info) {
  const map = {};
  for (const f of info.footage) {
    const src = path.resolve(dir, f.file);
    if (!ex(src)) die('нет файла нарезки: ' + f.file);
    const p = probe(src);
    if (!p.w) die(f.file + ': нет видеодорожки');
    f.dur = Math.max(0.04, Math.min(f.dur == null ? 1e9 : f.dur, p.dur - f.in, info.D - f.at));
    const sc = Math.min(1, Math.max(f.w * f.over / p.w, f.h * f.over / p.h));
    const ow = Math.max(2, Math.round(p.w * sc / 2) * 2), oh = Math.max(2, Math.round(p.h * sc / 2) * 2), st = fs.statSync(src);
    const key = crypto.createHash('sha1').update([src, st.size, st.mtimeMs, f.in, f.dur, info.fps, ow, oh].join('|')).digest('hex').slice(0, 12);
    const cd = path.resolve(dir, '.cache', key);
    if (!ex(path.join(cd, '.done'))) {
      fs.mkdirSync(cd, { recursive: true });
      const hdr = p.hdr ? 'zscale=t=linear:npl=100,format=gbrpf32le,zscale=p=bt709,tonemap=hable:desat=0,zscale=t=bt709:m=bt709:r=tv,format=yuv420p,' : '';
      const r = run('ffmpeg', ['-y', '-v', 'error', '-ss', String(f.in), '-t', String(f.dur), '-i', src,
        '-vf', hdr + `fps=${info.fps},scale=${ow}:${oh}:flags=lanczos`, '-q:v', '2', path.join(cd, '%06d.jpg')]);
      if (r.status !== 0) die('не удалось разобрать ' + f.file + ' на кадры: ' + (r.stderr || '').slice(-300));
      fs.writeFileSync(path.join(cd, '.done'), '');
    }
    const count = fs.readdirSync(cd).filter(n => n.endsWith('.jpg')).length;
    if (!count) die(f.file + ': на отрезке ' + f.in + '…' + (f.in + f.dur) + ' с нет кадров');
    map[f.i] = { dir: 'file://' + cd, count, at: f.at };
  }
  return map;
}
async function open(dir, o) {
  const c = cfg(dir), { chromium } = playwright();
  const browser = await chromium.launch({ args: ['--font-render-hinting=none', '--disable-lcd-text', '--force-color-profile=srgb', '--allow-file-access-from-files'] });
  const ctx = await browser.newContext({ viewport: { width: c.W, height: c.H }, deviceScaleFactor: 1 });
  const clip = { x: 0, y: 0, width: c.W, height: c.H, scale: o.scale || 1 };
  await ctx.addInitScript(() => { window.__MO = true; });
  const logs = [], pages = [];
  for (let i = 0; i < (o.pages || 1); i++) {
    const page = await ctx.newPage();
    if (!i) {
      page.on('console', m => { if (m.type() === 'error') logs.push(m.text()); });
      page.on('pageerror', e => logs.push('скрипт: ' + e.message));
      page.on('requestfailed', r => { if (r.resourceType() !== 'media' && !/\.cache\//.test(r.url())) logs.push('не загрузилось: ' + r.url()); });
    }
    await page.goto('file://' + path.resolve(dir, 'index.html'));
    try { await page.waitForFunction('window.__moReady === true || !!window.__moError', null, { timeout: 30000 }); }
    catch (e) { await browser.close(); die('страница не дошла до M.ready() за 30 с (нет M.start(...)?). ' + logs.join(' | ')); }
    const err = await page.evaluate('window.__moError || ""');
    if (err) { await browser.close(); die('ошибка в коде анимации: ' + err + ' ' + logs.join(' | ')); }
    const cdp = await ctx.newCDPSession(page);
    if (o.alpha) {
      await cdp.send('Emulation.setDefaultBackgroundColorOverride', { color: { r: 0, g: 0, b: 0, a: 0 } });
      await page.addStyleTag({ content: '[data-bg]{display:none!important} img[data-footage]{visibility:hidden!important}' });
    }
    if (o.guides) await page.evaluate(() => {
      const g = document.createElement('div'), s = window.__motion.safe;
      g.style.cssText = `position:absolute;z-index:99999;top:${s.top}px;right:${s.right}px;bottom:${s.bottom}px;left:${s.left}px;outline:4px dashed #f0f;pointer-events:none`;
      g.setAttribute('data-decor', ''); document.getElementById('stage').appendChild(g);
    });
    pages.push({ page, cdp, clip });
  }
  const info = await pages[0].page.evaluate(() => { const m = window.__motion; return JSON.parse(JSON.stringify(Object.assign({}, m, { stray: m.stray(), grain: m.grain ? m.grain() : 0 }))); });
  const map = o.alpha ? {} : footage(dir, info);
  for (const p of pages) await p.page.evaluate(m => window.__motion.setFootage(m), map);
  return { browser, pages, info, logs, c };
}
async function shot(pg, t, jpeg) {
  await pg.page.evaluate(x => window.renderFrame(x), t);
  const r = await pg.cdp.send('Page.captureScreenshot', Object.assign({ clip: pg.clip, optimizeForSpeed: true }, jpeg ? { format: 'jpeg', quality: jpeg === true ? 90 : jpeg } : { format: 'png' }));
  return Buffer.from(r.data, 'base64');
}
async function grid(browser, items, cols, tw, file, title) {
  const ctx = await browser.newContext({ viewport: { width: cols * (tw + 8) + 8, height: 600 } }), page = await ctx.newPage();
  await page.setContent(`<body style="margin:4px;background:#111;font:13px monospace;color:#ddd">${title ? `<div style="padding:4px">${title}</div>` : ''}` +
    items.map(it => `<figure style="display:inline-block;margin:4px;width:${tw}px;vertical-align:top;background:repeating-conic-gradient(#333 0 25%,#222 0 50%) 0 0/16px 16px"><img style="width:100%;display:block" src="data:image/${it.buf[0] === 0x89 ? 'png' : 'jpeg'};base64,${it.buf.toString('base64')}"><figcaption style="background:#111;padding:2px 0">${it.label}</figcaption></figure>`).join('') + '</body>');
  await page.screenshot({ path: file, fullPage: true, type: 'jpeg', quality: 88 });
  await ctx.close();
}

/* ---------- звук ---------- */
const SR = 48000, KITV = 'v3';
const SYN = ['pop', 'bubble', 'tick', 'thud', 'scratch', 'ding', 'rise', 'drop', 'hit', 'boom', 'whoosh', 'swish', 'sweep', 'riser', 'downer', 'stutter', 'chime', 'ping', 'swell', 'shimmer', 'glitch', 'thock', 'blip'];
const ALIAS = { tap: 'tap_0', key: 'type_0', swipe: 'swipe_0', click: 'button', on: 'toggle_on', off: 'toggle_off', up: 'transition_up', down: 'transition_down', notify: 'notification', done: 'celebration' };
// роль звука задаёт уровень: удары громкие, детали на 12–18 дБ тише (так устроены референсы звукового дизайна)
const LEVEL = { boom: 0, stutter: -2, hit: -2, riser: -4, thud: -6, whoosh: -7, sweep: -7, downer: -8, glitch: -11, swish: -11, pop: -11, swell: -11, drop: -11, rise: -11, thock: -12, bubble: -13, ding: -13, chime: -13, tap: -13, swipe: -13, key: -15, ping: -16, shimmer: -16, blip: -17, scratch: -17, tick: -18 };
const ANCHOR = { hit: 'peak', whoosh: 'peak', swish: 'peak', sweep: 'peak', swell: 'peak', riser: 'end', rise: 'end' };   // какая точка звука встаёт на время события
const BRIGHT = new Set(['tick', 'ping', 'chime', 'shimmer', 'blip', 'ding', 'glitch']), LOWS = new Set(['boom', 'hit', 'thud', 'stutter', 'riser', 'downer']);
function wavWrite(file, ch) {
  const n = ch[0].length, c = ch.length, b = Buffer.alloc(44 + n * c * 2);
  b.write('RIFF', 0); b.writeUInt32LE(36 + n * c * 2, 4); b.write('WAVEfmt ', 8); b.writeUInt32LE(16, 16); b.writeUInt16LE(1, 20); b.writeUInt16LE(c, 22);
  b.writeUInt32LE(SR, 24); b.writeUInt32LE(SR * c * 2, 28); b.writeUInt16LE(c * 2, 32); b.writeUInt16LE(16, 34); b.write('data', 36); b.writeUInt32LE(n * c * 2, 40);
  for (let i = 0; i < n; i++) for (let j = 0; j < c; j++) b.writeInt16LE(Math.max(-32767, Math.min(32767, Math.round(ch[j][i] * 32767))), 44 + (i * c + j) * 2);
  fs.writeFileSync(file, b);
}
function decode(file, a, dur) {                    // любой звук → моно float 48 кГц
  const r = cp.spawnSync('ffmpeg', ['-v', 'error', ...(a ? ['-ss', String(a)] : []), ...(dur ? ['-t', String(dur)] : []), '-i', file, '-vn', '-ac', '1', '-ar', String(SR), '-f', 'f32le', '-'], { maxBuffer: 1 << 30 });
  if (r.status !== 0) die('ffmpeg не прочитал звук из ' + file);
  return new Float32Array(r.stdout.buffer.slice(r.stdout.byteOffset, r.stdout.byteOffset + (r.stdout.length & ~3)));
}
function svf(x, fc, Q, mode) {                     // фильтр с плавающей частотой: fc — число или функция от времени
  let a = 0, b = 0; const k = 1 / Q, f = typeof fc === 'function' ? fc : () => fc;
  for (let i = 0; i < x.length; i++) { const g = Math.tan(Math.PI * Math.min(0.45, f(i / SR) / SR)), v1 = (a + g * (x[i] - b)) / (1 + g * (g + k)), v2 = b + g * v1; a = 2 * v1 - a; b = 2 * v2 - b; x[i] = mode === 'lp' ? v2 : mode === 'hp' ? x[i] - k * v1 - v2 : v1; }
  return x;
}
function synth(name, v) {                          // звуки, собранные по замерам референсов: низ с телом, длинные хвосты, подъезды
  let sd = (v * 9973 + name.length * 131 + 7) >>> 0;
  const R = () => ((sd = sd * 1664525 + 1013904223 >>> 0) / 4294967296) * 2 - 1, mul = [1, 1.12, 0.89, 1.26, 1.06][v % 5], m2 = [1, 1.06, 0.94, 1.1, 0.97][v % 5];
  const tone = (dur, f, a) => { const x = new Float32Array(Math.round(dur * SR)); let ph = 0; for (let i = 0; i < x.length; i++) { const t = i / SR; ph += 2 * Math.PI * f(t) / SR; x[i] = Math.sin(ph) * a(t); } return x; };
  const noise = (dur, a) => { const x = new Float32Array(Math.round(dur * SR)); for (let i = 0; i < x.length; i++) x[i] = R() * a(i / SR); return x; };
  const add = (...xs) => { const s = new Float32Array(Math.max(...xs.map(x => x.length))); xs.forEach(x => { for (let i = 0; i < x.length; i++) s[i] += x[i]; }); return s; };
  const at = (x, t) => { const s = new Float32Array(x.length + Math.round(t * SR)); s.set(x, Math.round(t * SR)); return s; };
  const sat = (x, d) => { const n = Math.tanh(d); for (let i = 0; i < x.length; i++) x[i] = Math.tanh(x[i] * d) / n; return x; };
  const E = (t, tau) => Math.exp(-t / tau), up = (t, a) => Math.min(1, t / a), bell = (t, d) => Math.sin(Math.PI * Math.min(1, t / d)) ** 2;
  const fade = (x, sec) => { const n = Math.min(x.length, Math.round(sec * SR)); for (let i = 0; i < n; i++) x[x.length - 1 - i] *= (i / n) ** 2; return x; };   // хвост уходит плавно, без щелчка
  const wh = (d, f0, f1, pkp, low, air) => {         // подъезд: шум с плавающей полосой, пик ближе к концу
    const pk = d * pkp, env = t => (t < pk ? Math.pow(t / pk, 1.8) : Math.pow(Math.max(0, 1 - (t - pk) / (d - pk)), 2.2));
    return add(svf(noise(d, env), t => (f0 + (f1 - f0) * env(t)) * mul, 1.1, 'bp'), svf(noise(d, t => low * env(t)), t => 90 + 110 * env(t), 0.7, 'lp'), tone(d, t => (70 + 90 * env(t)) * m2, t => low * 0.4 * env(t)), svf(noise(d, t => air * env(t) ** 2), t => 3000 + 3000 * env(t), 0.8, 'bp'));
  };
  switch (name) {
    case 'pop': return tone(0.14, t => (170 + 380 * E(t, 0.018)) * mul, t => up(t, 0.002) * E(t, 0.03));
    case 'bubble': return tone(0.16, t => (260 + 500 * Math.min(1, t / 0.09)) * mul, t => up(t, 0.004) * E(t, 0.045));
    case 'tick': return add(tone(0.03, () => 5800 * m2, t => up(t, 0.0005) * E(t, 0.005)), svf(noise(0.02, t => 0.6 * E(t, 0.003)), 10500, 0.7, 'hp'));
    case 'blip': return add(tone(0.05, () => 2980 * m2, t => up(t, 0.001) * E(t, 0.012)), tone(0.05, () => 1557 * m2, t => 0.3 * E(t, 0.015)));
    case 'thock': return add(tone(0.1, () => 265 * mul, t => up(t, 0.001) * E(t, 0.022)), tone(0.1, () => 157 * mul, t => 0.6 * E(t, 0.03)), tone(0.03, () => 1180 * mul, t => 0.35 * E(t, 0.006)), svf(noise(0.02, t => 0.5 * E(t, 0.003)), 4000, 0.7, 'bp'));
    case 'thud': return add(tone(0.3, t => 50 + 80 * E(t, 0.035), t => up(t, 0.003) * E(t, 0.07)), tone(0.3, t => 100 + 160 * E(t, 0.035), t => 0.3 * E(t, 0.05)), svf(noise(0.03, t => 0.3 * E(t, 0.006)), 300, 1, 'bp'));
    case 'hit': { const d = 0.5, pk = 0.16, env = t => (t < pk ? Math.pow(t / pk, 2.2) : E(t - pk, 0.075)), f = t => (48 + 46 * E(t, 0.11)) * m2;   // мягкий низкий удар с подъездом: пик на 0,16 с
      return sat(add(tone(d, f, env), tone(d, t => f(t) * 2, t => 0.4 * env(t)), tone(d, t => f(t) * 3, t => 0.18 * env(t)), svf(noise(d, t => 0.6 * env(t)), t => 180 + 300 * E(t, 0.1), 0.9, 'bp')), 1.7); }
    case 'boom': { const d = 1.5, f = t => (41 + 70 * E(t, 0.085)) * m2, env = t => up(t, 0.004) * (t < 0.22 ? 1 : E(t - 0.22, 0.3)) * (1 - 0.15 * Math.sin(2 * Math.PI * 5.5 * t) * Math.min(1, t / 0.4));   // большой удар: падение тона и хвост больше секунды
      return fade(sat(add(tone(d, f, env), tone(d, t => f(t) * 2.01, t => 0.32 * env(t) * E(t, 0.25)), tone(d, t => f(t) * 3.02, t => 0.12 * env(t) * E(t, 0.15)), svf(noise(0.5, t => 0.9 * E(t, 0.06)), t => 120 + 500 * E(t, 0.04), 0.8, 'bp'), svf(noise(0.03, t => 0.6 * E(t, 0.005)), 2200, 0.7, 'lp')), 1.8), 0.5); }
    case 'whoosh': return wh(0.55, 220, 950, 0.62, 0.8, 0.1);
    case 'swish': return wh(0.26, 500, 2400, 0.55, 0.25, 0.25);
    case 'sweep': return wh(0.95, 160, 700, 0.66, 0.9, 0.06);
    case 'riser': { const d = 1.0, g = t => (Math.exp(4.2 * t / d) - 1) / (Math.exp(4.2) - 1), cut = t => Math.min(1, (d - t) / 0.012);   // нарастание с обрывом: конец звука встаёт на удар
      return sat(add(svf(noise(d, t => g(t) * cut(t)), t => 300 * Math.pow(14, t / d) * mul, 1.6, 'bp'), tone(d, t => 110 * Math.pow(4, t / d) * m2, t => 0.4 * g(t) * cut(t)), tone(d, t => 111.5 * Math.pow(4.05, t / d) * m2, t => 0.3 * g(t) * cut(t)), tone(d, t => 38 + 34 * t / d, t => 0.7 * g(t) * cut(t))), 1.3); }
    case 'downer': { const d = 0.9, g = t => E(t, 0.22) * up(t, 0.004);
      return fade(add(svf(noise(d, t => 0.8 * g(t)), t => 250 * Math.pow(16, 1 - t / d) * mul, 1.4, 'bp'), tone(d, t => 70 * Math.pow(6, 1 - t / d) * m2, t => 0.5 * g(t)), tone(d, t => 36 + 30 * (1 - t / d), t => 0.6 * g(t))), 0.3); }
    case 'stutter': { const d = 0.85, rate = 16 * m2, x = new Float32Array(Math.round(d * SR)); let ph = 0;   // дроблёный бас, сползающий вниз
      for (let i = 0; i < x.length; i++) { const t = i / SR, u = (t * rate) % 1, gate = u < 0.72 ? Math.pow(Math.sin(Math.PI * u / 0.72), 0.6) : 0, f = (46 + 150 * E(t, 0.3)) * (1 + 0.5 * (1 - u)); ph += 2 * Math.PI * f / SR; let s = 0; for (let h = 1; h <= 9; h++) s += Math.sin(ph * h) / h; x[i] = s * gate * Math.min(1, (d - t) / 0.2) * up(t, 0.005); }
      return sat(svf(x, t => 400 + 3600 * E(t, 0.28), 0.8, 'lp'), 2); }
    case 'chime': { const one = () => add(...[[1479, 1, 0.34], [1865, 0.9, 0.34], [2958, 0.22, 0.2], [3730, 0.16, 0.18], [9941, 0.1, 0.12], [10740, 0.07, 0.1]].map(p => tone(1.4, () => p[0] * mul, t => p[1] * up(t, 0.002) * E(t, p[2]))));
      const s2 = one(); for (let i = 0; i < s2.length; i++) s2[i] *= 0.8; return fade(add(one(), at(s2, 0.07)), 0.5); }
    case 'ping': return fade(add(tone(1.0, () => 1760 * mul, t => 0.6 * up(t, 0.002) * E(t, 0.26)), tone(1.0, () => 3520 * mul, t => 0.8 * up(t, 0.002) * E(t, 0.2))), 0.4);
    case 'swell': { const d = 0.75, env = t => (t < 0.3 ? (t / 0.3) ** 2 : E(t - 0.3, 0.09));
      return add(tone(d, () => 554 * mul, env), tone(d, () => 831 * mul, t => 0.3 * env(t)), tone(d, () => 1108 * mul, t => 0.09 * env(t)), tone(d, () => 277 * mul, t => 0.25 * env(t)), svf(noise(d, t => 0.2 * env(t)), 600, 1, 'bp')); }
    case 'shimmer': { const d = 0.6; return add(...[1500, 1602, 1716, 3473, 3682, 4145, 4545, 4905].map((f, i) => tone(d, t => f * mul * (1 + 0.5 * t / d), t => bell(t, d) * (0.6 + 0.4 * Math.sin(t * (38 + i * 7))) / (1 + i * 0.25)))); }
    case 'glitch': { const d = 0.28, x = new Float32Array(Math.round(d * SR)); let i = 0;
      while (i < x.length) { const n = Math.round((0.012 + 0.03 * (R() + 1) / 2) * SR), kind = (R() + 1) * 1.5 | 0, f = 300 + 1500 * (R() + 1) / 2, hold = 4 + ((R() + 1) * 3 | 0); let h = 0;
        for (let j = 0; j < n && i < x.length; j++, i++) { if (kind === 0) { if (j % hold === 0) h = R(); x[i] = h * 0.8; } else if (kind === 1) x[i] = Math.sin(2 * Math.PI * f * j / SR) > 0 ? 0.5 : -0.5; } }
      for (let j = 0; j < x.length; j++) x[j] *= Math.min(1, (x.length - j) / 2400); return svf(x, 5000, 0.7, 'lp'); }
    case 'scratch': { const d = 0.26 + 0.04 * (v % 3); return svf(noise(d, t => bell(t, d) * (0.6 + 0.4 * Math.sin(t * (150 + 30 * v)))), 2400 * mul, 1.2, 'bp'); }
    case 'ding': return fade(add(tone(0.9, () => 587 * mul, t => up(t, 0.002) * E(t, 0.28)), tone(0.5, () => 1620 * mul, t => 0.3 * E(t, 0.1)), tone(0.3, () => 3170 * mul, t => 0.12 * E(t, 0.04))), 0.3);
    case 'rise': return add(tone(0.6, t => 180 + 420 * (t / 0.6) ** 2, t => 0.6 * (t / 0.6) ** 2 * Math.min(1, (0.6 - t) / 0.02)), svf(noise(0.6, t => 0.5 * (t / 0.6) ** 2 * Math.min(1, (0.6 - t) / 0.02)), 1800, 0.8, 'bp'));
    case 'drop': return tone(0.32, t => 90 + 380 * E(t, 0.07), t => up(t, 0.004) * E(t, 0.1));
  }
  return null;
}
function kit(id) {                                 // набор звуков: записанные (snd-lib) + синтез; у каждого посчитаны пик и длина
  id = /^0[123]$/.test(id) ? id : '01';
  const kd = path.join(CACHE, 'sfx', KITV + '-' + id);
  if (ex(path.join(kd, 'meta.json'))) return kd;
  if (!ex(mod('snd-lib'))) npmi(['snd-lib']);
  fs.mkdirSync(kd, { recursive: true });
  const sd = path.join(mod('snd-lib'), 'assets', 'sounds', 'sprite', id), map = JSON.parse(fs.readFileSync(path.join(sd, 'audioSprite.json'), 'utf8')).spritemap, all = decode(path.join(sd, 'audioSprite.mp3')), meta = {};
  const norm = x => {
    let pk = 0; for (let i = 0; i < x.length; i++) pk = Math.max(pk, Math.abs(x[i]));
    if (pk < 1e-4) return null;
    let a = 0, e = x.length - 1; while (a < e && Math.abs(x[a]) < pk * 0.003) a++; while (e > a && Math.abs(x[e]) < pk * 0.004) e--;
    const y = x.slice(Math.max(0, a - 48), Math.min(x.length, e + 1200)), w = 1440; let best = 0, acc = 0, bi = 0;
    for (let i = 0; i < y.length; i++) { acc += y[i] * y[i]; if (i >= w) acc -= y[i - w] * y[i - w]; if (acc / w > best) { best = acc / w; bi = i; } }
    const g = Math.min(0.1 / Math.sqrt(best + 1e-12), 0.6 / pk), f = Math.min(y.length, 480);
    for (let i = 0; i < y.length; i++) y[i] *= g * Math.min(1, (y.length - i) / f);
    y.peak = Math.max(0, bi - w / 2) / SR;
    return y;
  };
  const save = (n, x) => { if (!x) return; wavWrite(path.join(kd, n + '.wav'), [x]); meta[n] = { peak: +x.peak.toFixed(3), dur: +(x.length / SR).toFixed(3) }; };
  const names = Object.keys(map).sort((p, q) => map[p].start - map[q].start);
  names.forEach((n, i) => save(n, norm(all.slice(Math.round(map[n].start * SR), Math.min(all.length, Math.round((names[i + 1] ? map[names[i + 1]].start : all.length / SR) * SR))))));
  SYN.forEach(n => { for (let v = 0; v < 5; v++) save(n + '_0' + (v + 1), norm(synth(n, v))); });
  fs.writeFileSync(path.join(kd, 'meta.json'), JSON.stringify(meta));
  return kd;
}
function sfxFile(kd, name, n) {
  const b = ALIAS[name] || name, v = (n % 5) + 1;
  for (const c of [b + v + '.wav', b + '_0' + v + '.wav', b + '.wav']) if (ex(path.join(kd, c))) return path.join(kd, c);
  return null;
}
function fft(re, im, inv) {
  const n = re.length;
  for (let i = 1, j = 0; i < n; i++) { let b = n >> 1; for (; j & b; b >>= 1) j ^= b; j ^= b; if (i < j) { let t = re[i]; re[i] = re[j]; re[j] = t; t = im[i]; im[i] = im[j]; im[j] = t; } }
  for (let len = 2; len <= n; len <<= 1) { const ang = 2 * Math.PI / len * (inv ? -1 : 1), wr = Math.cos(ang), wi = Math.sin(ang);
    for (let i = 0; i < n; i += len) { let cr = 1, ci = 0; for (let k = 0; k < len / 2; k++) { const a = i + k, b = a + len / 2, xr = re[b] * cr - im[b] * ci, xi = re[b] * ci + im[b] * cr; re[b] = re[a] - xr; im[b] = im[a] - xi; re[a] += xr; im[a] += xi; const t = cr * wr - ci * wi; ci = cr * wi + ci * wr; cr = t; } } }
  if (inv) for (let i = 0; i < n; i++) { re[i] /= n; im[i] /= n; }
}
function room(L, Rr, wetDb) {                      // общий «воздух»: короткая комната на все эффекты, без низа — склеивает звуки и даёт хвосты
  const N = L.length, hl = Math.round(0.9 * SR); let n = 1; while (n < N + hl) n <<= 1;
  [L, Rr].forEach((x, c) => {
    let sd = 911 + c * 7717; const rnd = () => ((sd = sd * 1664525 + 1013904223 >>> 0) / 4294967296) * 2 - 1, h = new Float32Array(hl), h2 = new Float32Array(hl), pre = Math.round(0.012 * SR);
    for (let i = pre; i < hl; i++) { const t = (i - pre) / SR; h[i] = rnd() * Math.exp(-t / 0.15); h2[i] = rnd() * Math.exp(-t / 0.08) * 0.6; }
    svf(h, 900, 0.5, 'bp'); svf(h2, 3600, 0.6, 'bp');
    const ar = new Float64Array(n), ai = new Float64Array(n), br = new Float64Array(n), bi = new Float64Array(n), send = svf(Float32Array.from(x), 220, 0.7, 'hp');
    ar.set(send); for (let i = 0; i < hl; i++) br[i] = h[i] + h2[i];
    fft(ar, ai); fft(br, bi);
    for (let i = 0; i < n; i++) { const r = ar[i] * br[i] - ai[i] * bi[i]; ai[i] = ar[i] * bi[i] + ai[i] * br[i]; ar[i] = r; }
    fft(ar, ai, true);
    let ed = 0, ew = 0; for (let i = 0; i < N; i++) { ed += send[i] * send[i]; ew += ar[i] * ar[i]; }
    const g = Math.pow(10, wetDb / 20) * Math.sqrt(ed / (ew + 1e-12));
    for (let i = 0; i < N; i++) x[i] += ar[i] * g;
  });
}
function sfxBus(dir, info, a, b, voiced) {         // все M.sfx(...) → одна стереодорожка: якоря, уровни по ролям, общий «воздух»
  const evs = (info.sfx || []).filter(e => e.at >= a - 4 && e.at < b + 2).sort((x, y) => x.at - y.at);
  if (!evs.length) return null;
  const kd = kit(info.kit), meta = JSON.parse(fs.readFileSync(path.join(kd, 'meta.json'), 'utf8')), N = Math.round((b - a) * SR), L = new Float32Array(N), Rr = new Float32Array(N), cache = {}, cnt = {}, miss = new Set(), seen = new Set();
  const rd = f => { const bf = fs.readFileSync(f), x = new Float32Array((bf.length - 44) >> 1); for (let i = 0; i < x.length; i++) x[i] = bf.readInt16LE(44 + i * 2) / 32768; return x; };
  let used = 0;
  for (const e of evs) {
    const n = cnt[e.name] = (cnt[e.name] || 0) + 1, own = /\.(wav|mp3|m4a|ogg|aac|flac)$/i.test(e.name), f = own ? path.resolve(dir, e.name) : sfxFile(kd, e.name, n - 1);
    if (!f || !ex(f)) { miss.add(e.name); continue; }
    const x = cache[f] || (cache[f] = own ? decode(f) : rd(f)), h = (Math.imul(n, 2654435761) >>> 0) / 4294967296, m = meta[path.basename(f, '.wav')] || { peak: 0, dur: x.length / SR };
    const rate = e.rate || 1 + (h - 0.5) * 0.07, anchor = e.anchor || (own ? 'start' : ANCHOR[e.name]) || 'start', off = anchor === 'peak' ? m.peak : anchor === 'end' ? m.dur : 0, t0 = e.at - off / rate;
    const key = e.name + '|' + Math.round(t0 * 33); if (seen.has(key)) continue; seen.add(key);   // два одинаковых звука в один кадр сливаются в один
    const lv = (own ? 0 : LEVEL[e.name] == null ? -12 : LEVEL[e.name]) + (e.gain || 0) + ((h * 7) % 1 - 0.5) * 1.5 + (voiced && BRIGHT.has(e.name) ? -5 : 0);
    const g = Math.pow(10, lv / 20), pan = LOWS.has(e.name) ? 0 : e.pan == null ? ((h * 13) % 1 - 0.5) * 1.1 : e.pan;   // низ по центру, детали разложены по сторонам
    const gl = g * Math.cos((pan + 1) * Math.PI / 4) * 1.414, gr = g * Math.sin((pan + 1) * Math.PI / 4) * 1.414, o0 = Math.round((t0 - a) * SR), len = Math.floor(x.length / rate);
    for (let i = 0; i < len; i++) { const j = o0 + i; if (j < 0) continue; if (j >= N) break; const p = i * rate, k0 = p | 0, v = x[k0] + ((x[k0 + 1] || 0) - x[k0]) * (p - k0); L[j] += v * gl; Rr[j] += v * gr; }
    used++;
  }
  room(L, Rr, voiced ? -18 : -12);
  let pk = 0; for (let i = 0; i < N; i++) pk = Math.max(pk, Math.abs(L[i]), Math.abs(Rr[i]));
  if (pk > 0.97) { const g = 0.97 / pk; for (let i = 0; i < N; i++) { L[i] *= g; Rr[i] *= g; } }
  const file = path.join(dir, 'out', '.sfx.wav');
  wavWrite(file, [L, Rr]);
  return { file, n: used, miss: [...miss] };
}
function loud(file, af) {                          // громкость по EBU R128: интегральная, диапазон, пик, максимум momentary
  const e = run('ffmpeg', ['-nostats', '-v', 'info', '-i', file, '-vn', '-af', (af ? af + ',' : '') + 'ebur128=peak=true', '-f', 'null', '-']).stderr || '';
  const sm = e.slice(e.lastIndexOf('Summary:')), g = re => { const m = sm.match(re); return m ? +m[1] : NaN; };
  let mx = -99; for (const m of e.matchAll(/\sM:\s*(-?[\d.]+)/g)) mx = Math.max(mx, +m[1]);
  return { I: g(/I:\s+(-?[\d.]+) LUFS/), LRA: g(/LRA:\s+([\d.]+) LU/), TP: g(/Peak:\s+(-?[\d.]+) dBFS/), M: mx };
}
function bands(file) {                             // баланс звука по полосам, дБ: низ до 120 Гц, тело 120–500, середина 500–4000, верх от 4000
  return ['lowpass=f=120,lowpass=f=120', 'highpass=f=120,lowpass=f=500', 'highpass=f=500,lowpass=f=4000', 'highpass=f=4000,highpass=f=4000'].map(af => {
    const e = run('ffmpeg', ['-nostats', '-v', 'info', '-i', file, '-vn', '-af', af + ',astats=measure_perchannel=none:measure_overall=RMS_level', '-f', 'null', '-']).stderr || '', m = e.match(/RMS level dB:\s*(-?[\d.]+|-inf)/);
    return m && m[1] !== '-inf' ? +(+m[1]).toFixed(1) : -90; });
}
function audio(dir, info, a, b) {                  // шины: голос, музыка, прочее, эффекты → уровни по замеру → общий микс
  const od = path.join(dir, 'out'), len = (b - a).toFixed(3), tr = [];
  fs.mkdirSync(od, { recursive: true });
  for (const t of [...info.footage.filter(f => f.audio), ...info.audio]) {
    const src = path.resolve(dir, t.file);
    if (!ex(src)) die('нет аудиофайла: ' + t.file);
    const p = probe(src);
    if (!p.audio) { out('! в ' + t.file + ' нет звука — пропущен'); continue; }
    let at = t.at - a, inn = t.in, dur = Math.min(t.dur == null ? 1e9 : t.dur, p.dur - t.in, info.D - t.at);
    if (at < 0) { inn -= at; dur += at; at = 0; }
    dur = Math.min(dur, b - a - at);
    if (dur > 0.02) tr.push({ src, at, in: inn, dur, gain: t.gain || 0, fi: t.fadeIn || 0, fo: t.fadeOut || 0, duck: !!t.duck, kind: t.voice ? 'voice' : t.i != null ? 'other' : 'music' });
  }
  const bus = kind => {
    const ts = tr.filter(t => t.kind === kind);
    if (!ts.length) return null;
    const file = path.join(od, '.' + kind + '.wav'), inp = [], f = [];
    ts.forEach((t, i) => { inp.push('-i', t.src);
      f.push(`[${i}:a]atrim=start=${t.in.toFixed(4)}:duration=${t.dur.toFixed(4)},asetpts=PTS-STARTPTS,aresample=${SR},aformat=sample_fmts=fltp:channel_layouts=stereo,volume=${t.gain}dB` +
        (t.fi ? `,afade=t=in:st=0:d=${t.fi}` : '') + (t.fo ? `,afade=t=out:st=${Math.max(0, t.dur - t.fo).toFixed(3)}:d=${t.fo}` : '') + `,adelay=${Math.round(t.at * 1000)}:all=1[t${i}]`); });
    f.push(ts.map((t, i) => `[t${i}]`).join('') + (ts.length > 1 ? `amix=inputs=${ts.length}:normalize=0:duration=longest,` : '') + `apad,atrim=0:${len}[o]`);
    const r = run('ffmpeg', ['-y', '-v', 'error', ...inp, '-filter_complex', f.join(';'), '-map', '[o]', '-c:a', 'pcm_s16le', file]);
    if (r.status !== 0) die('не удалось собрать дорожку «' + kind + '»: ' + (r.stderr || '').slice(-300));
    return file;
  };
  const V = bus('voice'), Mu = bus('music'), O = bus('other'), Sx = sfxBus(dir, info, a, b, !!V);
  if (!V && !Mu && !O && !Sx) return null;
  const lv = V ? loud(V) : null, vI = lv && isFinite(lv.I) && lv.I > -50 ? lv.I : null, parts = [], rep = [], fx = n => (n > 0 ? '+' : '') + n.toFixed(1);
  let mI = null;
  if (V) { parts.push([V, 0, '']); rep.push('голос ' + (vI == null ? 'очень тихий' : vI.toFixed(1) + ' LUFS')); }
  if (O) parts.push([O, 0, '']);
  if (Mu) { const lm = loud(Mu); let g = 0, af = '';
    if (vI != null && isFinite(lm.I)) { g = vI - info.under - lm.I; af = 'equalizer=f=1000:t=q:w=1:g=-4,equalizer=f=2800:t=q:w=1.2:g=-6'; rep.push(`музыка на ${info.under} LU ниже голоса (${(lm.I + g).toFixed(1)} LUFS, сдвиг ${fx(g)} дБ, середина прибрана)`); }
    else rep.push('музыка ' + (isFinite(lm.I) ? lm.I.toFixed(1) + ' LUFS' : 'без замера'));
    mI = isFinite(lm.I) ? lm.I + g : null; parts.push([Mu, g, af, tr.some(t => t.kind === 'music' && t.duck) && V]); }
  if (Sx) { const ls = loud(Sx.file), ref = vI != null ? vI - 7 : mI != null ? mI + 6 : -12, g = ref - ls.M + (info.sfxGain || 0), bd = bands(Sx.file);
    parts.push([Sx.file, g, vI != null ? 'lowpass=f=6500' : '']);
    rep.push(`эффекты: ${Sx.n} шт., громкий момент ${(ls.M + g).toFixed(1)} LUFS` + (vI != null ? ` (на ${(vI - ls.M - g).toFixed(1)} ниже голоса)` : '') + `, баланс низ/тело/середина/верх ${bd.map(v => Math.round(v - Math.max(...bd))).join('/')} дБ`);
    if (Sx.miss.length) out('! нет таких звуков в наборе: ' + Sx.miss.join(', ') + ' — список: mo sfx'); }
  const inp = [], f = [], lab = [];
  parts.forEach((p, i) => { inp.push('-i', p[0]); f.push(`[${i}:a]aformat=sample_fmts=fltp:channel_layouts=stereo,volume=${p[1].toFixed(2)}dB${p[2] ? ',' + p[2] : ''}[p${i}]`); lab.push(`[p${i}]`); });
  const di = parts.findIndex(p => p[3]);
  if (di >= 0) { f.push(`[p0]asplit=2[pv][sc]`, `[p${di}][sc]sidechaincompress=threshold=0.03:ratio=6:attack=20:release=400[pd]`); lab[0] = '[pv]'; lab[di] = '[pd]'; }
  f.push(lab.join('') + (lab.length > 1 ? `amix=inputs=${lab.length}:normalize=0:duration=longest,` : '') + `apad,atrim=0:${len}[m]`);
  const wav = path.join(od, '.mix.wav'), base = ['-y', '-v', 'info', ...inp], LU = info.loudness, only = !V && !Mu && !O;   // only: в ролике одни эффекты
  let ln = 'alimiter=limit=0.89:level=false';
  if (only && typeof LU === 'number') {             // одни эффекты: сохраняем динамику ударов, уровень ставим по пику, а не по средней громкости
    const tmp = path.join(od, '.pre.wav'), r0 = run('ffmpeg', [...base, '-filter_complex', f.join(';'), '-map', '[m]', '-c:a', 'pcm_f32le', tmp]);
    if (r0.status !== 0) die('не удалось свести звук: ' + (r0.stderr || '').slice(-400));
    const l0 = loud(tmp), g = Math.min(isFinite(l0.TP) ? -1.5 - l0.TP : 0, isFinite(l0.I) ? LU + 1 - l0.I : 99);
    ln = `volume=${g.toFixed(2)}dB,alimiter=limit=0.84:level=false`;
  } else if (typeof LU === 'number' && isFinite(LU)) {   // data-loudness="off" — дорожки сводятся как есть; иначе два прохода: измерить, затем линейно привести к цели
    const r = run('ffmpeg', [...base, '-filter_complex', f.join(';') + `;[m]loudnorm=I=${LU}:TP=-2:LRA=11:print_format=json[o]`, '-map', '[o]', '-f', 'null', '-']);
    const j = (r.stderr.match(/\{[^{}]*"input_i"[^{}]*\}/) || [])[0];
    if (r.status !== 0 || !j) die('не удалось свести звук: ' + (r.stderr || '').slice(-400));
    const m = JSON.parse(j);
    ln = isFinite(+m.input_i) ? `loudnorm=I=${LU}:TP=-2:LRA=11:measured_I=${m.input_i}:measured_TP=${m.input_tp}:measured_LRA=${m.input_lra}:measured_thresh=${m.input_thresh}:offset=${m.target_offset}:linear=true` : ln;
  }
  const r2 = run('ffmpeg', [...base, '-filter_complex', f.join(';') + `;[m]${ln},aresample=${SR}[o]`, '-map', '[o]', '-c:a', 'pcm_s16le', wav]);
  if (r2.status !== 0) die('не удалось свести звук: ' + (r2.stderr || '').slice(-400));
  const lf = loud(wav);
  rep.push(`итог ${lf.I.toFixed(1)} LUFS, пик ${lf.TP.toFixed(1)} dBFS до сжатия`);
  return { wav, n: tr.length + (Sx ? 1 : 0), report: rep.join(' · ') };
}

/* ---------- команды ---------- */
function init(dir, o) {
  if (ex(path.join(dir, 'index.html')) && !o.force) die(dir + '/index.html уже есть (--force чтобы перезаписать)');
  const [W, H] = String(o.size || '1080x1920').toLowerCase().split('x').map(Number);
  if (!(W && H)) die('--size в виде 1080x1920');
  const safe = o.safe || (H > W * 1.5 ? 'social' : W > H ? 'broadcast' : 'feed');
  ['media', 'fonts', 'out', 'lib'].forEach(d => fs.mkdirSync(path.join(dir, d), { recursive: true }));
  if (!ex(mod('gsap'))) npmi(['gsap']);
  LIBS.forEach(n => fs.copyFileSync(path.join(mod('gsap'), 'dist', n + '.min.js'), path.join(dir, 'lib', n + '.min.js')));
  fs.copyFileSync(path.join(TOOL, 'motion.js'), path.join(dir, 'motion.js'));
  const fc = path.join(dir, 'fonts', 'fonts.css');
  if (!ex(fc)) fs.writeFileSync(fc, '');
  if (!ex(path.join(dir, 'data.js'))) fs.writeFileSync(path.join(dir, 'data.js'), '/* данные проекта (иконки, слова речи, лица) — дописывает mo data */\n');
  fs.writeFileSync(path.join(dir, 'index.html'), fs.readFileSync(path.join(TOOL, 'template.html'), 'utf8')
    .replace(/\{\{W\}\}/g, W).replace(/\{\{H\}\}/g, H).replace(/\{\{FPS\}\}/g, o.fps || 30).replace(/\{\{DUR\}\}/g, o.dur || 10).replace(/\{\{SAFE\}\}/g, safe));
  out(`Проект ${dir}: ${W}×${H}, ${o.fps || 30} fps, ${o.dur || 10} с, безопасная зона «${safe}». Правь index.html; файлы пользователя клади в media/.`);
}
function fonts(dir, specs, o) {
  const fd = path.join(dir, 'fonts'), fc = path.join(fd, 'fonts.css');
  fs.mkdirSync(fd, { recursive: true });
  let css = ex(fc) ? fs.readFileSync(fc, 'utf8') : '';
  for (const sp of specs) {
    const [fam, ws] = sp.split(':'), pkg = '@fontsource/' + fam.trim().toLowerCase().replace(/[\s_]+/g, '-');
    if (!ex(mod(pkg))) npmi([pkg]);
    let name = fam;
    for (const w of (ws || '400,700').split(',')) {
      const f = path.join(mod(pkg), w.trim() + '.css');
      if (!ex(f)) die(`у ${fam} нет начертания ${w}. Есть: ` + fs.readdirSync(mod(pkg)).filter(n => /^\d+\.css$/.test(n)).map(n => n.replace('.css', '')).join(', '));
      const blocks = fs.readFileSync(f, 'utf8').match(/\/\*[^*]*\*\/\s*@font-face\s*\{[^}]*\}/g) || [];
      const keep = blocks.filter(b => /-(cyrillic|cyrillic-ext|latin|latin-ext)-\d+-normal \*\//.test(b));
      if (!keep.some(b => /-cyrillic-\d/.test(b)) && !o.latin) die(`в ${fam} ${w} нет кириллицы — русский текст уйдёт в запасной шрифт. Возьми другой шрифт (Inter, Montserrat, Manrope, Unbounded, Onest, Golos Text, Rubik, Geologica, Oswald) или добавь --latin, если текст только латиницей.`);
      for (let b of keep) {
        const id = b.match(/\/\*\s*([^*]+?)\s*\*\//)[1];
        name = (b.match(/font-family:\s*['"]([^'"]+)/) || [])[1] || name;
        if (css.includes('/* ' + id + ' */')) continue;
        b = b.replace(/,\s*url\([^)]*\.woff\)\s*format\(['"]woff['"]\)/, '').replace(/url\(\.\/files\/([^)]+)\)/, (m, n) => { fs.copyFileSync(path.join(mod(pkg), 'files', n), path.join(fd, n)); return 'url(./' + n + ')'; });
        css += b + '\n';
      }
    }
    out(`Шрифт готов: font-family: "${name}" (${ws || '400,700'})`);
  }
  fs.writeFileSync(fc, css);
}
async function stills(dir, o) {
  const c = cfg(dir), times = String(o._[2] || '0').split(',').map(Number);
  const S = await open(dir, { scale: o.full ? 1 : Math.min(1, 1280 / Math.max(c.W, c.H)), guides: o.guides });
  const od = path.join(dir, 'out', 'stills');
  fs.mkdirSync(od, { recursive: true });
  for (const t of times) {
    const f = path.join(od, 't_' + t.toFixed(2).padStart(6, '0') + '.png');
    fs.writeFileSync(f, await shot(S.pages[0], Math.round(t * c.fps) / c.fps, false));
    out(f);
  }
  if (S.logs.length) out('! ' + S.logs.join('\n! '));
  await S.browser.close();
}
async function sheet(dir, o) {
  const c = cfg(dir), a = +(o.from || 0), b = Math.min(c.D, +(o.to || c.D)), vert = c.H > c.W, tw = +(o.tw || (vert ? 256 : 400));
  let times = [];
  if (o.at) times = String(o.at).split(',').map(v => Math.round(+v * c.fps) / c.fps);   // свои моменты: доска стиля
  else if (o.frames) for (let f = Math.round(a * c.fps); f <= Math.round(b * c.fps) && times.length < 72; f += +(o.frames === true ? 1 : o.frames)) times.push(f / c.fps);
  else { const n = +(o.n || 12); for (let i = 0; i < n; i++) times.push(Math.round((a + (b - a - 1 / c.fps) * i / Math.max(1, n - 1)) * c.fps) / c.fps); }
  const S = await open(dir, { scale: tw / c.W, guides: o.guides }), items = [];
  for (const t of times) items.push({ buf: await shot(S.pages[0], t, true), label: t.toFixed(2) + ' с' });
  const f = o.out || path.join(dir, 'out', o.frames ? `strip_${a}-${b}.jpg` : (o.from || o.to) ? `sheet_${a}-${b}.jpg` : 'sheet.jpg');
  fs.mkdirSync(path.dirname(f), { recursive: true });
  await grid(S.browser, items, o.at ? items.length : vert ? 6 : 4, tw, f, o.title);
  if (S.logs.length) out('! ' + S.logs.join('\n! '));
  await S.browser.close();
  out(f);
}
async function render(dir, o) {
  const draft = !!o.draft, alpha = o.alpha === true ? 'mov' : o.alpha;
  if (alpha && !['mov', 'webm', 'png'].includes(alpha)) die('--alpha mov | webm | png');
  const S = await open(dir, { scale: draft ? 0.5 : +(o.scale || 1), alpha: !!alpha, pages: Math.max(1, Math.min(+(o.workers || 2), os.cpus().length)) });
  const info = S.info, fps = info.fps, a = +(o.from || 0), b = Math.min(info.D, +(o.to || info.D));
  const mb = alpha ? 1 : Math.max(1, Math.min(8, Math.round(+(o.mblur === true ? 4 : o.mblur == null ? (draft ? 1 : 4) : o.mblur) || 1)));   // подкадров на кадр: размытие движения
  const f0 = Math.round(a * fps), NF = Math.round(b * fps) - f0, N = NF * mb, jpeg = (draft || mb > 1) && !alpha;   // подкадры усредняются, им хватает JPEG
  if (NF <= 0) die('пустой диапазон --from/--to');
  const od = path.join(dir, 'out');
  fs.mkdirSync(od, { recursive: true });
  const file = path.resolve(o.out || path.join(od, alpha ? (alpha === 'png' ? 'overlay_png' : 'overlay.' + alpha) : (draft ? 'draft' : 'final') + (o.from || o.to ? `_${a}-${b}` : '') + '.mp4'));
  const au = alpha || o['no-audio'] ? null : audio(dir, info, a, b);
  const vf = (mb > 1 ? `tmix=frames=${mb},select='eq(mod(n\\,${mb})\\,${mb - 1})',setpts=N/(${fps}*TB),` : '') + 'scale=trunc(iw/2)*2:trunc(ih/2)*2:out_color_matrix=bt709:out_range=tv,format=yuv420p' + (info.grain > 0 ? `,noise=c0s=${Math.min(30, Math.round(info.grain * 60))}:c0f=t+u` : '');
  let ff = null, proc = null, err = '';
  if (alpha === 'png') fs.mkdirSync(file, { recursive: true });
  else {
    ff = ['-y', '-v', 'error', '-f', 'image2pipe', '-framerate', String(fps * mb), '-c:v', jpeg ? 'mjpeg' : 'png', '-i', '-'];
    if (alpha === 'mov') ff.push('-c:v', 'prores_ks', '-profile:v', '4444', '-pix_fmt', 'yuva444p10le', '-vendor', 'apl0');
    else if (alpha === 'webm') ff.push('-c:v', 'libvpx-vp9', '-pix_fmt', 'yuva420p', '-b:v', '0', '-crf', '28', '-auto-alt-ref', '0');
    else {
      if (au) ff.push('-i', au.wav, '-map', '0:v', '-map', '1:a', '-c:a', 'aac', '-b:a', '192k');
      ff.push('-vf', vf);
      ff.push('-c:v', 'libx264', '-preset', draft ? 'veryfast' : 'medium', '-crf', draft ? '23' : '16', '-maxrate', '10M', '-bufsize', '20M', '-colorspace', 'bt709', '-color_primaries', 'bt709', '-color_trc', 'bt709', '-movflags', '+faststart', '-t', (NF / fps).toFixed(3));
    }
    ff.push('-r', String(fps), file);
    proc = cp.spawn('ffmpeg', ff, { stdio: ['pipe', 'ignore', 'pipe'] });
    proc.stderr.on('data', d => { err += d; });
    proc.stdin.on('error', () => {});
  }
  let closed = false;
  const gone = proc ? new Promise(r => proc.on('close', () => { closed = true; r(); })) : null;
  const buf = new Map(), t0 = Date.now();
  let next = 0, written = 0, chain = Promise.resolve(), lastLog = 0;
  const put = async (i, b) => {
    if (!proc) return fs.promises.writeFile(path.join(file, String(i + 1).padStart(6, '0') + '.png'), b);
    if (closed) return;
    if (!proc.stdin.write(b)) await Promise.race([new Promise(r => proc.stdin.once('drain', r)), gone]);
  };
  const flush = async () => {
    while (buf.has(written)) {
      const b = buf.get(written); buf.delete(written);
      await put(written, b); written++;
      if (Date.now() - lastLog > 3000 || written === N) {
        lastLog = Date.now();
        const el = (Date.now() - t0) / 1000, eta = Math.round(el / written * (N - written));
        fs.writeFileSync(path.join(od, '.progress'), JSON.stringify({ frame: written, total: N, eta }));
        out(`кадр ${written}/${N}, осталось ~${eta} с`);
      }
    }
  };
  await Promise.all(S.pages.map(async pg => {
    for (;;) {
      const i = next++;
      if (i >= N || closed) break;
      while (i - written > 8 && !closed) await sleep(4);
      buf.set(i, await shot(pg, (f0 + Math.floor(i / mb) + (i % mb) * 0.5 / mb) / fps, jpeg && (draft ? 90 : 96)));
      chain = chain.then(flush);
      await chain;
    }
  }));
  await chain;
  await S.browser.close();
  if (proc) {
    proc.stdin.end();
    await gone;
    const code = proc.exitCode;
    if (code !== 0 || written < N) die('ffmpeg упал: ' + err.slice(-600));
    const p = probe(file), mbs = (fs.statSync(file).size / 1048576).toFixed(1);
    out(`ГОТОВО ${file}\n${p.w}×${p.h}, ${p.dur.toFixed(2)} с, ${p.codec}, размытие движения: ${mb > 1 ? mb + ' подкадра' : 'нет'}, ${mbs} МБ, рендер ${((Date.now() - t0) / 1000).toFixed(0)} с\nЗвук: ${p.audio && au ? au.report : 'нет'}`);
    if (Math.abs(p.dur - NF / fps) > 0.1) out('! длительность файла не совпала с ожидаемой ' + (NF / fps).toFixed(2) + ' с — проверь');
  } else out(`ГОТОВО ${file}/ — ${NF} PNG с альфой`);
  if (S.logs.length) out('! ' + S.logs.join('\n! '));
}

/* ---------- проверка ---------- */
function collect() {
  const st = document.getElementById('stage'), W = window.__motion.W, H = window.__motion.H;
  const w = document.createTreeWalker(st, NodeFilter.SHOW_TEXT), blocks = new Map(), fontq = [];
  window.__moN = window.__moN || 0;
  let n;
  while ((n = w.nextNode())) {
    if (!n.nodeValue.trim()) continue;
    const el = n.parentElement;
    if (!el || el.closest('[data-decor],script,style')) continue;
    let blk = el.closest('[data-text],[data-cap]');
    if (!blk) { blk = el; while (blk.parentElement && blk !== st && (getComputedStyle(blk).display.startsWith('inline') || /^mo-[lwc]/.test(blk.className))) blk = blk.parentElement; }
    let op = 1, vis = true;
    const rg = document.createRange();
    rg.selectNodeContents(n);
    const r0 = rg.getBoundingClientRect();
    let x1 = r0.left, y1 = r0.top, x2 = r0.right, y2 = r0.bottom;
    for (let e = el; e && e !== document.body; e = e.parentElement) {
      const cs = getComputedStyle(e);
      op *= parseFloat(cs.opacity);
      if (cs.visibility === 'hidden' || cs.display === 'none') vis = false;
      if (cs.clipPath && cs.clipPath !== 'none') x2 = x1;   // блок под шторкой перехода: покоящимся не считается
      if (e !== el && e !== st && st.contains(e) && cs.overflow !== 'visible') { const b = e.getBoundingClientRect(); x1 = Math.max(x1, b.left); y1 = Math.max(y1, b.top); x2 = Math.min(x2, b.right); y2 = Math.min(y2, b.bottom); }
    }
    const area = r0.width * r0.height, frac = area > 0 ? Math.max(0, x2 - x1) * Math.max(0, y2 - y1) / area : 0;
    if (!area) vis = false;
    const cs = getComputedStyle(el), sc = el.offsetWidth ? el.getBoundingClientRect().width / el.offsetWidth : 1;
    const ins = Math.min(r0.height, parseFloat(cs.fontSize) * sc) * 0.2;   // поля шрифта сверху и снизу не считаем текстом
    if (vis && op > 0.01 && !el.dataset.moF) { el.dataset.moF = ++window.__moN; fontq.push({ k: el.dataset.moF, fam: cs.fontFamily, text: n.nodeValue.trim().slice(0, 30), kids: [...el.querySelectorAll('*')].map(c => getComputedStyle(c).fontFamily.split(',')[0].replace(/["']/g, '').trim().toLowerCase()) }); }
    if (!blk.dataset.moB) blk.dataset.moB = ++window.__moN;
    let b = blocks.get(blk);
    if (!b) blocks.set(blk, b = { id: blk.dataset.moB, text: '', x1: 1e9, y1: 1e9, x2: -1e9, y2: -1e9, op: 1, frac: 1, vis: false, fs: 1e9,
      small: blk.closest('[data-small]') != null, bleed: blk.closest('[data-bleed]') != null, cap: blk.closest('[data-cap]') != null });
    const wp = el.classList.contains('mo-c') ? el.parentElement : null;
    b.text += (b.text && !(wp && wp === b.lw) ? ' ' : '') + n.nodeValue.trim(); b.lw = wp;
    if (vis) { b.vis = true; b.x1 = Math.min(b.x1, r0.left); b.y1 = Math.min(b.y1, r0.top + ins); b.x2 = Math.max(b.x2, r0.right); b.y2 = Math.max(b.y2, r0.bottom - ins);
      b.op = Math.min(b.op, op); b.frac = Math.min(b.frac, frac); b.fs = Math.min(b.fs, parseFloat(cs.fontSize) * sc); } else b.op = 0;
  }
  const faces = [];                                  // где на экране лицо из нарезки (по M.data.faces) и сколько его видно в окне
  st.querySelectorAll('img[data-footage]').forEach(im => {
    const F = (window.M.data.faces || {})[im.dataset.footage.split('/').pop()];
    if (!F) return;
    let op = 1; for (let e = im; e && e !== document.body; e = e.parentElement) { const cs = getComputedStyle(e); op *= parseFloat(cs.opacity); if (cs.visibility === 'hidden' || cs.display === 'none') op = 0; }
    if (op < 0.15) return;
    const r = im.getBoundingClientRect(), k = Math.max(r.width / F.W, r.height / F.H), pos = getComputedStyle(im).objectPosition.split(' ').map(v => parseFloat(v) / 100);
    const ox = r.left + (r.width - F.W * k) * pos[0], oy = r.top + (r.height - F.H * k) * pos[1];
    let V = [Math.max(0, r.left), Math.max(0, r.top), Math.min(W, r.right), Math.min(H, r.bottom)];
    for (let e = im.parentElement; e && e !== st; e = e.parentElement) if (getComputedStyle(e).overflow !== 'visible') { const b = e.getBoundingClientRect(); V = [Math.max(V[0], b.left), Math.max(V[1], b.top), Math.min(V[2], b.right), Math.min(V[3], b.bottom)]; }
    const x1 = ox + (F.cx - F.w / 2) * k, y1 = oy + (F.cy - F.h / 2) * k, x2 = x1 + F.w * k, y2 = y1 + F.h * k * 1.1;
    faces.push({ x1, y1, x2, y2, frac: Math.max(0, Math.min(x2, V[2]) - Math.max(x1, V[0])) * Math.max(0, Math.min(y2, V[3]) - Math.max(y1, V[1])) / ((x2 - x1) * (y2 - y1)), sig: [x1, y1, x2].map(Math.round).join(',') });
  });
  return { faces, blocks: [...blocks.values()].map(b => Object.assign(b, { sig: [b.x1, b.y1, b.x2, b.y2].map(v => Math.round(v * 2)).join(',') + '|' + b.op.toFixed(2) + '|' + b.frac.toFixed(2) + '|' + b.text })), fontq, W, H };
}
async function check(dir) {
  const c = cfg(dir), E = [], Wn = [], seen = new Set();
  const add = (list, key, msg) => { if (!seen.has(key)) { seen.add(key); list.push(msg); } };
  const q = s => '«' + (s.length > 42 ? s.slice(0, 40) + '…' : s) + '»';
  // статический линт
  const files = fs.readdirSync(dir).filter(f => /\.(html|js|css)$/.test(f) && f !== 'motion.js');
  const RULES = [[/(^|[;{\s"'])(transition|animation)(-[a-z-]+)?\s*:/, 'CSS transition/animation — движение только через M.tl'], [/@keyframes/, '@keyframes — движение только через M.tl'],
    [/\b(setTimeout|setInterval|requestAnimationFrame)\s*\(/, 'таймеры и rAF привязаны к реальным часам — ставь всё на M.tl или M.onFrame'], [/Math\.random\s*\(/, 'Math.random() — возьми M.rand(seed)'],
    [/\bDate\.now\s*\(|new Date\s*\(|performance\.now\s*\(/, 'реальное время в коде — кадр должен зависеть только от t'], [/<video\b/i, '<video> не рендерится покадрово — используй <img data-footage>'],
    [/(src|href)\s*=\s*["']https?:\/\/|url\(\s*["']?https?:\/\/|@import\s+["']https?:/, 'внешняя ссылка — в рендере сети нет, клади файлы локально'], [/repeat\s*:\s*-1/, 'repeat:-1 делает таймлинию бесконечной — задай число повторов']];
  for (const f of files) fs.readFileSync(path.join(dir, f), 'utf8').replace(/<!--[\s\S]*?-->|\/\*[\s\S]*?\*\//g, '').split('\n').forEach((ln, i) => {
    for (const [re, msg] of RULES) if (re.test(ln)) add(E, f + i + msg, `[код] ${f}:${i + 1} ${msg}`);
  });
  const S = await open(dir, { scale: 0.5 }), pg = S.pages[0], info = S.info, fps = info.fps, D = info.D;
  S.logs.forEach(l => add(E, l, '[страница] ' + l));
  if (info.tlDur > D + 1 / fps) add(E, 'dur', `[тайминг] анимация длится ${info.tlDur.toFixed(2)} с, а ролик ${D} с — хвост обрежется`);
  if (info.stray) add(E, 'stray', `[код] ${info.stray} аним. вне M.tl (gsap.to/from без таймлинии) — при рендере они не двигаются`);
  // детерминизм: тот же кадр после другой перемотки. Сравниваем пиксели с допуском — на стыках тайлов Chromium сам даёт единичные расхождения
  const cmp = await S.browser.newPage(), same = (a, b2) => cmp.evaluate(async ([x, y]) => { const px = async u => { const im = await createImageBitmap(await (await fetch(u)).blob()), c = new OffscreenCanvas(im.width, im.height).getContext('2d'); c.drawImage(im, 0, 0); return c.getImageData(0, 0, im.width, im.height).data; };
    const p = await px(x), q2 = await px(y); let n = 0; for (let k = 0; k < p.length; k += 4) if (Math.abs(p[k] - q2[k]) + Math.abs(p[k + 1] - q2[k + 1]) + Math.abs(p[k + 2] - q2[k + 2]) > 48) n++; return n / (p.length / 4); }, [a, b2].map(v => 'data:image/png;base64,' + v.toString('base64')));
  for (const t of [0.31, 0.62, 0.9].map(k => Math.round(D * k * fps) / fps)) {
    await pg.page.evaluate(() => window.renderFrame(0));
    const h1 = await shot(pg, t, false);
    await pg.page.evaluate(d => window.renderFrame(d), D);
    await pg.page.evaluate(x => window.renderFrame(x), Math.max(0, t - 0.5));
    if (await same(h1, await shot(pg, t, false)) > 0.0005) add(E, 'det', `[детерминизм] кадр ${t.toFixed(2)} с выходит разным при разной перемотке — где-то состояние копится вне M.tl`);
  }
  await cmp.close();
  // проход по времени
  await S.pages[0].cdp.send('DOM.enable'); await S.pages[0].cdp.send('CSS.enable');
  const step = Math.max(1, Math.round(fps / 10)) / fps, k = Math.min(info.W, info.H) / 1080, minFs = 44 * k, sz = info.safe;
  const read = new Map(), fontSeen = new Map(), sigs = [], voiced = info.footage.some(f => f.voice) || info.audio.some(a2 => a2.voice);
  let prev = new Map(), fsig = '', frun = 0;
  const onFace = new Map(), life = new Map();
  for (let t = 0; t <= D + 1e-6; t += step) {
    await pg.page.evaluate(x => window.renderFrame(x), Math.min(D, t));
    const r = await pg.page.evaluate(collect), cur = new Map(r.blocks.map(b => [b.id, b]));
    for (const fq of r.fontq) {
      const decl = fq.fam.split(',')[0].replace(/["']/g, '').trim(), key = decl + '|' + [...new Set(fq.text)].sort().join('');
      if (fontSeen.has(key)) continue;
      fontSeen.set(key, 1);
      if (/^(sans-serif|serif|monospace|system-ui|ui-[a-z-]+)$/i.test(decl)) { add(Wn, 'gen' + decl, `[шрифт] ${q(fq.text)} — шрифт не задан явно (${decl}), на другой машине будет другим`); continue; }
      try {
        const doc = await pg.cdp.send('DOM.getDocument', { depth: 0 });
        const nd = await pg.cdp.send('DOM.querySelector', { nodeId: doc.root.nodeId, selector: `[data-mo-f="${fq.k}"]` });
        const pf = await pg.cdp.send('CSS.getPlatformFontsForNode', { nodeId: nd.nodeId });
        for (const u of pf.fonts) {
          const un = u.familyName.toLowerCase(), dn = decl.toLowerCase();
          if (/emoji/.test(un) || un.startsWith(dn) || dn.startsWith(un) || (fq.kids || []).some(kf => kf && (un.startsWith(kf) || kf.startsWith(un)))) continue;   // вложенный элемент со своим шрифтом — не подмена
          add(E, 'font' + decl + un, `[шрифт] ${q(fq.text)}: объявлен ${decl}, а ${u.glyphCount} глиф. нарисованы шрифтом ${u.familyName} — подмена (нет кириллицы или шрифт не подключён)`);
        }
      } catch (e) {}
    }
    const rest = [];
    for (const fc of r.faces) {
      frun = fc.sig === fsig ? frun + 1 : 0; fsig = fc.sig;
      if (frun === 3 && fc.frac > 0.05 && fc.frac < 0.85) add(Wn, 'crop' + fc.sig, `[лицо] окно обрезает лицо: видно ${Math.round(fc.frac * 100)}% (${t.toFixed(1)} с) — поправь форму окна или задай face в M.win`);
      if (fc.frac > 0.3) for (const b of r.blocks) if (b.vis && b.op >= 0.9 && Math.min(b.x2, fc.x2) - Math.max(b.x1, fc.x1) > 4 && Math.min(b.y2, fc.y2) - Math.max(b.y1, fc.y1) > 4) { const n = (onFace.get(b.id) || 0) + 1; onFace.set(b.id, n); if (n === 3) add(Wn, 'face' + b.id, `[лицо] ${q(b.text)} стоит на лице (${t.toFixed(1)} с)`); }
    }
    for (const b of r.blocks) if (b.vis && b.op >= 0.5 && !b.cap && b.x2 > 4 && b.y2 > 4 && b.x1 < r.W - 4 && b.y1 < r.H - 4) { const n = (life.get(b.id) || 0) + step; life.set(b.id, n); if (n > 6 && n - step <= 6) add(Wn, 'hang' + b.id, `[висит] ${q(b.text)} на экране дольше 6 с — так задумано?`); }
    for (const b of r.blocks) {
      const p = prev.get(b.id), still = p && p.sig === b.sig, ok = b.vis && b.op >= 0.9 && b.frac >= 0.6;
      if (b.cap && ok && b.x2 > b.x1) { rest.push(b); continue; }   // субтитры не «стоят», но наложение на графику ловим
      if (still && b.vis && b.op >= 0.9 && b.frac > 0.25 && b.frac < 0.6) add(Wn, 'clip' + b.id, `[обрезка] ${q(b.text)} обрезан контейнером с overflow (${t.toFixed(1)} с)`);
      if (!(still && ok)) continue;
      const at = ` (${t.toFixed(1)} с)`;
      if (b.x2 < 4 || b.y2 < 4 || b.x1 > r.W - 4 || b.y1 > r.H - 4) continue;   // целиком за кадром (соседняя сцена на холсте камеры)
      read.set(b.id, { t: (read.get(b.id) || { t: 0 }).t + step, text: b.text, cap: b.cap });
      rest.push(b);
      if (b.x1 < -2 || b.y1 < -2 || b.x2 > r.W + 2 || b.y2 > r.H + 2) add(info.camera ? Wn : E, 'out' + b.id, `[кадр] ${q(b.text)} обрезан краем кадра${at}` + (info.camera ? ' — если это сосед на холсте камеры, раздвинь сцены' : ''));
      else if (!b.bleed && (b.x1 < sz.left - 2 || b.y1 < sz.top - 2 || b.x2 > r.W - sz.right + 2 || b.y2 > r.H - sz.bottom + 2)) add(Wn, 'safe' + b.id, `[зона] ${q(b.text)} вне безопасной зоны${at}: x ${Math.round(b.x1)}–${Math.round(b.x2)}, y ${Math.round(b.y1)}–${Math.round(b.y2)}`);
      if (!b.small && b.fs < minFs - 0.5) add(Wn, 'fs' + b.id, `[кегль] ${q(b.text)} — ${Math.round(b.fs)} px, минимум для этого кадра ${Math.round(minFs)} px (или пометь data-small)`);
    }
    for (let i = 0; i < rest.length; i++) for (let j = i + 1; j < rest.length; j++) {
      const A = rest[i], B = rest[j];
      if (Math.min(A.x2, B.x2) - Math.max(A.x1, B.x1) > 6 && Math.min(A.y2, B.y2) - Math.max(A.y1, B.y1) > 6) add(Wn, 'ov' + A.id + '_' + B.id, `[наложение] ${q(A.text)} и ${q(B.text)} перекрываются (${t.toFixed(1)} с)`);
    }
    sigs.push(r.blocks.filter(b => b.vis && b.op > 0.05 && !b.cap).map(b => b.sig).join(';'));
    prev = cur;
  }
  for (const [id, v] of read) {
    const ch = v.text.replace(/\s/g, '').length, need = Math.max(0.3, ch / (voiced ? 34 : 20));   // под речь титр ещё и слышен
    if (!v.cap && v.t + 1e-6 < need) add(Wn, 'rd' + id, `[чтение] ${q(v.text)} (${ch} зн.) стоит читаемым ${v.t.toFixed(1)} с, нужно ≥ ${need.toFixed(1)} с`);
  }
  for (const b of prev.values()) if (!read.has(b.id) && !b.cap && b.text.length > 2) add(Wn, 'nv' + b.id, `[чтение] ${q(b.text)} ни разу не стоит неподвижно и полностью видимым`);
  if (!sigs[0] && !info.footage.length) add(Wn, 'first', '[начало] на первом кадре нет текста — если это обложка или хук, первый кадр должен быть содержательным');
  const tail = sigs.slice(-Math.round(1 / step));
  if (!info.loop && tail.some(s => s !== tail[tail.length - 1])) add(Wn, 'hold', '[финал] в последнюю секунду текст ещё меняется — нет удержания финального кадра (≥ 1 с)');
  const evs = (info.events || []).filter(t => t < D); let lastEv = 0;
  if (!evs.length || evs[0] > 0.5) add(Wn, 'hook', `[хук] первое движение начинается на ${evs.length ? evs[0].toFixed(1) : '—'} с — в первые полсекунды уже должно что-то происходить`);
  [...evs, D - 1].forEach(t => { if (t - lastEv > 2.2) add(Wn, 'gap' + lastEv, `[ритм] с ${lastEv.toFixed(1)} по ${t.toFixed(1)} с не начинается ни одно новое движение — добавь событие или оставь осознанно`); lastEv = Math.max(lastEv, t); });
  await S.browser.close();
  out(`ПРОВЕРКА ${dir}: ${info.W}×${info.H}, ${fps} fps, ${D} с; движений ${evs.length} (${(evs.length / D).toFixed(1)} в секунду), звуков ${(info.sfx || []).length}; метки: ${Object.entries(info.labels).map(([n, t]) => n + ' ' + t.toFixed(2)).join(', ') || 'нет'}`);
  out(E.length ? `ОШИБКИ (${E.length}) — исправить до рендера:\n  ✗ ` + E.join('\n  ✗ ') : 'Ошибок нет.');
  out(Wn.length ? `ПРЕДУПРЕЖДЕНИЯ (${Wn.length}) — исправить или осознанно оставить:\n  ! ` + Wn.join('\n  ! ') : 'Предупреждений нет.');
  out('Проверка не видит вкус, иерархию и ритм — дальше смотри sheet и stills.');
  process.exit(E.length ? 1 : 0);
}

/* ---------- данные проекта: иконки, слова, лица ---------- */
function icons(o) {                                // mo icons rocket clock — поиск по 5000 контурным иконкам Tabler
  if (!ex(mod('@tabler/icons'))) npmi(['@tabler/icons']);
  const all = fs.readdirSync(path.join(mod('@tabler/icons'), 'icons', 'outline')).map(n => n.replace('.svg', ''));
  for (const w of o._.slice(1)) out(w + ': ' + (all.filter(n => n.includes(w.toLowerCase())).slice(0, 40).join(' ') || 'нет совпадений (ищи по-английски: clock, rocket, chart…)'));
  if (o._.length < 2) out('mo icons <слово…> — найдёт имена; потом mo data <dir> --icons имя1,имя2. Всего ' + all.length);
}
function data(dir, o) {                            // пишет data.js: M.data.icons / words / faces
  const dj = path.join(dir, 'data.js');
  let js = ex(dj) ? fs.readFileSync(dj, 'utf8') : '';
  const put = (kind, key, val) => { const tag = `/*${kind}:${key}*/`; js = js.split('\n').filter(l => l && !l.startsWith(tag)).join('\n') + `\n${tag}M.data.${kind}[${JSON.stringify(key)}]=${JSON.stringify(val)};\n`; out(`data.js: M.data.${kind}[${JSON.stringify(key)}]`); };
  if (o.icons) { if (!ex(mod('@tabler/icons'))) npmi(['@tabler/icons']);
    for (const n of String(o.icons).split(',').map(x => x.trim()).filter(Boolean)) { const f = path.join(mod('@tabler/icons'), 'icons', 'outline', n + '.svg');
      if (!ex(f)) { out('! нет иконки ' + n + ' — поищи: mo icons ' + n.split('-')[0]); continue; }
      put('icons', n, fs.readFileSync(f, 'utf8').replace(/^[\s\S]*?<svg[^>]*>/, '').replace(/<\/svg>\s*$/, '').replace(/<path stroke="none"[^>]*\/>/, '').replace(/\s+/g, ' ').trim()); } }
  if (o.words) { const j = JSON.parse(fs.readFileSync(o.words, 'utf8')); put('words', path.basename(o.words).replace(/(_words)?\.json$/, ''), j.words || j); }
  if (o.faces) { const j = JSON.parse(fs.readFileSync(o.faces, 'utf8')), md = k => { const v = j.faces.map(q => q[k]).sort((x, y) => x - y); return v[v.length >> 1]; };
    if (!j.faces.length) out('! в ' + o.faces + ' нет лиц — окно будет кадрироваться по умолчанию');
    else { const w = md(3), h = md(4); put('faces', path.basename(j.file), { W: j.W, H: j.H, cx: Math.round(md(1) + w / 2), cy: Math.round(md(2) + h / 2), w: Math.round(w), h: Math.round(h) }); } }
  fs.writeFileSync(dj, js);
}
function faces(o) {                                // mo faces <видео> [шаг] → <имя>.faces.json (OpenCV ставится сам)
  const f = o._[1], py = path.join(CACHE, 'py');
  if (!f || !ex(f)) die('mo faces <видео> [шаг_с]');
  const env = Object.assign({}, process.env, { PYTHONPATH: py });
  if (run('python3', ['-c', 'import cv2,numpy;assert cv2.__version__[0]<"5"'], { env }).status !== 0) {
    const r = run('python3', ['-m', 'pip', 'install', '-q', '--break-system-packages', '--target', py, 'opencv-python-headless<5', 'numpy']);
    if (r.status !== 0) die('не удалось поставить OpenCV (нет сети?): ' + (r.stderr || '').slice(-200));
  }
  const r = run('python3', [path.join(TOOL, 'faces.py'), f, String(o._[2] || 0.2)], { env });
  if (r.status !== 0) die('faces.py: ' + (r.stderr || '').slice(-300));
  const j = JSON.parse(r.stdout), file = path.basename(f).replace(/\.[^.]+$/, '') + '.faces.json', n = Math.round(probe(f).dur / (+o._[2] || 0.2));
  fs.writeFileSync(file, r.stdout);
  out(`${file}: лицо найдено в ${j.faces.length} из ~${n} отсчётов` + (j.faces.length ? '' : ' — проверь лист кадров, детектор мог не узнать профиль'));
}

/* ---------- разбор готового видео: референс или свой рендер ---------- */
function measure(file) {                           // ритм движения, склейки, палитра, звук
  const p = probe(file);
  if (!p.w) die(file + ': нет видеодорожки');
  const fr = Math.min(30, Math.round(p.fps) || 30), gw = p.w > p.h ? 128 : 96, gh = Math.max(2, Math.round(gw * p.h / p.w / 2) * 2), sz = gw * gh;
  const raw = cp.spawnSync('ffmpeg', ['-v', 'error', '-i', file, '-vf', `fps=${fr},scale=${gw}:${gh},format=gray`, '-f', 'rawvideo', '-'], { maxBuffer: 1 << 30 }).stdout;
  const n = Math.floor(raw.length / sz), E = new Float32Array(n);
  for (let i = 1; i < n; i++) { let s = 0; const a = (i - 1) * sz, b = i * sz; for (let j = 0; j < sz; j++) s += Math.abs(raw[a + j] - raw[b + j]); E[i] = s / sz; }
  const srt = Array.from(E).sort((x, y) => x - y), thr = Math.max(0.12, srt[Math.floor(n * 0.2)] * 2 + 0.08), cuts = [], bursts = [], holds = [];
  for (let i = 2; i < n - 1; i++) if (E[i] > 12 && E[i] > 3.5 * Math.max(E[i - 1], E[i + 1], 0.5)) cuts.push(+(i / fr).toFixed(2));
  let st = -1, q = 0;
  for (let i = 0; i <= n; i++) { const on = i < n && E[i] > thr;
    if (on) { if (st < 0) st = i; q = 0; } else if (st >= 0 && (++q > fr * 0.1 || i === n)) { const e = i - q + 1; if (e - st >= 2) { let pk = st; for (let j = st; j < e; j++) if (E[j] > E[pk]) pk = j; bursts.push({ s: +(st / fr).toFixed(2), e: +(e / fr).toFixed(2), peak: +((pk - st) / Math.max(1, e - st - 1)).toFixed(2), amp: +E[pk].toFixed(1) }); } st = -1; q = 0; } }
  let last = 0; bursts.forEach(b => { if (b.s - last >= 0.3) holds.push([last, b.s]); last = b.e; }); if (n / fr - last >= 0.3) holds.push([last, +(n / fr).toFixed(2)]);
  const med = v => { const a = v.slice().sort((x, y) => x - y); return a.length ? a[a.length >> 1] : 0; }, mov = bursts.reduce((s, b) => s + b.e - b.s, 0);
  const pal = {}, rgb = cp.spawnSync('ffmpeg', ['-v', 'error', '-i', file, '-vf', `fps=${Math.max(0.2, Math.min(4, 24 / p.dur)).toFixed(3)},scale=48:-2`, '-pix_fmt', 'rgb24', '-f', 'rawvideo', '-'], { maxBuffer: 1 << 30 }).stdout;
  for (let i = 0; i + 2 < rgb.length; i += 3) { const k = (rgb[i] >> 4) << 8 | (rgb[i + 1] >> 4) << 4 | rgb[i + 2] >> 4, c = pal[k] || (pal[k] = [0, 0, 0, 0]); c[0]++; c[1] += rgb[i]; c[2] += rgb[i + 1]; c[3] += rgb[i + 2]; }
  const cols = Object.values(pal).sort((x, y) => y[0] - x[0]).slice(0, 40).map(c => ({ share: c[0] / (rgb.length / 3), rgb: [c[1] / c[0], c[2] / c[0], c[3] / c[0]].map(Math.round) })), top = [];
  for (const c of cols) { const m = top.find(t => Math.hypot(t.rgb[0] - c.rgb[0], t.rgb[1] - c.rgb[1], t.rgb[2] - c.rgb[2]) < 40); if (m) m.share += c.share; else if (top.length < 6) top.push(c); }
  top.forEach(t => { t.hex = '#' + t.rgb.map(v => v.toString(16).padStart(2, '0')).join('').toUpperCase(); t.share = +(t.share * 100).toFixed(1); });
  const R = { file, w: p.w, h: p.h, fps: p.fps, dur: +p.dur.toFixed(2), fr, E: Array.from(E, v => +v.toFixed(2)), cuts, bursts, holds, palette: top,
    stat: { moving: +(mov / (n / fr) * 100).toFixed(0), perSec: +(bursts.length / (n / fr)).toFixed(2), burst: +med(bursts.map(b => b.e - b.s)).toFixed(2), hold: +med(holds.map(h => h[1] - h[0])).toFixed(2), first: bursts.length ? bursts[0].s : null,
      shape: +med(bursts.filter(b => b.e - b.s > 0.2).map(b => b.peak)).toFixed(2), longestHold: +Math.max(0, ...holds.map(h => h[1] - h[0])).toFixed(2) } };
  if (p.audio) { R.loud = loud(file); R.bands = bands(file); try { R.beat = beatInfo(file); } catch (e) {}  }
  return R;
}
const shapeName = v => (v < 0.34 ? 'резкий старт, плавная остановка (ease-out)' : v < 0.6 ? 'разгон и торможение (in-out)' : 'разгон к концу (ease-in)');
async function rhythm(browser, list, file) {       // картинка: кривая движения, склейки и звуковые удары
  const Wd = 1600, rows = list.map((R, k) => { const mx = Math.max(4, ...R.E.slice().sort((a, b) => a - b).slice(0, Math.floor(R.E.length * 0.98))), x = t => (40 + t / R.dur * (Wd - 60)).toFixed(1), y0 = 150 + k * 190;
    return `<text x="40" y="${y0 - 132}" fill="#ddd" font-size="15">${path.basename(R.file)} · ${R.dur} с · движение ${R.stat.moving}% времени · вспышка ${R.stat.burst} с · пауза ${R.stat.hold} с</text>` +
      `<polyline fill="none" stroke="${k ? '#7cf' : '#fc5'}" stroke-width="1.5" points="${R.E.map((v, i) => x(i / R.fr) + ',' + (y0 - Math.sqrt(Math.min(1, v / mx)) * 110).toFixed(1)).join(' ')}"/>` +
      R.cuts.map(c => `<line x1="${x(c)}" x2="${x(c)}" y1="${y0 - 118}" y2="${y0}" stroke="#f55" stroke-dasharray="4 3"/>`).join('') +
      (R.beat ? R.beat.hits.map(h => `<circle cx="${x(h)}" cy="${y0 + 12}" r="4" fill="#6f6"/>`).join('') : '') +
      Array.from({ length: Math.floor(R.dur) + 1 }, (_, s) => `<text x="${x(s)}" y="${y0 + 34}" fill="#888" font-size="11" text-anchor="middle">${s}</text><line x1="${x(s)}" x2="${x(s)}" y1="${y0}" y2="${y0 + 5}" stroke="#666"/>`).join('') +
      `<line x1="40" x2="${Wd - 20}" y1="${y0}" y2="${y0}" stroke="#555"/>`; }).join('');
  const ctx = await browser.newContext({ viewport: { width: Wd, height: 60 + list.length * 190 } }), page = await ctx.newPage();
  await page.setContent(`<body style="margin:0;background:#111"><svg width="${Wd}" height="${60 + list.length * 190}" font-family="monospace">${rows}<text x="40" y="${50 + list.length * 190}" fill="#888" font-size="12">кривая — сколько меняется кадр; красный пунктир — склейка; зелёные точки — звуковые удары; внизу секунды</text></svg></body>`);
  await page.screenshot({ path: file }); await ctx.close();
}
async function frames(browser, file, times, cols, tw, outFile, title, label) {
  const tmp = fs.mkdtempSync(path.join(os.tmpdir(), 'mo-')), items = [];
  times.forEach((t, i) => { const jf = path.join(tmp, i + '.jpg'); run('ffmpeg', ['-y', '-v', 'error', '-ss', Math.max(0, t).toFixed(3), '-i', file, '-frames:v', '1', '-vf', `scale=${tw}:-2`, '-q:v', '4', jf]); if (ex(jf)) items.push({ buf: fs.readFileSync(jf), label: (label ? label(i) : '') + t.toFixed(2) + ' с' }); });
  await grid(browser, items, cols, tw, outFile, title);
}
function report(R) {
  const s = R.stat, L = [`${path.basename(R.file)}: ${R.w}×${R.h}, ${R.fps} fps, ${R.dur} с`,
    `Ритм: движение ${s.moving}% времени; ${R.bursts.length} вспышек движения (${s.perSec} в секунду), типичная длится ${s.burst} с, типичная пауза ${s.hold} с, самая длинная пауза ${s.longestHold} с; первое движение на ${s.first} с`,
    `Характер движения: пик на ${s.shape} длины вспышки — ${shapeName(s.shape)}`,
    `Жёсткие склейки (${R.cuts.length}): ${R.cuts.join(' ') || 'нет'}`,
    `Палитра: ${R.palette.map(c => c.hex + ' ' + c.share + '%').join(', ')}`];
  if (R.loud) L.push(`Звук: ${R.loud.I} LUFS, диапазон ${R.loud.LRA} LU, пик ${R.loud.TP} dBFS; баланс низ/тело/середина/верх ${R.bands.map(v => Math.round(v - Math.max(...R.bands))).join('/')} дБ` + (R.beat ? `; темп ≈ ${R.beat.bpm} BPM (уверенность ${R.beat.confidence}); ударов ${R.beat.hits.length}: ${R.beat.hits.join(' ')}` : ''));
  else L.push('Звука нет.');
  return L.join('\n');
}
async function analyze(o) {                        // mo analyze <видео> [--out папка]
  const f = o._[1]; if (!f || !ex(f)) die('mo analyze <видео> [--out папка]');
  const R = measure(f), od = o.out || f.replace(/\.[^.\/]+$/, '') + '_an', vert = R.h > R.w, tw = vert ? 256 : 400, browser = await playwright().chromium.launch();
  fs.mkdirSync(od, { recursive: true });
  await frames(browser, f, Array.from({ length: 24 }, (_, i) => R.dur * (i + 0.5) / 24), vert ? 8 : 6, tw, path.join(od, 'sheet.jpg'), path.basename(f));
  const ev = R.bursts.filter(b => b.e - b.s >= 0.15).slice(0, 16), bt = [];
  ev.forEach(b => bt.push(b.s, (b.s + b.e) / 2, b.e + 0.05));
  if (bt.length) await frames(browser, f, bt, vert ? 9 : 6, vert ? 200 : 300, path.join(od, 'moves.jpg'), 'каждое движение: начало · середина · конец', i => ['▶ ', '· ', '■ '][i % 3]);
  await rhythm(browser, [R], path.join(od, 'rhythm.png'));
  await browser.close();
  if (R.loud) { run('ffmpeg', ['-y', '-v', 'error', '-i', f, '-lavfi', 'showspectrumpic=s=1600x420:legend=0:scale=log:fscale=log', '-frames:v', '1', path.join(od, 'spectrum.png')]);
    run('ffmpeg', ['-y', '-v', 'error', '-i', f, '-vn', '-ar', String(SR), '-ac', '2', path.join(od, 'audio.wav')]); }
  fs.writeFileSync(path.join(od, 'analysis.json'), JSON.stringify(Object.assign({}, R, { E: undefined })));
  out(report(R));
  out('Вспышки движения (с): ' + R.bursts.map(b => b.s + '–' + b.e).join(', '));
  out(`Файлы в ${od}/: sheet.jpg (24 кадра), moves.jpg (начало-середина-конец движений), rhythm.png (кривая движения)` + (R.loud ? ', spectrum.png, audio.wav' : '') + ', analysis.json. Ход отдельного движения покадрово: mo strip <видео> --from a --to b');
}
async function compare(o) {                        // mo compare <референс> <мой> [--out файл.jpg]
  const fa = o._[1], fb = o._[2]; if (!fa || !fb || !ex(fa) || !ex(fb)) die('mo compare <референс> <мой ролик> [--out файл.jpg]');
  const A = measure(fa), B = measure(fb), vert = A.h > A.w, tw = vert ? 200 : 320, N = 8, browser = await playwright().chromium.launch(), of = o.out || 'compare.jpg', tmp = fs.mkdtempSync(path.join(os.tmpdir(), 'mo-')), items = [];
  for (const [R, nm] of [[A, 'реф '], [B, 'мой ']]) for (let i = 0; i < N; i++) { const t = R.dur * (i + 0.5) / N, jf = path.join(tmp, nm.trim() + i + '.jpg'); run('ffmpeg', ['-y', '-v', 'error', '-ss', t.toFixed(3), '-i', R.file, '-frames:v', '1', '-vf', `scale=${tw}:-2`, '-q:v', '4', jf]); if (ex(jf)) items.push({ buf: fs.readFileSync(jf), label: nm + t.toFixed(2) + ' с' }); }
  await grid(browser, items, N, tw, of, 'сверху референс, снизу мой ролик — те же относительные моменты');
  await rhythm(browser, [A, B], of.replace(/\.\w+$/, '') + '_rhythm.png'); await browser.close();
  const rs = (R, m) => Array.from({ length: m }, (_, i) => { const a = Math.floor(i * R.E.length / m), b = Math.max(a + 1, Math.floor((i + 1) * R.E.length / m)); let s = 0; for (let j = a; j < b; j++) s += Math.min(R.E[j], 12); return s / (b - a); });
  const x = rs(A, 120), y = rs(B, 120), mx = x.reduce((s, v) => s + v, 0) / 120, my = y.reduce((s, v) => s + v, 0) / 120; let sxy = 0, sxx = 0, syy = 0;
  for (let i = 0; i < 120; i++) { sxy += (x[i] - mx) * (y[i] - my); sxx += (x[i] - mx) ** 2; syy += (y[i] - my) ** 2; }
  const corr = sxy / Math.sqrt(sxx * syy + 1e-9), pd = A.palette.reduce((s, c) => s + c.share * Math.min(...B.palette.map(d => Math.hypot(c.rgb[0] - d.rgb[0], c.rgb[1] - d.rgb[1], c.rgb[2] - d.rgb[2]))), 0) / A.palette.reduce((s, c) => s + c.share, 0);
  const row = (n, a, b, bad) => out(`  ${n.padEnd(28)} ${String(a).padEnd(22)} ${String(b).padEnd(22)} ${bad ? '← расходится' : ''}`), d = (a, b, tol) => Math.abs(a - b) > tol;
  out('СВЕРКА'); row('параметр', 'референс', 'мой ролик');
  row('длина, с', A.dur, B.dur, d(A.dur, B.dur, 0.5)); row('движение, % времени', A.stat.moving, B.stat.moving, d(A.stat.moving, B.stat.moving, 15));
  row('вспышек в секунду', A.stat.perSec, B.stat.perSec, d(A.stat.perSec, B.stat.perSec, Math.max(0.3, A.stat.perSec * 0.3))); row('типичная вспышка, с', A.stat.burst, B.stat.burst, d(A.stat.burst, B.stat.burst, Math.max(0.12, A.stat.burst * 0.4)));
  row('типичная пауза, с', A.stat.hold, B.stat.hold, d(A.stat.hold, B.stat.hold, Math.max(0.2, A.stat.hold * 0.5))); row('пик движения (0–1)', A.stat.shape, B.stat.shape, d(A.stat.shape, B.stat.shape, 0.2));
  row('жёстких склеек', A.cuts.length, B.cuts.length, d(A.cuts.length, B.cuts.length, Math.max(1, A.cuts.length * 0.3)));
  row('палитра', A.palette.slice(0, 4).map(c => c.hex).join(' '), B.palette.slice(0, 4).map(c => c.hex).join(' '), pd > 45);
  if (A.loud && B.loud) { const bal = R => R.bands.map(v => Math.round(v - Math.max(...R.bands))); row('громкость, LUFS', A.loud.I, B.loud.I, d(A.loud.I, B.loud.I, 2)); row('баланс низ/тело/сер/верх', bal(A).join('/'), bal(B).join('/'), bal(A).some((v, i) => Math.abs(v - bal(B)[i]) > 8)); row('диапазон, LU', A.loud.LRA, B.loud.LRA, d(A.loud.LRA, B.loud.LRA, 4));
    if (A.beat && B.beat) { row('темп, BPM', A.beat.bpm, B.beat.bpm, d(A.beat.bpm, B.beat.bpm, 3)); row('звуковых ударов', A.beat.hits.length, B.beat.hits.length, d(A.beat.hits.length, B.beat.hits.length, Math.max(2, A.beat.hits.length * 0.3))); } }
  else if (A.loud && !B.loud) out('  ! у референса есть звук, у моего ролика нет');
  out(`  сходство ритма движения: ${corr.toFixed(2)} (1 — кривые совпадают; ниже 0,5 — ритм другой); расхождение палитры: ${pd.toFixed(0)} (0 — те же цвета; больше 45 — заметно другие)`);
  out(`Кадры рядом: ${of}; кривые: ${of.replace(/\.\w+$/, '')}_rhythm.png. Цифры не видят шрифт, композицию и форму объектов — их сверь глазами по кадрам.`);
}
async function strip(o) {                          // mo strip <видео> --from a --to b [--frames шаг] — покадровый ход движения в любом видео
  const f = o._[1]; if (!f || !ex(f)) die('mo strip <видео> --from с --to с [--frames шаг]');
  const p = probe(f), a = +(o.from || 0), b = Math.min(p.dur, +(o.to || a + 1)), stp = +(o.frames === true || !o.frames ? 1 : o.frames), fr = Math.round(p.fps) || 30, ts = [];
  for (let i = Math.round(a * fr); i <= Math.round(b * fr) && ts.length < 60; i += stp) ts.push(i / fr);
  const browser = await playwright().chromium.launch(), vert = p.h > p.w, of = o.out || f.replace(/\.[^.\/]+$/, '') + `_strip_${a}-${b}.jpg`;
  await frames(browser, f, ts, vert ? 10 : 6, +(o.tw || (vert ? 180 : 300)), of, `${path.basename(f)} ${a}–${b} с, каждый ${stp}-й кадр`); await browser.close(); out(of);
}
function grab(o) {                                 // mo grab <видео> 1.2,3.4 [--grid] — кадры 1:1, с сеткой 10 % для замера положений
  const f = o._[1], od = f && f.replace(/\.[^.\/]+$/, '') + '_frames';
  if (!f || !ex(f) || !o._[2]) die('mo grab <видео> 1.2,3.4 [--grid]');
  fs.mkdirSync(od, { recursive: true });
  for (const t of String(o._[2]).split(',').map(Number)) { const of = path.join(od, 't_' + t.toFixed(2).padStart(6, '0') + '.png');
    run('ffmpeg', ['-y', '-v', 'error', '-ss', t.toFixed(3), '-i', f, '-frames:v', '1', ...(o.grid ? ['-vf', 'drawgrid=w=iw/10:h=ih/10:t=2:c=magenta@0.7'] : []), of]); out(of); }
}

/* ---------- музыка-подложка ---------- */
function music(o) {                                // mo music <out.wav> --mood drive|chill|pulse|calm --dur 15 [--bpm N] [--key 0..11] [--seed N]
  const file = o._[1], mood = o.mood || 'drive', dur = +(o.dur || 15), key = +(o.key || 9), tail = Math.min(1.4, dur * 0.12);
  let bpm = +(o.bpm || { drive: 122, chill: 88, pulse: 108, calm: 76 }[mood] || 110);
  const nb = Math.max(1, Math.round((dur - tail) / (240 / bpm))); bpm = +(nb * 240 / (dur - tail)).toFixed(2);   // темп подгоняется так, чтобы целое число тактов кончилось перед хвостом
  if (!file) die('mo music <out.wav> --mood drive|chill|pulse|calm --dur 15 [--bpm N]');
  const N = Math.round(dur * SR), L = new Float32Array(N), Rr = new Float32Array(N), beat = 60 / bpm, bar = beat * 4, bars = nb;
  let sd = (+(o.seed || 1) * 7919) >>> 0; const rnd = () => (sd = sd * 1664525 + 1013904223 >>> 0) / 4294967296;
  const hz = n => 440 * Math.pow(2, (n - 69) / 12), prog = [[0, 3, 7, 10], [-4, 0, 3, 7], [-9, -5, -2, 3], [-2, 2, 5, 9]].map(c => c.map(n => 57 + ((key + 3) % 12) + n));   // i7 – VImaj7 – III – VII
  const put = (t, len, fn, g, pan) => { const a = Math.round(t * SR), n = Math.round(len * SR), gl = g * Math.cos(((pan || 0) + 1) * Math.PI / 4), gr = g * Math.sin(((pan || 0) + 1) * Math.PI / 4); let st = { ph: 0, y: 0, y2: 0 };
    for (let i = 0; i < n && a + i < N; i++) { if (a + i < 0) continue; const v = fn(i / SR, st); L[a + i] += v * gl; Rr[a + i] += v * gr; } };
  const kick = t => put(t, 0.3, (x, s) => { s.ph += 2 * Math.PI * (46 + 110 * Math.exp(-x / 0.03)) / SR; return Math.sin(s.ph) * Math.exp(-x / 0.11) * Math.min(1, x / 0.002); }, 0.9, 0);
  const snare = t => put(t, 0.2, (x, s) => { const nz = rnd() * 2 - 1; s.y += 0.5 * (nz - s.y); s.ph += 2 * Math.PI * 190 / SR; return ((nz - s.y) * 0.5 + s.y * 0.5) * Math.exp(-x / 0.06) + 0.4 * Math.sin(s.ph) * Math.exp(-x / 0.04); }, 0.38, 0.1);
  const hat = (t, g) => put(t, 0.06, (x, s) => { const nz = rnd() * 2 - 1; s.y += 0.75 * (nz - s.y); return (nz - s.y) * Math.exp(-x / 0.014); }, g, -0.25);
  const bass = (t, n, len) => put(t, len, (x, s) => { s.ph += hz(n) / SR; const saw = (s.ph % 1) * 2 - 1; s.y += 0.045 * (saw - s.y); s.y2 += 0.045 * (s.y - s.y2); return s.y2 * Math.min(1, x / 0.005) * Math.min(1, (len - x) / 0.03); }, 0.6, 0);
  const note = (t, n, len, g, pan, at, cut) => put(t, len, (x, s) => { s.ph += hz(n) / SR; const w = ((s.ph % 1) * 2 - 1) * 0.5 + (((s.ph * 1.007) % 1) * 2 - 1) * 0.5; s.y += cut * (w - s.y); s.y2 += cut * (s.y - s.y2); return s.y2 * Math.min(1, x / at) * Math.min(1, (len - x) / 0.06); }, g, pan);
  const pluck = (t, n, g, pan) => put(t, 0.35, (x, s) => { s.ph += 2 * Math.PI * hz(n) / SR; return (Math.sin(s.ph) + 0.3 * Math.sin(2 * s.ph)) * Math.exp(-x / 0.09) * Math.min(1, x / 0.003); }, g, pan);
  for (let b = 0; b < bars; b++) { const t0 = b * bar, ch = prog[b % 4], lastBar = b === bars - 1;
    if (mood === 'calm') { ch.forEach((n, i) => note(t0, n, bar * 1.05, 0.11, (i - 1.5) * 0.3, 0.5, 0.05)); for (let i = 0; i < 4; i++) if (rnd() < 0.6) pluck(t0 + i * beat + beat / 2, ch[(i * 3) % 4] + 12, 0.1, rnd() - 0.5); continue; }
    ch.forEach((n, i) => { if (mood === 'chill') note(t0, n, bar, 0.1, (i - 1.5) * 0.3, 0.02, 0.06); else for (const k of mood === 'drive' ? [0.5, 1.5, 2.5, 3.5] : [0, 1.5, 3]) note(t0 + k * beat, n, beat * 0.3, 0.12, (i - 1.5) * 0.3, 0.004, 0.12); });
    for (let i = 0; i < 4; i++) { const t = t0 + i * beat, sw = mood === 'chill' ? beat * 0.08 : 0;
      if (mood === 'drive' || i % 2 === 0 || (mood === 'chill' && i === 2 && b % 2)) kick(t); if (mood === 'chill' && i === 2) kick(t + beat * 0.75);
      if (i % 2 === 1) snare(t); hat(t + beat / 2 + sw, 0.14); if (mood !== 'chill') hat(t, 0.06); if (lastBar && i === 3) { hat(t + beat * 0.25, 0.1); hat(t + beat * 0.75, 0.12); }
      if (mood === 'drive') bass(t + beat / 2, ch[0] - 24, beat * 0.42); else if (i % 2 === 0) bass(t, ch[0] - 24, beat * (mood === 'chill' ? 1.4 : 0.9));
      if (mood === 'pulse') for (let e = 0; e < 2; e++) pluck(t + e * beat / 2, ch[(i * 2 + e + b) % 4] + 12, 0.13, (e ? 0.4 : -0.4)); } }
  const tE = bars * bar; kick(tE); prog[0].forEach((n, i) => note(tE, n, Math.max(0.3, dur - tE), 0.12, (i - 1.5) * 0.3, 0.004, 0.08));
  const dl = Math.round(beat * 0.75 * SR); for (let i = dl; i < N; i++) { L[i] += Rr[i - dl] * 0.16; Rr[i] += L[i - dl] * 0.16; }
  let pk = 0; for (let i = 0; i < N; i++) pk = Math.max(pk, Math.abs(L[i]), Math.abs(Rr[i]));
  const fo = Math.round(Math.min(1.2, dur - tE + 0.3) * SR); for (let i = 0; i < N; i++) { const g = 0.7 / pk * Math.min(1, (N - i) / fo); L[i] *= g; Rr[i] *= g; }
  wavWrite(file, [L, Rr]);
  out(`${file}: «${mood}», ${bpm} BPM (доля ${beat.toFixed(4)} с, такт ${bar.toFixed(3)} с), ${bars} тактов, финальный удар на ${tE.toFixed(2)} с, длина ${dur} с. Это служебная подложка: трек пользователя или референса всегда лучше.`);
}

/* ---------- медиа ---------- */
async function probeCmd(o) {
  for (const f of o._.slice(1)) {
    if (!ex(f)) die('нет файла ' + f);
    const p = probe(f);
    out(`${f}: ${p.w ? `${p.w}×${p.h}, ${p.fps} fps, ${p.codec}, ` : ''}${p.dur.toFixed(2)} с, звук: ${p.audio ? 'есть' : 'нет'}${p.hdr ? ', HDR (при разборе на кадры сводится в SDR)' : ''}`);
    if (!o.sheet || !p.w) continue;
    const n = +(o.n || 12), vert = p.h > p.w, tw = vert ? 256 : 400, items = [], tmp = fs.mkdtempSync(path.join(os.tmpdir(), 'mo-'));
    for (let i = 0; i < n; i++) {
      const t = p.dur * (i + 0.5) / n, jf = path.join(tmp, i + '.jpg');
      run('ffmpeg', ['-y', '-v', 'error', '-ss', t.toFixed(3), '-i', f, '-frames:v', '1', '-vf', `scale=${tw}:-2`, '-q:v', '4', jf]);
      if (ex(jf)) items.push({ buf: fs.readFileSync(jf), label: t.toFixed(2) + ' с' });
    }
    const browser = await playwright().chromium.launch(), file = path.resolve(path.basename(f).replace(/\.[^.]+$/, '') + '_sheet.jpg');
    await grid(browser, items, vert ? 6 : 4, tw, file, path.basename(f));
    await browser.close();
    out('  кадры: ' + file);
  }
}
function pcm(file, a, b, af) {
  const r = cp.spawnSync('ffmpeg', ['-v', 'error', ...(a ? ['-ss', String(a)] : []), ...(b ? ['-to', String(b)] : []), '-i', file, '-ac', '1', '-ar', '11000', ...(af ? ['-af', af] : []), '-f', 's16le', '-'], { maxBuffer: 1 << 30 });
  if (r.status !== 0) die('ffmpeg не прочитал звук из ' + file);
  return new Int16Array(r.stdout.buffer, r.stdout.byteOffset, r.stdout.length >> 1);
}
function env(x) {                                   // лог-энергия, шаг 10 мс
  const n = Math.floor(x.length / 110), e = new Float64Array(n);
  for (let i = 0; i < n; i++) { let s = 0; for (let j = 0; j < 110; j++) { const v = x[i * 110 + j]; s += v * v; } e[i] = Math.log(1 + s / 110); }
  return e;
}
function beatInfo(f, from, to) {
  const a = +(from || 0), eF = env(pcm(f, from, to)), eL = env(pcm(f, from, to, 'lowpass=f=150')), n = eF.length, on = new Float64Array(n);
  if (n < 300) throw new Error('слишком короткий звук для анализа');
  for (let i = 1; i < n; i++) on[i] = Math.max(0, eF[i] - eF[i - 1]) + 1.5 * Math.max(0, eL[i] - eL[i - 1]);
  const mean = on.reduce((s, v) => s + v, 0) / n, sd = Math.sqrt(on.reduce((s, v) => s + (v - mean) ** 2, 0) / n);
  const at = x => { const i = Math.floor(x), fr = x - i; return i + 1 < n && i >= 0 ? on[i] * (1 - fr) + on[i + 1] * fr : 0; };
  const ac = lag => { let s = 0; const m = n - Math.ceil(lag) - 1; for (let i = 0; i < m; i += 2) s += on[i] * at(i + lag); return s / Math.max(1, m); };
  let best = { s: -1, P: 50 };
  for (let P = 30; P <= 100; P += 0.05) {             // период в шагах по 10 мс: 200…60 BPM
    const bpm = 6000 / P, prior = Math.exp(-0.5 * (Math.log2(bpm / 120) / 0.9) ** 2);
    const s = (ac(P) + 0.7 * ac(2 * P) + 0.5 * ac(3 * P) + 0.4 * ac(4 * P)) * prior;
    if (s > best.s) best = { s, P };
  }
  let P = best.P, ph = 0, pb = -1;                    // уточняем период и фазу по всей длине трека
  for (let p = best.P - 0.6; p <= best.P + 0.6; p += 0.01) for (let x = 0; x < p; x += 0.25) {
    let s = 0, c = 0;
    for (let t = x; t < n; t += p) { s += at(t); c++; }
    if (s / c > pb) { pb = s / c; ph = x; P = p; }
  }
  pb *= Math.ceil((n - ph) / P);
  const grid = [];
  for (let t = ph; t < n; t += P) grid.push(+(a + t / 100).toFixed(3));
  const conf = pb / Math.max(1e-9, grid.length) / (mean + 1e-9);
  const hits = [];
  for (let i = 2; i < n - 2; i++) if (on[i] > mean + 2.5 * sd && on[i] >= on[i - 1] && on[i] >= on[i + 1] && on[i] >= on[i - 2] && on[i] >= on[i + 2]) hits.push({ t: +(a + i / 100).toFixed(2), v: on[i] });
  const top = hits.sort((x, y) => y.v - x.v).slice(0, 24).sort((x, y) => x.t - y.t).map(h => h.t);
  const sec = [];
  for (let i = 0; i + 200 <= n; i += 200) { let s = 0; for (let j = 0; j < 200; j++) s += eF[i + j]; sec.push(s / 200); }
  const jumps = [];
  for (let i = 1; i < sec.length; i++) if (sec[i] - sec[i - 1] > 0.35) jumps.push(+(a + i * 2).toFixed(1));
  return { bpm: +(6000 / P).toFixed(1), beat: +(P / 100).toFixed(4), first: grid[0], confidence: +conf.toFixed(2), beats: grid, hits: top, rises: jumps };
}
function beats(o) {
  const f = o._[1];
  if (!f || !ex(f)) die('mo beats <аудио или видео> [--from с] [--to с]');
  let res; try { res = beatInfo(f, o.from, o.to); } catch (e) { die(e.message); }
  const grid = res.beats, top = res.hits, jumps = res.rises, conf = res.confidence;
  if (o.json) fs.writeFileSync(o.json, JSON.stringify(res));
  out(`Темп ≈ ${res.bpm} BPM (доля ${res.beat} с), первая доля ${res.first} с, уверенность ${res.confidence}${conf < 2 ? ' — ритм неявный, сетке не доверяй' : ''}`);
  out('Доли: ' + grid.slice(0, 64).join(' ') + (grid.length > 64 ? ' …' : ''));
  out('Сильные удары: ' + (top.join(' ') || 'нет'));
  out('Подъёмы энергии (≈): ' + (jumps.join(' ') || 'нет'));
  out('Это оценка по огибающей: сильную долю такта и точный темп сверь на слух или у пользователя.');
}
function sfxCmd(o) {
  const kd = kit(o.kit || '01'), fl = fs.readdirSync(kd).filter(n => n.endsWith('.wav')).map(n => n.replace('.wav', ''));
  out(`Набор ${path.basename(kd)} (выбирается data-sfx на #stage: 01 мягкий, 02 тёплый низкий, 03 механический). Имена для M.sfx(имя, время):`);
  out('  удары (низ): hit — мягкий удар с подъездом, boom — большой удар с хвостом, thud — короткий толчок, stutter — дроблёный бас');
  out('  движение: swish (короткий), whoosh, sweep (длинный), riser — нарастание до удара, downer — спад после удара');
  out('  детали: thock key tap tick blip pop bubble scratch glitch swipe; тональные: chime ping swell shimmer ding rise drop');
  out('  одиночные: click select on off up down notify done caution disabled. Готовые связки: M.hit(t, "s"|"m"|"l"), M.ticks(t, {n, gap})');
  out('  свой файл: M.sfx("media/звук.wav", t). Все файлы набора: ' + fl.join(' '));
}
async function doctor(o) {
  out('mo v3 · node ' + process.version);
  for (const b of ['ffmpeg', 'ffprobe']) out(b + ': ' + (run(b, ['-version']).status === 0 ? 'есть' : 'НЕТ — поставь FFmpeg'));
  const enc = run('ffmpeg', ['-hide_banner', '-encoders']).stdout || '';
  ['libx264', 'prores_ks', 'libvpx-vp9', 'aac'].forEach(e => out('  кодек ' + e + ': ' + (enc.includes(e) ? 'есть' : 'НЕТ')));
  try { const b = await playwright().chromium.launch(); out('chromium: ' + b.version()); await b.close(); } catch (e) { out('chromium: НЕ запускается — ' + String(e.message).split('\n')[0]); }
  out('gsap: ' + (ex(mod('gsap')) ? 'в кэше' : run('npm', ['view', 'gsap', 'version']).status === 0 ? 'доступен в npm' : 'НЕТ доступа к npm'));
  if (!o.selftest) return;
  const d = fs.mkdtempSync(path.join(os.tmpdir(), 'mo-self-')), ix = path.join(d, 'index.html');
  init(d, { size: '540x960', dur: 1.2, force: true });
  fs.writeFileSync(ix, fs.readFileSync(ix, 'utf8').replace("M.text('#s1 h1', { at: 0.15, in: 'rise' });", "M.text('#s1 h1', { at: 0.1, in: 'pop' }); M.draw.underline('#s1 h1', { at: 0.4 }); M.draw.burst([0.5, 0.3], { at: 0.5 });"));
  await render(d, { draft: true, out: path.join(d, 't.mp4') });
  const p = probe(path.join(d, 't.mp4'));
  out('САМОТЕСТ: ' + (p.w && p.audio ? 'пройден (кадры, приёмы и звук собираются)' : 'НЕ ПРОЙДЕН'));
}

(async () => {
  const o = args(process.argv.slice(2)), cmd = o._[0], dir = o._[1];
  const need = () => { if (!dir) die('укажи папку проекта'); return dir; };
  if (cmd === 'init') init(need(), o);
  else if (cmd === 'fonts') fonts(need(), o._.slice(2), o);
  else if (cmd === 'check') await check(need());
  else if (cmd === 'stills') await stills(need(), o);
  else if (cmd === 'sheet') await sheet(need(), o);
  else if (cmd === 'render') await render(need(), o);
  else if (cmd === 'probe') await probeCmd(o);
  else if (cmd === 'beats') beats(o);
  else if (cmd === 'data') data(need(), o);
  else if (cmd === 'icons') icons(o);
  else if (cmd === 'faces') faces(o);
  else if (cmd === 'sfx') sfxCmd(o);
  else if (cmd === 'music') music(o);
  else if (cmd === 'analyze') await analyze(o);
  else if (cmd === 'compare') await compare(o);
  else if (cmd === 'strip') await strip(o);
  else if (cmd === 'grab') grab(o);
  else if (cmd === 'doctor') await doctor(o);
  else out(['mo init <dir> [--size 1080x1920] [--fps 30] [--dur 10] [--safe social|reels|tiktok|shorts|feed|broadcast|none|t,r,b,l]',
    'mo fonts <dir> "Unbounded:700,900" "Manrope:400,600" [--latin]',
    'mo probe <файлы…> [--sheet] [--n 12]      что за видео/звук + лист кадров',
    'mo beats <аудио> [--from с] [--to с]      темп, доли, удары',
    'mo check <dir>                            линт + шрифты + зоны + чтение + темп + детерминизм',
    'mo stills <dir> 0.5,2,4 [--guides] [--full]',
    'mo sheet <dir> [--n 12] [--from с --to с] [--frames [шаг]] [--at 0.5,2,4 --tw 480 --title "A" --out файл] [--guides]',
    'mo render <dir> [--draft] [--mblur 0-8] [--from с --to с] [--alpha mov|webm|png] [--no-audio] [--out файл]',
    'mo icons <слово…>                         найти контурные иконки',
    'mo data <dir> [--icons a,b] [--words f.json] [--faces f.faces.json]   данные в data.js',
    'mo faces <видео> [шаг]                    где лицо',
    'mo sfx [--kit 01|02|03]                   имена звуков',
    'mo music <out.wav> --mood drive|chill|pulse|calm --dur 15',
    'mo analyze <видео>                        ритм, палитра, звук, кадры движений (референс или свой рендер)',
    'mo compare <референс> <мой>               сверка рядом + таблица',
    'mo strip <видео> --from с --to с [--tw 300] [--out файл]   покадровый ход движения',
    'mo grab <видео> 1.2,3.4 [--grid]          кадры 1:1 (с сеткой 10 % — мерить положения)',
    'mo doctor [--selftest]'].join('\n'));
})().catch(e => die(e && e.stack || String(e)));
````

<!-- file: motion.js -->
````js
/* motion.js v3 — помощник страницы и набор приёмов. Подключается после gsap. Вся анимация живёт в M.tl. */
(function () {
  const stage = document.getElementById('stage'), d = stage.dataset;
  const W = +d.width, H = +d.height, fps = +(d.fps || 30), D = +d.duration;
  const k = Math.min(W, H) / 1080, NS = 'http://www.w3.org/2000/svg', PI2 = Math.PI * 2;
  const SAFE = { social: [288, 192, 672, 120], reels: [269, 65, 672, 65], tiktok: [240, 120, 660, 120],
    shorts: [288, 192, 672, 48], feed: [100, 80, 100, 80], none: [0, 0, 0, 0] };
  let s = d.safe === 'broadcast' ? [H * 0.05, W * 0.05, H * 0.05, W * 0.05]
    : SAFE[d.safe] ? SAFE[d.safe].map(v => v * k)
    : d.safe ? d.safe.split(',').map(Number) : SAFE.none;
  s = s.map(Math.round);
  ['t', 'r', 'b', 'l'].forEach((n, i) => stage.style.setProperty('--safe-' + n, s[i] + 'px'));
  stage.style.setProperty('--u', W / 100 + 'px');
  const css = document.createElement('style');            // базовые правила живут здесь, чтобы их нельзя было потерять
  css.textContent = `html,body{margin:0;background:transparent}
#stage{position:relative;overflow:hidden;width:${W}px;height:${H}px}
.bg,.scene,.fill{position:absolute;inset:0}
.safe{position:absolute;top:var(--safe-t);right:var(--safe-r);bottom:var(--safe-b);left:var(--safe-l);display:flex;flex-direction:column;justify-content:center}
.win{position:absolute;overflow:hidden}
img[data-footage]{position:absolute;inset:0;width:100%;height:100%;object-fit:cover}
svg.mo-ov{position:absolute;left:0;top:0;overflow:visible;pointer-events:none}
.mo-l-mask{overflow-clip-margin:.2em}
.mo-cap{position:absolute;inset:0;display:flex;justify-content:center;align-items:center}
.mo-capl{display:flex;flex-wrap:wrap;gap:0 .32em;justify-content:center;text-align:center}
.mo-capl span{display:inline-block}
.mo-cur{position:absolute;width:.09em;height:1em;background:currentColor}
.mo-icon{display:block;width:100%;height:100%;overflow:visible}`;
  document.head.prepend(css);
  gsap.registerPlugin(...['CustomEase', 'SplitText', 'DrawSVGPlugin', 'MotionPathPlugin', 'MorphSVGPlugin'].map(n => window[n]).filter(Boolean));
  if (window.CustomEase) [['swift', '0.8,0,0.1,1'], ['glide', '0.22,1,0.36,1'], ['punch', '0.34,1.56,0.64,1'], ['ant', '0.36,-0.3,0.12,1']].forEach(e => CustomEase.create(e[0], e[1]));

  const q = t => (typeof t === 'string' ? stage.querySelector(t) : t);
  const qa = t => (typeof t === 'string' ? [...stage.querySelectorAll(t)] : t.nodeType ? [t] : [...t]);
  const tl = gsap.timeline({ paused: true, defaults: { ease: 'expo.out', duration: 0.5 } });
  const hooks = [], sfxList = [], marks = [], nudges = new Map();
  let fmap = {};
  const mk = (tag, attrs, parent) => { const e = document.createElementNS(NS, tag); for (const a in attrs) e.setAttribute(a, attrs[a]); if (parent) parent.appendChild(e); return e; };
  const box = (el, root, tight) => {                // рамка элемента в координатах кадра (или root) в покое: трансформации на время замера снимаются
    const st = [], off = e => { if (e.style && e.style.transform && e.style.transform !== 'none') { st.push([e, e.style.transform]); e.style.transform = 'none'; } };
    for (let e = el; e && e !== document.body; e = e.parentElement) off(e);
    if (tight) el.querySelectorAll('*').forEach(off);
    let r = el.getBoundingClientRect();
    if (tight && getComputedStyle(el).display !== 'inline') { const g = document.createRange(); g.selectNodeContents(el); const t = g.getBoundingClientRect(); if (t.width > 1) r = t; }
    const o = (root || stage).getBoundingClientRect();
    st.forEach(v => { v[0].style.transform = v[1]; });
    return { x: r.left - o.left, y: r.top - o.top, w: r.width, h: r.height, cx: r.left - o.left + r.width / 2, cy: r.top - o.top + r.height / 2 };
  };
  const pre = new WeakMap(), preK = {}; let kseq = 0;  // рамки, снятые до разбивки текста: SplitText режет вложенные элементы
  const rbox = (t, host) => { if (isBox(t)) return Object.assign({ cx: t.x + t.w / 2, cy: t.y + t.h / 2 }, t); const el = q(t), a = preK[el.dataset.moK] || pre.get(el) || box(el, null, true), h = host && host !== stage ? box(host) : { x: 0, y: 0 };
    return { x: a.x - h.x, y: a.y - h.y, w: a.w, h: a.h, cx: a.cx - h.x, cy: a.cy - h.y }; };
  const scenes = [], sceneAt = t => { const s2 = scenes.filter(v => v[1] <= t + 0.01).pop(); return s2 ? s2[0] : stage; };
  const isBox = t => t && typeof t === 'object' && !t.nodeType && !Array.isArray(t) && 'w' in t;
  const hostOf = el => (el && el.closest && el.closest('[data-layer],.scene')) || stage;
  const hostFor = (t, o) => (o.host ? q(o.host) : Array.isArray(t) || isBox(t) ? sceneAt(o.at || 0) : hostOf(q(t)));   // точки и рамки живут в сцене, которая на экране в момент at
  const layer = host => host.querySelector(':scope > svg.mo-ov') || mk('svg', { class: 'mo-ov', width: W, height: H, 'data-decor': '', style: host === stage ? 'z-index:800' : '' }, host);
  const spot = (v, host) => {                        // точка: [x,y] в px (или долях кадра, если оба ≤ 1) либо центр элемента
    if (Array.isArray(v)) return v[0] <= 1 && v[1] <= 1 ? [v[0] * W, v[1] * H] : v;
    const b = rbox(v, host); return [b.cx, b.cy];
  };
  const sfx = (o, def, t, x) => { const n = o.sfx === undefined ? def : o.sfx; if (n) M.sfx(n, t, x); };
  const move = (o, a, dur, size, land, g) => {       // звук перемещения из двух слоёв: подъезд с пиком на середине хода и низкий удар в момент остановки
    if (o.sfx === false) return; if (typeof o.sfx === 'string') return M.sfx(o.sfx, a + dur * 0.5);
    const nm = size > 1 ? 'sweep' : size ? 'whoosh' : 'swish', len = size > 1 ? 0.95 : size ? 0.55 : 0.26;
    M.sfx(nm, a + dur * 0.5, { rate: Math.max(0.75, Math.min(1.35, len / (dur * 1.15))), gain: g || 0 });
    if (land) M.sfx(size ? 'hit' : 'thock', a + dur * land, { gain: (g || 0) + (size > 1 ? 0 : size ? -4 : -2) });
  };
  const accent = o => o.color || 'var(--accent)', hf = 0.5 / fps;   // линия становится видимой на полкадра позже старта: нулевая длина даёт точку
  const split = (el, chars) => {
    let sp = el.__sp;
    if (!sp) { pre.set(el, box(el, null, true)); el.querySelectorAll('*').forEach(c => { c.dataset.moK = ++kseq; preK[kseq] = box(c, null, true); });
      sp = el.__sp = SplitText.create(el, { type: 'lines,words', mask: 'lines', linesClass: 'mo-l', wordsClass: 'mo-w' }); gsap.set(sp.masks, { overflow: 'visible' }); }
    if (chars && !sp.ch) sp.ch = SplitText.create(sp.words, { type: 'chars', charsClass: 'mo-c' }).chars;
    return sp;
  };
  const IN = { rise: { yPercent: 118 }, drop: { yPercent: -118 }, fade: { autoAlpha: 0, y: '0.4em' }, slide: { autoAlpha: 0, x: '-0.7em' },
    pop: { autoAlpha: 0, scale: 0.2 }, blur: { autoAlpha: 0, scale: 1.25, filter: 'blur(16px)' }, flip: { autoAlpha: 0, rotationX: -92 } };
  const REST = { yPercent: 0, y: 0, x: 0, autoAlpha: 1, scale: 1, rotationX: 0, filter: 'blur(0px)' };
  const rest = from => { const r = {}; for (const a in from) r[a] = REST[a]; return r; };
  const vis = (el, a, b) => { gsap.set(el, { visibility: a > 0 ? 'hidden' : 'inherit' }); if (a > 0) tl.set(el, { visibility: 'inherit' }, a); if (b != null && b < D) tl.set(el, { visibility: 'hidden' }, b); };

  const M = window.M = {
    W, H, fps, D, tl, k, safe: { top: s[0], right: s[1], bottom: s[2], left: s[3] }, data: { icons: {}, faces: {}, words: {} },
    f: n => n / fps,                               // кадры → секунды
    snap: t => Math.round(t * fps) / fps,          // время на сетку кадров
    box,
    rand(seed) {                                   // детерминированный генератор вместо Math.random
      let a = (seed || 1) >>> 0;
      return () => { a = a + 0x6D2B79F5 | 0; let t = Math.imul(a ^ a >>> 15, 1 | a);
        t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t; return ((t ^ t >>> 14) >>> 0) / 4294967296; };
    },
    scene(target, start, end) {                    // сцена видна только на [start, end)
      const el = q(target);
      vis(el, start, end); scenes.push([el, start]); scenes.sort((x, y) => x[1] - y[1]);
      tl.addLabel(el.id || 'scene@' + start, start);
      return el;
    },
    seq(list) {                                    // [['#s1',0], ['#s2',3.2,'push-left',{dur:.6}], …] — сцены и переходы между ними
      const ov = () => { const e = document.createElement('div'); e.className = 'fill'; e.setAttribute('data-decor', ''); e.style.zIndex = 900; e.style.pointerEvents = 'none'; stage.appendChild(e); return e; };
      list.forEach((it, i) => {
        const el = q(it[0]), t = it[1], type = it[2] || 'cut', o = it[3] || {}, nx = list[i + 1], prev = i ? q(list[i - 1][0]) : null;
        const cover = /^(bars|diag|circle)/.test(type), dur = type === 'cut' ? 0 : o.dur || (cover ? 0.8 : 0.65), a = t - dur / 2, dir = (type.split('-')[1] || 'left');
        const nd = nx && nx[2] && nx[2] !== 'cut' ? (nx[3] && nx[3].dur) || (/^(bars|diag|circle)/.test(nx[2]) ? 0.8 : 0.65) : 0;
        if (getComputedStyle(el).zIndex === 'auto') el.style.zIndex = i + 1;
        vis(el, !i || cover ? t : type === 'zoom' ? a + dur * 0.4 : a, nx ? nx[1] + (/^(bars|diag|circle)/.test(nx[2] || '') ? 0 : nd / 2) : null);
        tl.addLabel(el.id || 'scene@' + t, t); marks.push(t); scenes.push([el, t]); scenes.sort((x, y) => x[1] - y[1]);
        if (!i || type === 'cut') return;
        const ax = /up|down/.test(dir) ? 'yPercent' : 'xPercent', sg = /left|up/.test(dir) ? 1 : -1, ez = o.ease || 'swift';
        if (type.startsWith('push')) { tl.fromTo(el, { [ax]: 100 * sg }, { [ax]: 0, duration: dur, ease: ez }, a).to(prev, { [ax]: -100 * sg, duration: dur, ease: ez }, a); move(o, a, dur, 1, 0.8); }
        else if (type.startsWith('slide')) { tl.fromTo(el, { [ax]: 100 * sg }, { [ax]: 0, duration: dur, ease: ez }, a).to(prev, { [ax]: -24 * sg, duration: dur, ease: ez }, a); move(o, a, dur, 1, 0.8); }
        else if (type === 'zoom') { tl.to(prev, { scale: 1.7, opacity: 0, duration: dur * 0.5, ease: 'power3.in' }, a).fromTo(el, { scale: 0.55, opacity: 0 }, { scale: 1, opacity: 1, duration: dur * 0.6, ease: 'expo.out' }, a + dur * 0.4); move(o, a, dur, 1, 0.62, 2); }
        else if (type === 'iris') { const x = (o.x == null ? 0.5 : o.x) * 100, y = (o.y == null ? 0.5 : o.y) * 100; tl.fromTo(el, { clipPath: `circle(0% at ${x}% ${y}%)` }, { clipPath: `circle(150% at ${x}% ${y}%)`, duration: dur, ease: o.ease || 'power3.inOut' }, a).set(el, { clipPath: 'none' }, a + dur); sfx(o, 'swell', t); sfx(o, 'shimmer', a, { gain: -3 }); }
        else if (type.startsWith('wipe')) { const c = { left: ['0 0 0 100%'], right: ['0 100% 0 0'], up: ['100% 0 0 0'], down: ['0 0 100% 0'] }[dir][0]; tl.fromTo(el, { clipPath: `inset(${c})` }, { clipPath: 'inset(0 0 0 0%)', duration: dur, ease: ez }, a).set(el, { clipPath: 'none' }, a + dur); move(o, a, dur, 1, 0.8); }
        else if (cover) {
          const host = ov(), cols = o.colors || ['var(--accent)', 'var(--fg)'], h = dur / 2;
          if (type.startsWith('bars')) { const n = o.n || 5, vert = /up|down/.test(dir);
            for (let j = 0; j < n; j++) { const b = document.createElement('div'); b.style.cssText = `position:absolute;background:${cols[j % cols.length]};` + (vert ? `top:0;bottom:0;left:${j * 100 / n}%;width:${100 / n + 0.2}%` : `left:0;right:0;top:${j * 100 / n}%;height:${100 / n + 0.2}%`); host.appendChild(b);
              const p = vert ? 'scaleY' : 'scaleX', o1 = vert ? (sg > 0 ? '50% 100%' : '50% 0%') : (sg > 0 ? '100% 50%' : '0% 50%'), o2 = vert ? (sg > 0 ? '50% 0%' : '50% 100%') : (sg > 0 ? '0% 50%' : '100% 50%'), st = j * h * 0.4 / n;
              tl.fromTo(b, { [p]: 0, transformOrigin: o1 }, { [p]: 1, duration: h * 0.6, ease: 'power3.inOut' }, a + st).set(b, { transformOrigin: o2 }, t + st).to(b, { [p]: 0, duration: h * 0.6, ease: 'power3.inOut' }, t + st + 0.001); } }
          else if (type.startsWith('diag')) cols.slice(0, 2).forEach((c, j) => { const b = document.createElement('div'); b.style.cssText = `position:absolute;top:-10%;bottom:-10%;left:-40%;width:180%;background:${c};transform:skewX(-14deg)`; host.appendChild(b);
            tl.fromTo(b, { xPercent: 112 * sg }, { xPercent: -112 * sg, duration: dur * (1 - j * 0.12), ease: 'power2.inOut' }, a + j * dur * 0.12); });
          else { const x = (o.x == null ? 0.5 : o.x) * 100, y = (o.y == null ? 0.5 : o.y) * 100; host.style.background = cols[0];
            tl.fromTo(host, { clipPath: `circle(0% at ${x}% ${y}%)` }, { clipPath: `circle(150% at ${x}% ${y}%)`, duration: h, ease: 'power3.in' }, a).to(host, { autoAlpha: 0, duration: h * 0.7, ease: 'power2.out' }, t); }
          move(o, a, dur, 1, 0.52, 1);
        }
      });
    },
    text(target, o = {}) {                         // вход текста: {at, in: rise|drop|fade|slide|pop|blur|flip|type, by: lines|words|chars, stagger, dur, ease, cps, cursor}
      const kind = o.in || 'rise', by = o.by || (kind === 'type' ? 'chars' : 'words'); let end = o.at || 0;
      qa(target).forEach((el, n) => {
        const sp = split(el, by === 'chars'), parts = by === 'chars' ? sp.ch : by === 'lines' ? sp.lines : sp.words, t0 = (o.at || 0) + n * (o.gap == null ? 0.12 : o.gap);
        if (kind === 'type') {
          const cps = o.cps || 22; gsap.set(parts, { autoAlpha: 0 });
          parts.forEach((c, i) => { tl.set(c, { autoAlpha: 1 }, t0 + i / cps); if (cps <= 16 || i % 2 === 0) sfx(o, 'key', t0 + i / cps); if (i % 6 === 0) marks.push(t0 + i / cps); });
          end = Math.max(end, t0 + parts.length / cps);
          if (o.cursor !== false) { const cur = document.createElement('span'); cur.className = 'mo-cur'; cur.setAttribute('data-decor', ''); if (getComputedStyle(el).position === 'static') el.style.position = 'relative'; el.appendChild(cur);
            const tE = t0 + parts.length / cps + (o.cursorHold == null ? 0.8 : o.cursorHold);
            hooks.push(t => { const i = Math.min(parts.length - 1, Math.floor((t - t0) * cps)), on = t >= t0 - 0.3 && t < tE && (t < t0 + parts.length / cps || Math.floor(t * 3) % 2 === 0);
              cur.style.visibility = on ? 'visible' : 'hidden'; if (!on) return; const c = parts[Math.max(0, i)], b = box(c, el); cur.style.left = (i < 0 ? b.x : b.x + b.w) + 'px'; cur.style.top = b.y + (b.h - cur.offsetHeight) / 2 + 'px'; }); }
          return;
        }
        const from = Object.assign({}, IN[kind] || IN.rise), masked = kind === 'rise' || kind === 'drop';
        if (kind === 'pop') from.transformOrigin = '50% 85%'; if (kind === 'flip') { from.transformOrigin = '50% 100%'; from.transformPerspective = 700; }
        const dur = o.dur || (kind === 'pop' ? 0.45 : by === 'chars' ? 0.45 : 0.6), stg = o.stagger == null ? (by === 'chars' ? 0.018 : by === 'lines' ? 0.09 : 0.05) : o.stagger;
        if (masked) { gsap.set(sp.masks, { overflow: 'clip' }); }
        tl.fromTo(parts, from, Object.assign(rest(from), { duration: dur, ease: o.ease || (kind === 'pop' ? 'back.out(2.2)' : 'expo.out'), stagger: stg }), t0);
        const tE = t0 + dur + stg * (parts.length - 1); end = Math.max(end, tE);
        if (masked) tl.set(sp.masks, { overflow: 'visible' }, tE + 0.01);
        if (kind === 'pop') { parts.slice(0, 8).forEach((p, i) => sfx(o, 'pop', t0 + i * stg, { rate: 0.9 + 0.04 * i })); if (parts.length > 2) sfx(o, 'hit', t0 + stg * Math.min(parts.length, 8) + dur * 0.25, { gain: -5 }); }
        else if (masked || kind === 'slide' || kind === 'flip') { sfx(o, 'swish', t0 + dur * 0.2, { gain: -2 }); sp.lines.forEach((l, i) => sfx(o, 'tap', t0 + i * 0.08 + dur * 0.3)); }
        else if (kind === 'blur') sfx(o, 'swell', t0 + dur * 0.5, { gain: -5 });
      });
      return end;
    },
    textOut(target, o = {}) {                      // выход текста: {at, out: rise|drop|fade|pop|blur, by, dur}
      const kind = o.out || 'rise', by = o.by || 'words';
      qa(target).forEach(el => {
        const sp = split(el, by === 'chars'), parts = by === 'chars' ? sp.ch : by === 'lines' ? sp.lines : sp.words, t0 = o.at || 0;
        const to = { rise: { yPercent: -118 }, drop: { yPercent: 118 }, fade: { autoAlpha: 0, y: '-0.3em' }, pop: { autoAlpha: 0, scale: 0.2 }, blur: { autoAlpha: 0, filter: 'blur(14px)' } }[kind];
        if (kind === 'rise' || kind === 'drop') tl.set(sp.masks, { overflow: 'clip' }, t0 - 0.01);
        tl.to(parts, Object.assign({ duration: o.dur || 0.3, ease: o.ease || 'power3.in', stagger: o.stagger == null ? 0.02 : o.stagger }, to), t0);
      });
    },
    show(target, o = {}) {                         // вход блоков целиком: {at, in: pop|rise|drop|slide|fade|wipe|grow|blur, stagger, dur, ease}
      const kind = o.in || 'pop', els = qa(target), at = o.at || 0, u = 70 * k;
      const from = { pop: { autoAlpha: 0, scale: 0.3 }, rise: { autoAlpha: 0, y: u }, drop: { autoAlpha: 0, y: -u }, slide: { autoAlpha: 0, x: -u * 1.4 }, fade: { autoAlpha: 0 }, blur: { autoAlpha: 0, scale: 1.2, filter: 'blur(18px)' },
        wipe: { clipPath: 'inset(0 100% 0 0)' }, grow: { scaleX: 0, transformOrigin: '0% 50%' } }[kind];
      const to = kind === 'wipe' ? { clipPath: 'inset(0 0% 0 0)' } : kind === 'grow' ? { scaleX: 1 } : rest(from), stg = o.stagger == null ? 0.07 : o.stagger;
      if (kind === 'wipe') { gsap.set(els, { autoAlpha: 0 }); tl.set(els, { autoAlpha: 1 }, at); }
      tl.fromTo(els, from, Object.assign(to, { duration: o.dur || (kind === 'pop' ? 0.5 : 0.6), ease: o.ease || (kind === 'pop' ? 'back.out(1.9)' : kind === 'wipe' || kind === 'grow' ? 'power3.inOut' : 'expo.out'), stagger: stg }), at);
      if (kind === 'pop') els.slice(0, 8).forEach((e, i) => sfx(o, 'pop', at + i * stg, { rate: 0.9 + 0.05 * i }));
      else { sfx(o, 'swish', at + 0.12, { gain: -3 }); els.slice(0, 8).forEach((e, i) => sfx(o, kind === 'wipe' || kind === 'grow' ? 'tick' : 'tap', at + i * stg + (o.dur || 0.6) * 0.3)); }
      return at + (o.dur || 0.6) + stg * (els.length - 1);
    },
    hide(target, o = {}) {                         // выход блоков: {at, out: pop|rise|drop|slide|fade|wipe, stagger, dur}
      const kind = o.out || 'fade', u = 70 * k, to = { pop: { autoAlpha: 0, scale: 0.3 }, rise: { autoAlpha: 0, y: -u }, drop: { autoAlpha: 0, y: u }, slide: { autoAlpha: 0, x: u * 1.4 }, fade: { autoAlpha: 0 }, wipe: { clipPath: 'inset(0 0 0 100%)' } }[kind];
      tl.to(qa(target), Object.assign({ duration: o.dur || 0.3, ease: o.ease || 'power3.in', stagger: o.stagger == null ? 0.03 : o.stagger }, to), o.at || 0);
    },
    captions(words, o = {}) {                      // субтитры по словам: {into, mode: karaoke|pop, max, offset, from, to, hi, upper, hold}
      const host = q(o.into), off = o.offset || 0, max = o.max || 16, mode = o.mode || 'karaoke', groups = [], base = getComputedStyle(host).color;
      const ws = words.filter(w => w.s >= (o.from == null ? -1e9 : o.from) && w.s < (o.to == null ? 1e9 : o.to)); let cur = [], len = 0;
      ws.forEach((w, i) => { const nx = ws[i + 1]; cur.push(w); len += w.w.length + 1;
        if (!nx || /[.!?…]$/.test(w.w) || nx.s - w.e > 0.4 || len + nx.w.length > max) { groups.push(cur); cur = []; len = 0; } });
      groups.forEach((g, gi) => {
        const line = document.createElement('div'), nx = groups[gi + 1], t0 = g[0].s + off, t1 = Math.min(g[g.length - 1].e + off + (o.hold == null ? 0.3 : o.hold), nx ? nx[0].s + off : D);
        line.className = 'mo-cap'; line.setAttribute('data-cap', ''); host.appendChild(line); const inner = document.createElement('div'); inner.className = 'mo-capl'; line.appendChild(inner);
        const sps = g.map(w => { const sp = document.createElement('span'), tx = w.w.replace(/[.,;:…]+$/, ''); sp.textContent = o.upper ? tx.toUpperCase() : tx; inner.appendChild(sp); return sp; });
        vis(line, Math.max(0.001, t0), t1); g.forEach(w => marks.push(w.s + off));
        if (mode === 'pop') g.forEach((w, i) => tl.fromTo(sps[i], { autoAlpha: 0, scale: 0.5, transformOrigin: '50% 80%' }, { autoAlpha: 1, scale: 1, duration: 0.22, ease: 'back.out(2.4)' }, Math.max(0.001, w.s + off)));
        else { tl.fromTo(inner, { scale: 0.86 }, { scale: 1, duration: 0.18, ease: 'back.out(2)' }, Math.max(0.001, t0));
          g.forEach((w, i) => { const a = Math.max(0.002, w.s + off), b = Math.max(a + 0.08, (g[i + 1] ? g[i + 1].s : w.e + 0.2) + off);
            tl.to(sps[i], { color: o.hi || 'var(--accent)', scale: 1.06, duration: 0.09, ease: 'power2.out' }, a); if (!o.keep) tl.to(sps[i], { color: base, scale: 1, duration: 0.12, ease: 'power2.out' }, b); }); }
      });
      return groups;
    },
    count(target, o) {                             // счётчик: {from,to,at,dur,ease,decimals,prefix,suffix}
      const el = q(target), p = { v: o.from || 0 }, dec = o.decimals || 0;
      const fmt = o.format || (v => {
        const [a, b] = Math.abs(v).toFixed(dec).split('.');
        return (o.prefix || '') + (v < 0 ? '−' : '') + a.replace(/\B(?=(\d{3})+(?!\d))/g, String.fromCharCode(160)) + (b ? ',' + b : '') + (o.suffix || '');
      });
      el.style.fontVariantNumeric = 'tabular-nums';
      el.textContent = fmt(o.to);                  // до первого кадра в элементе стоит итоговое число: по нему меряются акценты вокруг
      const dur = o.dur || 1, at = o.at || 0;
      tl.to(p, { v: o.to, duration: dur, ease: o.ease || 'power3.out' }, at);
      const ez = gsap.parseEase(o.ease || 'power3.out'), v0 = o.from || 0;
      hooks.push(t => { const u = Math.max(0, Math.min(1, (Math.floor(t * fps + 1e-6) / fps - at) / dur)), s2 = fmt(v0 + (o.to - v0) * ez(u)); if (el.textContent !== s2) el.textContent = s2; });   // цифра — чистая функция от номера кадра: в подкадрах размытия и на любой странице рендера она одна и та же
      for (let i = 0, n = Math.min(18, Math.round(dur * 14)); i < n; i++) sfx(o, 'tick', at + dur * (1 - Math.pow(1 - i / n, 2.2)) * 0.9, { rate: 0.8 + 0.5 * i / n, gain: -5 + 5 * i / n });   // тики ускоряются и поднимаются, число «приземляется» звоном
      if (o.sfx === undefined) { M.sfx('thock', at + dur * 0.93); M.sfx('chime', at + dur * 0.93, { gain: -2 }); }
    },
    draw: {                                        // рисованные акценты: линия прорисовывается поверх вёрстки
      path(dd, o = {}) {
        const host = o.host ? q(o.host) : sceneAt(o.at || 0), p = mk('path', { d: dd, fill: 'none', stroke: accent(o), 'stroke-width': o.width || 6 * k, 'stroke-linecap': 'round', 'stroke-linejoin': 'round' }, layer(host)), at = o.at || 0, dur = o.dur || 0.45;
        tl.fromTo(p, { drawSVG: '0% 0%' }, { drawSVG: '0% 100%', duration: dur, ease: o.ease || 'power2.inOut' }, at);
        gsap.set(p, { autoAlpha: 0 }); tl.set(p, { autoAlpha: 1 }, at + hf);
        sfx(o, 'scratch', at);
        return { el: p, end: at + dur, out(t, dd2) { tl.to(p, { drawSVG: '100% 100%', duration: dd2 || 0.3, ease: 'power2.in' }, t).set(p, { autoAlpha: 0 }, t + (dd2 || 0.3)); return this; } };
      },
      underline(t, o = {}) {
        const host = hostFor(t, o), b = rbox(t, host), r = M.rand(o.seed || b.x * 7 + b.y), sw = o.width || Math.max(5 * k, b.h * 0.07), y = b.y + b.h * (o.dy == null ? (o.style === 'wave' ? 1.07 : 1) : o.dy), x1 = b.x - b.w * 0.02, x2 = b.x + b.w * 1.02, j = a => (r() - 0.5) * 2 * a;
        let dd = `M${x1} ${y + j(sw * 0.4)}`;
        if (o.style === 'wave') { const n = Math.max(4, Math.round(b.w / (sw * 5))); for (let i = 1; i <= n; i++) { const xx = x1 + (x2 - x1) * i / n; dd += ` Q${xx - (x2 - x1) / n / 2} ${y + (i % 2 ? -1 : 1) * sw * 1.8} ${xx} ${y}`; } }
        else if (o.style === 'scribble') { const n = 5; for (let i = 1; i <= n; i++) dd += ` L${i % 2 ? x2 - j(b.w * 0.04) - b.w * 0.03 : x1 + b.w * 0.05 + j(b.w * 0.04)} ${y + i * sw * 0.55 + j(sw * 0.3)}`; }
        else dd += ` Q${(x1 + x2) / 2 + j(b.w * 0.12)} ${y - sw * 0.9 + j(sw)} ${x2} ${y + j(sw * 0.5)}`;
        return M.draw.path(dd, Object.assign({ host, width: sw }, o));
      },
      strike(t, o = {}) { return M.draw.underline(t, Object.assign({ dy: 0.52 }, o)); },
      circle(t, o = {}) {                          // обвести от руки
        const host = hostFor(t, o), b = rbox(t, host), r = M.rand(o.seed || b.x + b.y * 3), pad = o.pad == null ? b.h * 0.28 : o.pad, rx = b.w / 2 + pad * 1.4, ry = b.h / 2 + pad, a0 = -2.4 + r() * 0.5, pts = [];
        for (let i = 0; i <= 26; i++) { const u = i / 26, a = a0 + u * PI2 * 1.09, g = 1 + u * 0.07 + (r() - 0.5) * 0.03; pts.push([b.cx + Math.cos(a) * rx * g, b.cy + Math.sin(a) * ry * g]); }
        return M.draw.path(`M${pts[0][0].toFixed(1)} ${pts[0][1].toFixed(1)}` + pts.slice(1, -1).map((p, i) => ` Q${p[0].toFixed(1)} ${p[1].toFixed(1)} ${((p[0] + pts[i + 2][0]) / 2).toFixed(1)} ${((p[1] + pts[i + 2][1]) / 2).toFixed(1)}`).join(''), Object.assign({ host, dur: 0.55 }, o));
      },
      brackets(t, o = {}) {                        // уголки-рамка вокруг элемента
        const host = hostFor(t, o), b = rbox(t, host), p = o.pad == null ? 14 * k : o.pad, L = o.len || Math.min(b.w, b.h) * 0.22, x1 = b.x - p, y1 = b.y - p, x2 = b.x + b.w + p, y2 = b.y + b.h + p;
        const c = [[x1, y1 + L, x1, y1, x1 + L, y1], [x2 - L, y1, x2, y1, x2, y1 + L], [x2, y2 - L, x2, y2, x2 - L, y2], [x1 + L, y2, x1, y2, x1, y2 - L]];
        return c.map((v, i) => M.draw.path(`M${v[0]} ${v[1]} L${v[2]} ${v[3]} L${v[4]} ${v[5]}`, Object.assign({ host, dur: 0.3 }, o, { at: (o.at || 0) + i * 0.05, sfx: i ? false : o.sfx })));
      },
      arrow(a, b2, o = {}) {                       // изогнутая стрелка от a к b (точка или элемент)
        const host = hostFor(a, o), A = spot(a, host), B = spot(b2, host), dx = B[0] - A[0], dy = B[1] - A[1], len = Math.hypot(dx, dy), g = o.gap == null ? 0 : o.gap;
        const ux = dx / len, uy = dy / len, S = [A[0] + ux * g, A[1] + uy * g], E = [B[0] - ux * g, B[1] - uy * g], bend = (o.bend == null ? 0.22 : o.bend) * len, C = [(S[0] + E[0]) / 2 - uy * bend, (S[1] + E[1]) / 2 + ux * bend];
        const tx = E[0] - C[0], ty = E[1] - C[1], tn = Math.hypot(tx, ty), hx = tx / tn, hy = ty / tn, hl = o.head || Math.max(18 * k, Math.min(len * 0.16, 46 * k)), rot = (c, s2) => [E[0] - (hx * c - hy * s2) * hl, E[1] - (hy * c + hx * s2) * hl], h1 = rot(0.85, 0.5), h2 = rot(0.85, -0.5);
        const sh = M.draw.path(`M${S[0]} ${S[1]} Q${C[0]} ${C[1]} ${E[0]} ${E[1]}`, Object.assign({ host }, o));
        M.draw.path(`M${h1[0]} ${h1[1]} L${E[0]} ${E[1]} L${h2[0]} ${h2[1]}`, Object.assign({ host }, o, { at: sh.end - 0.1, dur: 0.18, sfx: false }));
        return sh;
      },
      burst(c, o = {}) {                           // лучи-искры из точки
        const el = Array.isArray(c) ? null : c, host = hostFor(c, o), bb = el ? rbox(el, host) : null, P = el ? [bb.cx, bb.cy] : spot(c, host), n = o.n || (el ? 12 : 8), at = o.at || 0, dur = o.dur || 0.5, a0 = o.angle == null ? 0.26 : o.angle, sv = layer(host);
        const pad = o.pad == null ? 26 * k : o.pad, len = o.len || 70 * k, rx = el ? bb.w / 2 + pad : o.r1 || 40 * k, ry = el ? bb.h / 2 + pad : o.r1 || 40 * k;   // вокруг элемента лучи стартуют за его рамкой
        for (let i = 0; i < n; i++) { const a = a0 + i / n * PI2, cx = Math.cos(a), sy = Math.sin(a), e1 = 1 / Math.hypot(cx / rx, sy / ry), e2 = el ? e1 + len : o.r2 || 110 * k, p = mk('path', { d: `M${(P[0] + cx * e1).toFixed(1)} ${(P[1] + sy * e1).toFixed(1)} L${(P[0] + cx * e2).toFixed(1)} ${(P[1] + sy * e2).toFixed(1)}`, fill: 'none', stroke: accent(o), 'stroke-width': o.width || 6 * k, 'stroke-linecap': 'round' }, sv);
          gsap.set(p, { autoAlpha: 0 }); tl.set(p, { autoAlpha: 1 }, at + hf).fromTo(p, { drawSVG: '0% 0%' }, { drawSVG: '0% 100%', duration: dur * 0.45, ease: 'power2.out' }, at).to(p, { drawSVG: '100% 100%', duration: dur * 0.55, ease: 'power2.in' }, at + dur * 0.45).set(p, { autoAlpha: 0 }, at + dur); }
        sfx(o, 'pop', at); sfx(o, 'shimmer', at, { gain: -3 });
      },
      ring(c, o = {}) {                            // расходящееся кольцо
        const host = hostFor(c, o), P = spot(c, host), at = o.at || 0, e = mk('circle', { cx: P[0], cy: P[1], r: o.r1 || 10 * k, fill: 'none', stroke: accent(o), 'stroke-width': o.width || 10 * k }, layer(host));
        gsap.set(e, { autoAlpha: 0 }); tl.set(e, { autoAlpha: 1 }, at).to(e, { attr: { r: o.r2 || 160 * k, 'stroke-width': 0.1 }, duration: o.dur || 0.6, ease: 'expo.out' }, at).set(e, { autoAlpha: 0 }, at + (o.dur || 0.6));
        sfx(o, 'bubble', at);
      },
      confetti(c, o = {}) {                        // хлопушка из точки
        const host = hostFor(c, o), P = spot(c, host), n = o.n || 28, r = M.rand(o.seed || 11), at = o.at || 0, cols = o.colors || ['var(--accent)', 'var(--fg)'], sv = layer(host), dur = o.dur || 1.3;
        for (let i = 0; i < n; i++) { const a = -Math.PI / 2 + (r() - 0.5) * (o.spread || 2.2), v = (0.45 + r() * 0.55) * (o.power || 520 * k), w = (10 + r() * 14) * k, e = mk('rect', { x: P[0] - w / 2, y: P[1] - w / 4, width: w, height: w / 2, rx: 2, fill: cols[i % cols.length] }, sv);
          gsap.set(e, { autoAlpha: 0, transformOrigin: '50% 50%' }); tl.set(e, { autoAlpha: 1 }, at)
            .to(e, { x: Math.cos(a) * v, duration: dur, ease: 'power2.out' }, at).to(e, { y: Math.sin(a) * v, duration: dur * 0.4, ease: 'power2.out' }, at).to(e, { y: Math.sin(a) * v + (300 + r() * 500) * k, duration: dur * 0.6, ease: 'power2.in' }, at + dur * 0.4)
            .to(e, { rotation: (r() - 0.5) * 900, duration: dur, ease: 'none' }, at).to(e, { autoAlpha: 0, duration: 0.2 }, at + dur - 0.2); }
        sfx(o, 'pop', at); sfx(o, 'shimmer', at + 0.05);
      },
      highlight(t, o = {}) {                       // маркер за словом (фон проезжает под текстом)
        const e0 = q(t), at = o.at || 0, els = (e0.dataset.moK ? qa(`[data-mo-k="${e0.dataset.moK}"]`) : [e0]).filter(e => e.textContent.trim()), dur = (o.dur || 0.4) / els.length;   // после разбивки текста слово может лежать в нескольких кусках
        els.forEach((el, i) => { if (getComputedStyle(el).display === 'block') el.style.width = 'fit-content'; el.style.backgroundImage = `linear-gradient(${accent(o)},${accent(o)})`; el.style.backgroundRepeat = 'no-repeat'; el.style.backgroundPosition = '0 ' + (o.y || '60%');
          tl.fromTo(el, { backgroundSize: '0% ' + (o.h || '88%') }, { backgroundSize: '100% ' + (o.h || '88%'), duration: dur, ease: 'power3.inOut' }, at + i * dur); if (o.ink) tl.to(el, { color: o.ink, duration: 0.2 }, at + i * dur + 0.1); });
        sfx(o, 'swish', at + (o.dur || 0.4) * 0.5, { gain: -2 });
      }
    },
    icon(name, o = {}) {                           // иконка из набора (mo data --icons): {into, at, draw, color, stroke}
      const ic = M.data.icons[name], host = q(o.into); if (!ic) throw new Error('нет иконки «' + name + '» — добавь: mo data <dir> --icons ' + name);
      const sv = mk('svg', { class: 'mo-icon', viewBox: '0 0 24 24', fill: 'none', stroke: o.color || 'currentColor', 'stroke-width': o.stroke || 1.6, 'stroke-linecap': 'round', 'stroke-linejoin': 'round', 'data-decor': '' }, host);
      sv.innerHTML = ic; const ps = [...sv.querySelectorAll('path,circle,line,rect,polyline,ellipse')], at = o.at || 0, dur = o.draw == null ? 0.7 : o.draw;
      if (dur) { gsap.set(ps, { autoAlpha: 0 }); ps.forEach((p, i) => tl.set(p, { autoAlpha: 1 }, at + i * dur * 0.5 / ps.length + hf).fromTo(p, { drawSVG: '0% 0%' }, { drawSVG: '0% 100%', duration: dur * 0.75, ease: 'power2.inOut' }, at + i * dur * 0.5 / ps.length)); sfx(o, 'scratch', at); sfx(o, 'tick', at + dur, { gain: 3 }); }
      return sv;
    },
    win(target, o = {}) {                          // окно с видео: формы-пресеты и кадрирование по лицу. {face: M.data.faces['clip.mp4']}
      const el = q(target), img = el.querySelector('img[data-footage]'), S = M.safe, sw = W - S.left - S.right, sh = H - S.top - S.bottom, f = o.face, cur = {};
      const P = { full: [0, 0, W, H, 0], card: [S.left, S.top + sh * 0.19, sw, sh * 0.62, sw * 0.05], top: [S.left, S.top, sw, sh * 0.5, sw * 0.05], bottom: [S.left, S.top + sh * 0.5, sw, sh * 0.5, sw * 0.05],
        wide: [S.left, S.top + (sh - sw * 0.5625) / 2, sw, sw * 0.5625, sw * 0.04], circle: [S.left + sw * 0.14, S.top + sh * 0.04, sw * 0.72, sw * 0.72, sw * 0.36], pill: [S.left, S.top + (sh - sw * 0.44) / 2, sw, sw * 0.44, sw * 0.22],
        left: [S.left, S.top, sw * 0.47, sh, sw * 0.03], right: [S.left + sw * 0.53, S.top, sw * 0.47, sh, sw * 0.03],
        'pip-tl': [S.left, S.top, sw * 0.36, sw * 0.45, sw * 0.045], 'pip-tr': [S.left + sw * 0.64, S.top, sw * 0.36, sw * 0.45, sw * 0.045], 'pip-bl': [S.left, S.top + sh - sw * 0.45, sw * 0.36, sw * 0.45, sw * 0.045], 'pip-br': [S.left + sw * 0.64, S.top + sh - sw * 0.45, sw * 0.36, sw * 0.45, sw * 0.045] };
      const res = (sh2, ov) => { let b;
        if (typeof sh2 === 'string' && sh2.startsWith('off')) { const dr = sh2.split('-')[1] || 'left'; b = Object.assign({}, cur); if (dr === 'left') b.x = -cur.w * 1.15; if (dr === 'right') b.x = W + cur.w * 0.15; if (dr === 'up') b.y = -cur.h * 1.15; if (dr === 'down') b.y = H + cur.h * 0.15; }
        else { const p = typeof sh2 === 'string' ? P[sh2] : null; if (typeof sh2 === 'string' && !p) throw new Error('нет формы окна «' + sh2 + '»'); b = p ? { x: p[0], y: p[1], w: p[2], h: p[3], r: p[4], rot: 0 } : Object.assign({ rot: 0, r: 0 }, cur, sh2); }
        return Object.assign(b, ov); };
      const fit = b => { if (!f) return { objectPosition: '50% ' + (o.focusY == null ? 20 : o.focusY) + '%' };
        const sc = Math.max(b.w / f.W, b.h / f.H), ox = f.W * sc - b.w, oy = f.H * sc - b.h, top = Math.min(f.cy * sc - b.h * 0.42, (f.cy - f.h * 0.95) * sc - b.h * 0.07);
        return { objectPosition: (ox > 1 ? Math.max(0, Math.min(1, (f.cx * sc - b.w / 2) / ox)) * 100 : 50).toFixed(1) + '% ' + (oy > 1 ? Math.max(0, Math.min(1, top / oy)) * 100 : 50).toFixed(1) + '%' }; };
      const css2 = b => ({ left: b.x, top: b.y, width: b.w, height: b.h, borderRadius: b.r, rotation: b.rot || 0 });
      const api = { el, shapes: P, box: () => Object.assign({}, cur), shape: (sh2, ov) => res(sh2, ov),
        set(sh2, ov) { Object.assign(cur, res(sh2, ov)); gsap.set(el, css2(cur)); if (img) gsap.set(img, fit(cur)); return api; },
        to(sh2, at, o2 = {}) { const b = res(sh2, o2.box), dur = o2.dur || 0.7, ease = o2.ease || 'expo.inOut'; Object.assign(cur, b);
          tl.to(el, Object.assign({ duration: dur, ease }, css2(b)), at); if (img && !String(sh2).startsWith('off')) tl.to(img, Object.assign({ duration: dur, ease }, fit(b)), at);
          move(o2, at, dur, 1, String(sh2).startsWith('off') ? 0 : 0.64); return api; } };
      return api.set(o.shape || 'full');
    },
    camera(target) {                               // камера над большим холстом: {x,y} — точка холста в центре кадра
      const el = q(target), st = { x: W / 2, y: H / 2, zoom: 1, rot: 0 }, deps = [...el.querySelectorAll('[data-depth]')];
      el.style.transformOrigin = '0 0'; el.setAttribute('data-layer', ''); M.cam = true;
      hooks.push(() => { el.style.transform = `translate(${W / 2}px,${H / 2}px) scale(${st.zoom}) rotate(${st.rot}deg) translate(${-st.x}px,${-st.y}px)`;
        deps.forEach(c => { const dz = 1 - +c.dataset.depth; c.style.translate = `${(st.x - W / 2) * dz}px ${(st.y - H / 2) * dz}px`; }); });
      const api = { state: st,
        set(v) { Object.assign(st, v); return api; },
        to(v, at, o = {}) { tl.to(st, Object.assign({ duration: o.dur || 0.9, ease: o.ease || 'swift' }, v), at); move(o, at, o.dur || 0.9, 2, 0.72, -2); return api; },
        focus(t, at, o = {}) { const b = box(q(t), el), v = { x: b.cx, y: b.cy, zoom: o.zoom || Math.min(W / b.w, H / b.h) * (o.fill || 0.72), rot: o.rot || 0 }; return at == null ? api.set(v) : api.to(v, at, o); } };
      return api;
    },
    float(target, o = {}) {                        // живое покачивание на удержании (не для мелкого текста)
      qa(target).forEach((el, i) => { const r = M.rand((o.seed || 5) + i * 13), p = [r(), r(), r()], per = o.period || 4, ax = o.x == null ? 8 * k : o.x, ay = o.y == null ? 12 * k : o.y, ar = o.rot || 0;
        (nudges.get(el) || nudges.set(el, []).get(el)).push(t => [ax * Math.sin(PI2 * (t / per + p[0])), ay * Math.sin(PI2 * (t / (per * 0.73) + p[1])), ar * Math.sin(PI2 * (t / (per * 1.31) + p[2]))]); });
    },
    shake(target, o = {}) {                        // короткая встряска на акцент
      const at = o.at || 0, dur = o.dur || 0.35, amp = o.amp || 16 * k;
      qa(target).forEach(el => (nudges.get(el) || nudges.set(el, []).get(el)).push(t => { const u = (t - at) / dur; if (u < 0 || u > 1) return [0, 0, 0]; const dc = (1 - u) * (1 - u); return [amp * dc * Math.sin(u * 38), amp * 0.6 * dc * Math.cos(u * 31), 0]; }));
      sfx(o, 'boom', at, { gain: -4 });
    },
    grain(amount) { stage.dataset.grain = amount == null ? 0.12 : amount; },   // плёночное зерно: накладывается при сборке файла (в стоп-кадрах его нет)
    chart(target, o) {                             // {type: bars|line|donut, data: [{label,value}], at, dur, hi, unit}
      const el = q(target), data = o.data, at = o.at || 0, dur = o.dur || 0.9, max = Math.max(...data.map(x => x.value)), hi = o.hi == null ? data.findIndex(x => x.value === max) : o.hi, col = i => (i === hi ? accent(o) : o.dim || 'color-mix(in srgb, currentColor 32%, transparent)');
      if (o.type === 'donut') { const v = data[0].value / (o.total || 100), R = 42, sv = mk('svg', { viewBox: '0 0 100 100', class: 'mo-icon', 'data-decor': '' }, el);
        mk('circle', { cx: 50, cy: 50, r: R, fill: 'none', stroke: col(-1), 'stroke-width': o.width || 9 }, sv);
        const a = mk('circle', { cx: 50, cy: 50, r: R, fill: 'none', stroke: accent(o), 'stroke-width': o.width || 9, 'stroke-linecap': 'round', transform: 'rotate(-90 50 50)' }, sv);
        tl.fromTo(a, { drawSVG: '0% 0%' }, { drawSVG: '0% ' + v * 100 + '%', duration: dur, ease: 'power3.out' }, at); sfx(o, 'swell', at + dur * 0.5); return sv; }
      if (o.type === 'line') { const sv = mk('svg', { viewBox: '0 0 100 60', class: 'mo-icon', preserveAspectRatio: 'none', 'data-decor': '' }, el), min = o.min == null ? Math.min(...data.map(x => x.value)) * 0.8 : o.min;
        const pt = data.map((x, i) => [4 + 92 * i / (data.length - 1), 56 - 50 * (x.value - min) / (max - min)]), p = mk('path', { d: 'M' + pt.map(v => v.join(' ')).join(' L'), fill: 'none', stroke: accent(o), 'stroke-width': o.width || 1.6, 'stroke-linecap': 'round', 'stroke-linejoin': 'round', 'vector-effect': 'non-scaling-stroke' }, sv);
        if (o.width == null) p.setAttribute('stroke-width', 7 * k);
        tl.fromTo(p, { drawSVG: '0% 0%' }, { drawSVG: '0% 100%', duration: dur, ease: 'power2.inOut' }, at); sfx(o, 'scratch', at); return sv; }
      el.style.display = 'flex'; el.style.alignItems = 'flex-end'; el.style.gap = el.style.gap || '3%';
      data.forEach((x, i) => { const c = document.createElement('div'), b = document.createElement('div'), n = document.createElement('div'), l = document.createElement('div'), t0 = at + i * 0.09;
        c.style.cssText = 'flex:1;height:100%;display:flex;flex-direction:column;justify-content:flex-end;align-items:center;gap:.35em'; b.style.cssText = `width:100%;height:${Math.max(2, x.value / max * 72)}%;background:${col(i)};border-radius:${o.radius || '.25em .25em 0 0'};transform-origin:50% 100%`;
        n.textContent = '0'; l.textContent = x.label || ''; l.setAttribute('data-small', ''); n.setAttribute('data-small', ''); c.append(n, b, l); el.appendChild(c);
        tl.fromTo(b, { scaleY: 0 }, { scaleY: 1, duration: dur, ease: 'power3.out' }, t0).fromTo([n, l], { autoAlpha: 0 }, { autoAlpha: 1, duration: 0.3 }, t0);
        M.count(n, { to: x.value, at: t0, dur, suffix: o.unit || '', decimals: o.decimals || 0, sfx: false }); sfx(o, 'thock', t0 + dur * 0.5, { rate: 0.9 + 0.06 * i }); });
    },
    sfx(name, at, o) { sfxList.push(Object.assign({ name, at: Math.max(0, +at || 0) }, o)); },   // звук на таймлинии: {gain (дБ), rate, pan, anchor: start|peak|end}
    hit(at, size, o = {}) {                        // акцент из слоёв: 's' — щелчок с телом; 'm' — подъезд и низкий удар; 'l' — нарастание, большой удар с хвостом и звон
      const g = o.gain || 0;
      if (size === 'l') { M.sfx('riser', at, { gain: g }); M.sfx('boom', at, { gain: g }); M.sfx('chime', at + 0.04, { gain: g - 3 }); }
      else if (size === 's') M.sfx('thock', at, { gain: g });
      else { M.sfx('whoosh', at - 0.04, { gain: g }); M.sfx('hit', at, { gain: g }); }
    },
    ticks(at, o = {}) {                            // очередь щелчков: {n, gap, name, rise: false — ровная, gain}
      const n = o.n || 6, gap = o.gap || 0.05;
      for (let i = 0; i < n; i++) M.sfx(o.name || 'tick', at + i * gap, { rate: o.rise === false ? 1 : 0.85 + 0.5 * i / n, gain: (o.gain || 0) - (o.rise === false ? 0 : 5 * (1 - i / Math.max(1, n - 1))) });
      return at + n * gap;
    },
    onFrame(fn) { hooks.push(fn); },               // fn(t, frame) — для canvas и всего процедурного
    start(build) {                                 // грузит шрифты → строит таймлинию → открывает renderFrame
      const faces = [...document.fonts].map(f => f.load().catch(e => console.error('Шрифт не загрузился: ' + f.family)));
      Promise.all(faces).then(() => document.fonts.ready).then(() => { build(M); M.ready(); })
        .catch(e => { console.error('Ошибка сборки: ' + (e && e.stack || e)); window.__moError = String(e && e.message || e); });
    },
    ready() {
      const real = tl.duration();
      if (real < D) tl.set({}, {}, D);
      hooks.push(t => nudges.forEach((fns, el) => { let x = 0, y = 0, r = 0; fns.forEach(fn => { const v = fn(t); x += v[0]; y += v[1]; r += v[2]; }); el.style.translate = x.toFixed(2) + 'px ' + y.toFixed(2) + 'px'; el.style.rotate = r.toFixed(3) + 'deg'; }));
      nudges.forEach((fns, el) => gsap.getProperty(el, 'x'));   // GSAP запоминает трансформацию элемента до первого покачивания, иначе впечёт его в свой расчёт
      const foot = [...stage.querySelectorAll('img[data-footage]')];
      const num = (v, def) => (v == null || v === '' ? def : +v);
      const ev = marks.slice(); tl.getChildren(true, true, false).forEach(c => { if (c.duration() <= 0.02) return; let t = c.startTime(), p = c.parent; while (p && p !== tl) { t = p.startTime() + t / (p.timeScale() || 1); p = p.parent; } ev.push(+t.toFixed(3)); });
      window.__motion = {
        W, H, fps, D, tlDur: real, labels: tl.labels, safe: M.safe, loop: d.loop != null, loudness: num(d.loudness, -14),
        footage: foot.map((im, i) => ({ i, file: im.dataset.footage, in: num(im.dataset.in, 0), at: num(im.dataset.at, 0),
          dur: num(im.dataset.dur, null), w: im.offsetWidth, h: im.offsetHeight, over: num(im.dataset.oversize, 1),
          audio: im.dataset.audio != null, gain: num(im.dataset.gain, 0), duck: false, voice: im.dataset.voice != null })),
        audio: [...document.querySelectorAll('audio[src]')].map(a => ({ file: a.getAttribute('src'), at: num(a.dataset.at, 0),
          in: num(a.dataset.in, 0), dur: num(a.dataset.dur, null), gain: a.dataset.gain == null ? null : +a.dataset.gain, fadeIn: num(a.dataset.fadeIn, 0),
          fadeOut: num(a.dataset.fadeOut, 0), duck: a.dataset.duck != null, voice: a.dataset.voice != null })),
        sfx: sfxList, kit: d.sfx || '01', sfxGain: num(d.sfxGain, 0), under: num(d.musicUnder, 17), grain: () => num(stage.dataset.grain, 0), events: ev.sort((a, b) => a - b).filter((t, i, a) => !i || t - a[i - 1] > 0.1),
        camera: !!M.cam, stray: () => gsap.globalTimeline.getChildren(false, true, true).filter(c => c !== tl).length,
        setFootage(m) { fmap = m; }
      };
      window.renderFrame = async t => {
        t = Math.max(0, Math.min(D, t));
        nudges.forEach((fns, el) => { el.style.translate = 'none'; el.style.rotate = 'none'; });
        tl.seek(t, false);
        const fr = Math.round(t * fps);
        hooks.forEach(h => h(t, fr));
        await Promise.all(foot.map((im, i) => {
          const m = fmap[i];
          if (!m || getComputedStyle(im).visibility === 'hidden') return;
          const n = Math.max(0, Math.min(m.count - 1, Math.floor((t - m.at) * fps + 1e-6)));
          const src = m.dir + '/' + String(n + 1).padStart(6, '0') + '.jpg';
          if (im.getAttribute('src') === src) return;
          im.src = src;
          return im.decode().catch(() => {});
        }));
      };
      if (!window.__MO) {                          // открыли файл в обычном браузере — просто проигрываем
        const fit = () => { stage.style.transformOrigin = '0 0';
          stage.style.transform = 'scale(' + Math.min(innerWidth / W, innerHeight / H) + ')'; };
        fit(); addEventListener('resize', fit);
        tl.eventCallback('onUpdate', () => hooks.forEach(h => h(tl.time(), Math.round(tl.time() * fps))));
        tl.repeat(-1).play(0);
      }
      window.__moReady = true;
    }
  };
})();
````

<!-- file: template.html -->
````html
<!doctype html>
<html lang="ru">
<head>
<meta charset="utf-8">
<link rel="stylesheet" href="fonts/fonts.css">
<style>
  /* Токены стиля: всё, что повторяется в ролике, берётся отсюда. Базовые правила (#stage, .scene, .safe, .win, .fill) уже есть в motion.js */
  :root { --bg: #0B0B0C; --fg: #F4F1EA; --accent: #FF5A1F; --font-display: "Inter"; --font-text: "Inter"; }
  #stage { color: var(--fg); font-family: var(--font-text), sans-serif; }
  .bg { background: var(--bg); }
  h1 { font: 800 calc(var(--u) * 10)/1.05 var(--font-display), sans-serif; margin: 0; }
</style>
</head>
<body>
<div id="stage" data-width="{{W}}" data-height="{{H}}" data-fps="{{FPS}}" data-duration="{{DUR}}" data-safe="{{SAFE}}" data-sfx="01">
  <div class="bg" data-bg></div>
  <!-- Окно с видео: <div class="win" id="win"><img data-footage="media/clip.mp4" data-in="0" data-at="0" data-audio data-voice></div> -->
  <section class="scene" id="s1">
    <div class="safe"><h1>Заголовок</h1></div>
  </section>
</div>
<!-- Музыка: <audio src="media/music.mp3" data-at="0" data-in="0" data-fade-out="1.5"></audio> -->
<script src="lib/gsap.min.js"></script>
<script src="lib/CustomEase.min.js"></script>
<script src="lib/SplitText.min.js"></script>
<script src="lib/DrawSVGPlugin.min.js"></script>
<script src="lib/MotionPathPlugin.min.js"></script>
<script src="lib/MorphSVGPlugin.min.js"></script>
<script src="motion.js"></script>
<script src="data.js"></script>
<script>
M.start(({ tl, f, D }) => {
  M.seq([['#s1', 0]]);
  M.text('#s1 h1', { at: 0.15, in: 'rise' });
});
</script>
</body>
</html>
````

<!-- file: faces.py -->
````python
# python3 faces.py <видео> [шаг_с] > faces.json — рамки лиц в пикселях исходника (OpenCV, каскад Хаара)
import sys, json, subprocess
import numpy as np, cv2
f, step = sys.argv[1], float(sys.argv[2]) if len(sys.argv) > 2 else 0.1
pr = json.loads(subprocess.run(['ffprobe','-v','error','-select_streams','v','-show_streams','-of','json',f],capture_output=True,text=True).stdout)['streams'][0]
W, H = pr['width'], pr['height']
for sd in pr.get('side_data_list', []):
    if abs(int(sd.get('rotation', 0))) % 180 == 90: W, H = H, W
k = min(1.0, 640 / max(W, H)); w, h = int(W * k) // 2 * 2, int(H * k) // 2 * 2
raw = subprocess.run(['ffmpeg','-v','error','-i',f,'-vf',f'fps={1/step},scale={w}:{h}','-f','rawvideo','-pix_fmt','gray','-'],capture_output=True).stdout
n = len(raw) // (w * h)
det = cv2.CascadeClassifier(cv2.data.haarcascades + 'haarcascade_frontalface_default.xml')
out = []
for i in range(n):
    g = np.frombuffer(raw, np.uint8, w * h, i * w * h).reshape(h, w)
    r = det.detectMultiScale(g, 1.1, 5, minSize=(int(w * 0.08), int(w * 0.08)))
    if len(r):
        x, y, bw, bh = max(r, key=lambda b: b[2] * b[3])
        out.append([round(i * step, 2), x / k, y / k, bw / k, bh / k])
print(json.dumps({'file': f, 'W': W, 'H': H, 'faces': out}))
````

<!-- file: asr.cjs -->
````js
#!/usr/bin/env node
/* asr — распознавание речи без сети: whisper-small (npm sts-whisper-small) + transformers.js.
   node asr.cjs <аудио/видео> [--lang ru|auto] [--json out.json] [--srt out.srt] → слова с таймкодами */
'use strict';
const path = require('path'), fs = require('fs'), cp = require('child_process');
const A = process.argv.slice(2), opt = k => { const i = A.indexOf('--' + k); return i >= 0 ? A[i + 1] : null; };
const file = A[0], lang = opt('lang') || 'ru';
if (!file || !fs.existsSync(file)) { console.error('asr <файл> [--lang ru|auto] [--json f] [--srt f]'); process.exit(1); }
(async () => {
  const { pipeline, env } = await import('@huggingface/transformers');
  env.allowRemoteModels = false; env.allowLocalModels = true;
  env.localModelPath = path.join(__dirname, 'node_modules', 'sts-whisper-small', 'models');
  const r = cp.spawnSync('ffmpeg', ['-v', 'error', '-i', file, '-ac', '1', '-ar', '16000', '-f', 'f32le', '-'], { maxBuffer: 1 << 30 });
  if (r.status !== 0 || !r.stdout.length) throw new Error('ffmpeg не достал звук из ' + file);
  const a = new Float32Array(r.stdout.buffer.slice(r.stdout.byteOffset, r.stdout.byteOffset + r.stdout.length));
  const asr = await pipeline('automatic-speech-recognition', 'Xenova/whisper-small', { dtype: 'q8' });
  const o = await asr(a, Object.assign({ task: 'transcribe', return_timestamps: 'word', chunk_length_s: 30, stride_length_s: 5 }, lang === 'auto' ? {} : { language: lang }));
  const words = o.chunks.map(c => ({ w: c.text.trim(), s: c.timestamp[0], e: c.timestamp[1] })).filter(x => x.w);
  words.forEach(x => { const mx = 0.25 + 0.09 * x.w.length; if (x.e - x.s > mx + 0.3) x.s = +(x.e - mx).toFixed(2); });   // модель растягивает слово на паузу перед ним — подрезаем начало
  const res = { text: o.text.trim(), words };
  if (opt('json')) fs.writeFileSync(opt('json'), JSON.stringify(res, null, 1));
  if (opt('srt')) {
    const ts = t => { const m = Math.round(t * 1000), p = (n, l) => String(n).padStart(l, '0'); return `${p(Math.floor(m / 3600000), 2)}:${p(Math.floor(m / 60000) % 60, 2)}:${p(Math.floor(m / 1000) % 60, 2)},${p(m % 1000, 3)}`; };
    const out = []; let cur = [];
    const flush = () => { if (cur.length) out.push(`${out.length + 1}\n${ts(cur[0].s)} --> ${ts(cur[cur.length - 1].e)}\n${cur.map(x => x.w).join(' ')}\n`); cur = []; };
    words.forEach((x, i) => { cur.push(x); const gap = words[i + 1] ? words[i + 1].s - x.e : 9; if (/[.!?]$/.test(x.w) || gap > 0.5 || cur.length >= 7) flush(); });
    flush(); fs.writeFileSync(opt('srt'), out.join('\n'));
  }
  console.log(JSON.stringify(res));
})().catch(e => { console.error('ASR ОШИБКА: ' + (e && e.stack || e)); process.exit(1); });
````

<!-- file: copy.md -->
````md
# Режим «Копия референса»

Пользователь прислал чужой ролик (или ссылку) и хочет так же. Сначала пойми, насколько «так же». Если из сообщения неясно, спроси один раз, с вариантами:

| Уровень | Что переносится | Что своё |
|---|---|---|
| **Один в один** | композиция, цвета, шрифт, ритм, движение, переходы, звук | только содержимое: тексты, логотип, видео и скрины пользователя |
| **Только звук** | звуковой дизайн и музыка референса, привязка звуков к действиям | визуал делается своим путём (выбор из трёх направлений) |
| **Вдохновиться** | приёмы, ритм и настроение | стиль выбирается из трёх направлений, одно из них ближе всего к референсу |

Чужие логотипы, тексты и интерфейсы в ролик не попадают ни на каком уровне. Прямая просьба «один в один» важнее сохранённой привычки пользователя выбирать из трёх стилей: стиль здесь задан референсом.

## 1. Разбор референса

Ссылку, которая не открылась, не пересказывай по памяти: попроси файл mp4.

```
mo analyze ref.mp4 --out ref_an
```
Команда печатает ритм (доля движения, длина типичной вспышки и паузы, первое движение), характер кривых, склейки, палитру в hex, громкость, темп и список звуковых ударов, а в папку кладёт:
- `sheet.jpg` — 24 кадра по всему ролику;
- `moves.jpg` — начало, середина и конец каждого движения;
- `rhythm.png` — кривая движения со склейками и звуковыми ударами;
- `spectrum.png` — спектрограмма звука (видно, где низкие толчки, где щелчки, где музыка);
- `audio.wav` — звуковая дорожка.

Посмотри все картинки. Для каждого важного движения сними покадровую полосу: `mo strip ref.mp4 --from 1.2 --to 1.9 --tw 480`. По ней видно то, чего нет в цифрах: откуда входит объект, есть ли перелёт, чем сменяется сцена. Положения и размеры меряй по кадрам 1:1 с сеткой в 10 % кадра: `mo grab ref.mp4 1.9,4.4,6.9 --grid` (бери моменты, когда сцена стоит). Полупрозрачные «веера» и смазанные края в кадрах движения — это размытие движения, а не элемент дизайна: финальный рендер добавит его сам.

## 2. Паспорт референса (запиши в `ref_an/passport.md`, без него код не пиши)

- Кадр: размер, fps, длина. Сцены: таблица «время · что на экране · чем вошло · чем ушло · сколько стоит».
- Цвета: hex из `mo analyze`, какой из них фон, какой акцент.
- Типографика: что за шрифт (ближайший с кириллицей из доступных), кегль в долях высоты кадра, регистр, трекинг, сколько строк, как выровнен.
- Композиция каждой сцены: где объекты в долях кадра, что с чем выровнено.
- Движение: типичная длительность входа и выхода (из вспышек), кривая (пик вспышки: до 0,34 — резкий старт и торможение; 0,34–0,6 — разгон и торможение), перелёты, каскады, размытие.
- Переходы: какой приём между сценами и сколько длится.
- Звук: музыка (есть ли, темп, характер), таблица «звуковой удар → действие на экране», общая громкость.
- Слоты содержимого: «надпись референса → чем заменяем». Тексты пользователя бери дословно. Нет замены — спроси или оставь слот пустым и скажи об этом. Иконки и картинки — тоже содержимое: подбери под новый смысл, но в том же стиле, размере и месте, и назови замену пользователю.

## 3. Сборка

**Один в один.** Стиль не выбирается: позиции, размеры, цвета, длительности и порядок берутся из паспорта. Длительности, каскады и скорость печати задавай явно (`dur`, `stagger`, `cps`; умолчания приёмов перечислены в справочнике), кривую подбирай по характеру вспышки. Моменты действий ставь по кадрам (`mo strip`), а не по списку звуковых ударов: в нём музыка и эффекты вперемешку. Текст длиннее референсного: кегль не уменьшай молча — скажи и предложи перенос или сокращение. Другая длина ролика: убирай или повторяй целые сцены, а не растягивай тайминги.

Звук — из самого референса: `audio.wav` ставится дорожкой `<audio>`, на `#stage` — `data-loudness="off"`, чтобы дорожка осталась как есть; при другой длине ролика режь по тактам (кратно 4 долям из `mo beats`). Свои звуки поверх не добавляй (`sfx: false` у приёмов или `data-sfx-gain="-60"`), если дорожка референса уже содержит эффекты. Предупреди пользователя: дорожка может быть под чужой лицензией, для публикации нужны права или замена.

**Только звук.** Визуал идёт обычным путём. По `spectrum.png` и списку ударов запиши, из чего собран звук референса: где большие удары и нарастания, чем озвучены переходы, есть ли очереди щелчков, звон, печать. Свои звуки ставь тех же ролей на те же действия (имена — в разделе «Звук» справочника). `mo compare` покажет строку «баланс низ/тело/сер/верх» и число звуковых ударов у референса и у тебя: доводи, пока строка не перестанет расходиться. Музыку референса бери, только если у пользователя есть на неё права.

**Вдохновиться.** Из паспорта берутся ритм (доля движения, длина вспышек и пауз), система переходов и два-три фирменных приёма. Палитра, шрифты и композиция свои.

## 4. Сверка (обязательна для «один в один»)

```
mo render <dir> --draft
mo compare ref.mp4 <dir>/out/draft.mp4 --out cmp.jpg
```
Команда кладёт кадры референса и результата рядом на одних и тех же относительных моментах, рисует обе кривые движения и печатает таблицу с пометкой «← расходится». Посмотри `cmp.jpg` и `cmp_rhythm.png` сам. Расхождения вне замены содержимого исправь и сверь заново. Ориентир: сходство ритма ≥ 0,6, расхождение палитры < 30, строк «расходится» нет или каждая объяснима другим содержимым.

Цифры не видят шрифт, композицию и форму объектов: это сверяется глазами по `cmp.jpg` и полосам `mo strip` обоих роликов.

## 5. Сдача

Файл, `cmp.jpg` (референс и результат рядом), таблица сверки и честный список того, что не совпало и почему. «Похоже» без цифр не пиши.
````

<!-- file: review.md -->
````md
# Рецензия на моушн-ролик

Ты смотришь чужой ролик свежим взглядом. Автор кода не видит собственных слабых мест, поэтому рецензент — отдельный человек. Не хвали и не переписывай код: найди, что не так.

Тебе дали: задание пользователя, лист кадров (`sheet.jpg`), лист движений (`moves.jpg`: начало, середина и конец каждого движения), кривую движения (`rhythm.png`), вывод `mo analyze` и `mo check`, строку «Звук:» из рендера. Посмотри каждую картинку.

Ответь по пунктам, у каждого — оценка 1–10 и конкретика с таймкодом:

1. **Хук.** Что происходит в первую секунду? Есть ли движение в первые 0,3 с и понятна ли мысль к 1,5 с? Остановит ли это палец в ленте?
2. **Читаемость на телефоне.** Кегль, контраст, не лезет ли текст друг на друга, на лицо, за край. Хватает ли времени прочесть.
3. **Плотность и ритм.** По кривой: есть ли провалы длиннее 2 с, не проседает ли последняя треть, не одинаковы ли все вспышки.
4. **Графика или подписи.** Есть ли визуализация смысла (иконки, счётчики, схемы, рисованные акценты), или это текст на плашках. Сколько сцен из общего числа — просто текст.
5. **Разнообразие.** Повторяется ли одна композиция и один приём входа из сцены в сцену.
6. **Стиль.** Держится ли одно направление: палитра, шрифты, форма объектов. Нет ли шаблонного вида (градиенты, стекло, свечение).
7. **Окно с видео** (если есть): меняет ли форму, уходит ли из кадра, не срезана ли голова.
8. **Звук** (по строке «Звук:» из рендера и выводу `mo analyze`, на слух ты не судишь): есть ли большой удар на хуке и в финале, не стоит ли он на каждой сцене; баланс полос — не ушёл ли звук в середину и верх; нет ли тишины дольше секунды в середине ролика.
9. **Соответствие заданию.** Всё ли, что просил пользователь, есть в ролике; нет ли выдуманных фактов.

В конце — **пять главных правок по убыванию важности**, каждая с таймкодом и одним предложением «что сделать». Ритм на слух и ощущение скорости по картинкам не оцениваются: если не можешь судить, так и напиши.
````
