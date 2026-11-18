<script lang="ts">
    import { RefreshCw, Download, Server, Trash2, AlertTriangle, ShieldAlert, Terminal, Lock, Cpu, CheckCircle2, ArrowUpCircle } from 'lucide-svelte';
    import { firmwareStatus } from "$lib/firmwareStatus";
    import PatchesCard from "$lib/PatchesCard.svelte";

    let loadingAction = $state<string | null>(null);
    let logs = $state<string>('');
    let showConfirmDialog = $state<boolean>(false);
    let actionToConfirm = $state<{id: string, name: string, description: string, icon: any} | null>(null);

    let isChangingPassword = $state(false);
    let authSaving = $state(false);
    let p_username = $state('admin');
    let p_current = $state('');
    let p_new = $state('');
    let p_confirm = $state('');
    let p_error = $state('');
    let p_success = $state('');

    const tools = [
        {
            id: 'debug-bundle',
            name: '產生除錯套件',
            description: '將系統記錄與設定收集成可下載的 ZIP 檔',
            icon: Download,
            danger: false,
            action: async () => {
                loadingAction = 'debug-bundle';
                try {
                    const res = await fetch('/api/tools?action=debug-bundle', { method: 'POST' });
                    if (res.ok) {
                        const blob = await res.blob();
                        const url = window.URL.createObjectURL(blob);
                        const a = document.createElement('a');
                        a.href = url;
                        a.download = 'debug-bundle.zip';
                        document.body.appendChild(a);
                        a.click();
                        a.remove();
                        window.URL.revokeObjectURL(url);
                        logs = '已產生並下載除錯套件。\n' + logs;
                    } else {
                        logs = `無法產生除錯套件： ${res.statusText}\n` + logs;
                    }
                } catch (e: any) {
                    logs = `錯誤：${e.message}\n` + logs;
                } finally {
                    loadingAction = null;
                }
            }
        },
        { id: 'backup-partitions', name: '備份磁碟分割區', description: '備份重要的系統分割區', icon: Server, danger: false },
        { id: 'config-reset', name: '重設 Rinkhals 設定', description: '將設定檔還原為預設值（會重新啟動服務）', icon: RefreshCw, danger: true },
        { id: 'clean-rinkhals', name: '清理舊版 Rinkhals', description: '移除先前安裝 Rinkhals 遺留的檔案', icon: Trash2, danger: true },
        { id: 'uninstall-rinkhals', name: '解除安裝 Rinkhals', description: '完全移除 Rinkhals，並重新啟動至原廠韌體', icon: ShieldAlert, danger: true }
    ];

    async function executeTool(id: string) {
        loadingAction = id;
        logs = `正在執行 ${id}…\n` + logs;
        try {
            const res = await fetch(`/api/tools?action=${id}`, { method: 'POST' });
            const data = await res.json();
            if (data.success) {
                logs = `執行成功（${id}）：\n${data.output}\n` + logs;
            } else {
                logs = `執行失敗（${id}）：\n${data.output || '未知錯誤'}\n` + logs;
            }
        } catch (e: any) {
            logs = `發生錯誤（${id}）： ${e.message}\n` + logs;
        } finally {
            loadingAction = null;
        }
    }

    function confirmAction(tool: any) {
        if (tool.danger) {
            actionToConfirm = tool;
            showConfirmDialog = true;
        } else {
            if (tool.action) {
                tool.action();
            } else {
                executeTool(tool.id);
            }
        }
    }

    function executeConfirmed() {
        if (actionToConfirm) {
            if (actionToConfirm.action) {
                actionToConfirm.action();
            } else {
                executeTool(actionToConfirm.id);
            }
            showConfirmDialog = false;
            actionToConfirm = null;
        }
    }

    async function changePassword() {
        p_error = "";
        p_success = "";
        if (!p_current) {
            p_error = "請輸入目前的密碼以確認變更。";
            return;
        }
        if (p_new.length < 5) {
            p_error = "新密碼至少須有 5 個字元。";
            return;
        }
        if (p_new !== p_confirm) {
            p_error = "兩次輸入的新密碼不一致。";
            return;
        }
        authSaving = true;
        try {
            const creds = btoa(`${p_username}:${p_current}`);
            const host = import.meta.env.DEV ? "http://localhost:8090" : "";
            const testRes = await fetch(`${host}/api/auth/status`, {
                headers: { 'Authorization': `Basic ${creds}` }
            });

            if (!testRes.ok) {
                p_error = "目前的使用者名稱或密碼不正確。";
                authSaving = false;
                return;
            }

            const res = await fetch(`${host}/api/auth/change`, {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                    'Authorization': `Basic ${creds}`
                },
                body: JSON.stringify({ username: p_username, password: p_new })
            });

            if (res.ok) {
                p_success = "密碼已更新。請使用新的帳號密碼重新登入。頁面將在 3 秒後重新整理…";
                setTimeout(() => { window.location.reload(); }, 3000);
            } else {
                p_error = "無法更新密碼。";
            }
        } catch (e) {
            p_error = "更新帳號密碼時發生網路或伺服器錯誤。";
        }
        authSaving = false;
    }
