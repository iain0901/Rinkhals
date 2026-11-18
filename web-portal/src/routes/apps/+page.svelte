<script lang="ts">
	import { onMount, onDestroy } from "svelte";
	import QRCode from "qrcode";
	import ReportField from "$lib/ReportField.svelte";
	import { printerState } from "$lib/printerState";
	import {
		Boxes,
		Play,
		Square,
		RotateCw,
		Sliders,
		Cpu,
		HardDrive,
		Search,
		X,
		RotateCcw,
		Loader2,
		AlertCircle,
		Download,
		CheckCircle2,
		RefreshCw,
		Package,
		ExternalLink,
		Trash2,
		AlertTriangle,
		Copy,
		Check
	} from "lucide-svelte";

	type Property = {
		key: string;
		display: string;
		type: string;
		options?: string[];
		default: string;
		value: string;
		overridden: boolean;
		advanced?: boolean;
	};

	type App = {
		id: string;
		name: string;
		description: string;
		version: string;
		source: "system" | "user";
		enabled: boolean;
		running: boolean;
		requirements?: { memory?: number; cpu?: number };
		properties: Property[];
	};

	type CatalogApp = {
		id: string;
		name: string;
		description: string;
		version: string;
		requirements?: { memory?: number; cpu?: number };
		depends?: string[];
		available_for_model: boolean;
		download_url?: string;
		asset_size?: number;
		installed: boolean;
		installed_version?: string;
		upstream?: { type: string; repo: string; strip_prefix?: string; branch?: string };
		upstream_version?: string;
		upstream_url?: string;
	};

	type Catalog = {
		release: string;
		fetched_at: string;
		model: string;
		asset_group: string;
		apps: CatalogApp[];
		notice?: string;
	};

	const apiHost = import.meta.env.DEV ? "http://localhost:8090" : "";

	let tab = $state<"installed" | "catalog">("installed");

	// Show-advanced is UX gating, not a security boundary. Persisted so the
	// preference survives reloads; default off so the typical user sees only
	// the knobs the app author considered safe to expose.
	let showAdvanced = $state(false);
	if (typeof window !== "undefined") {
		showAdvanced = window.localStorage.getItem("rinkhals-show-advanced") === "1";
	}
	$effect(() => {
		if (typeof window === "undefined") return;
		window.localStorage.setItem("rinkhals-show-advanced", showAdvanced ? "1" : "0");
	});

	// A property is shown when it isn't advanced, or when the user has opted in.
	function visibleProperties(props: Property[]): Property[] {
		return props.filter((p) => showAdvanced || !p.advanced);
	}
	function hasVisibleProperties(app: App): boolean {
		return visibleProperties(app.properties).length > 0;
	}

	let apps = $state<App[]>([]);
	let loading = $state(true);
	let error = $state("");

	let catalog = $state<Catalog | null>(null);
	let catalogLoading = $state(false);
	let catalogError = $state("");
	let installingId = $state<string | null>(null);

	let searchQuery = $state("");
	let stateFilter = $state<"all" | "enabled" | "disabled" | "running">("all");
	let sourceFilter = $state<"all" | "system" | "user">("all");
	let catalogSearch = $state("");

	let busyApps = $state<Set<string>>(new Set());
	let uninstallTarget = $state<App | null>(null);
	let uninstallRunning = $state(false);
	let configureApp = $state<App | null>(null);

	// Per-card description expansion. We only show the "Show more" affordance
	// when the description is actually overflowing its clamp, measured at
	// runtime so the threshold is correct regardless of viewport width or
	// font size. Card ids are unique across both tabs (installed app id ==
	// catalog app id for the same app), so a single Map works for both.
	let truncatedDescs = $state<Set<string>>(new Set());
	let expandedDescs = $state<Set<string>>(new Set());

	// Render a URL into a canvas as a QR code. The action re-renders whenever
	// the URL parameter changes, so polling /api/apps and seeing the property
	// value update automatically refreshes the displayed QR without any
	// manual coordination from the caller.
	function qrCanvas(node: HTMLCanvasElement, url: string) {
		const render = (value: string) => {
			if (!value) return;
			QRCode.toCanvas(node, value, {
				margin: 1,
				width: 220,
				color: { dark: "#1a1a1a", light: "#ffffff" },
				errorCorrectionLevel: "M"
			}).catch((e) => console.error("QR render failed", e));
		};
		render(url);
		return {
			update(newUrl: string) {
				render(newUrl);
			}
		};
	}

	let copiedKey = $state<string | null>(null);
	async function copyToClipboard(key: string, text: string) {
		try {
			await navigator.clipboard.writeText(text);
			copiedKey = key;
			setTimeout(() => {
				if (copiedKey === key) copiedKey = null;
			}, 1500);
		} catch (e) {
			showToast("複製失敗", "err");
		}
	}

	function toggleDesc(id: string) {
		const next = new Set(expandedDescs);
		if (next.has(id)) next.delete(id);
		else next.add(id);
		expandedDescs = next;
	}

	// Svelte action: measures whether the clamped description is overflowing
	// and adds the id to truncatedDescs if so. Re-checks on viewport resize.
	// Skipped while the card is expanded (no clamp -> would always look like
	// it fits, which would hide the toggle button mid-interaction).
	function descAction(node: HTMLElement, currentId: string) {
		let id = currentId;
		const check = () => {
			requestAnimationFrame(() => {
				if (expandedDescs.has(id)) return;
				const overflows = node.scrollHeight > node.clientHeight + 1;
				const has = truncatedDescs.has(id);
				if (overflows && !has) {
					truncatedDescs = new Set([...truncatedDescs, id]);
				} else if (!overflows && has) {
					const next = new Set(truncatedDescs);
					next.delete(id);
					truncatedDescs = next;
				}
			});
		};
		check();
		const ro = new ResizeObserver(check);
		ro.observe(node);
		return {
			update(newId: string) {
				id = newId;
				check();
			},
			destroy() {
				ro.disconnect();
			}
		};
	}
	let pendingConfig = $state<Record<string, string>>({});
	let configSaving = $state(false);
	let toast = $state<{ text: string; tone: "ok" | "err" } | null>(null);

	let pollHandle: ReturnType<typeof setInterval> | undefined;

	async function fetchApps() {
		try {
			const res = await fetch(`${apiHost}/api/apps`);
			if (!res.ok) throw new Error(`HTTP ${res.status}`);
			const fresh: App[] = await res.json();
			apps = fresh;
			error = "";
		} catch (e: any) {
			error = e.message || String(e);
		} finally {
			loading = false;
		}
	}

	async function fetchCatalog(force = false) {
		catalogLoading = true;
		try {
			const url = `${apiHost}/api/catalog${force ? "?refresh=1" : ""}`;
			const res = await fetch(url);
			if (!res.ok) throw new Error(`HTTP ${res.status}`);
			catalog = await res.json();
			catalogError = "";
		} catch (e: any) {
			catalogError = e.message || String(e);
		} finally {
			catalogLoading = false;
		}
	}

	async function installCatalogApp(entry: CatalogApp) {
		if (!entry.available_for_model || !entry.download_url) return;
		installingId = entry.id;
		try {
			const res = await fetch(`${apiHost}/api/catalog/${entry.id}/install`, { method: "POST" });
			const data = await res.json().catch(() => ({}));
			if (!res.ok || data.success === false) {
				throw new Error(data.output || `HTTP ${res.status}`);
			}
			// Refresh installed list first so we can locate the new app below.
			await Promise.all([fetchApps(), fetchCatalog(true)]);

			if (data.needs_config) {
				// Left disabled on purpose: open the configure drawer so the user
				// can set required values, then enable from there.
				showToast(`已安裝 ${entry.name}，請先設定再啟用`);
				tab = "installed";
				const installed = apps.find((a) => a.id === entry.id);
				if (installed) openConfigure(installed);
			} else if (data.auto_enabled) {
				showToast(`已安裝並啟動 ${entry.name}`);
			} else {
				showToast(`已安裝 ${entry.name}`);
			}
		} catch (e: any) {
			showToast(e.message ? `安裝失敗：${e.message}` : "安裝失敗", "err");
		} finally {
			installingId = null;
		}
	}

	// Enable + start an app, optionally saving pending config first. Used by the
	// configure drawer to complete the "configure then enable" flow for apps that
	// were installed but left disabled because they needed configuration.
	async function saveAndEnable(app: App) {
		if (hasPendingChanges()) {
			await saveConfig();
		}
		try {
			await callApp(app.id, "/enable", { method: "POST" });
			await callApp(app.id, "/action", {
				method: "POST",
				body: JSON.stringify({ action: "start" })
			});
			showToast(`已啟用並啟動 ${app.name}`);
			const refreshed = apps.find((a) => a.id === app.id);
			if (refreshed) configureApp = refreshed;
		} catch {
			// callApp already surfaced an error toast
		}
	}

	onMount(() => {
		fetchApps();
		fetchCatalog(false);
		pollHandle = setInterval(fetchApps, 5000);
	});

	onDestroy(() => {
		if (pollHandle) clearInterval(pollHandle);
	});

	function showToast(text: string, tone: "ok" | "err" = "ok") {
		toast = { text, tone };
		setTimeout(() => {
			toast = null;
		}, 2500);
	}

	function markBusy(id: string, on: boolean) {
		const next = new Set(busyApps);
		if (on) next.add(id);
		else next.delete(id);
		busyApps = next;
	}

	async function callApp(id: string, path: string, init: RequestInit = {}) {
		markBusy(id, true);
		try {
			const res = await fetch(`${apiHost}/api/apps/${id}${path}`, {
				...init,
				headers: { "Content-Type": "application/json", ...(init.headers || {}) }
			});
			if (!res.ok) throw new Error(`HTTP ${res.status}`);
			const data = await res.json().catch(() => ({}));
			if (data && data.success === false) {
				throw new Error(data.output || "操作失敗");
			}
			await fetchApps();
			return data;
		} catch (e: any) {
			showToast(e.message && e.message !== "操作失敗" ? `操作失敗：${e.message}` : "操作失敗", "err");
			throw e;
		} finally {
			markBusy(id, false);
		}
	}

	async function toggleEnabled(app: App) {
		try {
			await callApp(app.id, app.enabled ? "/disable" : "/enable", { method: "POST" });
			showToast(`已${app.enabled ? "停用" : "啟用"} ${app.name}`);
		} catch {
			// toast already shown
		}
	}

	async function confirmUninstall(app: App) {
		if (app.source !== "user") return;
		uninstallTarget = app;
	}

	async function performUninstall() {
		if (!uninstallTarget) return;
		const app = uninstallTarget;
		uninstallRunning = true;
		try {
			const res = await fetch(`${apiHost}/api/apps/${app.id}`, { method: "DELETE" });
			const data = await res.json().catch(() => ({}));
			if (!res.ok || data.success === false) {
				throw new Error(data.output || `HTTP ${res.status}`);
			}
			showToast(`已解除安裝 ${app.name}`);
			// If the drawer is open on this app, close it: the app is gone.
			if (configureApp?.id === app.id) configureApp = null;
			// Refresh both lists. The catalog's "Installed" pill should flip back to "Install".
			await Promise.all([fetchApps(), fetchCatalog(true)]);
		} catch (e: any) {
			showToast(e.message ? `解除安裝失敗：${e.message}` : "解除安裝失敗", "err");
		} finally {
			uninstallRunning = false;
			uninstallTarget = null;
		}
	}

	async function runAction(app: App, action: "start" | "stop" | "restart") {
		try {
			await callApp(app.id, "/action", {
				method: "POST",
				body: JSON.stringify({ action })
			});
			showToast(`已對 ${app.name} 送出${{ start: "啟動", stop: "停止", restart: "重新啟動" }[action]}指令`);
		} catch {}
	}

	function openConfigure(app: App) {
		configureApp = app;
		pendingConfig = {};
	}

	function closeConfigure() {
		configureApp = null;
		pendingConfig = {};
	}

	function pendingValue(prop: Property): string {
		return pendingConfig[prop.key] ?? prop.value;
	}

	function hasPendingChanges(): boolean {
		return Object.keys(pendingConfig).length > 0;
	}

	async function saveConfig() {
		if (!configureApp || !hasPendingChanges()) return;
		configSaving = true;
		try {
			const res = await fetch(`${apiHost}/api/apps/${configureApp.id}/config`, {
				method: "PUT",
				headers: { "Content-Type": "application/json" },
				body: JSON.stringify(pendingConfig)
			});
			const data = await res.json().catch(() => ({}));
			if (!res.ok || data.success === false) {
				throw new Error(data.output || `HTTP ${res.status}`);
			}
			showToast("設定已儲存");
			await fetchApps();
			const refreshed = apps.find((a) => a.id === configureApp!.id);
			if (refreshed) configureApp = refreshed;
			pendingConfig = {};
		} catch (e: any) {
			showToast(e.message ? `儲存失敗：${e.message}` : "儲存失敗", "err");
		} finally {
			configSaving = false;
		}
	}

	async function resetProperty(prop: Property) {
		if (!configureApp) return;
		try {
			const res = await fetch(`${apiHost}/api/apps/${configureApp.id}/config/${prop.key}`, {
				method: "DELETE"
			});
			if (!res.ok) throw new Error(`HTTP ${res.status}`);
			showToast(`${prop.display || prop.key} 已還原為預設值`);
			const fresh = { ...pendingConfig };
			delete fresh[prop.key];
			pendingConfig = fresh;
			await fetchApps();
			const refreshed = apps.find((a) => a.id === configureApp!.id);
			if (refreshed) configureApp = refreshed;
		} catch (e: any) {
			showToast(e.message ? `重設失敗：${e.message}` : "重設失敗", "err");
		}
	}

	async function clearAllOverrides() {
		if (!configureApp) return;
		try {
			const res = await fetch(`${apiHost}/api/apps/${configureApp.id}/config`, {
				method: "DELETE"
			});
			if (!res.ok) throw new Error(`HTTP ${res.status}`);
			showToast("已清除所有自訂設定");
			pendingConfig = {};
			await fetchApps();
			const refreshed = apps.find((a) => a.id === configureApp!.id);
			if (refreshed) configureApp = refreshed;
		} catch (e: any) {
			showToast(e.message ? `清除失敗：${e.message}` : "清除失敗", "err");
		}
	}

	let visibleCatalog = $derived(
		(catalog?.apps ?? []).filter((a) => {
			if (!catalogSearch) return true;
			const q = catalogSearch.toLowerCase();
			return (
				a.id.toLowerCase().includes(q) ||
				a.name.toLowerCase().includes(q) ||
				(a.description || "").toLowerCase().includes(q)
			);
		})
	);

	function formatSize(bytes?: number): string {
		if (!bytes || bytes <= 0) return "";
		if (bytes < 1024) return `${bytes} B`;
		if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
		return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
	}

	function relativeTime(iso?: string): string {
		if (!iso) return "";
		try {
			const then = new Date(iso).getTime();
			const diffMs = Date.now() - then;
			const m = Math.floor(diffMs / 60000);
			if (m < 1) return "剛剛";
			if (m < 60) return `${m} 分鐘前`;
			const h = Math.floor(m / 60);
			if (h < 24) return `${h} 小時前`;
			return `${Math.floor(h / 24)} 天前`;
		} catch {
			return "";
		}
	}

	let visibleApps = $derived(
		apps.filter((a) => {
			if (searchQuery) {
				const q = searchQuery.toLowerCase();
				if (
					!a.id.toLowerCase().includes(q) &&
					!a.name.toLowerCase().includes(q) &&
					!(a.description || "").toLowerCase().includes(q)
				) {
					return false;
				}
			}
			if (stateFilter === "enabled" && !a.enabled) return false;
			if (stateFilter === "disabled" && a.enabled) return false;
			if (stateFilter === "running" && !a.running) return false;
			if (sourceFilter !== "all" && a.source !== sourceFilter) return false;
			return true;
		})
	);

	function statusPill(app: App): { label: string; classes: string } {
		if (!app.enabled) {
			return {
				label: "已停用",
				classes: "bg-surface-accent text-ink-faint border-line-soft"
			};
		}
		if (app.running) {
			return {
				label: "執行中",
				classes: "bg-brand-soft text-brand border-brand/30"
			};
		}
		return {
			label: "已停止",
			classes: "bg-surface-warm text-accent-hover border-accent/30"
		};
	}
