---
layout: "post"
title: "How to use Monaco editor with Phoenix LiveView and esbuild"
date: "2023-05-15 12:00:00"
categories: Elixir
excerpt: "Monaco is state-of-the-art browser-based code editor that powers VS Code. Let's see how to integrate it with Phoenix LiveView through esbuild."
---

Monaco Editor is state-of-the-art code editor, packed with features like syntax coloring, IntelliSense[^1], validation and much more. It powers VS Code and since it is browser-based, it can be integrated into web-apps too. If you ever used Elixir's Livebook, you've seen it action.

This article documents step by step how to integrate Monaco editor with LiveView and Phoenix's default bundler for the web, esbuild. The source code shown here can be found in GiHub repository: [szajbus/phoenix_monaco_example](https://github.com/szajbus/phoenix_monaco_example). Let's start.

## Add dependency

First, we need to add the npm dependency. Phoenix keeps frontend assets in `assets` folder by default, so the path needs to be specified.

```bash
$ npm install monaco-editor --prefix assets
```

## Create editor component

To properly initialize the editor and render it on the page, we need to execute some client-side JavaScript once its container is added to DOM and is mounted by LiveView. We can use `phx-hook` to achieve it.

Let's start with the template. The editor's container will be rendered at 100% width and height relative to its parent element.

Through `phx-hook` we connect the container to `CodeEditor` JavaScript object which will manage its lifecycle, we'll define it in next step. We also want the Monaco to control the contents, so we specify `phx-update="ignore"` to prevent the LiveView from re-rendering the container.

Lastly we pass `@language` and `@code` from socket's `assigns` via data attributes.

```html
<div
  id="code-editor"
  class="flex w-full h-full"
  phx-hook="CodeEditor"
  phx-update="ignore"
  data-language={@language}
  data-code={@code}
>
  <div class="w-full h-full" data-el-code-editor />
</div>
```

Next, let's create the hook object in `assets/js/hooks/code_editor.js`.

It defines `mounted` callback to create an instance of Monaco editor and render it when the component is mounted and `destroyed` callback to clean up when it's not needed anymore. We access the actual element the hook is added to via `this.el` and the dynamic variables via its dataset.

