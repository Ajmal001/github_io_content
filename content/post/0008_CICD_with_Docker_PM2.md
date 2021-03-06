---
title: "利用 pm2 管理 node 服務，結合 Docker Image 達到持續交付"
date: 2018-05-20T22:05:42+08:00
draft: false
author: "Jimmy Lin"
isCJKLanguage: true
tags: ["Build", "Docker"]
categories: ["DevOps"]
---

簡單來說，我們最終的目的是：**CICD + Zero Downtime 佈署 Nextjs 的服務**。其實網路上找找有蠻多方法的，像是利用 Kubernetes _(K8S)_，或是直接整合 AWS、GCP 或是 Azure，不過礙於專案不想要放到外面去，又因為 K8S 過於複雜，學習曲線太高，我們最終選擇了這個相對來說簡單許多的方式。雖然簡單但是因為之前沒有碰過 DevOps 的工作，也花了一些時間，希望這篇文章可以幫助到一些有類似需求的人。

## PM2 管理 Nextjs 服務
[PM2](http://pm2.keymetrics.io/docs/usage/quick-start/) 是一個可以用來管理 Node 服務的一個軟體，安裝方法很簡單，不僅用了 Node 裡面的 Cluter 可以根據硬體看要起幾個 Process，還可以讓多個 Service 聽同一個 port，達到 Load Balance 的效果，另外結合  [Keymetrics](https://keymetrics.io/) 還可以監控你的服務狀態，集多種功能於一身。[這篇文章](https://larrylu.blog/nodejs-pm2-cluster-455ffbd7671)簡單介紹了一下 PM2，另外也針對 Load Balance 的效能做了一些簡單的測試，值得花時間看一下。這邊我想要多加著墨的是 `ecosystem.config.js`，一個 PM2 專屬的 config 檔，藉由設定這個檔案我們可以根據不同的開發環境給予程式不同的環境變數。直接看我們使用的 `ecosystem.config.js`

```javascript
module.exports = {
  apps : [
    {
      name: 'SERVICE_NAME',
      script: '.server/index.js',
      instances: 2,
      exec_mode: 'cluster',
      log_date_format: "YYYY-MM-DD HH:mm Z",
      env: {
        NODE_ENV: 'development',
        MONGO_URL: 'mongodb://IP_ADDRESS',
        MONGO_PORT: DB_PORT_NUMBER,
        PORT: PORT_NUMBER
      },
      env_production : {
        NODE_ENV: 'production',
        MONGO_URL: 'mongodb://IP_ADDRESS',
        MONGO_PORT: DB_PORT_NUMBER,
        PORT: PORT_NUMBER
      }
    }
  ]
};
```
簡單解釋一下，pm2 起這個專案的時候，會開兩個 instance，就是跑兩個 service，都聽 `PORT_NUMBER`；專案的起始 script 是 `.server/index.js`；db 使用 MondoDB，分成 Development 和 Production 環境，會連到不同的 `IP_ADDRESS` 以及不同的 `DB_PORT_NUMBER`。上面這些設定全都可以寫在 `ecosystem.config.js` 裡面，詳細內容請看[官網](http://pm2.keymetrics.io/docs/usage/application-declaration/)。這樣在下次 service 重起的時候，會根據指令後面帶的 `--update-env` 或是 `--env production` 選擇使用不同的環境變數，官網也有針對環境變數更詳細的[解說](http://pm2.keymetrics.io/docs/usage/environment/)。根據上面的設定檔，使用下面的指令：

```bash
## 使用預設的 env 設定，也就是 NODE_ENV=development 的那組
pm2 start ecosystem.config.js 

## 使用 env_production 設定，也就是 NODE_ENV=production 的那組
pm2 start ecosystem.config.js --env production
```

## 製作 PM2 Docker Image 銜接 Nextjs 服務
PM2 團隊提供了 [Docker Integration](https://hub.docker.com/r/keymetrics/pm2/)，可以用它們所提供的 Base Image 加上一些 Build 自己專案的指令，就可以利用 Dockerfile 新增一個帶有 PM2 環境的 Image，看一下下面這段 Dockerfile

```bash
## 使用 PM2 的 Docker Image 做為 based image
FROM keymetrics/pm2:latest-alpine

## 複製本地資料夾至 Docker Image 中
COPY src src/
COPY package.json .
COPY pm2.json .

## 執行 npm install
ENV NPM_CONFIG_LOGLEVEL warn
RUN npm install --production

## 利用 PM2-runtime 來啟動 Nextjs service，並直接使用 env_production 中的設定
CMD [ "pm2-runtime", "start", "ecosystem.config.js", "--env", "production" ]
```
利用這種方式，每次只要執行

```bash
docker run IMAGE_NAME
## 或
docker start CONTAINER_NAME
```

就可以讓你的專案藉由 PM2 來管理，可以利用

```bash
## 監看有多少個 process 正在執行
docker exec CONTAINER_NAME pm2 list 

## 目前只啟動一個 app
┌──────────┬────┬─────────┬─────┬────────┬─────────┬────────┬─────┬───────────┬──────┬──────────┐
│ App name │ id │ mode    │ pid │ status │ restart │ uptime │ cpu │ mem       │ user │ watching │
├──────────┼────┼─────────┼─────┼────────┼─────────┼────────┼─────┼───────────┼──────┼──────────┤
│ app_name │ 0  │ cluster │ 16  │ online │ 0       │ 11D    │ 0%  │ 84.5 MB   │ node │ disabled │
└──────────┴────┴─────────┴─────┴────────┴─────────┴────────┴─────┴───────────┴──────┴──────────┘
 Use `pm2 show <id|name>` to get more details about an app

```

不過我們不是用這種方法管理我們的程式的，原因是：**我們不想要把整個 Image 放到 Docker Cloud 上**。雖然 Build 成 Image 很方便，也可以確保每一次的更新都是一個新的 Image，不過最後還是因為隱私的關係作罷。那我們怎麼做？我們只有上傳上面那份 Dockerfile 產生的 image，這個 image 只有空殼，沒有真正 build 出來的程式碼。之後再利用 volume 讓 Docker 看到 build 出來的檔案，最後使用的 Service 以及 Stack 來啟動我們的 service。

## 使用 Docker Service 和 Stack 達到 Zero Downtime 部署
Docker 上的 [Service 教學](https://docs.docker.com/get-started/part3/)、[Stack 教學](https://docs.docker.com/get-started/part5/)都還算清楚，我也是按照教學拼拼湊湊的。要使用 Service，要先建立 `docker-compose.yml`，這個檔案幫你管理你所需要的所有 Docker image。假設今天你的 service 需要 nginx + mongodb + nodejs 的環境，如果手動起三個不同的 image 又要根據不同的 image 給不同的參數，非常不容易管理，看起來也很凌亂，`docker-compose.yml` 就是幫你把所有你需要的 docker image 都列出來，之後只要執行簡單的 `docker servie on` 或是 `docker stack deploy -c docker-compose.yml SERVICE_NAME` 就可以一次起多個 image，另外還順便幫你把環境都設定好。我們的 service 目前只有起一個 image，看一下我們用的設定檔：

```yml
version: '3'
services:
  website:
    image: USER_NAME/REPO_NAME:TAG
    volumes:
      - "SOURCE_DIRECTORY_HOST:DESTINATION_DIRECTORY_DOCKER"
    ports:
      - "3000:3000"
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 20s
```

要讓 docker 可以執行 service，需要先把這個 node 變成 Swarm 模式：

```bash
## 初始化這台機器成為 swarm node
docker swarm init
```

這時候要部署，有兩三種情況：
1. 程式碼修改，重新 build
2. docker-compose.yml 檔案修改，重新 deploy
3. ecosystem.config.js 檔案修改，重新啟動 service

第一種情況，不需要重新執行任何指令，原因是透過 volume 把 build 好的程式碼關連到 docker 中，docker 會自動更新，網頁也會自動刷新。

第二種情況，需要執行下面指令來重新 deploy 啟動中的 service，這邊可以看一下 `docker-compose.yml` 中的 `deploy` 的部分，只有下 deploy 指令，才會根據這邊的設定來執行，上面的設定是說

- Deploy 2 個 replicas。
- 一次更新一個 replica，並且間隔 20 秒在再更新下一個。

```bash
## 根據 docker-compose.yml 中的 deploy 部分 deploy stack
docker stack deploy -c ./cicd-utils/docker-compose.yml SERVICE_NAME

## 檢查 stack 狀態
docker stask ls

NAME                SERVICES
SERVICE_NAME        1

```

第三種情況，需要重新啟動 docker 的 service，會重新下載 Docker Hub 上面的 image，並且更新，

```bash
## 強制重新啟動 service
docker service update --image USER_NAME/REPO_NAME:TAG --force SERVICE_NAME_website

## 檢查 service 狀態
docker service ls

ID            NAME                  MODE        REPLICAS  IMAGE                    PORTS
m0dwq97pf7vp  SERVICE_NAME_website  replicated  2/2       USER_NAME/REPO_NAME:TAG  *:3000->3000/tcp
```

使用上面兩行指令，就可以做到 zero downtime deploy 了，可以在部署的時候下 `docker service ls`，會明顯看到有一個 service 先被關掉了，但是服務還可以連到。

其實透過 Docker 掛載 volume，利用 service 以及 stack 啟動服務還蠻直覺的，管理上也很方便，結合 Jenkins，就可以把 CI/CD 都連在一起。這次算是第一次自己搞 DevOps 的工作，學到很多，如果設定有疑問可直接留言，看到都會回。





