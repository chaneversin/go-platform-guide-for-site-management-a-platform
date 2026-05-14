# Multi-Tenant Website Management Platform — Go Guide

> Target: Go 1.21+, Linux server, 4 GB RAM  
> Architecture: one Go binary manages N client sites, runs commands locally or over SSH, persists state in SQLite.

---

## 1. The Blueprint — Data Model + HTML Template

### `models/business.go`
```go
package models

// Business holds all data for one client site.
// Add fields freely — the template will pick them up automatically.
type Business struct {
    ID       string      // unique slug, e.g. "acme-electric"
    Name     string
    Bio      string
    Location string
    Phone    string
    Email    string
    PriceList []PriceItem
}

type PriceItem struct {
    Service string
    Price   string // keep as string → "GH₵ 120 / hr" works fine
}
```

### `templates/site.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ .Name }}</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; }
    body   { font-family: system-ui, sans-serif; color: #1a1a1a; }
    header { background: #0d3349; color: #fff; padding: 2rem; text-align: center; }
    main   { max-width: 760px; margin: 2rem auto; padding: 0 1rem; }
    table  { width: 100%; border-collapse: collapse; margin-top: 1rem; }
    th, td { padding: .75rem 1rem; border: 1px solid #ddd; text-align: left; }
    th     { background: #f4f4f4; }
  </style>
</head>
<body>
  <header>
    <h1>{{ .Name }}</h1>
    <p>{{ .Location }}</p>
  </header>

  <main>
    <section>
      <h2>About Us</h2>
      <p>{{ .Bio }}</p>
    </section>

    <section style="margin-top:2rem">
      <h2>Contact</h2>
      <p>📞 {{ .Phone }} &nbsp;|&nbsp; ✉ {{ .Email }}</p>
    </section>

    {{ if .PriceList }}
    <section style="margin-top:2rem">
      <h2>Price List</h2>
      <table>
        <thead><tr><th>Service</th><th>Price</th></tr></thead>
        <tbody>
          {{ range .PriceList }}
          <tr><td>{{ .Service }}</td><td>{{ .Price }}</td></tr>
          {{ end }}
        </tbody>
      </table>
    </section>
    {{ end }}
  </main>
</body>
</html>
```

---

## 2. The Generator — Compile Template → Save `index.html`

### `generator/generator.go`
```go
package generator

import (
    "fmt"
    "html/template"
    "os"
    "path/filepath"

    "yourmodule/models"
)

// templatePath is relative to where the binary runs.
// Embed it at compile time (see tip below) for a single-binary deploy.
const templatePath = "templates/site.html"

// GenerateSite parses the template, creates the client directory,
// and writes a fresh index.html.  Returns the output path.
func GenerateSite(biz models.Business, outputRoot string) (string, error) {
    // 1. Parse template (cache this in production — parse once, execute many)
    tmpl, err := template.ParseFiles(templatePath)
    if err != nil {
        return "", fmt.Errorf("parse template: %w", err)
    }

    // 2. Ensure client directory exists
    clientDir := filepath.Join(outputRoot, biz.ID)
    if err := os.MkdirAll(clientDir, 0755); err != nil {
        return "", fmt.Errorf("create dir: %w", err)
    }

    // 3. Create / truncate the output file
    outPath := filepath.Join(clientDir, "index.html")
    f, err := os.Create(outPath)
    if err != nil {
        return "", fmt.Errorf("create file: %w", err)
    }
    defer f.Close()

    // 4. Execute template with business data
    if err := tmpl.Execute(f, biz); err != nil {
        return "", fmt.Errorf("execute template: %w", err)
    }

    return outPath, nil
}
```

### Embed the template for single-binary deployment (Go 1.16+)
```go
// Add this to generator.go instead of reading from disk:
import _ "embed"

//go:embed ../templates/site.html
var rawTemplate string

func parsedTemplate() (*template.Template, error) {
    return template.New("site").Parse(rawTemplate)
}
```

### Quick usage
```go
biz := models.Business{
    ID:       "acme-electric",
    Name:     "Acme Electrical Works",
    Bio:      "Licensed electricians serving Accra since 2010.",
    Location: "Accra, Ghana",
    Phone:    "+233 20 000 0000",
    Email:    "info@acme.gh",
    PriceList: []models.PriceItem{
        {Service: "Wiring (per room)", Price: "GH₵ 350"},
        {Service: "Panel installation", Price: "GH₵ 1,200"},
    },
}

path, err := generator.GenerateSite(biz, "./sites")
// creates: ./sites/acme-electric/index.html
```

---

## 3. The Control — Local Reboot via `os/exec`

### `control/local.go`
```go
package control

import (
    "bytes"
    "fmt"
    "os/exec"
    "strings"
)

// RestartService runs: systemctl restart <serviceName>
// The caller must have sudo rights or run as root.
func RestartService(serviceName string) error {
    // Allowlist prevents command injection
    if err := validateServiceName(serviceName); err != nil {
        return err
    }

    cmd := exec.Command("systemctl", "restart", serviceName)

    var stderr bytes.Buffer
    cmd.Stderr = &stderr

    if err := cmd.Run(); err != nil {
        return fmt.Errorf("systemctl restart %s: %w — %s",
            serviceName, err, stderr.String())
    }
    return nil
}

// ReloadNginx does a graceful config reload (zero downtime).
func ReloadNginx() error {
    cmd := exec.Command("nginx", "-s", "reload")
    out, err := cmd.CombinedOutput()
    if err != nil {
        return fmt.Errorf("nginx reload: %w — %s", err, out)
    }
    return nil
}

// validateServiceName is a simple allowlist guard.
func validateServiceName(name string) error {
    allowed := []string{"nginx", "apache2", "caddy", "your-app"}
    for _, a := range allowed {
        if name == a { return nil }
    }
    return fmt.Errorf("service %q not in allowlist", name)
}

// RunCommand is a generic helper for trusted internal calls.
func RunCommand(name string, args ...string) (string, error) {
    out, err := exec.Command(name, args...).CombinedOutput()
    return strings.TrimSpace(string(out)), err
}
```

---

## 4. Remote Management — SSH Reboot

### Install the package
```bash
go get golang.org/x/crypto/ssh
```

### `control/remote.go`
```go
package control

import (
    "fmt"
    "os"
    "time"

    "golang.org/x/crypto/ssh"
)

// RemoteClient wraps an SSH connection to one server.
type RemoteClient struct {
    client *ssh.Client
    Host   string
}

// NewRemoteClient connects with an SSH private key file.
// keyPath is typically ~/.ssh/id_ed25519
func NewRemoteClient(user, host, keyPath string) (*RemoteClient, error) {
    keyBytes, err := os.ReadFile(keyPath)
    if err != nil {
        return nil, fmt.Errorf("read key: %w", err)
    }

    signer, err := ssh.ParsePrivateKey(keyBytes)
    if err != nil {
        return nil, fmt.Errorf("parse key: %w", err)
    }

    cfg := &ssh.ClientConfig{
        User:            user,
        Auth:            []ssh.AuthMethod{ssh.PublicKeys(signer)},
        HostKeyCallback: ssh.InsecureIgnoreHostKey(), // ← replace in prod (see below)
        Timeout:         10 * time.Second,
    }

    conn, err := ssh.Dial("tcp", host+":22", cfg)
    if err != nil {
        return nil, fmt.Errorf("dial %s: %w", host, err)
    }

    return &RemoteClient{client: conn, Host: host}, nil
}

// Run executes a single command on the remote host and returns stdout+stderr.
func (r *RemoteClient) Run(command string) (string, error) {
    sess, err := r.client.NewSession()
    if err != nil {
        return "", fmt.Errorf("new session: %w", err)
    }
    defer sess.Close()

    out, err := sess.CombinedOutput(command)
    return string(out), err
}

// RestartRemoteService runs systemctl restart over SSH.
func (r *RemoteClient) RestartRemoteService(service string) error {
    if err := validateServiceName(service); err != nil {
        return err
    }
    out, err := r.Run("sudo systemctl restart " + service)
    if err != nil {
        return fmt.Errorf("remote restart %s on %s: %w — %s",
            service, r.Host, err, out)
    }
    return nil
}

// Close tears down the SSH connection.
func (r *RemoteClient) Close() { r.client.Close() }
```

### Verifying the host key in production (important)
```go
// Replace InsecureIgnoreHostKey() with this:
import "golang.org/x/crypto/ssh/knownhosts"

hostKey, err := knownhosts.New(os.ExpandEnv("$HOME/.ssh/known_hosts"))
if err != nil { log.Fatal(err) }

cfg.HostKeyCallback = hostKey
```

### Usage example
```go
remote, err := control.NewRemoteClient("ubuntu", "192.168.1.50", "~/.ssh/id_ed25519")
if err != nil { log.Fatal(err) }
defer remote.Close()

if err := remote.RestartRemoteService("nginx"); err != nil {
    log.Printf("reboot failed: %v", err)
}
```

---

## 5. Persistence — Clients in SQLite (recommended over JSON)

SQLite is perfect here: zero server, single file, supports concurrent reads, handles thousands of rows easily within 4 GB RAM.

### Install driver
```bash
go get github.com/mattn/go-sqlite3   # needs CGO — gcc must be installed
# OR for a pure-Go alternative:
go get modernc.org/sqlite
```

### `db/store.go`
```go
package db

import (
    "database/sql"
    "fmt"
    "time"

    _ "modernc.org/sqlite" // pure Go, no CGO
)

type Client struct {
    ID            string
    BusinessName  string
    ServerHost    string
    ServiceName   string
    SiteStatus    string // "live" | "building" | "error"
    LastRestarted time.Time
    CreatedAt     time.Time
}

type Store struct { db *sql.DB }

func NewStore(path string) (*Store, error) {
    db, err := sql.Open("sqlite", path)
    if err != nil {
        return nil, err
    }

    // Enable WAL mode — faster writes, safe concurrent reads
    db.Exec("PRAGMA journal_mode=WAL;")

    if err := migrate(db); err != nil {
        return nil, err
    }
    return &Store{db: db}, nil
}

func migrate(db *sql.DB) error {
    _, err := db.Exec(`
        CREATE TABLE IF NOT EXISTS clients (
            id              TEXT PRIMARY KEY,
            business_name   TEXT NOT NULL,
            server_host     TEXT,
            service_name    TEXT DEFAULT 'nginx',
            site_status     TEXT DEFAULT 'building',
            last_restarted  DATETIME,
            created_at      DATETIME DEFAULT CURRENT_TIMESTAMP
        );
    `)
    return err
}

func (s *Store) UpsertClient(c Client) error {
    _, err := s.db.Exec(`
        INSERT INTO clients (id, business_name, server_host, service_name, site_status)
        VALUES (?, ?, ?, ?, ?)
        ON CONFLICT(id) DO UPDATE SET
            business_name = excluded.business_name,
            server_host   = excluded.server_host,
            service_name  = excluded.service_name,
            site_status   = excluded.site_status;
    `, c.ID, c.BusinessName, c.ServerHost, c.ServiceName, c.SiteStatus)
    return err
}

func (s *Store) GetClient(id string) (Client, error) {
    var c Client
    row := s.db.QueryRow(
        `SELECT id, business_name, server_host, service_name,
                site_status, last_restarted, created_at
         FROM clients WHERE id = ?`, id)

    err := row.Scan(&c.ID, &c.BusinessName, &c.ServerHost, &c.ServiceName,
        &c.SiteStatus, &c.LastRestarted, &c.CreatedAt)
    return c, err
}

func (s *Store) ListClients() ([]Client, error) {
    rows, err := s.db.Query(
        `SELECT id, business_name, server_host, service_name,
                site_status, last_restarted, created_at
         FROM clients ORDER BY created_at DESC`)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var out []Client
    for rows.Next() {
        var c Client
        if err := rows.Scan(&c.ID, &c.BusinessName, &c.ServerHost, &c.ServiceName,
            &c.SiteStatus, &c.LastRestarted, &c.CreatedAt); err != nil {
            return nil, err
        }
        out = append(out, c)
    }
    return out, rows.Err()
}

func (s *Store) MarkRestarted(id string) error {
    _, err := s.db.Exec(
        `UPDATE clients SET last_restarted = CURRENT_TIMESTAMP WHERE id = ?`, id)
    return err
}

func (s *Store) UpdateStatus(id, status string) error {
    _, err := s.db.Exec(
        `UPDATE clients SET site_status = ? WHERE id = ?`, status, id)
    return err
}

func (s *Store) Close() error { return s.db.Close() }
```

### JSON alternative (simpler, for < 50 clients)
```go
// db/json_store.go
package db

import (
    "encoding/json"
    "os"
    "sync"
)

type JSONStore struct {
    mu      sync.RWMutex
    path    string
    Clients map[string]Client
}

func NewJSONStore(path string) (*JSONStore, error) {
    s := &JSONStore{path: path, Clients: map[string]Client{}}
    data, err := os.ReadFile(path)
    if err == nil {
        json.Unmarshal(data, &s.Clients)
    }
    return s, nil
}

func (s *JSONStore) Save() error {
    s.mu.RLock()
    defer s.mu.RUnlock()
    data, err := json.MarshalIndent(s.Clients, "", "  ")
    if err != nil { return err }
    return os.WriteFile(s.path, data, 0644)
}

func (s *JSONStore) Set(c Client) {
    s.mu.Lock()
    s.Clients[c.ID] = c
    s.mu.Unlock()
}
```

---

## Project Layout

```
platform/
├── main.go
├── models/
│   └── business.go
├── generator/
│   └── generator.go
├── control/
│   ├── local.go
│   └── remote.go
├── db/
│   ├── store.go        ← SQLite
│   └── json_store.go   ← JSON fallback
├── templates/
│   └── site.html
├── sites/              ← generated client dirs live here
│   ├── acme-electric/
│   │   └── index.html
│   └── bob-plumbing/
│       └── index.html
├── clients.db          ← SQLite database
└── go.mod
```

---

## Putting It All Together — `main.go`

```go
package main

import (
    "log"

    "yourmodule/control"
    "yourmodule/db"
    "yourmodule/generator"
    "yourmodule/models"
)

func main() {
    // 1. Open database
    store, err := db.NewStore("./clients.db")
    if err != nil { log.Fatal(err) }
    defer store.Close()

    // 2. Define a client
    biz := models.Business{
        ID:       "acme-electric",
        Name:     "Acme Electrical Works",
        Bio:      "Licensed electricians serving Accra since 2010.",
        Location: "Accra, Ghana",
        Phone:    "+233 20 000 0000",
        Email:    "info@acme.gh",
        PriceList: []models.PriceItem{
            {Service: "Wiring (per room)",  Price: "GH₵ 350"},
            {Service: "Panel installation", Price: "GH₵ 1,200"},
        },
    }

    // 3. Generate the site
    store.UpdateStatus(biz.ID, "building")
    path, err := generator.GenerateSite(biz, "./sites")
    if err != nil { log.Fatal(err) }
    log.Printf("Site generated: %s", path)
    store.UpdateStatus(biz.ID, "live")

    // 4. Save/update client record
    store.UpsertClient(db.Client{
        ID:           biz.ID,
        BusinessName: biz.Name,
        ServerHost:   "192.168.1.50",
        ServiceName:  "nginx",
        SiteStatus:   "live",
    })

    // 5a. Local restart
    if err := control.ReloadNginx(); err != nil {
        log.Printf("local reload: %v", err)
    }

    // 5b. Remote restart
    remote, err := control.NewRemoteClient("ubuntu", "192.168.1.50", "~/.ssh/id_ed25519")
    if err != nil { log.Printf("ssh connect: %v", err); return }
    defer remote.Close()

    if err := remote.RestartRemoteService("nginx"); err != nil {
        log.Printf("remote restart: %v", err)
    } else {
        store.MarkRestarted(biz.ID)
        log.Println("Remote nginx restarted ✓")
    }
}
```

---

## Memory & Performance Tips for 4 GB RAM

| Concern | Fix |
|---|---|
| Template parsing | Parse once at startup, store in a `sync.Map` or package-level var — not on every request |
| SSH connections | Use a connection pool; keep connections alive with `keepalive` packets |
| Concurrent site generation | Use a `semaphore` channel to cap goroutines: `make(chan struct{}, 4)` |
| SQLite under load | `PRAGMA journal_mode=WAL` + `PRAGMA synchronous=NORMAL` covers most needs |
| Binary size | `go build -ldflags="-s -w"` strips debug info — cuts binary to ~6 MB |

### Concurrency cap example
```go
sem := make(chan struct{}, 4) // max 4 parallel builds

for _, biz := range clients {
    sem <- struct{}{}         // acquire slot
    go func(b models.Business) {
        defer func() { <-sem }() // release slot
        generator.GenerateSite(b, "./sites")
    }(biz)
}
```

---

## Next Steps

1. **Wrap in an HTTP API** — use `net/http` or [Chi](https://github.com/go-chi/chi) to expose endpoints like `POST /generate` and `POST /reboot/:clientID`
2. **Add auth** — protect the reboot endpoint with a bearer token stored in an env var
3. **Nginx vhost automation** — generate a `.conf` file per client and symlink it into `/etc/nginx/sites-enabled/`
4. **Watch for changes** — use `fsnotify` to auto-regenerate when a client's data file changes
