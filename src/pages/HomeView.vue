<script setup lang="ts">
// Owner: Arnav Badola
import { ref, computed, useTemplateRef, shallowRef, provide, nextTick, triggerRef } from 'vue';
// import gameListData from '../assets/gamelist.json';
import { onClickOutside, refDebounced, tryOnMounted } from '@vueuse/core';
import { invoke } from '@tauri-apps/api/core';
import { randomString } from '@/utils/random-string';
import { GameActionsProvider, GameExecutable, type Game } from '@/types/types';
import IconVerified from '@/components/IconVerified.vue';
import { isEmpty } from 'lodash-es';
import GameExecutables from '@/components/GameExecutables.vue';
import { GameActionsKey } from '@/constants/constants';
import { path } from '@tauri-apps/api';
import { emit } from '@tauri-apps/api/event';
import { useFetchGameList } from '@/composables/fetch-gamelist';
import { useGlobalState } from '@/composables/app-state';
import TimedNotification from '@/components/TimedNotification.vue';


type DialogKey = 
    'none' | 
    'rpc_message_1'|
    'no_game_selected';;

// Game list from JSON file
// const gameDB = ref<Game[]>([]);

const {
    gameDB,
    isLoadingBundled,
    isLoadingDiscord,
    isLoadingGH,
    fetchGameList,
    isReadyGH,
    isReadyBundled,
    isReadyDiscord,
    allFetchDone,
} = useFetchGameList()
const { addLog } = useGlobalState();
const shouldShowNotificationContainer = computed(() => {
    return isLoadingGH.value || isLoadingDiscord.value || isLoadingBundled.value ||
           (isReadyGH.value || isReadyDiscord.value || isReadyBundled.value);
});

const dialogRef = useTemplateRef<HTMLDialogElement>('dialogRef');
const searchResultContainerRef = useTemplateRef<HTMLElement>('searchResultContainerRef')
const dialogMessage = ref('');
const isDialogOpen = ref(false);
const dialogKey = ref<DialogKey>('none')
const isConnectedToRPC = ref(false);
const isConnecting = ref(false);

// Search functionality
const searchQuery = shallowRef('');
const debouncedSearchQuery = refDebounced(searchQuery, 300)

const searchResultsIsOpen = ref(false);
const isOnSearchResults = ref(false);

// Game status
const currentlyPlaying = ref<string | null>(null);


onClickOutside(searchResultContainerRef, () => {
    searchResultsIsOpen.value = false;
})

// const searchResults = computed(() => {
//     if (!debouncedSearchQuery.value) return [];
//     const query = debouncedSearchQuery.value.toLowerCase();
//     return gameDB.value.filter(game =>
//         game.name.toLowerCase().includes(query) ||
//         game.aliases?.some(alias => alias.toLowerCase().includes(query))
//     );
// });

const searchResults = computed(() => {
    const query = debouncedSearchQuery.value?.toString().trim().toLowerCase();
    if (!query) return [];
    return gameDB.value.filter(game => {
        const nameMatch = game.name?.toLowerCase().includes(query);
        const aliasMatch = Array.isArray(game.aliases)
            ? game.aliases.some(alias => alias?.toLowerCase().includes(query))
            : false;
        const exeMatch = Array.isArray(game.executables)
            ? game.executables.some(exe => exe.name?.toLowerCase().includes(query))
            : false;
        return nameMatch || aliasMatch || exeMatch;
    });
});

// Selected games list
const gameList = ref<Game[]>([]);
// const selectedGame = ref<Game | null>(null);
const selectedGameId = ref<string | null | undefined>(null);

const selectedGame = computed(() => {
    if (!selectedGameId.value) return null;
    const found = gameList.value.find(g => g.uid === selectedGameId.value);
    console.log('selectedGame computed - selectedGameId:', selectedGameId.value, 'found:', found);
    return found || null;
});

function closeSearchResults() {
    searchResultsIsOpen.value = false;
}
function openSearchResults() {
    searchResultsIsOpen.value = true;
}

// Function to add a game to the selected list
function addGameToList(game: Game) {
    if (!gameList.value.some(g => g.id === game.id)) {
        gameList.value.push({
            uid: randomString(),
            ...game
        });
    }

    closeSearchResults();
}

