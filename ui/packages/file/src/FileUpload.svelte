<script lang="ts">
	import { createEventDispatcher, tick } from "svelte";
	import { Upload, ModifyUpload } from "@gradio/upload";
	import type { FileData } from "@gradio/upload";
	import { BlockLabel } from "@gradio/atoms";
	import { File } from "@gradio/icons";

	import {
		display_file_name,
		download_files,
		display_file_size
	} from "./utils";

	export let value: null | FileData | Array;

	export let drop_text: string = "Drop a file";
	export let or_text: string = "or";
	export let upload_text: string = "click to upload";
	export let label: string = "";
	export let show_label: boolean;
	export let file_count: string;

	async function handle_upload({ detail }: CustomEvent<FileData>) {
		value = detail;
		await tick();
		dispatch("change", value);
	}

	function handle_clear({ detail }: CustomEvent<null>) {
		value = null;
		dispatch("change", value);
		dispatch("clear");
	}

	const dispatch = createEventDispatcher<{
		change: FileData | null;
		clear: undefined;
		drag: boolean;
	}>();

	let dragging = false;
	$: dispatch("drag", dragging);
</script>

<BlockLabel {show_label} Icon={File} label={label || "File"} />

{#if value === null && file_count === "single"}
	<Upload on:load={handle_upload} filetype="file" bind:dragging>
		{drop_text}
		<br />- {or_text} -<br />
		{upload_text}
	</Upload>
{:else if value === null}
	<Upload on:load={handle_upload} filetype="file" {file_count} bind:dragging>
		{drop_text}
		<br />- {or_text} -<br />
		{upload_text}
	</Upload>
{:else}
	<div
		class="file-preview overflow-y-scroll w-full max-h-60 flex flex-col justify-between mt-7 mb-7 dark:text-slate-200"
	>
		<ModifyUpload on:clear={handle_clear} absolute />
		{#if Array.isArray(value)}
			{#each value as file}
				<div class="flex flex-row w-full justify-between">
					<div class="file-name w-5/12 p-2">
						{display_file_name(file)}
					</div>
					<div class="file-size w-3/12 p-2">
						{display_file_size(file)}
					</div>
					<div class="file-size w-3/12 p-2 hover:underline">
						<a
							href={download_files(file)}
							download
							class="text-indigo-600 hover:underline dark:text-indigo-300"
							>Download</a
						>
					</div>
				</div>
			{/each}
		{:else}
			<div class="flex flex-row">
				<div class="file-name p-2">
					{display_file_name(value)}
				</div>
				<div class="file-size  p-2">
					{display_file_size(value)}
				</div>
				<div class="file-size p-2 hover:underline">
					<a
						href={download_files(value)}
						download
						class="text-indigo-600 hover:underline dark:text-indigo-300"
						>Download</a
					>
				</div>
			</div>
		{/if}
	</div>
{/if}
