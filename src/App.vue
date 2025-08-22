<template>
  <div class="min-h-screen bg-neutral-950 text-neutral-50 p-6 pb-24">
    <div class="w-full flex flex-col gap-6">

      <header v-if="showUI" class="flex flex-col gap-2">
        <h1 class="text-2xl md:text-3xl font-bold">{{ pageTitle || 'Twitch VOD Multi-Stream Sync (Vue)' }}</h1>
      </header>

      <!-- Správa (vlastník) -->
      <section v-if="showUI" class="bg-neutral-900/60 p-4 rounded-2xl flex flex-col gap-3">
        <div class="flex items-center justify-between">
          <h2 class="font-semibold">Správa</h2>
          <div class="text-xs text-neutral-400">
            Formulář pro přidání VOD je viditelný, jen pokud cookie <code>vod_owner</code> odpovídá klíči níže.
          </div>
        </div>

        <!-- Název / title -->
        <div class="flex flex-wrap gap-2 items-center">
          <input
              v-model="pageTitle"
              placeholder="Název zápasu/projektu (uloží se do URL)"
              class="flex-1 min-w-[260px] bg-neutral-800 rounded-xl px-3 py-2 outline-none"
              @keydown.stop
          />
          <span class="text-xs text-neutral-400">Tip: např. „Playoffs – Team A vs Team B (Map 3)“</span>
        </div>

        <div class="flex flex-wrap gap-2 items-center">
          <input v-model="ownerKey" placeholder="Vlastnický klíč (uloží se do URL)" class="min-w-[220px] bg-neutral-800 rounded-xl px-3 py-2 outline-none" @keydown.stop />
          <input v-model="ownerCookieInput" placeholder="Nastavit cookie vod_owner (sem klíč)" class="min-w-[220px] bg-neutral-800 rounded-xl px-3 py-2 outline-none" @keydown.stop />
          <button @click="applyOwnerCookie" class="px-4 py-2 rounded-xl bg-neutral-700 hover:bg-neutral-600">Nastavit cookie</button>
          <span class="text-sm" :class="isOwner ? 'text-emerald-400' : 'text-amber-400'">
            Stav: {{ isOwner ? 'máš oprávnění přidávat VOD' : 'bez oprávnění' }}
          </span>
        </div>
      </section>

      <!-- Globální čas / synchronizace + FILTRY -->
      <section v-if="showUI" class="bg-neutral-900/60 p-4 rounded-2xl flex flex-col gap-3">
        <div class="flex flex-wrap items-center gap-3">
          <div class="text-lg font-semibold">Globální čas T: <span class="tabular-nums">{{ formatHMS(globalElapsed) }}</span></div>
          <button @click="syncToCurrentGlobal" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">Synchronizovat na T</button>

          <div class="w-px h-6 bg-neutral-800 mx-1" />

          <input v-model="globalTargetHms" placeholder="Zadat T (HH:MM:SS)" class="w-44 bg-neutral-800 rounded-xl px-3 py-1.5 outline-none text-sm" @keydown.stop />
          <button @click="seekAllToGlobalTarget" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">Přeskočit</button>

          <div class="w-px h-6 bg-neutral-800 mx-3" />

          <!-- Filtr týmů -->
          <label class="text-sm opacity-80">Zobrazit:</label>
          <select v-model="filterTeam" class="bg-neutral-800 rounded-xl px-3 py-1.5 text-sm outline-none">
            <option value="ALL">Vše</option>
            <option value="A">Jen Tým A</option>
            <option value="B">Jen Tým B</option>
          </select>

          <!-- Filtr hráčů (multi-select) -->
          <label class="text-sm opacity-80">Hráči:</label>
          <select multiple v-model="selectedPlayers" class="min-w-[180px] max-w-[280px] h-[2.6rem] bg-neutral-800 rounded-xl px-2 py-1.5 text-sm outline-none">
            <option v-for="p in uniquePlayers" :key="p" :value="p">{{ p }}</option>
          </select>
          <button @click="selectAllPlayers" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">Vybrat vše</button>
          <button @click="clearPlayerSelection" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">Vymazat výběr</button>
        </div>

        <!-- Volitelně sloupce gridu -->
        <div class="flex flex-wrap items-center gap-2">
          <label class="text-sm opacity-80">Sloupce:</label>
          <select v-model.number="manualCols" class="bg-neutral-800 rounded-xl px-3 py-1.5 text-sm outline-none">
            <option :value="0">Auto (max 5)</option>
            <option :value="1">1</option>
            <option :value="2">2</option>
            <option :value="3">3</option>
            <option :value="4">4</option>
            <option :value="5">5</option>
          </select>
        </div>
      </section>

      <!-- Hlavní layout: vlevo přehrávače, vpravo kapitoly -->
      <div class="grid gap-6" :style="gridColumnsStyle">
        <div class="flex flex-col gap-6">
          <!-- Form (jen pro vlastníka) -->
          <section v-if="isOwner" class="grid gap-3 bg-neutral-900/60 p-4 rounded-2xl">
            <div class="flex gap-2 flex-wrap items-center">
              <input v-model="vodInput" placeholder="VOD (v123456789 nebo URL)" class="flex-1 min-w-[240px] bg-neutral-800 rounded-xl px-3 py-2 outline-none" @keydown.stop />
              <input v-model="offsetInput" placeholder="Začátek ve VODu (HH:MM:SS)" class="w-44 bg-neutral-800 rounded-xl px-3 py-2 outline-none" @keydown.stop />
              <input v-model="playerNameInput" placeholder="Jméno hráče" class="w-44 bg-neutral-800 rounded-xl px-3 py-2 outline-none" @keydown.stop />
              <select v-model="teamInput" class="bg-neutral-800 rounded-xl px-3 py-2 outline-none">
                <option value="A">Tým A</option>
                <option value="B">Tým B</option>
              </select>
              <button @click="addVod" class="px-4 py-2 rounded-xl bg-neutral-700 hover:bg-neutral-600">Přidat VOD</button>
            </div>
            <p v-if="status" class="text-sm text-red-400">{{ status }}</p>
          </section>
          <section v-else class="bg-neutral-900/40 p-3 rounded-xl text-sm text-neutral-400">
            Formulář pro přidávání je skrytý (nemáš oprávnění). Nastav shodný klíč a cookie v sekci „Správa“.
          </section>

          <!-- Pokud jsou viditelné přesně 2 streamy, zobraz je vedle sebe v jedné mřížce bez ohledu na tým -->
          <section v-if="totalVisibleCount === 2" class="grid gap-4" :style="gridStyleFor(2)">
            <Tile
              v-for="item in combinedVisibleEntries"
              :key="item.elementId"
              :item="item"
              :audio-element-id="audioElementId"
              @set-audio="setAudio(item)"
              @toggle-mute="toggleMute(item)"
              @remove="removeByElementId(item.elementId)"
              @update-offset="updateTileOffset(item, $event)"
              @focus="openFocus(item)"
            />
          </section>

          <!-- Jinak standardně rozdělené podle týmů -->
          <template v-else>
            <!-- GRID: Team A -->
            <h3 class="text-lg font-semibold">Tým A</h3>
            <section class="grid gap-4" :style="gridStyleFor(visibleCount(entriesTeamA))">
              <Tile
                v-for="item in entriesTeamA"
                :key="item.elementId"
                :item="item"
                :audio-element-id="audioElementId"
                v-show="isVisible(item, 'A')"
                @set-audio="setAudio(item)"
                @toggle-mute="toggleMute(item)"
                @remove="removeByElementId(item.elementId)"
                @update-offset="updateTileOffset(item, $event)"
                @focus="openFocus(item)"
              />
            </section>

            <!-- GRID: Team B -->
            <h3 class="text-lg font-semibold mt-4">Tým B</h3>
            <section class="grid gap-4" :style="gridStyleFor(visibleCount(entriesTeamB))">
              <Tile
                v-for="item in entriesTeamB"
                :key="item.elementId"
                :item="item"
                :audio-element-id="audioElementId"
                v-show="isVisible(item, 'B')"
                @set-audio="setAudio(item)"
                @toggle-mute="toggleMute(item)"
                @remove="removeByElementId(item.elementId)"
                @update-offset="updateTileOffset(item, $event)"
                @focus="openFocus(item)"
              />
            </section>
          </template>
        </div>

        <!-- Pravý panel: Kapitoly -->
        <aside v-if="showUI" class="chapters-panel bg-neutral-900/60 p-4 rounded-2xl sticky top-4">
          <div class="flex items-center justify-between mb-2">
            <h2 class="font-semibold">Kapitoly</h2>
          </div>

          <div class="flex flex-col gap-2 mb-3">
            <input v-model="chapterNameInput" placeholder="Název kapitoly" class="bg-neutral-800 rounded-xl px-3 py-2 outline-none" @keydown.stop />
            <div class="flex gap-2">
              <input v-model="chapterTimeInput" placeholder="Čas (HH:MM:SS nebo sekundy)" class="flex-1 bg-neutral-800 rounded-xl px-3 py-2 outline-none" @keydown.stop />
              <button @click="addChapterManual" class="px-3 py-2 rounded-xl bg-neutral-700 hover:bg-neutral-600">Přidat</button>
            </div>
            <button @click="addChapterNow" class="px-3 py-2 rounded-xl bg-neutral-800 hover:bg-neutral-700">+ Kapitolu (teď)</button>
            <div class="text-xs text-neutral-400">Kapitoly i „vybraná kapitola“ se ukládají do URL (param <code>state</code>).</div>
          </div>

          <div v-if="sortedChapters.length === 0" class="text-neutral-400 text-sm">Zatím žádné kapitoly.</div>

          <ul v-else class="space-y-2">
            <li
                v-for="(ch, idx) in sortedChapters"
                :key="idx"
                class="rounded-xl px-3 py-2"
                :class="selectedChapterSeconds === ch.tSeconds ? 'bg-violet-700/30 border border-violet-600' : 'bg-neutral-800'"
            >
              <!-- 1) horní řádek: název + čas -->
              <div class="flex items-center justify-between gap-2">
                <button class="text-left underline hover:no-underline" @click="goToChapter(ch)">
                  {{ ch.name || 'Kapitola' }} — {{ formatHMS(ch.tSeconds) }}
                </button>
              </div>

              <!-- 2) spodní řádek: akční tlačítka -->
              <div class="mt-2 flex flex-wrap items-center gap-2">
                <button
                    class="px-2 py-1 text-xs rounded-lg"
                    :class="selectedChapterSeconds === ch.tSeconds ? 'bg-violet-600' : 'bg-neutral-700 hover:bg-neutral-600'"
                    @click="setSelectedChapter(ch)"
                    title="Označit kapitolu (uloží do URL)"
                >
                  Označit
                </button>

                <button
                    class="px-2 py-1 text-xs rounded-lg bg-neutral-700 hover:bg-neutral-600"
                    @click="editChapterTime(idx)"
                >
                  Upravit čas
                </button>

                <button
                    class="px-2 py-1 text-xs rounded-lg bg-neutral-700 hover:bg-neutral-600"
                    @click="renameChapter(idx)"
                >
                  Přejmenovat
                </button>

                <button
                    class="opacity-70 hover:opacity-100"
                    @click="removeChapter(idx)"
                    title="Smazat"
                >
                  ×
                </button>
              </div>
            </li>
          </ul>
        </aside>
      </div>
    </div>

    <!-- Spodní ovládací panel -->
    <div class="fixed bottom-0 left-0 right-0 z-50">
      <div class="backdrop-blur bg-neutral-900/80 border-t border-neutral-800 px-4 py-2 flex flex-wrap items-center gap-2 w-full">
        <!-- čas T + skok -->
        <div class="flex items-center gap-2">
          <span class="text-sm opacity-90">T:</span>
          <span class="text-sm font-mono tabular-nums">{{ formatHMS(globalElapsed) }}</span>
          <input v-model="globalTargetHms" placeholder="HH:MM:SS" class="w-28 bg-neutral-800 rounded-lg px-2 py-1 outline-none text-sm" @keydown.stop />
          <button @click="seekAllToGlobalTarget" class="px-2.5 py-1.5 rounded-lg bg-neutral-800 hover:bg-neutral-700 text-sm">Přeskočit</button>
        </div>

        <div class="w-px h-6 bg-neutral-800 mx-1" />

        <!-- minutové posuny -->
        <button @click="seekAll(-60)" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">⏪ −60s</button>
        <button @click="seekAll(-5)" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">⏪ −5s</button>

        <!-- play/pause -->
        <button v-if="isGloballyPaused" @click="resumeAll" class="px-4 py-1.5 rounded-xl bg-violet-600 hover:bg-violet-500 text-sm font-medium">Play (␣)</button>
        <button v-else @click="pauseAll" class="px-4 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm font-medium">Pause (␣)</button>

        <!-- vpřed -->
        <button @click="seekAll(5)" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">⏩ +5s</button>
        <button @click="seekAll(60)" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">⏩ +60s</button>

        <div class="w-px h-6 bg-neutral-800 mx-1" />

        <button @click="restartAll" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">Spustit odznova</button>
        <button @click="syncToCurrentGlobal" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">Synchronizovat (S)</button>

        <div class="w-px h-6 bg-neutral-800 mx-1" />

        <label class="text-sm opacity-80">Audio tým:</label>
        <select v-model="activeAudioTeam" @change="applyTeamAudio()" class="bg-neutral-800 rounded-xl px-3 py-1.5 text-sm outline-none">
          <option value="A">Tým A</option>
          <option value="B">Tým B</option>
        </select>

        <div class="w-px h-6 bg-neutral-800 mx-1" />

        <label class="text-sm opacity-80">Sloupce:</label>
        <select v-model.number="manualCols" class="bg-neutral-800 rounded-xl px-3 py-1.5 text-sm outline-none">
          <option :value="0">Auto (max 5)</option>
          <option :value="1">1</option>
          <option :value="2">2</option>
          <option :value="3">3</option>
          <option :value="4">4</option>
          <option :value="5">5</option>
        </select>

        <div class="w-px h-6 bg-neutral-800 mx-1" />
        <button @click="addChapterNow" class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm">+ Kapitolu (teď)</button>
        <div class="ml-auto" />
        <button
          @click="showUI = !showUI"
          class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm"
          :title="showUI ? 'Skrýt panely (kapitoly, filtry, správa)' : 'Zobrazit panely'"
        >
          {{ showUI ? '🗖' : '🗗' }}
        </button>
      </div>
    </div>

    <!-- Modal: Focus player -->
    <div v-if="isFocusOpen" class="fixed inset-0 z-[60] flex items-center justify-center">
      <div class="absolute inset-0 bg-black/70" @click="closeFocus"></div>
      <div class="relative w-[92vw] max-w-6xl bg-neutral-900 rounded-2xl border border-neutral-800 shadow-2xl overflow-hidden">
        <div class="flex items-center justify-between px-4 py-3 border-b border-neutral-800">
          <div class="font-semibold">
            {{ focusItem?.playerName || focusItem?.label || 'Stream' }}
            <span v-if="focusItem">— Tým {{ focusItem.team }}</span>
          </div>
          <div class="flex items-center gap-2">
            <button class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm" @click="syncFocusToGlobal">Sync na T</button>
            <button class="px-3 py-1.5 rounded-xl bg-neutral-800 hover:bg-neutral-700 text-sm" @click="closeFocus">Zavřít (Esc)</button>
          </div>
        </div>
        <div class="aspect-video relative bg-black">
          <div id="focus-player" class="absolute inset-0"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, reactive, ref, watch, computed, nextTick, defineComponent, h } from 'vue'

