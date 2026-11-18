<script lang="ts">
	import {
		CheckCircle2,
		XCircle,
		AlertCircle,
		Loader2,
		Copy,
		Check,
		Globe,
		Server,
		ShieldCheck,
		Network,
		Users
	} from "lucide-svelte";

	type Props = { value: string; appId: string };
	let { value, appId }: Props = $props();

	let parsed = $derived.by<Record<string, any> | null>(() => {
		if (!value) return null;
		try {
			return JSON.parse(value);
		} catch {
			return null;
		}
	});

	let copied = $state<string | null>(null);
	async function copy(key: string, text: string) {
		try {
			await navigator.clipboard.writeText(text);
			copied = key;
			setTimeout(() => {
				if (copied === key) copied = null;
			}, 1500);
		} catch {}
	}

	// Tailscale-known states, mapped to a (tone, label, icon) tuple. Unknown
	// values fall through to a neutral grey.
	const tsStateTone: Record<string, { tone: string; label: string }> = {
		Running: { tone: "ok", label: "已連線" },
		Starting: { tone: "warn", label: "正在啟動" },
		NeedsLogin: { tone: "warn", label: "需要登入" },
		NoState: { tone: "warn", label: "正在初始化" },
		Stopped: { tone: "muted", label: "已停止" }
	};
</script>

{#if !parsed}
	<div class="bg-surface border border-line-soft rounded-xl p-3 text-center text-ink-faint text-sm">
		<Loader2 size={18} class="inline-block animate-spin mr-1.5 align-middle" />
		正在等待狀態資訊…
	</div>
{:else if appId === "tailscale"}
	{@const state = parsed.state ?? "Unknown"}
	{@const stateMeta = tsStateTone[state] ?? { tone: "muted", label: state === "Unknown" ? "未知" : state }}
	{@const isRunning = state === "Running"}
	<div class="bg-surface border border-line-soft rounded-xl divide-y divide-line-soft">
		<!-- Connection state -->
		<div class="px-4 py-3 flex items-center justify-between gap-3">
			<div class="flex items-center gap-2">
				{#if isRunning}
					<CheckCircle2 size={18} class="text-brand" />
				{:else if stateMeta.tone === "warn"}
					<AlertCircle size={18} class="text-accent" />
				{:else}
					<XCircle size={18} class="text-ink-faint" />
				{/if}
				<span class="font-medium text-ink">{stateMeta.label}</span>
			</div>
			{#if parsed.tailnet_name}
				<span class="text-[11px] text-ink-faint truncate" title={parsed.tailnet_name}>
					{parsed.tailnet_name}
				</span>
			{/if}
		</div>

		<!-- Identity / address rows. We only render rows whose values are present
		     so a not-yet-connected node doesn't show a stack of empty fields. -->
		{#if parsed.tailnet_ip}
			<div class="px-4 py-2.5 flex items-center gap-2 text-sm">
				<Globe size={15} class="text-ink-faint shrink-0" />
				<span class="text-ink-muted shrink-0">Tailnet IP 位址</span>
				<button
					type="button"
					onclick={() => copy("ip", parsed.tailnet_ip)}
					class="ml-auto inline-flex items-center gap-1 font-mono text-ink hover:text-brand transition-colors"
					title="複製"
				>
					{parsed.tailnet_ip}
					{#if copied === "ip"}
						<Check size={12} class="text-brand" />
					{:else}
						<Copy size={12} class="opacity-60" />
					{/if}
				</button>
			</div>
		{/if}
		{#if parsed.magic_dns}
			<div class="px-4 py-2.5 flex items-center gap-2 text-sm">
				<Network size={15} class="text-ink-faint shrink-0" />
				<span class="text-ink-muted shrink-0">MagicDNS</span>
				<button
					type="button"
					onclick={() => copy("dns", parsed.magic_dns)}
					class="ml-auto inline-flex items-center gap-1 font-mono text-ink hover:text-brand transition-colors truncate"
					title={parsed.magic_dns}
				>
					<span class="truncate">{parsed.magic_dns}</span>
					{#if copied === "dns"}
						<Check size={12} class="text-brand shrink-0" />
					{:else}
						<Copy size={12} class="opacity-60 shrink-0" />
					{/if}
				</button>
			</div>
		{/if}
		{#if parsed.hostname}
			<div class="px-4 py-2.5 flex items-center gap-2 text-sm">
				<Server size={15} class="text-ink-faint shrink-0" />
				<span class="text-ink-muted shrink-0">主機名稱</span>
				<span class="ml-auto font-mono text-ink truncate" title={parsed.hostname}>
					{parsed.hostname}
				</span>
			</div>
		{/if}

		<!-- Flags. SSH, exit-node, peers - keep it brief; toggles for these live
		     in the editable property rows below. -->
		<div class="px-4 py-2.5 flex flex-wrap items-center gap-3 text-[12px]">
			<span class="inline-flex items-center gap-1.5 {parsed.ssh ? 'text-brand' : 'text-ink-faint'}">
				<ShieldCheck size={13} />
				SSH：{parsed.ssh ? "開啟" : "關閉"}
			</span>
			<span class="inline-flex items-center gap-1.5 {parsed.exit_node ? 'text-accent-hover' : 'text-ink-faint'}">
				<Globe size={13} />
				出口節點：{parsed.exit_node ? "已宣告" : "關閉"}
			</span>
			{#if isRunning}
				<span class="inline-flex items-center gap-1.5 text-ink-faint ml-auto">
					<Users size={13} /> {parsed.peer_count ?? 0} 個對等節點
				</span>
			{/if}
		</div>
	</div>
{:else}
	<!-- Generic fallback: show the parsed object as a compact key/value list.
	     Apps that publish "report" data without a custom renderer here still
	     get something useful in the drawer. -->
	<div class="bg-surface border border-line-soft rounded-xl divide-y divide-line-soft">
		{#each Object.entries(parsed) as [k, v]}
			<div class="px-4 py-2 flex items-center gap-3 text-sm">
				<span class="text-ink-muted shrink-0">{k}</span>
				<span class="ml-auto font-mono text-ink truncate" title={String(v)}>{String(v)}</span>
			</div>
		{/each}
	</div>
{/if}
