# Workstation prep

_Targets .NET 10 / C# 14. See also: [README](./README.md), [the source job description](../plan.md), and the tool notes referenced below — [dotMemory](./profiling-dotmemory.md), [dotTrace](./profiling-dottrace.md), [PerfView](./profiling-perfview.md), [VS Diagnostic Tools](./profiling-visual-studio-diagnostic-tools.md), [ETW](./etw-event-tracing-for-windows.md), [EventPipe](./eventpipe.md), [dotnet-trace](./dotnet-trace.md), [dotnet-counters](./dotnet-counters.md), [dotnet-dump](./dotnet-dump.md), [BenchmarkDotNet](./benchmarkdotnet.md), [Load testing](./load-testing.md), [SQL index analysis](./sql-index-analysis.md), [SQL query plan analysis](./sql-query-plan-analysis.md), [RavenDB performance](./ravendb-performance.md)._

A consolidated install list to exercise every technique in the HEALTH_AND_PERFORMANCE folder end-to-end on Windows 11 or a recent Linux/macOS workstation. The README is an index of *what to learn*; this page is the inventory of *what to install before you can practice it*. Run through the sections in order — later tooling assumes earlier installs (the `dotnet` CLI before `dotnet tool install`, a SQL Server instance before SSMS connects to it).

Each entry names the tool, the install command (per platform when it differs), and the note(s) that depend on it. Pure documentation pointers (web-only viewers, hosted services) are listed without install commands.

> Targeting principle: pick **.NET 10 SDK** as primary, keep **.NET 9** and **.NET 8 LTS** SDKs side-by-side for BenchmarkDotNet multi-runtime jobs, and avoid mixing major versions inside a single benchmark project.

## 1. Core SDKs and IDEs

| Tool | Why | Install |
|---|---|---|
| .NET 10 SDK | Every note targets .NET 10 / C# 14 | Windows: `winget install Microsoft.DotNet.SDK.10` · macOS: `brew install --cask dotnet-sdk` · Linux: see [learn.microsoft.com/dotnet/core/install/linux](https://learn.microsoft.com/dotnet/core/install/linux) |
| .NET 9 SDK | BDN multi-runtime `RuntimeMoniker.Net90` | `winget install Microsoft.DotNet.SDK.9` / `brew install --cask dotnet-sdk@9` |
| .NET 8 SDK (LTS) | BDN multi-runtime `RuntimeMoniker.Net80`; legacy comparisons | `winget install Microsoft.DotNet.SDK.8` |
| Visual Studio 2022 (17.x) | VS Diagnostic Tools, Performance Profiler, debugger PerfTips | `winget install Microsoft.VisualStudio.2022.Community` (or `.Professional` / `.Enterprise`) — make sure the **".NET desktop development"** and **"ASP.NET and web development"** workloads plus **".NET profiling tools"** individual component are checked |
| JetBrains Rider | Hosts dotMemory / dotTrace as integrated panels; cross-platform | `winget install JetBrains.Rider` · macOS: `brew install --cask rider` |
| Git | Required by JetBrains Toolbox / VS install scripts and most BDN/k6 workflows | `winget install Git.Git` |

Verify:

```bash
dotnet --list-sdks        # 8.x, 9.x, 10.x all present
dotnet --list-runtimes    # Microsoft.NETCore.App + AspNetCore.App on each major
```

Sources: [`benchmarkdotnet.md`](./benchmarkdotnet.md), [`profiling-visual-studio-diagnostic-tools.md`](./profiling-visual-studio-diagnostic-tools.md), [`profiling-dotmemory.md`](./profiling-dotmemory.md), [`profiling-dottrace.md`](./profiling-dottrace.md).

## 2. .NET diagnostics CLI tools (global)

All of these attach over the same diagnostic IPC socket and are routinely used together. Install in one block:

```bash
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-dump
dotnet tool install -g dotnet-gcdump
dotnet tool install -g dotnet-monitor
dotnet tool install -g dotnet-stack
dotnet tool install -g dotnet-symbol
```

Update later with `dotnet tool update -g <name>`. Ensure `~/.dotnet/tools` (Linux/macOS) or `%USERPROFILE%\.dotnet\tools` (Windows) is on `PATH` — newer SDK installers do this automatically.

