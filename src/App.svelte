<script lang="ts">
    import { onDestroy, onMount, tick } from "svelte";

    type Track = {
        id: string;
        artist: string;
        year: number;
        title: string;
        release: string;
        streams: number;
        audioPath: string;
        coverPath: string;
    };

    const stages = [2, 4, 8, 16];
    const baseUrl = import.meta.env.BASE_URL;
    const assetBaseUrl = "https://music.mudkip.dev/";
    const assetUrls = new Map<string, string>();

    let tracks: Track[] = [];
    let artists: string[] = [];
    let yearsByArtist = new Map<string, number[]>();
    let selectedYears: Record<string, boolean> = {};
    let libraryLoading = true;
    let libraryError = "";
    let gameTracks: Track[] = [];
    let downloading = false;
    let downloaded = 0;
    let downloadTotal = 0;
    let downloadError = "";
    let previousId = "";
    let current: Track | undefined;
    let stage = 0;
    let guess = "";
    let searchOpen = false;
    let status = "";
    let finished = false;
    let playing = false;
    let fullPlayback = false;
    let audio: HTMLAudioElement;
    let stopTimer: ReturnType<typeof setTimeout> | undefined;

    $: selectedCount = tracks.filter((track) => selectedYears[selectionKey(track.artist, track.year)]).length;
    $: suggestions = guess.trim()
        ? tracks.filter(
            (track) =>
                normalize(track.title).includes(normalize(guess)) ||
                normalize(track.release).includes(normalize(guess)),
        )
        : [];

    function selectionKey(artist: string, year: number): string {
        return `${artist}:${year}`;
    }

    async function loadLibrary() {
        try {
            const response = await fetch(`${assetBaseUrl}tracks.json`);
            if (!response.ok) throw new Error(`The music library returned ${response.status}.`);
            const library = await response.json();
            if (!Array.isArray(library)) throw new Error("The music library is invalid.");

            tracks = library as Track[];
            artists = [...new Set(tracks.map((track) => track.artist))].sort();
            yearsByArtist = new Map(
                artists.map((artist) => [
                    artist,
                    [...new Set(tracks.filter((track) => track.artist === artist).map((track) => track.year))].sort(),
                ]),
            );
            selectedYears = Object.fromEntries(
                tracks.map((track) => [selectionKey(track.artist, track.year), true]),
            );
        } catch (error) {
            libraryError = error instanceof Error ? error.message : "The music library could not be loaded.";
        } finally {
            libraryLoading = false;
        }
    }

    function artistSelection(artist: string): { all: boolean; some: boolean } {
        const choices = (yearsByArtist.get(artist) ?? []).map((year) => selectedYears[selectionKey(artist, year)]);
        return { all: choices.every(Boolean), some: choices.some(Boolean) };
    }

    function toggleArtist(artist: string, checked: boolean) {
        for (const year of yearsByArtist.get(artist) ?? []) selectedYears[selectionKey(artist, year)] = checked;
        selectedYears = { ...selectedYears };
    }

    function toggleYear(artist: string, year: number, checked: boolean) {
        selectedYears = { ...selectedYears, [selectionKey(artist, year)]: checked };
    }

    async function startGame() {
        if (selectedCount === 0 || downloading) return;

        gameTracks = tracks.filter((track) => selectedYears[selectionKey(track.artist, track.year)]);
        const assetPaths = [
            ...gameTracks.map((track) => track.audioPath),
            ...new Set(tracks.map((track) => track.coverPath)),
        ];
        const queue = [...assetPaths];
        const createdUrls: string[] = [];
        let downloadFailure: Error | undefined;
        downloading = true;
        downloaded = 0;
        downloadTotal = assetPaths.length;
        downloadError = "";

        try {
            const workers = Array.from({ length: Math.min(6, queue.length) }, async () => {
                while (queue.length > 0 && !downloadFailure) {
                    const assetPath = queue.shift();
                    if (!assetPath) return;
                    try {
                        const response = await fetch(`${assetBaseUrl}${assetPath.replace(/^\//, "")}`);
                        if (!response.ok) throw new Error(`Could not download ${assetPath.split("/").at(-1)}.`);
                        const objectUrl = URL.createObjectURL(await response.blob());
                        assetUrls.set(assetPath, objectUrl);
                        createdUrls.push(objectUrl);
                        downloaded += 1;
                    } catch (error) {
                        downloadFailure = error instanceof Error ? error : new Error("The music download failed.");
                    }
                }
            });
            await Promise.all(workers);
            if (downloadFailure) throw downloadFailure;
            current = chooseTrack();
            await tick();
            await playClip();
        } catch (error) {
            for (const objectUrl of createdUrls) URL.revokeObjectURL(objectUrl);
            assetUrls.clear();
            downloadError = error instanceof Error ? error.message : "The music download failed.";
        } finally {
            downloading = false;
        }
    }

    function chooseTrack(): Track {
        const pool = gameTracks.filter((track) => track.id !== previousId);
        return pool[Math.floor(Math.random() * pool.length)];
    }

    function normalize(value: string): string {
        return value
            .toLocaleLowerCase()
            .normalize("NFKD")
            .replace(/[\u0300-\u036f]/g, "")
            .replace(/[^a-z0-9]/g, "");
    }

    async function playClip() {
        stopClip();
        fullPlayback = false;
        audio.currentTime = 0;

        try {
            await audio.play();
        } catch {
            status = "Press Play to hear the song.";
        }
    }

    function startClipTimer() {
        clearTimeout(stopTimer);
        playing = true;
        if (!fullPlayback) stopTimer = setTimeout(stopClip, stages[stage] * 1000);
    }

    async function playFullSong() {
        stopClip();
        fullPlayback = true;
        audio.currentTime = 0;

        try {
            await audio.play();
        } catch {
            status += " Press Play to hear the song.";
        }
    }

    function stopClip() {
        clearTimeout(stopTimer);
        if (audio) audio.pause();
        playing = false;
    }

    function submitGuess() {
        if (!current || !guess.trim() || finished) return;
        searchOpen = false;

        if (normalize(guess) === normalize(current.title)) {
            stopClip();
            status = "Correct.";
            finished = true;
            void playFullSong();
            return;
        }

        if (stage < stages.length - 1) {
            stopClip();
            stage += 1;
            guess = "";
            status = "Incorrect.";
            void playClip();
            return;
        }

        stopClip();
        status = "Incorrect.";
        finished = true;
        void playFullSong();
    }

    function skip() {
        if (finished) return;
        stopClip();

        if (stage < stages.length - 1) {
            stage += 1;
            status = "";
            void playClip();
        } else {
            status = "";
            finished = true;
            void playFullSong();
        }
    }

    async function nextSong() {
        stopClip();
        previousId = current?.id ?? "";
        current = chooseTrack();
        stage = 0;
        guess = "";
        searchOpen = false;
        status = "";
        finished = false;
        await tick();
        audio.load();
        await playClip();
    }

    function chooseSuggestion(track: Track) {
        guess = track.title;
        status = "";
        searchOpen = false;
    }

    onMount(() => {
        void loadLibrary();
    });

    onDestroy(() => {
        stopClip();
        for (const objectUrl of assetUrls.values()) URL.revokeObjectURL(objectUrl);
    });
