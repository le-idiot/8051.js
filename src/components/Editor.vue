<template>
  <div id="editor">
    <div v-show="isError" class="ui large negative message">
      <div class="header">
        Error at line {{ lineNumber }}
      </div>
      <p>{{ errorMessage }}</p>
    </div>
    <pre id="ace-editor">{{ code }}</pre>
    <div id="buttons" class="ui buttons">
      <button id="run-btn" class="ui green button">Run</button>
      <div class="or"></div>
      <button id="debug-btn" class="ui blue button">Debug</button>
      <div class="or"></div>
      <button id="reset-btn" class="ui grey button">Reset</button>
    </div>
  </div>
</template>

<script>
import { delay, forEach, isEmpty, isUndefined } from 'lodash';
import { mapMutations } from 'vuex';

import ace from 'brace';
import 'brace/theme/tomorrow';
import 'brace/mode/assembly_x86';

const Range = ace.acequire('ace/range').Range;

export default {
  data() {
    return {
      code: '',
      errorMessage: '',
      isError: false,
      lineNumber: 0,
      markers: [],
    };
  },
  mounted() {
    const editor = ace.edit(this.$el.querySelector('#ace-editor'));
    const editorSession = editor.getSession();
    this.e = editorSession;

    editor.setTheme('ace/theme/tomorrow');
    editorSession.setMode('ace/mode/assembly_x86');

    this.$el.querySelector('#run-btn').addEventListener('click', () => this.runEditor());
    this.$el.querySelector('#debug-btn').addEventListener('click', () => this.debugEditor());
    this.$el.querySelector('#reset-btn').addEventListener('click', () => this.resetEditor());
    editorSession.on('change', () => this.resetEditor());
    window.addEventListener('beforeunload', (e) => {
      if (!isEmpty(editor.getValue())) {
        e.preventDefault();
      }
    });

    editor.focus();
  },
  methods: {
    ...mapMutations([
      'run',
      'debug',
      'reset',
    ]),
    runEditor() {
      this.run(this.e.getValue());
      this.highlightLine();
    },
    debugEditor() {
      this.debug(this.e.getValue());
      this.highlightLine();
    },
    resetEditor() {
      this.removeMarkers();
      this.reset(this.e.getValue());
    },
    highlightLine() {
      this.removeMarkers();
      const result = this.$store.state.result;

      if (!isUndefined(result.line)) {
        this.markers.push(this.e.addMarker(
          new Range(result.line, 0, result.line, Infinity),
          `highlight-line ${result.status ? 'current' : 'error'}`,
          'fullLine',
        ));
      }

      if (!isUndefined(result.nextLine)) {
        this.markers.push(this.e.addMarker(
          new Range(result.nextLine, 0, result.nextLine, Infinity),
          'highlight-line next',
          'fullLine',
        ));
      }

      if (!result.status) {
        this.isError = true;
        delay(() => { this.isError = false; }, 2000);
        this.errorMessage = result.msg;
        this.lineNumber = result.line + 1;
      }
    },
    removeMarkers() {
      if (!isUndefined(this.markers)) {
        forEach(this.markers, (marker) => {
          this.e.removeMarker(marker);
        });
        this.markers.length = 0;
      }
    },
  },
};
</script>

<style scoped>
.message {
  z-index: 100;
}

#ace-editor {
  position: absolute;
  top: 0;
  left: 0;
  bottom: 2.5em;
  right: 0;
}

#buttons {
  position: absolute;
  left: 0;
  bottom: 0;
  right: 0;
}
</style>
