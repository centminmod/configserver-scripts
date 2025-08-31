**THIS DOCUMENT CONTAINS AI-GENERATED CONTENT THAT IS NOT VERIFIED, NOT TESTED, AND MAY CONTAIN SIGNIFICANT INACCURACIES OR COMPLETE FABRICATIONS (HALLUCINATIONS).**

These documents were generated out of curiosity by asking different AI models what it would theoretically take to update CSF (ConfigServer Security & Firewall) to support nftables. They are **NOT** official documentation, **NOT** tested implementations, and **NOT** endorsed by any CSF developers or maintainers.

CSF nftables Enablement Plan for AlmaLinux/Rocky/RHEL 10

Objective: Add an optional nftables backend to CSF that preserves current behavior on iptables-based systems while enabling full functionality on nft-only EL10 (AlmaLinux 10, Rocky 10, RHEL 10) systems.

Scope: Core firewall management (csf.pl), daemon (lfd.pl), configuration (ConfigServer::Config.pm and csf.*.conf), OS detection (os.pl), environment tests (csftest.pl), and documentation. Keep firewalld mutually exclusive as today.

Summary of Findings (code inspection)
- iptables is hardwired across the codebase:
  - csf.pl builds all rules with `$config{IPTABLES}`/`$config{IP6TABLES}` including filter, nat, mangle, raw, ipset integration, SMTP_BLOCK owner matching, SYNFLOOD/PORTFLOOD/CONNLIMIT/PORTKNOCKING, messenger NAT redirects, Docker NAT, and TRACE.
    - References: `code-copy/csf/csf.pl` lines creating chains and rules (examples): `-N LOGACCEPT` (~991), `-t nat -A PREROUTING ... REDIRECT` (~5102+), `-m set --match-set` (~1962+), `-m recent` (~2897+), `-m connlimit` (~2915+), logging with `-j LOG --log-prefix` (~1098+).
    - Fast path uses `iptables-restore` batches and ipset `restore` (faststart sections ~5890, ~5710+).
  - lfd.pl updates runtime chains and ipsets via iptables/ip6tables and ipset (e.g. ~5051+, ~5500+), watches `IPTABLES_LOG`.
  - Config loader (ConfigServer/Config.pm) validates iptables binaries and versions and derives features (RAW/MANGLE/SMTP_REDIRECT6/etc.) by probing iptables tables; fails if iptables missing.
  - os.pl only toggles Debian/Ubuntu alternatives to `iptables-nft` if present; does not support nft-only EL10.
  - csftest.pl executes iptables commands directly to probe kernel module capabilities.
  - Conf files (`csf.*.conf`) contain only iptables/ip6tables/ipset paths and options.
- ipset is optionally used for large sets and is referenced heavily. In nft-only environments ipset may be absent/deprecated; native nft sets should be used.
- firewalld is explicitly refused at runtime (csf.pl and lfd.pl check `systemctl is-active firewalld` and error). That expectation should continue in nft mode to avoid rule conflicts.

Design Principles
- Backward compatible by default: retain iptables backend (no changes for EL7/EL8/EL9/Ubuntu/Debian).
- Add an optional nftables backend for EL10 (and any nft-only host) controlled by a new config flag and auto-detection.
- Do not depend on firewalld; manage our own nft tables (unique name) and hooks. Continue to fail if firewalld is running.
- Minimize invasive rewrites by introducing a backend abstraction: a small internal API that csf/lfd call instead of shelling straight to iptables; initial implementation supports both backends. Phase in usage where rules are currently constructed.
- Preserve log prefixes so lfd’s existing parsers continue to work without major regex changes.

User-Facing Configuration Changes (csf.*.conf)
- New settings (append near existing tool paths):
  - FIREWALL_BACKEND = "auto"  # one of auto|iptables|nftables
  - NFT = "/usr/sbin/nft"
  - NFT_TABLE_FAMILY = "inet"   # use inet to cover IPv4/IPv6 in one table
  - NFT_TABLE_NAME = "csf"
  - NFT_SET_HASHSIZE = "1024"   # analogous to LF_IPSET_HASHSIZE (advisory)
  - NFT_SET_MAXELEM = "65536"   # analogous to LF_IPSET_MAXELEM (advisory)
  - NFT_LOG_LEVEL = "info"      # kernel log level for nft log actions
  - NFT_PRIORITY_FILTER = "0"   # chain hook priority for filter chains
  - NFT_PRIORITY_NAT = "0"      # chain hook priority for nat chains
- Document behavior:
  - auto: choose nftables when `nft` exists and iptables does not, or OS detected as EL10; otherwise iptables.
  - iptables: current behavior.
  - nftables: force nft backend, error if `nft` not executable.
- Keep existing iptables/ipset settings for legacy; ignore them in nft mode unless needed for compatibility.

Backend Detection and Platform Hooks
- os.pl:
  - Detect EL10 by parsing `/etc/redhat-release` for “release 10”.
  - Set default `FIREWALL_BACKEND=nftables` for EL10 if `nft` exists.
  - Continue setting iptables alternatives on Debian/Ubuntu when iptables-nft exists (unchanged logic).
  - Do not try to install iptables on EL10; prepare for nft-only.
- ConfigServer::Config.pm:
  - Parse FIREWALL_BACKEND and new NFT* settings.
  - If backend is nftables:
    - Do not croak when iptables binaries are missing.
    - Probe nft capabilities instead of iptables (see “Feature Probing” below) and derive: NAT6/NAT4 availability, RAW/MANGLE equivalents not required (nft has one “inet” family; no separate mangle/raw CLI needed).
    - Preserve CHECKs against firewalld (still must be disabled).
  - If backend is iptables: unchanged behavior.

Internal Backend Abstraction
- Add a thin internal API used by csf.pl and lfd.pl; two concrete implementations:
  - ConfigServer::Firewall::IPTables (shim that shells to iptables/ip6tables/ipset exactly as today)
  - ConfigServer::Firewall::NFTables (shells to nft and manages nft sets)
