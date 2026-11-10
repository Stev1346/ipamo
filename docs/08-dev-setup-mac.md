# 08 – Development Setup (macOS)

> **Status:** Draft for approval · **Applies to:** v1
> **Related:** [README](../README.md) (running the app) · [07 – Build plan](07-build-plan.md) (M0)

This is a one-time setup for a Mac (Apple Silicon or Intel) to build, run and test Ipamo. Every tool has a command that verifies it. Once all the checks in §9 pass, go to the README.

---

## 1. Summary

| Tool | Why | Install method |
| --- | --- | --- |
| Homebrew | Package manager for the command-line tools below | Script from brew.sh |
| Git + GitHub CLI | Source control; PRs from the terminal | Homebrew |
| .NET 10 SDK | Build, run and test the API | Microsoft installer (`.pkg`) |
| Docker Desktop | Local PostgreSQL and integration tests (Testcontainers) | Docker installer (`.dmg`) |
| `psql` | PostgreSQL command-line client | Homebrew (`libpq`) |
| VS Code + extensions | Editor, debugging, test explorer, DB browsing | Microsoft download + CLI |
| Claude Code | Senior-developer partner | Anthropic installer |
| Repo-local .NET tools | `dotnet-ef`, `reportgenerator` | `dotnet tool restore` (in the repo) |

## 2. Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# On Apple Silicon, follow the "Next steps" it prints to add brew to your PATH, then:
brew --version
```

## 3. Git and GitHub CLI

```bash
brew install git gh
git --version
gh --version

git config --global user.name  "Stephen Okanlawon"
git config --global user.email "<your GitHub email>"
git config --global init.defaultBranch main
gh auth login          # choose GitHub.com → HTTPS → log in with a browser
```

## 4. .NET 10 SDK

1. Download the **.NET 10 SDK** macOS installer from <https://dotnet.microsoft.com/download/dotnet/10.0>. Pick **Arm64** for Apple Silicon or **x64** for Intel.
2. Run the `.pkg`, then open a **new** terminal.

```bash
dotnet --list-sdks     # expect a 10.0.x line
dotnet --info          # "RID: osx-arm64" on Apple Silicon
```

**Why the official installer instead of Homebrew?** Microsoft's `.pkg` installs to `/usr/local/share/dotnet`, which is where VS Code's C# Dev Kit and other tools expect it. The repo's `global.json` pins the SDK feature band, so everyone builds with the same compiler.

**Trust the HTTPS development certificate** (one time; macOS asks for your password to update the Keychain):

```bash
dotnet dev-certs https --trust
dotnet dev-certs https --check --trust   # expect "A trusted certificate was found"
```

## 5. Docker Desktop

1. Download Docker Desktop for Mac from <https://www.docker.com/products/docker-desktop/>. Choose the Apple Silicon or Intel build.
2. Install it, start it, and wait until the whale icon in the menu bar shows "Docker Desktop is running."
3. **Settings → Advanced:** enable **"Allow the default Docker socket to be used."** Testcontainers looks for `/var/run/docker.sock`, and without this setting integration tests can fail to find Docker.
4. **Settings → Resources:** at least 4 GB of memory is plenty.

```bash
docker --version
docker compose version
docker run --rm hello-world          # prints "Hello from Docker!"
ls -l /var/run/docker.sock           # should exist
```

**Alternative:** OrbStack is a lighter Docker-compatible runtime for macOS, and Testcontainers works with it too. Docker Desktop is the default here because its documentation and troubleshooting coverage are wider. Run one or the other, not both.

## 6. PostgreSQL client (`psql`)

You don't need a local PostgreSQL *server*, because the database runs in Docker. You only need the client:

```bash
brew install libpq
brew link --force libpq             # puts psql on your PATH
psql --version                      # expect 17.x or newer
```

Once the repo's database is running (`docker compose up -d`), connect with:

```bash
psql "host=localhost port=5433 dbname=ipamo user=ipamo"
# then: \dt  (list tables)   \d equipment  (describe a table)   \q  (quit)
```

If you already have Postgres.app or a Homebrew PostgreSQL server on port 5432, it's fine. Ipamo uses **5433**, so they don't clash.

## 7. VS Code and extensions

1. Install VS Code from <https://code.visualstudio.com/>.
2. Open it, press **⌘⇧P**, and run **"Shell Command: Install 'code' command in PATH."**
3. Install the extensions:

```bash
code --install-extension ms-dotnettools.csdevkit         # C# Dev Kit: solution explorer, test explorer
code --install-extension ms-dotnettools.csharp           # C# language support (installed with Dev Kit)
code --install-extension ms-ossdata.vscode-pgsql         # PostgreSQL: browse tables, run queries
code --install-extension ms-azuretools.vscode-docker     # Containers: see and manage containers
code --install-extension EditorConfig.EditorConfig       # respects the repo's .editorconfig
code --install-extension humao.rest-client               # run .http request files
code --install-extension DavidAnson.vscode-markdownlint  # keeps the docs tidy
code --install-extension github.vscode-github-actions    # CI workflow editing and status
```

| Extension | Notes |
| --- | --- |
| C# Dev Kit | Free for individual and open-source use. Sign in with a Microsoft account when it prompts. |
| PostgreSQL (Microsoft) | Add a connection: host `localhost`, port `5433`, database `ipamo`, user `ipamo`, and the password from `.env` |
| REST Client | Lets you keep `Ipamo.Api.http` request files alongside Swagger |

**Recommended settings** go in the repo's `.vscode/settings.json` (created in M0). Format on save is on, and the test explorer is enabled.

## 8. Claude Code

Install it by following Anthropic's current instructions at <https://docs.claude.com/en/docs/claude-code/overview>, then:

```bash
claude --version
```

Run `claude` from the repo root. It reads [`CLAUDE.md`](../CLAUDE.md) automatically.

## 9. Verify everything

Run this block. Every line should print a version or a success message, with no errors:

```bash
brew --version | head -1
git --version
gh auth status
dotnet --list-sdks | grep '^10\.'
dotnet dev-certs https --check --trust
docker info --format '{{.ServerVersion}}'
docker compose version
psql --version
code --version | head -1
code --list-extensions | grep -E 'csdevkit|vscode-pgsql|vscode-docker'
claude --version
openssl rand -base64 8      # LibreSSL ships with macOS; used to generate secrets
```

Then, **inside the repo** (once it exists after M0):

```bash
dotnet tool restore                 # installs dotnet-ef and reportgenerator from .config/dotnet-tools.json
dotnet ef --version                 # expect 10.x
dotnet reportgenerator --help | head -1
```

## 10. Troubleshooting

| Symptom | Fix |
| --- | --- |
| `dotnet: command not found` after installing | Open a new terminal. If it's still missing, check that `/usr/local/share/dotnet` is on your `PATH`. |
| Integration tests fail with "Docker is either not running or misconfigured" | Start Docker Desktop and check §5 step 3 (the default Docker socket setting). |
| `docker compose up` fails with "port 5433 is already allocated" | Something else is using 5433: `lsof -i :5433`. Stop it, or change the port in `docker-compose.yml` **and** your connection string. |
| Browser warns about the certificate at `https://localhost:7180` | Run `dotnet dev-certs https --clean`, then `dotnet dev-certs https --trust`, and restart the browser. |
| `psql: command not found` | Run `brew link --force libpq`, then open a new terminal. |
| C# Dev Kit shows no projects | Open the **repo root folder** (the one with `Ipamo.slnx`), not `src/`. |
| Tests are slow the first time | Docker is pulling `postgres:17` (about 150 MB). Later runs reuse the image. |