We limit ourselves to some most basic configuration options here, the [full list of available customization options](https://microsoft.github.io/monaco-editor/docs.html#interfaces/editor.IStandaloneEditorConstructionOptions.html) is available in official docs.

```js
import * as monaco from "monaco-editor";

const CodeEditor = {
  mounted() {
    const container = this.el.querySelector("[data-el-code-editor]");
    const { language, code } = this.el.dataset;

    this.editor = monaco.editor.create(container, {
      theme: "vs-dark",
      language: language,
      value: code,
      minimap: {
        enabled: false
      }
      // ... other options
    });
  },

  destroyed() {
    if (this.editor) this.editor.dispose();
  },
};

export default CodeEditor;
```

We also need to tell the LiveSocket to use our hook by adding the following to `assets/app.js`.

```javascript
import CodeEditor from "./hooks/code_editor";

let liveSocket = new LiveSocket("/live", Socket, {
  hooks: {
    CodeEditor,
    // ... possibly other hooks
  },
  // ... rest of options
});
```

## Bundle with esbuild

With many other libraries that should be enough, but with Monaco we need some adjustments to the bundling process.

First of all, when we import `monaco-editor` JavaScript code, we are also pulling in the accompanying CSS and other assets bundled with it, like fonts.

Esbuild is not configured to bundle fonts by default, so we need to change it. Let's update the `:default` profile in `config/config.exs` and add `file` loader for `ttf` files. It is enough that the source file is simply copied to output directory without any processing. The loader will automatically embed the file name in the bundle as a string.

```elixir
config :esbuild,
  version: "0.14.41",
  default: [
    args: ~w(
        js/app.js
        --bundle
        --target=es2017
        --outdir=../priv/static/assets
        --external:/fonts/*
        --external:/images/*
        --loader:.ttf=file
      ),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ]
```

With the above configuration the editor should render correctly, but it is quite possible that the rest of application's CSS is now broken. That's because esbuild also bundles CSS and outputs the resulting file as `app.css` which conflicts with the `app.css` bundled by `tailwind` library. The CSS imported with `monaco-editor` and bundled by esbuild simply overwrite the CSS produced by tailwind.

To resolve this conflict, let's rename our `assets/css/app.css` to `assets/css/style.css` and reconfigure tailwind in `config/confix.exs`.

```elixir
config :tailwind,
  version: "3.2.4",
  default: [
    args: ~w(
      --config=tailwind.config.js
      --input=css/style.css
      --output=../priv/static/assets/style.css
    ),
    cd: Path.expand("../assets", __DIR__)
  ]
```

## Utilize web workers

Monaco provides syntax coloring for myriad of programming languages (it's even possible to add one for your custom language if needed), so let's support some popular ones here. For performance reasons we want to offload their work to web workers in order not to block the main thread of execution.

Web workers, naturally, are not imported by default, so we need to again adjust our esbuild configuration to include them. This time however, we'll define a separate esbuild profile for two reasons:

* they will be loaded on demand, so we don't want to bundle them with rest of the code
* they are external files, so we don't need to watch them for changes and rebuild in dev

Here's the new esbuild profile.

```elixir
config :esbuild,
  version: "0.14.41",
  default: [
    # default profile defined above
  ],
  monaco_editor: [
    args: ~w(
        node_modules/monaco-editor/esm/vs/editor/editor.worker.js
        node_modules/monaco-editor/esm/vs/language/css/css.worker.js
        node_modules/monaco-editor/esm/vs/language/html/html.worker.js
        node_modules/monaco-editor/esm/vs/language/json/json.worker.js
        node_modules/monaco-editor/esm/vs/language/typescript/ts.worker.js
        --bundle
        --target=es2017
        --outdir=../priv/static/assets/monaco-editor
      ),
    cd: Path.expand("../assets", __DIR__)
  ]
```

We also need to include it in assets build pipelines which are defined in `mix.exs`.

```elixir
  defp aliases do
    [
      "assets.build": [
        "tailwind default",
        "esbuild default",
        "esbuild monaco_editor"
      ],
      "assets.deploy": [
        "tailwind default --minify",
        "esbuild default --minify",
        "esbuild monaco_editor --minify",
        "phx.digest"
      ],
      # ... other aliases
    ]
  end
```

At this point we should rebuild the assets locally because Phoenix will not do it for us (we don't define watchers for these files in `config/dev.exs`).

```bash
$ mix assets.build
```

Finally, we must tell Monaco where to get these workers from and how to apply them. Let's modify our component to define `MonacoEnvironment`.

```javascript
const CodeEditor = {
  mounted() {
    self.MonacoEnvironment = {
      globalAPI: true,
      getWorkerUrl(_workerId, label) {
        switch (label) {
          case "css":
          case "less":
          case "scss":
            return "/assets/monaco-editor/language/css/css.worker.js";
          case "html":
          case "handlebars":
          case "razor":
            return "/assets/monaco-editor/language/html/html.worker.js";
          case "json":
            return "/assets/monaco-editor/language/json/json.worker.js";
          case "javascript":
          case "typescript":
            return "/assets/monaco-editor/language/typescript/ts.worker.js";
          default:
            return "/assets/monaco-editor/editor/editor.worker.js";
        }
      },
    };

    // the rest of the code may remain unchanged
  }
}
```

That's it. We have a basic code editor that supports syntax coloring for several languages.

The source code presented in this article can be found in GiHub repository: [szajbus/phoenix_monaco_example](https://github.com/szajbus/phoenix_monaco_example).

[^1]: IntelliSense is a general term for various code editing features, such as: code completion, parameter info, quick info, and member lists.
