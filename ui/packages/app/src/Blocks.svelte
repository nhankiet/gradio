<script lang="ts">
	import type { SvelteComponentTyped } from "svelte";
	import { onMount } from "svelte";
	import { component_map } from "./components/directory";
	import { loading_status, app_state } from "./stores";
	import type { LoadingStatus } from "./components/StatusTracker/types";

	import { _ } from "svelte-i18n";
	import { setupi18n } from "./i18n";
	import Render from "./Render.svelte";
	import ApiDocs from "./ApiDocs.svelte";
	import { tick } from "svelte";
	import logo from "./images/logo.svg";
	setupi18n();

	interface Component {
		id: number;
		type: string;
		has_modes?: boolean;
		props: {
			name?: keyof typeof component_map;
			css?: Record<string, string>;
			visible?: boolean;
			elem_id?: string;
			[key: string]: unknown;
			value?: unknown;
		};
		instance?: SvelteComponentTyped;
		component?: any;
		children?: Array<Component>;
	}

	interface LayoutNode {
		id: number;
		children: Array<LayoutNode>;
	}

	interface Dependency {
		trigger: string;
		targets: Array<number>;
		inputs: Array<number>;
		outputs: Array<number>;
		backend_fn: boolean;
		js: string | null;
		scroll_to_output: boolean;
		show_progress: boolean;
		frontend_fn?: Function;
		status_tracker: number | null;
		status?: string;
		queue: boolean | null;
		api_name: string | null;
		documentation?: Array<Array<string>>;
	}

	export let root: string;
	export let fn: (...args: any) => Promise<unknown>;
	export let components: Array<Component>;
	export let layout: LayoutNode;
	export let dependencies: Array<Dependency>;
	export let theme: string;

	export let enable_queue: boolean;
	export let title: string = "Gradio";
	export let analytics_enabled: boolean = false;
	export let target: HTMLElement;
	export let css: string;
	export let id: number = 0;
	export let autoscroll: boolean = false;

	$: app_state.update((s) => ({ ...s, autoscroll }));

	let rootNode: Component = { id: layout.id, type: "column", props: {} };
	components.push(rootNode);

	const AsyncFunction = Object.getPrototypeOf(async function () {}).constructor;
	dependencies.forEach((d) => {
		if (d.js) {
			try {
				d.frontend_fn = new AsyncFunction(
					"__fn_args",
					`let result = await (${d.js})(...__fn_args);
					return ${d.outputs.length} === 1 ? [result] : result;`
				);
			} catch (e) {
				console.error("Could not parse custom js method.");
				console.error(e);
			}
		}
	});
	let show_api_docs = dependencies.some((d) => "documentation" in d);
	let api_docs_visible = false;

	function is_dep(
		id: number,
		type: "inputs" | "outputs",
		deps: Array<Dependency>
	) {
		let dep_index = 0;
		for (;;) {
			const dep = deps[dep_index];
			if (dep === undefined) break;

			let dep_item_index = 0;
			for (;;) {
				const dep_item = dep[type][dep_item_index];
				if (dep_item === undefined) break;
				if (dep_item === id) return true;
				dep_item_index++;
			}

			dep_index++;
		}

		return false;
	}

	const dynamic_ids = components.reduce((acc, { id, props }) => {
		const is_input = is_dep(id, "inputs", dependencies);
		const is_output = is_dep(id, "outputs", dependencies);

		if (!is_input && !is_output && has_no_default_value(props.value))
			acc.add(id); // default dynamic
		if (is_input) acc.add(id);

		return acc;
	}, new Set());

	function has_no_default_value(value: any) {
		return (
			(Array.isArray(value) && value.length === 0) ||
			value === "" ||
			value === 0 ||
			!value
		);
	}

	let instance_map = components.reduce((acc, next) => {
		acc[next.id] = next;
		return acc;
	}, {} as { [id: number]: Component });

	function load_component<T extends keyof typeof component_map>(
		name: T
	): Promise<{ name: T; component: SvelteComponentTyped }> {
		return new Promise(async (res, rej) => {
			try {
				const c = await component_map[name]();
				res({ name, component: c });
			} catch (e) {
				console.error("failed to load: " + name);
				rej(e);
			}
		});
	}

	async function walk_layout(node: LayoutNode) {
		let instance = instance_map[node.id];
		const _component = (await _component_map.get(instance.type)).component;
		instance.component = _component.Component;
		if (_component.modes.length > 1) {
			instance.has_modes = true;
		}

		if (node.children) {
			instance.children = node.children.map((v) => instance_map[v.id]);
			await Promise.all(node.children.map((v) => walk_layout(v)));
		}
	}

	const component_set = new Set();
	const _component_map = new Map();
	components.forEach((c) => {
		const _c = load_component(c.type);
		component_set.add(_c);
		_component_map.set(c.type, _c);
	});

	let ready = false;
	Promise.all(Array.from(component_set)).then(() => {
		walk_layout(layout)
			.then(async () => {
				ready = true;

				await tick();
				window.__gradio_loader__[id].$set({ status: "complete" });
			})
			.catch((e) => {
				console.error(e);
				window.__gradio_loader__[id].$set({ status: "error" });
			});
	});

	function set_prop(obj: Component, prop: string, val: any) {
		if (!obj?.props) {
			obj.props = {};
		}
		obj.props[prop] = val;
		rootNode = rootNode;
	}

	let handled_dependencies: Array<number[]> = [];

	async function handle_mount() {
		await tick();

		var a = target.getElementsByTagName("a");

		for (var i = 0; i < a.length; i++) {
			const _target = a[i].getAttribute("target");
			if (_target !== "_blank") a[i].setAttribute("target", "_blank");
		}

		dependencies.forEach(
			(
				{ targets, trigger, inputs, outputs, queue, backend_fn, frontend_fn },
				i
			) => {
				const target_instances: [number, Component][] = targets.map((t) => [
					t,
					instance_map[t]
				]);

				// page events
				if (
					targets.length === 0 &&
					!handled_dependencies[i]?.includes(-1) &&
					trigger === "load" &&
					// check all input + output elements are on the page
					outputs.every((v) => instance_map[v].instance) &&
					inputs.every((v) => instance_map[v].instance)
				) {
					fn({
						action: "predict",
						backend_fn,
						frontend_fn,
						payload: {
							fn_index: i,
							data: inputs.map((id) => instance_map[id].props.value)
						},
						queue: queue === null ? enable_queue : queue
					})
						.then((output) => {
							output.data.forEach((value, i) => {
								if (
									typeof value === "object" &&
									value !== null &&
									value.__type__ == "update"
								) {
									for (const [update_key, update_value] of Object.entries(
										value
									)) {
										if (update_key === "__type__") {
											continue;
										} else {
											instance_map[outputs[i]].props[update_key] = update_value;
										}
									}
									rootNode = rootNode;
								} else {
									instance_map[outputs[i]].props.value = value;
								}
							});
						})
						.catch((error) => {
							console.error(error);
							loading_status.update(i, "error", 0, 0);
						});

					handled_dependencies[i] = [-1];
				}

				target_instances.forEach(([id, { instance }]: [number, Component]) => {
					if (handled_dependencies[i]?.includes(id) || !instance) return;
					instance?.$on(trigger, () => {
						if (loading_status.get_status_for_fn(i) === "pending") {
							return;
						}

						// page events
						fn({
							action: "predict",
							backend_fn,
							frontend_fn,
							payload: {
								fn_index: i,
								data: inputs.map((id) => instance_map[id].props.value)
							},
							output_data: outputs.map((id) => instance_map[id].props.value),
							queue: queue === null ? enable_queue : queue
						})
							.then((output) => {
								output.data.forEach((value, i) => {
									if (
										typeof value === "object" &&
										value !== null &&
										value.__type__ === "update"
									) {
										for (const [update_key, update_value] of Object.entries(
											value
										)) {
											if (update_key === "__type__") {
												continue;
											} else {
												instance_map[outputs[i]].props[update_key] =
													update_value;
											}
										}
										rootNode = rootNode;
									} else {
										instance_map[outputs[i]].props.value = value;
									}
								});
							})
							.catch((error) => {
								console.error(error);
								loading_status.update(i, "error", 0, 0);
							});
					});

					if (!handled_dependencies[i]) handled_dependencies[i] = [];
					handled_dependencies[i].push(id);
				});
			}
		);
	}

	function handle_destroy(id: number) {
		handled_dependencies = handled_dependencies.map((dep) => {
			return dep.filter((_id) => _id !== id);
		});
	}

	$: set_status($loading_status);

	dependencies.forEach((v, i) => {
		loading_status.register(i, v.inputs, v.outputs);
	});

	function set_status(statuses: Record<number, LoadingStatus>) {
		if (window.__gradio_mode__ === "website") {
			return;
		}
		for (const id in statuses) {
			let loading_status = statuses[id];
			let dependency = dependencies[loading_status.fn_index];
			loading_status.scroll_to_output = dependency.scroll_to_output;
			loading_status.visible = dependency.show_progress;

			set_prop(instance_map[id], "loading_status", loading_status);
		}
		const inputs_to_update = loading_status.get_inputs_to_update();
		for (const [id, pending_status] of inputs_to_update) {
			set_prop(instance_map[id], "pending", pending_status === "pending");
		}
	}

	function handle_darkmode() {
		let url = new URL(window.location.toString());

		const color_mode: "light" | "dark" | "system" | null = url.searchParams.get(
			"__theme"
		) as "light" | "dark" | "system" | null;

		if (color_mode !== null) {
			if (color_mode === "dark") {
				target.classList.add("dark");
			} else if (color_mode === "system") {
				use_system_theme();
			}
			// light is default, so we don't need to do anything else
		} else if (url.searchParams.get("__dark-theme") === "true") {
			target.classList.add("dark");
		} else {
			use_system_theme();
		}
	}

	function use_system_theme() {
		update_scheme();
		window
			?.matchMedia("(prefers-color-scheme: dark)")
			?.addEventListener("change", update_scheme);

		function update_scheme() {
			const is_dark =
				window?.matchMedia?.("(prefers-color-scheme: dark)").matches ?? null;

			if (is_dark) target.classList.add("dark");
		}
	}

	if (window.__gradio_mode__ !== "website") {
		handle_darkmode();
	}