/** ---------- Stav ---------- */
const pageTitle = ref('Twitch VOD Multi-Stream Sync (Vue)') // uloží se do URL

const allEntries = reactive([]) // { id, label, playerName, elementId, muted, offsetSeconds, offsetHms, team, overlayText, overlayVisible, _overlayTimer }
const vodInput = ref('')
const offsetInput = ref('')
const playerNameInput = ref('')
const teamInput = ref('A')
const status = ref('')
const players = new Map()
const parentHost = typeof window !== 'undefined' ? window.location.hostname : 'localhost'

const filterTeam = ref('ALL') // ALL | A | B
const manualCols = ref(0)     // 0 = auto (min(count,5)), jinak pevný počet
const selectedPlayers = ref([]) // string[]; prázdné = všichni

const audioElementId   = ref(null) // ID dlaždice, která má hrát zvuk
const activeAudioTeam  = ref('A')  // tým, z něhož má hrát zvuk (první *viditelný* stream)

const isGloballyPaused = ref(false)
// Unmute povolen až po uživatelské interakci (kvůli autoplay policy)
const allowUnmute = ref(false)
// Zobrazení/skrývání horních panelů a pravého panelu kapitol
const showUI = ref(true)

/** Kapitoly */
const chapters = reactive([]) // [{ name: string, tSeconds: number }]
const chapterNameInput = ref('')
const chapterTimeInput = ref('')
const selectedChapterSeconds = ref(null) // číslo; ukládá se do URL
const sortedChapters = computed(() =>
    [...chapters].sort((a, b) => (a?.tSeconds || 0) - (b?.tSeconds || 0))
)

