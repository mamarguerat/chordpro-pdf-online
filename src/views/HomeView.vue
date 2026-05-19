<template>
  <main>
    <form
      class="no-print"
      @submit.prevent
    >
      <label for="chordpro-input">Chord chart in ChordPro format:</label>
      <textarea
        ref="textareaRef"
        v-model="input"
        id="chordpro-input"
        rows="10"
      />

      <fieldset>
        <legend>Import</legend>

        <div>
          <label for="jem-song-number">JEM song number:</label>
          <div class="jem-import-row">
            <input
              id="jem-song-number"
              type="text"
              v-model="jemSongNumber"
              placeholder=""
            >
            <button
              @click="onImportJemSong()"
              type="button"
              class="jem-import-button"
            >
              Import JEM
            </button>
          </div>
        </div>
      </fieldset>

      <fieldset class="headings-fieldset">
        <legend>Headings</legend>

        <div
          v-for="field in metadataFields"
          :key="field"
          :class="[
            'heading-field',
            { 'heading-field--compact': field === 'tempo' || field === 'key' },
          ]"
        >
          <label :for="`heading-${field}`">{{ metadataLabels[field] }}:</label>
          <template v-if="field === 'time'">
            <select
              :id="`heading-${field}`"
              v-model="selectedTimeSignature"
            >
              <option value="">Select time signature</option>
              <option
                v-for="signature in commonTimeSignatures"
                :key="signature"
                :value="signature"
              >
                {{ signature }}
              </option>
            </select>
          </template>
          <input
            v-else
            :id="`heading-${field}`"
            type="text"
            v-model="headingValues[field]"
          >
        </div>
      </fieldset>

      <fieldset>
        <legend>Settings</legend>

        <div>
          <label for="chordpro-transpose">Transpose: </label>
          <input
            type="number"
            id="chordpro-transpose"
            v-model="transposition"
            max="12"
            min="-12"
            required
          >
        </div>

        <div>
          <label for="fontSizeInput">Font size:</label>
          <input
            id="fontSizeInput"
            min="1"
            type="number"
            v-model="fontSize"
          />
        </div>

        <div>
          <label for="columnSelect">Number of columns:</label>
          <select
            id="columnSelect"
            name="Columns"
            v-model="columns"
          >
            <option
              :value="1"
              default
            >
              One
            </option>
            <option
              :value="2"
            >
              Two
            </option>
          </select>
        </div>
        <button
          @click="onPrint()"
          type="button"
          class="print-button"
        >
          Print
        </button>
        <button
          @click="onSave()"
          type="button"
          class="save-button"
        >
          Save
        </button>		
        <button
          @click="onMidi()"
          type="button"
          class="midi-button"
        >
          Midi
        </button>	
        <button
          @click="onPlay()"
          type="button"
          class="play-button"
        >
          Play
        </button>			
      </fieldset>
    </form>
    <p
      v-if="loadingError"
      class="import-error no-print"
    >{{ loadingError }}</p>
    <LoaderBars
      style="margin: 0 auto; "
      v-if="isLoading"
    />
    <div
      class="print-view"
      :style="{
        '--font-size': `${validatedFontSize}px`
      }"
    >
      <ChordChart
        :columns="columns"
        :transposition="validatedTransposition"
        @click="onChordChartClick"
      />
    </div>
  </main>
</template>

<script setup>
const DEFAULT_FONT_SIZE = 12;
import { computed, inject, defineAsyncComponent, onBeforeUnmount, provide, reactive, ref, watch } from 'vue';
import { Key } from 'chordsheetjs';
const ChordChart = defineAsyncComponent(() => import('../components/ChordChart.vue'));
const LoaderBars = defineAsyncComponent(() => import('../components/LoaderBars.vue'));
const input = inject('chordProInput');
const isLoading = inject('isLoading');
const loadingError = inject('loadingError');
const importJemSongByNumber = inject('importJemSongByNumber');

