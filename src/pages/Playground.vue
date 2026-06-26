<template>
  <div class="min-h-screen bg-[radial-gradient(circle_at_top,_rgba(56,189,248,0.17),transparent_28%),radial-gradient(circle_at_bottom_right,_rgba(168,85,247,0.18),transparent_25%),#020617] px-4 py-10 text-slate-100">
    <div class="mx-auto max-w-5xl space-y-8 rounded-[2.5rem] border border-white/10 bg-slate-950/90 p-8 shadow-[0_40px_120px_-45px_rgba(56,189,248,0.45)] backdrop-blur-xl">
      <div class="grid gap-6 lg:grid-cols-[1.2fr_0.8fr] items-start">
        <div class="space-y-4">
          <div class="space-y-2">
            <p class="text-sm uppercase tracking-[0.35em] text-cyan-300/80">Playground</p>
            <h1 class="text-4xl font-black tracking-tight text-white sm:text-5xl">Studio drive for RPC and game tests.</h1>
            <p class="max-w-xl text-slate-300 leading-8">Trigger Discord RPC calls, watch logging activity, and experiment with backend game actions in a sleek control panel.</p>
          </div>
          <div class="rounded-[2rem] border border-cyan-500/15 bg-cyan-500/5 p-5 shadow-[inset_0_0_0_1px_rgba(56,189,248,0.1)]">
            <p class="text-xs uppercase tracking-[0.35em] text-cyan-200/80">Owner</p>
            <p class="mt-3 text-lg font-semibold text-white">Arnav Badola</p>
          </div>
          <button class="inline-flex items-center justify-center rounded-full bg-gradient-to-r from-cyan-500 to-violet-500 px-6 py-3 text-sm font-semibold text-slate-950 shadow-2xl shadow-cyan-500/20 transition hover:-translate-y-0.5"
            @click="discordTest">
            {{ isConnected ? 'Disconnect' : 'Connect Test' }}
          </button>
          <div class="text-sm text-slate-400">Connected: <span class="font-semibold text-white">{{ isConnected ? 'Yes' : 'No' }}</span></div>
        </div>

        <div class="rounded-[2rem] border border-slate-800/90 bg-slate-950/85 p-6 shadow-[0_25px_60px_-30px_rgba(15,23,42,0.8)]">
          <div class="flex items-center justify-between mb-5">
            <div>
              <h2 class="text-lg font-semibold text-white">Session Console</h2>
              <p class="text-sm text-slate-400">Live RPC and action log output</p>
            </div>
            <button class="rounded-full bg-slate-900/80 px-4 py-2 text-sm text-slate-200 transition hover:bg-slate-800"
              @click="clearLogs">Clear Logs</button>
          </div>
          <div class="max-h-80 overflow-y-auto rounded-[1.75rem] border border-slate-800/90 bg-slate-950/90 p-4 text-sm leading-6">
            <div v-if="logs.length === 0" class="text-slate-500">No logs available. Execute an RPC test to see activity.</div>
            <ul v-else class="space-y-3">
              <li v-for="(log, index) in logs" :key="index" class="rounded-2xl bg-slate-900/70 px-4 py-3">
                <div class="flex flex-wrap items-center gap-3 text-xs text-slate-400">
                  <span>{{ new Date(log.timestamp).toLocaleString() }}</span>
                  <span class="rounded-full border border-slate-700/80 bg-slate-900/60 px-2 py-1" :class="{
                    'text-sky-300 border-sky-400/20': log.type === 'info',
                    'text-rose-300 border-rose-400/20': log.type === 'error',
                    'text-amber-300 border-amber-400/20': log.type === 'warning',
                    'text-emerald-300 border-emerald-400/20': log.type === 'debug'
                  }">{{ log.type.toUpperCase() }}</span>
                </div>
                <p class="mt-2 text-slate-200">{{ log.message }}</p>
              </li>
            </ul>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
// Owner: Arnav Badola
import { onMounted, ref } from 'vue';
import { invoke } from '@tauri-apps/api/core';
import { emit } from '@tauri-apps/api/event';
import { useGlobalState } from '@/composables/app-state';

const ActivityKind = {
  Playing: 0,
  Listening: 2,
  Watching: 3,
  Competing: 5
} as const;

const isConnected = ref(false);

const { logs, addLog, clearLogs } = useGlobalState();

function discordTest() {
  const appIdCode = '1361728268088381706';

  if (isConnected.value) {
    console.log('Disconnecting from Discord');
    emit('event_disconnect');
    isConnected.value = false;
    return;
  }

  invoke('connect_to_discord_rpc_3', {
    activity_json: JSON.stringify({
      app_id: appIdCode,
      details: 'Game Hub RPC',
      state: '/game-hub',
      activity_kind: ActivityKind.Watching,
      timestamp: createAgoTimestamp('1h 30m')
    }),
    action: 'connect'
  });
  isConnected.value = true;
}

function createAgoTimestamp(input: string) {
  const time = input.split(' ');
  let hours = 0;
  let minutes = 0;

  for (let i = 0; i < time.length; i++) {
    if (time[i].includes('h')) {
      hours = parseInt(time[i]);
    } else if (time[i].includes('m')) {
      minutes = parseInt(time[i]);
    }
  }

  const date = new Date();
  date.setHours(date.getHours() - hours);
  date.setMinutes(date.getMinutes() - minutes);

  return Math.floor(date.getTime() / 1000);
}

onMounted(() => {
});
</script>

<style scoped></style>
