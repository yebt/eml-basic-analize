<script setup lang="ts">
import { toast } from 'toaster-ts';
import { ref, computed } from 'vue'

// -------------------------------------------------------------------
// TYPES
interface Header {
  name: string,
  value: string
}

interface MailData {
  headers: Header[]
}

interface IpData {
  ip: string,
  country_name: string,
  country_code: string,
  region_name: string,
  city: string,
  lat: string,
  lon: string,
  isp: string,
}

// -------------------------------------------------------------------
const emailData = ref<MailData | null>(null);
const loading = ref(false);
const error = ref('');
const searchTerm = ref('');
const ipGeoData = ref<IpData[]>([]);
const extractedIPs = ref<string[]>([]);
const fileInput = ref<HTMLHtmlElement | null>(null)

const filteredHeaders = computed(() => {
  if (!emailData.value || !searchTerm.value) {
    return emailData.value?.headers || [];
  }
  return emailData.value.headers.filter(header =>
    header.name.toLowerCase().includes(searchTerm.value.toLowerCase()) ||
    header.value.toLowerCase().includes(searchTerm.value.toLowerCase())
  );
});

const uniqueCountries = computed(() => {
  const countries = ipGeoData.value.map(ip => ip.country_name).filter(Boolean);
  return [...new Set(countries)];
});

const handleFileSelect = (event: Event) => {
  const target = event.target as HTMLInputElement
  const files = target.files ?? []
  if (!files) {
    return
  }
  const file = files[0]
  if (file) {
    processEMLFile(file);
  }
};

const handleFileDrop = (event: DragEvent) => {
  const file = event.dataTransfer?.files[0];
  if (file && file.name.endsWith('.eml')) {
    processEMLFile(file);
  } else {
    error.value = 'Por favor selecciona un archivo .eml válido';
  }
};

const processEMLFile = async (file: File) => {
  loading.value = true;
  error.value = '';

  try {
    const content = await file.text();
    const parsed = parseEMLContent(content);
    emailData.value = parsed;

    // Extract IPs and get geolocation
    const ips = extractIPAddresses(parsed.headers);
    extractedIPs.value = ips;

    if (ips.length > 0) {
      await getGeoLocationData(ips);
    }
  } catch (err) {
    error.value = 'Error al procesar el archivo: ' + (err as Error).message;
  } finally {
    loading.value = false;
  }
};

const parseEMLContent = (content: string) => {
  const lines = content.split('\n');
  const headers = [];
  let body = '';
  let headerSection = true;
  let currentHeader = { name: '', value: '' };

  for (let line of lines) {
    if (headerSection) {
      if (line.trim() === '') {
        if (currentHeader.name) {
          headers.push({ ...currentHeader });
        }
        headerSection = false;
        continue;
      }

      if (line.match(/^[\w-]+:/)) {
        if (currentHeader.name) {
          headers.push({ ...currentHeader });
        }
        const [name, ...valueParts] = line.split(':');
        currentHeader = {
          name: name.trim(),
          value: valueParts.join(':').trim()
        };
      } else if (line.startsWith(' ') || line.startsWith('\t')) {
        currentHeader.value += ' ' + line.trim();
      }
    } else {
      body += line + '\n';
    }
  }

  return { headers, body: body.trim() };
};