/** Globální čas */
const globalElapsed = ref(0)      // v sekundách
const globalTargetHms = ref('')   // vstup pro manuální skok

/** Správa / vlastník */
const ownerKey = ref('')
const ownerCookieInput = ref('')
const isOwner = computed(() => ownerKey.value && getCookie('vod_owner') === ownerKey.value)

/** Focus modal */
const isFocusOpen = ref(false)
const focusItem = ref(null) // položka, na kterou se zaměřujeme
let focusPlayer = null      // Twitch.Player pro modal

/** ---------- Kolekce dle týmů (bez odmountování) ---------- */
const entriesTeamA = computed(() => allEntries.filter(e => (e.team || '').toUpperCase().trim() === 'A'))
const entriesTeamB = computed(() => allEntries.filter(e => (e.team || '').toUpperCase().trim() === 'B'))

/** ---------- Viditelnost / filtrování ---------- */
const uniquePlayers = computed(() => {
  const set = new Set(allEntries.map(e => (e.playerName || e.label || '').trim()).filter(Boolean))
  return Array.from(set).sort((a,b)=>a.localeCompare(b))
})
function matchesPlayerFilter(e) {
  if (!selectedPlayers.value || selectedPlayers.value.length === 0) return true
  const name = (e.playerName || e.label || '').trim()
  return selectedPlayers.value.includes(name)
}
function isVisible(item, team) {
  if (filterTeam.value === 'A' && team !== 'A') return false
  if (filterTeam.value === 'B' && team !== 'B') return false
  return matchesPlayerFilter(item)
}
function visibleCount(list) {
  if (!Array.isArray(list)) return 0
  const team = list === entriesTeamA.value ? 'A' : 'B'
  return list.filter(e => isVisible(e, team)).length
}

