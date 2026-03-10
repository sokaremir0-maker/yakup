// ==UserScript==
// @name         Click Helper [bloxd io/bloxd/bloxd.io]
// @namespace    https://bloxd.io
// @version      1.5.1
// @description  Auto Clicker (L/R), Custom CPS, Account Generator, Killaura, BHOP, ViewModel, Player ESP.
// @author       MakeItOrBreakIt
// @license      MIT
// @match        *://*.bloxd.io/*
// @grant        none
// @run-at       document-start
// ==/UserScript==
/*
 * MIT License
 * Copyright (c) 2026 MakeItOrBreakIt
 */
(() => {
  'use strict';
  const attempt = (fn, fb = null) => { try { return fn(); } catch (_) { return fb; } };
  const vals = o => Object.values(o ?? {});
  const ks = o => Object.keys(o ?? {});
 
  const bloxd = {
    noa: null,
    init() {
      const orig = Function.prototype.call;
      let captured = false;
 
      Function.prototype.call = function (thisArg, ...args) {
        if (!captured) {
          const c = args[0];
          if (c && c.entities && c.bloxd) {
            captured = true;
            window.noa = c;
            bloxd.noa = c;
            console.log("%cHooked noaInstance:", "color:#2ECC71;font-weight:bold;", c);
            Function.prototype.call = orig;
          }
        }
        return orig.apply(this, [thisArg, ...args]);
      };
    }
  };
 
  const game = {
    _held: null,
    inGame() { return !!(bloxd.noa?.bloxd?.client?.msgHandler); },
    getPos(id) { return attempt(() => bloxd.noa.entities.getState(id, 'position').position); },
    getIds() { return attempt(() => bloxd.noa?.bloxd?.getPlayerIds?.() ?? {}, {}); },
    getMovement(id = 1) { return attempt(() => bloxd.noa.entities.getState(id, 'movement')); },
    getHeld(id = 1) {
      if (!bloxd.noa) return null;
      if (!this._held) this._held = vals(bloxd.noa.entities).find(fn => {
        if (typeof fn !== 'function' || fn.length !== 1) return false;
        const s = fn.toString();
        return s.length < 80 && s.includes(').') && !s.includes('opWrapper');
      });
      return attempt(() => this._held?.(id));
    },
    doAttack(id) {
      attempt(() => {
        const item = this.getHeld(1);
        const fn = item?.doAttack ?? item?.breakingItem?.doAttack;
        if (!fn) return;
        let offset = [0, 0, 0];
        const jitter = state.killaura.jitter;
        if (jitter > 0) {
          const cam = bloxd.noa?.camera;
          if (cam && cam.heading !== undefined && cam.pitch !== undefined) {
            const yaw = cam.heading + (Math.random() - 0.5) * jitter * 0.35;
            const pitch = cam.pitch + (Math.random() - 0.5) * jitter * 0.25;
            const amt = jitter * 0.12;
            offset = [
              Math.sin(yaw) * Math.cos(pitch) * amt,
              Math.sin(pitch) * amt,
              Math.cos(yaw) * Math.cos(pitch) * amt
            ];
          } else {
            offset = [
              (Math.random() - 0.5) * jitter,
              (Math.random() - 0.5) * jitter * 0.8,
              (Math.random() - 0.5) * jitter * 0.6
            ];
          }
        }
        fn.call(item, offset, id.toString(), 'HeadMesh');
      });
    },
    fireInput(action, down) {
      attempt(() => {
        const inp = bloxd.noa?.inputs;
        if (!inp) return;
        (down ? inp.down : inp.up)._events?.[action]?.(0);
      });
    },
    updateESP(enabled) {
      attempt(() => {
        if (!bloxd.noa) return;
        const r = vals(bloxd.noa)[12];
        if (!r) return;
        const tm = vals(r).find(v => v?.thinMeshes)?.thinMeshes;
        if (Array.isArray(tm)) {
          tm.forEach(i => {
            const m = i?.meshVariations?.__DEFAULT__?.mesh;
            if (m && typeof m.renderingGroupId === "number") {
              m.renderingGroupId = enabled ? 2 : 0;
            }
          });
        }
      });
    }
  };
 
  const state = {
    killaura: { enabled: false, delay: 50, range: 6.7, jitter: 0.35, _last: 0 },
    clicker: { left: { enabled: false, iv: null }, right: { enabled: false, iv: null }, cpsMin: 15, cpsMax: 18 },
    movement: { bhop: false, bhopChance: 100 },
    visuals: { esp: false },
    viewmodel: { enabled: false, pos: { x: 0, y: 0, z: 0 }, rot: { x: 0, y: 0, z: 0 }, spin: false, spinSpeed: { x: 0, y: 2, z: 0 }, spinAngle: { x: 0, y: 0, z: 0 }, _lastTime: 0 },
    binds: { ka: 'KeyK', lc: 'KeyR', rc: 'KeyF', bhop: 'KeyZ', esp: 'KeyV', vm: '', menu: 'ShiftRight' }
  };
 
  let savedBinds = {};
  const bindsStr = localStorage.getItem('clickHelper-keybinds');
  if (bindsStr) { try { savedBinds = JSON.parse(bindsStr); } catch (_) {} }
  Object.assign(state.binds, savedBinds);
 
  let savedSettings = {};
  const settingsStr = localStorage.getItem('clickHelper-settings');
  if (settingsStr) { try { savedSettings = JSON.parse(settingsStr); } catch (_) {} }
  if (savedSettings.killaura) Object.assign(state.killaura, savedSettings.killaura);
  if (savedSettings.clicker) {
    state.clicker.cpsMin = savedSettings.clicker.cpsMin ?? state.clicker.cpsMin;
    state.clicker.cpsMax = savedSettings.clicker.cpsMax ?? state.clicker.cpsMax;
  }
  if (savedSettings.movement) state.movement.bhopChance = savedSettings.movement.bhopChance ?? state.movement.bhopChance;
  if (savedSettings.viewmodel) {
    if (savedSettings.viewmodel.pos) Object.assign(state.viewmodel.pos, savedSettings.viewmodel.pos);
    if (savedSettings.viewmodel.rot) Object.assign(state.viewmodel.rot, savedSettings.viewmodel.rot);
    if (savedSettings.viewmodel.spinSpeed) Object.assign(state.viewmodel.spinSpeed, savedSettings.viewmodel.spinSpeed);
  }
 
  function saveBinds() { localStorage.setItem('clickHelper-keybinds', JSON.stringify(state.binds)); }
  function saveSettings() {
    const settings = {
      killaura: { delay: state.killaura.delay, range: state.killaura.range, jitter: state.killaura.jitter },
      clicker: { cpsMin: state.clicker.cpsMin, cpsMax: state.clicker.cpsMax },
      movement: { bhopChance: state.movement.bhopChance },
      viewmodel: { pos: state.viewmodel.pos, rot: state.viewmodel.rot, spinSpeed: state.viewmodel.spinSpeed }
    };
    localStorage.setItem('clickHelper-settings', JSON.stringify(settings));
  }
  let saveTimeout;
  function debouncedSave() {
    clearTimeout(saveTimeout);
    saveTimeout = setTimeout(() => { saveBinds(); saveSettings(); }, 300);
  }
 
  function toggleClicker(side) {
    const s = state.clicker[side];
    const action = side === 'left' ? 'primary-fire' : 'alt-fire';
    s.enabled = !s.enabled;
    clearTimeout(s.iv);
    if (s.enabled) {
      const tick = () => {
        if (!s.enabled) return;
        game.fireInput(action, true);
        setTimeout(() => game.fireInput(action, false), 20);
        let next = 1000 / (state.clicker.cpsMin + Math.random() * (state.clicker.cpsMax - state.clicker.cpsMin));
        s.iv = setTimeout(tick, Math.max(10, next));
      };
      tick();
    }
    return s.enabled;
  }
 
  function clearAndReload() {
    document.cookie.split(';').forEach(c => {
      const n = c.split('=')[0].trim();
      [`path=/`, `path=/;domain=${location.hostname}`, `path=/;domain=.${location.hostname}`]
        .forEach(p => document.cookie = `${n}=;expires=Thu, 01 Jan 1970 00:00:00 GMT;${p}`);
    });
    localStorage.clear();
    sessionStorage.clear();
    location.reload();
  }
 
  function bindDisplay(code) {
    if (!code) return '—';
    return code.replace('Key', '').replace('Digit', '').replace('ShiftRight', 'RS').replace('ShiftLeft', 'LS')
      .replace('ControlLeft', 'LC').replace('ControlRight', 'RC').replace('AltLeft', 'LA').replace('AltRight', 'RA')
      .replace('Backquote', '`').replace('Backslash', '\\').replace('BracketLeft', '[').replace('BracketRight', ']')
      .replace('Semicolon', ';').replace('Quote', "'").replace('Comma', ',').replace('Period', '.')
      .replace('Slash', '/').replace('Minus', '-').replace('Equal', '=').replace('Space', 'SPC')
      .replace('Tab', 'TAB').replace('CapsLock', 'CAPS').replace('Enter', 'ENT').replace('Escape', 'ESC').replace('Arrow', '');
  }
 
  function safeAppend(el) {
    if (document.body) return document.body.appendChild(el);
    const observer = new MutationObserver(() => {
      if (document.body) { observer.disconnect(); document.body.appendChild(el); }
    });
    observer.observe(document.documentElement, { childList: true });
  }
 
  function buildUI() {
    if (document.getElementById('ch-root')) return;
 
    const css = document.createElement('style');
    css.textContent = `
      @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
      @import url('https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/all.min.css');
      #ch-root, #ch-root * { box-sizing: border-box; }
      :root {
        --ghost-bg: rgba(18, 18, 24, 0.95);
        --ghost-panel: rgba(28, 28, 40, 0.85);
        --ghost-panel-border: rgba(255,255,255,0.06);
        --ghost-accent-1: #7c5cfc;
        --ghost-accent-2: #c84cff;
        --ghost-text: #e8e4f0;
        --ghost-text-dim: #8a8499;
        --ghost-on: #7c5cfc;
        --ghost-off: rgba(60,55,80,0.6);
        --ghost-danger: #ff4466;
        --ghost-success: #44ffaa;
      }
      #ch-root { position: fixed; top: 20px; left: 20px; width: 300px; background: var(--ghost-bg); border: 1px solid var(--ghost-panel-border); border-radius: 10px; font-family: 'Inter', sans-serif; font-size: 11px; color: var(--ghost-text); z-index: 99999; box-shadow: 0 16px 40px rgba(0,0,0,0.6), 0 0 1px rgba(124,92,252,0.4); user-select: none; transition: opacity 0.3s ease, transform 0.3s ease; backdrop-filter: blur(16px) saturate(1.2); transform-origin: center center; }
      #ch-root.hidden { opacity: 0; pointer-events: none; transform: scale(0.92); }
      #ch-root.mini #ch-body, #ch-root.mini #ch-tabs { display: none; }
      #ch-hdr { display: flex; align-items: center; justify-content: space-between; padding: 12px 16px; background: linear-gradient(-45deg, #7c5cfc, #c84cff, #5c9cff, #7c5cfc); background-size: 300% 300%; animation: ghostGradient 4s ease infinite; cursor: grab; border-radius: 9px 9px 0 0; }
      @keyframes ghostGradient { 0%{background-position:0% 50%} 50%{background-position:100% 50%} 100%{background-position:0% 50%} }
      #ch-title { font-weight: 700; font-size: 14px; color: #fff; text-shadow: 0 1px 4px rgba(0,0,0,0.3); letter-spacing: 0.5px; }
      #ch-title span { opacity: 0.7; font-weight: 500; font-size: 10px; margin-left: 6px; }
      .ch-hbtn { background: rgba(255,255,255,0.2); border: none; color: #fff; font-size: 12px; cursor: pointer; padding: 2px 8px; border-radius: 4px; transition: 0.2s; display: flex; align-items: center; justify-content: center; width: 28px; height: 24px; }
      .ch-hbtn:hover { background: rgba(255,255,255,0.4); }
      .ch-hbtn i { font-size: 13px; }
 
      /* === TABS === */
      #ch-tabs { display: flex; background: rgba(0,0,0,0.25); border-bottom: 1px solid var(--ghost-panel-border); padding: 0; margin: 0; gap: 0; }
      .ch-tab { flex: 1; padding: 10px 0 9px 0; text-align: center; font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: 0.8px; color: var(--ghost-text-dim); cursor: pointer; transition: color 0.2s, background 0.2s, box-shadow 0.2s; position: relative; border: none; background: transparent; outline: none; font-family: 'Inter', sans-serif; }
      .ch-tab:hover { color: var(--ghost-text); background: rgba(124,92,252,0.06); }
      .ch-tab.active { color: #fff; }
      .ch-tab.active::after { content: ''; position: absolute; bottom: 0; left: 15%; right: 15%; height: 2.5px; background: linear-gradient(90deg, var(--ghost-accent-1), var(--ghost-accent-2)); border-radius: 2px 2px 0 0; box-shadow: 0 0 8px rgba(124,92,252,0.5); }
      .ch-tab i { margin-right: 5px; font-size: 12px; }
 
      /* === TAB PANELS === */
      .ch-tab-panel { display: none; flex-direction: column; gap: 8px; }
      .ch-tab-panel.active { display: flex; }
 
      #ch-body { padding: 10px; display: flex; flex-direction: column; gap: 8px; max-height: calc(100vh - 120px); overflow-y: auto; }
      #ch-body::-webkit-scrollbar { width: 4px; }
      #ch-body::-webkit-scrollbar-thumb { background: var(--ghost-accent-1); border-radius: 4px; }
      .ch-sec { display: flex; flex-direction: column; gap: 6px; background: var(--ghost-panel); padding: 10px 12px; border-radius: 6px; border: 1px solid var(--ghost-panel-border); }
      .ch-lbl { font-size: 10px; font-weight: 700; color: var(--ghost-text-dim); text-transform: uppercase; letter-spacing: 1px; padding-bottom: 4px; border-bottom: 1px solid rgba(255,255,255,0.04); margin-bottom: 2px; }
      .mod-row { display: flex; align-items: center; justify-content: space-between; padding: 2px 0; }
      .mod-left { display: flex; align-items: center; gap: 8px; flex: 1; }
      .mod-name { font-size: 12px; font-weight: 600; color: var(--ghost-text); }
      .tg { width: 30px; height: 16px; background: var(--ghost-off); border-radius: 8px; position: relative; cursor: pointer; transition: 0.25s; flex-shrink: 0; }
      .tg::after { content: ''; width: 12px; height: 12px; background: #888; border-radius: 50%; position: absolute; top: 2px; left: 2px; transition: 0.25s; }
      .tg.on { background: var(--ghost-on); box-shadow: 0 0 10px rgba(124,92,252,0.4); }
      .tg.on::after { left: 16px; background: #fff; }
      .kb { background: rgba(0,0,0,0.3); border: 1px solid rgba(255,255,255,0.08); color: var(--ghost-text-dim); font-size: 9px; font-weight: 600; padding: 2px 6px; border-radius: 4px; cursor: pointer; transition: 0.2s; display: flex; align-items: center; gap: 4px; font-family: monospace; }
      .kb:hover { background: rgba(124,92,252,0.15); border-color: rgba(124,92,252,0.4); color: var(--ghost-text); }
      .kb.listening { border-color: var(--ghost-accent-2); color: var(--ghost-accent-2); animation: kbpulse 0.8s ease infinite; }
      @keyframes kbpulse { 0%,100%{opacity:1;} 50%{opacity:0.4;} }
      .kb-x { font-size: 8px; color: rgba(255,100,100,0.5); cursor: pointer; }
      .kb-x:hover { color: var(--ghost-danger); }
      .val-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; }
      .val-item { display: flex; flex-direction: column; gap: 4px; }
      .val-label { font-size: 9px; color: var(--ghost-text-dim); font-weight: 600; text-transform: uppercase; }
      .val-inp { background: rgba(0,0,0,0.3); border: 1px solid rgba(255,255,255,0.08); border-radius: 4px; color: var(--ghost-text); font-family: monospace; font-size: 11px; padding: 4px 6px; text-align: center; outline: none; transition: 0.2s; width: 100%; }
      .val-inp:focus { border-color: var(--ghost-accent-1); box-shadow: 0 0 8px rgba(124,92,252,0.2); }
      .sl-row { display: flex; align-items: center; gap: 8px; margin-top: 6px; width: 100%; }
      .sl-label { font-size: 10px; color: var(--ghost-text-dim); font-weight: 600; width: 16px; flex-shrink: 0; }
      input[type=range] { flex: 1; -webkit-appearance: none; background: transparent; cursor: pointer; }
      input[type=range]::-webkit-slider-runnable-track { height: 4px; background: rgba(255,255,255,0.08); border-radius: 2px; }
      input[type=range]::-webkit-slider-thumb { height: 12px; width: 12px; border-radius: 50%; background: #fff; cursor: pointer; -webkit-appearance: none; margin-top: -4px; box-shadow: 0 0 6px rgba(124,92,252,0.6); }
      .sl-val-inp { width: 44px; flex-shrink: 0; background: rgba(0,0,0,0.3); border: 1px solid rgba(255,255,255,0.1); color: var(--ghost-accent-1); font-family: monospace; font-size: 10px; font-weight: 600; text-align: center; border-radius: 4px; outline: none; padding: 3px 0; }
      .sl-val-inp:focus { border-color: var(--ghost-accent-1); }
      .ch-action-btn { width: 100%; padding: 8px; border-radius: 6px; border: 1px solid rgba(255,68,102,0.3); background: rgba(255,68,102,0.1); color: #ff6688; font-weight: 700; font-size: 11px; cursor: pointer; transition: 0.2s; text-transform: uppercase; letter-spacing: 0.5px; }
      .ch-action-btn:hover { background: rgba(255,68,102,0.2); border-color: #ff4466; color: #ff8899; }
      /* Text input fields */
      .ch-text-inp { background: rgba(0,0,0,0.3); border: 1px solid rgba(255,255,255,0.08); border-radius: 4px; color: var(--ghost-text); font-family: 'Inter', sans-serif; font-size: 11px; padding: 6px 8px; outline: none; transition: 0.2s; width: 100%; }
      .ch-text-inp:focus { border-color: var(--ghost-accent-1); box-shadow: 0 0 8px rgba(124,92,252,0.2); }
      .ch-text-inp::placeholder { color: var(--ghost-text-dim); opacity: 0.6; }
      #ch-status { padding: 8px 16px; background: rgba(0,0,0,0.3); border-top: 1px solid var(--ghost-panel-border); font-size: 9px; color: var(--ghost-text-dim); display: flex; justify-content: space-between; align-items: center; border-radius: 0 0 9px 9px; font-weight: 600; letter-spacing: 0.5px; }
      #ch-status .dot { width: 6px; height: 6px; border-radius: 50%; display: inline-block; margin-right: 6px; }
      #ch-status .dot.green { background: var(--ghost-success); box-shadow: 0 0 8px rgba(68,255,170,0.5); }
      #ch-status .dot.red { background: var(--ghost-danger); box-shadow: 0 0 8px rgba(255,68,102,0.5); }
	  #ch-gen { background: rgba(255,68,102,0.3) !important; color: #ff8899 !important; }
	  #ch-gen:hover { background: rgba(255,68,102,0.5) !important; }
	  #ch-root input[type="text"],
	  #ch-root input[type="number"],
	  #ch-root textarea {
		   background: rgba(0,0,0,0.3) !important;
		   border: 1px solid rgba(255,255,255,0.08) !important;
		   border-radius: 4px !important;
		   color: var(--ghost-text) !important;
		   font-family: 'Inter', monospace !important;
		   font-size: 11px !important;
		   padding: 4px 6px !important;
		   outline: none !important;
		   transition: 0.2s !important;
		   box-shadow: none !important;
	  }
 
	  #ch-root input[type="text"]:focus,
	  #ch-root input[type="number"]:focus,
	  #ch-root textarea:focus {
		   border-color: var(--ghost-accent-1) !important;
		   box-shadow: 0 0 8px rgba(124,92,252,0.2) !important;
	  }
 
	  #ch-root input::placeholder,
	  #ch-root textarea::placeholder {
		   color: var(--ghost-text-dim) !important;
		   opacity: 0.6 !important;
	  }
    `;
    document.head.appendChild(css);
 
    const mkToggle = (id, name, bindKey) => {
      const disp = bindDisplay(state.binds[bindKey]);
      const hasKey = !!state.binds[bindKey];
      return `
        <div class="mod-row" id="row-${id}">
          <div class="mod-left">
            <div class="tg" id="tg-${id}"></div>
            <span class="mod-name">${name}</span>
          </div>
          ${bindKey ? `
          <div class="mod-right">
            <div class="kb" id="bind-${bindKey}" data-action="${bindKey}">
              <span class="kb-text">${hasKey ? disp : '—'}</span>
              ${hasKey ? `<span class="kb-x" data-unbind="${bindKey}">✕</span>` : `<span class="kb-x" data-unbind="${bindKey}" style="display:none">✕</span>`}
            </div>
          </div>` : ''}
        </div>`;
    };
 
    const mkSlider = (id, label, min, max, val, step = '1') =>
      `<div class="sl-row">
        <span class="sl-label">${label}</span>
        <input type="range" id="${id}" min="${min}" max="${max}" value="${val}" step="${step}">
        <input type="number" class="sl-val-inp" id="${id}-v" value="${val}" step="${step}" min="${min}" max="${max}">
      </div>`;
 
    const mkTextInput = (id, placeholder, value = '') =>
      `<input type="text" class="ch-text-inp" id="${id}" placeholder="${placeholder}" value="${value}">`;
 
    const root = document.createElement('div');
    root.id = 'ch-root';
    root.innerHTML = `
      <div id="ch-hdr">
        <span id="ch-title">CLICK HELPER <span>v1.5.1</span></span>
        <div style="display:flex;gap:5px;">
          <button class="ch-hbtn" id="ch-gen" title="New Account" style="background: rgba(255,68,102,0.3); color: #ff8899;"><i class="fa-solid fa-user"></i></button>
          <button class="ch-hbtn" id="ch-inject" title="Re-Inject"><i class="fa-solid fa-syringe"></i></button>
          <button class="ch-hbtn" id="ch-min" title="Minimize">—</button>
        </div>
      </div>
      <div id="ch-tabs">
        <button class="ch-tab active" data-tab="combat"><i class="fa-solid fa-crosshairs"></i>Combat</button>
        <button class="ch-tab" data-tab="movement"><i class="fa-solid fa-person-running"></i>Move</button>
        <button class="ch-tab" data-tab="visuals"><i class="fa-solid fa-eye"></i>Visuals</button>
      </div>
      <div id="ch-body">
        <!-- COMBAT TAB -->
        <div class="ch-tab-panel active" data-panel="combat">
          <div class="ch-sec">
            <div class="ch-lbl">Kill Aura</div>
            ${mkToggle('ka', 'Kill Aura', 'ka')}
            <div style="margin-top: 6px;">
              ${mkSlider('ka-delay', 'Delay', 0, 1000, state.killaura.delay, 5)}
              ${mkSlider('ka-range', 'Range', 1, 20, state.killaura.range, 0.1)}
              ${mkSlider('ka-jitter', 'Jitter', 0, 0.6, state.killaura.jitter, 0.01)}
            </div>
          </div>
          <div class="ch-sec">
            <div class="ch-lbl">Auto Clicker</div>
            ${mkToggle('lc', 'Left Click', 'lc')}
            ${mkToggle('rc', 'Right Click', 'rc')}
            <div class="val-grid" style="margin-top: 6px;">
              <div class="val-item"><span class="val-label">Min CPS</span><input class="val-inp" id="inp-cmin" type="number" value="${state.clicker.cpsMin}" step="1" min="1" max="50"></div>
              <div class="val-item"><span class="val-label">Max CPS</span><input class="val-inp" id="inp-cmax" type="number" value="${state.clicker.cpsMax}" step="1" min="1" max="50"></div>
            </div>
          </div>
        </div>
 
        <!-- MOVEMENT TAB -->
        <div class="ch-tab-panel" data-panel="movement">
          <div class="ch-sec">
            <div class="ch-lbl">Movement</div>
            ${mkToggle('bhop', 'BHOP', 'bhop')}
            <div style="margin-top: 4px;">
              ${mkSlider('bhop-chance', '%', 1, 100, state.movement.bhopChance)}
            </div>
          </div>
        </div>
 
        <!-- VISUALS TAB -->
        <div class="ch-tab-panel" data-panel="visuals">
          <div class="ch-sec">
            <div class="ch-lbl">ESP</div>
            ${mkToggle('esp', 'Player ESP', 'esp')}
          </div>
          <div class="ch-sec">
            <div class="ch-lbl">ViewModel</div>
            ${mkToggle('vm', 'ViewModel', 'vm')}
            <div style="font-size: 8px; font-weight: 700; color: var(--ghost-text-dim); margin-top: 8px;">POSITION</div>
            ${mkSlider('vm-px', 'X', -3, 3, state.viewmodel.pos.x, '0.05')}
            ${mkSlider('vm-py', 'Y', -3, 3, state.viewmodel.pos.y, '0.05')}
            ${mkSlider('vm-pz', 'Z', -3, 3, state.viewmodel.pos.z, '0.05')}
            <div style="font-size: 8px; font-weight: 700; color: var(--ghost-text-dim); margin-top: 10px;">ROTATION</div>
            ${mkSlider('vm-rx', 'X', -3.14, 3.14, state.viewmodel.rot.x, '0.01')}
            ${mkSlider('vm-ry', 'Y', -3.14, 3.14, state.viewmodel.rot.y, '0.01')}
            ${mkSlider('vm-rz', 'Z', -3.14, 3.14, state.viewmodel.rot.z, '0.01')}
            <div style="border-top: 1px solid rgba(255,255,255,0.04); margin: 8px 0 4px 0;"></div>
            ${mkToggle('spin', 'Spin Tool', '')}
            <div style="font-size: 8px; font-weight: 700; color: var(--ghost-text-dim); margin-top: 6px;">SPIN SPEED</div>
            ${mkSlider('spin-sx', 'X', -10, 10, state.viewmodel.spinSpeed.x, '0.1')}
            ${mkSlider('spin-sy', 'Y', -10, 10, state.viewmodel.spinSpeed.y, '0.1')}
            ${mkSlider('spin-sz', 'Z', -10, 10, state.viewmodel.spinSpeed.z, '0.1')}
          </div>
        </div>
      </div>
      <div id="ch-status">
        <span><span class="dot red" id="st-dot"></span><span id="st-txt">Waiting for game</span></span>
        <span style="opacity:0.6">RSHIFT = MENU</span>
      </div>
    `;
 
    safeAppend(root);
 
    // === TAB SWITCHING ===
    const tabs = root.querySelectorAll('.ch-tab');
    const panels = root.querySelectorAll('.ch-tab-panel');
    tabs.forEach(tab => {
      tab.addEventListener('click', () => {
        const target = tab.dataset.tab;
        tabs.forEach(t => t.classList.remove('active'));
        panels.forEach(p => p.classList.remove('active'));
        tab.classList.add('active');
        const panel = root.querySelector(`.ch-tab-panel[data-panel="${target}"]`);
        if (panel) panel.classList.add('active');
      });
    });
 
    // === BINDINGS ===
    const bindToggle = (id, onChange) => {
      const row = document.getElementById(`row-${id}`);
      const tg = document.getElementById(`tg-${id}`);
      if (!row || !tg) return;
      row.addEventListener('click', e => {
        if (e.target.closest('.kb') || e.target.classList.contains('kb-x') || e.target.tagName === 'INPUT') return;
        const res = onChange();
        tg.classList.toggle('on', res);
      });
    };
 
    bindToggle('ka', () => state.killaura.enabled = !state.killaura.enabled);
    bindToggle('lc', () => toggleClicker('left'));
    bindToggle('rc', () => toggleClicker('right'));
    bindToggle('bhop', () => state.movement.bhop = !state.movement.bhop);
    bindToggle('esp', () => state.visuals.esp = !state.visuals.esp);
    bindToggle('vm', () => state.viewmodel.enabled = !state.viewmodel.enabled);
    bindToggle('spin', () => state.viewmodel.spin = !state.viewmodel.spin);
 
    let listeningBind = null;
    root.addEventListener('click', e => {
      const unbindEl = e.target.closest('[data-unbind]');
      if (unbindEl) {
        e.stopPropagation();
        const action = unbindEl.dataset.unbind;
        state.binds[action] = '';
        const kbEl = document.getElementById(`bind-${action}`);
        if (kbEl) {
          kbEl.querySelector('.kb-text').innerText = '—';
          const x = kbEl.querySelector('.kb-x'); if (x) x.style.display = 'none';
        }
        if (listeningBind === action) { kbEl?.classList.remove('listening'); listeningBind = null; }
        debouncedSave();
        return;
      }
      const kbEl = e.target.closest('.kb');
      if (!kbEl) return;
      const action = kbEl.dataset.action;
      if (!action) return;
      if (listeningBind) {
        const prev = document.getElementById(`bind-${listeningBind}`);
        if (prev) {
          prev.classList.remove('listening');
          prev.querySelector('.kb-text').innerText = bindDisplay(state.binds[listeningBind]) || '—';
        }
      }
      listeningBind = action;
      kbEl.classList.add('listening');
      kbEl.querySelector('.kb-text').innerText = '…';
    });
 
    const toggleUiById = (id) => {
      const tg = document.getElementById(`tg-${id}`);
      if (!tg) return;
      let res = false;
      switch (id) {
        case 'ka': res = (state.killaura.enabled = !state.killaura.enabled); break;
        case 'lc': res = toggleClicker('left'); break;
        case 'rc': res = toggleClicker('right'); break;
        case 'bhop': res = (state.movement.bhop = !state.movement.bhop); break;
        case 'esp': res = (state.visuals.esp = !state.visuals.esp); break;
        case 'vm': res = (state.viewmodel.enabled = !state.viewmodel.enabled); break;
      }
      tg.classList.toggle('on', res);
    };
 
    const preventInput = () => attempt(() => { if (bloxd.noa?.inputs) bloxd.noa.inputs._paused = true; });
    const restoreInput = () => attempt(() => { if (bloxd.noa?.inputs) bloxd.noa.inputs._paused = false; });
 
    const bindInp = (id, obj, key) => {
      const el = document.getElementById(id);
      if (!el) return;
      el.addEventListener('input', e => { obj[key] = parseFloat(e.target.value) || 0; debouncedSave(); });
      el.addEventListener('focus', preventInput);
      el.addEventListener('blur', restoreInput);
      el.addEventListener('keydown', e => e.stopPropagation(), true);
    };
    bindInp('inp-cmin', state.clicker, 'cpsMin');
    bindInp('inp-cmax', state.clicker, 'cpsMax');
 
    const bindSlider = (id, cb) => {
      const range = document.getElementById(id);
      const num = document.getElementById(id + '-v');
      if (!range || !num) return;
      const update = (valStr) => {
        let v = parseFloat(valStr);
        if (isNaN(v)) return;
        range.value = v; num.value = v; cb(v); debouncedSave();
      };
      range.addEventListener('input', e => update(e.target.value));
      num.addEventListener('input', e => update(e.target.value));
      num.addEventListener('focus', preventInput);
      num.addEventListener('blur', restoreInput);
      num.addEventListener('keydown', e => e.stopPropagation(), true);
    };
 
    bindSlider('ka-delay', v => state.killaura.delay = Math.max(0, Math.floor(v)));
    bindSlider('ka-range', v => state.killaura.range = v);
    bindSlider('ka-jitter', v => state.killaura.jitter = v);
    bindSlider('bhop-chance', v => state.movement.bhopChance = v);
    bindSlider('vm-px', v => state.viewmodel.pos.x = v);
    bindSlider('vm-py', v => state.viewmodel.pos.y = v);
    bindSlider('vm-pz', v => state.viewmodel.pos.z = v);
    bindSlider('vm-rx', v => state.viewmodel.rot.x = v);
    bindSlider('vm-ry', v => state.viewmodel.rot.y = v);
    bindSlider('vm-rz', v => state.viewmodel.rot.z = v);
    bindSlider('spin-sx', v => state.viewmodel.spinSpeed.x = v);
    bindSlider('spin-sy', v => state.viewmodel.spinSpeed.y = v);
    bindSlider('spin-sz', v => state.viewmodel.spinSpeed.z = v);
 
    // Bind all text inputs to pause game input on focus
    root.querySelectorAll('.ch-text-inp').forEach(el => {
      el.addEventListener('focus', preventInput);
      el.addEventListener('blur', restoreInput);
      el.addEventListener('keydown', e => e.stopPropagation(), true);
    });
 
    document.getElementById('ch-gen').addEventListener('click', () => {
      if (confirm("Are you sure you want to clear cookies/local storage and generate a new account?")) clearAndReload();
    });
 
    document.getElementById('ch-inject').addEventListener('click', () => bloxd.init());
 
    let mini = false, visible = true;
    document.getElementById('ch-min').addEventListener('click', () => {
      mini = !mini; root.classList.toggle('mini', mini);
    });
 
    document.addEventListener('keydown', e => {
      if (listeningBind) {
        e.preventDefault(); e.stopPropagation();
        const kbEl = document.getElementById(`bind-${listeningBind}`);
        if (e.code === 'Escape') {
          if (kbEl) { kbEl.querySelector('.kb-text').innerText = bindDisplay(state.binds[listeningBind]) || '—'; kbEl.classList.remove('listening'); }
        } else if (e.code === 'Backspace') {
          state.binds[listeningBind] = '';
          if (kbEl) {
            kbEl.querySelector('.kb-text').innerText = '—';
            kbEl.classList.remove('listening');
            const x = kbEl.querySelector('.kb-x'); if (x) x.style.display = 'none';
          }
        } else {
          state.binds[listeningBind] = e.code;
          if (kbEl) {
            kbEl.querySelector('.kb-text').innerText = bindDisplay(e.code);
            kbEl.classList.remove('listening');
            const x = kbEl.querySelector('.kb-x'); if (x) x.style.display = '';
          }
        }
        listeningBind = null;
        debouncedSave();
        return;
      }
      if (e.code === state.binds.menu) {
        visible = !visible;
        root.classList.toggle('hidden', !visible);
      }
      if (document.activeElement?.tagName === 'INPUT') return;
      for (const [id, key] of Object.entries(state.binds)) {
        if (id !== 'menu' && key && e.code === key) toggleUiById(id);
      }
    }, true);
 
    const hdr = document.getElementById('ch-hdr');
    let ox = 0, oy = 0, mx = 0, my = 0;
    hdr.addEventListener('mousedown', e => {
      if (e.target !== hdr && e.target.id !== 'ch-title' && !e.target.closest('#ch-title')) return;
      e.preventDefault();
      mx = e.clientX; my = e.clientY;
      const move = ev => {
        ox = mx - ev.clientX; oy = my - ev.clientY;
        mx = ev.clientX; my = ev.clientY;
        root.style.top = (root.offsetTop - oy) + 'px';
        root.style.left = (root.offsetLeft - ox) + 'px';
        root.style.right = 'unset';
      };
      const up = () => {
        document.removeEventListener('mousemove', move);
        document.removeEventListener('mouseup', up);
        hdr.style.cursor = 'grab';
      };
      document.addEventListener('mousemove', move);
      document.addEventListener('mouseup', up);
      hdr.style.cursor = 'grabbing';
    });
  }
 
  let inGame = false;
  function rafLoop(ts) {
    requestAnimationFrame(rafLoop);
    const dot = document.getElementById('st-dot');
    const txt = document.getElementById('st-txt');
    const nowInGame = game.inGame();
    if (nowInGame !== inGame) {
      inGame = nowInGame;
      if (dot && txt) {
        if (inGame) { dot.className = 'dot green'; txt.innerText = 'IN GAME'; }
        else { dot.className = 'dot red'; txt.innerText = 'Waiting for game'; }
      }
    }
    if (!inGame) return;
 
    // ESP Update
    game.updateESP(state.visuals.esp);
 
    // Killaura
    if (state.killaura.enabled && Date.now() - state.killaura._last >= state.killaura.delay) {
      const selfPos = game.getPos(1);
      if (selfPos) {
        for (const id of vals(game.getIds())) {
          if (id == 1) continue;
          const p = game.getPos(id);
          if (!p) continue;
          if (Math.hypot(p[0]-selfPos[0], p[1]-selfPos[1], p[2]-selfPos[2]) <= state.killaura.range) {
            state.killaura._last = Date.now();
            game.doAttack(id);
          }
        }
      }
    }
 
    // BHOP
    const inputs = bloxd.noa?.inputs?.state;
    const mov = game.getMovement(1);
    if (state.movement.bhop && inputs && mov) {
      const isMoving = inputs.forward || inputs.backward || inputs.left || inputs.right;
      const onGround = attempt(() => mov.isOnGround());
      if (onGround && isMoving && Math.random() * 100 <= state.movement.bhopChance) {
        inputs.jump = true;
      } else {
        inputs.jump = false;
      }
    }
 
    // ViewModel
    if (state.viewmodel.enabled) {
      const item = game.getHeld(1);
      if (item) {
        const vdt = state.viewmodel._lastTime ? (ts - state.viewmodel._lastTime) / 1000 : 0;
        state.viewmodel._lastTime = ts;
 
        if (state.viewmodel.spin && item?.toolType === "Tool") {
          state.viewmodel.spinAngle.x += state.viewmodel.spinSpeed.x * vdt;
          state.viewmodel.spinAngle.y += state.viewmodel.spinSpeed.y * vdt;
          state.viewmodel.spinAngle.z += state.viewmodel.spinSpeed.z * vdt;
        }
 
        if (item.firstPersonPosOffset) {
          item.firstPersonPosOffset.x = state.viewmodel.pos.x;
          item.firstPersonPosOffset.y = state.viewmodel.pos.y;
          item.firstPersonPosOffset.z = state.viewmodel.pos.z;
        }
 
        if (item.firstPersonRotation && item?.toolType === "Tool") {
          item.firstPersonRotation.x = state.viewmodel.rot.x + (state.viewmodel.spin ? state.viewmodel.spinAngle.x : 0);
          item.firstPersonRotation.y = state.viewmodel.rot.y + (state.viewmodel.spin ? state.viewmodel.spinAngle.y : 0);
          item.firstPersonRotation.z = state.viewmodel.rot.z + (state.viewmodel.spin ? state.viewmodel.spinAngle.z : 0);
        }
      }
    }
  }
 
  let uiBuilt = false;
  function initUI() {
    if (!uiBuilt) { uiBuilt = true; buildUI(); }
  }
 
  if (document.readyState === 'loading') document.addEventListener('DOMContentLoaded', initUI);
  else initUI();
 
  bloxd.init();
 
  let coreStarted = false;
  const poll = setInterval(() => {
    if (!uiBuilt) initUI();
    if (window.noa && !coreStarted) {
      coreStarted = true;
      bloxd.noa = window.noa;
      requestAnimationFrame(rafLoop);
      clearInterval(poll);
    }
  }, 250);
})();
