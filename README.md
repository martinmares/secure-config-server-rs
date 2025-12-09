# Secure Config Server

A small Rust service that acts as a **secure, Spring Cloud Config-compatible server** with a few extra features:

* Serves configuration from a Git repository (local `file://` or remote).
* Mostly drop-in compatible with **Spring Cloud Config Server JSON endpoints**
  (`/{application}/{profile}` and `/{application}/{profile}/{label}`).
* Supports **labels as Git refs** (branches/tags/commits).
* Supports **search paths** via configurable `subpath` (like `search-paths` in Spring).
* Extra **file endpoint** for non-Spring clients (shell, Python, C++, …).
* Simple templating for text files using `{{ VAR }}` placeholders filled from environment variables.
* Optional **HTTP Basic Auth** using environment variables.
* Small HTML UI on `/ui` with current repo / branch / commit information.
* Structured logging via `tracing`.

---

## 1. High-level architecture

The server is stateless and reads all configuration from a Git repository.

* On startup it:
  * Parses `config.yaml`.
  * Clones or fetches the configured Git repository into a local `workdir`.
  * Builds a map of environment variables (including any secrets you exported before starting the server).
* In the background it periodically runs `git fetch` / `git reset --hard origin/<branch>` to keep `workdir` up to date.
* For each HTTP request it:
  * Resolves the Git **ref** (branch/tag/commit) based on the `label` parameter (or default branch if `label` is missing).
  * Reads the requested file(s) via `git show <ref>:<subpath>/<path>` - no local checkout switching.
  * For text files, applies `{{ VAR }}` templating using the pre-loaded environment map.
  * For Spring-style requests, flattens YAML into a JSON structure compatible with Spring Cloud Config.

Git is accessed via the `git` CLI, so **`git` must be installed** and on `PATH`.

---

## 2. Spring Cloud Config compatibility

The main goal is to behave like Spring Cloud Config for the JSON environment endpoints.

### 2.1 Endpoints

Supported JSON endpoints:

* `GET /{application}/{profile}`
* `GET /{application}/{profile}/{label}`

The response has the same shape as Spring Cloud Config’s `/env` endpoint:

```json
{
  "name": "config-client",
  "profiles": ["dev"],
  "label": "main",
  "version": "b4e0c15a536cc16b6b0c2a78c188bfaa3246704c",
  "state": "",
  "propertySources": [
    {
      "name": "git:file:///path/to/repo/config-client-dev.yml",
      "source": {
        "demo.message": "Hello from MAIN branch (dev profile)",
        "demo.number": "42"
      }
    }
  ]
}
```

Notes:

* `label` is:
  * `null` when no label is provided.
  * The string from the URL when provided (e.g. `release`).
* `version` is the **commit hash** (`git rev-parse`) of the resolved ref.
* `propertySources`:
  * Contains a single flattened source if any config files were found.
  * Is an **empty array** when no files are found - HTTP **200** with empty `propertySources` (like Spring), not 404.

### 2.2 Label → Git ref

The label is resolved as:

* If label is present:
  * try `<label>`
  * then `origin/<label>`
* If label is absent:
  * use the configured default branch `git.branch`
  * or `origin/<git.branch>`

All file reads are done using:

```bash
git -C <workdir> show <ref>:<subpath>/<path>
```

so multiple labels/branches can be requested concurrently.

### 2.3 Search paths (`subpath`)

Spring has `search-paths`. Here we use:

```yaml
git:
  subpath: "test"
```

This acts as the **root within the repo**. All file lookups behave as if the repo root was `<repo>/<subpath>`.

Example:

* repo: `file:///…/config.repo`
* `subpath: "test"`
* config file in Git: `test/config-client-dev.yml`
* request: `GET /config-client/dev`
* server will read `config-client-dev.yml` from `ref:test/config-client-dev.yml`.

---

## 3. Extra file endpoint

For non-Spring clients (shell, Python, C++, …) there is a generic file endpoint:

* `GET /file/{label}/{*path}`

Semantics:

* Uses the same label → Git ref resolution.
* Combines `subpath` and `{*path}` and reads via `git show`.

Content types:

* If the file looks like **binary** (null byte or non-UTF-8) → returns `application/octet-stream` (or guessed MIME) as raw bytes.
* If the file is **text**:
  * Applies templating: `{{ VAR_NAME }}` → replaced with value from environment (if present).
  * Returns text with a guessed MIME type (based on extension) or `text/plain`.

This allows usage like:

```bash
# Fetch a shell snippet with templated exports
curl -u "$AUTH_USERNAME:$AUTH_PASSWORD"   "http://config-server/file/main/env/test.env"   | bash
```

---

## 4. Templating

Any **text** file is processed before sending:

* Pattern: `{{ VAR_NAME }}` (double curly braces).
* The server uses a pre-built `HashMap<String, String>` of environment variables taken from the process environment on startup.
* Values are looked up by exact variable name.

Example in Git:

```yaml
spring:
  datasource:
    url: "{{ TSM_DB_URL }}?currentSchema=um"
    username: "{{ TSM_DB_USER }}"
    password: "{{ TSM_DB_PASSWORD }}"
```

If the process is started with:

```bash
export TSM_DB_URL=jdbc:postgresql://db/tsm
export TSM_DB_USER=tsm
export TSM_DB_PASSWORD=secret
```

then the server will return the YAML with these values substituted.

> The server **does not decrypt secrets itself**.
> If you have encrypted files (e.g. `env.secured.json`), decrypt them before starting the server and export them as environment variables.

---

## 5. Configuration (`config.yaml`)

Main configuration lives in `config.yaml`.

Example:

```yaml
git:
  repo_url: "file:///Users/you/dev/config-server-proxy.config.repo"
  branch: "main"
  workdir: "/Users/you/dev/config-server-proxy.config.wrk"
  subpath: "test"                # optional, acts like search-paths
  refresh_interval_secs: 30       # how often to fetch/reset

http:
  bind_addr: "127.0.0.1:8899"
  base_path: "/"                  # e.g. "/config" behind a reverse proxy
```

Fields:

### `git` section

* `repo_url` - Git URL, can be `file:///…`, `https://…`, `ssh://…` etc.
* `branch` - default branch to clone and use when `label` is not provided.
* `workdir` - local directory for the clone.
* `subpath` - optional, path within the repo that acts as config root (like `search-paths`).
* `refresh_interval_secs` - how often the background task runs `git fetch` and `git reset --hard origin/<branch>`.

### `http` section

* `bind_addr` - address and port to bind, e.g. `0.0.0.0:8080`.
* `base_path` - optional prefix path, e.g. `"/secure-config"`.
  All routes are then available under this prefix:
  * `/secure-config/{application}/{profile}`
  * `/secure-config/file/{label}/{*path}`
  * `/secure-config/ui`

---

## 6. Authentication

HTTP Basic Auth is optional and controlled purely by environment variables.

* If both are set:

  ```bash
  export AUTH_USERNAME="myuser"
  export AUTH_PASSWORD="mypassword"
  ```

  then Basic Auth is **required** for all endpoints (including `/ui`).

* If one or both are missing:

  * Basic Auth is **disabled** (anyone can access the server).

These env vars are **not** stored anywhere; they are only used in memory.

---

## 7. HTML UI (`/ui`)

A small dark-themed UI is available at:

* `GET /ui` (or `${base_path}/ui` if you use a prefix)

It shows:

* **Repository URL**
* **Branch**
* **Workdir**
* **Subpath**
* **Commit hash** (for the default branch/ref)
* **Commit date** (ISO-8601, committer date)

Authentication:

* Protected by the same Basic Auth mechanism:
  * username: `AUTH_USERNAME`
  * password: `AUTH_PASSWORD`

The UI is static HTML + Fomantic UI, with values injected at runtime using Rust’s `include_str!` and simple placeholder replacement.

---

## 8. Logging