const forceRerenderKey = ref(0); 
// Function to remove a game from the selected list
function removeGameFromList(game: Game) {
    const gameId = game.uid;
    gameList.value = gameList.value.filter(game => game.uid !== gameId);
    if (selectedGame.value?.uid === gameId) { 
        // selectedGame.value = null;
        selectedGameId.value = null;
        forceRerenderKey.value++; 
    }
}

function selectGame(game: Game) {
    // selectedGame.value = game;
    selectedGameId.value = game?.uid;
    searchResultsIsOpen.value = false;
}

function canCreateDummyGame(game: Game | null) {
    if (!game) {
        return false;
    }
    // we can only create a dummy game if the game is not installed or game is not running
    return !game.is_installed
}

function canPlayGame(game: Game | null) {
    if (!game) {
        return false;
    }
    // we can only play a game if the game is installed and not running
    return (game.is_installed && !game.is_running) ?? false;
}

function isExecutableRunning(executable: GameExecutable) {
    // Check if the executable is running
    return executable.is_running ?? false;
}
function isGameExecutableInstalled(executable: GameExecutable) {
    // Check if the executable is installed
    return executable.is_installed ?? false;
}

function isGameInstalled(game: Game | null) {
    if (!game) {
        return false;
    }
    // we can only play a game if the game is installed and not running
    return game.is_installed ?? false;
}


// Create a dummy game
async function createDummyGame(game: Game | null, executable: GameExecutable) {
    if (!game) {
        return;
    }
    const gameUid = game.uid;
    const gameToInstall = gameList.value.find(g => g.uid === gameUid);
    const executableItem = gameToInstall?.executables.find(exe => exe.name === executable.name);
    if (gameToInstall && executableItem) {
        const payload =  { 
            path: executable.path,
            executable_name: executable.filename,
            path_len: executable.segments,
            app_id: Number(gameToInstall.id),
        }
        console.log(payload);
        const result = await invoke('create_fake_game', payload)
        console.log('Game created:', result);
        gameToInstall.is_installed = true;
        executableItem.is_installed = true;
        return true;
    }
}


async function installAndPlay({game, executable}: {game: Game, executable: GameExecutable}) {
    if (!game) {
        return;
    }
    const gameCreated = await createDummyGame(game, executable);
    if (gameCreated) {
        playGame({game, executable});
    } else {
        console.error('Failed to create game');
        addLog('error', 'Failed to create game');
    }
}
// Play game function
async function playGame({game, executable}: {game: Game, executable: GameExecutable}) {
    if (!game) {
        return;
    }
    const gameUid = game.uid;
    try {
        console.log(`Playing game: ${gameUid}`);
        addLog('info', `Playing game: ${game.name}`);
        addLog('info', `Executable: ${executable.name}`);
        currentlyPlaying.value = game.id;
        // find the game in the list
        const gameToPlay = gameList.value.find(g => g.uid === gameUid);
        const executableItem = gameToPlay?.executables.find(exe => exe.name === executable.name);
        if (gameToPlay && executableItem) {
            const payload =  { 
                name: game.name,
                path: executable.path,
                executable_name: executable.filename,
                path_len: executable.segments,
                app_id: Number(gameToPlay.id),
                exec_path: path.join(executable.path!, executable.filename!),
            } 
            await invoke('run_background_process', payload);
            gameToPlay.is_running = true;
            executableItem.is_running = true; 
        }
        // In a real app, this would invoke a Tauri command to launch the game
       
    } catch (error) {
        console.error('Failed to launch game:', error);
    }
}

// Stop playing
async function stopPlaying({game, executable}: {game: Game, executable: GameExecutable}) {
    if (!game) {
        return;
    }
    console.log('Stopped playing game');
    const gameUid = game.uid;
    
    currentlyPlaying.value = null;

    const gameToPlay = gameList.value.find(g => g.uid === gameUid);
    const executableItem = gameToPlay?.executables.find(exe => exe.name === executable.name);
    if (gameToPlay && executableItem) {
        try {
            await invoke('stop_process', {
                exec_name: executable.filename!
            })
            addLog('info', `Stopped game process: ${game.name}`);
            addLog('info', `Stopped Executable: ${executable.name}`);
        } catch (error) {
            console.error('Failed to stop game process:', error);
            const errorMessage = (error instanceof Error) ? error.message : String(error);
            addLog('error', 'Failed to stop game process' + errorMessage);
            // Even if stopping fails, we still update the state
            gameToPlay.is_running = false;
            executableItem.is_running = false;
        } finally {
            gameToPlay.is_running = false;
            executableItem.is_running = false;
        }
    }
}

