<script setup lang="ts">
import * as monaco from "monaco-editor";
import { computed, nextTick, onMounted, ref, watch, watchEffect } from "vue";

import EditorWorker from "monaco-editor/esm/vs/editor/editor.worker?worker";
import { registerPHPSnippetLanguage } from "~/lib/register-php-snippet";
import CssWorker from "monaco-editor/esm/vs/language/css/css.worker?worker";
import JSONWorker from "monaco-editor/esm/vs/language/json/json.worker?worker";
import HtmlWorker from "monaco-editor/esm/vs/language/html/html.worker?worker";
import TsWorker from "monaco-editor/esm/vs/language/typescript/ts.worker?worker";
import { preview, store } from "~/composables/store";
import { DEFAULT_EDITOR_CONFIG } from "~/constants";
import { CompiledTheme } from "~/composables/theme-utils";

const props = defineProps<{
  theme: CompiledTheme;
  width: number;
  blockId: string;
}>();

const blockItem = computed(() => {
  return store.value.blocks.find((block) => block.id === props.blockId)!;
});

(self as any).MonacoEnvironment = {
  getWorker(_: string, label: string) {
    if (["typescript", "javascript"].includes(label)) {
      return new TsWorker();
    }
    if (["css", "scss"].includes(label)) {
      return new CssWorker();
    }
    if (["html", "handlebars", "twig"].includes(label)) {
      return new HtmlWorker();
    }
    if (["json", "yaml"].includes(label)) {
      return new JSONWorker();
    }
    return new EditorWorker();
  },
};

const editorRef = ref<monaco.editor.IStandaloneCodeEditor>();
const container = ref<HTMLDivElement>();
// const diffContainer = ref<HTMLDivElement>();