Logging is implemented with [`tracing`](https://crates.io/crates/tracing) and `tracing-subscriber`:

* Default level is `info`.
* You can control it with `RUST_LOG`, for example:

```bash
RUST_LOG=info ./secure-config-server --config=config.yaml
RUST_LOG=debug ./secure-config-server --config=config.yaml
RUST_LOG=secure_config_server=debug,info ./secure-config-server --config=config.yaml
```

Git operations are logged at `info` (start) and `error` (on failure). Spring/file handlers log `warn`/`error` for invalid requests and internal errors.

---

## 9. Running locally

Prerequisites:

* A recent stable Rust toolchain (`cargo`).
* `git` installed and available on `PATH`.

### 9.1 Prepare a local Git repo

```bash
mkdir -p ~/dev/config-server-proxy.config.repo
cd ~/dev/config-server-proxy.config.repo
git init
git checkout -b main

cat > config-client-dev.yml <<'EOF'
demo:
  message: "Hello from MAIN branch (dev profile) {{FOO}}{{BAR}}"
  number: 42
EOF

git add config-client-dev.yml
git commit -m "Initial config"
```

Optional: create a second branch:

```bash
git checkout -b release
cat > config-client-dev.yml <<'EOF'
demo:
  message: "Hello from RELEASE branch (dev profile)"
EOF
git commit -am "Release config"
git checkout main
```

### 9.2 Create `config.yaml`

```yaml
git:
  repo_url: "file:///Users/you/dev/config-server-proxy.config.repo"
  branch: "main"
  workdir: "/Users/you/dev/config-server-proxy.config.wrk"
  # subpath: "test"
  refresh_interval_secs: 30

http:
  bind_addr: "127.0.0.1:8899"
  base_path: "/"
```

### 9.3 Build and run

```bash
cargo build
cargo run -- --config=config.yaml
```

With Basic Auth:

```bash
export AUTH_USERNAME=myuser
export AUTH_PASSWORD=mypass

cargo run -- --config=config.yaml
```

---

## 10. Testing endpoints

Assuming `AUTH_USERNAME=myuser`, `AUTH_PASSWORD=mypass`.

### 10.1 Spring-compatible JSON

```bash
# main branch
curl -s -u myuser:mypass http://127.0.0.1:8899/config-client/dev | jq

# explicit label "main"
curl -s -u myuser:mypass http://127.0.0.1:8899/config-client/dev/main | jq

# label "release" (branch/tag/commit)
curl -s -u myuser:mypass http://127.0.0.1:8899/config-client/dev/release | jq

# default profile (no config files) - must return 200 with empty propertySources
curl -s -u myuser:mypass http://127.0.0.1:8899/config-client/default | jq
```

### 10.2 File endpoint

```bash
# raw / templated text file
curl -s -u myuser:mypass   "http://127.0.0.1:8899/file/main/config-client-dev.yml"

# arbitrary file from label "release"
curl -s -u myuser:mypass   "http://127.0.0.1:8899/file/release/path/to/any.file"
```

### 10.3 UI

Open in browser:

```text
http://127.0.0.1:8899/ui
```

(or with your base path prefix)

---

## 11. Using with Spring Boot Config Client

You can point a Spring Boot app to this server in the same way as to a standard Spring Cloud Config Server.

Example (application side):

```yaml
spring:
  application:
    name: config-client

  profiles:
    active: dev

  config:
    import: "optional:configserver:http://myuser:mypass@localhost:8899"
```

Or via command line:

```bash
java -jar configclient.jar   --spring.profiles.active=dev   --spring.config.import="optional:configserver:http://myuser:mypass@localhost:8899"
```

Notes:

* Spring will first try the `default` profile; the server responds with 200 + empty `propertySources`, which is acceptable even without any `config-client.yml` in the repo.
* Then Spring loads `profiles='dev'` and merges `config-client-dev.yml`, `application.yml` etc., just like with the original Config Server.
* Labels in URLs (`/{application}/{profile}/{label}`) map to Git refs exactly as described above.

---

## 12. License

This project is licensed under the [MIT License](./LICENSE).
