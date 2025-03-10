<svelte:options accessors={true} />

<script lang="ts">
	import { createEventDispatcher } from "svelte";
	import { _ } from "svelte-i18n";
	import { UploadText } from "@gradio/atoms";

	import type { FileData } from "@gradio/upload";
	import type { LoadingStatus } from "@gradio/statustracker/types";

	import Audio from "./Audio.svelte";
	import { StatusTracker } from "@gradio/statustracker";
	import { Block } from "@gradio/atoms";

	import { normalise_file } from "@gradio/upload";

	const dispatch = createEventDispatcher<{
		change: typeof value;
		stream: typeof value;
		error: string;
	}>();

	export let elem_id = "";
	export let elem_classes: string[] = [];
	export let visible = true;
	export let mode: "static" | "dynamic";
	export let value: null | FileData | string = null;
	export let name: string;
	export let source: "microphone" | "upload";
	export let label: string;
	export let root: string;
	export let show_label: boolean;
	export let pending: boolean;
	export let streaming: boolean;
	export let root_url: null | string;
	export let container = true;
	export let scale: number | null = null;
	export let min_width: number | undefined = undefined;
	export let loading_status: LoadingStatus;
	export let autoplay = false;

	let old_value: null | FileData | string = null;

	let _value: null | FileData;
	$: _value = normalise_file(value, root, root_url);

	$: {
		if (JSON.stringify(value) !== JSON.stringify(old_value)) {
			old_value = value;
			dispatch("change");
		}
	}

	let dragging: boolean;
</script>

<Block
	variant={mode === "dynamic" && value === null && source === "upload"
		? "dashed"
		: "solid"}
	border_mode={dragging ? "focus" : "base"}
	padding={false}
	{elem_id}
	{elem_classes}
	{visible}
	{container}
	{scale}
	{min_width}
>
	<StatusTracker {...loading_status} />
	<Audio
		{label}
		{show_label}
		value={_value}
		on:change={({ detail }) => (value = detail)}
		on:stream={({ detail }) => {
			value = detail;
			dispatch("stream", value);
		}}
		on:drag={({ detail }) => (dragging = detail)}
		{name}
		{source}
		{pending}
		{streaming}
		{autoplay}
		on:edit
		on:play
		on:pause
		on:stop
		on:end
		on:start_recording
		on:stop_recording
		on:upload
		on:error={({ detail }) => {
			loading_status = loading_status || {};
			loading_status.status = "error";
			dispatch("error", detail);
		}}
	>
		<UploadText type="audio" />
	</Audio>
</Block>
