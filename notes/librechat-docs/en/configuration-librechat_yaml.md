---
title: Custom Config
icon: BookOpen
description: Create, mount, and configure the librechat.yaml file for custom AI endpoints and advanced LibreChat settings
---

> [!NOTE]
> **官方原文副本** — 線上版：<https://www.librechat.ai/docs/configuration/librechat_yaml>（中文版：<https://www.librechat.ai/zh/docs/configuration/librechat_yaml>）
> 原始檔：[LibreChat-AI/librechat.ai `content/docs/configuration/librechat_yaml/index.mdx`](https://github.com/LibreChat-AI/librechat.ai/blob/408527900701f755d2817a02d0c9c6befe0f57df/content/docs/configuration/librechat_yaml/index.mdx)，commit `4085279`，2026-11-25 下載。
> 標示 **📝 本機附註** 的區塊為本機補充，其餘為原文。本機依這頁的 Setup 步驟完成掛載：[issue #6](https://github.com/JacobHsu/LibreChat/issues/6)。

## What is librechat.yaml?

The `librechat.yaml` file is LibreChat's main configuration file for custom AI endpoints, model settings, interface options, and advanced features like MCP servers and agents. It is optional -- LibreChat works with sensible defaults if the file does not exist.

Follow the steps below to create the file, mount it for your deployment type, and verify it works.

<Callout type="info" title="If you only remember one thing">

For Docker installs, editing `librechat.yaml` is not enough. The file must exist in the project root, be mounted into the API container, and LibreChat must be restarted before changes appear in the UI.

</Callout>

<Callout type="info" title="Prefer a UI? Use the Admin Panel">

The [**LibreChat Admin Panel**](/docs/features/admin_panel) manages this same configuration from a browser -- including per-role and per-group overrides that take effect at login without restarting LibreChat. It ships with the official Docker Compose stacks. Use `librechat.yaml` for file-driven or bootstrap setup, and the admin panel for ongoing management.

</Callout>

## Setup

<Steps>
  <Step>

### Locate or Create the File

Create a new `librechat.yaml` in your project root (the same directory as your `.env` file):

```bash
touch librechat.yaml
```

You can also copy the [example config](/docs/configuration/librechat_yaml/example) as a starting point:

```bash
cp librechat.example.yaml librechat.yaml
```

<Callout type="info" title="Alternative File Path">

You can set a custom file path using the `CONFIG_PATH` environment variable:

```bash filename=".env"
CONFIG_PATH="/alternative/path/to/librechat.yaml"
```

</Callout>

> [!NOTE]
> **📝 本機附註：選了第三種做法——最小設定**
> - 官方給兩種：`touch` 一個空檔，或 `cp librechat.example.yaml`。本機兩種都沒用，而是只寫兩行，與下方「Example: Adding OpenRouter」的開頭相同：
>   ```yaml
>   version: 1.3.19
>   cache: true
>   ```
> - 不複製完整範例的原因：`librechat.example.yaml` 有 1932 行，而且已啟用 xAI、Groq、Mistral、OpenRouter 等需要 API key 的自訂 endpoint（`apiKey: '${XAI_API_KEY}'` 等）。這一步只想驗證「掛載是否成功」，照抄範例會把它和「endpoint 缺 key」混在一起。
> - 範例檔仍是最好的參考：要加哪個設定，就從範例複製那一段過來。

  </Step>
  <Step>

### Mount the Config File

<Tabs items={['Docker', 'Local']}>
  <Tabs.Tab>

Docker needs a volume mount to access your `librechat.yaml` file inside the container.

**Copy the example override file:**

```bash
cp docker-compose.override.yml.example docker-compose.override.yml
```

**Edit `docker-compose.override.yml`** and ensure the librechat.yaml volume mount is uncommented:

```yaml filename="docker-compose.override.yml"
services:
  api:
    volumes:
      - type: bind
        source: ./librechat.yaml
        target: /app/librechat.yaml
```

This uses the [docker-compose.override.yml](/docs/configuration/docker_override) pattern -- Docker Compose automatically merges it with the main `docker-compose.yml`, so your customizations survive updates.

> [!NOTE]
> **📝 本機附註：怎麼從範例檔「取消註解」**
> - `docker-compose.override.yml.example` 全部都是註解。照檔頭說明，只取消註解兩處：第 24 行的 `services:`，以及第 26-31 行「USE LIBRECHAT CONFIG FILE」那一段。其他段落（LOCAL BUILD、MONGO-EXPRESS 等）維持註解。
> - 取消註解後的有效內容只有：
>   ```yaml
>   services:
>     api:
>       volumes:
>       - type: bind
>         source: ./librechat.yaml
>         target: /app/librechat.yaml
>   ```
> - 啟動前用 `docker compose config` 檢查合併結果：api 原本的 6 個掛載（`.env`、`images`、`uploads`、`logs`、`skill`、`librechat-data`）都還在，`librechat.yaml` 被**附加**成第 7 個。這就是 Docker Override 頁說的「清單是附加，不是取代」。
> - 在 Git Bash 用 `sed -i` 編輯會把檔案的 CRLF 換行改成 LF，之後 `diff` 會顯示整份檔案都不同；比對時加 `diff --strip-trailing-cr`。compose 兩種換行都能讀。

  </Tabs.Tab>
  <Tabs.Tab>

Place `librechat.yaml` in the project root directory (the same directory as your `.env` file). No additional mounting is needed for local installations.

  </Tabs.Tab>
</Tabs>

  </Step>
  <Step>

### Restart LibreChat

<Tabs items={['Docker', 'Local']}>
  <Tabs.Tab>

```bash
docker compose down && docker compose up -d
```

  </Tabs.Tab>
  <Tabs.Tab>

Stop the running process (Ctrl+C) and restart:

```bash
npm run backend
```

  </Tabs.Tab>
</Tabs>

  </Step>
  <Step>

### Verify It Works

Open LibreChat in your browser. If your configuration includes custom endpoints, you should see them in the model selector dropdown.

If the server fails to start, check the logs for validation errors:

```bash
docker compose logs api
```

> [!NOTE]
> **📝 本機附註：實際驗證結果**
> 
> 官方的驗證是「在模型選單看到自訂 endpoint」。最小設定沒有 endpoint，所以改用下面幾個方法確認：
> 
> | 檢查 | 結果 |
> |---|---|
> | api log 的 `ENOENT ... /app/librechat.yaml` | 消失 |
> | api log 的 `Outdated Config version: undefined` | 消失，代表有讀到 `version` |
> | api log 新出現的那一行 | `Custom config file loaded (digest: 14c3bfd16e0f)` |
> | 容器內 `/app/librechat.yaml` | 與主機的檔案內容一致 |
> | `docker compose down` 之後資料是否還在 | 還在：`users=1`，Admin Panel 仍是登入狀態 |
> 
> **確認 bind mount 會即時反映主機的檔案**：
> 1. 在主機的 `librechat.yaml` 加上：
>    ```yaml
>    interface:
>      customWelcome: 'Mount test (issue #6): edited on the host'
>    ```
> 2. 只執行 `docker compose restart api`，不用 `down`。
> 3. 結果：
>    - digest 變成 `9f1aa0fa25cf`。
>    - 登入後的新對話頁顯示這段歡迎詞。
>    - 把那兩行刪掉再重啟，digest 又變回 `14c3bfd16e0f`。digest 是檔案內容的雜湊，內容相同就得到相同的 digest，可以用它快速判斷 LibreChat 讀到的是哪個版本。
> 
> **陷阱**：沒登入時用 `curl http://localhost:3080/api/config` 看不到 `interface.customWelcome`，但設定其實已經生效。前端是在登入後從 `startupConfig.interface.customWelcome` 讀取（`client/src/components/Chat/Landing.tsx`），所以要在登入狀態下看實際畫面才準。

  </Step>
</Steps>

## Example: Adding OpenRouter

This example walks through adding [OpenRouter](https://openrouter.ai/) as a custom endpoint -- one of the most popular configurations.

**1. Get an API key** from [openrouter.ai/keys](https://openrouter.ai/keys).

**2. Add the key to your `.env` file:**

```bash filename=".env"
OPENROUTER_KEY=sk-or-v1-your-key-here
```

<Callout type="warning" title="Environment Variable Name">

Use `OPENROUTER_KEY`, not `OPENROUTER_API_KEY`. Using `OPENROUTER_API_KEY` will override the OpenAI endpoint to use OpenRouter as well.

</Callout>

**3. Add the endpoint to `librechat.yaml`:**

```yaml filename="librechat.yaml"
version: 1.3.19
cache: true
endpoints:
  custom:
    - name: "OpenRouter"
      apiKey: "${OPENROUTER_KEY}"
      baseURL: "https://openrouter.ai/api/v1"
      models:
        default: ["meta-llama/llama-3-70b-instruct"]
        fetch: true
      titleConvo: true
      titleModel: "meta-llama/llama-3-70b-instruct"
      dropParams: ["stop"]
      modelDisplayLabel: "OpenRouter"
```

**4. Restart LibreChat** (see restart commands above) and select OpenRouter from the model selector.

For the full annotated config file with more endpoint examples, see the [example configuration](/docs/configuration/librechat_yaml/example).

## Reference

For detailed field-level documentation, see the reference pages below.

<Cards num={2}>
  <Cards.Card title="AI Endpoints" href="/docs/configuration/librechat_yaml/ai_endpoints" arrow>
    Compatible AI providers and example endpoint configurations
  </Cards.Card>
  <Cards.Card
    title="Object Structure"
    href="/docs/configuration/librechat_yaml/object_structure/config"
    arrow
  >
    Complete field reference for every librechat.yaml option
  </Cards.Card>
</Cards>

## Troubleshooting

### Change Does Not Show in LibreChat

If you edited `librechat.yaml` and nothing changed in the UI:

> [!NOTE]
> **📝 本機附註：** 第 5 點的 `/toolkit/yaml_checker` 連結用底線，實際網址是 <https://www.librechat.ai/toolkit/yaml-checker>。另外，改 `librechat.yaml` 只需要 `docker compose restart api`；第 3 點的 `down && up -d` 是改了 override 檔時才需要。

1. Confirm the file is in the LibreChat project root unless you set `CONFIG_PATH`.
2. For Docker, confirm the file is mounted in `docker-compose.override.yml`.
3. Restart LibreChat with `docker compose down && docker compose up -d`.
4. Check the API logs with `docker compose logs api`.
5. Validate the file with the [YAML Validator](/toolkit/yaml_checker).

Custom endpoints such as OpenRouter only appear after all three pieces are correct: `.env` contains the key, `librechat.yaml` defines the endpoint, and Docker can read the mounted config file.

### Configuration Validation

<Callout type="error" title="Configuration Validation">

LibreChat exits with an error (exit code 1) if `librechat.yaml` contains validation errors. This fail-fast behavior catches configuration issues early.

To validate your YAML syntax before restarting, use the [YAML Validator](/toolkit/yaml_checker) or [yamlchecker.com](https://yamlchecker.com/).

</Callout>

### Server Exits Immediately on Startup

If your server exits immediately after starting, this is likely a configuration validation error.

**To diagnose:**

1. Check server logs: `docker compose logs api`
2. Validate your YAML syntax with the [YAML Validator](/toolkit/yaml_checker)
3. Common errors: incorrect indentation, missing colons, invalid values, and unknown keys. An unknown or misspelled top-level key fails validation and stops the server. Inside nested blocks, strict ones (such as `telemetry`) reject unknown keys the same way, while others silently ignore them

**Temporary workaround** (not recommended for production):

```bash filename=".env"
CONFIG_BYPASS_VALIDATION=true
```

<Callout type="warning" title="Warning">

`CONFIG_BYPASS_VALIDATION=true` causes the server to skip validation and use default configuration. Always fix the validation errors instead.

</Callout>
