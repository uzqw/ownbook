# ownbook

**A self-hosted ebook reader your LLM can read.**

Upload PDF/EPUB/MOBI, read them in the browser on any device, keep your
progress in sync — and let Claude, Cursor, or any MCP client browse your
library and quote chapters through the built-in read-only MCP server.

![Library](docs/screenshots/v2-library.png)

<img src="docs/screenshots/good-reader.png" width="700" />

<p>
  <img src="docs/screenshots/v2-m-library.png" width="280" />
  <img src="docs/screenshots/m-dark-reader.png" width="280" />
</p>

**One command:**

```bash
docker run -d -p 18094:18093 -v ownbook-data:/app/pb_data uzqw/ownbook:latest
```

Open <http://127.0.0.1:18094> — first-run bootstrap creates the schema and a
demo account automatically.

## Why not Calibre-Web / Kavita?

- Single container, single Go binary + PocketBase — no external DB, no sidecars.
- Per-device reading progress, bookmarks, and notes sync through one backend.
- **Embedded MCP server**: your LLM assistant can list shelves, walk TOCs, and
  read page ranges — no plugins, no extra service. (See [MCP server](#mcp-server).)

⭐ If this is useful, a star helps others find it.

## Deployment

### Docker Hub image

The published image currently supports `linux/amd64`:

```bash
docker run -d \
  --name ownbook \
  --restart unless-stopped \
  -p 18094:18093 \
  -v ownbook-data:/app/pb_data \
  uzqw/ownbook:latest
```

Open <http://127.0.0.1:18094>. On first start the container automatically creates
the PocketBase schema and the default accounts shown below; no separate bootstrap
command is required.

Upgrade without losing books or reading data:

```bash
docker pull uzqw/ownbook:latest
docker rm -f ownbook
docker run -d --name ownbook --restart unless-stopped \
  -p 18094:18093 -v ownbook-data:/app/pb_data uzqw/ownbook:latest
```

### Build from source

A Linux host only needs Docker with Compose v2 and `make`:

```bash
git clone https://github.com/uzqw/ownbook.git
cd ownbook
make deploy
```

The command creates `.env` when missing, builds the image, and starts the app at
<http://127.0.0.1:18094>. Review the generated `.env` credentials before exposing
that port publicly. Set `DOCKER_HOST_PORT` to use another host port.

Deployment stores PocketBase data under
`${APP_DATA_ROOT}/ownbook/pb_data`. Set `APP_DATA_ROOT` to an absolute
application data root, or leave it unset to use the XDG user default:

```text
${XDG_DATA_HOME:-$HOME/.local/share}/uzqw/apps
```

The deployment path resolver rejects repository-local and unsafe roots. Path
parameterization does not migrate existing data; follow the migration runbook
before deploying against a new data directory.

For local development, run the foreground services with:

```bash
task setup
task doctor
task dev
```

The backend initializes missing collections and default accounts automatically.
`task bootstrap` remains available to reconcile schema changes manually.

Default local backend address:

```text
http://127.0.0.1:8090
```

### Platform-managed HTTPS

The Docker image serves HTTP; configure HTTPS in your own Caddy, Nginx, or other
reverse proxy. The local platform deployment's `https://${PLATFORM_HOST}:18094`
endpoint uses Caddy's internal CA.
Every client must trust the gateway's **root CA**; merely bypassing the browser
warning for one leaf certificate is not sufficient. Caddy rotates its default
12-hour leaf certificates, and an untrusted replacement makes in-page requests
fail with `Failed to fetch` until a top-level navigation handles the certificate
again.

For the local platform deployment, download the public root certificate over the
non-TLS gateway endpoint, install it in the client operating system, and restart
the browser:

```bash
curl http://${PLATFORM_HOST}:18089/caddy-local-root.crt \
  -o caddy-local-root.crt
```

Install it on Arch/Manjaro Linux:

```bash
sudo trust anchor --store caddy-local-root.crt
sudo update-ca-trust
```

Or on Debian/Ubuntu Linux:

```bash
sudo install -m 0644 caddy-local-root.crt \
  /usr/local/share/ca-certificates/caddy-local-root.crt
sudo update-ca-certificates
```

Fully quit and reopen the browser after changing the trust store. See the
platform `ACCESS.md` for macOS, Fedora/RHEL, Chrome/Chromium NSS, and Firefox
instructions. Preserve the platform Caddy data directory; replacing it creates
a new root CA that must be installed on every client again.

## Features

- PocketBase-based authentication and file storage.
- Upload, parse, list, and delete PDF, EPUB, and MOBI books.
- Page image rendering and extracted page text for reading.
- Bookmarks, notes, reading progress, and per-book metadata.
- Shared reading state across devices through one deployed backend.
- Read-only MCP server so LLM clients can browse the library and read chapters.

## MCP server

The backend embeds a read-only [Model Context Protocol](https://modelcontextprotocol.io)
server at `<base-url>/mcp` (streamable HTTP, same PocketBase process, no second
service). Any MCP client can list the library, walk a book's table of contents, and
read chapters or page ranges.

Use the URL you already reach the app on:

- `task dev` serves the backend directly, so it is the default local address
  `http://127.0.0.1:8090/mcp`.
- A deployed stack is reached through its reverse proxy — on the local platform
  deployment, `https://${PLATFORM_HOST}:18094/mcp`. The compose stack publishes no
  host port, so there is no `127.0.0.1:<port>` shortcut to the running container.

Authentication reuses PocketBase user tokens: send `Authorization: Bearer <token>`.
Every tool is scoped to the token's user, so one account cannot read another
account's books; mint the token from the account that owns the books. Credentials
live in `.env` as `APP_USER_EMAIL` / `APP_USER_PASSWORD`:

```bash
set -a; . ./.env; set +a
TOKEN=$(curl -s -X POST "${EBOOK_BASE_URL:-https://${PLATFORM_HOST}:18094}"\
  /api/collections/users/auth-with-password \
  -H 'Content-Type: application/json' \
  --data "$(jq -nc --arg i "$APP_USER_EMAIL" --arg p "$APP_USER_PASSWORD" \
    '{identity:$i,password:$p}')" | jq -r .token)
export EBOOK_PB_TOKEN="$TOKEN"
```

The `demo@e.co` / `demo1234` pair in `.env.example` only matches a stack
bootstrapped from that file; a stack with its own `.env` has its own accounts.

Tokens are not permanent. The `users` collection sets `authToken.duration` to
`432000`, i.e. 5 days, so any token a client caches goes stale on its own and the
client has to mint a fresh one once calls start failing with
`401 {"error":"invalid auth token"}`.

Tools:

| Tool               | Arguments                                           | Returns                                                    |
| ------------------ | --------------------------------------------------- | ---------------------------------------------------------- |
| `list_books`       | –                                                   | Books with `title`, `author`, `page_count`, `parse_status` |
| `get_toc`          | `book_id`                                           | TOC tree: `title`, `page`, `level`, `children`             |
| `get_section_text` | `book_id`, `section_title`, `start_page` (optional) | Section text plus a `next_page` cursor                     |
| `get_pages`        | `book_id`, `from_page`, `to_page`                   | Text for an explicit page range                            |
| `search_in_book`   | `book_id`, `query`                                  | Matches as `page_number` and line snippets                 |

Behaviour worth knowing when calling the tools:

- `get_section_text` ends a section at the page before the next same-or-higher-level
  TOC entry (the last section runs to the end of the book), returns about ten pages per
  call, and pages through longer sections with `next_page`.
- Sections that share their start page with a sibling come back as that single page with
  `is_partial: true` and a `note`, never as empty text.
- Page text is joined with `--- p.N ---` separators; responses over roughly 100KB are cut
  with a `[truncated: ...]` marker and `truncated: true`.
- The route is registered for `POST` only. The frontend's `GET /{path...}` static route
  conflicts with an all-methods `/mcp` pattern in Go's `ServeMux`, and streamable HTTP
  clients send JSON-RPC exclusively over `POST`.

Any client that accepts a URL plus headers can use it directly:

```json
{
  "mcpServers": {
    "ebook": {
      "url": "https://${PLATFORM_HOST}:18094/mcp",
      "headers": { "Authorization": "Bearer ${EBOOK_PB_TOKEN}" }
    }
  }
}
```

For `pi`, put that block in a file and pass `--mcp-config <file>`; the adapter supports
`"auth": "bearer"` with `"bearerTokenEnv": "EBOOK_PB_TOKEN"` instead of a literal header,
and prefixes tool names with the server key (`ebook_list_books`, `ebook_get_toc`, ...).

Because the token expires in 5 days, a `bearerTokenEnv` holding a static token goes
stale silently. Point `bearerToken` at a command instead: a leading `!` marks it as a
command, and the adapter re-runs it every time the server connects, so the token is
always fresh:

```json
{
  "mcpServers": {
    "ebook": {
      "url": "https://${PLATFORM_HOST}:18094/mcp",
      "auth": "bearer",
      "bearerToken": "!${HOME}/.config/mcp/ebook-token.sh",
      "lifecycle": "lazy"
    }
  }
}
```

where that script runs the `auth-with-password` call above and prints nothing but the
token.

Smoke test with `curl`. The session is stateful: capture the `Mcp-Session-Id` response
header from `initialize` and echo it on every later call, or the server answers
`404 Invalid session ID`:

```bash
# initialize, capturing the session id
SID=$(curl -s -D - -o /dev/null -X POST https://${PLATFORM_HOST}:18094/mcp \
  -H "Authorization: Bearer $EBOOK_PB_TOKEN" -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"0"}}}' \
  | sed -n 's/^[Mm]cp-[Ss]ession-[Ii]d: *//p' | tr -d '\r')

# any later call must echo it back
curl -s -X POST https://${PLATFORM_HOST}:18094/mcp \
  -H "Authorization: Bearer $EBOOK_PB_TOKEN" -H "Mcp-Session-Id: $SID" \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'
```

## Development

Requirements:

- Go
- Node.js and npm
- Task

Useful commands:

```bash
task setup       # install locked frontend and Go dependencies
task doctor      # check the runtime environment without modifying it
task dev         # run the frontend and backend in the foreground
task bootstrap   # create collections and demo users
task fmt:check   # verify frontend and Go formatting
task lint        # run frontend, shell, and Go linters
task typecheck   # run frontend type checking
task test        # run frontend and backend tests
task build       # build production frontend and backend artifacts
task config      # validate manifest and merged Compose configuration
task check       # run all checks above in the documented order
task deploy:down # stop deployment without deleting persistent data
task deploy:logs # show deployment logs
```

Default accounts created on first startup:

```text
User:  demo@e.co  / demo1234
Admin: admin@e.co / admin123
```

Change these credentials before exposing the service publicly. Existing accounts
and an existing `.env` are not changed automatically.

## Repository layout

```text
src/                         Vue frontend
backend/cmd/ebook-pocketbase PocketBase Go extension
pb_hooks/                    PocketBase hooks
scripts/                     bootstrap and container scripts
fonts/                       required CJK font asset for EPUB rendering
```

## Repository hygiene

This repository intentionally excludes or ignores:

- PocketBase runtime data under `.local/` and `backend/pb_data/`
- Local environment files such as `.env`
- Build outputs such as `dist/`
- Dependency directories such as `node_modules/`
- Local agent/runtime metadata such as `.omx/`, `.agents/`, and `omx_wiki/`
- Generated backend binary outputs

## Font asset

`fonts/DroidSansFallback.ttf` is kept in the repository because EPUB/CJK rendering depends on it for consistent output across devices.