</script>

<svelte:head>
	<title>應用程式 - Rinkhals</title>
</svelte:head>

<div class="space-y-6 max-w-7xl mx-auto w-full">
	<header class="pb-4 border-b border-line-soft">
		<!-- Row 1: title + tab toggle (fixed positions) -->
		<div class="flex items-start justify-between gap-4">
			<div class="min-w-0">
				<p class="text-xs uppercase tracking-wider text-ink-faint font-medium">應用程式</p>
				<h2 class="text-3xl font-semibold text-ink mt-1 tracking-tight flex items-center gap-2">
					<Boxes size={26} class="text-brand" />
					應用程式
				</h2>
				<!-- Reserved subtitle slot; height fixed so swapping text doesn't shift layout -->
				<p class="text-ink-muted text-sm mt-2 h-5">
					{tab === "installed"
						? "啟用、啟動及設定這台印表機上已安裝的應用程式。"
						: "瀏覽並安裝 Rinkhals.Apps 提供的社群應用程式。"}
				</p>
			</div>

			<div class="inline-flex bg-surface border border-line-soft rounded-lg p-0.5 shrink-0">
				<button
					type="button"
					onclick={() => (tab = "installed")}
					class="px-3 py-1.5 rounded-md text-sm font-medium transition-colors {tab === 'installed' ? 'bg-canvas text-brand shadow-sm' : 'text-ink-muted hover:text-ink'}"
				>
					已安裝
				</button>
				<button
					type="button"
					onclick={() => (tab = "catalog")}
					class="px-3 py-1.5 rounded-md text-sm font-medium transition-colors {tab === 'catalog' ? 'bg-canvas text-brand shadow-sm' : 'text-ink-muted hover:text-ink'}"
				>
					應用程式目錄
				</button>
			</div>
		</div>

		<!-- Row 2: filter slot, fixed height, contents swap per tab. The flex
		     wrapper has a stable min-height so the row below it never moves. -->
		<div class="flex flex-wrap items-center gap-2 mt-4 min-h-[40px]">
			{#if tab === "installed"}
				<div class="relative">
					<Search class="absolute left-3 top-1/2 -translate-y-1/2 text-ink-faint" size={15} />
					<input
						bind:value={searchQuery}
						type="text"
						placeholder="搜尋…"
						class="bg-canvas text-ink rounded-lg pl-9 pr-3 py-2 border border-line-soft focus:outline-none focus:border-brand focus:ring-2 focus:ring-brand/20 text-sm w-52"
					/>
				</div>
				<select
					bind:value={stateFilter}
					class="bg-canvas text-ink border border-line-soft rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-brand focus:ring-2 focus:ring-brand/20"
				>
					<option value="all">所有狀態</option>
					<option value="enabled">已啟用</option>
					<option value="disabled">已停用</option>
					<option value="running">執行中</option>
				</select>
				<select
					bind:value={sourceFilter}
					class="bg-canvas text-ink border border-line-soft rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-brand focus:ring-2 focus:ring-brand/20"
				>
					<option value="all">所有來源</option>
					<option value="system">系統</option>
					<option value="user">使用者</option>
				</select>
				<label
					class="inline-flex items-center gap-2 px-3 py-2 bg-canvas border border-line-soft rounded-lg text-sm text-ink-2 cursor-pointer select-none"
					title="在設定面板中顯示進階設定"
				>
					<input type="checkbox" bind:checked={showAdvanced} class="rounded border-line text-brand focus:ring-brand/30" />
					<span>進階</span>
				</label>
			{:else}
				<div class="relative">
					<Search class="absolute left-3 top-1/2 -translate-y-1/2 text-ink-faint" size={15} />
					<input
						bind:value={catalogSearch}
						type="text"
						placeholder="搜尋應用程式目錄…"
						class="bg-canvas text-ink rounded-lg pl-9 pr-3 py-2 border border-line-soft focus:outline-none focus:border-brand focus:ring-2 focus:ring-brand/20 text-sm w-52"
					/>
				</div>
				<button
					type="button"
					onclick={() => fetchCatalog(true)}
					disabled={catalogLoading}
					class="flex items-center gap-1.5 px-3 py-2 bg-canvas hover:bg-surface-warm border border-line-soft rounded-lg text-sm text-ink-2 disabled:opacity-50 transition-colors"
				>
					{#if catalogLoading}
						<Loader2 size={14} class="animate-spin" />
					{:else}
						<RefreshCw size={14} />
					{/if}
					重新整理
				</button>
			{/if}
		</div>

		<!-- Row 3: info strip, fixed height. Empty on the Installed tab so
		     switching tabs never shifts the grid below. -->
		<div class="mt-3 h-5 text-[11px] text-ink-faint flex flex-wrap items-center gap-3">
			{#if tab === "catalog" && catalog}
				<span class="inline-flex items-center gap-1">
					<Package size={12} /> 發行版本 <span class="font-mono text-ink-muted">{catalog.release || "（未知）"}</span>
				</span>
				<span>印表機： <span class="font-mono text-ink-muted">{catalog.model || "（未知）"}</span></span>
				{#if catalog.asset_group}
					<span>SWU 群組： <span class="font-mono text-ink-muted">{catalog.asset_group}</span></span>
				{/if}
				{#if catalog.fetched_at}
					<span>擷取時間：{relativeTime(catalog.fetched_at)}</span>
				{/if}
				<a
					href="https://github.com/rinkhals-community/Rinkhals.Apps"
					target="_blank"
					rel="noopener noreferrer"
					class="inline-flex items-center gap-1 text-brand hover:underline ml-auto"
				>
					來源 <ExternalLink size={11} />
				</a>
			{/if}
		</div>
	</header>

	{#if !$printerState.can_install}
		<div class="bg-surface-warm border border-accent/40 rounded-xl px-4 py-3 flex items-start gap-3">
			<AlertCircle size={18} class="text-accent shrink-0 mt-0.5" />
			<div class="flex-1 text-sm">
				<p class="font-medium text-ink">印表機忙碌中</p>
				<p class="text-ink-muted mt-0.5">
					{$printerState.reason ?? "列印進行中"}。列印完成前無法安裝或解除安裝，仍可啟動、停止、重新啟動或設定應用程式。
				</p>
			</div>
		</div>
	{/if}

	{#if tab === "installed"}
	{#if error}
		<div class="bg-surface-accent border border-coral/40 rounded-xl px-4 py-3 text-coral text-sm flex items-center gap-2">
			<AlertCircle size={16} />
			無法載入應用程式：{error}
		</div>
	{/if}

	{#if loading && apps.length === 0}
		<div class="text-ink-faint animate-pulse">正在載入應用程式…</div>
	{:else if visibleApps.length === 0}
		<div class="text-ink-faint italic">沒有符合目前篩選條件的應用程式。</div>
	{:else}
		<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
			{#each visibleApps as app (app.id)}
				{@const pill = statusPill(app)}
				{@const isBusy = busyApps.has(app.id)}
				<div class="bg-canvas border border-line-soft rounded-xl p-5 flex flex-col gap-3 hover:border-brand/40 transition-colors">
					<div class="flex items-start justify-between gap-3">
						<div class="min-w-0">
							<div class="flex items-center gap-2 flex-wrap">
								<h3 class="text-base font-semibold text-ink truncate">{app.name}</h3>
								{#if app.version}
									<span class="text-[11px] text-ink-faint font-mono">v{app.version}</span>
								{/if}
								<span class="text-[10px] uppercase tracking-wider px-1.5 py-0.5 rounded {app.source === 'user' ? 'bg-surface-warm text-accent-hover border border-accent/30' : 'bg-surface text-ink-muted border border-line-soft'}">
									{app.source === 'user' ? '使用者' : app.source === 'system' ? '系統' : app.source}
								</span>
							</div>
							<p class="text-[11px] text-ink-faint font-mono mt-0.5">{app.id}</p>
						</div>

						<!-- Enable toggle -->
						<button
							type="button"
							onclick={() => toggleEnabled(app)}
							disabled={isBusy}
							title={app.enabled ? "停用" : "啟用"}
							aria-label="切換啟用狀態"
							class="shrink-0 relative inline-flex h-6 w-11 items-center rounded-full transition-colors disabled:opacity-50 {app.enabled ? 'bg-brand' : 'bg-surface border border-line-soft'}"
						>
							<span class="inline-block h-4 w-4 transform rounded-full bg-white shadow transition-transform {app.enabled ? 'translate-x-6' : 'translate-x-1'}"></span>
						</button>
					</div>

					{#if app.description}
						<div>
							<p
								use:descAction={app.id}
								class="text-sm text-ink-muted {expandedDescs.has(app.id) ? '' : 'line-clamp-2'}"
							>{app.description}</p>
							{#if truncatedDescs.has(app.id) || expandedDescs.has(app.id)}
								<button
									type="button"
									onclick={() => toggleDesc(app.id)}
									class="text-[11px] text-brand hover:underline mt-0.5"
								>
									{expandedDescs.has(app.id) ? '顯示較少' : '顯示更多'}
								</button>
							{/if}
						</div>
					{/if}

					<div class="flex items-center gap-2 flex-wrap">
						<span class="px-2.5 py-1 rounded-full text-[11px] font-semibold uppercase tracking-wider border {pill.classes}">
							{pill.label}
						</span>
						{#if app.requirements?.memory}
							<span class="inline-flex items-center gap-1 text-[11px] text-ink-faint">
								<HardDrive size={11} /> {app.requirements.memory} MB
							</span>
						{/if}
						{#if app.requirements?.cpu}
							<span class="inline-flex items-center gap-1 text-[11px] text-ink-faint">
								<Cpu size={11} /> {app.requirements.cpu}%
							</span>
						{/if}
					</div>

					<div class="flex items-center gap-2 mt-auto pt-3 border-t border-line-soft">
						{#if app.enabled}
							{#if app.running}
								<button
									type="button"
									onclick={() => runAction(app, "stop")}
									disabled={isBusy}
									class="flex items-center gap-1.5 px-2.5 py-1.5 bg-surface hover:bg-surface-warm border border-line-soft rounded-lg text-xs font-medium text-ink-2 disabled:opacity-50 transition-colors"
								>
									<Square size={13} /> 停止
								</button>
								<button
									type="button"
									onclick={() => runAction(app, "restart")}
									disabled={isBusy}
									class="flex items-center gap-1.5 px-2.5 py-1.5 bg-surface hover:bg-surface-warm border border-line-soft rounded-lg text-xs font-medium text-ink-2 disabled:opacity-50 transition-colors"
								>
									<RotateCw size={13} /> 重新啟動
								</button>
							{:else}
								<button
									type="button"
									onclick={() => runAction(app, "start")}
									disabled={isBusy}
									class="flex items-center gap-1.5 px-2.5 py-1.5 bg-brand hover:bg-brand-hover text-white rounded-lg text-xs font-medium disabled:opacity-50 transition-colors"
								>
									<Play size={13} /> 啟動
								</button>
							{/if}
						{/if}
						{#if hasVisibleProperties(app)}
							<button
								type="button"
								onclick={() => openConfigure(app)}
								class="ml-auto flex items-center gap-1.5 px-2.5 py-1.5 bg-canvas hover:bg-brand-soft border border-line-soft rounded-lg text-xs font-medium text-brand transition-colors"
							>
								<Sliders size={13} /> 設定
								{#if visibleProperties(app.properties).some((p) => p.overridden)}
									<span class="ml-1 inline-block w-1.5 h-1.5 rounded-full bg-accent" title="有自訂設定"></span>
								{/if}
							</button>
						{/if}
						{#if app.source === "user"}
							<button
								type="button"
								onclick={() => confirmUninstall(app)}
								disabled={isBusy || !$printerState.can_install}
								title={!$printerState.can_install ? "印表機忙碌時無法執行" : "解除安裝此應用程式"}
								aria-label="解除安裝 {app.name}"
								class="{hasVisibleProperties(app) ? '' : 'ml-auto '}p-1.5 rounded-lg text-ink-faint hover:text-coral hover:bg-surface-accent transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
							>
								<Trash2 size={14} />
							</button>
						{/if}
						{#if isBusy}
							<Loader2 size={14} class="text-ink-faint animate-spin ml-auto" />
						{/if}
					</div>
				</div>
			{/each}
		</div>
	{/if}
	{:else}
		<!-- Catalog tab -->
		{#if catalogError}
			<div class="bg-surface-accent border border-coral/40 rounded-xl px-4 py-3 text-coral text-sm flex items-center gap-2">
				<AlertCircle size={16} />
				無法載入應用程式目錄：{catalogError}
			</div>
		{/if}

		{#if catalog?.notice}
			<div class="bg-surface-warm border border-accent/30 rounded-xl px-4 py-3 text-accent-hover text-sm flex items-center gap-2">
				<AlertCircle size={16} />
				{catalog.notice}
			</div>
		{/if}

		{#if catalogLoading && !catalog}
			<div class="text-ink-faint animate-pulse">正在從 Rinkhals.Apps 載入應用程式目錄…</div>
		{:else if !catalog || visibleCatalog.length === 0}
			<div class="text-ink-faint italic">應用程式目錄中沒有符合條件的項目。</div>
		{:else}
			<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
				{#each visibleCatalog as entry (entry.id)}
					{@const installing = installingId === entry.id}
					{@const upgradeAvailable = entry.installed && entry.installed_version && entry.version && entry.installed_version !== entry.version}
					<div class="bg-canvas border border-line-soft rounded-xl p-5 flex flex-col gap-3 hover:border-brand/40 transition-colors">
						<div class="flex items-start justify-between gap-3">
							<div class="min-w-0">
								<div class="flex items-center gap-2 flex-wrap">
									<h3 class="text-base font-semibold text-ink truncate">{entry.name}</h3>
									{#if entry.version}
										<span class="text-[11px] text-ink-faint font-mono">v{entry.version}</span>
									{/if}
								</div>
								<p class="text-[11px] text-ink-faint font-mono mt-0.5">{entry.id}</p>
							</div>
							{#if entry.installed && !upgradeAvailable}
								<span class="inline-flex items-center gap-1 px-2 py-0.5 rounded-full bg-brand-soft text-brand text-[11px] font-semibold border border-brand/30">
									<CheckCircle2 size={11} />
									已安裝
								</span>
							{:else if upgradeAvailable}
								<span class="inline-flex items-center gap-1 px-2 py-0.5 rounded-full bg-surface-warm text-accent-hover text-[11px] font-semibold border border-accent/30">
									v{entry.installed_version} -&gt; v{entry.version}
								</span>
							{/if}
						</div>

						{#if entry.description}
							<div>
								<p
									use:descAction={entry.id}
									class="text-sm text-ink-muted {expandedDescs.has(entry.id) ? '' : 'line-clamp-3'}"
								>{entry.description}</p>
								{#if truncatedDescs.has(entry.id) || expandedDescs.has(entry.id)}
									<button
										type="button"
										onclick={() => toggleDesc(entry.id)}
										class="text-[11px] text-brand hover:underline mt-0.5"
									>
										{expandedDescs.has(entry.id) ? '顯示較少' : '顯示更多'}
									</button>
								{/if}
							</div>
						{/if}

						<div class="flex items-center gap-3 flex-wrap text-[11px] text-ink-faint">
							{#if entry.requirements?.memory}
								<span class="inline-flex items-center gap-1"><HardDrive size={11} /> {entry.requirements.memory} MB</span>
							{/if}
							{#if entry.requirements?.cpu}
								<span class="inline-flex items-center gap-1"><Cpu size={11} /> {entry.requirements.cpu}%</span>
							{/if}
							{#if entry.asset_size}
								<span class="inline-flex items-center gap-1"><Download size={11} /> {formatSize(entry.asset_size)}</span>
							{/if}
						</div>

						{#if entry.upstream_version}
							{@const stale = entry.upstream_version !== entry.version}
							<div class="flex items-center gap-2 text-[11px] {stale ? 'text-coral' : 'text-ink-faint'}">
								<span class="inline-flex items-center gap-1">
									上游版本：
									{#if entry.upstream_url}
										<a href={entry.upstream_url} target="_blank" rel="noopener noreferrer" class="font-mono hover:underline inline-flex items-center gap-0.5">
											{entry.upstream_version}<ExternalLink size={10} />
										</a>
									{:else}
										<span class="font-mono">{entry.upstream_version}</span>
									{/if}
								</span>
								{#if stale}
									<span class="px-1.5 py-0.5 rounded bg-surface-accent border border-coral/30 uppercase tracking-wider text-[9px] font-semibold" title="套件內附的 v{entry.version} 落後於上游版本">
										內附版本較舊
									</span>
								{/if}
							</div>
						{/if}

						<div class="flex items-center gap-2 mt-auto pt-3 border-t border-line-soft">
							{#if !entry.available_for_model}
								<span class="text-[11px] text-ink-faint italic">這台印表機沒有可用的 SWU</span>
							{:else if entry.installed && !upgradeAvailable}
								<button
									type="button"
									onclick={() => installCatalogApp(entry)}
									disabled={installing || !$printerState.can_install}
									title={!$printerState.can_install ? "印表機忙碌時無法執行" : "重新安裝此應用程式"}
									class="flex items-center gap-1.5 px-2.5 py-1.5 bg-surface hover:bg-surface-warm border border-line-soft rounded-lg text-xs font-medium text-ink-2 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
								>
									{#if installing}
										<Loader2 size={13} class="animate-spin" /> 正在重新安裝…
									{:else}
										<RotateCw size={13} /> 重新安裝
									{/if}
								</button>
							{:else}
								<button
									type="button"
									onclick={() => installCatalogApp(entry)}
									disabled={installing || !$printerState.can_install}
									title={!$printerState.can_install ? "印表機忙碌時無法執行" : (upgradeAvailable ? "升級至最新版" : "安裝此應用程式")}
									class="flex items-center gap-1.5 px-2.5 py-1.5 bg-brand hover:bg-brand-hover text-white rounded-lg text-xs font-medium disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
								>
									{#if installing}
										<Loader2 size={13} class="animate-spin" /> 正在安裝…
									{:else}
										<Download size={13} /> {upgradeAvailable ? "升級" : "安裝"}
									{/if}
								</button>
							{/if}
							{#if entry.download_url}
								<a
									href={entry.download_url}
									target="_blank"
									rel="noopener noreferrer"
									class="ml-auto text-[11px] text-ink-faint hover:text-brand inline-flex items-center gap-1"
									title="直接下載 SWU"
								>
									SWU <ExternalLink size={11} />
								</a>
							{/if}
						</div>
					</div>
				{/each}
			</div>
		{/if}
	{/if}
</div>

<!-- Configure drawer -->
{#if configureApp}
	{@const app = configureApp}
	<!-- svelte-ignore a11y_click_events_have_key_events -->
	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div
		class="fixed inset-0 bg-ink/40 backdrop-blur-sm z-40 flex items-stretch justify-end"
		onclick={closeConfigure}
	>
		<div
			class="bg-canvas border-l border-line w-full max-w-md h-full overflow-y-auto shadow-2xl"
			onclick={(e) => e.stopPropagation()}
		>
			<div class="px-5 py-4 border-b border-line-soft flex items-start justify-between sticky top-0 bg-canvas">
				<div>
					<p class="text-xs uppercase tracking-wider text-ink-faint font-medium">設定</p>
					<h3 class="text-lg font-semibold text-ink mt-0.5">{app.name}</h3>
					<p class="text-[11px] text-ink-faint font-mono">{app.id}</p>
				</div>
				<button
					type="button"
					onclick={closeConfigure}
					class="p-1.5 rounded-lg text-ink-muted hover:text-ink hover:bg-surface transition-colors"
					aria-label="關閉"
				>
					<X size={18} />
				</button>
			</div>

			<div class="p-5 space-y-5">
				{#if !app.enabled}
					<div class="bg-surface-warm border border-accent/30 rounded-lg px-3 py-2.5 text-[13px] text-accent-hover">
						此應用程式已安裝但尚未啟用。請在下方調整設定，再選取<span class="font-medium">「啟用應用程式」</span>來啟動。
					</div>
				{/if}
				{#if visibleProperties(app.properties).length === 0}
					<p class="text-ink-muted text-sm italic">
						{#if app.properties.length > 0}
							此應用程式只有進階設定。請在應用程式頁面勾選<span class="font-medium">「進階」</span>以顯示這些設定。
						{:else}
							此應用程式沒有可調整的設定。
						{/if}
					</p>
				{:else}
					{#each visibleProperties(app.properties) as prop (prop.key)}
						<div class="space-y-1.5">
							<div class="flex items-center justify-between gap-2">
								<label for="prop-{prop.key}" class="text-sm font-medium text-ink">
									{prop.display || prop.key}
								</label>
								{#if prop.overridden || pendingConfig[prop.key] !== undefined}
									<button
										type="button"
										onclick={() => resetProperty(prop)}
										class="text-[11px] text-ink-faint hover:text-brand inline-flex items-center gap-1 transition-colors"
									>
										<RotateCcw size={11} /> 重設
									</button>
								{/if}
							</div>

							{#if prop.type === "enum" && prop.options}
								<select
									id="prop-{prop.key}"
									value={pendingValue(prop)}
									onchange={(e) => {
										const v = (e.target as HTMLSelectElement).value;
										if (v === prop.value) {
											const fresh = { ...pendingConfig };
											delete fresh[prop.key];
											pendingConfig = fresh;
										} else {
											pendingConfig = { ...pendingConfig, [prop.key]: v };
										}
									}}
									class="w-full bg-canvas text-ink border border-line rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-brand/30 focus:border-brand"
								>
									{#each prop.options as opt}
										<option value={opt}>{opt}</option>
									{/each}
								</select>
							{:else if prop.type === "report"}
								<!-- A "report" property is read-only structured output written by
								     the app's daemon (e.g. Tailscale publishes its connection state
								     here). The value is a JSON-encoded string; we parse it and let
								     the renderer pick what to show. -->
								<ReportField value={prop.value} appId={app.id} />
							{:else if prop.type === "qr"}
								<!-- A "qr" property is read-only output written by the app's daemon
								     (e.g. Tailscale login URL, OctoEverywhere link). We render the
								     URL as a scannable QR code with the raw URL as a fallback. -->
								{#if prop.value}
									<div class="bg-surface border border-line-soft rounded-xl p-4 flex flex-col items-center gap-3">
										<canvas use:qrCanvas={prop.value} class="rounded-md bg-white"></canvas>
										<p class="text-[11px] text-ink-muted text-center">請用手機掃描，或開啟下方網址。</p>
										<div class="w-full flex items-stretch gap-1.5">
											<a
												href={prop.value}
												target="_blank"
												rel="noopener noreferrer"
												class="flex-1 min-w-0 px-2.5 py-1.5 bg-canvas border border-line-soft rounded-md text-[11px] font-mono text-brand hover:bg-brand-soft transition-colors truncate"
												title={prop.value}
											>
												{prop.value}
											</a>
											<button
												type="button"
												onclick={() => copyToClipboard(prop.key, prop.value)}
												class="shrink-0 px-2 py-1.5 bg-canvas hover:bg-surface-warm border border-line-soft rounded-md text-ink-muted hover:text-ink transition-colors"
												title="複製網址"
												aria-label="複製網址"
											>
												{#if copiedKey === prop.key}
													<Check size={13} class="text-brand" />
												{:else}
													<Copy size={13} />
												{/if}
											</button>
										</div>
										<p class="text-[11px] text-ink-faint text-center">
											應用程式提供的網址可能會過期。若掃描失敗，請從應用程式卡片重新啟動，以取得新的 QR 碼。
										</p>
									</div>
								{:else if app.running}
									<div class="bg-surface border border-line-soft rounded-xl p-4 text-center">
										<Loader2 size={22} class="text-brand animate-spin mx-auto mb-2" />
										<p class="text-sm text-ink">正在等待應用程式提供登入網址…</p>
										<p class="text-[11px] text-ink-faint mt-1">通常在啟動後幾秒內就會提供。</p>
									</div>
								{:else}
									<div class="bg-surface-warm border border-accent/30 rounded-xl p-4 text-center text-ink-2">
										<p class="text-sm">請啟動應用程式以取得登入網址。</p>
										<p class="text-[11px] text-ink-muted mt-1">背景服務須在執行中才能產生登入網址。</p>
									</div>
								{/if}
							{:else}
								<input
									id="prop-{prop.key}"
									type="text"
									value={pendingValue(prop)}
									oninput={(e) => {
										const v = (e.target as HTMLInputElement).value;
										if (v === prop.value) {
											const fresh = { ...pendingConfig };
											delete fresh[prop.key];
											pendingConfig = fresh;
										} else {
											pendingConfig = { ...pendingConfig, [prop.key]: v };
										}
									}}
									class="w-full bg-canvas text-ink border border-line rounded-lg px-3 py-2 text-sm font-mono focus:outline-none focus:ring-2 focus:ring-brand/30 focus:border-brand"
								/>
							{/if}

							<p class="text-[11px] text-ink-faint">
								預設值：<span class="font-mono">{prop.default || "—"}</span>
								{#if prop.overridden}
									&middot; <span class="text-accent-hover">正在使用自訂值</span>
								{/if}
							</p>
						</div>
					{/each}
				{/if}
				{#if visibleProperties(app.properties).length > 0 && app.properties.length > visibleProperties(app.properties).length}
					<p class="text-[11px] text-ink-faint italic pt-2 border-t border-line-soft">
						有 {app.properties.length - visibleProperties(app.properties).length} 項進階設定未顯示。請在應用程式頁面勾選<span class="font-medium">「進階」</span>以顯示這些設定。
					</p>
				{/if}
			</div>

			<div class="px-5 py-4 border-t border-line-soft flex items-center justify-between gap-2 sticky bottom-0 bg-canvas">
				<button
					type="button"
					onclick={clearAllOverrides}
					disabled={!app.properties.some((p) => p.overridden)}
					class="text-xs text-coral hover:underline disabled:opacity-40 disabled:no-underline"
				>
					清除所有自訂設定
				</button>
				<div class="flex items-center gap-2">
					<button
						type="button"
						onclick={closeConfigure}
						class="px-3 py-1.5 rounded-lg text-sm font-medium bg-surface hover:bg-surface-warm border border-line-soft text-ink"
					>
						關閉
					</button>
					<button
						type="button"
						onclick={saveConfig}
						disabled={!hasPendingChanges() || configSaving}
						class="px-3 py-1.5 rounded-lg text-sm font-medium disabled:opacity-40 flex items-center gap-1.5 {app.enabled ? 'bg-brand hover:bg-brand-hover text-white' : 'bg-surface hover:bg-surface-warm border border-line-soft text-ink'}"
					>
						{#if configSaving}
							<Loader2 size={13} class="animate-spin" />
						{/if}
						儲存
					</button>
					{#if !app.enabled}
						<button
							type="button"
							onclick={() => saveAndEnable(app)}
							disabled={configSaving}
							class="px-3 py-1.5 rounded-lg text-sm font-medium bg-brand hover:bg-brand-hover text-white disabled:opacity-40 flex items-center gap-1.5"
						>
							<Play size={13} />
							啟用應用程式
						</button>
					{/if}
				</div>
			</div>
		</div>
	</div>
{/if}

<!-- Toast -->
<!-- Uninstall confirmation -->
{#if uninstallTarget}
	<!-- svelte-ignore a11y_click_events_have_key_events -->
	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div
		class="fixed inset-0 bg-ink/40 backdrop-blur-sm flex items-center justify-center z-50 p-4"
		onclick={() => !uninstallRunning && (uninstallTarget = null)}
	>
		<div
			class="bg-canvas border border-line rounded-2xl p-6 max-w-md w-full shadow-2xl"
			onclick={(e) => e.stopPropagation()}
		>
			<div class="flex items-center text-coral mb-3">
				<AlertTriangle size={22} class="mr-2" />
				<h2 class="text-lg font-semibold text-ink">要解除安裝 {uninstallTarget.name} 嗎？</h2>
			</div>
			<p class="text-ink-2 text-sm mb-2">
				系統會先停止此應用程式，再移除應用程式及其所有已儲存的設定。
			</p>
			<p class="text-ink-muted text-xs mb-5">
				<span class="font-mono">{uninstallTarget.id}</span> &middot; user app
			</p>
			<div class="flex justify-end gap-2">
				<button
					type="button"
					onclick={() => (uninstallTarget = null)}
					disabled={uninstallRunning}
					class="px-4 py-2 rounded-lg font-medium bg-surface hover:bg-surface-warm border border-line-soft text-ink disabled:opacity-50"
				>
					取消
				</button>
				<button
					type="button"
					onclick={performUninstall}
					disabled={uninstallRunning || !$printerState.can_install}
					title={!$printerState.can_install ? "印表機忙碌時無法執行" : ""}
					class="px-4 py-2 rounded-lg font-medium bg-coral hover:opacity-90 text-white flex items-center gap-2 disabled:opacity-50 disabled:cursor-not-allowed"
				>
					{#if uninstallRunning}
						<Loader2 size={14} class="animate-spin" />
					{:else}
						<Trash2 size={14} />
					{/if}
					解除安裝
				</button>
			</div>
		</div>
	</div>
{/if}

{#if toast}
	<div class="fixed bottom-5 right-5 z-50">
		<div class="px-4 py-2.5 rounded-lg shadow-lg border text-sm font-medium {toast.tone === 'err' ? 'bg-surface-accent text-coral border-coral/40' : 'bg-canvas text-ink border-line-soft'}">
			{toast.text}
		</div>
	</div>
{/if}
