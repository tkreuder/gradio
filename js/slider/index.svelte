<script lang="ts">
	import Static from "./static";
	import Interactive from "./interactive";

	import type { LoadingStatus } from "@gradio/statustracker/types";

	export let elem_id = "";
	export let elem_classes: string[] = [];
	export let visible = true;
	export let value = 0;
	export let label = "Slider";
	export let info: string | undefined = undefined;
	export let container = true;
	export let scale: number | null = null;
	export let min_width: number | undefined = undefined;
	export let minimum: number;
	export let maximum: number;
	export let step: number;
	export let mode: "static" | "dynamic";
	export let show_label: boolean;

	export let loading_status: LoadingStatus;
	export let value_is_output = false;
</script>

{#if mode === "static"}
	<Static
		bind:value
		bind:value_is_output
		{label}
		{info}
		{show_label}
		{minimum}
		{maximum}
		{step}
		{visible}
		{elem_id}
		{elem_classes}
		{container}
		{scale}
		{min_width}
		{loading_status}
		on:input
		on:change
		on:release
	></Static>
{:else}
	<Interactive
		bind:value
		bind:value_is_output
		{label}
		{info}
		{show_label}
		{minimum}
		{maximum}
		{step}
		{visible}
		{elem_id}
		{elem_classes}
		{container}
		{scale}
		{min_width}
		{loading_status}
		on:input
		on:change
		on:release
	></Interactive>
{/if}
