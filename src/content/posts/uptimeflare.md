---
title: 超高校级的监控服务：UptimeFlare！基于CF Worker！自托管！声明式！新手友好！
published: 2026-01-03T03:10:53
description: 谁不想拥有一个监控自己服务的服务呢？
image: ../assets/images/uptimeflare.png
draft: false
lang: ""
---
# 前言
本来这个教程应该是永远都不会出的，因为在此之前，我曾经给大家介绍了一个无需自托管的监控服务：[UptimeRobot](/posts/uptimerobot/) 

但是，就在最近我再次查看控制台，发现之前创建的监控全部都没了，咱也不知道是被官方删了还是号被黑客大手子肘击了，总之，我现在不得不要重建监控服务了

# 原理
首先，UptimeFlare是一个基于Cloudflare Worker+KV的监控服务

它的原理非常简单，一共由三个部分组成
- **前端**：放在Cloudflare Page，用于给用户展示zhandianzhuangt
- **后端**：放在Cloudflare Worker，通过 Worker 自带的 **Cron** 每分钟 检查站点状态，并将状态持久化进 **KV** 
![](../assets/images/uptimeflare-1.png)
![](../assets/images/uptimeflare-2.png)

# 正式开始
首先我们需要 **Fork** 项目，你可以选择原项目（English），或者我的项目（中文）

::github{repo="lyc8503/UptimeFlare"}
::github{repo="afoim/UptimeFlare"}

首先，我们先尝试将其部署到Cloudflare

创建一个Cloudflare API Token（编辑Workers）
![](../assets/images/msedge_VsxMMnGtBE.gif)

接下来将该Token绑定到你的Github仓库
![](../assets/images/uptimeflare-4.png)

最后，来到 `Action` 页面，手动创建一个 `Deploy to Cloudflare` 的工作流
![](../assets/images/uptimeflare-5.png)

等待工作流运行结束，你应该可以在Cloudflare仪表板看见一个新的Page，新的Worker和新的KV
![](../assets/images/uptimeflare-6.png)
![](../assets/images/uptimeflare-7.png)

点开 Page，注意不要点错了
![](../assets/images/uptimeflare-8.png)

绑定你的域名，尝试访问
![](../assets/images/uptimeflare-9.png)

如果你能看到一个初始的监控页面，则正常
![](../assets/images/uptimeflare-10.png)

接下来，我们开始自定义该监控

编辑根目录的 `uptime.config.ts` 

这里是默认的汉化过后的配置，拷贝过去并定制成你的版本吧~

```ts
/**
 * ============================
 * UptimeFlare 通用配置模板
 * ============================
 * 使用前请将示例内容替换为你自己的信息
 */

/* ----------------------------
 * 状态页面配置
 * ---------------------------- */
const pageConfig = {
  // 状态页面标题
  title: "我的服务状态页",

  // 状态页顶部链接
  links: [
    { link: "https://example.com", label: "主页" },
    { link: "https://github.com/example", label: "GitHub" },
  ],

  // [可选] 监控分组
  // 未列入分组的监控将不会显示在状态页（但仍会被监控）
  /*
  group: {
    "Web 服务": ["web_main", "web_backup"],
    "服务器": ["server_ssh"],
  },
  */
};

/* ----------------------------
 * Worker 监控配置
 * ---------------------------- */
const workerConfig = {
  // 除非状态发生变化，否则最多每 N 分钟写入一次 KV
  kvWriteCooldownMinutes: 3,

  // [可选] HTTP Basic Auth
  // passwordProtection: "username:password",

  /* --------------------------
   * 监控列表
   * -------------------------- */
  monitors: [
    /**
     * HTTP / HTTPS 监控示例
     */
    {
      id: "web_main", // 唯一 ID（不要随意修改，否则历史会丢失）
      name: "主站",
      method: "HEAD", // GET / HEAD / POST ...
      target: "https://example.com/",
      statusPageLink: "https://example.com/",
      expectedCodes: [200],
      timeout: 10000,
      hideLatencyChart: false,
    },

    /**
     * TCP 端口存活监控示例（SSH / 数据库等）
     */
    {
      id: "server_ssh",
      name: "服务器 SSH",
      method: "TCP_PING",
      target: "127.0.0.1:22",
      timeout: 5000,
    },
  ],

  /* --------------------------
   * 通知配置
   * -------------------------- */
  notification: {
    // 时区
    timeZone: "Asia/Shanghai",

    // 宽限期（分钟）
    // 连续失败 N 分钟后才发送告警
    gracePeriod: 5,

    // [可选] Apprise
    // appriseApiServer: "https://apprise.example.com/notify",
    // recipientUrl: "tgram://bottoken/chatid",
  },

  /* --------------------------
   * 回调函数
   * -------------------------- */
  callbacks: {
    /**
     * 监控状态变化时触发
     */
    onStatusChange: async (
      env: any,
      monitor: any,
      isUp: boolean,
      timeIncidentStart: number,
      timeNow: number,
      reason: string
    ) => {
      const statusText = isUp ? "恢复正常 (UP)" : "服务中断 (DOWN)";
      console.log(
        `[状态变更] ${monitor.name} -> ${statusText}，原因：${reason}`
      );

      // 示例：在此处接入 邮件 / Webhook / 飞书 / Telegram 等
      // env 中可以读取 Worker 环境变量
    },

    /**
     * 事件持续期间（每分钟调用一次）
     */
    onIncident: async (
      env: any,
      monitor: any,
      timeIncidentStart: number,
      timeNow: number,
      reason: string
    ) => {
      // 可用于升级告警、统计持续时间等
    },
  },
};

// ⚠️ 必须导出，否则编译失败
export { pageConfig, workerConfig };

```