const metadataFields = ['title', 'artist', 'time', 'key', 'subtitle', 'copyright', 'footer', 'tempo'];
const metadataLabels = {
  title: 'Title',
  artist: 'Artist',
  subtitle: 'Subtitle',
  key: 'Key',
  time: 'Time',
  tempo: 'Tempo',
  copyright: 'Copyright',
  footer: 'Footer',
};
const commonTimeSignatures = ['2/4', '3/4', '4/4', '6/8', '12/8'];

const caretPosition = ref(0);
const columns = ref(2);
const fontSize = ref(DEFAULT_FONT_SIZE);
const headingValues = reactive(createEmptyHeadingValues());
const isHydratingHeadingValues = ref(false);
const jemSongNumber = ref('');
const selectedTimeSignature = ref('');
const textareaRef = ref(null);
const transposition = ref(0);
const defaultDocumentTitle = document.title;

provide('caretPosition', caretPosition);

watch(input, () => {
  caretPosition.value = getCaretPosition();
  hydrateHeadingValuesFromInput();
}, { immediate: true });

watch(() => metadataFields.map((field) => headingValues[field]).join('\u0001'), () => {
  if (isHydratingHeadingValues.value) {
    return;
  }

  const updatedInput = rewriteInputHeadingDirectives(input.value || '');
  if (updatedInput !== input.value) {
    input.value = updatedInput;
  }
});

watch(() => headingValues.time, () => {
  syncTimeSignatureInputs();
}, { immediate: true });

watch(selectedTimeSignature, (newValue) => {
  if (newValue !== headingValues.time) {
    headingValues.time = newValue;
  }
});

watch(() => `${headingValues.title}\u0001${headingValues.key}\u0001${jemSongNumber.value}\u0001${validatedTransposition.value}`, () => {
  const pdfTitle = buildPdfDocumentTitle();
  document.title = pdfTitle || defaultDocumentTitle;
}, { immediate: true });

onBeforeUnmount(() => {
  document.title = defaultDocumentTitle;
});

const validatedFontSize = computed(() => {
  if (fontSize.value > 0) {
    return fontSize.value;
  }
  return DEFAULT_FONT_SIZE;
});

const validatedTransposition = computed(() => {
  const transpositionWithDefault = transposition.value || 0;
  return Math.min(12, Math.max(-12, transpositionWithDefault || 0));
});

function getCaretPosition() {
  return textareaRef.value ? textareaRef.value.selectionStart : 0;
}

function onChordChartClick(event) {
  const closestElementWithIndex = event.target.closest('[data-index]');
  if (closestElementWithIndex) {
    const index = parseInt(closestElementWithIndex.getAttribute('data-index'), 10);
    caretPosition.value = index;
    if (textareaRef.value) {
      textareaRef.value.setSelectionRange(caretPosition.value, caretPosition.value);
    }
  }
  if (textareaRef.value) {
    textareaRef.value.focus();
  }
};

async function onImportJemSong() {
  if (!importJemSongByNumber) {
    return;
  }

  jemSongNumber.value = normalizeJemSongNumber(jemSongNumber.value);

  try {
    await importJemSongByNumber(jemSongNumber.value);
  } catch {
    // Error message is set by App.vue and displayed via loadingError.
  }
}

function normalizeJemSongNumber(songNumber) {
  const trimmedSongNumber = `${songNumber || ''}`.trim();
  if (!trimmedSongNumber) {
    return '';
  }

  if (/^[0-9]+$/.test(trimmedSongNumber)) {
    return trimmedSongNumber.padStart(3, '0');
  }

  return trimmedSongNumber;
}

function getJemSongNumberFromTitle(titleText) {
  const titleMatch = `${titleText || ''}`.match(/^JEM\s+([^\s-]+)\s*-\s+/i);
  if (!titleMatch) {
    return '';
  }

  return normalizeJemSongNumber(titleMatch[1]);
}

function stripJemPrefixFromTitle(titleText) {
  return `${titleText || ''}`
    .replace(/^JEM\s+[^\s-]+\s*-\s+/i, '')
    .trim();
}

