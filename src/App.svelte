<script lang="ts">
  import { Notebook } from '@jupyter-kit/svelte';
  import type { Ipynb } from '@jupyter-kit/core';
  import { python as pythonHighlight } from '@jupyter-kit/core/langs/python';
  import { createEditorPlugin } from '@jupyter-kit/editor-codemirror';
  import { createKatexCdnPlugin } from '@jupyter-kit/katex-cdn';
  import { createWidgetsPlugin } from '@jupyter-kit/widgets';
  import {
    createPyodideExecutor,
    type PyodideStatus,
  } from '@jupyter-kit/executor-pyodide';
  import { python as pythonEditor } from '@codemirror/lang-python';

  import showcase from './showcase.json';

  // --- Themes ---------------------------------------------------------------
  //
  // Stylesheets are imported as strings (Vite `?inline`) directly from the
  // published `@jupyter-kit/theme-all` package. The static imports let the
  // bundler tree-shake per-chunk: each select triggers a chunk download, the
  // unused 20 stylesheets never load in practice.

  import chesterishCss from '@jupyter-kit/theme-all/chrome/chesterish.css?inline';
  import darkCss from '@jupyter-kit/theme-all/chrome/dark.css?inline';
  import darkbroncoCss from '@jupyter-kit/theme-all/chrome/darkbronco.css?inline';
  import defaultCss from '@jupyter-kit/theme-all/chrome/default.css?inline';
  import dorkulaCss from '@jupyter-kit/theme-all/chrome/dorkula.css?inline';
  import grade3Css from '@jupyter-kit/theme-all/chrome/grade3.css?inline';
  import gruvboxdCss from '@jupyter-kit/theme-all/chrome/gruvboxd.css?inline';
  import gruvboxlCss from '@jupyter-kit/theme-all/chrome/gruvboxl.css?inline';
  import chromeMonokaiCss from '@jupyter-kit/theme-all/chrome/monokai.css?inline';
  import oceans16Css from '@jupyter-kit/theme-all/chrome/oceans16.css?inline';
  import onedorkCss from '@jupyter-kit/theme-all/chrome/onedork.css?inline';
  import solarizeddCss from '@jupyter-kit/theme-all/chrome/solarizedd.css?inline';
  import solarizedlCss from '@jupyter-kit/theme-all/chrome/solarizedl.css?inline';

  import draculaCss from '@jupyter-kit/theme-all/syntax/dracula.css?inline';
  import githubLightCss from '@jupyter-kit/theme-all/syntax/github-light.css?inline';
  import syntaxMonokaiCss from '@jupyter-kit/theme-all/syntax/monokai.css?inline';
  import oneDarkCss from '@jupyter-kit/theme-all/syntax/one-dark.css?inline';
  import oneLightCss from '@jupyter-kit/theme-all/syntax/one-light.css?inline';
  import solarizedDarkCss from '@jupyter-kit/theme-all/syntax/solarized-dark.css?inline';
  import solarizedLightCss from '@jupyter-kit/theme-all/syntax/solarized-light.css?inline';
  import vscDarkPlusCss from '@jupyter-kit/theme-all/syntax/vsc-dark-plus.css?inline';

  const CHROMES: Record<string, string> = {
    default: defaultCss,
    dark: darkCss,
    monokai: chromeMonokaiCss,
    onedork: onedorkCss,
    solarizedd: solarizeddCss,
    solarizedl: solarizedlCss,
    chesterish: chesterishCss,
    grade3: grade3Css,
    gruvboxd: gruvboxdCss,
    gruvboxl: gruvboxlCss,
    oceans16: oceans16Css,
    dorkula: dorkulaCss,
    darkbronco: darkbroncoCss,
  };

  const SYNTAXES: Record<string, string> = {
    'one-dark': oneDarkCss,
    'one-light': oneLightCss,
    monokai: syntaxMonokaiCss,
    dracula: draculaCss,
    'github-light': githubLightCss,
    'solarized-dark': solarizedDarkCss,
    'solarized-light': solarizedLightCss,
    'vsc-dark-plus': vscDarkPlusCss,
  };

  const CHROME_NAMES = Object.keys(CHROMES);
  const SYNTAX_NAMES = Object.keys(SYNTAXES);

  function injectTheme(
    map: Record<string, string>,
    name: string,
    styleId: string,
  ): void {
    const css = map[name];
    if (!css) return;
    let node = document.getElementById(styleId) as HTMLStyleElement | null;
    if (!node) {
      node = document.createElement('style');
      node.id = styleId;
      document.head.append(node);
    }
    node.textContent = css;
  }

  // --- State --------------------------------------------------------------

  let ipynb = $state<Ipynb>(showcase as Ipynb);
  let filename = $state<string>('showcase.ipynb');
  let chrome = $state<string>('default');
  let syntax = $state<string>('one-dark');
  let status = $state<PyodideStatus>('idle');

  $effect(() => injectTheme(CHROMES, chrome, 'jk-chrome-theme'));
  $effect(() => injectTheme(SYNTAXES, syntax, 'jk-syntax-theme'));

  // One executor per page — re-used across notebook swaps so Pyodide state
  // (imported modules, defined variables) persists.
  const executor = createPyodideExecutor({
    autoloadImports: true,
    onStatus: (s) => {
      status = s;
    },
  });

  const plugins = [
    createKatexCdnPlugin(),
    createWidgetsPlugin(),
    createEditorPlugin({ languages: { python: pythonEditor() } }),
  ];

  const languages = [pythonHighlight];

  const onUpload = (e: Event) => {
    const file = (e.target as HTMLInputElement).files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        ipynb = JSON.parse(reader.result as string) as Ipynb;
        filename = file.name;
      } catch (err) {
        alert(`Invalid ipynb JSON: ${String(err)}`);
      }
    };
    reader.readAsText(file);
  };
</script>

<div>
  <div class="toolbar">
    <label class="field">
      <span class="label">chrome</span>
      <select bind:value={chrome}>
        {#each CHROME_NAMES as t (t)}
          <option value={t}>{t}</option>
        {/each}
      </select>
    </label>
    <label class="field">
      <span class="label">syntax</span>
      <select bind:value={syntax}>
        {#each SYNTAX_NAMES as t (t)}
          <option value={t}>{t}</option>
        {/each}
      </select>
    </label>
    <label class="field">
      <span class="label">open .ipynb</span>
      <input type="file" accept=".ipynb,application/json" onchange={onUpload} />
    </label>
    <span class="status">pyodide: {status}</span>
  </div>
  <Notebook
    {ipynb}
    language="python"
    {languages}
    {plugins}
    {executor}
    {filename}
  />
</div>

<style>
  .toolbar {
    display: flex;
    gap: 12px;
    align-items: center;
    flex-wrap: wrap;
    padding: 8px 12px;
    background: #f6f6f6;
    border-bottom: 1px solid #ddd;
    font-family: system-ui, sans-serif;
    font-size: 13px;
  }
  .field {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }
  .label {
    font-size: 10px;
    text-transform: uppercase;
    opacity: 0.6;
    letter-spacing: 0.5px;
  }
  .status {
    margin-left: auto;
    font-family: monospace;
    font-size: 12px;
    padding: 4px 8px;
    background: #fff;
    border: 1px solid #ddd;
    border-radius: 4px;
  }
</style>