/** ---------- Grid ---------- */
function gridStyleFor(count) {
  const cols = manualCols.value > 0 ? manualCols.value : Math.max(1, Math.min(count || 1, 5))
  return { display: 'grid', gridTemplateColumns: `repeat(${cols}, minmax(0, 1fr))` }
}

const combinedVisibleEntries = computed(() => {
  const list = [...entriesTeamA.value, ...entriesTeamB.value]
  return list.filter(e => isVisible(e, (e.team || '').toUpperCase().trim()))
})
const totalVisibleCount = computed(() => combinedVisibleEntries.value.length)
const gridColumnsStyle = computed(() => (
  showUI.value
    ? { gridTemplateColumns: '1fr minmax(260px, 18vw)', alignItems: 'start' }
    : { gridTemplateColumns: '1fr', alignItems: 'start' }
))

/** ---------- Utility ---------- */
function parseHMS(value) {
  if (value == null || value === '') return 0
  const s = String(value).trim()
  if (/^\d+$/.test(s)) return Math.max(0, parseInt(s, 10))
  const parts = s.split(':').map(Number).filter(n => !Number.isNaN(n))
  if (parts.length === 1) return Math.max(0, parts[0])
  if (parts.length === 2) return Math.max(0, parts[0] * 60 + parts[1])
  if (parts.length === 3) return Math.max(0, parts[0] * 3600 + parts[1] * 60 + parts[2])
  return 0
}
function formatHMS(total) {
  const t = Math.max(0, Math.floor(total || 0))
  const h = Math.floor(t / 3600)
  const m = Math.floor((t % 3600) / 60)
  const s = t % 60
  const pad = (n)=>String(n).padStart(2,'0')
  return h > 0 ? `${h}:${pad(m)}:${pad(s)}` : `${m}:${pad(s)}`
}

/** Cookies */
function setCookie(name, value, days = 365) {
  const d = new Date()
  d.setTime(d.getTime() + (days*24*60*60*1000))
  document.cookie = `${name}=${encodeURIComponent(value)}; expires=${d.toUTCString()}; path=/; SameSite=Lax`
}
function getCookie(name) {
  const m = document.cookie.match(new RegExp('(^| )' + name + '=([^;]+)'))
  return m ? decodeURIComponent(m[2]) : ''
}
function applyOwnerCookie() {
  const v = ownerCookieInput.value.trim()
  if (!v) return
  setCookie('vod_owner', v, 365)
}

/** ---------- Overlay helper ---------- */
function flashOverlayAll(text, targets = allEntries) {
  for (const item of targets) {
    if (item._overlayTimer) { clearTimeout(item._overlayTimer); item._overlayTimer = null }
    item.overlayText = text
    item.overlayVisible = true
    item._overlayTimer = setTimeout(() => {
      item.overlayVisible = false
      item._overlayTimer = null
    }, 500)
  }
}

/** ---------- Twitch SDK ---------- */
function loadTwitchSDK() {
  return new Promise((resolve, reject) => {
    if (window.Twitch && window.Twitch.Embed) return resolve(window.Twitch)
    const script = document.createElement('script')
    script.src = 'https://player.twitch.tv/js/embed/v1.js'
    script.async = true
    script.onload = () => resolve(window.Twitch)
    script.onerror = reject
    document.body.appendChild(script)
  })
}

/** ---------- Player lifecycle ---------- */
async function ensurePlayers() {
  await loadTwitchSDK()
  await nextTick()

  for (const item of allEntries) {
    if (players.has(item.elementId)) continue
    const mountEl = document.getElementById(item.elementId)
    if (!mountEl) continue

    const player = new window.Twitch.Player(item.elementId, {
      width: '100%',
      height: '100%',
      parent: [parentHost],
      video: item.id,
      muted: item.muted,
    })
    player.addEventListener(window.Twitch.Player.READY, () => {
      try { player.setMuted(item.muted) } catch {}
    })
    players.set(item.elementId, player)
  }

  // cleanup orphaned
  for (const [key, p] of Array.from(players.entries())) {
    if (!allEntries.find(e => e.elementId === key)) {
      try { p.pause?.() } catch {}
      players.delete(key)
    }
  }
}

/** ---------- Audio řízení (jen jeden stream hraje) ---------- */
function enforceSingleAudio() {
  for (const e of allEntries) {
    const p = players.get(e.elementId)
    const shouldMute = !(audioElementId.value && e.elementId === audioElementId.value && allowUnmute.value)
    e.muted = shouldMute
    try { p?.setMuted?.(shouldMute) } catch {}
  }
}
function setAudio(item) {
  allowUnmute.value = true
  audioElementId.value = item.elementId
  enforceSingleAudio()
  saveStateToURL()
}
function applyTeamAudio() {
  allowUnmute.value = true
  const team = (activeAudioTeam.value || 'A').toUpperCase()
  const list = team === 'A' ? entriesTeamA.value : entriesTeamB.value
  const firstVisible = list.find(e => isVisible(e, team))
  audioElementId.value = firstVisible ? firstVisible.elementId : null
  enforceSingleAudio()
  saveStateToURL()
}