onMounted(async () => {
  if (!container.value) return;
  // if (!diffContainer.value) return;

  const editor = monaco.editor.create(container.value, {
    ...DEFAULT_EDITOR_CONFIG,
  });
  editorRef.value = editor;
  // const diffEditor = monaco.editor.createDiffEditor(diffContainer.value, DEFAULT_EDITOR_CONFIG);
  const activeContainer = computed(() => container.value);
  // const activeContainer = computed(() => (store.value.diff && !preview.value ? diffContainer.value : container.value));
  const activeEditor = computed(() => editor);
  // const activeEditor = computed(() => (store.value.diff && !preview.value ? diffEditor.getModifiedEditor() : editor));
  const editorModel = monaco.editor.createModel(blockItem.value.content, blockItem.value.language);
  // const diffEditorOriginalModel = monaco.editor.createModel(blockItem.value.content, blockItem.value.language);
  // const diffEditorModifiedModel = monaco.editor.createModel(blockItem.value.content, blockItem.value.language);

  editor.setModel(editorModel);
  // diffEditor.setModel({
  //   original: diffEditorOriginalModel,
  //   modified: diffEditorModifiedModel,
  // });

  editor.updateOptions({ tabSize: 2 });
  // diffEditor.getOriginalEditor().updateOptions({ tabSize: 2 });
  // diffEditor.getModifiedEditor().updateOptions({ tabSize: 2 });

  // monaco.languages
  //   .getLanguages()
  //   .find((l) => l.id === "javascript")
  //   ?.loader()
  //   .then(({ language }) => {
  //     monaco.languages.setMonarchTokensProvider("javascript", {
  //       ...language,
  //       tokenizer: {
  //         ...language.tokenizer,
  //         common: [[/\b(?:\w+(?=\())\b/, "function"], ...language.tokenizer.common],
  //       },
  //     });
  //   });

  registerPHPSnippetLanguage(monaco.languages);

  monaco.languages.typescript.javascriptDefaults.setDiagnosticsOptions({
    noSuggestionDiagnostics: true,
    noSemanticValidation: true,
    noSyntaxValidation: true,
  });

  monaco.languages.typescript.typescriptDefaults.setDiagnosticsOptions({
    noSuggestionDiagnostics: true,
    noSemanticValidation: true,
    noSyntaxValidation: true,
  });

  const getEditorHeight = () => {
    if (!activeContainer.value) return 0;
    return activeEditor.value.getContentHeight() + 12;
  };

  const autoHeight = async () => {
    if (!activeContainer.value) return;
    const contentHeight = getEditorHeight();
    activeContainer.value.style.height = `${contentHeight}px`;
    if (props.width) {
      activeEditor.value.layout({ width: props.width, height: contentHeight });
    }
  };

  editor.onDidContentSizeChange(autoHeight);

  if (!preview.value) {
    editor.onDidChangeModelContent(() => {
      blockItem.value.content = editor.getModel()?.getValue() || "";
      // diffEditorOriginalModel.setValue(blockItem.value.content);
      // diffEditorModifiedModel.setValue(blockItem.value.content);
    });

    activeEditor.value.onDidBlurEditorWidget(() => {
      activeEditor.value.setSelection(new monaco.Selection(0, 0, 0, 0));
    });

    // diffEditor.getModifiedEditor().onDidContentSizeChange(autoHeight);
    autoHeight();

    watch(() => store.value.diff, autoHeight);

    watchEffect(() => {
      monaco.editor.defineTheme(`chalk-${props.theme.key}`, props.theme.monaco);
      monaco.editor.setTheme(`chalk-${props.theme.key}`);
    });

    watch(
      () => store.value.showLineNumbers,
      (show) => {
        editor.updateOptions({
          lineDecorationsWidth: show ? 16 : 0,
          lineNumbersMinChars: 1,
          lineNumbers: show ? "on" : "off",
        });
        // diffEditor.updateOptions({
        //   lineDecorationsWidth: show ? 16 : 0,
        //   lineNumbersMinChars: 1,
        //   lineNumbers: show ? "on" : "off",
        // });
      },
      { immediate: true }
    );

    watchEffect(() => {
      monaco.editor.setModelLanguage(editorModel, blockItem.value?.language);
      // monaco.editor.setModelLanguage(diffEditorOriginalModel, blockItem.value?.language);
      // monaco.editor.setModelLanguage(diffEditorModifiedModel, blockItem.value?.language);
    });
  }

  watch(
    preview,
    async (data) => {
      if (!data) return;
      editor.updateOptions({
        wordWrap: "off",
        lineNumbersMinChars: data.showLineNumbers ? 1 : 0,
        lineDecorationsWidth: data.showLineNumbers ? 16 : 0,
        lineNumbers: data.showLineNumbers ? "on" : "off",
      });
      editorModel.setValue(data.content);
      await nextTick();
      if (!activeContainer.value) return;
      const height = getEditorHeight();
      activeContainer.value.style.height = `${height}px`;
      activeEditor.value.layout();
      monaco.editor.setModelLanguage(editorModel, data.language);
      monaco.editor.defineTheme(`chalk-${props.theme.key}`, props.theme.monaco);
      monaco.editor.setTheme(`chalk-${props.theme.key}`);
    },
    {
      immediate: true,
    }
  );

  const source = container.value.querySelector(".monaco-editor");
  // const diffSource = diffContainer.value.querySelector(".modified-in-monaco-diff-editor");
  const target = document.querySelector<HTMLDivElement>("[data-editor-frame-container]");
  const handleScroll = (event: Event): void => {
    if (!target) return;
    target.scrollTop += (event as WheelEvent).deltaY;
  };
  source?.addEventListener("wheel", handleScroll);
  // diffSource?.addEventListener("wheel", handleScroll);

  watch(() => props.width, autoHeight, {
    flush: "post",
  });
  watch(
    () => store.value.fontLigatures,
    async (fontLigatures) => {
      editor.updateOptions({ fontLigatures });
      monaco.editor.remeasureFonts();
    },
    {
      immediate: true,
    }
  );

  // watch(
  //   () => props.width,
  //   (width) => {
  //     editor.layout({ width, height: 200 });
  //   }
  // );

  watch(
    () => store.value.fontFamily,
    async (fontFamily) => {
      await document.fonts.load(`12px ${fontFamily}`);
      editor.updateOptions({ fontFamily });
      monaco.editor.remeasureFonts();
    },
    {
      immediate: true,
    }
  );
});
</script>

<template>
  <div
    :style="({
      '--lineNumbersColor': theme.mode === 'dark' ? 'rgba(255,255,255,0.25)' : 'rgba(0,0,0,0.25)',
    } as any)"
    class="transition-opacity duration-500 delay-200"
    :class="{
      'opacity-0': !props.width,
    }"
  >
    <!-- <div id="diff-editor" ref="diffContainer" class="-mb-3" /> -->
    <div ref="container" class="-mb-3" />
  </div>
</template>

<style>
.iPadShowKeyboard,
.codicon-light-bulb,
.diffOverview {
  display: none !important;
}
.editor.modified {
  left: 0 !important;
}
.detected-link {
  text-decoration: none !important;
}
.original-in-monaco-diff-editor .overflow-guard {
  opacity: 0;
}
.monaco-diff-editor .codicon-diff-remove,
.monaco-diff-editor .codicon-diff-insert {
  opacity: 0 !important;
}
.monaco-editor .line-numbers,
.monaco-editor .line-numbers.active-line-number {
  color: var(--lineNumbersColor) !important;
}
.monaco-editor .squiggly-error {
  display: none;
}
.monaco-editor .unicode-highlight {
  display: none;
}
</style>
