const $ = new Env("声荐组合任务");

// ================= 参数解析 (专门兼容 Loon 的对象传参) =================
const ARGS = (() => {
    let val = "0"; 
    if (typeof $argument !== "undefined" && $argument) {
        // 如果 Loon 传过来的是对象 [object Object]
        if (typeof $argument === "object") {
            // 尝试提取 summary_notify 键值，如果拿不到就取对象里的第一个值
            val = $argument.summary_notify || Object.values($argument)[0] || "0";
        } else {
            // 如果是字符串，清理掉多余符号
            val = String($argument).trim().replace(/[\"\{\}\[\]]/g, "");
        }
    }
    // 强制转换为字符串 "1" 或 "0"
    val = String(val) === "1" ? "1" : "0";
    return { notify: val };
})();

const tokenKey = "shengjian_auth_token";
const STATS_KEY = "shengjian_daily_stats";
const LAST_RUN_HOUR = 22; 

const currentHour = new Date().getHours();
const isLastRun = (currentHour === LAST_RUN_HOUR);

const rawToken = $.read(tokenKey);
const token = rawToken ? (rawToken.startsWith("Bearer ") ? rawToken : `Bearer ${rawToken}`) : null;

const commonHeaders = {
  "Authorization": token,
  "Content-Type": "application/json",
  "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 16_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 MicroMessenger/8.0.64 NetType/4G Language/zh_CN",
  "Referer": "https://servicewechat.com/wxa25139b08fe6e2b6/23/page-frame.html"
};

// ================= 数据持久化 =================
function getDailyStats() {
    const today = new Date().toISOString().slice(0, 10);
    let stats = {};
    try { stats = JSON.parse($.read(STATS_KEY) || "{}"); } catch (e) { stats = {}; }
    if (stats.date !== today || !Array.isArray(stats.logs)) {
        stats = { date: today, logs: [] };
    }
    return stats;
}

function saveDailyStats(stats) {
    $.write(JSON.stringify(stats), STATS_KEY);
}

// ================= 业务请求 =================
function signIn() {
  return new Promise((resolve) => {
    $.put({ url: "https://xcx.myinyun.com:4438/napi/gift", headers: commonHeaders, body: "{}" }, (err, res, data) => {
      try {
        const result = JSON.parse(data);
        if (result.msg === "ok") resolve({ message: `✅ 签到: ${result.data?.prizeName || "成功"}` });
        else if (String(result.msg).includes("已经")) resolve({ message: '📋 签到: 已用完' });
        else resolve({ message: `🚫 签到: ${result.msg}` });
      } catch { resolve({ message: '🤯 签到解析失败' }); }
    });
  });
}

function claimFlower() {
  return new Promise((resolve) => {
    $.post({ url: "https://xcx.myinyun.com:4438/napi/flower/get", headers: commonHeaders, body: "{}" }, (err, res, data) => {
      if (data === "true") resolve({ message: '🌺 已领花' });
      else resolve({ message: '👍 已领过/未到时' });
    });
  });
}

// ================= 主逻辑 =================
(async () => {
  // 调试日志输出，确认参数是否正确解析
  console.log(`[调试] 收到原始参数类型: ${typeof $argument}`);
  console.log(`[调试] 收到原始参数内容: ${JSON.stringify($argument)}`);
  console.log(`[调试] 解析后的通知模式: ${ARGS.notify} (1:单次, 0:汇总)`);
  
  if (!token) {
    $.notify("❌ 声荐任务失败", "未找到令牌", "请先打开小程序获取。");
    return $.done();
  }

  const [signRes, flowerRes] = await Promise.all([signIn(), claimFlower()]);
  const currentResult = `${signRes.message} | ${flowerRes.message}`;
  const logWithTime = `[${currentHour}点] ${currentResult}`;
  
  let dailyStats = getDailyStats();
  dailyStats.logs.push(logWithTime);
  saveDailyStats(dailyStats);

  // 核心判定逻辑
  if (ARGS.notify === "1") {
      // 模式 1: 每次运行都弹窗（单次内容）
      $.notify("🔔 声荐单次通知", `时间: ${currentHour}点`, currentResult);
  } else if (isLastRun) {
      // 模式 0: 仅在汇总时间进行汇总弹窗
      const summaryBody = `📅 日期: ${dailyStats.date}\n🔄 运行次数: ${dailyStats.logs.length}\n───────────\n${dailyStats.logs.join("\n")}`;
      $.notify("📊 声荐每日汇总", "", summaryBody);
  } else {
      console.log("[通知] 静默模式：仅记录日志，等待22点汇总。");
  }

  $.done();
})().catch((e) => { console.log(e); $.done(); });

// ================= Env 层 (精简版) =================
function Env(name) {
  this.name = name;
  this.read = (k) => (typeof $persistentStore !== "undefined" ? $persistentStore.read(k) : $prefs.valueForKey(k));
  this.write = (v, k) => (typeof $persistentStore !== "undefined" ? $persistentStore.write(v, k) : $prefs.setValueForKey(v, k));
  this.notify = (t, s, b) => (typeof $notification !== "undefined" ? $notification.post(t, s, b) : console.log(`${t}\n${s}\n${b}`));
  this.put = (r, c) => (typeof $httpClient !== "undefined" ? $httpClient.put(r, c) : $http.put(r, c));
  this.post = (r, c) => (typeof $httpClient !== "undefined" ? $httpClient.post(r, c) : $http.post(r, c));
  this.done = (v = {}) => typeof $done !== "undefined" && $done(v);
}