- Minimal method surface (initial):
  - ruleset_init(): Prepare baseline structures (tables, base chains, default policies). For iptables: current wipe + chain creation; for nft: create `table inet csf { chain input hook input priority X; policy accept; ... }`, chains `output`, `forward`; and `table ip nat`, `table ip6 nat` when needed.
  - ruleset_flush(): Remove CSF-owned objects only. iptables: current flush of chains; nft: `nft delete table inet csf` plus csf-owned nat tables or selective flush to avoid touching foreign rules.
  - chain_create(name), chain_flush(name), chain_delete(name), chain_rename(old,new), chain_append(chain, rule), chain_insert(chain, rule, pos?), chain_delete_rule(chain, rule-spec?). For nft, “rule” is a string in nft grammar; for iptables, it is the existing CLI payload (preserve today’s call-sites and add a translator when necessary; see below).
  - set_create(name, family v4/v6/inet, with_timeout bool): Create set for IPs; nft: `set name { type ipv4_addr; flags interval; }` (v4) / `ipv6_addr` (v6). If with_timeout, enable `timeout` on set and use “add element … { 1.2.3.4 timeout 3600s }`.
  - set_add(name, addr, [timeout]), set_del(name, addr), set_flush(name).
  - batch_apply(chunks): iptables uses `iptables-restore` (as today) and `ipset restore`; nft uses `nft -f` with a composed transaction script to be atomic.
  - log_rule helpers (emit nft `log prefix "…" level …` exactly matching existing prefixes used by lfd). Ensure prefixes like `Firewall: *TCP_IN Blocked* ` mirror current values so lfd regex continues to match.

Feature Probing (ConfigServer::Config.pm)
- nft backend probes:
  - `nft -v` must exist.
  - Check `table inet` support by `nft list tables` and parse; fallback to `table ip`+`table ip6` if inet unsupported (should be supported on EL10).
  - NAT support: try creating ephemeral “dummy” nat tables inside a disposable transaction and then delete them to determine NAT availability for IPv4 and IPv6; set booleans analogous to `NAT6`.
  - No WAITLOCK for nft; disable WAITLOCK logic in nft mode.
  - RAW/MANGLE booleans unnecessary; remove references in nft mode or set to 0.

Mapping iptables Features to nftables
- Chains and hooks:
  - iptables INPUT/OUTPUT/FORWARD → nft `chain input/output/forward` with `type filter hook … priority <NFT_PRIORITY_FILTER>; policy accept` under `table inet csf`.
  - LOGDROPIN/LOGDROPOUT/ALLOWIN/ALLOWOUT/DENYIN/DENYOUT/CC_*/PORTFLOOD/CONNLIMIT/SYNFLOOD/etc. → nft subchains within `table inet csf` invoked from the primary hooks.
  - Keep policy accept and implement explicit drop/reject rules as in current CSF.
- Logging:
  - `-j LOG --log-prefix 'x'` → `log prefix "x" level $NFT_LOG_LEVEL`. Include `ct state`/protocol matches as today for accuracy; main requirement is preserving the prefix string for lfd.
  - `DROP_IP_LOGGING` chains translate to a `log …` followed by `drop`/`reject`.
- Multiport:
  - `-m multiport --dports 80,443,8080` → `tcp dport {80,443,8080}` (or `udp dport {…}`) in nft.
  - Ranges like `135:139` → `tcp dport 135-139`.
- Conntrack/state:
  - `-m conntrack --ctstate ESTABLISHED,RELATED` → `ct state { established, related } accept`.
  - SPI toggles map to presence of these ct state rules.
- Owner matching (SMTP_BLOCK, UID/GID filters):
  - `-m owner --uid-owner X` → `meta skuid X`.
  - `--gid-owner X` → `meta skgid X`.
  - `-o lo` etc. map to `oif lo`/`iif lo`.
- Reject/drop:
  - `-j REJECT --reject-with tcp-reset` → `reject with tcp reset`.
  - Plain drop → `drop`.
- ipset integration → nft sets:
  - chain_DENY/chain_GDENY/etc.: create nft sets with `type ipv4_addr` (v4) and `type ipv6_addr` (v6). For `inet` chains where direction matters, use `ip saddr @set` or `ip daddr @set` (and `ip6 saddr/daddr` when not using inet family).
  - `-m set --match-set NAME src` → `ip saddr @NAME` in nft; for dst use `ip daddr @NAME`.
  - `LF_IPSET` flag controls whether sets are used. In nft mode, prefer nft sets regardless of external ipset tool availability.
- SYNFLOOD:
  - Current: limit module and chain. nft: create chain `SYNFLOOD` and rules `tcp flags syn / limit rate $SYNFLOOD_RATE burst $SYNFLOOD_BURST packets counter return` then log/drop fallback.
- PORTFLOOD and PORTKNOCKING (recent):
  - iptables uses `-m recent` — no direct nft equivalent. Implement via nft sets with timeout:
    - For PORTFLOOD per (proto,port): define a set keyed by saddr with `timeout` and maintain hit counting using repeated insertions and `numgen`/meter is overkill; simplest: emulate window by adding element with `timeout=$seconds`; use `limit rate` for pure rate-limiting per source. For accuracy close to recent+hitcount, combine `limit rate` with per-port sets (document approximation).
    - For PORTKNOCKING sequences: create per-step sets with timeout; on each knock port, `ip saddr @PK_S{n-1}` and if matched, add to `PK_Sn` with timeout; final rule allows port if in `PK_Sn`.
  - Clearly document behavioral differences; preserve existing config flags and disable feature if strict parity cannot be ensured.
- CONNLIMIT:
  - iptables `-m connlimit --connlimit-above N`: nft uses `ct count over N` limited to a flow dimension. Use `ct state new`+`tcp dport X`+`ct count over N` then log and `reject with tcp reset`.
- TRACE:
  - iptables raw+TRACE → nft uses `meta nftrace set 1` rule gated by `ip saddr X tcp flags syn` within a dedicated `inet` chain with hook prerouting (raw-like), or use `log flags all` for trace. Provide new `--trace` implementation accordingly.
- NAT and REDIRECT (messenger, docker):
  - Messenger NAT redirects in iptables PREROUTING → nft under `table ip nat` (v4) and `table ip6 nat` (v6):
    - `chain prerouting { type nat hook prerouting priority $NFT_PRIORITY_NAT; }`
    - `tcp dport {ports} ip saddr $blocked redirect to :$messenger_port` per feature.
  - SMTP_REDIRECT: same redirect mapping under `chain output { type nat hook output … }`.
  - Docker support: keep current defaults disabled unless explicitly enabled. In nft mode implement MASQUERADE in `postrouting` for configured docker ipv4/ipv6 networks; avoid interfering with Docker’s own rules by scoping to our device/network only. Document that docker-managed nft rules may conflict and that CSF should not manage docker firewall when Docker is controlling `nft`.

Applying the Backend in csf.pl
- Introduce a backend handle early after config load:
  - Determine backend based on FIREWALL_BACKEND and binary presence.
  - Instantiate `ConfigServer::Firewall::IPTables` or `ConfigServer::Firewall::NFTables`.
- Replace direct `&syscommand(__LINE__,"$config{IPTABLES} …")` in the following areas first (phased):
  1) Startup/shutdown paths: flush/policy reset, base chains creation, base accept/log/drop wiring.
  2) Set-based global allow/deny, allow/deny (csf.allow/csf.deny), CC_* chains: create sets and connect to chains.
  3) Messenger NAT redirects + allow rules.
  4) Optional features: SYNFLOOD, PORTFLOOD, CONNLIMIT, PORTKNOCKING.
  5) TRACE rules.
- FASTSTART logic:
  - Keep batching for iptables via `iptables-restore` as today.
  - For nft: gather a composed nft configuration blob and issue `nft -f` once. Include table/chain/set definitions and element additions in a single transaction. On error, abort and revert to TESTING safeguards.
- Error and lock handling:
  - Disable WAITLOCK attempts for nft.
  - Reuse csf’s TESTING cron safeguard by switching emergency flush to `nft delete table inet csf` plus CSF-owned nat tables. Never `nft flush ruleset` globally.

lfd.pl Runtime Updates
- Replace iptables/ipset update calls with backend:
  - Where lfd repopulates blocklists/CC sets or swaps chains, use `set_add/del/flush` and `chain_*` APIs to update nft sets and chains.
  - Maintain logging prefixes to keep existing lfd parsing.
- Keep `IPTABLES_LOG` setting name for compatibility; in nft mode document that log messages still appear in the same kernel log, and we rely on identical prefixes.

ConfigServer::Config.pm Changes
- Parsing:
  - Accept FIREWALL_BACKEND and NFT* keys.
  - If nft backend selected, skip hard failure when iptables path missing; instead require `NFT` path.
- Feature derivation:
  - Replace iptables `--version` and table probing with nft probes to set booleans for NAT4/NAT6 availability, and mark RAW/MANGLE flags as 0 or unused.
  - Preserve SPI toggles but ensure nft ct rules get added when LF_SPI/IPV6_SPI enabled.
- DROP_OUT value validation currently shells iptables commands; in nft mode, run a small nft dry-run (create temp chain, add reject tcp reset rule) to validate support and set fallback to DROP if not possible.

os.pl
- Add EL10 detection and set FIREWALL_BACKEND default to nftables if `nft` exists.
- Do not attempt to set iptables alternatives on EL10.
- Leave Debian/Ubuntu iptables-nft alternatives logic unchanged.

csftest.pl (Environment Test)
- Add a new nft path (run when backend=nftables or auto detects nft-only):
  - Test `nft` presence and version.
  - Transaction test to create ephemeral `table inet csf_test { chain input { type filter hook input priority 0; } }` then delete.
  - Add sample rules to validate:
    - filter accept/drop/log with prefix
    - multiport: `tcp dport {9998,9999}` log
    - owner match: `meta skuid 0 log`
    - ct state match
    - NAT redirect (if nat supported): `table ip nat { chain output { type nat hook output priority 0; tcp dport 9999 redirect to :9900 } }`
    - set functionality: create set, add element, rule `ip saddr @set drop`
  - Report failures similar to current module tests, mapping iptables module names to nft “capabilities”.
- Keep legacy iptables tests when backend=iptables.

Documentation and UI
- Document new settings and how to enable nft backend. Warn that firewalld must be disabled.
- Note behavioral differences for PORTFLOOD/PORTKNOCKING approximations under nft (sets with timeout + limit).
- Update any UI rule displays to query backend and fetch rules via:
  - iptables backend: `iptables -L` (current behavior)
  - nft backend: `nft list table inet csf` and `nft list table ip nat`/`ip6 nat` when configured.

Detailed Code Change Outline
1) Add modules:
   - `code-copy/csf/ConfigServer/Firewall.pm` (factory + common types)
   - `code-copy/csf/ConfigServer/Firewall/IPTables.pm` (thin shim delegates to current syscommand/ipset)
   - `code-copy/csf/ConfigServer/Firewall/NFTables.pm`
     - Implements methods listed in Backend Abstraction using `nft` CLI.
     - Helpers to compose nft batch file text for FASTSTART, chain wiring, set management, NAT rules, logging rules.

2) csf.pl:
   - After config load, instantiate backend via factory. Abort if firewalld is active (existing check remains).
   - Replace:
     - Global flush/policy ops with backend.ruleset_flush()/ruleset_init().
     - Chain creation `-N …` with backend.chain_create.
     - Rule adds `-A/-I` with backend.chain_append/insert and nft rule strings produced by small helper converters where necessary.
     - ipset flows with backend.set_*.
   - FASTSTART:
     - Route to backend.batch_apply for nft; leave iptables path intact.
   - TRACE (`--trace`): switch to nft ‘nftrace’ in nft mode.
   - Messenger (`domessenger` paths): create NAT rules under nft nat tables.

3) lfd.pl:
   - Use backend.set_* and chain_* in places where iptables/ipset updates occur (e.g., blocklist swaps, GALLOW/GDENY updates, CC filters, temp allow/deny bookkeeping). Keep on-disk temp files unchanged.
   - Continue to call existing UI/notification flows.

4) ConfigServer/Config.pm:
   - Parse and validate new nft settings. Replace iptables version/path checks in nft mode with nft.
   - Probing functions for NAT availability; set `MESSENGER6`, `SMTP_REDIRECT6`, etc., based on nft nat tables support.
   - Disable WAITLOCK in nft mode.

5) os.pl:
   - Default EL10 to nft backend; leave others unchanged.

6) csftest.pl:
   - Add `--nft` path (auto-selected) performing the capability tests outlined above. Retain legacy path.

7) Conf files (`csf.generic.conf` and panel overlays):
   - Add FIREWALL_BACKEND, NFT*, with comments and sane defaults.
   - In Debian/Ubuntu/SuSE overlays, set correct `NFT` path if standardized; otherwise leave default `/usr/sbin/nft`.

8) Emergency flush & TESTING mode:
   - In nft mode, replace cron flush commands with `nft delete table inet csf ; nft delete table ip nat ; nft delete table ip6 nat` guarded to only remove CSF-owned tables when present.

Validation Plan
- Unit-like verification via csftest.pl in both backends.
- Manual spot checks on EL10 minimal VM:
  - Start csf with TESTING=1, ensure no lockout, verify tables and rules created by `nft list ruleset | sed -n '/table inet csf/,/}/p'`.
  - Verify log lines appear with identical prefixes in `/var/log/messages` or distro-equivalent. Confirm lfd detects and reacts.
  - Test temp allow/deny, messenger redirect, SYNFLOOD, CONNLIMIT, CC filters, IPv6 if enabled.

Risks and Mitigations
- Port flooding/knocking parity: nft does not have iptables “recent”; implement best-effort sets with timeout+limit. Document the difference and allow feature disable if strict behavior is required.
- Docker interaction: Docker may manage its own nft rules. Scope CSF docker support to specific device/network and document that enabling both can conflict.
- Rule ordering: nft hook priorities matter. Use a default priority of 0 (filter default). If other rules present, consider exposing priority settings to admins.
- Emergency flush safety: Only delete CSF-owned tables to avoid affecting other nft tables.

Deliverables Checklist
- New modules: Firewall factory + nft backend.
- Config additions and loader changes.
- csf.pl/lfd.pl backend integration in incremental high-impact areas.
- csftest.pl nft capability path.
- Updated docs explaining backend selection, caveats, and EL10 guidance.

Phased Implementation Plan
1) Foundations (backend detection, modules, config parsing, base nft table/chain scaffolding) → enable csf start/stop without functional rules.
2) Core filter parity (ALLOWIN/OUT, DENYIN/OUT, LOGDROP chains, SPI ct rules, loopback, basic TCP/UDP/ICMP rules, CC allow/deny via sets).
3) NAT features (messenger redirects, SMTP_REDIRECT; Docker MASQUERADE scoped).
4) Advanced protections (SYNFLOOD, CONNLIMIT; PORTFLOOD/PORTKNOCKING best-effort).
5) lfd runtime updates (dynamic set updates, blocklist swaps) + csftest.pl nft path.
6) Documentation polish, defaults for EL10, and validation.

Notes on Double-Checking
- Preserved log prefixes to keep lfd unchanged (critical for parity).
- Avoid global nft ruleset flush; only touch CSF-owned tables.
- Used `inet` family to unify v4/v6 filter chains while keeping nat in `ip` and `ip6` tables as needed.
- Owner match mapping validated: nft ‘meta skuid/skgid’ corresponds to iptables owner module used by SMTP_BLOCK.
- Multiport and reject-with tcp-reset have direct nft equivalents.
- Firewalld mutual exclusion retained to avoid conflicts.


Deep-Dive Details and Examples

1) Exact Rule Mappings (iptables → nft)
- Base accept/drop/log
  - iptables: `-j LOG --log-prefix 'Firewall: *TCP_IN Blocked* '` → nft: `log prefix "Firewall: *TCP_IN Blocked* " level $NFT_LOG_LEVEL`
  - iptables: `-j ACCEPT` → nft: `accept`
  - iptables: `-j DROP` → nft: `drop`
  - iptables: `-j REJECT --reject-with tcp-reset` → nft: `reject with tcp reset`

- Device/loopback
  - iptables: `-I INPUT -i lo -j ACCEPT` → nft: `iif lo accept`
  - iptables: `-I OUTPUT -o lo -j ACCEPT` → nft: `oif lo accept`

- Conntrack/SPI
  - iptables: `-m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT` → nft: `ct state { established, related } accept`

- Protocol/ports
  - iptables: `-p tcp --dport 80 -j ACCEPT` → nft: `tcp dport 80 accept`
  - iptables: `-p tcp -m multiport --dports 80,443,8080 -j ACCEPT` → nft: `tcp dport {80,443,8080} accept`
  - iptables: `-p tcp --dport 135:139 -j DROP` → nft: `tcp dport 135-139 drop`
  - iptables: `-p udp --dport 53 -j ACCEPT` → nft: `udp dport 53 accept`
  - iptables: `-p icmp -j ACCEPT` → nft (v4): `icmp type { echo-request, echo-reply, destination-unreachable, time-exceeded, parameter-problem } accept` (or `ip protocol icmp accept` if allowing broadly)
  - ip6tables: `-p icmpv6 -j ACCEPT` → nft: `icmpv6 type { echo-request, echo-reply, destination-unreachable, packet-too-big, time-exceeded, parameter-problem, nd-neighbor-solicit, nd-neighbor-advert, nd-router-solicit, nd-router-advert, mld-listener-query } accept`

- Owner match (SMTP_BLOCK and UID/GID rules)
  - iptables: `-m owner --uid-owner 0 -j ACCEPT` → nft: `meta skuid 0 accept`
  - iptables: `-m owner --gid-owner 48 -j ACCEPT` → nft: `meta skgid 48 accept`

- ipset matches
  - iptables: `-m set --match-set chain_DENY src -j DROP` → nft: `ip saddr @chain_DENY drop`
  - Destination match: `-m set --match-set chain_DENY dst -j DROP` → nft: `ip daddr @chain_DENY drop`

- NAT/REDIRECT
  - iptables: `-t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8888` → nft (ip nat): `tcp dport 80 redirect to :8888`
  - iptables: `-t nat -A OUTPUT -p tcp --dport 25 -j REDIRECT --to-ports 587` → nft (ip nat): `tcp dport 25 redirect to :587`
  - iptables: `-t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE` → nft: `ip saddr 172.17.0.0/16 oifname != "docker0" masquerade`

- Connection limits
  - iptables: `-m connlimit --connlimit-above 50 -j REJECT --reject-with tcp-reset` → nft: `ct state new ct count over 50 reject with tcp reset`

- SYN flood
  - iptables chain `SYNFLOOD` with limit → nft:
    - `tcp flags syn limit rate $SYNFLOOD_RATE burst $SYNFLOOD_BURST accept`
    - then `log …` + `drop` fallback.

- Port flood (recent) approximation
  - iptables: `-m recent --update --seconds S --hitcount C --name PORT` → nft approximation (see section 3 below) using `limit rate` and per-port dynamic sets.

- Trace
  - iptables: `-t raw -I PREROUTING -p tcp --syn -s X -j TRACE` → nft: `meta nftrace set 1` combined with `ip saddr X tcp flags syn log prefix "TRACE:" level debug` (or accept/log-only as needed).


2) nft Table/Chain/Set Structure and Batch Examples
- Filter tables (unified inet family):
  - Table: `table inet csf {}`
  - Chains (subset): `input`, `output`, `forward`, `LOCALINPUT`, `LOCALOUTPUT`, `ALLOWIN`, `ALLOWOUT`, `DENYIN`, `DENYOUT`, `LOGDROPIN`, `LOGDROPOUT`, feature chains `SYNFLOOD`, `PORTFLOOD`, `CONNLIMIT`, `GALLOWIN`, `GALLOWOUT`, `GDENYIN`, `GDENYOUT`, etc.
  - Hooked chains: `input hook input priority $NFT_PRIORITY_FILTER; policy accept`, `output hook output …`, `forward hook forward …`.

- NAT tables (split by IP family due to nftables constraints):
  - IPv4: `table ip csf_nat { chain prerouting { type nat hook prerouting priority $NFT_PRIORITY_NAT; } chain output { type nat hook output priority $NFT_PRIORITY_NAT; } chain postrouting { type nat hook postrouting priority $NFT_PRIORITY_NAT; } }`
  - IPv6: `table ip6 csf_nat6 { … }` when any IPv6 NAT feature is enabled.

- Sets (examples):
  - Deny set v4: `set chain_DENY { type ipv4_addr; flags interval; }`
  - Deny set v6: `set chain_6_DENY { type ipv6_addr; flags interval; }`
  - Country code allow: `set cc_us { type ipv4_addr; }` and `set cc_6_us { type ipv6_addr; }`
  - Messenger temporary sets: `set MESSENGER { type ipv4_addr; flags timeout; }` and v6 equivalent when needed.

- Minimal bootstrap batch (nft -f) for initial startup:
```
flush ruleset meta mark 0xdeadbeef # no-op marker (avoid global flush in production)

