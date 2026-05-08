/**
 * 69云机场自动签到脚本
 * 修复版 v2.1 | 适配Loon/Surge | 支持多账号/静默模式
 * 
 * 功能说明：
 * 1. 自动解析Loon/Surge传入的账号参数
 * 2. 多账号顺序签到（最多5个）
 * 3. 静默模式支持（不推送通知）
 * 4. 完整错误日志追踪
 * 5. 账号隐私保护（自动掩码邮箱）
 */

// ========= 核心工具函数（必须提前定义） =========
function maskEmail(email) {
    if (!email || !email.includes("@")) return "未知账号";
    const [name, domain] = email.split("@");
    return name[0] + "***" + name.slice(-1) + "@" + domain;
}

function log(msg, level = "info") {
    const time = new Date().toISOString().replace("T", " ").substring(0, 19);
    const prefix = level === "error" ? "❌" : level === "warn" ? "⚠️" : "ℹ️";
    console.log(`[${time}] ${prefix} ${msg}`);
}

// ========= 参数解析器 =========
let isSilent = false;
let accounts = [];

function parseParams() {
    try {
        let arg = (typeof $argument !== "undefined" && $argument) ? $argument : "";
        log(`原始参数类型: ${typeof arg}`, "info");
        
        // 情况1: Loon传入对象 (标准格式)
        if (typeof arg === "object" && arg !== null && !Array.isArray(arg)) {
            log(`解析到Loon对象参数: ${JSON.stringify(arg)}`, "info");
            
            // 处理静默模式
            if (arg["静默运行"] === "#" || arg["silent"] === "#") {
                isSilent = true;
                log("检测到静默运行模式", "info");
            }
            
            // 解析5个账号字段
            for (let i = 1; i <= 5; i++) {
                const key = `账号和密码${i}`;
                if (arg[key] && typeof arg[key] === "string" && arg[key].includes("@")) {
                    const sep = arg[key].includes(":") ? ":" : (arg[key].includes(",") ? "," : null);
                    if (sep) {
                        const [email, password] = arg[key].split(sep).map(s => s.trim());
                        if (email && password && email.includes("@")) {
                            accounts.push({ email, password });
                            log(`✅ 账号${i}解析成功: ${maskEmail(email)}`, "info");
                        }
                    }
                }
            }
        }
        // 情况2: Surge字符串传参 (兼容模式)
        else if (typeof arg === "string") {
            log(`解析到Surge字符串参数: ${arg}`, "info");
            if (arg.includes("silent=#")) isSilent = true;
            
            const accountStr = arg.split("&silent=")[0] || arg;
            const parts = accountStr.split("#").filter(p => p.trim() !== "" && p.includes("@"));
            
            parts.forEach((p, idx) => {
                const sep = p.includes(":") ? ":" : (p.includes(",") ? "," : null);
                if (sep) {
                    const [email, password] = p.split(sep).map(s => s.trim());
                    if (email && password) {
                        accounts.push({ email, password });
                        log(`✅ 账号${idx+1}解析成功: ${maskEmail(email)}`, "info");
                    }
                }
            });
        }
        
        log(`最终解析到 ${accounts.length} 个有效账号`, "info");
        if (accounts.length === 0) {
            log("⚠️ 未检测到有效账号，请检查参数格式", "warn");
        }
    } catch (e) {
        log(`参数解析失败: ${e.message}`, "error");
        $done();
    }
}

// ========= 网络请求封装 =========
function httpRequest(url, method, headers, body = null) {
    return new Promise((resolve, reject) => {
        const opts = { url, method, headers };
        if (body) opts.body = JSON.stringify(body);
        
        $httpClient[method.toLowerCase()](opts, (err, resp, data) => {
            if (err) return reject(err);
            try {
                resolve({ 
                    status: resp.status, 
                    headers: resp.headers,
                    body: data ? JSON.parse(data) : null 
                });
            } catch (e) {
                resolve({ status: resp.status, body: data });
            }
        });
    });
}