function getExecutables(game: Game) {
    return game.executables.map(exe => exe.name)
}

async function handleTestRPC(game: Game | null) {
    let state = isConnectedToRPC.value ? 'disconnect' : 'connect';

    console.log('Testing RPC for game:', game);
    if (!game && state === 'connect') {
        showDialog('no_game_selected');
        return;
    }
    if (state === 'disconnect' || isConnecting.value) {
        // await invoke('connect_to_discord_rpc_2', { app_id: "0", discord_state: "disconnect" })
        // invoke('connect_to_discord_rpc_3', {
        //     activity_json: JSON.stringify({
        //         app_id: selectedGame.value?.id
        //     }),
        //     action: 'disconnect',
        // })
        emit('event_disconnect');
        
        isConnectedToRPC.value = false;
        game!.is_running = false;
        currentlyPlaying.value = null;
        isConnecting.value = false;
        return;
    }
    showDialog('rpc_message_1');
}

async function continueRPCRisk(game: Game | null) {
    if (!game) {
        return;
    }
    const gameUid = game.uid;
    const gameToTest = gameList.value.find(g => g.uid === gameUid);
    if (gameToTest) {
        console.log('Testing RPC for game:', gameToTest);
        isConnecting.value = true;
        // invoke('connect_to_discord_rpc_2', { app_id: gameToTest.id, discord_state: "connect" })
        invoke('connect_to_discord_rpc_3', {
            activity_json: JSON.stringify({
                app_id: gameToTest.id,
            }),
            action: 'connect',
        })
        .then(() => {
            isConnectedToRPC.value = true;
            gameToTest.is_running = true;
            currentlyPlaying.value = gameToTest.id;
            isConnecting.value = false;
        })

        hideDialog();
    }
}

function handleSearchBlur() {
    setTimeout(() => {
        if (!isOnSearchResults.value) {
            searchResultsIsOpen.value = false;
        }
    }, 200);
}

function showDialog(message: DialogKey) {
    isDialogOpen.value = true;
    dialogMessage.value = message;
    dialogKey.value = message;
    if(!isEmpty(message)) {
        dialogRef.value?.showModal();
    }
}

function hideDialog() {
    dialogRef.value?.close(); 
    dialogMessage.value = '';
    isDialogOpen.value = false;
}


provide<GameActionsProvider>(GameActionsKey, {
    canPlayGame,
    isGameInstalled,
    isExecutableRunning,
    isGameExecutableInstalled,
});
</script>

