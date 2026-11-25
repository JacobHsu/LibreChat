---
title: Docker Override
icon: Docker
description: Use a docker-compose.override.yml file to customize LibreChat's Docker setup without editing the main docker-compose.yml.
---

> [!NOTE]
> **官方原文副本** — 線上版：<https://www.librechat.ai/docs/configuration/docker_override>（中文版：<https://www.librechat.ai/zh/docs/configuration/docker_override>）
> 原始檔：[LibreChat-AI/librechat.ai `content/docs/configuration/docker_override.mdx`](https://github.com/LibreChat-AI/librechat.ai/blob/408527900701f755d2817a02d0c9c6befe0f57df/content/docs/configuration/docker_override.mdx)，commit `4085279`，2026-11-25 下載。
> 標示 **📝 本機附註** 的區塊為本機補充，其餘為原文。
> 狀態：本機**尚未建立** override，目前完全使用官方 `docker-compose.yml`。

A Docker Compose override file lets you change the default configuration in `docker-compose.yml` without editing or duplicating it. Override files are mainly for local customizations. When you run `docker compose up`, Compose merges `docker-compose.yml` with `docker-compose.override.yml` automatically.

<Callout type="info" title="More examples">
See `docker-compose.override.yml.example` in the repository for a fuller set of override snippets you can copy from.
</Callout>

## Configure the Override

<Steps>
<Step>

**Create the override file.** If you don't already have one, copy the example. Docker Compose picks it up automatically when you run `docker compose` commands.

```bash filename="terminal"
cp docker-compose.override.yml.example docker-compose.override.yml
```

</Step>
<Step>

**Edit the override file.** Open `docker-compose.override.yml` in your editor, then uncomment and customize the sections you need.

<Callout type="warning" title="One entry per service">
Each service name (`api`, `mongodb`, `meilisearch`, ...) can appear only once. To override multiple settings on a single service, combine them under that one entry.
</Callout>

</Step>
<Step>

**Apply the changes.** Run Docker Compose as usual. It merges `docker-compose.yml` and `docker-compose.override.yml` for you.

```bash filename="terminal"
docker compose up -d
```

</Step>
<Step>

**Verify the changes.** List the running containers and their properties, such as ports, to confirm your overrides took effect.

```bash filename="terminal"
docker ps
```

</Step>
</Steps>

## Examples

To mount your `librechat.yaml` config file so Docker can use it for [Custom Endpoints & Configuration](/docs/configuration/librechat_yaml):

```yaml filename="docker-compose.override.yml"
services:
  api:
    volumes:
      - ./librechat.yaml:/app/librechat.yaml
```

To build the `api` image locally, mount the config file, and use an older MongoDB that doesn't require AVX support:

```yaml filename="docker-compose.override.yml"
services:
  api:
    volumes:
      - ./librechat.yaml:/app/librechat.yaml
    image: librechat
    build:
      context: .
      target: node

  mongodb:
    image: mongo:4.4.18
```

<Callout type="warning" title="Watch exposed ports">
Exposing MongoDB or Meilisearch ports to the public can leave your data vulnerable. Avoid default ports for production or sensitive environments.
</Callout>

> [!NOTE]
> **📝 本機附註：override 的合併規則與本機用法**
> - **短語法 vs 長語法**：上面範例用短語法 `./librechat.yaml:/app/librechat.yaml`，來源檔不存在時 Docker 會建立同名**資料夾**；[local-docker.md](local-docker.md) 用的長語法 `type: bind` 則會直接報錯。先建立 `librechat.yaml` 再啟動，兩種都可以。
> - **清單是「附加」不是「取代」**：`volumes`、`ports` 這類清單會跟主檔合併。例如在 override 加 `ports` 不會移除主檔的 `"${PORT}:${PORT}"`；要取代得用 `!override`（見 local-docker.md 的 Port Already in Use）。
> - **build 範例的 `image: librechat`**：從原始碼建置時給映像取本機名稱。別取成跟 registry 一樣的 `librechat-dev`，避免跟官方映像混淆。
> - **GUI 連 MongoDB**：預設 MongoDB 不對外開 port（且以 `--noauth` 執行），要連 Compass 等工具時只綁 127.0.0.1：
>   ```yaml
>   services:
>     mongodb:
>       ports:
>         - "127.0.0.1:27017:27017"
>   ```
> - **同一台機器跑多套 LibreChat**：主檔的 `container_name` 是寫死的，第二套要在 override 逐一改名（本機 `libre-chat-sep` 的做法）：
>   ```yaml
>   services:
>     api:            { container_name: LibreChat-sep }
>     admin-panel:    { container_name: admin-panel-sep }
>     mongodb:        { container_name: chat-mongodb-sep }
>     meilisearch:    { container_name: chat-meilisearch-sep }
>     vectordb:       { container_name: vectordb-sep }
>     rag_api:        { container_name: rag_api-sep }
>   ```
>   再搭配 `.env` 的 `PORT`、`ADMIN_PANEL_PORT` 換 port。
> - 檢查合併後的最終設定：`docker compose config`，比官方的 `docker ps` 更早發現錯誤（不用真的啟動）。
> - `docker-compose.override.yml` 已被 `.gitignore` 排除，不會進 git。

## Using `deploy-compose.yml`

With a non-default Compose file such as `deploy-compose.yml`, the override is not loaded automatically. Pass both files explicitly with `-f` (or `--file`); settings in later files override or add to earlier ones.

The override file can have any name, though you may already have `docker-compose.override.yml` in place. Run commands like so:

```bash filename="terminal"
docker compose -f deploy-compose.yml -f docker-compose.override.yml pull
docker compose -f deploy-compose.yml -f docker-compose.override.yml up
```

## Reference

- **Order of precedence:** values in the override file take precedence over the same values in `docker-compose.yml`.
- **Security:** when customizing ports and exposing services publicly, be conscious of the security implications and avoid defaults for production.

For more detail, see the official Docker documentation:

- [Understanding multiple Compose files](https://docs.docker.com/compose/how-tos/multiple-compose-files/extends/)
- [Merge Compose files](https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/)
- [Specifying multiple Compose files](https://docs.docker.com/compose/reference/#specifying-multiple-compose-files)