// ========= 核心业务逻辑 =========
async function performLogin(email, password) {
    try {
        const loginUrl = "https://api.69yun99.com/api/v1/passport/auth/login";
        const headers = {
            "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 15_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148",
            "Content-Type": "application/json"
        };
        
        const response = await httpRequest(loginUrl, "POST", headers, {
            email,
            password
        });
        
        if (response.status === 200 && response.body && response.body.data && response.body.data.token) {
            return response.body.data.token;
        }
        
        throw new Error(`登录失败 [${response.status}] ${response.body?.message || '未知错误'}`);
    } catch (e) {
        throw new Error(`登录请求异常: ${e.message}`);
    }
}

async function performCheckin(token) {
    try {
        const checkinUrl = "https://api.69yun99.com/api/v1/user/check-in";
        const headers = {
            "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 15_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148",
            "Authorization": `Bearer ${token}`
        };
        
        const response = await httpRequest(checkinUrl, "POST", headers);
        
        if (response.status === 200 && response.body && response.body.data) {
            return response.body.data;
        }
        
        throw new Error(`签到失败 [${response.status}] ${response.body?.message || '未知错误'}`);
    } catch (e) {
        throw new Error(`签到请求异常: ${e.message}`);
    }
}

// ========= 结果处理器 =========
function handleResult(result, email) {
    const maskedEmail = maskEmail(email);
    const msgParts = [
        `**69云机场签到结果**`,
        `📧 账号: ${maskedEmail}`,
        `⏱️ 时间: ${new Date().toLocaleString()}`,
        `✅ 签到成功! 今日获得: ${result.today || '未知'} MB`,
        `📊 当前总量: ${result.sum || '未知'} MB`
    ];
    
    if (result.continuous) {
        msgParts.push(`🔥 连续签到: ${result.continuous} 天`);
    }
    
    return msgParts.join("\n");
}

function handleError(error, email) {
    const maskedEmail = maskEmail(email);
    return [
        `**69云机场签到失败**`,
        `📧 账号: ${maskedEmail}`,
        `⏱️ 时间: ${new Date().toLocaleString()}`,
        `❌ 错误: ${error.message}`
    ].join("\n");
}

// ========= 主执行流程 =========
async function main() {
    parseParams();
    
    if (accounts.length === 0) {
        if (!isSilent) $notification.post("69云签到", "⚠️ 账号错误", "未检测到有效账号，请检查配置");
        log("终止执行：无有效账号", "error");
        return $done();
    }

    let successCount = 0;
    let results = [];
    
    for (const [index, account] of accounts.entries()) {
        try {
            log(`开始处理账号 ${index + 1}/${accounts.length}: ${maskEmail(account.email)}`, "info");
            
            // 1. 登录获取token
            const token = await performLogin(account.email, account.password);
            log(`✅ 账号 ${maskEmail(account.email)} 登录成功`, "info");
            
            // 2. 执行签到
            const result = await performCheckin(token);
            successCount++;
            
            // 3. 生成结果
            const message = handleResult(result, account.email);
            results.push(message);
            log(`✅ 账号 ${maskEmail(account.email)} 签到成功: +${result.today} MB`, "info");
            
        } catch (e) {
            const errorMsg = handleError(e, account.email);
            results.push(errorMsg);
            log(`❌ 账号 ${maskEmail(account.email)} 签到失败: ${e.message}`, "error");
        }
    }

    // ========= 结果汇总 =========
    const summary = [
        `**69云机场签到汇总**`,
        `📅 日期: ${new Date().toLocaleDateString()}`,
        `👤 账号: ${accounts.length} 个`,
        `✅ 成功: ${successCount} 个`,
        `❌ 失败: ${accounts.length - successCount} 个`
    ].join("\n");
    
    // 静默模式不推送通知
    if (!isSilent && results.length > 0) {
        $notification.post(
            "69云签到", 
            `成功: ${successCount}/${accounts.length}`, 
            [summary, ...results].join("\n\n")
        );
    }
    
    log(`执行完成: 成功${successCount}/${accounts.length}个账号`, "info");
    $done();
}

// ========= 脚本入口 =========
!(async () => {
    try {
        await main();
    } catch (e) {
        log(`脚本全局异常: ${e.message}`, "error");
        if (!isSilent) {
            $notification.post("69云签到", "❌ 脚本错误", e.message);
        }
        $done();
    }
})();
