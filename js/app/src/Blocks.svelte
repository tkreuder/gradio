<script lang="ts">
	import { tick } from "svelte";
	import { _ } from "svelte-i18n";
	import type { client } from "@gradio/client";

	import { component_map } from "./components/directory";
	import { create_loading_status_store, app_state } from "./stores";
	import type { LoadingStatusCollection } from "./stores";

	import type {
		ComponentMeta,
		Dependency,
		LayoutNode,
		Documentation
	} from "./components/types";
	import { setupi18n } from "./i18n";
	import Render from "./Render.svelte";
	import { ApiDocs } from "./api_docs/";
	import type { ThemeMode } from "./components/types";
	import { Toast } from "@gradio/statustracker";
	import type { ToastMessage } from "@gradio/statustracker";
	import type { ShareData } from "@gradio/utils";

	import logo from "./images/logo.svg";
	import api_logo from "./api_docs/img/api-logo.svg";

	setupi18n();

	export let root: string;
	export let components: ComponentMeta[];
	export let layout: LayoutNode;
	export let dependencies: Dependency[];

	export let title = "Gradio";
	export let analytics_enabled = false;
	export let target: HTMLElement;
	export let autoscroll: boolean;
	export let show_api = true;
	export let show_footer = true;
	export let control_page_title = false;
	export let app_mode: boolean;
	export let theme_mode: ThemeMode;
	export let app: Awaited<ReturnType<typeof client>>;
	export let space_id: string | null;

	let loading_status = create_loading_status_store();

	$: app_state.update((s) => ({ ...s, autoscroll }));

	let rootNode: ComponentMeta = {
		id: layout.id,
		type: "column",
		props: {},
		has_modes: false,
		instance: {} as ComponentMeta["instance"],
		component: {} as ComponentMeta["component"]
	};

	components.push(rootNode);

	const AsyncFunction = Object.getPrototypeOf(async function () {}).constructor;
	dependencies.forEach((d) => {
		if (d.js) {
			const wrap = d.backend_fn
				? d.inputs.length === 1
				: d.outputs.length === 1;
			try {
				d.frontend_fn = new AsyncFunction(
					"__fn_args",
					`let result = await (${d.js})(...__fn_args);
					return (${wrap} && !Array.isArray(result)) ? [result] : result;`
				);
			} catch (e) {
				console.error("Could not parse custom js method.");
				console.error(e);
			}
		}
	});

	let params = new URLSearchParams(window.location.search);
	let api_docs_visible = params.get("view") === "api";
	function set_api_docs_visible(visible: boolean): void {
		api_docs_visible = visible;
		let params = new URLSearchParams(window.location.search);
		if (visible) {
			params.set("view", "api");
		} else {
			params.delete("view");
		}
		history.replaceState(null, "", "?" + params.toString());
	}

	function is_dep(
		id: number,
		type: "inputs" | "outputs",
		deps: Dependency[]
	): boolean {
		for (const dep of deps) {
			for (const dep_item of dep[type]) {
				if (dep_item === id) return true;
			}
		}
		return false;
	}

	const dynamic_ids: Set<number> = new Set();
	for (const comp of components) {
		const { id, props } = comp;
		const is_input = is_dep(id, "inputs", dependencies);
		if (
			is_input ||
			(!is_dep(id, "outputs", dependencies) &&
				has_no_default_value(props?.value))
		) {
			dynamic_ids.add(id);
		}
	}

	function has_no_default_value(value: any): boolean {
		return (
			(Array.isArray(value) && value.length === 0) ||
			value === "" ||
			value === 0 ||
			!value
		);
	}

	let instance_map = components.reduce(
		(acc, next) => {
			acc[next.id] = next;
			return acc;
		},
		{} as { [id: number]: ComponentMeta }
	);

	type LoadedComponent = {
		Component: ComponentMeta["component"];
		modes?: string[];
		document?: (arg0: Record<string, unknown>) => Documentation;
	};

	async function load_component<T extends ComponentMeta["type"]>(
		name: T
	): Promise<{
		name: T;
		component: LoadedComponent;
	}> {
		try {
			const c = await component_map[name]();
			return {
				name,
				component: c as LoadedComponent
			};
		} catch (e) {
			console.error(`failed to load: ${name}`);
			console.error(e);
			throw e;
		}
	}

	const component_set = new Set<
		Promise<{ name: ComponentMeta["type"]; component: LoadedComponent }>
	>();
	const _component_map = new Map<
		ComponentMeta["type"],
		Promise<{ name: ComponentMeta["type"]; component: LoadedComponent }>
	>();

	async function walk_layout(node: LayoutNode): Promise<void> {
		let instance = instance_map[node.id];
		const _component = (await _component_map.get(instance.type))!.component;
		instance.component = _component.Component;
		if (_component.document) {
			instance.documentation = _component.document(instance.props);
		}
		if (_component.modes && _component.modes.length > 1) {
			instance.has_modes = true;
		}

		if (node.children) {
			instance.children = node.children.map((v) => instance_map[v.id]);
			await Promise.all(node.children.map((v) => walk_layout(v)));
		}
	}

	components.forEach(async (c) => {
		const _c = load_component(c.type);
		component_set.add(_c);
		_component_map.set(c.type, _c);
	});

	export let ready = false;
	Promise.all(Array.from(component_set)).then(() => {
		walk_layout(layout)
			.then(async () => {
				ready = true;
			})
			.catch((e) => {
				console.error(e);
			});
	});

	function handle_update(data: any, fn_index: number): void {
		const outputs = dependencies[fn_index].outputs;
		data?.forEach((value: any, i: number) => {
			const output = instance_map[outputs[i]];
			output.props.value_is_output = true;
			if (
				typeof value === "object" &&
				value !== null &&
				value.__type__ === "update"
			) {
				for (const [update_key, update_value] of Object.entries(value)) {
					if (update_key === "__type__") {
						continue;
					} else {
						output.props[update_key] = update_value;
					}
				}
			} else {
				output.props.value = value;
			}
		});
		rootNode = rootNode;
	}

	let submit_map: Map<number, ReturnType<typeof app.submit>> = new Map();

	function set_prop<T extends ComponentMeta>(
		obj: T,
		prop: string,
		val: any
	): void {
		if (!obj?.props) {
			obj.props = {};
		}
		obj.props[prop] = val;
		rootNode = rootNode;
	}
	let handled_dependencies: number[][] = [];

	let messages: (ToastMessage & { fn_index: number })[] = [];
	function new_message(
		message: string,
		fn_index: number,
		type: ToastMessage["type"]
	): ToastMessage & { fn_index: number } {
		return {
			message,
			fn_index,
			type,
			id: ++_error_id
		};
	}

	let _error_id = -1;

	let user_left_page = false;
	document.addEventListener("visibilitychange", function () {
		if (document.visibilityState === "hidden") {
			user_left_page = true;
		}
	});

	const MESSAGE_QUOTE_RE = /^'([^]+)'$/;

	const DUPLICATE_MESSAGE =
		"There is a long queue of requests pending. Duplicate this Space to skip.";
	const MOBILE_QUEUE_WARNING =
		"On mobile, the connection can break if this tab is unfocused or the device sleeps, losing your position in queue.";
	const MOBILE_RECONNECT_MESSAGE =
		"Lost connection due to leaving page. Rejoining queue...";
	const SHOW_DUPLICATE_MESSAGE_ON_ETA = 15;
	const SHOW_MOBILE_QUEUE_WARNING_ON_ETA = 10;
	const is_mobile_device =
		/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
			navigator.userAgent
		);
	let showed_duplicate_message = false;
	let showed_mobile_warning = false;

	async function trigger_api_call(
		dep_index: number,
		event_data: unknown = null
	): Promise<void> {
		let dep = dependencies[dep_index];
		const current_status = loading_status.get_status_for_fn(dep_index);
		messages = messages.filter(({ fn_index }) => fn_index !== dep_index);
		if (dep.cancels) {
			await Promise.all(
				dep.cancels.map(async (fn_index) => {
					const submission = submit_map.get(fn_index);
					submission?.cancel();
					return submission;
				})
			);
		}

		if (current_status === "pending" || current_status === "generating") {
			return;
		}

		let payload = {
			fn_index: dep_index,
			data: dep.inputs.map((id) => instance_map[id].props.value),
			event_data: dep.collects_event_data ? event_data : null
		};

		if (dep.frontend_fn) {
			dep
				.frontend_fn(
					payload.data.concat(
						dep.outputs.map((id) => instance_map[id].props.value)
					)
				)
				.then((v: unknown[]) => {
					if (dep.backend_fn) {
						payload.data = v;
						make_prediction();
					} else {
						handle_update(v, dep_index);
					}
				});
		} else {
			if (dep.backend_fn) {
				make_prediction();
			}
		}

		function make_prediction(): void {
			const submission = app
				.submit(payload.fn_index, payload.data as unknown[], payload.event_data)
				.on("data", ({ data, fn_index }) => {
					handle_update(data, fn_index);
				})
				.on("status", ({ fn_index, ...status }) => {
					//@ts-ignore
					loading_status.update({
						...status,
						status: status.stage,
						progress: status.progress_data,
						fn_index
					});
					if (
						!showed_duplicate_message &&
						space_id !== null &&
						status.position !== undefined &&
						status.position >= 2 &&
						status.eta !== undefined &&
						status.eta > SHOW_DUPLICATE_MESSAGE_ON_ETA
					) {
						showed_duplicate_message = true;
						messages = [
							new_message(DUPLICATE_MESSAGE, fn_index, "warning"),
							...messages
						];
					}
					if (
						!showed_mobile_warning &&
						is_mobile_device &&
						status.eta !== undefined &&
						status.eta > SHOW_MOBILE_QUEUE_WARNING_ON_ETA
					) {
						showed_mobile_warning = true;
						messages = [
							new_message(MOBILE_QUEUE_WARNING, fn_index, "warning"),
							...messages
						];
					}

					if (status.stage === "complete") {
						dependencies.map(async (dep, i) => {
							if (dep.trigger_after === fn_index) {
								trigger_api_call(i);
							}
						});

						submission.destroy();
					}
					if (status.broken && is_mobile_device && user_left_page) {
						window.setTimeout(() => {
							messages = [
								new_message(MOBILE_RECONNECT_MESSAGE, fn_index, "error"),
								...messages
							];
						}, 0);
						trigger_api_call(dep_index, event_data);
						user_left_page = false;
					} else if (status.stage === "error") {
						if (status.message) {
							const _message = status.message.replace(
								MESSAGE_QUOTE_RE,
								(_, b) => b
							);
							messages = [
								new_message(_message, fn_index, "error"),
								...messages
							];
						}
						dependencies.map(async (dep, i) => {
							if (
								dep.trigger_after === fn_index &&
								!dep.trigger_only_on_success
							) {
								trigger_api_call(i);
							}
						});

						submission.destroy();
					}
				})
				.on("log", ({ log, fn_index, level }) => {
					messages = [new_message(log, fn_index, level), ...messages];
				});

			submit_map.set(dep_index, submission);
		}
	}

	function trigger_share(title: string | undefined, description: string): void {
		if (space_id === null) {
			return;
		}
		const discussion_url = new URL(
			`https://huggingface.co/spaces/${space_id}/discussions/new`
		);
		if (title !== undefined && title.length > 0) {
			discussion_url.searchParams.set("title", title);
		}
		discussion_url.searchParams.set("description", description);
		window.open(discussion_url.toString(), "_blank");
	}

	function handle_error_close(e: Event & { detail: number }): void {
		const _id = e.detail;
		messages = messages.filter((m) => m.id !== _id);
	}

	const is_external_url = (link: string | null): boolean =>
		!!(link && new URL(link, location.href).origin !== location.origin);

	let attached_error_listeners: number[] = [];
	let shareable_components: number[] = [];
	async function handle_mount(): Promise<void> {
		await tick();

		var a = target.getElementsByTagName("a");

		for (var i = 0; i < a.length; i++) {
			const _target = a[i].getAttribute("target");
			const _link = a[i].getAttribute("href");

			// only target anchor tags with external links
			if (is_external_url(_link) && _target !== "_blank")
				a[i].setAttribute("target", "_blank");
		}

		dependencies.forEach((dep, i) => {
			let { targets, trigger, inputs, outputs } = dep;
			const target_instances: [number, ComponentMeta][] = targets.map((t) => [
				t,
				instance_map[t]
			]);

			// page events
			if (
				targets.length === 0 &&
				!handled_dependencies[i]?.includes(-1) &&
				trigger === "load" &&
				// check all input + output elements are on the page
				outputs.every((v) => instance_map?.[v].instance) &&
				inputs.every((v) => instance_map?.[v].instance)
			) {
				trigger_api_call(i);
				handled_dependencies[i] = [-1];
			}

			// component events
			target_instances
				.filter((v) => !!v && !!v[1])
				.forEach(([id, { instance }]: [number, ComponentMeta]) => {
					if (handled_dependencies[i]?.includes(id) || !instance) return;
					instance?.$on(trigger, (event_data: any) => {
						trigger_api_call(i, event_data.detail);
					});

					if (!handled_dependencies[i]) handled_dependencies[i] = [];
					handled_dependencies[i].push(id);
				});
		});
		// share events
		components.forEach((c) => {
			if (
				c.props.show_share_button &&
				!shareable_components.includes(c.id) // only one share listener per component
			) {
				shareable_components.push(c.id);
				c.instance.$on("share", (event_data) => {
					const { title, description } = event_data.detail as ShareData;
					trigger_share(title, description);
				});
			}
		});

		components.forEach((c) => {
			if (!attached_error_listeners.includes(c.id)) {
				if (c.instance) {
					attached_error_listeners.push(c.id);
					c.instance.$on("error", (event_data: any) => {
						messages = [
							new_message(event_data.detail, -1, "error"),
							...messages
						];
					});
				}
			}
		});
	}

	function handle_destroy(id: number): void {
		handled_dependencies = handled_dependencies.map((dep) => {
			return dep.filter((_id) => _id !== id);
		});
	}

	$: set_status($loading_status);

	dependencies.forEach((v, i) => {
		loading_status.register(i, v.inputs, v.outputs);
	});

	function set_status(statuses: LoadingStatusCollection): void {
		for (const id in statuses) {
			let loading_status = statuses[id];
			let dependency = dependencies[loading_status.fn_index];
			loading_status.scroll_to_output = dependency.scroll_to_output;
			loading_status.show_progress = dependency.show_progress;

			set_prop(instance_map[id], "loading_status", loading_status);
		}
		const inputs_to_update = loading_status.get_inputs_to_update();
		for (const [id, pending_status] of inputs_to_update) {
			set_prop(instance_map[id], "pending", pending_status === "pending");
		}
	}