/** ---------- Přidávání / mazání ---------- */
async function addVod() {
  status.value = ''
  let raw = vodInput.value.trim()
  const team = (teamInput.value || '').toUpperCase().trim()
  if (!raw) { status.value = 'Zadej VOD.'; return }
  if (!(team === 'A' || team === 'B')) { status.value = 'Vyber tým (A nebo B) – je povinný.'; return }

  const urlMatch = raw.match(/videos\/(\d+)/)
  if (urlMatch) raw = `v${urlMatch[1]}`
  if (!/^v\d+$/.test(raw)) {
    status.value = 'Zadej prosím platné VOD (např. v123456789 nebo celé URL Twitche).'
    return
  }
  const offsetHms = offsetInput.value.trim() || '0:00'
  const offsetSeconds = parseHMS(offsetHms)
  const item = {
    id: raw,
    label: `VOD ${raw}`,
    playerName: playerNameInput.value?.trim() || '',
    elementId: `tw-${Date.now()}-${Math.random().toString(16).slice(2)}`,
    muted: true,
    offsetSeconds,
    offsetHms,
    team,
    overlayText: '',
    overlayVisible: false,
    _overlayTimer: null,
  }
  allEntries.push(item)
  // pokud je aktivní výběr hráčů, automaticky přidej nového hráče do výběru,
  // aby po přidání nebyl skrytý filtrem
  if (selectedPlayers.value && selectedPlayers.value.length > 0) {
    const nm = (item.playerName || item.label || '').trim()
    if (nm && !selectedPlayers.value.includes(nm)) {
      selectedPlayers.value = [...selectedPlayers.value, nm]
    }
  }
  vodInput.value = ''
  offsetInput.value = ''
  playerNameInput.value = ''
  teamInput.value = team

  await ensurePlayers()

  if ((activeAudioTeam.value || 'A').toUpperCase() === team) {
    applyTeamAudio()
  } else {
    enforceSingleAudio()
  }
  saveStateToURL()
}
function removeByElementId(elId) {
  const idx = allEntries.findIndex(e => e.elementId === elId)
  if (idx >= 0) {
    const [removed] = allEntries.splice(idx, 1)
    if (removed && players.has(removed.elementId)) {
      try { players.get(removed.elementId)?.pause?.() } catch {}
      players.delete(removed.elementId)
    }
    if (audioElementId.value === removed.elementId) {
      audioElementId.value = null
      applyTeamAudio()
    } else {
      enforceSingleAudio()
    }
    saveStateToURL()
  }
}
function toggleMute(item) {
  if (audioElementId.value === item.elementId) {
    audioElementId.value = null
    applyTeamAudio()
  } else {
    setAudio(item)
  }
  saveStateToURL()
}

/** ---------- Playback ---------- */
function seekAllToOffsets() {
  for (const item of allEntries) {
    const p = players.get(item.elementId)
    if (!p) continue
    try { p.seek(item.offsetSeconds || 0) } catch {}
  }
}
function restartAll() {
  seekAllToOffsets()
  let acted = 0
  for (const item of allEntries) {
    const p = players.get(item.elementId)
    if (!p) continue
    const shouldMute = !(audioElementId.value && item.elementId === audioElementId.value)
    item.muted = shouldMute
    try { p.setMuted(shouldMute) } catch {}
    try { p.play(); acted++ } catch {}
    setTimeout(() => { try { p.play() } catch {} }, 300)
  }
  isGloballyPaused.value = false
  updateSelectedChapterByTime(0)
  flashOverlayAll('⏮️ Restart')
  status.value = `Spuštěno od offsetu u ${acted} přehrávačů.`
}
function pauseAll() {
  let paused = 0
  // pause all grid players
  for (const [, p] of players) {
    try { p.pause?.(); paused++ } catch {}
    // belt-and-suspenders: try pausing again shortly after (some iframes ignore first call)
    setTimeout(() => { try { p.pause?.() } catch {} }, 150)
  }
  // also pause focus player if open
  if (focusPlayer) {
    try { focusPlayer.pause?.() } catch {}
    setTimeout(() => { try { focusPlayer.pause?.() } catch {} }, 150)
  }
  isGloballyPaused.value = true
  flashOverlayAll('⏸️ Pause')
  status.value = `Pozastaveno ${paused} přehrávačů.`
}
function resumeAll() {
  let acted = 0

  // 0) Ujisti se, že máme inicializované přehrávače
  ensurePlayers().then(() => {
    // 1) Všem nastavíme mute (autoplay policy)
    for (const [, p] of players) {
      try { p.setMuted(true) } catch {}
    }
    if (focusPlayer) {
      try { focusPlayer.setMuted(true) } catch {}
    }

    // 2) Zkusíme spustit přehrávání u všech hráčů
    for (const [, p] of players) {
      try { p.play(); acted++ } catch {}
      // druhý pokus o chvilku později
      setTimeout(() => { try { p.play() } catch {} }, 200)
      // třetí pokus: malý seek „šťouch“ a znovu play (některé iframy potřebují změnu pozice)
      setTimeout(() => {
        try {
          const cur = typeof p.getCurrentTime === 'function' ? (p.getCurrentTime() || 0) : 0
          if (cur >= 0) {
            try { p.seek(Math.max(0, cur + 0.001)) } catch {}
            try { p.play() } catch {}
          }
        } catch {}
      }, 350)
    }
    if (focusPlayer) {
      try { focusPlayer.play?.() } catch {}
      setTimeout(() => { try { focusPlayer.play?.() } catch {} }, 200)
      setTimeout(() => {
        try {
          const cur = typeof focusPlayer.getCurrentTime === 'function' ? (focusPlayer.getCurrentTime() || 0) : 0
          if (cur >= 0) {
            try { focusPlayer.seek(Math.max(0, cur + 0.001)) } catch {}
            try { focusPlayer.play?.() } catch {}
          }
        } catch {}
      }, 350)
    }

    // 3) Po chvíli znovu vynutíme pouze jeden audio stream
    setTimeout(() => {
      allowUnmute.value = true
      enforceSingleAudio()
    }, 500)

    isGloballyPaused.value = false
    flashOverlayAll('▶️ Play')
    status.value = `Pokračuji u ${acted} přehrávačů.`
  })
}
async function seekAll(deltaSeconds) {
  await loadTwitchSDK()
  let changed = 0
  for (const item of allEntries) {
    const p = players.get(item.elementId)
    if (!p) continue
    try {
      const cur = typeof p.getCurrentTime === 'function' ? (p.getCurrentTime() || 0) : 0
      p.seek(Math.max(0, cur + deltaSeconds))
      changed++
    } catch {}
  }
  updateSelectedChapterByTime(getGlobalElapsedSeconds())
  flashOverlayAll(deltaSeconds >= 0 ? (deltaSeconds >= 60 ? '⏩ +60s' : '⏩ +5s') : (Math.abs(deltaSeconds) >= 60 ? '⏪ −60s' : '⏪ −5s'))
  status.value = `${deltaSeconds >= 0 ? 'Posunuto vpřed' : 'Vráceno zpět'} o ${Math.abs(deltaSeconds)}s u ${changed} přehrávačů.`
}

