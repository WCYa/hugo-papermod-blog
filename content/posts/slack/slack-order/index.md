---
title: "🤖 Slack 機器人：訂便當機器人"
date: 2025-11-24T13:01:57+08:00
draft: false
tags: ["slack", "app", "bot"]
---

## 為什麼有這個 side project

過往工作的團隊都是跟外面店家訂午餐，並且有各種門檻：需要滿多少錢 or 需要幾份以上才能外送。這造成負責訂餐的同仁需要先統計資訊後才能向店家訂餐，且都是大家透過 Slack 傳訊息給負責人，不只耗費人力去統計且偶爾會出錯造成餐點數量不對。

這個 side project 就是為了減少這件事所耗費的人力與時間成本，讓收集、統計訂單資訊的這種工作更有效率和降低出錯機率。

這個 project 是使用 [Bolt for Python](https://github.com/slackapi/bolt-python) 建立 socket mode 的 Slack app，這模式有不用固定 IP、不用域名、不用合法 SSL 憑證等等優勢。

## Demo

![Demo](slack-order_demo.gif)

## 注意事項

1. 此專案是使用 socket mode，沒辦法部屬到 Slack Marketplace。建議只在公司內部、私人 workspace 使用。
2. 如果同時運作兩個 socket mode app，原則上應該會踢出第一個 app，但因為使用 docker compose 且設定 `restart: always` 的關係，可能兩個 app 會持續重新啟動互相競爭，造成功能異常。

## 重頭建立 Slack app

> 你可以直接使用我已經設定好的 manifest 檔案去建立 Slack app，跳至 [Create your app from a manifest](#使用-manifest-檔案建立-slack-app)

1. [建立你的 Slack app](https://api.slack.com/apps/new)：使用 `from scratch` 的方式，建立之後你將進入你的 Slack app 基本資訊頁面。或從 [Your Apps](https://api.slack.com/apps) 進入剛剛建立的 Slack app。
2. 啟用 Socket Mode 與建立 App-level token：點選側邊欄的 `Socket Mode`，將 `Enable Socket Mode` 的 Toggle Button 開啟，這將協助你建立 App-level token 和需要的 Scope。建立完後可將 App-level token 先複製下來，此 Token 應該由 `xapp-` 開頭。
3. 設定 Slash Commands：點選側邊欄的 `Slash Commands`，點選 `Create new Command`，指令名稱可隨意，例如：
   - Command：`/order`
   - Short Description：`開啟午餐訂單`
4. 調整 Bot Token Scopes：點選側邊欄的 `OAuth & Permissions`，滾動到 Scopes 部份的 `Bot Token Scopes` 區域，點選 `Add an OAuth Scope` 新增以下 scopes：
   - `commands`：上面開啟 Slash Commands 就會新增了
   - `chat:write`：Slack app 發送訊息到頻道
   - `pins:write`：透過 `/order` slash command 開啟的訊息會被 pined
5. 安裝 Slack App 到 Workspace：點選側邊欄的 `Install App`，點選 `Install to {Your workspace}`。安裝完後同頁面會出現 Bot User OAuth Token，應該為 `xoxb-` 開頭，同樣複製起來。
6. 將 Slack App 加入某個 Slack 頻道：
   1. 打開你的 Slack
   2. 對 Slack 頻道按右鍵，點選 `View channel details`
   3. 切到 `Integrations` 分頁
   4. 按 `Add an App`，選擇剛剛建立的 Slack app

## 使用 manifest 檔案建立 Slack app

1. [建立你的 Slack app](https://api.slack.com/apps/new)：使用 `from a manifest` 的方式，貼上以下 manifest json
2. 建立好 Slack app 後，一樣需要將其安裝到 Workspace 和從 Slack 頻道加入 App

```json
{
    "display_information": {
        "name": "slack-order"
    },
    "features": {
        "bot_user": {
            "display_name": "slack-order",
            "always_online": false
        },
        "slash_commands": [
            {
                "command": "/order",
                "description": "開啟午餐訂單",
                "should_escape": false
            }
        ]
    },
    "oauth_config": {
        "scopes": {
            "bot": [
                "chat:write",
                "commands",
                "pins:write"
            ]
        }
    },
    "settings": {
        "interactivity": {
            "is_enabled": true
        },
        "org_deploy_enabled": false,
        "socket_mode_enabled": true,
        "token_rotation_enabled": false
    }
}
```

## 運作 Slack app

1. 透過 docker compose 去運行 app，所以主機需要先安裝 Docker Engine
2. clone 專案

```bash
git clone https://github.com/WCYa/slack-order.git
```

3. 從範例複製環境變數設定檔

```bash
cd slack-order
cp env_template .env
```

4. 編輯 `.env`，新增以下兩個 tokens
   - SLACK_APP_TOKEN='xapp-...'
   - SLACK_BOT_TOKEN='xoxb-...'
5. 運行

```bash
docker compose up
# or run container in background
docker compose up -d
```