const extractIPAddresses = (headers: Header[]): string[] => {
  const ipRegex = /\b(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\b/g;
  const ips = new Set<string>();

  headers.forEach(header => {
    const matches = header.value.match(ipRegex);
    if (matches) {
      matches.forEach(ip => {
        // Filter out private IP ranges
        if (!isPrivateIP(ip)) {
          ips.add(ip);
        }
      });
    }
  });

  return Array.from(ips);
};

const isPrivateIP = (ip: string) => {
  const parts = ip.split('.').map(Number);
  return (
    (parts[0] === 10) ||
    (parts[0] === 172 && parts[1] >= 16 && parts[1] <= 31) ||
    (parts[0] === 192 && parts[1] === 168) ||
    (parts[0] === 127)
  );
};

const getGeoLocationData = async (ips: string[]) => {
  const geoPromises = ips.map(async (ip: string): Promise<IpData | null> => {
    try {
      // Using ip-api.com free API (no key required)
      const response = await fetch(`http://ip-api.com/json/${ip}?fields=status,message,country,countryCode,region,regionName,city,lat,lon,isp,query`);
      const data = await response.json();

      if (data.status === 'success') {
        return {
          ip: ip,
          country_name: data.country,
          country_code: data.countryCode,
          region_name: data.regionName,
          city: data.city,
          lat: data.lat,
          lon: data.lon,
          isp: data.isp
        } as IpData;
      }
      return null;
    } catch (err) {
      console.error(`Error getting geo data for ${ip}:`, err);
      return null;
    }
  });

  const results = await Promise.all(geoPromises);
  const filteredResults = results.filter((ipdataEl: IpData | null) => !!ipdataEl);
  ipGeoData.value = filteredResults
};

const getCountryFlag = (countryCode: string | null) => {
  if (!countryCode) return '🏳️';
  const flagOffset = 0x1F1E6;
  const asciiOffset = 0x41;
  const firstChar = countryCode.charCodeAt(0) - asciiOffset + flagOffset;
  const secondChar = countryCode.charCodeAt(1) - asciiOffset + flagOffset;
  return String.fromCodePoint(firstChar, secondChar);
};

const copyToClipboard = async (text: string) => {
  try {
    await navigator.clipboard.writeText(text);
    toast.success('Header copied')
  } catch (err) {
    console.error('Error al copiar:', err);
  }
};

const clearSearch = () => {
  searchTerm.value = '';
};

</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <!-- Header -->
    <div class="text-center mb-8">
      <h1 class="text-4xl font-bold text-slate-800 mb-2">📧 EML Analyzer</h1>
      <p class="text-slate-600">Analiza headers de correo y geolocaliza IPs de origen</p>
    </div>

    <!-- File Upload Section -->
    <div class="bg-white rounded-xl shadow-lg p-6 mb-8">
      <div
        class="border-2 border-dashed border-slate-300 rounded-lg p-8 text-center hover:border-blue-400 transition-colors"
        @dragover.prevent @drop.prevent="handleFileDrop" @click="fileInput?.click()">
        <input ref="fileInput" type="file" accept=".eml" @change="handleFileSelect" class="hidden">
        <div class="text-6xl mb-4">📁</div>
        <h3 class="text-xl font-semibold text-slate-700 mb-2">Selecciona un archivo .eml</h3>
        <p class="text-slate-500">Arrastra y suelta o haz clic para seleccionar</p>
      </div>
    </div>

    <!-- Loading State -->
    <transition name="fade">
      <div v-if="loading" class="bg-white rounded-xl shadow-lg p-8 text-center mb-8">
        <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-500 mx-auto mb-4"></div>
        <p class="text-slate-600">Analizando archivo y obteniendo geolocalización...</p>
      </div>
    </transition>

    <!-- Results Section -->
    <transition name="slide">
      <div v-if="emailData && !loading" class="space-y-6">
        <!-- Search Bar -->
        <div class="bg-white rounded-xl shadow-lg p-6">
          <div class="flex gap-4">
            <input v-model="searchTerm" placeholder="Buscar en headers..."
              class="flex-1 px-4 py-2 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
            <button @click="clearSearch"
              class="px-4 py-2 bg-slate-200 text-slate-700 rounded-lg hover:bg-slate-300 transition-colors">
              Limpiar
            </button>
          </div>
        </div>

        <!-- IP Geolocation Section -->
        <div class="bg-white rounded-xl shadow-lg p-6" v-if="ipGeoData.length > 0">
          <h2 class="text-2xl font-bold text-slate-800 mb-4 flex items-center">
            🌍 Geolocalización de IPs
          </h2>
          <div class="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
            <div v-for="ipData in ipGeoData" :key="ipData.ip"
              class="bg-gradient-to-r from-blue-50 to-indigo-50 rounded-lg p-4 border border-blue-200">
              <div class="flex items-center justify-between mb-2">
                <span class="font-mono text-sm bg-blue-100 px-2 py-1 rounded">{{ ipData.ip }}</span>
                <span class="text-2xl">{{ getCountryFlag(ipData.country_code) }}</span>
              </div>
              <div class="space-y-1 text-sm">
                <div><strong>País:</strong> {{ ipData.country_name || 'N/A' }}</div>
                <div><strong>Ciudad:</strong> {{ ipData.city || 'N/A' }}</div>
                <div><strong>Región:</strong> {{ ipData.region_name || 'N/A' }}</div>
                <div><strong>ISP:</strong> {{ ipData.isp || 'N/A' }}</div>
                <div v-if="ipData.lat && ipData.lon" class="text-xs text-slate-500">
                  Coords: {{ ipData.lat }}, {{ ipData.lon }}
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Headers Table -->
        <div class="bg-white rounded-xl shadow-lg p-6">
          <h2 class="text-2xl font-bold text-slate-800 mb-4 flex items-center">
            📋 Headers del Correo
            <span class="ml-2 text-sm font-normal text-slate-500">({{ filteredHeaders.length }} headers)</span>
          </h2>

          <div class="overflow-x-auto">
            <table class="min-w-full">
              <thead>
                <tr class="border-b border-slate-200">
                  <th class="text-left py-3 px-4 font-semibold text-slate-700">Header</th>
                  <th class="text-left py-3 px-4 font-semibold text-slate-700">Valor</th>
                  <th class="text-left py-3 px-4 font-semibold text-slate-700">Acciones</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(header, index) in filteredHeaders" :key="index"
                  class="border-b border-slate-100 hover:bg-slate-50 transition-colors">
                  <td class="py-3 px-4 font-medium text-slate-800">{{ header.name }}</td>
                  <td class="py-3 px-4 text-slate-600 max-w-md">
                    <div class="truncate" :title="header.value">{{ header.value }}</div>
                  </td>
                  <td class="py-3 px-4">
                    <button @click="copyToClipboard(header.value)"
                      class="text-blue-600 hover:text-blue-800 text-sm underline cursor-pointer">
                      Copiar
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- Email Content Preview -->
        <div class="bg-white rounded-xl shadow-lg p-6" v-if="emailData.body">
          <h2 class="text-2xl font-bold text-slate-800 mb-4">📄 Contenido del Correo</h2>
          <div class="bg-slate-50 rounded-lg p-4 max-h-96 overflow-y-auto">
            <pre class="whitespace-pre-wrap text-sm text-slate-700">{{ emailData.body }}</pre>
          </div>
        </div>

        <!-- Summary Statistics -->
        <div class="bg-white rounded-xl shadow-lg p-6">
          <h2 class="text-2xl font-bold text-slate-800 mb-4">📊 Resumen</h2>
          <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
            <div class="text-center p-4 bg-blue-50 rounded-lg">
              <div class="text-2xl font-bold text-blue-600">{{ emailData.headers.length }}</div>
              <div class="text-sm text-slate-600">Headers Totales</div>
            </div>
            <div class="text-center p-4 bg-green-50 rounded-lg">
              <div class="text-2xl font-bold text-green-600">{{ extractedIPs.length }}</div>
              <div class="text-sm text-slate-600">IPs Encontradas</div>
            </div>
            <div class="text-center p-4 bg-purple-50 rounded-lg">
              <div class="text-2xl font-bold text-purple-600">{{ ipGeoData.length }}</div>
              <div class="text-sm text-slate-600">IPs Geolocalizadas</div>
            </div>
            <div class="text-center p-4 bg-orange-50 rounded-lg">
              <div class="text-2xl font-bold text-orange-600">{{ uniqueCountries.length }}</div>
              <div class="text-sm text-slate-600">Países Únicos</div>
            </div>
          </div>
        </div>
      </div>
    </transition>

    <!-- Error Message -->
    <transition name="fade">
      <div v-if="error" class="bg-red-50 border border-red-200 rounded-xl p-6 mb-8">
        <h3 class="text-lg font-semibold text-red-800 mb-2">❌ Error</h3>
        <p class="text-red-700">{{ error }}</p>
      </div>
    </transition>
  </div>
</template>

<style>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.slide-enter-active,
.slide-leave-active {
  transition: all 0.3s ease;
}

.slide-enter-from,
.slide-leave-to {
  transform: translateY(-10px);
  opacity: 0;
}
</style>
