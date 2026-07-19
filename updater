// ============================================
// WORKER UPDATER - v4.0
// Downloads complete code from GitHub and deploys to target worker
// ============================================

// ============================================
// ===== VERSION COMPARE =====
// ============================================
function cmpVersions(a, b) {
    const strip = (v) => String(v).replace(/^v/, "").trim();
    const pa = strip(a).split(".").map(Number);
    const pb = strip(b).split(".").map(Number);
    for (let i = 0; i < Math.max(pa.length, pb.length); i++) {
        let na = pa[i] || 0;
        let nb = pb[i] || 0;
        if (na > nb) return 1;
        if (nb > na) return -1;
    }
    return 0;
}

// ============================================
// ===== DEPLOY TO CLOUDFLARE =====
// ============================================
async function deployWorkerToCloudflare(accountId, apiToken, workerName, code) {
    // Get current bindings to preserve them
    let currentBindings = [];
    let currentCompatibilityDate = "2024-03-01";
    let currentCompatibilityFlags = ["allow_eval_during_startup"];
    
    try {
        // Get current worker settings
        const settingsRes = await fetch(
            `https://api.cloudflare.com/client/v4/accounts/${accountId}/workers/scripts/${encodeURIComponent(workerName)}/settings`,
            { headers: { Authorization: `Bearer ${apiToken}` } }
        );
        const settingsJson = await settingsRes.json();
        if (settingsJson.success && settingsJson.result) {
            if (settingsJson.result.bindings) {
                currentBindings = settingsJson.result.bindings;
            }
            if (settingsJson.result.compatibility_date) {
                currentCompatibilityDate = settingsJson.result.compatibility_date;
            }
            if (settingsJson.result.compatibility_flags) {
                currentCompatibilityFlags = settingsJson.result.compatibility_flags;
            }
        }
    } catch (e) {
        // If we can't get settings, use defaults
    }

    const metadata = {
        main_module: "_worker.js",
        compatibility_date: currentCompatibilityDate,
        compatibility_flags: currentCompatibilityFlags,
        bindings: currentBindings,
    };

    const form = new FormData();
    form.append(
        "metadata",
        new Blob([JSON.stringify(metadata)], { type: "application/json" })
    );
    form.append(
        "_worker.js",
        new Blob([code], { type: "application/javascript+module" }),
        "_worker.js"
    );

    return await fetch(
        `https://api.cloudflare.com/client/v4/accounts/${accountId}/workers/scripts/${encodeURIComponent(workerName)}`,
        {
            method: "PUT",
            headers: { Authorization: `Bearer ${apiToken}` },
            body: form,
        }
    );
}

// ============================================
// ===== FETCH COMPLETE CODE FROM GITHUB =====
// ============================================
async function fetchCompleteCodeFromGitHub(repo = "mahbodrahimi/nahan", branch = "main") {
    try {
        // 1. Fetch the main worker code
        const codeUrl = `https://raw.githubusercontent.com/mahbodrahimi/nahan/refs/heads/main/_worker.js`;
        const codeRes = await fetch(codeUrl);
        if (!codeRes.ok) throw new Error(`Failed to fetch _worker.js: HTTP ${codeRes.status}`);
        const code = await codeRes.text();
        
        if (!code || code.length < 100) {
            throw new Error("Invalid code received (too short)");
        }

        // 2. Fetch version
        let version = "unknown";
        try {
            const versionUrl = `https://raw.githubusercontent.com/mahbodrahimi/nahan/refs/heads/main/version`;
            const versionRes = await fetch(versionUrl);
            if (versionRes.ok) {
                version = (await versionRes.text()).trim();
            }
        } catch (e) {
            console.warn("Could not fetch version:", e.message);
        }

        // 3. Fetch changelog (what's new)
        let whatsNew = "";
        try {
            const wnUrl = `https://raw.githubusercontent.com/mahbodrahimi/nahan/refs/heads/main/whatnew`;
            const wnRes = await fetch(wnUrl);
            if (wnRes.ok) {
                whatsNew = await wnRes.text();
            }
        } catch (e) {
            console.warn("Could not fetch changelog:", e.message);
        }

        // 4. Check if code is obfuscated or contains imports
        const hasImports = code.includes("import ") || code.includes("export ");
        const isObfuscated = code.includes("_0x") || code.includes("xor") || code.includes("decode");

        return {
            success: true,
            code: code,
            version: version,
            whatsNew: whatsNew,
            hasImports: hasImports,
            isObfuscated: isObfuscated,
            size: code.length,
            lines: code.split('\n').length
        };
    } catch (e) {
        return {
            success: false,
            error: e.message
        };
    }
}