| Tool | Used in | Purpose |
|---|---|---|
| `dotnet-trace` | [`dotnet-trace.md`](./dotnet-trace.md), [`eventpipe.md`](./eventpipe.md) | EventPipe trace capture; produces `.nettrace` |
| `dotnet-counters` | [`dotnet-counters.md`](./dotnet-counters.md) | Live counter monitor (CPU, GC, threadpool, ASP.NET, custom `Meter`) |
| `dotnet-dump` | [`dotnet-dump.md`](./dotnet-dump.md), [`memory-leaks-in-dotnet.md`](./memory-leaks-in-dotnet.md) | Cross-platform process dump capture + SOS REPL |
| `dotnet-gcdump` | [`dotnet-dump.md`](./dotnet-dump.md) | Managed-heap-only graph; opens in PerfView / dotMemory |
| `dotnet-monitor` | [`dotnet-counters.md`](./dotnet-counters.md), [`eventpipe.md`](./eventpipe.md), [`dotnet-dump.md`](./dotnet-dump.md) | Sidecar product (rule-based capture, Prometheus scrape, REST). Bake into images for prod |
| `dotnet-stack` | [`dotnet-trace.md`](./dotnet-trace.md) | Cheap thread-stack snapshot — companion to a counter spike |
| `dotnet-symbol` | [`profiling-perfview.md`](./profiling-perfview.md), [`dotnet-dump.md`](./dotnet-dump.md) | Downloads matching runtime PDBs / SOS for off-machine analysis |

## 3. JetBrains profilers

```powershell
# Windows
winget install JetBrains.Toolbox      # easiest path; toolbox installs/updates dotMemory, dotTrace, Rider together
# OR standalone:
winget install JetBrains.dotMemory
winget install JetBrains.dotTrace
```

```bash
# macOS / Linux (dotTrace only on Linux; dotMemory is Windows-only)
brew install --cask jetbrains-toolbox
```

Both ship as standalone GUIs, as Rider/ReSharper plugins, and as command-line binaries (`dotMemory.exe`, `dotTrace.exe` / `dotmemory`, `dottrace`) for headless / CI use. Licensing: 30-day trial, JetBrains personal account, Toolbox subscription, or paid commercial license — match to your org policy.

| Tool | Platform | Used in |
|---|---|---|
| dotMemory | Windows GUI; CLI on Windows/macOS/Linux for snapshots | [`profiling-dotmemory.md`](./profiling-dotmemory.md), [`memory-leaks-in-dotnet.md`](./memory-leaks-in-dotnet.md), [`finalization-and-finalizer-queue.md`](./finalization-and-finalizer-queue.md) |
| dotTrace | Windows + Linux | [`profiling-dottrace.md`](./profiling-dottrace.md), [`async-await-performance.md`](./async-await-performance.md), [`thread-contention.md`](./thread-contention.md) |

## 4. Microsoft / Windows profiling stack

