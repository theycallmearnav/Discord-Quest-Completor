<script setup lang="ts">
import { Pages, useGlobalState } from '@/composables/app-state';

// Owner: Arnav Badola
// Layout component for consistent page structure

const appState = useGlobalState();
const { page, setPage } = appState;

</script>

<template>
  <div class="flex flex-col min-h-screen bg-slate-950 text-slate-100">
    <header class="relative overflow-visible border-b border-slate-800/70 bg-slate-950/95 shadow-[0_30px_100px_-45px_rgba(15,23,42,0.75)] backdrop-blur-xl z-50">
      <div class="pointer-events-none absolute -left-16 top-8 h-36 w-36 rounded-full bg-cyan-500/18 blur-3xl"></div>
      <div class="pointer-events-none absolute -right-16 bottom-8 h-44 w-44 rounded-full bg-violet-500/18 blur-3xl"></div>
      <div class="container relative mx-auto max-w-7xl px-6 py-4 flex flex-col gap-4 md:flex-row md:items-center md:justify-between">
        <div class="flex items-center gap-4">
          <div class="flex h-12 w-12 items-center justify-center rounded-3xl bg-gradient-to-br from-cyan-500 to-indigo-500 text-white shadow-lg shadow-cyan-500/20">
            <span class="text-xl font-black">A</span>
          </div>
          <div>
            <h2 class="text-xl font-bold tracking-tight text-white">ORIXA</h2>
            <p class="text-sm text-slate-400">Owned by Arnav Badola</p>
          </div>
        </div>
        <nav>
          <ul class="flex flex-wrap items-center gap-3 text-sm font-medium">
            <li>
              <button type="button"
                class="rounded-full px-4 py-2 transition-all duration-200"
                :class="page === Pages.HOME ? 'bg-cyan-500 text-slate-950 shadow-lg shadow-cyan-500/20' : 'text-slate-300 hover:text-white hover:bg-slate-800/70'"
                @click="setPage(Pages.HOME)"
              >
                Home
              </button>
            </li>
            <li>
              <button type="button"
                class="rounded-full px-4 py-2 transition-all duration-200"
                :class="page === Pages.PLAYGROUND ? 'bg-cyan-500 text-slate-950 shadow-lg shadow-cyan-500/20' : 'text-slate-300 hover:text-white hover:bg-slate-800/70'"
                @click="setPage(Pages.PLAYGROUND)"
              >
                Playground
              </button>
            </li>
          </ul>
        </nav>
      </div>
    </header>
    
    <main class="flex-grow overflow-y-auto">
      <slot></slot>
    </main>
    
    <footer class="bg-slate-950 border-t border-slate-800 mt-auto">
      <div class="container mx-auto px-6 py-4 text-center text-sm text-slate-400">
        &copy; 2026 Arnav Badola · discord quest completor
      </div>
    </footer>
  </div>
</template>