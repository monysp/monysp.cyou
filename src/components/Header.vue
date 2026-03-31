<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue';
import axios from 'axios';

const discordStatusColor = ref('text-catppuccin-gray');
const spotify = ref(null);
const discordStatus = ref('offline');
const vscodeActivity = ref(null);
const ws = ref(null);
const languageColors = ref({});
const age = ref(0);
const ageDecimal = ref(0);

const fetchLanguageColors = async () => {
  try {
    const response = await axios.get('https://raw.githubusercontent.com/ozh/github-colors/master/colors.json');
    languageColors.value = response.data;
  } catch (error) {
    console.error('Error fetching language colors:', error);
  }
};

const getLanguageFromFile = (filename) => {
  if (!filename) return 'default';
  const ext = filename.split('.').pop().toLowerCase();
  const extensionMap = {
    // Programming Languages
    'js': 'JavaScript',
    'ts': 'TypeScript',
    'py': 'Python',
    'java': 'Java',
    'html': 'HTML',
    'css': 'CSS',
    'php': 'PHP',
    'c': 'C',
    'cpp': 'C++',
    'cs': 'C#',
    'go': 'Go',
    'rb': 'Ruby',
    'rs': 'Rust',
    'swift': 'Swift',
    'kt': 'Kotlin',
    'scala': 'Scala',
    'groovy': 'Groovy',
    'pl': 'Perl',
    'lua': 'Lua',
    'r': 'R',
    'dart': 'Dart',
    'f': 'Fortran',
    'f90': 'Fortran',
    'hs': 'Haskell',
    'jl': 'Julia',
    'lisp': 'Lisp',
    'ml': 'OCaml',
    'pas': 'Pascal',
    'sql': 'SQL',
    'sh': 'Shell',
    'vb': 'Visual Basic',
    'asm': 'Assembly',
    'clj': 'Clojure',
    'erl': 'Erlang',
    'ex': 'Elixir',
    'fs': 'F#',
    'm': 'Objective-C',
    'php': 'PHP',
    'rkt': 'Racket',
    'scm': 'Scheme',
    'tex': 'TeX',
    'vim': 'Vim script',
    'zig': 'Zig',

    // Web Technologies and Frameworks
    'jsx': 'React',
    'tsx': 'React',
    'vue': 'Vue',
    'svelte': 'Svelte',
    'angular': 'Angular',
    'ember': 'Ember',
    'backbone': 'Backbone.js',
    'preact': 'Preact',
    'mjs': 'Node.js',
    'graphql': 'GraphQL',
    'prisma': 'Prisma',
    'astro': 'Astro',

    // Markup and Style
    'md': 'Markdown',
    'scss': 'SCSS',
    'sass': 'Sass',
    'less': 'Less',
    'styl': 'Stylus',
    'pug': 'Pug',
    'haml': 'Haml',
    'slim': 'Slim',
    'xml': 'XML',
    'yaml': 'YAML',
    'toml': 'TOML',

    // Data Formats
    'json': 'JSON',
    'csv': 'CSV',

    // Configuration
    'dockerfile': 'Dockerfile',
    'ini': 'INI',
    'editorconfig': 'EditorConfig',
    'gitignore': 'Git',
    'env': 'DotENV',

    // Build Tools
    'gradle': 'Gradle',
    'maven': 'Maven',
    'ant': 'Ant',

    // Others
    'ipynb': 'Jupyter Notebook',
    'proto': 'Protocol Buffers',
    'sol': 'Solidity',
  };
  return extensionMap[ext] || 'default';
};

const currentLanguageColor = computed(() => {
  if (vscodeActivity.value) {
    const language = getLanguageFromFile(vscodeActivity.value.details.split(' ').pop());
    return `color: ${languageColors.value[language]?.color || '#4F5D95'}`;
  }
  return '';
});

const vscodeStatus = computed(() => {
  if (!vscodeActivity.value) return null;
  
  if (vscodeActivity.value.details.toLowerCase().includes('idling')) {
    return 'idling';
  } else {
    return {
      details: vscodeActivity.value.details,
      state: vscodeActivity.value.state
    };
  }
});

const calculateAge = () => {
  const birthDate = new Date('2008-12-22');
  const today = new Date();
  
  // Calculate exact age in years with decimal places
  const diffTime = today - birthDate;
  const msPerYear = 365.25 * 24 * 60 * 60 * 1000;
  const exactAge = diffTime / msPerYear;
  
  age.value = Math.floor(exactAge);
  ageDecimal.value = (exactAge - age.value).toFixed(10).substring(2); // เอาเฉพาะตัวเลขหลัง 0.
};

// Update age every 10ms to show smooth increment
const updateAgeInterval = () => {
  setInterval(() => {
    calculateAge();
  }, 10);
};