| Tool | Purpose | Install |
|---|---|---|
| **PerfView** | ETW + EventPipe analysis; GC stats, allocation attribution, stack folding | Download `PerfView.exe` from [github.com/microsoft/perfview/releases](https://github.com/microsoft/perfview/releases) — single-file binary, no installer. Run elevated for kernel providers |
| **Windows Performance Toolkit (WPT)** | `xperf`, `wpr`, `wpa` — kernel + user-mode capture, modern timeline UI | Install via the Windows ADK: `winget install Microsoft.WindowsADK` then run the ADK installer and tick **Windows Performance Toolkit** |
| **Sysinternals ProcDump** | Crashed-process / rule-based dump capture (the runtime can't dump itself when SIGKILL'd) | `winget install Microsoft.Sysinternals.ProcDump` (or grab the whole [Sysinternals Suite](https://learn.microsoft.com/sysinternals/downloads/sysinternals-suite)) |
| **WinDbg (preview)** | Native dump fallback when SOS in `dotnet-dump analyze` falls short | `winget install Microsoft.WinDbg` |
| **TraceEvent / PerfView libraries** | Programmatic `.etl` / `.nettrace` reading from C# | NuGet `Microsoft.Diagnostics.Tracing.TraceEvent` (see §6) |

Set the symbol path once so PerfView, WinDbg, and `dotnet-dump analyze` resolve framework frames:

```powershell
# Windows (persistent, current user)
setx _NT_SYMBOL_PATH "srv*C:\symbols*https://msdl.microsoft.com/download/symbols"
```

```bash
# Linux / macOS (shell rc)
export _NT_SYMBOL_PATH="srv*$HOME/.symbols*https://msdl.microsoft.com/download/symbols"
```

Sources: [`profiling-perfview.md`](./profiling-perfview.md), [`etw-event-tracing-for-windows.md`](./etw-event-tracing-for-windows.md), [`dotnet-dump.md`](./dotnet-dump.md).

## 5. NuGet packages (per benchmarking / diagnostics project)

Add to a Release-mode console project that hosts your benchmarks and diagnostic harnesses:

```bash
dotnet new console -n Acme.Bench -f net10.0
cd Acme.Bench

# BenchmarkDotNet core + diagnosers
dotnet add package BenchmarkDotNet
dotnet add package BenchmarkDotNet.Diagnostics.Windows         # [EtwProfiler] (Windows only)
dotnet add package BenchmarkDotNet.Diagnostics.dotMemory       # optional; pairs with the JetBrains profiler

# Programmatic diagnostics
dotnet add package Microsoft.Diagnostics.NETCore.Client        # drive EventPipe from code (sidecars, custom triggers)
dotnet add package Microsoft.Diagnostics.Tracing.TraceEvent    # parse .etl / .nettrace in code (PerfView's library)

# Load testing (in-proc .NET)
dotnet add package NBomber
dotnet add package NBomber.Http
```

| Package | Used in |
|---|---|
| `BenchmarkDotNet` (≥ 0.14) | [`benchmarkdotnet.md`](./benchmarkdotnet.md), every "Senior-level gotchas" referencing `[MemoryDiagnoser]` |
| `BenchmarkDotNet.Diagnostics.Windows` | [`benchmarkdotnet.md`](./benchmarkdotnet.md), [`etw-event-tracing-for-windows.md`](./etw-event-tracing-for-windows.md) |
| `Microsoft.Diagnostics.NETCore.Client` | [`eventpipe.md`](./eventpipe.md) — programmatic `EventPipeSession` |
| `Microsoft.Diagnostics.Tracing.TraceEvent` | [`etw-event-tracing-for-windows.md`](./etw-event-tracing-for-windows.md) — `TraceLog`, `ClrTraceEventParser` |
| `NBomber` / `NBomber.Http` | [`load-testing.md`](./load-testing.md) — open-model `Simulation.Inject` |
| `RavenDB.Client` | [`ravendb-performance.md`](./ravendb-performance.md), [`document-database-query-optimization.md`](./document-database-query-optimization.md) |

> Builds must be `-c Release` for any benchmark to run — BDN refuses Debug.

## 6. Load-testing tooling

```powershell
# Windows
winget install k6.k6
```

```bash
# macOS
brew install k6

# Debian / Ubuntu
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
   --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
   | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt update && sudo apt install k6
```

| Tool | Use | Install |
|---|---|---|
| **k6** (Grafana) | First-class open-model HTTP/gRPC; CI-friendly; default for [`load-testing.md`](./load-testing.md) | see above |
| **NBomber** | When the scenario needs in-proc .NET helpers (DI handlers, EF) | NuGet — see §5 |
| **Azure Load Testing** | Managed runners (JMeter / k6 scripts), Azure Monitor integration | `az extension add --name load`; portal- or CLI-driven, no local runtime needed |

Legacy alternatives appear in the comparison table inside [`load-testing.md`](./load-testing.md) (JMeter, wrk2, Vegeta, Bombardier, Gatling). Install them only if a specific scenario calls for one — k6 + NBomber cover ~95% of .NET team needs.

## 7. Database tooling

### 7.1 SQL Server

```powershell
# Local SQL Server 2022 Developer + management UIs
winget install Microsoft.SQLServer.2022.Developer
winget install Microsoft.SQLServerManagementStudio       # SSMS — Query Store UI, plan visualisation
winget install Microsoft.AzureDataStudio                  # cross-platform alternative
winget install Microsoft.SQLCMD                           # CLI
winget install Microsoft.SqlPackage                       # DACPAC / BACPAC
```

```bash
# Docker (any host)
docker pull mcr.microsoft.com/mssql/server:2022-latest
docker run --name sql -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=YourStrong!Passw0rd" \
  -p 1433:1433 -d mcr.microsoft.com/mssql/server:2022-latest

# macOS / Linux: use Azure Data Studio + sqlcmd (no SSMS)
brew install --cask azure-data-studio
brew install sqlcmd
```

Used by: [`sql-query-plan-analysis.md`](./sql-query-plan-analysis.md), [`sql-index-analysis.md`](./sql-index-analysis.md), [`relational-database-query-optimization.md`](./relational-database-query-optimization.md), [`pagination-strategies.md`](./pagination-strategies.md), [`aggregation-efficiency.md`](./aggregation-efficiency.md), [`bulk-operations.md`](./bulk-operations.md), [`connection-pooling.md`](./connection-pooling.md).

### 7.2 PostgreSQL 16

```powershell
winget install PostgreSQL.PostgreSQL.16
winget install PostgreSQL.pgAdmin
winget install dbeaver.dbeaver
```

```bash
# macOS
brew install postgresql@16 pgadmin4
brew install --cask dbeaver-community

# Ubuntu / Debian
sudo apt install postgresql-16 postgresql-contrib pgadmin4 dbeaver-ce

# Docker
docker run --name pg -e POSTGRES_PASSWORD=postgres -p 5432:5432 -d postgres:16
```

After install, enable the extensions used in the notes:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;     -- cumulative query stats
CREATE EXTENSION IF NOT EXISTS pg_qualstats;           -- missing-index signal
CREATE EXTENSION IF NOT EXISTS pg_hint_plan;           -- per-query hints
```

Used by: the same SQL analysis notes as §7.1, plus the Postgres deltas inline.

### 7.3 RavenDB 6+

```powershell
# Windows binary install
winget install RavenDB.RavenDB
```

```bash
# Cross-platform via Docker (recommended for dev)
docker pull ravendb/ravendb:6.0-ubuntu-latest
docker run -d --name ravendb -p 8080:8080 -p 38888:38888 \
  -e RAVEN_Setup_Mode=Unsecured \
  -e RAVEN_License_Eula_Accepted=true \
  ravendb/ravendb:6.0-ubuntu-latest
```

RavenDB Studio is built-in — open `http://localhost:8080` after the container is up. The C# driver is the `RavenDB.Client` NuGet package (§5).

Used by: [`ravendb-performance.md`](./ravendb-performance.md), [`document-database-query-optimization.md`](./document-database-query-optimization.md), [`bulk-operations.md`](./bulk-operations.md).

## 8. Containers & Kubernetes

The container/Kubernetes capture flows in [`dotnet-trace.md`](./dotnet-trace.md), [`dotnet-counters.md`](./dotnet-counters.md), [`dotnet-dump.md`](./dotnet-dump.md), and [`eventpipe.md`](./eventpipe.md) all assume a working `kubectl` and a `dotnet-monitor` sidecar image.

```powershell
# Windows
winget install Docker.DockerDesktop
winget install Kubernetes.kubectl
winget install Helm.Helm                  # optional; many sample charts use it
```

```bash
# macOS
brew install --cask docker
brew install kubectl helm

# Linux
# follow https://docs.docker.com/engine/install/ ; kubectl: https://kubernetes.io/docs/tasks/tools/

# Pull the dotnet-monitor sidecar image once
docker pull mcr.microsoft.com/dotnet/monitor
```

For a local cluster: `kind`, `minikube`, or Docker Desktop's built-in Kubernetes — any will work for the diagnostic-port and shared-`emptyDir` patterns the notes describe.

## 9. Trace viewers (web — no install)

| Viewer | Reads | URL |
|---|---|---|
| Speedscope | `.speedscope.json` from `dotnet-trace convert --format Speedscope` | https://www.speedscope.app |
| Perfetto | Chromium / `chrome://tracing` JSON from `dotnet-trace convert --format Chromium` | https://ui.perfetto.dev |
| Windows Performance Analyzer (WPA) | `.etl` (already installed via §4 WPT) | n/a — desktop app |

PerfView and Visual Studio handle `.nettrace` and `.etl` directly; the web viewers are most useful for sharing a trace with someone who has neither installed.

## 10. Verification checklist

Run after the sections above. Each command should succeed without error.

```bash
# 1. SDKs
dotnet --list-sdks                    # expect 8.x, 9.x, 10.x

# 2. Diagnostics CLI
dotnet-trace --version
dotnet-counters --version
dotnet-dump --version
dotnet-gcdump --version
dotnet-monitor --version
dotnet-stack --version
dotnet-symbol --version

# 3. Profilers
PerfView /?                           # prints help (Windows)
xperf -help                            # WPT installed (Windows)
procdump -?                            # ProcDump (Windows)

# 4. Load testing
k6 version

# 5. Containers / k8s
docker --version
kubectl version --client
docker image inspect mcr.microsoft.com/dotnet/monitor >/dev/null && echo "dotnet-monitor image present"

# 6. Databases
sqlcmd -?                              # SQL Server CLI (Windows / cross-plat)
psql --version                         # PostgreSQL CLI
curl -sS http://localhost:8080/admin/stats >/dev/null && echo "RavenDB up"  # if you started the container

# 7. NuGet sanity (in a scratch project)
dotnet new console -n PrepCheck -f net10.0
cd PrepCheck
dotnet add package BenchmarkDotNet
dotnet build -c Release
```

If any line fails, jump back to the section that installed the offender and re-verify before moving on. The order above matches the dependency chain — a broken `dotnet --list-sdks` will cascade into every subsequent failure, so always fix earlier checks first.
