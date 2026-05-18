<template>
  <RouterView />
</template>

<script setup>
import { provide, ref } from 'vue';
import { RouterView } from 'vue-router'

const searchParams = new URLSearchParams(location.search);
const googleDriveFileId = searchParams.get('googleDriveFileId');
const jemNumber = searchParams.get('jemNumber');
const chordProInput = ref('');
const isLoading = ref(true);
const loadingError = ref('');
provide('chordProInput', chordProInput);
provide('isLoading', isLoading);
provide('loadingError', loadingError);
provide('importJemSongByNumber', importJemSongByNumber);

initializeInput();

async function initializeInput() {
  loadingError.value = '';
  isLoading.value = true;

  if (jemNumber) {
    try {
      await importJemSongByNumber(jemNumber);
    } catch {
      isLoading.value = false;
    }
    return;
  }

  if (googleDriveFileId) {
    try {
      const response = await fetch(`https://www.googleapis.com/drive/v3/files/${googleDriveFileId}?alt=media`, {
        method: 'GET',
        headers: {
          'X-goog-api-key': import.meta.env.VITE_GOOGLE_API_KEY,
        }
      });

      if (!response.ok) {
        throw new Error('Network response was not ok');
      }

      chordProInput.value = normalizeImportedChordPro(await response.text());
      isLoading.value = false;
    } catch {
      setSampleInput();
    }
    return;
  }

  setSampleInput();
}

async function importJemSongByNumber(songNumber) {
  const normalizedSongNumber = normalizeJemSongNumber(songNumber);
  if (!normalizedSongNumber) {
    loadingError.value = 'Enter a JEM song number.';
    isLoading.value = false;
    throw new Error('Missing JEM song number');
  }

  loadingError.value = '';
  isLoading.value = true;

  try {
    const jemUrl = `https://jemaf.fr/ressources/chordPro/jem${encodeURIComponent(normalizedSongNumber)}.chordpro`;
    const responseBody = await fetchJemChordProText(jemUrl);
    if (!responseBody.trim()) {
      throw new Error('JEM import returned an empty response body');
    }

    chordProInput.value = normalizeImportedChordPro(responseBody, {
      jemSongNumber: normalizedSongNumber,
    });
    isLoading.value = false;
    return chordProInput.value;
  } catch {
    loadingError.value = `Unable to import JEM song ${normalizedSongNumber}. The source may be unavailable or blocked by CORS.`;
    chordProInput.value = '';
    isLoading.value = false;
    throw new Error('Unable to import JEM song');
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

async function fetchJemChordProText(jemUrl) {
  const fetchAttempts = [
    // Direct request first for environments where CORS is already allowed.
    () => fetch(jemUrl),
    // Public fallback proxies for static deployments without a backend.
    () => fetch(`https://api.allorigins.win/raw?url=${encodeURIComponent(jemUrl)}`),
    () => fetch(`https://corsproxy.io/?${encodeURIComponent(jemUrl)}`),
  ];

  let lastError = null;

  for (const attempt of fetchAttempts) {
    try {
      const response = await attempt();
      const text = await response.text();
      if (text.trim()) {
        return text;
      }

      if (!response.ok) {
        lastError = new Error(`JEM import failed with status ${response.status}`);
      }
    } catch (error) {
      lastError = error;
    }
  }

  throw lastError || new Error('Unable to load JEM content');
}

function normalizeImportedChordPro(chordPro, options = {}) {
  const parsedLines = `${chordPro || ''}`
    .split(/\r?\n/)
    .map((line) => parseDirectiveLine(line))
    .filter((entry) => entry !== null);

  const firstTitleIndex = parsedLines.findIndex((entry) => entry?.name === 'title');
  const firstArtistIndex = parsedLines.findIndex((entry) => entry?.name === 'artist');
  const firstSubtitleIndex = parsedLines.findIndex((entry) => entry?.name === 'subtitle');

  const jemSongNumber = `${options.jemSongNumber || ''}`.trim();
  if (jemSongNumber && firstTitleIndex >= 0) {
    const currentTitle = parsedLines[firstTitleIndex].value;
    const prefixedTitle = `JEM ${jemSongNumber} - ${currentTitle}`;
    if (!/^JEM\s+\S+\s+-\s+/i.test(currentTitle)) {
      parsedLines[firstTitleIndex].value = prefixedTitle;
    }
  }

  if (firstSubtitleIndex >= 0) {
    const subtitleValue = parsedLines[firstSubtitleIndex].value;
    if (firstArtistIndex >= 0) {
      parsedLines[firstArtistIndex].value = subtitleValue;
      parsedLines.splice(firstSubtitleIndex, 1);
    } else {
      parsedLines[firstSubtitleIndex].name = 'artist';
    }
  }

  return parsedLines
    .map((entry) => {
      if (entry.type === 'text') {
        return entry.value;
      }
      return `{${entry.name}: ${entry.value}}`;
    })
    .join('\n');
}

function parseDirectiveLine(line) {
  const match = line.match(/^\s*\{([^}\s:]+)\s*(?::\s*|\s+)?([^}]*)\}\s*$/);
  if (!match) {
    return {
      type: 'text',
      value: line,
    };
  }

  const directiveName = `${match[1] || ''}`.toLowerCase();
  const directiveValue = `${match[2] || ''}`.trim();
  const canonicalName = toCanonicalDirectiveName(directiveName, directiveValue);

  if (canonicalName === 'comment' && /^jemaf\.fr/i.test(directiveValue)) {
    return null;
  }

  return {
    type: 'directive',
    name: canonicalName,
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

function setSampleInput() {
  import('@/assets/sample-chart.js')
    .then(({ sampleChordProChart }) => {
      loadingError.value = '';
      chordProInput.value = sampleChordProChart;
      isLoading.value = false;
    });
}
</script>

<style scoped>
nav {
  display: flex;
  justify-content: center;
  gap: 1rem;
  padding: 1rem;
  background-color: #f0f0f0;
}
</style>