</script>

<svelte:head>
	{#if control_page_title}
		<title>{title}</title>
	{/if}
	{#if analytics_enabled}
		<script
			async
			defer
			src="https://www.googletagmanager.com/gtag/js?id=UA-156449732-1"
		></script>
		<script>
			window.dataLayer = window.dataLayer || [];
			function gtag() {
				dataLayer.push(arguments);
			}
			gtag("js", new Date());
			gtag("config", "UA-156449732-1");
		</script>
	{/if}
</svelte:head>

<div class="wrap" style:min-height={app_mode ? "100%" : "auto"}>
	<div class="contain" style:flex-grow={app_mode ? "1" : "auto"}>
		{#if ready}
			<Render
				has_modes={rootNode.has_modes}
				component={rootNode.component}
				id={rootNode.id}
				props={rootNode.props}
				children={rootNode.children}
				{dynamic_ids}
				{instance_map}
				{root}
				{target}
				{theme_mode}
				on:mount={handle_mount}
				on:destroy={({ detail }) => handle_destroy(detail)}
			/>
		{/if}
	</div>

	{#if show_footer}
		<footer>
			{#if show_api}
				<button
					on:click={() => {
						set_api_docs_visible(!api_docs_visible);
					}}
					class="show-api"
				>
					Use via API <img src={api_logo} alt="" />
				</button>
				<div>·</div>
			{/if}
			<a
				href="https://gradio.app"
				class="built-with"
				target="_blank"
				rel="noreferrer"
			>
				Built with Gradio
				<img src={logo} alt="logo" />
			</a>
		</footer>
	{/if}
</div>

{#if api_docs_visible && ready}
	<div class="api-docs">
		<!-- TODO: fix -->
		<!-- svelte-ignore a11y-click-events-have-key-events-->
		<!-- svelte-ignore a11y-no-static-element-interactions-->
		<div
			class="backdrop"
			on:click={() => {
				set_api_docs_visible(false);
			}}
		/>
		<div class="api-docs-wrap">
			<ApiDocs
				on:close={() => {
					set_api_docs_visible(false);
				}}
				{instance_map}
				{dependencies}
				{root}
				{app}
			/>
		</div>
	</div>
{/if}

{#if messages}
	<Toast {messages} on:close={handle_error_close} />
{/if}

<style>
	.wrap {
		display: flex;
		flex-grow: 1;
		flex-direction: column;
		width: var(--size-full);
		font-weight: var(--body-text-weight);
		font-size: var(--body-text-size);
	}

	footer {
		display: flex;
		justify-content: center;
		margin-top: var(--size-4);
		color: var(--body-text-color-subdued);
	}

	footer > * + * {
		margin-left: var(--size-2);
	}

	.show-api {
		display: flex;
		align-items: center;
	}
	.show-api:hover {
		color: var(--body-text-color);
	}

	.show-api img {
		margin-right: var(--size-1);
		margin-left: var(--size-2);
		width: var(--size-3);
	}

	.built-with {
		display: flex;
		align-items: center;
	}

	.built-with:hover {
		color: var(--body-text-color);
	}

	.built-with img {
		margin-right: var(--size-1);
		margin-left: var(--size-2);
		width: var(--size-3);
	}

	.api-docs {
		display: flex;
		position: fixed;
		top: 0;
		right: 0;
		z-index: var(--layer-5);
		background: rgba(0, 0, 0, 0.5);
		width: var(--size-screen);
		height: var(--size-screen-h);
	}

	.backdrop {
		flex: 1 1 0%;
		backdrop-filter: blur(4px);
	}

	.api-docs-wrap {
		box-shadow: var(--shadow-drop-lg);
		background: var(--background-fill-primary);
		overflow-x: hidden;
		overflow-y: auto;
	}

	@media (--screen-md) {
		.api-docs-wrap {
			border-top-left-radius: var(--radius-lg);
			border-bottom-left-radius: var(--radius-lg);
			width: 950px;
		}
	}

	@media (--screen-xxl) {
		.api-docs-wrap {
			width: 1150px;
		}
	}
</style>