/** ---------- Globální čas, kapitoly, sync ---------- */
function getGlobalElapsedSeconds() {
  for (const entry of allEntries) {
    const p = players.get(entry.elementId)
    if (p && typeof p.getCurrentTime === 'function') {
      try {
        const cur = p.getCurrentTime() || 0
        return Math.max(0, Math.floor(cur - (entry.offsetSeconds || 0)))
      } catch {}
    }
  }
  return 0
}
function tickGlobalElapsed() {
  if (isGloballyPaused.value) return
  globalElapsed.value = getGlobalElapsedSeconds()
  updateSelectedChapterByTime(globalElapsed.value)
}
function syncToCurrentGlobal() {
  const t = getGlobalElapsedSeconds()
  for (const item of allEntries) {
    const p = players.get(item.elementId)
    if (!p) continue
    const target = Math.max(0, (item.offsetSeconds || 0) + t)
    try { p.seek(target) } catch {}
  }
  if (isFocusOpen.value && focusItem.value && focusPlayer) {
    const target = Math.max(0, (focusItem.value.offsetSeconds || 0) + t)
    try { focusPlayer.seek(target) } catch {}
  }
  flashOverlayAll('🔄 Sync')
}
function seekAllToGlobalTarget() {
  const t = parseHMS(globalTargetHms.value)
  for (const item of allEntries) {
    const p = players.get(item.elementId)
    if (!p) continue
    const target = Math.max(0, (item.offsetSeconds || 0) + t)
    try { p.seek(target) } catch {}
  }
  if (isFocusOpen.value && focusItem.value && focusPlayer) {
    const target = Math.max(0, (focusItem.value.offsetSeconds || 0) + t)
    try { focusPlayer.seek(target) } catch {}
  }
  updateSelectedChapterByTime(t)
  flashOverlayAll('🔄 Sync')
}

/** Kapitoly */
function addChapterManual() {
  const name = chapterNameInput.value.trim() || 'Kapitola'
  const t = parseHMS(chapterTimeInput.value)
  const ch = { name, tSeconds: t }
  chapters.push(ch)
  chapterNameInput.value = ''
  chapterTimeInput.value = ''
  setSelectedChapter(ch) // hned označit a uložit do URL
}
function addChapterNow() {
  const t = getGlobalElapsedSeconds()
  const suggested = chapterNameInput.value.trim() || `Kapitola ${formatHMS(t)}`
  const name = window.prompt('Název kapitoly:', suggested) ?? ''
  const final = name.trim() || suggested
  const ch = { name: final, tSeconds: t }
  chapters.push(ch)
  chapterNameInput.value = ''
  chapterTimeInput.value = ''
  setSelectedChapter(ch) // rovnou označit a uložit
}
function renameChapter(sortedIndex) {
  const ch = sortedChapters.value[sortedIndex]
  if (!ch) return
  const curName = ch.name || 'Kapitola'
  const newName = window.prompt('Nový název kapitoly:', curName)
  if (newName == null) return
  // najít původní objekt v chapters
  const pos = chapters.findIndex(c => c.tSeconds === ch.tSeconds && c.name === ch.name)
  if (pos !== -1) {
    chapters[pos] = { ...chapters[pos], name: newName.trim() || curName }
    if (selectedChapterSeconds.value === ch.tSeconds) {
      saveStateToURL()
    }
  }
}
function editChapterTime(sortedIndex) {
  const ch = sortedChapters.value[sortedIndex]
  if (!ch) return
  const current = formatHMS(Math.max(0, Math.floor(ch.tSeconds || 0)))
  const input = window.prompt('Nový čas kapitoly (HH:MM:SS nebo sekundy):', current)
  if (input == null) return
  const seconds = parseHMS(input)
  // najít původní objekt v chapters a aktualizovat
  const pos = chapters.findIndex(c => c.tSeconds === ch.tSeconds && c.name === ch.name)
  if (pos !== -1) {
    const newT = Math.max(0, Math.floor(seconds))
    chapters[pos] = { ...chapters[pos], tSeconds: newT }
    if (selectedChapterSeconds.value === ch.tSeconds) {
      selectedChapterSeconds.value = newT
    }
    saveStateToURL()
  }
}
function removeChapter(sortedIndex) {
  const ch = sortedChapters.value[sortedIndex]
  const pos = chapters.findIndex(c => c.tSeconds === ch.tSeconds && c.name === ch.name)
  if (pos !== -1) chapters.splice(pos, 1)
  if (!sortedChapters.value.some(c => c.tSeconds === selectedChapterSeconds.value)) {
    selectedChapterSeconds.value = null
  }
  saveStateToURL()
}
function goToChapter(ch) {
  for (const item of allEntries) {
    const p = players.get(item.elementId)
    if (!p) continue
    const target = Math.max(0, (item.offsetSeconds || 0) + (ch.tSeconds || 0))
    try { p.seek(target) } catch {}
  }
  if (isFocusOpen.value && focusItem.value && focusPlayer) {
    const target = Math.max(0, (focusItem.value.offsetSeconds || 0) + (ch.tSeconds || 0))
    try { focusPlayer.seek(target) } catch {}
  }
  setSelectedChapter(ch) // uložit do URL + zvýraznit
}
function setSelectedChapter(ch) {
  selectedChapterSeconds.value = Math.max(0, Math.floor(ch.tSeconds || 0))
  saveStateToURL()
}
function updateSelectedChapterByTime(tSeconds) {
  const t = Math.max(0, Math.floor(tSeconds || 0))
  let candidate = null
  for (const ch of sortedChapters.value) {
    if ((ch.tSeconds || 0) <= t) candidate = ch
    else break
  }
  const newSel = candidate ? Math.max(0, Math.floor(candidate.tSeconds || 0)) : null
  if (newSel !== selectedChapterSeconds.value) {
    selectedChapterSeconds.value = newSel
    saveStateToURL()
  }
}

/** ---------- Editace offsetu na dlaždici ---------- */
function updateTileOffset(item, hms) {
  const secs = parseHMS(hms)
  item.offsetHms = hms || '0:00'
  item.offsetSeconds = secs
  const p = players.get(item.elementId)
  if (p) { try { p.seek(secs) } catch {} }
  if (isFocusOpen.value && focusItem.value?.elementId === item.elementId && focusPlayer) {
    try { focusPlayer.seek(secs) } catch {}
  }
  saveStateToURL()
}

