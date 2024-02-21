+++ 
draft = false
date = 2024-02-21T21:10:41+08:00
title = "手動部署到 Cloud Run 上"
description = ""
slug = ""
authors = []
tags = ["Google Cloud (GC)", "Cloud Run", "Serverless", "Docker", "Python"]
categories = []
externalLink = ""
series = []
+++

# 前言
在[先前的文章]({{< ref "/posts/cloudrun_gh_Actions/index.md" >}})中，分享了如何使用 GitHub Actions 進行 CI/CD，來自動化整個部署流程。
除了自動之外，手動部署也是另外一種體驗。這篇文章想和大家分享手動建立 Docker Image 後部署到 Cloud Run 的流程。
（更多關於 Cloud Run 的介紹可以參考[先前的文章]({{< ref "/posts/cloudrun_gh_Actions/index.md" >}})）

# 手動部署到 Cloud Run
## 步驟
1. [Local] 準備程式碼與 Dockerfile，並打包成 Docker Image
   - 打包時要注意作業系統，Cloud Run 要求：linux/amd64
   - reference 1: [official document](https://cloud.google.com/run/docs/container-contract#languages)
   - reference 2: [stack overflow](https://stackoverflow.com/a/68766137/21803201)
```dockerfile
# Dockerfile
FROM python:3.11
RUN apt-get update && apt-get install -y \
    gcc \
    libc-dev \
    libffi-dev \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /server
COPY . .
RUN pip install -r requirements.txt
CMD ["gunicorn", "main:app", "-k", "uvicorn.workers.UvicornWorker", "-w", "4", "--bind", "0.0.0.0:8000"]
```

```shell
docker build -t image_name --platform linux/amd64 -f Dockerfile .
```

2. 給予剛剛建立的 image 一個 tag ([document](https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling#tag))

```shell
docker tag image:v1 [your-region]-docker.pkg.dev/[your-project-id]/[repo-on-registry]/[image_name]
```

3. 上傳至 artifact registry

```shell
docker push [your-region]-docker.pkg.dev/[your-project-id]/[repo-on-registry]/[image_name]
```

4. 部署
   1. 至 Cloud Run 頁面，選擇「建立服務」![Imgur](https://imgur.com/aV7lJTz.png)
   2. 如果有上傳 artifact registry 成功，就可以去拉剛剛上傳的 image ![Imgur](https://imgur.com/cAGM6lL.png)
   3. 預設會聽 PORT 8080，如果有特殊需要可以修改 ![Imgur](https://imgur.com/BNXUyU0.png)
   4. 疑難排解 - [Official Document](https://cloud.google.com/run/docs/troubleshooting#container-failed-to-start)

# 後續可能遇到的問題 (會再另外寫文章與大家分享~)
1. 若有使用其他工具 (e.g., MongoDB Atlas)，可能會限制存取的 IP，那要怎麼辦？  
   -> Serverless VPC Access + VPC + Cloud NAT
2. 若需要套上自己的網域名稱，那要怎麼辦？  
   -> Domain Name + Load Balancer