// ============================================
// ===== FETCH REMOTE CLEAN IPS =====
// ============================================
const CLEAN_IPS_URL = "https://mahbodrahimi.ir/Nahan/iplist.txt";

async function fetchRemoteCleanIps() {
    try {
        const response = await fetch(CLEAN_IPS_URL, {
            headers: { 'Cache-Control': 'no-cache' },
            signal: AbortSignal.timeout(10000)
        });
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const text = await response.text();
        const ips = text
            .split(/[\r\n,;]+/)
            .map(line => line.trim())
            .filter(line => line && !line.startsWith('#'))
            .join('\n');
        return ips || null;
    } catch (e) {
        console.error('Failed to fetch remote clean IPs:', e.message);
        return null;
    }
}

// ============================================
// ===== GET REPO INFO =====
// ============================================
async function getRepoInfo(repo = "mahbodrahimi/nahan", branch = "main") {
    try {
        const apiUrl = `https://api.github.com/repos/${repo}/branches/${branch}`;
        const res = await fetch(apiUrl, {
            headers: { 'User-Agent': 'Cloudflare-Worker-Updater' }
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        return {
            success: true,
            branch: branch,
            commit: data.commit?.sha || "unknown",
            commitUrl: data.commit?.url || null,
            lastUpdate: data.commit?.commit?.author?.date || null
        };
    } catch (e) {
        return {
            success: false,
            error: e.message
        };
    }
}

// ============================================
// ===== MAIN UPDATE HANDLER =====
// ============================================
async function handleUpdate(request, env, ctx) {
    try {
        // ===== GET DATA FROM REQUEST =====
        let masterKey = null;
        let action = "check";
        let cfAccountId = null;
        let cfApiToken = null;
        let cfWorkerName = null;
        let repo = null;
        let branch = "main";
        let requestData = {};

        if (request.method === "POST") {
            try {
                requestData = await request.json();
            } catch (e) {
                requestData = {};
            }
            
            masterKey = requestData.masterKey || requestData.key || null;
            cfAccountId = requestData.cfAccountId || requestData.accountId || null;
            cfApiToken = requestData.cfApiToken || requestData.apiToken || null;
            cfWorkerName = requestData.cfWorkerName || requestData.workerName || null;
            repo = requestData.repo || "mahbodrahimi/nahan";
            branch = requestData.branch || "main";
            action = requestData.action || "check";
        } else if (request.method === "GET") {
            const url = new URL(request.url);
            masterKey = url.searchParams.get("masterKey") || url.searchParams.get("key") || null;
            cfAccountId = url.searchParams.get("cfAccountId") || url.searchParams.get("accountId") || null;
            cfApiToken = url.searchParams.get("cfApiToken") || url.searchParams.get("apiToken") || null;
            cfWorkerName = url.searchParams.get("cfWorkerName") || url.searchParams.get("workerName") || null;
            repo = url.searchParams.get("repo") || "mahbodrahimi/nahan";
            branch = url.searchParams.get("branch") || "main";
            action = url.searchParams.get("action") || "check";
        } else {
            return new Response(
                JSON.stringify({ success: false, error: "Method not allowed" }),
                { status: 405, headers: { "Content-Type": "application/json" } }
            );
        }

        // ===== CHECK MASTER KEY =====
        const envMasterKey = env.MASTER_KEY || "";
        if (envMasterKey && masterKey !== envMasterKey) {
            return new Response(
                JSON.stringify({ success: false, error: "Unauthorized. Invalid Master Key." }),
                { status: 401, headers: { "Content-Type": "application/json" } }
            );
        }
        if (!envMasterKey && !masterKey) {
            return new Response(
                JSON.stringify({ 
                    success: false, 
                    error: "Master Key is required. Please send 'masterKey' in the request." 
                }),
                { status: 401, headers: { "Content-Type": "application/json" } }
            );
        }

        // ============================================
        // ===== ACTION: CHECK =====
        // ============================================
        if (action === "check") {
            try {
                // Get current version from config
                const currentVersion = requestData.currentVersion || "2.9.4";
                
                // Fetch code info from GitHub
                const codeInfo = await fetchCompleteCodeFromGitHub(repo, branch);
                if (!codeInfo.success) {
                    throw new Error(codeInfo.error);
                }

                const latestVersion = codeInfo.version || "unknown";
                const isNewer = latestVersion !== "unknown" && cmpVersions(latestVersion, currentVersion) > 0;

                // Get repo info
                const repoInfo = await getRepoInfo(repo, branch);

                return new Response(
                    JSON.stringify({
                        success: true,
                        current: currentVersion,
                        latest: latestVersion,
                        updateAvailable: isNewer,
                        whatsNew: codeInfo.whatsNew || "",
                        codeSize: codeInfo.size,
                        codeLines: codeInfo.lines,
                        hasImports: codeInfo.hasImports,
                        isObfuscated: codeInfo.isObfuscated,
                        repo: repo,
                        branch: branch,
                        repoInfo: repoInfo,
                        targetWorker: cfWorkerName || env.CF_WORKER_NAME || "not-specified"
                    }),
                    { headers: { "Content-Type": "application/json" } }
                );
            } catch (e) {
                return new Response(
                    JSON.stringify({ 
                        success: false, 
                        error: "Failed to check for updates: " + e.message 
                    }),
                    { status: 500, headers: { "Content-Type": "application/json" } }
                );
            }
        }

        // ============================================
        // ===== ACTION: APPLY_UPDATE =====
        // ============================================
        if (action === "apply_update") {
            // ===== GET CREDENTIALS =====
            const accountId = cfAccountId || env.CF_ACCOUNT_ID || "";
            const apiToken = cfApiToken || env.CF_API_TOKEN || "";
            const workerName = cfWorkerName || env.CF_WORKER_NAME || "";

            // ===== Check Cloudflare credentials =====
            if (!accountId || !apiToken || !workerName) {
                let missing = [];
                if (!accountId) missing.push("Account ID");
                if (!apiToken) missing.push("API Token");
                if (!workerName) missing.push("Worker Name");
                
                return new Response(
                    JSON.stringify({
                        success: false,
                        error: `Cloudflare credentials missing. Please provide: ${missing.join(", ")}.\n` +
                               `Send in request: cfAccountId, cfApiToken, cfWorkerName`
                    }),
                    { status: 400, headers: { "Content-Type": "application/json" } }
                );
            }

            // ===== Fetch complete code from GitHub =====
            const codeInfo = await fetchCompleteCodeFromGitHub(repo, branch);
            if (!codeInfo.success) {
                return new Response(
                    JSON.stringify({
                        success: false,
                        error: "Failed to fetch code from GitHub: " + codeInfo.error
                    }),
                    { status: 502, headers: { "Content-Type": "application/json" } }
                );
            }

            const code = codeInfo.code;
            const latestVersion = codeInfo.version || "unknown";
            const currentVersion = "2.9.4";

            // ===== Check if already up to date =====
            if (latestVersion !== "unknown" && cmpVersions(latestVersion, currentVersion) <= 0) {
                return new Response(
                    JSON.stringify({
                        success: true,
                        message: `Already up to date (v${currentVersion})`,
                        current: currentVersion,
                        latest: latestVersion,
                        updated: false,
                        targetWorker: workerName
                    }),
                    { headers: { "Content-Type": "application/json" } }
                );
            }

            // ===== Deploy new code =====
            try {
                const deployRes = await deployWorkerToCloudflare(
                    accountId,
                    apiToken,
                    workerName,
                    code
                );

                const deployData = await deployRes.json();

                if (deployData.success) {
                    console.log(`✅ Worker "${workerName}" updated from v${currentVersion} to v${latestVersion}`);
                    console.log(`📦 Code size: ${codeInfo.size} bytes, ${codeInfo.lines} lines`);

                    // Send notification if Telegram is configured
                    if (env.TG_TOKEN && (env.TG_ADMIN_ID || env.TG_CHAT_ID)) {
                        const notifyChatId = env.TG_ADMIN_ID || env.TG_CHAT_ID;
                        const tgMsg = `✅ <b>Worker Auto-Updated</b>\n\n` +
                                     `📦 <b>Worker:</b> ${workerName}\n` +
                                     `📦 <b>New Version:</b> v${latestVersion}\n` +
                                     `🔄 <b>Previous:</b> v${currentVersion}\n` +
                                     `📄 <b>Code Size:</b> ${(codeInfo.size / 1024).toFixed(1)} KB\n` +
                                     `📝 <b>Lines:</b> ${codeInfo.lines}\n` +
                                     `📁 <b>Repo:</b> ${repo}/${branch}\n` +
                                     `⏰ <b>Time:</b> ${new Date().toLocaleString()}`;
                        
                        fetch(`https://api.telegram.org/bot${env.TG_TOKEN}/sendMessage`, {
                            method: 'POST',
                            headers: { 'Content-Type': 'application/json' },
                            body: JSON.stringify({
                                chat_id: notifyChatId,
                                text: tgMsg,
                                parse_mode: 'HTML'
                            })
                        }).catch(() => {});
                    }

                    return new Response(
                        JSON.stringify({
                            success: true,
                            message: `✅ Update applied successfully to "${workerName}"`,
                            current: currentVersion,
                            latest: latestVersion,
                            updated: true,
                            targetWorker: workerName,
                            codeSize: codeInfo.size,
                            codeLines: codeInfo.lines,
                            repo: repo,
                            branch: branch,
                            deployResult: deployData
                        }),
                        { headers: { "Content-Type": "application/json" } }
                    );
                } else {
                    return new Response(
                        JSON.stringify({
                            success: false,
                            error: "Deploy failed",
                            targetWorker: workerName,
                            details: deployData.errors || deployData
                        }),
                        { status: 500, headers: { "Content-Type": "application/json" } }
                    );
                }
            } catch (e) {
                return new Response(
                    JSON.stringify({
                        success: false,
                        error: "Deploy error: " + e.message,
                        targetWorker: workerName
                    }),
                    { status: 500, headers: { "Content-Type": "application/json" } }
                );
            }
        }

        // ============================================
        // ===== ACTION: VIEW_CODE =====
        // ============================================
        if (action === "view_code") {
            try {
                const codeInfo = await fetchCompleteCodeFromGitHub(repo, branch);
                if (!codeInfo.success) {
                    throw new Error(codeInfo.error);
                }

                // Return first 5000 characters of code for preview
                const preview = codeInfo.code.slice(0, 5000);
                const totalSize = codeInfo.code.length;

                return new Response(
                    JSON.stringify({
                        success: true,
                        repo: repo,
                        branch: branch,
                        version: codeInfo.version,
                        totalSize: totalSize,
                        preview: preview,
                        previewSize: preview.length,
                        hasMore: totalSize > 5000,
                        isObfuscated: codeInfo.isObfuscated,
                        hasImports: codeInfo.hasImports
                    }),
                    { headers: { "Content-Type": "application/json" } }
                );
            } catch (e) {
                return new Response(
                    JSON.stringify({
                        success: false,
                        error: "Failed to view code: " + e.message
                    }),
                    { status: 500, headers: { "Content-Type": "application/json" } }
                );
            }
        }

        // ============================================
        // ===== ACTION: DOWNLOAD_CODE =====
        // ============================================
        if (action === "download_code") {
            try {
                const codeInfo = await fetchCompleteCodeFromGitHub(repo, branch);
                if (!codeInfo.success) {
                    throw new Error(codeInfo.error);
                }

                // Return the full code as a file download
                return new Response(codeInfo.code, {
                    headers: {
                        'Content-Type': 'application/javascript',
                        'Content-Disposition': `attachment; filename="_worker.js"`,
                        'X-Version': codeInfo.version || 'unknown',
                        'X-Size': codeInfo.code.length.toString(),
                        'X-Lines': codeInfo.code.split('\n').length.toString(),
                        'X-Repo': repo,
                        'X-Branch': branch
                    }
                });
            } catch (e) {
                return new Response(
                    JSON.stringify({
                        success: false,
                        error: "Failed to download code: " + e.message
                    }),
                    { status: 500, headers: { "Content-Type": "application/json" } }
                );
            }
        }

        // ============================================
        // ===== ACTION: UPDATE_CLEAN_IPS =====
        // ============================================
        if (action === "update_clean_ips") {
            const newIps = await fetchRemoteCleanIps();
            if (!newIps) {
                return new Response(
                    JSON.stringify({ 
                        success: false, 
                        error: "Failed to fetch remote clean IPs" 
                    }),
                    { status: 502, headers: { "Content-Type": "application/json" } }
                );
            }

            return new Response(
                JSON.stringify({
                    success: true,
                    message: "Clean IPs fetched successfully",
                    ips: newIps,
                    count: newIps.split('\n').filter(Boolean).length
                }),
                { headers: { "Content-Type": "application/json" } }
            );
        }

        // ============================================
        // ===== ACTION: STATUS =====
        // ============================================
        if (action === "status") {
            const hasEnvCredentials = !!(env.CF_ACCOUNT_ID && env.CF_API_TOKEN && env.CF_WORKER_NAME);
            
            return new Response(
                JSON.stringify({
                    success: true,
                    currentVersion: "2.9.4",
                    hasCredentials: !!(cfAccountId || env.CF_ACCOUNT_ID) && 
                                    !!(cfApiToken || env.CF_API_TOKEN) && 
                                    !!(cfWorkerName || env.CF_WORKER_NAME),
                    hasEnvCredentials: hasEnvCredentials,
                    hasMasterKey: !!(env.MASTER_KEY || false),
                    status: "online",
                    info: {
                        credentialsSource: hasEnvCredentials ? "Environment Variables" : "Request",
                        accountId: (cfAccountId || env.CF_ACCOUNT_ID) ? "***" + (cfAccountId || env.CF_ACCOUNT_ID).slice(-4) : null,
                        workerName: cfWorkerName || env.CF_WORKER_NAME || null,
                        defaultRepo: "mahbodrahimi/nahan",
                        defaultBranch: "main"
                    }
                }),
                { headers: { "Content-Type": "application/json" } }
            );
        }

        // ============================================
        // ===== ACTION: UPDATE_STATUS =====
        // ============================================
        if (action === "update_status") {
            return new Response(
                JSON.stringify({
                    success: true,
                    message: "Updater is running",
                    version: "4.0",
                    status: "online",
                    endpoints: {
                        check: {
                            method: "POST",
                            description: "Check for updates from GitHub",
                            required: ["masterKey"],
                            optional: ["cfAccountId", "cfApiToken", "cfWorkerName", "repo", "branch"]
                        },
                        apply_update: {
                            method: "POST",
                            description: "Download complete code from GitHub and deploy to target worker",
                            required: ["masterKey", "cfAccountId", "cfApiToken", "cfWorkerName"],
                            optional: ["repo", "branch"]
                        },
                        view_code: {
                            method: "POST",
                            description: "View code preview from GitHub",
                            required: ["masterKey"],
                            optional: ["repo", "branch"]
                        },
                        download_code: {
                            method: "POST",
                            description: "Download full code from GitHub",
                            required: ["masterKey"],
                            optional: ["repo", "branch"]
                        },
                        update_clean_ips: {
                            method: "POST",
                            description: "Fetch clean IPs from remote source",
                            required: ["masterKey"]
                        },
                        status: {
                            method: "POST",
                            description: "Get updater status",
                            required: ["masterKey"]
                        }
                    }
                }),
                { headers: { "Content-Type": "application/json" } }
            );
        }

        // ===== Invalid action =====
        return new Response(
            JSON.stringify({
                success: false,
                error: "Invalid action. Available actions: check, apply_update, view_code, download_code, update_clean_ips, status, update_status"
            }),
            { status: 400, headers: { "Content-Type": "application/json" } }
        );

    } catch (e) {
        return new Response(
            JSON.stringify({ 
                success: false, 
                error: "Internal error: " + e.message 
            }),
            { status: 500, headers: { "Content-Type": "application/json" } }
        );
    }
}

// ============================================
// ===== EXPORT =====
// ============================================
export default {
    async fetch(request, env, ctx) {
        // ===== Handle CORS preflight =====
        if (request.method === "OPTIONS") {
            return new Response(null, {
                status: 204,
                headers: {
                    "Access-Control-Allow-Origin": "*",
                    "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
                    "Access-Control-Allow-Headers": "Content-Type, Authorization",
                    "Access-Control-Max-Age": "86400",
                },
            });
        }

        // ===== Handle request =====
        const response = await handleUpdate(request, env, ctx);
        
        // ===== Add CORS headers =====
        response.headers.set("Access-Control-Allow-Origin", "*");
        response.headers.set("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
        response.headers.set("Access-Control-Allow-Headers", "Content-Type, Authorization");
        
        return response;
    }
};