</script>

<svelte:head>
	<title>{title}</title>
	{#if analytics_enabled}
		<script
			async
			defer
			src="https://www.googletagmanager.com/gtag/js?id=UA-156449732-1"></script>
	{/if}
</svelte:head>

<div class="w-full flex flex-col">
	<div
		class="mx-auto container px-4 py-6 dark:bg-gray-950"
		class:flex-grow={window.__gradio_mode__ === "app"}
	>
		{#if api_docs_visible}
			<ApiDocs {components} {dependencies} {root} />
		{:else if ready}
			<Render
				component={rootNode.component}
				id={rootNode.id}
				props={rootNode.props}
				children={rootNode.children}
				{dynamic_ids}
				{instance_map}
				{theme}
				{root}
				on:mount={handle_mount}
				on:destroy={({ detail }) => handle_destroy(detail)}
			/>
		{/if}
	</div>
	<footer
		class="flex justify-center pb-6 text-gray-300 dark:text-gray-500 font-semibold"
	>
		{#if show_api_docs}
			<div
				class="cursor-pointer hover:text-gray-400 dark:hover:text-gray-400 transition-colors"
				on:click={() => {
					api_docs_visible = !api_docs_visible;
				}}
			>
				{#if api_docs_visible}hide{:else}view{/if} api
			</div>
			&nbsp; &bull; &nbsp;
		{/if}
		<a
			href="https://gradio.app"
			target="_blank"
			rel="noreferrer"
			class="group hover:text-gray-400 dark:hover:text-gray-400 transition-colors"
		>
			{$_("interface.built_with_Gradio")}
			<img
				class="h-[22px] ml-0.5 inline-block pb-0.5 filter grayscale opacity-50 group-hover:grayscale-0 group-hover:opacity-100 transition"
				src={logo}
				alt="logo"
			/>
		</a>
	</footer>
</div>