<template>
    <div class="min-h-screen overflow-hidden bg-slate-950 bg-[radial-gradient(circle_at_top,_rgba(99,102,241,0.18),transparent_30%),radial-gradient(circle_at_right,_rgba(56,189,248,0.12),transparent_20%)] text-slate-100 px-4 py-8">
        <!-- Center dialog -->
        <dialog id="dialog" class="dialogStyle inset-0 bg-slate-900/80 bg-opacity-80
        border border-slate-700 rounded-3xl shadow-2xl shadow-slate-950/40
        transition-opacity duration-300 ease-in-out z-50
        "
        style="left: 50%; top: 50%; transform: translate(-50%, -50%)"
        ref="dialogRef">
            <div class="flex flex-col items-center justify-center p-6" >
                <div class="mb-4 text-gray-500 dark:text-gray-400">
                    <div v-if="dialogKey === 'rpc_message_1'">
                        <p>
                        This is only a feature in development.  
                        </p>
                        <p class="my-2">
                            It works but due to the nature that it tricks Discord into thinking you are playing a game
                            by sending an RPC using actual game ID rather than letting Discord detect you have a game/application running. 
                        </p>
                        <p>
                        This may flag your account as suspicious for self-botting.
                        </p>
                    </div>

                    <div v-if="dialogKey === 'no_game_selected'">
                        <p>
                            No game selected. Please select a game from the list on the left.
                        </p>
                    </div>
                </div>
                <div class="gap-2 flex">
                    <button
                    
                    class="
                text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300 
                border border-gray-300 dark:border-gray-600 rounded-lg px-4 py-1"
                @click="hideDialog()">
                    <span  v-if="dialogKey == 'rpc_message_1'">
                        Cancel 
                    </span>
                    <span v-else>OK</span>
                </button>
                
                <button 
                v-if="dialogKey === 'rpc_message_1'"
                class="text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300 
                border border-gray-300 dark:border-gray-600 rounded-lg px-4 py-1"
                @click="continueRPCRisk(selectedGame)">
                    Accept risk and continue
                </button>
                </div>
            </div>
        </dialog>
        <div class="relative mx-auto mb-8 max-w-6xl overflow-hidden rounded-[2.5rem] border border-white/10 bg-slate-950/90 p-8 shadow-[0_40px_120px_-50px_rgba(56,189,248,0.75)] backdrop-blur-xl">
            <div class="pointer-events-none absolute -left-16 top-10 h-40 w-40 rounded-full bg-cyan-500/20 blur-3xl"></div>
            <div class="pointer-events-none absolute -right-12 bottom-10 h-52 w-52 rounded-full bg-violet-500/20 blur-3xl"></div>
            <div class="grid gap-6 lg:grid-cols-[1.45fr_0.8fr] items-center">
                <div class="space-y-5">
                    <p class="text-sm uppercase tracking-[0.45em] text-cyan-300/80">Arcade Hub</p>
                    <h1 class="text-5xl font-black tracking-tight text-white sm:text-6xl">
                        Discover verified games in a slick neon launcher.
                    </h1>
                    <p class="max-w-xl text-slate-300 leading-8">
                        Search instantly, queue favorites, and control game sessions from a premium glassmorphism interface made for desktop.
                    </p>
                    <div class="flex flex-wrap gap-3">
                        <span class="rounded-full border border-cyan-400/25 bg-cyan-500/10 px-4 py-2 text-xs uppercase tracking-[0.25em] text-cyan-100">RPC ready</span>
                        <span class="rounded-full border border-violet-400/25 bg-violet-500/10 px-4 py-2 text-xs uppercase tracking-[0.25em] text-violet-100">Smart search</span>
                        <span class="rounded-full border border-white/10 bg-white/5 px-4 py-2 text-xs uppercase tracking-[0.25em] text-slate-200">Modern UI</span>
                    </div>
                </div>
                <div class="rounded-[2rem] border border-violet-500/10 bg-slate-950/80 p-6 shadow-[inset_0_0_0_1px_rgba(148,163,184,0.12)]">
                    <p class="text-sm uppercase tracking-[0.3em] text-cyan-200/80">Quick status</p>
                    <div class="mt-6 grid gap-4">
                        <div class="rounded-3xl bg-slate-950/70 p-5 ring-1 ring-white/10">
                            <p class="text-sm text-slate-400">Selected games</p>
                            <p class="mt-3 text-3xl font-semibold text-white">{{ gameList.length }}</p>
                        </div>
                        <div class="rounded-3xl bg-slate-950/70 p-5 ring-1 ring-white/10">
                            <p class="text-sm text-slate-400">Active session</p>
                            <p class="mt-3 text-3xl font-semibold text-white">{{ currentlyPlaying ? 'Yes' : 'No' }}</p>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- refetch game list fetch status. will appear on top left -->
        <Transition 
            enter-active-class="transition-opacity duration-300 delay-100 ease-in-out"
            leave-active-class="transition-opacity duration-600 delay-100 ease-in-out"  
            enter-from-class="opacity-0 translate-y-2 ease-in-out"
            enter-to-class="opacity-100 translate-y-0 ease-in-out"
        >
            <div class="absolute top-20 left-4 z-20 " v-if="shouldShowNotificationContainer && !allFetchDone">
                <!-- Fetching from mirror loading indicator --> 
                <Transition 
                    enter-active-class="transition-opacity duration-300 delay-100 ease-in-out"
                    leave-active-class="transition-opacity duration-600 delay-100 ease-in-out"  
                    enter-from-class="opacity-0 translate-y-2 ease-in-out"
                    enter-to-class="opacity-100 translate-y-0 ease-in-out"
                >
                    <div v-if="isLoadingGH" class="text-sm text-gray-500 dark:text-gray-400">
                        Fetching game list from GitHub mirror... 
                      <div class="border-full h-2 w-2 bg-green-500 rounded-full inline-block ml-2 animate-pulse"></div>
                    </div>
                </Transition>
                <TimedNotification
                    :is-ready="isReadyGH" 
                    :duration="1500"
                    container-class="text-sm text-gray-500 dark:text-gray-400"
                > 
                    Game list from mirror fetched <span class="text-green-400">✓</span>
                </TimedNotification>

                <!-- Fetching from Discord loading indicator -->
                <Transition 
                    enter-active-class="transition-opacity duration-300 delay-100 ease-in-out"
                    leave-active-class="transition-opacity duration-600 delay-100 ease-in-out"  
                    enter-from-class="opacity-0 translate-y-2 ease-in-out"
                    enter-to-class="opacity-100 translate-y-0 ease-in-out"
                >
                    <div v-if="isLoadingDiscord" class="text-sm text-gray-500 dark:text-gray-400">
                        Fetching game list directly from Discord...
                        <div class="border-full h-2 w-2 bg-green-500 rounded-full inline-block ml-2 animate-pulse"></div>
                    </div>
                </Transition>
                <TimedNotification
                    :is-ready="isReadyDiscord" 
                    :duration="1500"
                    container-class="text-sm text-gray-500 dark:text-gray-400"
                > 
                    Game list from Discord fetched <span class="text-green-400">✓</span>
                </TimedNotification>

                
                <!-- Fetching from bundled loading indicator -->
                <Transition 
                    enter-active-class="transition-opacity duration-300 delay-100 ease-in-out"
                    leave-active-class="transition-opacity duration-600 delay-100 ease-in-out"  
                    enter-from-class="opacity-0 translate-y-2 ease-in-out"
                    enter-to-class="opacity-100 translate-y-0 ease-in-out"
                >
                    <div v-if="isLoadingBundled" class="text-sm text-gray-500 dark:text-gray-400">
                        Fetching game list from bundled game list...
                        <div class="border-full h-2 w-2 bg-green-500 rounded-full inline-block ml-2 animate-pulse"></div>
                    </div>
                </Transition>
                <TimedNotification
                    :is-ready="isReadyBundled" 
                    :duration="1500"
                    container-class="text-sm text-gray-500 dark:text-gray-400"
                > 
                    Game list from bundle pre-loaded <span class="text-green-400">✓</span>
                </TimedNotification>

            </div>
        </Transition>

        <!-- Search Bar -->
        <div class="mb-10">
            <div class="relative mx-auto max-w-4xl" ref="searchResultContainerRef">
                <div class="group relative overflow-hidden rounded-[2rem] border border-cyan-500/20 bg-slate-950/85 p-3 shadow-[0_20px_65px_-35px_rgba(56,189,248,0.75)] backdrop-blur-xl">
                    <div class="pointer-events-none absolute inset-x-0 top-0 h-24 bg-gradient-to-b from-cyan-500/10 to-transparent"></div>
                    <input v-model="searchQuery" type="text" placeholder="Search verified games..."
                        class="relative w-full rounded-[1.75rem] border border-slate-700/80 bg-slate-950/95 px-6 py-4 text-sm text-slate-100 placeholder:text-slate-500 outline-none ring-1 ring-slate-700 transition focus:border-cyan-400 focus:ring-2 focus:ring-cyan-400/30"
                        @focus="openSearchResults" @input="openSearchResults" @blur="handleSearchBlur" />

                    <button
                        @click="fetchGameList()"
                        class="absolute right-3 top-1/2 -translate-y-1/2 rounded-full bg-gradient-to-r from-cyan-500 to-indigo-500 px-4 py-2 text-xs font-semibold uppercase tracking-[0.18em] text-slate-950 shadow-lg shadow-cyan-500/20 transition hover:scale-[1.02] hover:shadow-cyan-500/30">
                        Refresh
                    </button>
                </div>
                <div v-if="searchResultsIsOpen" @click="isOnSearchResults = true"
                    class="absolute z-50 mt-3 w-full rounded-[1.75rem] border border-white/10 bg-slate-950/95 shadow-[0_30px_80px_-40px_rgba(15,23,42,0.9)] backdrop-blur-2xl">
                    <div v-if="searchResults.length > 0" class="divide-y divide-slate-800">
                        <div v-for="game in searchResults" :key="game.id"
                            class="p-4 hover:bg-slate-900/85 transition-colors duration-200">
                            <div class="flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
                                <div class="space-y-1">
                                    <p class="font-semibold text-white">{{ game.name }}</p>
                                    <p class="text-xs text-slate-500">ID: {{ game.id }}</p>
                                    <div class="flex flex-wrap gap-2 text-xs text-slate-400">
                                        <span class="rounded-full bg-white/5 px-2 py-1">{{ game.executables.length }} executables</span>
                                        <span class="rounded-full bg-white/5 px-2 py-1">{{ game.aliases?.length || 0 }} aliases</span>
                                    </div>
                                </div>
                                <button @click="addGameToList(game)"
                                    class="ml-0 mt-3 inline-flex rounded-full bg-gradient-to-r from-indigo-500 to-cyan-500 px-4 py-2 text-xs font-semibold text-white transition hover:scale-[1.02] sm:mt-0">
                                    Add game
                                </button>
                            </div>
                        </div>
                    </div>
                    <div v-if="searchResults.length === 0"
                        class="p-4 text-sm text-slate-400">
                        No games matched your search. Try a different title or keyword.
                    </div>
                </div>
            </div>
        </div>

        <!-- Two-Column Layout with right fixed column -->
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 relative">
            <!-- Left Column: Selected Games (scrollable) -->
            <!--  max-h-[70vh] overflow-y-auto : add these somewhere to just scroll the content  -->
            <div class="bg-white dark:bg-gray-800 p-4 rounded-lg shadow">
                <h2
                    class="text-xl font-bold text-gray-900 dark:text-white mb-4 sticky top-0 bg-white dark:bg-gray-800 py-2 z-10">
                    Games</h2>
                <div v-if="gameList.length === 0" class="text-gray-500 dark:text-gray-400 text-center py-8">
                    No games selected. Search and add games from the search bar.
                </div>
                <div v-else class="space-y-4">
                    <div v-for="game in gameList" :key="game.id" 
                        class="p-3 border border-gray-200 dark:border-gray-700 rounded-lg
                        hover:bg-gray-100 dark:hover:bg-gray-700/50 transition-colors 
                        duration-200 ease-in-out" 
                        :class="[
                            {
                                'ring-1 ring-violet-500/40 shadow-[0px_0px_8px_2px_#8e51ff50] bg-gray-100 dark:bg-gray-700/40': selectedGame?.uid === game.uid,
                            }
                        ]" @click="selectGame(game)"
                    >
                        <div class="flex justify-between items-center">
                            <div class="flex items-center gap-1">
                                <div class="font-medium text-gray-800 dark:text-white">{{ game.name }}</div>
                                <div class="relative inline-flex items-center">
                                    <div class="w-2 h-2 bg-white absolute rounded-full" style="left: 50%; top: 50%; transform: translate(-50%, -50%)"></div>
                                    <div class="relative inline-block">
                                     <IconVerified class="w-5 h-5 text-indigo-500 dark:text-indigo-400"></IconVerified>
                                    </div>
                                </div>
                            </div>
                            <button @click="removeGameFromList(game)" class="text-red-300 hover:text-red-400"
                                v-if="!game.is_running"> 
                                Remove
                            </button>
                        </div>
                        <div class="flex space-x-2 mt-2">
                            <!-- Previously play button was here -->
                            <div class="text-sm text-green-500 dark:text-green-400" v-if="game.is_running">
                                Running
                            </div>
                        </div>
                    </div>
                </div>
            </div>

            <!-- Right Column: Game Actions (fixed position) -->
            <div class="rounded-[2rem] border border-white/10 bg-slate-950/90 p-5 shadow-[0_35px_100px_-45px_rgba(15,23,42,0.85)] md:sticky md:top-4 self-start" :key="forceRerenderKey">
                <h2 class="text-xl font-bold text-white mb-4">Game Actions</h2>
                <div class="space-y-5">
                    <div class="text-slate-400 mb-2 text-sm" v-if="!selectedGame || selectedGame === null">
                        Select a game from the left to reveal actions.
                    </div>
                    
                    <div v-if="selectedGame" class="rounded-[1.75rem] border border-slate-800/90 bg-slate-950/80 p-4 text-slate-300">
                        <p class="text-sm text-slate-400">Selected game</p>
                        <p class="mt-2 text-lg font-semibold text-white">{{ selectedGame.name }}</p>
                        <div class="mt-3 text-sm leading-6 text-slate-400">
                            <p><span class="font-medium text-slate-200">ID:</span> {{ selectedGame.id }}</p>
                            <p v-if="selectedGame.aliases && selectedGame.aliases.length > 0"><span class="font-medium text-slate-200">Aliases:</span> {{ selectedGame.aliases.join(', ') }}</p>
                        </div>
                    </div>
                    <button @click="handleTestRPC(selectedGame)"
                        class="w-full rounded-full bg-gradient-to-r from-cyan-500 to-indigo-500 px-4 py-3 text-sm font-semibold text-slate-950 shadow-lg shadow-cyan-500/20 transition hover:scale-[1.01] hover:shadow-cyan-500/30">
                        {{ isConnecting || isConnectedToRPC ? 'Disconnect Gateway' : 'Test RPC' }}
                    </button>

                    <div class="border-t border-slate-800 my-4"></div>

                    <GameExecutables v-if="selectedGame" :game="selectedGame" 
                        @play="playGame"
                        @stop="stopPlaying"
                        @install_and_play="installAndPlay"
                    />

                    <!-- <button @click="playGame(selectedGame)" :disabled="!canPlayGame(selectedGame)"
                        class="w-full py-2 rounded-lg" :class="[
                            !canPlayGame(selectedGame)
                                ? 'bg-green-400 cursor-not-allowed text-gray-100'
                                : 'bg-green-600 hover:bg-green-600 text-white'
                        ]">
                        {{ currentlyPlaying === selectedGame?.id ? 'Playing...' : 'Play' }}
                    </button>

                    <button @click="stopPlaying(selectedGame)" :disabled="!selectedGame?.is_running" :class="[
                        'w-full py-2 rounded-lg',
                        !selectedGame?.is_running
                            ? 'bg-gray-400 cursor-not-allowed text-gray-200'
                            : 'bg-red-600 hover:bg-red-700 text-white'
                    ]">
                        Stop Playing
                    </button> -->
                </div>

                <!-- Divider -->
                <div class="border-t border-gray-200 dark:border-gray-700 my-5"></div>

                <div class="mt-6 rounded-[1.75rem] border border-slate-700/80 bg-slate-950/70 p-5 backdrop-blur-xl">
                    <h3 class="font-semibold text-white mb-2">Status</h3>
                    <div class="text-sm text-slate-400 mb-3">
                        Check Discord to see if it displays that you are playing a game.
                    </div>
                    <div v-if="currentlyPlaying" class="text-gray-500 dark:text-gray-400">
                        Currently playing: <span class="text-green-600"> {{gameList.find(g => g.id ===
                            currentlyPlaying)?.name }}</span>
                    </div>
                    <div v-else class="text-gray-500 dark:text-gray-400">
                        Not playing any game
                    </div>
                </div>

                <div v-if="selectedGame" class="my-4">
                    <h3 class="font-medium text-white mb-3">Game Details</h3>
                    <div class="rounded-[1.75rem] border border-slate-800/90 bg-slate-950/80 p-4 text-sm text-slate-300">
                        <div class="mb-3">
                            <p class="font-semibold text-slate-100">Aliases</p>
                            <p class="mt-2 text-slate-400">{{ selectedGame.aliases?.join(', ') || 'No aliases available' }}</p>
                        </div>
                        <div>
                            <p class="font-semibold text-slate-100">Executables</p>
                            <ul class="mt-2 space-y-1 text-slate-400">
                                <li v-for="exe in selectedGame.executables" :key="exe.name" class="flex items-center justify-between rounded-2xl bg-white/5 px-3 py-2">
                                    <span>{{ exe.name }}</span>
                                    <span class="text-xs text-slate-400">{{ exe.os }}</span>
                                </li>
                            </ul>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
.dialogStyle::backdrop {
    background-color: rgba(0, 0, 0, 0.7);
    backdrop-filter: blur(4px);
}
</style>