/** ---------- Focus modal ---------- */
async function openFocus(item) {
  await loadTwitchSDK()
  focusItem.value = item
  isFocusOpen.value = true
  await nextTick()

  if (focusPlayer) {
    try { focusPlayer.pause?.() } catch {}
    focusPlayer = null
  }
  focusPlayer = new window.Twitch.Player('focus-player', {
    width: '100%',
    height: '100%',
    parent: [parentHost],
    video: item.id,
    muted: true, // zvětšené okno je vždy beze zvuku
  })
  focusPlayer.addEventListener(window.Twitch.Player.READY, () => {
    try {
      const t = getGlobalElapsedSeconds()
      const target = Math.max(0, (focusItem.value?.offsetSeconds || 0) + t)
      focusPlayer.setMuted(true)
      focusPlayer.seek(target)
      if (!isGloballyPaused.value) focusPlayer.play()
    } catch {}
  })
}
function syncFocusToGlobal() {
  if (!isFocusOpen.value || !focusItem.value || !focusPlayer) return
  const t = getGlobalElapsedSeconds()
  const target = Math.max(0, (focusItem.value.offsetSeconds || 0) + t)
  try { focusPlayer.seek(target) } catch {}
}
function closeFocus() {
  isFocusOpen.value = false
  focusItem.value = null
  if (focusPlayer) {
    try { focusPlayer.pause?.() } catch {}
    focusPlayer = null
  }
}

/** ---------- URL persist ---------- */
function getSerializableState() {
  return {
    title: pageTitle.value || '',
    entries: allEntries.map(e => ({
      id: e.id,
      label: e.label,
      playerName: e.playerName || '',
      offsetSeconds: e.offsetSeconds,
      offsetHms: e.offsetHms,
      team: (e.team || '').toUpperCase().trim(),
    })),
    filterTeam: filterTeam.value,
    manualCols: manualCols.value,
    selectedPlayers: selectedPlayers.value || [],
    chapters: chapters.map(c => ({ name: c.name || '', tSeconds: Math.max(0, Math.floor(c.tSeconds || 0)) })),
    selectedChapterSeconds: selectedChapterSeconds.value ?? null,
    ownerKey: ownerKey.value || '',
    activeAudioTeam: activeAudioTeam.value || 'A',
    audioElementId: audioElementId.value || null,
  }
}
function saveStateToURL() {
  const json = JSON.stringify(getSerializableState())
  const qp = new URLSearchParams(window.location.search)
  qp.set('state', encodeURIComponent(json))
  const url = `${window.location.pathname}?${qp.toString()}${window.location.hash || ''}`
  window.history.replaceState(null, '', url)
}
function loadStateFromURL() {
  const qp = new URLSearchParams(window.location.search)
  const raw = qp.get('state')
  if (!raw) return null
  try {
    const obj = JSON.parse(decodeURIComponent(raw))
    if (typeof obj.title === 'string') pageTitle.value = obj.title || 'Twitch VOD Multi-Stream Sync (Vue)'
    if (obj && Array.isArray(obj.entries)) {
      allEntries.splice(0, allEntries.length)
      for (const it of obj.entries) {
        const team = (it.team || '').toUpperCase().trim()
        allEntries.push({
          id: it.id,
          label: it.label || `VOD ${it.id}`,
          playerName: it.playerName || '',
          elementId: `tw-${Date.now()}-${Math.random().toString(16).slice(2)}`,
          muted: true,
          offsetSeconds: typeof it.offsetSeconds === 'number' ? it.offsetSeconds : parseHMS(it.offsetHms || '0:00'),
          offsetHms: it.offsetHms || '0:00',
          team: team === 'A' || team === 'B' ? team : 'A',
          overlayText: '',
          overlayVisible: false,
          _overlayTimer: null,
        })
      }
    }
    if (['ALL','A','B'].includes(obj.filterTeam)) filterTeam.value = obj.filterTeam
    if ([0,1,2,3,4,5].includes(obj.manualCols)) manualCols.value = obj.manualCols
    if (Array.isArray(obj.selectedPlayers)) selectedPlayers.value = obj.selectedPlayers
    if (Array.isArray(obj.chapters)) {
      chapters.splice(0, chapters.length, ...obj.chapters.map(c => ({
        name: (c && typeof c.name === 'string') ? c.name : '',
        tSeconds: Math.max(0, Math.floor(c?.tSeconds || 0)),
      })))
    }
    if (typeof obj.selectedChapterSeconds === 'number') {
      selectedChapterSeconds.value = Math.max(0, Math.floor(obj.selectedChapterSeconds))
    }
    if (typeof obj.ownerKey === 'string') ownerKey.value = obj.ownerKey
    if (typeof obj.activeAudioTeam === 'string' && ['A','B'].includes(obj.activeAudioTeam.toUpperCase())) {
      activeAudioTeam.value = obj.activeAudioTeam.toUpperCase()
    }
    if (typeof obj.audioElementId === 'string') audioElementId.value = obj.audioElementId
    return obj
  } catch (e) {
    console.warn('Nepodařilo se načíst stav z URL:', e)
    return null
  }
}

/** ---------- lifecycle ---------- */
let timerId = null
// Automatické spuštění (ztlumeno) u všech přehrávačů po mountu
async function autoStartAllMuted() {
  await ensurePlayers()
  let started = 0
  for (const [, p] of players) {
    try { p.setMuted(true) } catch {}
    try { p.play(); started++ } catch {}
    setTimeout(() => { try { p.play() } catch {} }, 200)
  }
  isGloballyPaused.value = false
  status.value = `Automatické spuštění (ztlumeno) u ${started} přehrávačů.`
}
onMounted(async () => {
  const obj = loadStateFromURL()
  await ensurePlayers()
  await autoStartAllMuted()
  window.addEventListener('keydown', handleKey, { passive: false })
  timerId = setInterval(tickGlobalElapsed, 1000)

  // Pokud je v URL vybraná kapitola, hned na ni přeskoč
  if (obj && typeof obj.selectedChapterSeconds === 'number') {
    const t = Math.max(0, Math.floor(obj.selectedChapterSeconds))
    for (const item of allEntries) {
      const p = players.get(item.elementId)
      if (!p) continue
      const target = Math.max(0, (item.offsetSeconds || 0) + t)
      try { p.seek(target) } catch {}
    }
  }
})
watch([pageTitle, allEntries, filterTeam, manualCols, chapters, ownerKey, activeAudioTeam, selectedPlayers, selectedChapterSeconds], async () => {
  await ensurePlayers()
  enforceSingleAudio()
  clearTimeout(saveStateToURL._t)
  saveStateToURL._t = setTimeout(saveStateToURL, 150)
}, { deep: true })
onUnmounted(() => {
  window.removeEventListener('keydown', handleKey)
  for (const [, p] of players) { try { p.pause?.() } catch {} }
  players.clear()
  if (timerId) { clearInterval(timerId); timerId = null }
  if (focusPlayer) { try { focusPlayer.pause?.() } catch {}; focusPlayer = null }
})