const connectWebSocket = () => {
  ws.value = new WebSocket('wss://api.lanyard.rest/socket');

  ws.value.onopen = () => {
    ws.value.send(JSON.stringify({
      op: 2,
      d: { subscribe_to_id: '1035157898638139435' }
    }));
  };

  ws.value.onmessage = (event) => {
    const message = JSON.parse(event.data);
    if (message.t === "INIT_STATE" || message.t === "PRESENCE_UPDATE") {
      const data = message.d;
      
      spotify.value = data.spotify;
      vscodeActivity.value = data.activities.find(activity =>
        ["Visual Studio Code", "Code", "VSCodium"].includes(activity.name)
      ) || null;

      switch (data.discord_status) {
        case 'online':
          discordStatusColor.value = 'text-catppuccin-green';
          discordStatus.value = 'online';
          break;
        case 'idle':
          discordStatusColor.value = 'text-catppuccin-yellow';
          discordStatus.value = 'idle';
          break;
        case 'dnd':
          discordStatusColor.value = 'text-catppuccin-red';
          discordStatus.value = 'do not disturb';
          break;
        default:
          discordStatusColor.value = 'text-catppuccin-gray';
          discordStatus.value = 'offline';
          break;
      }
    }
  };

  ws.value.onerror = (error) => {
    console.error('WebSocket error', error);
  };

  ws.value.onclose = () => {
    console.log('WebSocket connection closed');
  };
};

onMounted(() => {
  calculateAge();
  updateAgeInterval();
  connectWebSocket();
  fetchLanguageColors();
});

onUnmounted(() => {
  if (ws.value) {
    ws.value.close();
  }
});
</script>

<template>
  <div
    class="font-sans font-black text-5xl rainbow-text"
  >
    monysp.cyou
  </div>
  <div class="text-catppuccin-gray text-lg font-mono tracking-wide mb-2">
    <span class="text-catppuccin-yellow">{{ age }}.{{ ageDecimal }}</span> y/o dumbo. building stuff and learning along the way.
  </div>
  <div class="text-sm mt-2">
    <div><span class="text-catppuccin-blue">discord</span> : <span class="text-catppuccin-gray">tls6</span> <span class="text-catppuccin-gray">[</span><span :class="{ 'text-catppuccin-green': discordStatus === 'online', 'text-catppuccin-yellow': discordStatus === 'idle', 'text-catppuccin-red': discordStatus === 'do not disturb', 'text-catppuccin-gray': discordStatus === 'offline' }">{{ discordStatus }}</span><span class="text-catppuccin-gray">]</span></div>
    <div><span class="text-catppuccin-green">spotify</span> : <span class="text-catppuccin-gray" v-if="spotify">{{ spotify.song }} - {{ spotify.artist }}</span><span class="text-catppuccin-gray" v-else>not playing</span></div>
  </div>
  <div v-if="vscodeActivity" class="text-sm mt-2">
    <span class="text-catppuccin-cyan">code</span>
    <span class="text-catppuccin-gray">:</span>
    <span v-if="typeof vscodeStatus === 'string'" class="text-catppuccin-gray">{{ vscodeStatus }}</span>
    <span v-else-if="vscodeStatus" class="text-catppuccin-gray">
      <span class="text-catppuccin-yellow">{{ vscodeStatus.details }}</span>
      <span class="text-catppuccin-gray">in</span>
      <span class="text-catppuccin-blue">Workspace:</span>
      <span class="text-catppuccin-green">{{ vscodeStatus.state }}</span>
    </span>
  </div>

  <div class="flex gap-6 mt-5 text-xl">
    <a
      href="https://github.com/monruchy/"
      target="_blank"
      class="flex items-center justify-center w-12 h-12 rounded-xl border border-catppuccin-gray bg-opacity-10 transition-colors duration-200 group"
      style="background-color: rgba(255,255,255,0.01);"
    >
      <font-awesome-icon
        :icon="['fab', 'github']" 
        class="w-6 h-6 text-catppuccin-gray transition-colors duration-200 group-hover:text-[#fff]"
      />
    </a>
    <a
      href="https://www.instagram.com/monruchy"
      target="_blank"
      class="flex items-center justify-center w-12 h-12 rounded-xl border border-catppuccin-gray bg-opacity-10 transition-colors duration-200 group"
      style="background-color: rgba(255,255,255,0.01);"
    >
      <font-awesome-icon
        :icon="['fab', 'instagram']"
        class="w-6 h-6 text-catppuccin-gray transition-colors duration-200 group-hover:text-[#E4405F]"
      />
    </a>
    <a
      href="https://discord.com/user/225933101958692864"
      target="_blank"
      class="flex items-center justify-center w-12 h-12 rounded-xl border border-catppuccin-gray bg-opacity-10 transition-colors duration-200 group"
      style="background-color: rgba(255,255,255,0.01);"
    >
      <font-awesome-icon
        :icon="['fab', 'discord']"
        class="w-6 h-6 text-catppuccin-gray transition-colors duration-200 group-hover:text-[#5865F2]"
      />
    </a>
    <a
      href="https://open.spotify.com/user/313rakwl6izaerayob7trim7cuma"
      target="_blank"
      class="flex items-center justify-center w-12 h-12 rounded-xl border border-catppuccin-gray bg-opacity-10 transition-colors duration-200 group"
      style="background-color: rgba(255,255,255,0.01);"
    >
      <font-awesome-icon
        :icon="['fab', 'spotify']"
        class="w-6 h-6 text-catppuccin-gray transition-colors duration-200 group-hover:text-[#1DB954]"
      />
    </a>
  </div>
</template>

<style scoped>
a.group:hover {
  border-color: #fff;
  background-color: rgba(255,255,255,0.04);
}

.rainbow-text {
  background: linear-gradient(
    270deg,
    #ff0080,
    #ff8c00,
    #ffe600,
    #40e0d0,
    #00ffea,
    #8a2be2,
    #ff0080
  );
  background-size: 200% 200%;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  animation: rainbow-move 6s ease-in-out infinite alternate;
  background-clip: text;
  text-fill-color: transparent;
}

@keyframes rainbow-move {
  0% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}
</style>