function sanitizeFilenamePart(value) {
  const cleanedValue = `${value || ''}`
    .replace(/[<>:"/\\|?*]/g, '-')
    .split('')
    .map((character) => (character.charCodeAt(0) < 32 ? '-' : character))
    .join('');

  return cleanedValue
    .replace(/\s+/g, ' ')
    .trim();
}

function buildPdfDocumentTitle() {
  const normalizedTitle = sanitizeFilenamePart(stripJemPrefixFromTitle(headingValues.title));
  const transposedKey = getTransposedKey(headingValues.key, validatedTransposition.value);
  const normalizedKey = sanitizeFilenamePart(transposedKey);
  const normalizedJemNumber = normalizeJemSongNumber(jemSongNumber.value) || getJemSongNumberFromTitle(headingValues.title);

  const prefixParts = ['JEM', normalizedJemNumber, normalizedKey].filter((part) => part);
  const prefix = prefixParts.join('-');

  if (!prefix) {
    return normalizedTitle;
  }

  if (!normalizedTitle) {
    return prefix;
  }

  return `${prefix} - ${normalizedTitle}`;
}

function getTransposedKey(keyText, delta) {
  const normalizedKey = `${keyText || ''}`.trim();
  if (!normalizedKey) {
    return '';
  }

  const parsedKey = Key.parse(normalizedKey);
  if (!parsedKey) {
    return normalizedKey;
  }

  return parsedKey.transpose(delta).toString();
}

function createEmptyHeadingValues() {
  return {
    title: '',
    artist: '',
    subtitle: '',
    key: '',
    time: '',
    tempo: '',
    copyright: '',
    footer: '',
  };
}

function parseHeadingValuesFromInput(chordPro) {
  const parsedValues = createEmptyHeadingValues();
  const lines = `${chordPro || ''}`.split(/\r?\n/);

  for (const line of lines) {
    const parsedDirective = parseDirectiveLine(line);
    if (!parsedDirective) {
      continue;
    }

    const { name, value } = parsedDirective;
    if (Object.hasOwn(parsedValues, name) && !parsedValues[name]) {
      parsedValues[name] = value;
    }
  }

  return parsedValues;
}

function parseDirectiveLine(line) {
  const match = line.match(/^\s*\{([^}\s:]+)\s*(?::\s*|\s+)?([^}]*)\}\s*$/);
  if (!match) {
    return null;
  }

  const directiveName = `${match[1] || ''}`.toLowerCase();
  const directiveValue = (match[2] || '').trim();

  return {
    name: toCanonicalDirectiveName(directiveName, directiveValue),
    value: directiveValue,
  };
}

function toCanonicalDirectiveName(name, value) {
  if (name === 't') {
    return 'title';
  }

  if (name === 'st') {
    return 'subtitle';
  }

  if (name === 'k') {
    return 'key';
  }

  if (name === 'c' || name === 'comment') {
    if (/^(©|\(c\)|copyright\b)/i.test(value)) {
      return 'copyright';
    }
    return 'comment';
  }

  return name;
}

function isHeadingDirectiveLine(line) {
  const parsedDirective = parseDirectiveLine(line);
  return !!parsedDirective && metadataFields.includes(parsedDirective.name);
}

function rewriteInputHeadingDirectives(chordPro) {
  const lines = `${chordPro || ''}`.split(/\r?\n/);
  const bodyLines = lines.filter((line) => !isHeadingDirectiveLine(line));

  while (bodyLines.length > 0 && bodyLines[0].trim() === '') {
    bodyLines.shift();
  }

  const headingDirectiveLines = metadataFields
    .filter((field) => headingValues[field])
    .map((field) => `{${field}: ${headingValues[field]}}`);

  if (headingDirectiveLines.length === 0) {
    return bodyLines.join('\n');
  }

  if (bodyLines.length === 0) {
    return headingDirectiveLines.join('\n');
  }

  return `${headingDirectiveLines.join('\n')}\n\n${bodyLines.join('\n')}`;
}