如果服务故障如何做通知？

UptimeFlare非常自由，你可以在 `callbacks` 中编写故障时要做的任何事情，这里以发送 `POST` 请求让 `Resend` 发送邮件给你举例

首先前往 https://resend.com/

添加一个域名（作为你的发信域名）
![](../assets/images/uptimeflare-11.png)

创建一个发信API Key
![](../assets/images/uptimeflare-12.png)

添加环境变量： `RESEND_API_KEY` 将其绑定到 **Action**

![](../assets/images/uptimeflare-13.png)

编辑 `uptime.config.ts` 的 `callbacks` 部分，注意更改 `resendPayload` 方法中的相关参数

示例代码：
```ts
  callbacks: {
    onStatusChange: async (
      env: any,
      monitor: any,
      isUp: boolean,
      timeIncidentStart: number,
      timeNow: number,
      reason: string
    ) => {
      // 当任何监控的状态发生变化时，将调用此回调
      // 在这里编写任何 Typescript 代码
  
      // 调用 Resend API 发送邮件通知
      // 务必在 Cloudflare Worker 的设置 -> 变量中配置: RESEND_API_KEY
      if (env.RESEND_API_KEY) {
        try {
          const statusText = isUp ? '恢复正常 (UP)' : '服务中断 (DOWN)';
          const color = isUp ? '#4ade80' : '#ef4444'; // green-400 : red-500
          const subject = `[${statusText}] ${monitor.name} 状态变更通知`;
          // 尝试格式化时间
          let timeString = new Date(timeNow * 1000).toISOString();
          try {
            timeString = new Date(timeNow * 1000).toLocaleString('zh-CN', { timeZone: 'Asia/Shanghai' });
          } catch (e) { /* ignore */ }
  
          const htmlContent = `
            <div style="font-family: sans-serif; padding: 20px; border: 1px solid #eee; border-radius: 5px;">
              <h2 style="color: ${color};">${statusText}</h2>
              <p><strong>监控名称:</strong> ${monitor.name}</p>
              <p><strong>时间:</strong> ${timeString}</p>
              <p><strong>原因:</strong> ${reason}</p>
              <hr style="border: none; border-top: 1px solid #eee; margin: 20px 0;">
              <p style="font-size: 12px; color: #888;">来自 UptimeFlare 监控报警</p>
            </div>
          `;
  
          const resendPayload = {
            from: "系统状态更新 <uptimeflare@你的域名.com>",
            to: ["你的@邮箱.com"],
            subject: subject,
            html: htmlContent,
          };
  
          const resp = await fetch('https://api.resend.com/emails', {
            method: 'POST',
            headers: {
              'Authorization': `Bearer ${env.RESEND_API_KEY}`,
              'Content-Type': 'application/json'
            },
            body: JSON.stringify(resendPayload)
          });
  
          if (!resp.ok) {
            console.error(`Resend API call failed: ${resp.status} ${await resp.text()}`);
          }
        } catch (e) {
          console.error(`Error calling Resend API: ${e}`);
        }
      }
  
      // 这不会遵循宽限期设置，并且在状态变化时立即调用
      // 如果您想实现宽限期，需要手动处理
    },
    onIncident: async (
      env: any,
      monitor: any,
      timeIncidentStart: number,
      timeNow: number,
      reason: string
    ) => {
      // 如果任何监控有正在进行的事件，此回调将每分钟调用一次
      // 在这里编写任何 Typescript 代码
  
  
    },
  },
}
```

接下来，当服务故障/重新上线就会通知你啦~
![](../assets/images/1dc1a98a404db83e909c1f87e8a115cf.png)

最终效果：
::url{href="https://ok.2x.nz"}