</script>

<div class="game-panel box-border h-144 min-w-0 overflow-x-hidden overflow-y-auto px-6 py-3 font-sans">
        <h1 class="text-2xl"><a class="text-inherit no-underline hover:underline" href={baseUrl}>Song Guesser</a></h1>

        {#if libraryLoading}
            <p>Loading music library...</p>
        {:else if libraryError}
            <p role="alert">{libraryError}</p>
        {:else if !current}
            <p>Select which artists you want. Your browser will download audio and cover art for each song.</p>

            {#each artists as artist}
                <section>
                    <label class="flex items-center gap-1 font-bold">
                        <input
                            type="checkbox"
                            checked={artistSelection(artist).all}
                            indeterminate={artistSelection(artist).some && !artistSelection(artist).all}
                            on:change={(event) => toggleArtist(artist, event.currentTarget.checked)}
                        />
                        {artist}
                    </label>

                    <div class="mt-1 grid grid-cols-[repeat(4,max-content)] gap-x-2 gap-y-1 pl-6">
                        {#each yearsByArtist.get(artist) ?? [] as year}
                            <label class="flex items-center gap-1">
                                <input
                                    type="checkbox"
                                    checked={selectedYears[selectionKey(artist, year)]}
                                    on:change={(event) => toggleYear(artist, year, event.currentTarget.checked)}
                                />
                                {year}
                            </label>
                        {/each}
                    </div>
                </section>
            {/each}

            <button class="mt-6 min-h-10 px-5" type="button" disabled={selectedCount === 0 || downloading} on:click={startGame}>
                {downloading ? `Downloading (${downloaded}/${downloadTotal} songs)` : `Start (${selectedCount} songs)`}
            </button>
            {#if downloading}
                <progress class="mt-3 block w-full" value={downloaded} max={downloadTotal}>{downloaded} of {downloadTotal}</progress>
            {/if}
            {#if downloadError}<p class="mt-3" role="alert">{downloadError}</p>{/if}
        {:else}
            <audio
                bind:this={audio}
                src={assetUrls.get(current.audioPath)}
                preload="auto"
                on:play={startClipTimer}
                on:ended={stopClip}
            ></audio>

            {#if !finished}
                <div class="mt-4 text-center">
                    <button class="box-border min-h-10 w-36 px-3" type="button" on:click={playing ? stopClip : playClip}>
                        {playing ? "Stop" : "Play"}
                    </button>
                </div>

                <form class="mt-4" on:submit|preventDefault={submitGuess}>
                    <div class="relative mx-auto mt-2 w-full max-w-sm">
                        <input
                            class="box-border min-h-10 w-full px-2"
                            id="guess"
                            bind:value={guess}
                            autocomplete="off"
                            role="combobox"
                            aria-autocomplete="list"
                            aria-expanded={searchOpen && suggestions.length > 0}
                            aria-controls="song-suggestions"
                            on:focus={() => (searchOpen = true)}
                            on:input={() => {
                                status = "";
                                searchOpen = true;
                            }}
                            on:blur={() => (searchOpen = false)}
                        />

                        {#if searchOpen && suggestions.length > 0}
                            <ul id="song-suggestions" class="suggestions absolute left-0 top-full z-10 m-0 max-h-72 w-full list-none overflow-y-auto p-0" role="listbox">
                                {#each suggestions as track}
                                    <li role="option" aria-selected={normalize(guess) === normalize(track.title)}>
                                        <button
                                            class="suggestion box-border flex min-h-16 w-full items-center gap-3 p-2 text-left"
                                            type="button"
                                            on:mousedown|preventDefault={() => chooseSuggestion(track)}
                                        >
                                            <img class="size-12 shrink-0 object-cover" src={assetUrls.get(track.coverPath)} alt="" />
                                            <span class="min-w-0">
                                                <strong class="block truncate">{track.title}</strong>
                                                <small class="block truncate">{track.release}</small>
                                            </span>
                                        </button>
                                    </li>
                                {/each}
                            </ul>
                        {/if}
                    </div>
                    <div class="mt-4 flex justify-center gap-2">
                        <button class="box-border min-h-10 w-28 px-3" type="submit" disabled={!guess.trim()}>Submit</button>
                        <button class="box-border min-h-10 w-28 px-3" type="button" on:click={skip}>{stage < stages.length - 1 ? "Skip" : "Reveal"}</button>
                    </div>
                </form>

                <output class="mt-4 block min-h-12" aria-live="polite">{status}</output>
                <progress class="mt-4 block w-full" value={stage + 1} max={stages.length}>{stage + 1} of {stages.length}</progress>
                <p class="mt-2 text-center">{stages[stage]} seconds</p>
            {:else}
                <div class="mt-6 text-center">
                    <img class="mx-auto block size-64 object-cover" src={assetUrls.get(current.coverPath)} alt={`Cover art for ${current.release}`} />
                    <h2 class="mb-0">{current.title}</h2>
                    <p class="mt-1">{current.release}</p>
                    <output class="mt-4 block" aria-live="polite">{status}</output>
                    <button class="mt-4 box-border min-h-10 w-28 px-3" type="button" on:click={nextSong}>Next song</button>
                </div>
            {/if}
        {/if}
</div>
