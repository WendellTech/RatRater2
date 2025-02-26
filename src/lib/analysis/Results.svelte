<script lang="ts">
  import { getContext } from "svelte";
  import type { Writable } from "svelte/store";
  import { file, view, type Loaded } from "$lib/state";
  import { createAnalysis, type Analysis, type Progress } from "./createAnalysis";
  import ObfuscationTable from "./ObfuscationTable.svelte";
  import FlagCard from "./FlagCard.svelte";

  type OfficialHash = { file: string; hash: string; source: string; time: number };
  const officialHashes: Writable<OfficialHash[]> = getContext("hashes");
  let officialFile: OfficialHash | undefined,
    officialName: OfficialHash | undefined,
    impersonating: string | undefined;
  $: {
    officialFile = $officialHashes.find((h) => h.hash == ($file as Loaded).hash);
    officialName = $officialHashes.find((h) => h.file == ($file as Loaded).file.name);
    impersonating =
      officialName && !(officialFile && officialFile.hash == officialName.hash)
        ? officialName.file
        : undefined;
  }

  let analysis: Writable<Analysis>;
  let progress: Writable<Progress>;
  $: if ("zip" in $file) {
    ({ analysis, progress } = createAnalysis());
  }

  $: obfuscation = Object.entries($analysis.obfuscation);
</script>

<div class="flex gap-2 max-lg:flex-col">
  <div class="info-layout rounded-2xl bg-background">
    <p class="m3-font-headline-small">
      {#if !$progress}
        Starting scan...
      {:else if $progress.total == 0}
        No checkable files
      {:else}
        {Math.floor(($progress.done / $progress.total) * 100)}% done
      {/if}
    </p>
  </div>
  {#if (obfuscation && obfuscation.length) || impersonating}
    <div class="info-layout rounded-2xl bg-background">
      <p class="m3-font-headline-small">Possible obfuscation</p>
      <ObfuscationTable {obfuscation} on:open />
      {#if impersonating}
        <div class="mt-2 w-full rounded-lg bg-tertiary-container p-2 text-on-tertiary-container">
          This file may be impersonating the official version of {impersonating}
        </div>
      {/if}
    </div>
  {/if}
  {#if $analysis.flagged || !$officialHashes || officialFile}
    <div
      class="info-layout rounded-2xl border-tertiary bg-background"
      class:border-4={$analysis.flagged}
    >
      {#if $analysis.flagged}
        {@const flag = $analysis.flagged}
        <p class="m3-font-headline-small text-center">Almost definitely a rat</p>
        <p class="text-center">Classification: {flag.name}</p>
        {#if flag.file}
          <button
            class="underline-hover truncate text-primary underline"
            on:click={() => ($view = { tab: "browser", editorFile: flag.file })}
          >
            File: <span class="font-mono">{flag.file}</span>
          </button>
        {:else}
          <p class="text-center">To prevent reverse engineering, you cannot see the cause</p>
        {/if}
      {:else if !$officialHashes}
        <p class="m3-font-headline-small text-center">Failed to load official hashes</p>
      {:else if officialFile}
        <p class="m3-font-headline-small text-center">Found in official sources</p>
        <table class="w-full">
          <tr>
            <th class="border-r border-outline-variant pr-2">File</th>
            <td class="pl-2">{officialFile.file}</td>
          </tr>
          <tr>
            <th class="border-r border-outline-variant pr-2">Source</th>
            <td class="pl-2">{officialFile.source}</td>
          </tr>
          <tr>
            <th class="border-r border-outline-variant pr-2">Added</th>
            <td class="pl-2">{new Date(officialFile.time).toLocaleString()}</td>
          </tr>
        </table>
      {/if}
    </div>
  {/if}
</div>
<div class="grid gap-4 md:grid-cols-2 xl:grid-cols-4 2xl:grid-cols-6">
  {#each Object.entries($analysis.flags) as [name, flag]}
    <FlagCard {name} {flag} />
  {/each}
</div>

<style lang="postcss">
  .info-layout {
    @apply flex flex-col items-center justify-center overflow-hidden p-4 lg:flex-1;
  }
</style>
