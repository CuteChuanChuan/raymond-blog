+++ 
draft = false
date = 2024-01-07T14:10:41+08:00
title = "結合 Cloud Run 和 GitHub Actions，自動部署 FastAPI server 到 Cloud Run 上"
description = ""
slug = ""
authors = []
tags = ["Google Cloud (GC)", "Cloud Run", "Serverless", "GitHub Actions", "CI/CD", "Docker", "Python"]
categories = []
externalLink = ""
series = []
+++

# 前言
設想一個情況：你製作了一個網站，今天你開發了一個很酷的新功能，想要趕快將程式碼更新到 server 上，讓使用者可以趕快享受這個新功能。  
但是當你連線到 server 時，卻發現 server 不知道發生什麼問題，導致你需要先花費大量的時間來處理 server 的問題，唯有解決 server 的問題才能更新程式碼。
這樣的狀況是自己維運 server 會遇到的問題，如果你想要讓自己專注在開發上，不想要再將時間花在 server 的維運上，那麼你可以考慮使用 Cloud Run，一個由 Google Cloud 的全代管 Serverless 服務。

# 何謂 Serverless 服務
由雲端服務的供應商 (e.g., AWS & GC) 處理與維運基礎架構，進而讓開發人員可以專注在開發系統核心功能上，不在需要花時間處理任何基礎設施。
一言以蔽之，Serverless 服務的目標就是讓開發人員專注在應用，而非架構。

# [Cloud Run 介紹](https://cloud.google.com/run?hl=en)
Cloud Run 是 Google Cloud 的全代管 Serverless 服務，它可以幫助開發人員在開發新的功能後快速部署。
它結合 Serverless 的特色與容器化的特色：(1) 根據需求自動擴展、(2) 用多少付多少。
另外，它具有額外的特色：(1) 可以和其他 GC 服務搭配，像是 Cloud Build & Artifact Registry、(2) 內建的 load balancer，可以自動導流流量、(3) 有提供 HTTPS 端點供使用者立即可以使用。

# Cloud Run 部署方式
Cloud Run 有兩種部署方式。第一種是手動，由開發人員自己建 image → 上傳 Artifact Registry → 在 Cloud Run 手動選擇 image 來部署。
第二種是自動，開發人員可以使用 GitHub Actions 來達到 CI/CD，而這也是這篇文章想要分享的部署方式。