</script>

<svelte:head>
    <title>系統管理 - Rinkhals</title>
</svelte:head>

<div class="space-y-6 h-full flex flex-col max-w-7xl mx-auto w-full">
    <header class="pb-4 border-b border-line-soft">
        <p class="text-xs uppercase tracking-wider text-ink-faint font-medium">管理</p>
        <h2 class="text-3xl font-semibold text-ink mt-1 tracking-tight">系統管理</h2>
        <p class="text-ink-muted text-sm mt-2">執行維護工作並設定印表機控制台。</p>
    </header>

    <!-- Firmware status card. Read-only summary that links to the Firmware page
         for the full catalog browse + (eventually) install flow. -->
    <section class="bg-canvas rounded-xl border border-line-soft p-5">
        <div class="flex items-start gap-3 mb-4">
            <div class="w-11 h-11 rounded-lg bg-brand-soft text-brand flex items-center justify-center shrink-0">
                <Cpu size={20} />
            </div>
            <div class="flex-1 min-w-0">
                <h3 class="text-base font-semibold text-ink">韌體狀態</h3>
                <p class="text-ink-muted text-sm mt-1">
                    {$firmwareStatus.model_name || "這台印表機"}目前安裝的版本。
                </p>
            </div>
            <a
                href="/firmware"
                class="inline-flex items-center gap-1.5 px-3 py-1.5 text-xs font-medium rounded-md border border-line-soft text-ink-muted hover:bg-surface-warm hover:text-ink transition-colors shrink-0"
            >
                開啟韌體頁面
            </a>
        </div>

        <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div class="rounded-lg border border-line-soft bg-surface px-4 py-3">
                <div class="text-xs uppercase tracking-wider text-ink-faint mb-1">Anycubic 韌體</div>
                <div class="text-lg font-semibold text-ink">{$firmwareStatus.current_firmware || "未知"}</div>
                {#if $firmwareStatus.latest_firmware}
                    <div class="mt-1 text-xs text-ink-muted flex items-center gap-1.5 flex-wrap">
                        <span>最新版： <span class="text-ink">{$firmwareStatus.latest_firmware}</span></span>
                        {#if $firmwareStatus.firmware_update_available}
                            <span class="inline-flex items-center gap-1 text-brand">
                                <ArrowUpCircle size={12} /> 有可用的更新
                            </span>
                        {:else}
                            <span class="inline-flex items-center gap-1 text-emerald">
                                <CheckCircle2 size={12} /> 已是最新版
                            </span>
                        {/if}
                    </div>
                {/if}
            </div>
            <div class="rounded-lg border border-line-soft bg-surface px-4 py-3">
                <div class="text-xs uppercase tracking-wider text-ink-faint mb-1">Rinkhals</div>
                <div class="text-lg font-semibold text-ink">{$firmwareStatus.current_rinkhals || "未知"}</div>
                {#if $firmwareStatus.latest_rinkhals}
                    <div class="mt-1 text-xs text-ink-muted flex items-center gap-1.5 flex-wrap">
                        <span>最新版： <span class="text-ink">{$firmwareStatus.latest_rinkhals}</span></span>
                        {#if $firmwareStatus.rinkhals_update_available}
                            <span class="inline-flex items-center gap-1 text-brand">
                                <ArrowUpCircle size={12} /> 有可用的更新
                            </span>
                        {:else}
                            <span class="inline-flex items-center gap-1 text-emerald">
                                <CheckCircle2 size={12} /> 已是最新版
                            </span>
                        {/if}
                    </div>
                {/if}
            </div>
        </div>
    </section>

    <PatchesCard />

    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
        {#each tools as tool}
            {@const Icon = tool.icon}
            <div class="bg-canvas rounded-xl p-5 border border-line-soft flex flex-col justify-between hover:border-brand/40 transition-colors">
                <div class="flex items-start mb-4">
                    <div class="w-11 h-11 rounded-lg flex items-center justify-center mr-3 shrink-0 {tool.danger ? 'bg-surface-accent text-coral' : 'bg-brand-soft text-brand'}">
                        <Icon size={20} />
                    </div>
                    <div>
                        <h3 class="font-semibold text-ink text-base">{tool.name}</h3>
                        <p class="text-ink-muted text-sm mt-1">{tool.description}</p>
                    </div>
                </div>
                <button
                    class="ml-auto w-full mt-2 py-2 px-4 rounded-lg font-medium transition-colors flex justify-center items-center
                    {tool.danger
                        ? 'bg-coral hover:opacity-90 text-white'
                        : 'bg-brand hover:bg-brand-hover text-white'}
                    {loadingAction ? 'opacity-50 cursor-not-allowed' : ''}"
                    disabled={loadingAction !== null}
                    onclick={() => confirmAction(tool)}
                >
                    {#if loadingAction === tool.id}
                        <div class="w-4 h-4 border-2 border-white/30 border-t-white rounded-full animate-spin mr-2"></div>
                        正在執行…
                    {:else}
                        執行
                    {/if}
                </button>
            </div>
        {/each}

        <!-- Password Change Card -->
        <div class="bg-canvas rounded-xl p-5 border border-line-soft flex flex-col justify-between hover:border-brand/40 transition-colors">
            <div>
                <div class="flex items-start mb-4">
                    <div class="w-11 h-11 rounded-lg bg-surface-warm text-accent flex items-center justify-center mr-3 shrink-0">
                        <Lock size={20} />
                    </div>
                    <div>
                        <h3 class="text-base font-semibold text-ink">網頁登入驗證</h3>
                        <p class="text-ink-muted text-sm mt-1">變更登入此控制台使用的帳號密碼。</p>
                    </div>
                </div>
            </div>
            <button
                class="w-full mt-2 px-4 py-2 rounded-lg font-medium transition-colors bg-surface hover:bg-surface-warm border border-line-soft text-ink"
                onclick={() => isChangingPassword = true}
            >
                變更帳號密碼
            </button>
        </div>
    </div>

    <!-- Output Logs -->
    <div class="flex-1 mt-2 bg-canvas rounded-xl border border-line-soft flex flex-col min-h-64 overflow-hidden">
        <div class="px-4 py-2 border-b border-line-soft flex items-center justify-between bg-surface">
            <h3 class="font-medium text-sm text-ink-muted flex items-center">
                <Terminal size={15} class="mr-2 text-brand" /> 操作輸出
            </h3>
        </div>
        <div class="p-4 flex-1 overflow-y-auto">
            <pre class="font-mono text-[13px] text-ink-2 whitespace-pre-wrap">{logs || '尚未執行任何操作。'}</pre>
        </div>
    </div>
</div>

{#if showConfirmDialog}
    <div class="fixed inset-0 bg-ink/40 backdrop-blur-sm flex items-center justify-center z-50 p-4">
        <div class="bg-canvas border border-line rounded-2xl p-6 max-w-md w-full shadow-2xl">
            <div class="flex items-center text-coral mb-3">
                <AlertTriangle size={26} class="mr-2" />
                <h2 class="text-lg font-semibold text-ink">高風險操作</h2>
            </div>

            <p class="text-ink-2 text-sm mb-2">
                確定要執行「<strong class="text-ink">{actionToConfirm?.name}</strong>」嗎？
            </p>
            <p class="text-sm text-ink-muted mb-5">
                {actionToConfirm?.description}。此操作難以復原。
            </p>

            <div class="flex justify-end gap-2">
                <button
                    class="px-4 py-2 rounded-lg font-medium bg-surface hover:bg-surface-warm border border-line-soft text-ink transition-colors"
                    onclick={() => { showConfirmDialog = false; actionToConfirm = null; }}
                >
                    取消
                </button>
                <button
                    class="px-4 py-2 rounded-lg font-medium bg-coral hover:opacity-90 text-white shadow-sm transition-colors"
                    onclick={executeConfirmed}
                >
                    確認執行
                </button>
            </div>
        </div>
    </div>
{/if}

{#if isChangingPassword}
    <div class="fixed inset-0 bg-ink/40 backdrop-blur-sm flex items-center justify-center z-50 p-4">
        <div class="bg-canvas border border-line rounded-2xl max-w-md w-full p-6 shadow-2xl">
            <h2 class="text-lg font-semibold text-ink mb-2 flex items-center gap-2">
                <Lock class="text-accent" size={20} /> 變更帳號密碼
            </h2>
            <p class="text-ink-muted text-sm mb-5 pb-4 border-b border-line-soft">
                更新登入此網頁介面所需的帳號密碼。
            </p>

            <form onsubmit={(e) => { e.preventDefault(); changePassword(); }} class="space-y-4">
                <div>
                    <label class="block text-xs font-medium text-ink-muted mb-1" for="p_user">使用者名稱</label>
                    <input id="p_user" type="text" bind:value={p_username} class="w-full bg-canvas border border-line rounded-lg px-3 py-2 text-ink focus:outline-none focus:ring-2 focus:ring-brand/30 focus:border-brand" required />
                </div>
                <div>
                    <label class="block text-xs font-medium text-ink-muted mb-1" for="p_curr">目前的密碼</label>
                    <input id="p_curr" type="password" bind:value={p_current} class="w-full bg-canvas border border-line rounded-lg px-3 py-2 text-ink focus:outline-none focus:ring-2 focus:ring-brand/30 focus:border-brand" required />
                </div>
                <div class="pt-2">
                    <label class="block text-xs font-medium text-ink-muted mb-1" for="p_new">新密碼</label>
                    <input id="p_new" type="password" bind:value={p_new} class="w-full bg-canvas border border-line rounded-lg px-3 py-2 text-ink focus:outline-none focus:ring-2 focus:ring-brand/30 focus:border-brand" required />
                </div>
                <div>
                    <label class="block text-xs font-medium text-ink-muted mb-1" for="p_conf">確認新密碼</label>
                    <input id="p_conf" type="password" bind:value={p_confirm} class="w-full bg-canvas border border-line rounded-lg px-3 py-2 text-ink focus:outline-none focus:ring-2 focus:ring-brand/30 focus:border-brand" required />
                </div>

                {#if p_error}
                    <p class="text-coral text-sm bg-surface-accent p-3 rounded-lg">{p_error}</p>
                {/if}
                {#if p_success}
                    <p class="text-brand text-sm bg-brand-soft p-3 rounded-lg">{p_success}</p>
                {/if}

                <div class="flex justify-end gap-2 pt-2">
                    <button
                        type="button"
                        class="px-4 py-2 rounded-lg font-medium bg-surface hover:bg-surface-warm border border-line-soft text-ink transition-colors"
                        onclick={() => { isChangingPassword = false; p_error = ''; p_success = ''; p_current = ''; p_new = ''; p_confirm = ''; }}
                        disabled={authSaving || p_success !== ''}
                    >
                        取消
                    </button>
                    <button
                        type="submit"
                        class="px-4 py-2 rounded-lg font-medium bg-brand hover:bg-brand-hover text-white shadow-sm transition-colors flex items-center justify-center disabled:opacity-50 {authSaving ? 'animate-pulse' : ''}"
                        disabled={authSaving || p_success !== ''}
                    >
                        {authSaving ? "正在儲存…" : "儲存帳號密碼"}
                    </button>
                </div>
            </form>
        </div>
    </div>
{/if}
