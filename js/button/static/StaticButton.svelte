<script lang="ts">
	import Button from "./Button.svelte";
	import { _ } from "svelte-i18n";

	export let elem_id = "";
	export let elem_classes: string[] = [];
	export let visible = true;
	export let value: string;
	export let variant: "primary" | "secondary" | "stop" = "secondary";
	export let mode: "static" | "dynamic" = "dynamic";
	export let size: "sm" | "lg" = "lg";
	export let scale: number | null = null;
	export let icon: string | null = null;
	export let link: string | null = null;
	export let min_width: number | undefined = undefined;
</script>

<Button
	{value}
	{variant}
	{elem_id}
	{elem_classes}
	{size}
	{scale}
	{link}
	{icon}
	{min_width}
	{visible}
	disabled={mode === "static"}
	on:click
>
	{$_(value)}
</Button>