# 結合 GitHub Actions & Cloud Run
## 前置準備
1. 首先要先在 GitHub 上面新增一個 Repo。
2. 電腦安裝 gcloud 並登入 Google Cloud 帳號，即可用command line 操控 Google Cloud (參考[此文](https://pythonviz.com/google-cloud/macos-install-google-cloud-sdk/))。

## 步驟
1. [Google Cloud] 新增一個專案
2. [Google Cloud] 處理 Auth Step 1：新增服務帳號、使用專案的 Workload Identity Federation 來進行 GitHub Actions 驗證。(讀者可以參考[這篇文章的步驟一](https://jcshawn.com/cloud-run-with-github-action/)，有更詳細的指令說明)
3. [Google Cloud] 處理 Auth Step 2：針對剛剛建立的服務帳號授予所需權限，讓這個服務帳號可以上傳 docker image 到 Artifact Registry，並且可以從 Artifact Registry 讀取 image，並且部署到 Cloud Run
   - 至 [Google Cloud IAM 與管理頁面](https://console.cloud.google.com/iam-admin/iam?authuser=0&hl=zh-TW&project=cloudrun-ghactions)，記得要確認是否是剛剛新建的那個專案
   - 身分與存取權管理 -> "授予存取權" -> 新增主體的部分填入剛剛新建立的服務帳號 -> 指派角色 (新增：服務帳戶使用者、Artifact Registry 寫入者、Artifact Registry 讀取者、Cloud Run 管理員)
4. [Local] 準備程式碼與 Dockerfile (requirements.txt 可見[此檔案](https://github.com/CuteChuanChuan/__blog__CloudRun.GHActions/blob/main/requirements.txt)，FastAPI server 架構可見下方示意)
   (我有建立一個 [demo repo](https://github.com/CuteChuanChuan/__blog__CloudRun.GHActions)，讀者也可以直接 clone repo)

```shell
.
├── app
│   ├── __init__.py
│   └── main.py
├── Dockerfile
└── requirements.txt
```

```python
# main.py
from fastapi import FastAPI
from typing import Union

app = FastAPI()


@app.get("/")
def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
def read_item(item_id: int, q: Union[str, None] = None):
    return {"item_id": item_id, "q": q}
```

```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /code
COPY ./requirements.txt /code/requirements.txt
RUN pip install --no-cache-dir --upgrade -r /code/requirements.txt
COPY ./app /code/app
# 常見的形式是：CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80"]
# 但因 Cloud Run 預設的 Port 是 8080，且會當作環境變數傳入到 build image 的過程中
# 若採用上方寫法會產生問題，所以需寫成下方的形式，$PORT 會去吃在 build image 時的環境變數 Port 
CMD uvicorn app.main:app --host 0.0.0.0 --port $PORT
```
5. [Google Cloud] 至 Artifact Registry 新增一個 repo 來放 image (後面的 yaml 檔會用到 repo 名稱)
6. [GitHub] 將機密資訊設定成 GitHub repo 的 secret，讓 GitHub Actions 可以使用這些機密資訊，同時避免直接寫在 yaml 上，造成資安問題  
   至前置準備 step 1 創立的 repo -> (上方 tab) Setting -> (左側 menu) Secrets and variables → Actions
   點擊 New repository secret 來新增兩個 secret：  
   (a) WIF_PROVIDER：步驟 2 最後得到的 Provider 資源名稱  
       (類似 projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider)  
   (b) WIF_SERVICE_ACCOUNT：步驟 2 最後得到的 Provider 資源名稱  
       (類似 github-service-account@你的專案名稱.iam.gserviceaccount.com)  
       (與此設定有關：```gcloud iam service-accounts create "github-service-account" --project "${PROJECT_ID}"```)
7. [Local] 準備 GitHub Actions 需要的 yaml 檔 (可參考[官方提供的 yaml 檔](https://github.com/google-github-actions/example-workflows/blob/main/workflows/deploy-cloudrun/cloudrun-docker.yml)，但其中有些地方需更新)  
   (可直接在 GitHub 上面設定，也可在 local 寫好之後 push 到 main branch，亦或是 pull request 到 main)
```yaml
name: Build and Deploy to Cloud Run

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  PROJECT_ID: 請放入你的 Google Cloud 專案 ID
  GAR_LOCATION: asia-east1 (依讀者需求調整 Artifact Registry 的存放位置，台灣是 asia-east1)
  SERVICE: 請放入你在步驟 5 建立的 repo 名稱
  REGION: asia-east1 (依讀者需求調整 Cloud Run 的位置，台灣是 asia-east1)

jobs:
  build-and-deploy:
    permissions:
      contents: 'read'
      id-token: 'write'

    runs-on: ubuntu-latest
    steps:

      - name: Checkout
        uses: actions/checkout@v2
        
      # 官方提供的版本較舊：'google-github-actions/auth@v0'
      - name: Google Auth
        id: auth
        uses: 'google-github-actions/auth@v2'
        with:
          token_format: 'access_token'
          workload_identity_provider: '${{ secrets.WIF_PROVIDER }}' # e.g. - projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider
          service_account: '${{ secrets.WIF_SERVICE_ACCOUNT }}' # e.g. - my-service-account@my-project.iam.gserviceaccount.com

      # 官方提供的版本較舊：'docker/login-action@v1'
      - name: Docker Auth
        id: docker-auth
        uses: 'docker/login-action@v3'
        with:
          username: 'oauth2accesstoken'
          password: '${{ steps.auth.outputs.access_token }}'
          registry: '${{ env.GAR_LOCATION }}-docker.pkg.dev'

      # 官方提供的路徑有錯，原為 /${{ env.SERVICE }}:${{ github.sha }}，應為 /${{ env.SERVICE }}/${{ env.SERVICE }}:${{ github.sha }}
      - name: Build and Push Container
        run: |-
          docker build -t "${{ env.GAR_LOCATION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.SERVICE }}/${{ env.SERVICE }}:${{ github.sha }}" ./
          docker push "${{ env.GAR_LOCATION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.SERVICE }}/${{ env.SERVICE }}:${{ github.sha }}"
      
      # 官方提供的路徑有錯，原為 /${{ env.SERVICE }}:${{ github.sha }}，應為 /${{ env.SERVICE }}/${{ env.SERVICE }}:${{ github.sha }}
      # 官方提供的版本較舊：uses: google-github-actions/deploy-cloudrun@v0
      - name: Deploy to Cloud Run
        id: deploy
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: ${{ env.SERVICE }}
          region: ${{ env.REGION }}
          image: ${{ env.GAR_LOCATION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.SERVICE }}/${{ env.SERVICE }}:${{ github.sha }}

      - name: Show Output
        run: echo ${{ steps.deploy.outputs.url }}

```
8. 確認 GitHub Actions 運行的狀況 ![Imgur](https://imgur.com/KRPkVn0.png)
9. [Google Cloud] 至 Cloud Run 點擊部署成功的服務 -> (上方 tab) 安全性 -> 驗證 -> 允許在未經驗證的情況下叫用 (開放給其他人可以直接連線)
10. 大功告成！可以點擊 Cloud RUn 上面的網址來確認！![Imgur](https://imgur.com/jfgWABf.png)


# 參考文章
1. [傳統伺服器 V.S. 無伺服器！如何找出適合自己的架構？](https://www.ecloudture.com/the-difference-between-traditional-server-and-serverless/)
2. [什麼是 Serverless 架構？優勢劣勢為何？](https://datadrivenai.wordpress.com/2019/10/08/%E4%BB%80%E9%BA%BC%E6%98%AF-serverless-%E6%9E%B6%E6%A7%8B%EF%BC%9F%E5%84%AA%E5%8B%A2%E5%8A%A3%E5%8B%A2%E7%82%BA%E4%BD%95%EF%BC%9F/)
3. [提高開發效率！用 GitHub Action 自動部署 Docker 容器到 Cloud Run 教學](https://jcshawn.com/cloud-run-with-github-action/)
4. [Cloud Run 是什麼？6大特色介紹與實作教學](https://blog.cloud-ace.tw/application-modernization/serverless/cloud-run-overview-and-tutorial/)
5. [[GCP] Cloud Run 六個實作: 解析麻雀雖小五臟俱全 | Cloud Run is Small but Complete](https://joehuang-pop.github.io/2020/05/01/GCP-Cloud-Run-%E5%85%AD%E5%80%8B%E5%AF%A6%E4%BD%9C-%E8%A7%A3%E6%9E%90%E9%BA%BB%E9%9B%80%E9%9B%96%E5%B0%8F%E4%BA%94%E8%87%9F%E4%BF%B1%E5%85%A8-Cloud-Run-is-Small-but-Complete/)