function hydrateHeadingValuesFromInput() {
  isHydratingHeadingValues.value = true;
  const parsedValues = parseHeadingValuesFromInput(input.value || '');
  metadataFields.forEach((field) => {
    headingValues[field] = parsedValues[field];
  });
  isHydratingHeadingValues.value = false;
}

function syncTimeSignatureInputs() {
  const timeValue = `${headingValues.time || ''}`.trim();

  if (!timeValue) {
    selectedTimeSignature.value = '';
    return;
  }

  if (commonTimeSignatures.includes(timeValue)) {
    selectedTimeSignature.value = timeValue;
    return;
  }

  selectedTimeSignature.value = '';
}

function onPlay() {
	const song = document.getElementById("chordpro-input").value;
	console.debug("Convert to midi and play", song);
	window.parent.postMessage(song, "*");
}

function getDefaultExportFilename() {
  return buildPdfDocumentTitle() || 'chord-chart';
}

function onSave() {
	console.debug("Save chordpro file");
  const choFilename = prompt("Enter filename", getDefaultExportFilename());
	
	if (choFilename) {
		const blob = new Blob([document.getElementById("chordpro-input").value], { type: 'text/plain' }); 	
		const anchor = document.createElement('a');
		anchor.href = window.URL.createObjectURL(blob);
		anchor.style = "display: none;";
    anchor.download = `${sanitizeFilenamePart(choFilename) || getDefaultExportFilename()}.cho`;
		document.body.appendChild(anchor);
		anchor.click();
		window.URL.revokeObjectURL(anchor.href); 
		document.body.removeChild(anchor);			
	}
  
}
	
async function onMidi() {
	console.debug("Generate Midi file");
  const midiFilename = prompt("Enter filename", getDefaultExportFilename());
	
	if (midiFilename) {
		const url = "/orinayo/cp2midi";
		const response = await fetch(url, {method: "POST", body: document.getElementById("chordpro-input").value});
		const blob = await response.blob();		
		const anchor = document.createElement('a');
		anchor.href = window.URL.createObjectURL(blob);
		anchor.style = "display: none;";
    anchor.download = `${sanitizeFilenamePart(midiFilename) || getDefaultExportFilename()}.mid`;
		document.body.appendChild(anchor);
		anchor.click();
		window.URL.revokeObjectURL(anchor.href);
		document.body.removeChild(anchor);		
	}
  
}

function onPrint() {
  window.print();
}
</script>

<style scoped>
main {
  font-size: 16px;

  @media screen and (max-width: 1280px) {
    font-size: 14px;
  }

  @media screen and (max-width: 1024px) {
    font-size: 13px;
  }

  @media screen and (max-width: 875px) {
    font-size: 12px;
  }

  @media screen and (max-width: 768px) {
    font-size: 11px;
  }

  @media screen and (max-width: 576px) {
    font-size: 10px;
  }

  @media screen and (max-width: 475px) {
    font-size: 9px;
  }
}

label {
  display: block;
  font-size: 12px;
  font-weight: bold;
  margin-bottom: 4px;
}

textarea {
  font-family: monospace;
  font-size: 14px;
  width: 100%;
}

input:invalid {
  border-color: red;
}

fieldset {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;

  > * {
    flex: 1;
  }
}

.headings-fieldset {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr)) minmax(120px, 0.65fr);
  align-items: end;
}

.headings-fieldset legend {
  grid-column: 1 / -1;
}

.heading-field {
  min-width: 0;
}

.heading-field--compact {
  grid-column: 4;
}

.headings-fieldset input,
.headings-fieldset select {
  width: 100%;
}

@media screen and (max-width: 875px) {
  .headings-fieldset {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .heading-field--compact {
    grid-column: auto;
  }
}

@media screen and (max-width: 576px) {
  .headings-fieldset {
    grid-template-columns: minmax(0, 1fr);
  }
}

.jem-import-row {
  display: flex;
  gap: 8px;
}

.import-error {
  color: #b00020;
  font-weight: bold;
  margin: 8px 0;
}

.print-view {
  font-size: var(--font-size, 16pt);

  @media print {
    outline: none;
    padding: 0;
    transform: none;
  }
}
</style>
