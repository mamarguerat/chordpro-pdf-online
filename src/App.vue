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

function buildJemFetchAttempts(jemUrl) {
  const encodedJemUrl = encodeURIComponent(jemUrl);
  const attempts = [
    // Direct request first for environments where CORS is already allowed.
    () => fetch(jemUrl),
    // Public fallback proxies for static deployments without a backend.
    () => fetch(`https://api.allorigins.win/raw?url=${encodedJemUrl}`),
    () => fetch(`https://api.codetabs.com/v1/proxy/?quest=${encodedJemUrl}`),
  ];

  // corsproxy.io dropped the anonymous legacy `?<url>` form. Its keyed API is
  // used first when a key is configured, and the keyless `?url=` form stays as
  // a last resort: it is accepted from github.io origins but rate limited.
  const corsProxyApiKey = `${import.meta.env.VITE_CORSPROXY_API_KEY || ''}`.trim();
  if (corsProxyApiKey) {
    attempts.push(() =>
      fetch(
        `https://corsproxy.io/?key=${encodeURIComponent(corsProxyApiKey)}&url=${encodedJemUrl}`,
      ),
    );
  }
  attempts.push(() => fetch(`https://corsproxy.io/?url=${encodedJemUrl}`));

  return attempts;
}

function isChordProResponseBody(contentType, text) {
  const normalizedContentType = `${contentType || ''}`.toLowerCase();
  if (normalizedContentType.includes('json') || normalizedContentType.includes('html')) {
    return false;
  }

  // Proxy failures answer with a JSON payload or an HTML error page. A ChordPro
  // file may start with a `{title: ...}` directive, so only JSON-looking and
  // markup-looking bodies are rejected here.
  const trimmedText = text.trim();
  return !/^(<|\[|\{\s*")/.test(trimmedText);
}

async function fetchJemChordProText(jemUrl) {
  let lastError = null;

  for (const attempt of buildJemFetchAttempts(jemUrl)) {
    try {
      const response = await attempt();
      if (!response.ok) {
        lastError = new Error(`JEM import failed with status ${response.status}`);
        continue;
      }

      const text = await response.text();
      if (!text.trim()) {
        lastError = new Error('JEM import returned an empty response body');
        continue;
      }

      if (!isChordProResponseBody(response.headers?.get?.('content-type'), text)) {
        lastError = new Error('JEM import returned a non-ChordPro response body');
        continue;
      }

      return text;
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