/** ---------- Hotkeys ---------- */
function handleKey(e) {
  const t = e.target
  // Při psaní do textových polí nezasahujeme
  if (t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA' || t.isContentEditable)) return
  // Když je fokus na SELECT/BUTTON/A, hotkeys necháme fungovat (případné defaulty zrušíme níže u konkrétních kláves)

  if (e.code === 'Space') {
    e.preventDefault()
    if (isGloballyPaused.value) resumeAll()
    else pauseAll()
    return
  }
  if (e.code === 'ArrowLeft') { e.preventDefault(); seekAll(-5); return }
  if (e.code === 'ArrowRight'){ e.preventDefault(); seekAll(5);  return }
  if (e.code === 'Digit1') {
    allowUnmute.value = true
    activeAudioTeam.value = 'A'; applyTeamAudio(); flashOverlayAll('🔊 A'); return
  }
  if (e.code === 'Digit2') {
    allowUnmute.value = true
    activeAudioTeam.value = 'B'; applyTeamAudio(); flashOverlayAll('🔊 B'); return
  }
  if (e.code === 'KeyS')   { e.preventDefault(); syncToCurrentGlobal(); return }
  if (e.code === 'Escape' && isFocusOpen.value) { e.preventDefault(); closeFocus(); return }
}

/** ---------- Akce pro filtr hráčů ---------- */
function selectAllPlayers() { selectedPlayers.value = [...uniquePlayers.value] }
function clearPlayerSelection() { selectedPlayers.value = [] }

/** ---------- Lokální komponenta Tile (render funkce) ---------- */
const Tile = defineComponent({
  name: 'Tile',
  props: {
    item: { type: Object, required: true },
    audioElementId: { type: String, default: null }
  },
  emits: ['set-audio', 'toggle-mute', 'remove', 'update-offset', 'focus'],
  data() {
    return {
      editOffset: false,
      offsetDraft: this.item.offsetHms || '0:00'
    }
  },
  watch: {
    'item.offsetHms'(v) {
      if (!this.editOffset) this.offsetDraft = v || '0:00'
    }
  },
  methods: {
    saveOffset() {
      this.$emit('update-offset', this.offsetDraft)
      this.editOffset = false
    }
  },
  render() {
    const it = this.item
    const headerLeft = h('div', {class: 'text-sm text-neutral-300'}, [
      h('div', {class: 'font-semibold text-neutral-100'}, it.playerName || it.label),
      h('div', {class: 'opacity-80 flex flex-wrap items-center gap-2'}, [
        h('span', `Start ve videu: ${it.offsetHms || '0:00'}`),
        h('button', {
          class: 'text-xs underline opacity-75 hover:opacity-100',
          onClick: () => (this.editOffset = !this.editOffset)
        }, this.editOffset ? 'Zavřít' : 'Upravit čas'),
        h('span', {
          class: `ml-2 inline-block text-xs px-2 py-0.5 rounded-full ${it.team === 'A' ? 'bg-sky-700' : 'bg-emerald-700'}`
        }, `Tým ${it.team}`),
        this.audioElementId === it.elementId
            ? h('span', {class: 'text-xs opacity-90'}, '🔊 audio') : null,
      ]),
      this.editOffset ? h('div', {class: 'mt-2 flex items-center gap-2'}, [
        h('input', {
          class: 'w-32 bg-neutral-800 rounded-xl px-3 py-1 outline-none text-sm',
          placeholder: 'HH:MM:SS',
          value: this.offsetDraft,
          onInput: e => (this.offsetDraft = e.target.value),
          onKeydown: e => e.stopPropagation(),
        }),
        h('button', {
          class: 'px-3 py-1 rounded-xl bg-neutral-700 hover:bg-neutral-600 text-sm',
          onClick: this.saveOffset
        }, 'Uložit'),
      ]) : null
    ])

    const headerRight = h('div', {class: 'flex gap-2'}, [
      h('button', {
        class: 'px-3 py-1 rounded-xl bg-neutral-700 hover:bg-neutral-600 text-sm',
        onClick: () => this.$emit('focus')
      }, 'Zvětšit'),
      h('button', {
        class: 'px-3 py-1 rounded-xl bg-neutral-700 hover:bg-neutral-600 text-sm',
        onClick: () => this.$emit('set-audio')
      }, this.audioElementId === it.elementId ? 'Audio ✓' : 'Audio'),
      h('button', {
        class: 'px-3 py-1 rounded-xl bg-neutral-700 hover:bg-neutral-600 text-sm',
        onClick: () => this.$emit('toggle-mute')
      }, it.muted ? 'Unmute' : 'Mute'),
      h('button', {
        class: 'px-3 py-1 rounded-xl bg-red-600 hover:bg-red-500 text-sm',
        onClick: () => this.$emit('remove')
      }, 'Odebrat'),
    ])

    const overlay = h('div', {
      class: 'pointer-events-none absolute inset-0 flex items-center justify-center transition-opacity duration-150',
      style: {opacity: it.overlayVisible ? '1' : '0', background: 'rgba(0,0,0,0.15)'}
    }, [h('div', {class: 'text-3xl md:text-4xl font-semibold drop-shadow'}, it.overlayText || '')])

    const playerDiv = h('div', {id: it.elementId, class: 'absolute inset-0 bg-black rounded-xl overflow-hidden'})

    const videoWrapper = h('div', {class: 'relative w-full aspect-video rounded-xl overflow-hidden'}, [overlay, playerDiv])

    return h('div', {class: 'relative bg-neutral-900 rounded-2xl p-3 shadow-lg flex flex-col gap-2'}, [
      h('div', {class: 'flex items-center justify-between'}, [headerLeft, headerRight]),
      videoWrapper
    ])
  }
})
</script>

<style scoped>
/* nic speciálního navíc */

.chapters-panel {
  /* sticky top-4 already keeps it pinned; limit height and enable internal scroll */
  max-height: calc(100vh - 7rem);
  overflow: auto;
}
/* optional, subtle scrollbar */
.chapters-panel::-webkit-scrollbar { width: 8px; height: 8px; }
.chapters-panel::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.15); border-radius: 9999px; }
.chapters-panel::-webkit-scrollbar-track { background: transparent; }
</style>