define CSF_LOG_LEVEL = info

table inet csf {
  set chain_DENY { type ipv4_addr; flags interval; }
  set chain_6_DENY { type ipv6_addr; flags interval; }
  set chain_GALLOW { type ipv4_addr; flags interval; }
  set chain_6_GALLOW { type ipv6_addr; flags interval; }

  chain input { type filter hook input priority 0; policy accept;
    iif lo accept
    ct state { established, related } accept
    jump LOCALINPUT
    jump DENYIN
    jump ALLOWIN
    jump LOGDROPIN
  }

  chain output { type filter hook output priority 0; policy accept;
    oif lo accept
    ct state { established, related } accept
    jump LOCALOUTPUT
    jump DENYOUT
    jump ALLOWOUT
    jump LOGDROPOUT
  }

  chain LOCALINPUT { }
  chain LOCALOUTPUT { }

  chain ALLOWIN { }
  chain ALLOWOUT { }

  chain DENYIN {
    ip saddr @chain_DENY drop
    ip6 saddr @chain_6_DENY drop
  }
  chain DENYOUT {
    ip daddr @chain_DENY drop
    ip6 daddr @chain_6_DENY drop
  }

  chain LOGDROPIN {
    meta l4proto tcp log prefix "Firewall: *TCP_IN Blocked* " level $CSF_LOG_LEVEL
    meta l4proto tcp drop
    meta l4proto udp log prefix "Firewall: *UDP_IN Blocked* " level $CSF_LOG_LEVEL
    meta l4proto udp drop
    ip protocol icmp log prefix "Firewall: *ICMP_IN Blocked* " level $CSF_LOG_LEVEL
    ip protocol icmp drop
    ip6 nexthdr icmpv6 log prefix "Firewall: *ICMP6IN Blocked* " level $CSF_LOG_LEVEL
    ip6 nexthdr icmpv6 drop
  }

  chain LOGDROPOUT {
    meta l4proto tcp log prefix "Firewall: *TCP_OUT Blocked* " level $CSF_LOG_LEVEL
    meta l4proto tcp drop
    meta l4proto udp log prefix "Firewall: *UDP_OUT Blocked* " level $CSF_LOG_LEVEL
    meta l4proto udp drop
    ip protocol icmp log prefix "Firewall: *ICMP_OUT Blocked* " level $CSF_LOG_LEVEL
    ip protocol icmp drop
    ip6 nexthdr icmpv6 log prefix "Firewall: *ICMP6OUT Blocked* " level $CSF_LOG_LEVEL
    ip6 nexthdr icmpv6 drop
  }
}

# NAT for Messenger/SMTP_REDIRECT when enabled
table ip csf_nat {
  chain prerouting { type nat hook prerouting priority 0; }
  chain output     { type nat hook output     priority 0; }
  chain postrouting{ type nat hook postrouting priority 0; }
}
```

- Adding allowed service ports from config:
```
# Allow TCP_IN: 22,80,443
add rule inet csf ALLOWIN tcp dport {22,80,443} accept
# Allow UDP_IN: 53
add rule inet csf ALLOWIN udp dport {53} accept
# Allow TCP_OUT/UDP_OUT similarly under ALLOWOUT
```

- Messenger PREROUTING redirect example (IPv4):
```
add rule ip csf_nat prerouting tcp dport {80,443} ip saddr @MESSENGER redirect to :8888
```

- SMTP_REDIRECT OUTPUT example (IPv4):
```
add rule ip csf_nat output tcp dport 25 meta skuid != 0 redirect to :587
```


3) Port Flood and Port Knocking in nft (Design Options)
- Background: iptables uses `-m recent` with hitcount and seconds windows; nftables has no direct equivalent. We propose two options:

Option A (Approximate, stateless rate limiting):
- Use `limit rate` and `burst` in feature chain `PORTFLOOD` per port/proto. This rate limit is rule-scoped (global), not per-source. Acceptable for general noise reduction but not identical to per-source enforcement.

Option B (Per-source approximation with dynamic sets):
- Create a per-port set with timeout, e.g., `set pf_80 { type ipv4_addr; flags timeout; }` and v6 equivalent.
- On first packet to port X: `update @pf_X { ip saddr timeout $seconds s } counter` then allow to continue; on rule entry also add a drop rule when the same saddr is present and a synthetic “cooldown” condition is satisfied via a secondary set.
- Practical construction:
  1) Rule 1: `ip saddr @pf_X log prefix "PortFlood:X" level info drop` (blocks within timeout window)
  2) Rule 2: `update @pf_X { ip saddr timeout $seconds s }` (arms the window for the source)
  3) Optionally combine with `limit rate over C/seconds` globally to reflect hitcount intent.
- Note: This approximates “first packet allowed, subsequent within window dropped” per source; it differs from “C hits allowed within S seconds then drop”. Document as best-effort.

Port Knocking (sequence of K steps):
- Prepare sets `PK_${port}_S1 … PK_${port}_SK` with timeout = `timeout` (from config).
- For step i (i ≥ 2): only add to `PK_Si` if `ip saddr @PK_S{i-1}` matched; otherwise ignore.
- Final port rule: if `ip saddr @PK_S{K}` matched, accept target service and (optionally) immediately `delete element` from `PK_S{K}` to force re-knock.
- Example for two-step knock 1111 → 2222 to open 22:
```
table inet csf {
  set PK_22_S1 { type ipv4_addr; flags timeout; }
  set PK_22_S2 { type ipv4_addr; flags timeout; }
  chain input {
    tcp dport 1111 update @PK_22_S1 { ip saddr timeout 30s }
    tcp dport 2222 ip saddr @PK_22_S1 update @PK_22_S2 { ip saddr timeout 30s }
    tcp dport 22 ip saddr @PK_22_S2 accept
  }
}
```
- This reproduces the sequence-gate logic of PORTKNOCKING without recent.


4) Backend API (Proposed Signatures and Behavior)
- Factory: `ConfigServer::Firewall->new(%config)` → `$fw` with methods:
  - `$fw->backend()` → `iptables` or `nftables`
  - `$fw->ruleset_init()` → create csf tables/chains (idempotent)
  - `$fw->ruleset_flush()` → remove CSF tables/chains only
  - `$fw->chain_create($name)` / `$fw->chain_flush($name)` / `$fw->chain_delete($name)` / `$fw->chain_rename($old,$new)`
  - `$fw->rule_append($chain, $spec)` / `$fw->rule_insert($chain, $pos, $spec)` / `$fw->rule_delete($chain, $handle_or_spec)`
  - `$fw->set_create($name, %opts)` where `%opts` includes `{ family => 'v4'|'v6'|'inet', timeout => bool, interval => bool }`
  - `$fw->set_add($name, $addr, $timeout_s)` / `$fw->set_del($name, $addr)` / `$fw->set_flush($name)`
  - `$fw->batch_apply(@chunks)` → apply atomically (iptables-restore or nft -f)
  - `$fw->nat_add($hook, $spec)` where `$hook` is `prerouting|output|postrouting` and `$spec` is rule string
- IPTables backend simply glues current `&syscommand` and ipset calls to these methods; NFTables backend composes nft grammar and tracks created objects.


5) Config Keys (Exact Additions and Defaults)
- FIREWALL_BACKEND = "auto"
- NFT = "/usr/sbin/nft"
- NFT_TABLE_FAMILY = "inet"
- NFT_TABLE_NAME = "csf"
- NFT_NAT4_TABLE = "csf_nat"
- NFT_NAT6_TABLE = "csf_nat6"
- NFT_LOG_LEVEL = "info"   # one of emerg|alert|crit|err|warn|notice|info|debug
- NFT_PRIORITY_FILTER = "0"
- NFT_PRIORITY_NAT = "0"
- NFT_SET_HASHSIZE = "1024"  # advisory, for documentation; nft doesn’t take hashsize directly
- NFT_SET_MAXELEM = "65536"   # advisory; we enforce via lfd/csf logic
- In nft mode: WAITLOCK forced to 0 and RAW/MANGLE flags ignored.


6) Backend Detection (Pseudocode)
- os.pl defaulting on EL10:
```
if (-e "/etc/redhat-release" && slurp(...) =~ /release\s+10\b/) {
  if (-x "/usr/sbin/nft") { set FIREWALL_BACKEND = "nftables"; }
}
```
- Config.pm post-parse selection:
```
if (FIREWALL_BACKEND eq "auto") {
  if (-x $NFT && (!-x $IPTABLES || nft_only_system())) { backend := nftables }
  else { backend := iptables }
} elsif (FIREWALL_BACKEND eq "nftables") { require -x $NFT } else { require -x $IPTABLES }

if (backend == nftables) {
  probe_nft_capabilities();
  WAITLOCK := 0; RAW/MANGLE := 0;  # not used
} else {
  legacy iptables version and table probes (unchanged)
}
```


7) Logging Parity and lfd Considerations
- Keep identical log prefixes: `Firewall: *TCP_IN Blocked* `, etc.; nft `log` supports `prefix` with max length ~64 bytes — current prefixes are much shorter.
- Ensure rsyslog/kern logging is active on EL10 to capture kernel logs; document typical log locations (`/var/log/messages`, `/var/log/kern.log`, or journal via systemd).
- lfd greps `IPTABLES_LOG`-configured files; we keep the same variable name and behavior. In nft mode, note in docs that it still applies to nft log output.


8) Chain Ordering and Jump Graph (nft)
- For each hook chain (input/output):
  1) Loopback accept
  2) `ct state established,related` accept
  3) Jump to `LOCAL*` (admin custom insertion points used by CSF)
  4) Jump to `DENY*` (set-based rejections)
  5) Jump to `ALLOW*` (opened ports, ranges, protocols)
  6) Fallback to `LOGDROP*` (logs then drop/reject per config)
- Feature chains (SYNFLOOD, PORTFLOOD, CONNLIMIT) are jumped from `LOCAL*` or inserted near ALLOW/DENY depending on current CSF order semantics in csf.pl; maintain parity when mapping.


9) NAT and Messenger Details
- Messenger redirects apply in PREROUTING (inbound) to 80/443/21 (configurable):
  - IPv4: `add rule ip csf_nat prerouting tcp dport {80,443} ip saddr @MESSENGER redirect to :$PORT`
  - IPv6: under `table ip6 csf_nat6` similarly when `MESSENGER6` is enabled.
- SMTP_REDIRECT applies in OUTPUT chain, excluding root if required by config (owner match):
  - `add rule ip csf_nat output tcp dport $SMTP_PORTS meta skuid 0 return`
  - `add rule ip csf_nat output tcp dport $SMTP_PORTS redirect to :$REDIR_PORT`
- Docker: optionally add `postrouting` MASQUERADE rules scoped to CSF’s configured docker networks and device; warn against conflicts when Docker engine manages its own nft rules.


10) csftest.pl: nft Capability Script (Sketch)
- Steps:
  1) `nft --version` and `nft list tables` sanity
  2) Create ephemeral `table inet csf_test` with `input` hook, add simple accept/log/drop rules, then delete
  3) Test set creation and membership: create `set t { type ipv4_addr; }`, add one element, rule `ip saddr @t drop`
  4) Test owner match: rule `meta skuid 0 log prefix "OWN"` (delete afterwards)
  5) NAT test if needed: create `table ip csf_test_nat` with `output` hook and a redirect rule; delete
  6) Report failures mapped to capabilities (Filter, Log, Sets, Owner, NAT)


11) Validation Checklist (EL10)
- Start csf with TESTING=1 (firewalld disabled):
  - `nft list table inet csf` exists; input/output chains populated
  - Allowed ports open from remote test client
  - Blocked port produces kernel log with expected prefix; lfd recognizes it
  - Temp allow/deny updates set membership; verify with `nft list set inet csf chain_DENY`
  - Messenger redirect sends HTTP/HTTPS to local messenger port; nat rule present in `ip csf_nat`
  - SYNFLOOD/CONNLIMIT rules present when enabled; verify behavior with simple load tests


12) Edge Cases and Safety
- Do not use `nft flush ruleset` — only delete CSF-owned tables (`inet csf`, `ip csf_nat`, `ip6 csf_nat6`).
- If admin already has `table inet csf`, detect owner conflict and either refuse to start or use a distinct name via config override.
- Hook priority exposure via config allows admins to integrate with pre-existing nft rules (e.g., have CSF run before/after custom chains).
- If NAT is unavailable (kernel or policy), automatically disable Messenger/SMTP_REDIRECT/DOCKER NAT and log warnings (mirrors current iptables behavior).


13) Example: Translating Current csf.pl Base Rules to nft
- iptables intent (simplified):
  - Set default ACCEPT policies for INPUT/OUTPUT/FORWARD, flush tables
  - Create LOGACCEPT, LOGDROPIN/OUT, ALLOWIN/OUT, DENYIN/OUT, LOCALINPUT/OUTPUT
  - Hook order: INPUT → LOCALINPUT → DENYIN → ALLOWIN → LOGDROPIN; similarly for OUTPUT
  - Add loopback accepts and ct state accepts to INPUT/OUTPUT early
- nft equivalent (assembled during ruleset_init):
  - As shown in batch example above (Section 2) — keep same jump graph and create subchains, then populate according to config variables (TCP_IN, UDP_IN, ICMP_IN, SPI flags, DROP_* logging, etc.).


14) Migration and Rollback
- Switching backends:
  - From iptables → nftables: Stop csf, ensure firewalld is inactive, set FIREWALL_BACKEND=nftables, start csf; validate using checklist.
  - From nftables → iptables: Stop csf, set FIREWALL_BACKEND=iptables, start csf. The nft tables `inet csf`, `ip csf_nat`, `ip6 csf_nat6` will be deleted by csf stop path.
- Disaster recovery: If lockout occurs, TESTING cron will remove CSF nft tables: `nft delete table inet csf; nft delete table ip csf_nat; nft delete table ip6 csf_nat6`.


15) Future Enhancements (Optional)
- Translator pathway: Implement a small translator for common iptables spec strings to nft grammar to minimize call-site conditional logic.
- Per-source rate limiting: Explore nftables meters with maps keyed by `ip saddr` to approach recent+hitcount semantics closer, or integrate lfd-driven dynamic set controls based on log-driven heuristics.
- Rule handles tracking: persist nft rule handles to support targeted deletions and updates more robustly.


Appendix: Worked Examples (iptables → nft)

Example 1: SMTP_BLOCK (no redirect)
- Intent: Restrict outbound SMTP to root and optionally specific UIDs/GIDs; others blocked/logged.
- iptables (current):
```
# create and hook
iptables -N SMTPOUTPUT
iptables -I OUTPUT -j SMTPOUTPUT
# default drop (or REJECT) for configured ports
iptables -I SMTPOUTPUT -p tcp -m multiport --dports 25,465,587 -j REJECT --reject-with tcp-reset
# root allowed
iptables -I SMTPOUTPUT -p tcp -m multiport --dports 25,465,587 -m owner --uid-owner 0 -j ACCEPT
```
- nft:
```
add chain inet csf SMTPOUTPUT
add rule inet csf output jump SMTPOUTPUT
# root allowed first
add rule inet csf SMTPOUTPUT tcp dport {25,465,587} meta skuid 0 accept
# default deny or reject
add rule inet csf SMTPOUTPUT tcp dport {25,465,587} reject with tcp reset   # or: drop
```

Example 2: SMTP_REDIRECT (outbound NAT redirect)
- iptables:
```
iptables -t nat -I OUTPUT -p tcp -m multiport --dports 25,465,587 -j REDIRECT --to-ports 2525
iptables -t nat -I OUTPUT -p tcp -m multiport --dports 25,465,587 -m owner --uid-owner 0 -j RETURN
```
- nft (ip nat):
```
add table ip csf_nat
add chain ip csf_nat output { type nat hook output priority 0; }
add rule  ip csf_nat output tcp dport {25,465,587} meta skuid 0 return
add rule  ip csf_nat output tcp dport {25,465,587} redirect to :2525
```

Example 3: Messenger (PREROUTING redirect + allow)
- iptables (simplified):
```
iptables -t nat -A PREROUTING -p tcp -m multiport --dports 80,443 -j REDIRECT --to-ports 8888
iptables -I INPUT -p tcp --dport 8888 -m limit --limit 100/s --limit-burst 150 -j ACCEPT
```
- nft:
```
# NAT redirect inbound to messenger port 8888
add table ip csf_nat
add chain ip csf_nat prerouting { type nat hook prerouting priority 0; }
add rule  ip csf_nat prerouting tcp dport {80,443} redirect to :8888
# Allow messenger service port with rate limit
add rule inet csf ALLOWIN tcp dport 8888 limit rate 100/second burst 150 packets accept
```

Example 4: Country Code Allow (cc_allow)
- iptables:
```
ipset create cc_us hash:net
iptables -A CC_ALLOW -m set --match-set cc_us src -j ACCEPT
```
- nft:
```
add set inet csf cc_us { type ipv4_addr; flags interval; }
add rule inet csf CC_ALLOW ip saddr @cc_us accept
```
Populate the set:
```
nft add element inet csf cc_us { 1.2.3.0/24, 5.6.7.8 }
```

Example 5: Permanent Deny (csf.deny)
- iptables with ipset:
```
ipset add chain_DENY 1.2.3.4
iptables -A DENYIN -m set --match-set chain_DENY src -j DROP
```
- nft:
```
add set inet csf chain_DENY { type ipv4_addr; flags interval; }
add rule inet csf DENYIN ip saddr @chain_DENY drop
# Adding an IP
nft add element inet csf chain_DENY { 1.2.3.4 }
```

Example 6: Temporary Deny (with timeout)
- iptables ‘recent’ approximation not needed with nft sets:
```
add set inet csf chain_DENY { type ipv4_addr; flags interval, timeout; }
nft add element inet csf chain_DENY { 1.2.3.4 timeout 3600s }
```

Example 7: CONNLIMIT on port 80 (100)
- iptables:
```
iptables -A INPUT -p tcp --syn --dport 80 -m connlimit --connlimit-above 100 -j CONNLIMIT
iptables -A CONNLIMIT -p tcp -j REJECT --reject-with tcp-reset
```
- nft:
```
# either inline
add rule inet csf input tcp dport 80 ct state new ct count over 100 reject with tcp reset
# or via chain
add chain inet csf CONNLIMIT
add rule inet csf input tcp dport 80 ct state new ct count over 100 jump CONNLIMIT
add rule inet csf CONNLIMIT log prefix "Firewall: *ConnLimit* " level info
add rule inet csf CONNLIMIT reject with tcp reset
```

Example 8: PORTFLOOD approximation (per-source window)
- nft:
```
add set inet csf pf_22 { type ipv4_addr; flags timeout; }
# Drop if in window
add rule inet csf input tcp dport 22 ip saddr @pf_22 log prefix "PortFlood:22" level info drop
# Arm window on first packet
add rule inet csf input tcp dport 22 update @pf_22 { ip saddr timeout 10s }
# Optional global limiter to approximate hitcount
add rule inet csf input tcp dport 22 limit rate over 5/10seconds drop
```

Example 9: TRACE equivalent
- iptables:
```
iptables -t raw -I PREROUTING -p tcp --syn -s 198.51.100.10 -j TRACE
```
- nft:
```
add chain inet csf trace_in { type filter hook prerouting priority -300; }
add rule inet csf trace_in ip saddr 198.51.100.10 tcp flags syn meta nftrace set 1 log prefix "TRACE:" level debug
```

Example 10: Docker MASQUERADE
- iptables:
```
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```
- nft:
```
add table ip csf_nat
add chain ip csf_nat postrouting { type nat hook postrouting priority 0; }
add rule  ip csf_nat postrouting ip saddr 172.17.0.0/16 oifname != "docker0" masquerade
```

Example 11: IPv6 allows (inet table)
- nft: (inet family covers both v4 and v6 for filter chains)
```
add rule inet csf ALLOWIN tcp dport 443 accept
add rule inet csf ALLOWIN udp dport 53 accept
```
For IPv6 NAT (if needed): use `table ip6 csf_nat6` analogues.

Example 12: Logging parity lines (blocked)
- nft:
```
add rule inet csf LOGDROPIN meta l4proto tcp log prefix "Firewall: *TCP_IN Blocked* " level info
add rule inet csf LOGDROPIN meta l4proto tcp drop
add rule inet csf LOGDROPOUT meta l4proto udp log prefix "Firewall: *UDP_OUT Blocked* " level info
add rule inet csf LOGDROPOUT meta l4proto udp drop
```


Appendix B: Advanced Allow/Deny Filters (csf.allow/csf.deny) → nft

Reference format (from readme):
- `tcp|in|s/d=port|s/d=ip|u=uid` (also `udp` or `icmp`, `g=gid` supported)
- Defaults: protocol=tcp, direction=in if omitted
- Port range: `2000_3000` → nft `dport 2000-3000`; multiport: `22,80,443` → `dport {22,80,443}`

Directional mapping
- `in` → chain `input`
- `out` → chain `output`

Endpoint mapping
- `s=IP` → source match: `ip saddr IP` (or `ip6 saddr` for IPv6 entries)
- `d=IP` → destination match: `ip daddr IP`

Port mapping
- `s=port` (source port) → `tcp sport X` / `udp sport X`
- `d=port` (destination port) → `tcp dport X` / `udp dport X`

Owner mapping
- `u=UID` → `meta skuid UID` (implies output direction in CSF; in nft, still place in output chain)
- `g=GID` → `meta skgid GID`

ICMP mapping
- CSF’s `d=ping` → nft icmp type `echo-request` (v4) or icmpv6 type `echo-request` (v6)

Concrete translations
1) `tcp|in|d=3306|s=11.22.33.44`
```
add rule inet csf ALLOWIN ip saddr 11.22.33.44 tcp dport 3306 accept
```

2) `tcp|out|d=22|d=11.22.33.44` (note: repeated d= is destination IP in readme example)
```
add rule inet csf ALLOWOUT ip daddr 11.22.33.44 tcp dport 22 accept
```

3) `d=22|s=44.33.22.11` (defaults tcp,in)
```
add rule inet csf ALLOWIN ip saddr 44.33.22.11 tcp dport 22 accept
```

4) `tcp|out|d=80||u=99` (UID 99 outbound to port 80)
```
add rule inet csf ALLOWOUT meta skuid 99 tcp dport 80 accept
```

5) `icmp|in|d=ping|s=44.33.22.11`
```
add rule inet csf ALLOWIN ip saddr 44.33.22.11 ip protocol icmp icmp type echo-request accept
```

6) `tcp|in|d=22|s=www.configserver.com` (used in csf.dyndns)
- Resolve FQDN to IP(s) first (as CSF already does), then add one rule or set elements per IP:
```
add rule inet csf ALLOWIN ip saddr 203.0.113.10 tcp dport 22 accept
add rule inet csf ALLOWIN ip saddr 203.0.113.11 tcp dport 22 accept
```

7) `d=22,80,443|s=44.33.22.11` (multiport)
```
add rule inet csf ALLOWIN ip saddr 44.33.22.11 tcp dport {22,80,443} accept
```

8) Port range example: `tcp|in|d=2000_3000|s=198.51.100.5`
```
add rule inet csf ALLOWIN ip saddr 198.51.100.5 tcp dport 2000-3000 accept
```

9) Source port match example: `udp|in|s=53|s=203.0.113.7`
```
add rule inet csf ALLOWIN ip saddr 203.0.113.7 udp sport 53 accept
```

10) Outbound GID restriction example: `tcp|out|d=443|g=48`
```
add rule inet csf ALLOWOUT meta skgid 48 tcp dport 443 accept
```

Notes
- In CSF, advanced allows/denies may be inserted into specific chains based on feature order; in nft backend, emit these into `ALLOWIN/ALLOWOUT` for allows and into `DENYIN/DENYOUT` (or a dedicated subchain) for denies to preserve behavior.
- When IPv6 addresses are present, emit parallel `ip6 saddr/daddr` rules or prefer inet family with dual stacks where possible.
- For dynamic DNS entries, retain CSF’s existing resolution cadence; on change, update rules or (preferably) sets to avoid rule churn.


Appendix C: DROP_NOLOG and Logging Controls in nft
- CSF’s `DROP_NOLOG` lists ports where drops should not be logged in LOGDROPIN chain.
- nft backend approach: place “no-log drop” rules for the specified ports before the general log+drop rules.

Example (no logging for tcp dport 113 and 500):
```
# No-log drops first
add rule inet csf LOGDROPIN tcp dport {113,500} drop
# Logged drops follow
add rule inet csf LOGDROPIN meta l4proto tcp log prefix "Firewall: *TCP_IN Blocked* " level info
add rule inet csf LOGDROPIN meta l4proto tcp drop
```
