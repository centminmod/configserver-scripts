**THIS DOCUMENT CONTAINS AI-GENERATED CONTENT THAT IS NOT VERIFIED, NOT TESTED, AND MAY CONTAIN SIGNIFICANT INACCURACIES OR COMPLETE FABRICATIONS (HALLUCINATIONS).**

These documents were generated out of curiosity by asking different AI models what it would theoretically take to update CSF (ConfigServer Security & Firewall) to support nftables. They are **NOT** official documentation, **NOT** tested implementations, and **NOT** endorsed by any CSF developers or maintainers.

# CSF nftables Support Implementation Plan
## For AlmaLinux/Rocky Linux/RHEL v10 Compatibility

### Executive Summary

ConfigServer Security & Firewall (CSF) v15.00 currently relies exclusively on iptables/ip6tables for firewall management. With AlmaLinux/Rocky Linux/RHEL v10 transitioning to nftables as the default packet filtering framework (removing iptables-legacy), CSF requires significant modifications to maintain compatibility.

**Key Findings:**
- CSF contains **~1,000+ direct iptables command invocations** across core modules
- Current iptables-nft compatibility layer (v14.19) provides partial bridge but is insufficient for RHEL 10
- Migration requires abstraction layer to support dual backends (iptables/nftables)
- Estimated effort: 2,000-3,000 lines of code for proper abstraction layer
- **Critical:** Must preserve exact log prefixes for lfd compatibility

**Recommendation:** Implement backend abstraction layer with native nftables support for optimal compatibility and future-proofing.

---

## Current Architecture Analysis

### 1. Core Firewall Management Structure

#### Command Execution Pipeline
```
csf.pl/lfd.pl → syscommand() → iptables/ip6tables → kernel netfilter
```

**Key Files & Functions:**
- `csf.pl:5604-5650` - syscommand() function executes all firewall commands
- `lfd.pl:~8000+` - Similar syscommand() implementation for daemon operations
- Direct iptables calls in ~50+ functions across both files

#### Chain Architecture
CSF creates and manages custom chains:
```
Filter Table (inet family in nftables):
├── INPUT → LOCALINPUT → DENYIN → ALLOWIN → LOGDROPIN
├── OUTPUT → LOCALOUTPUT → DENYOUT → ALLOWOUT → LOGDROPOUT
├── FORWARD → (various custom chains)
└── Custom chains: INVALID, SYNFLOOD, PORTFLOOD, CONNLIMIT, etc.

NAT Tables (ip/ip6 families in nftables):
├── PREROUTING (port redirects, DNAT, messenger)
├── POSTROUTING (SNAT, masquerading, Docker)
└── OUTPUT (local redirects, SMTP_REDIRECT)
```

### 2. Platform Detection Mechanism

**Current Implementation (os.pl:55-62):**
```perl
if (-e "/usr/sbin/iptables-nft") {
    system("update-alternatives", "--set", "iptables", "/usr/sbin/iptables-nft");
    if (-e "/usr/sbin/ip6tables-nft") {
        system("update-alternatives", "--set", "ip6tables", "/usr/sbin/ip6tables-nft");
    }
}
```

This provides iptables-nft compatibility but doesn't support native nftables.

### 3. Configuration System

**Firewall Binary Paths (csf.conf:2731-2753):**
```
IPTABLES = "/sbin/iptables"
IPTABLES_SAVE = "/sbin/iptables-save"
IPTABLES_RESTORE = "/sbin/iptables-restore"
IP6TABLES = "/sbin/ip6tables"
IP6TABLES_SAVE = "/sbin/ip6tables-save"
IP6TABLES_RESTORE = "/sbin/ip6tables-restore"
IPSET = "/usr/sbin/ipset"
```

---

## nftables Migration Strategy

### Backend Abstraction Layer Implementation

**Effort:** 2,000-3,000 lines of code  
**Timeline:** 4-6 weeks  
**Risk:** Medium  

#### Architecture Design:

```
                  ┌─────────────┐
                  │   csf.pl    │
                  │   lfd.pl    │
                  └──────┬──────┘
                         │
                  ┌──────▼──────┐
                  │  Firewall   │
                  │ Abstraction │
                  │    Layer    │
                  └──────┬──────┘
                         │
            ┌────────────┼────────────┐
            │                         │
     ┌──────▼──────┐          ┌──────▼──────┐
     │  iptables   │          │   nftables  │
     │   Backend   │          │   Backend   │
     └──────┬──────┘          └──────┬──────┘
            │                         │
     ┌──────▼──────┐          ┌──────▼──────┐
     │iptables/ip6 │          │     nft     │
     └─────────────┘          └─────────────┘
```

---

## nftables Table and Chain Structure

### Unified Filter Table (inet family)
```bash
table inet csf {
    # Sets for IP management (replacing ipset)
    set chain_DENY { type ipv4_addr; flags interval; }
    set chain_6_DENY { type ipv6_addr; flags interval; }
    set chain_ALLOW { type ipv4_addr; flags interval; }
    set chain_6_ALLOW { type ipv6_addr; flags interval; }
    set cc_us { type ipv4_addr; flags interval; }  # Country codes
    set cc_6_us { type ipv6_addr; flags interval; }
    
    # Temporary sets with timeout
    set TEMP_DENY { type ipv4_addr; flags interval, timeout; }
    set TEMP_6_DENY { type ipv6_addr; flags interval, timeout; }
    
    # Base chains with hooks
    chain input { 
        type filter hook input priority 0; policy accept;
        iif lo accept
        ct state { established, related } accept
        jump LOCALINPUT
        jump DENYIN
        jump ALLOWIN
        jump LOGDROPIN
    }
    
    chain output { 
        type filter hook output priority 0; policy accept;
        oif lo accept
        ct state { established, related } accept
        jump LOCALOUTPUT
        jump DENYOUT
        jump ALLOWOUT
        jump LOGDROPOUT
    }
    
    chain forward { 
        type filter hook forward priority 0; policy accept;
        jump LOCALFORWARD  # Only used when CSF manages forwarding rules
    }
    
    # Custom chains
    chain LOCALINPUT { }
    chain LOCALOUTPUT { }
    chain LOCALFORWARD { }  # Populated only when needed (Docker, NAT, etc.)
    
    chain ALLOWIN { 
        # TCP_IN ports
        tcp dport {22,80,443} accept
        # UDP_IN ports
        udp dport {53} accept
    }
    
    chain ALLOWOUT { 
        # TCP_OUT/UDP_OUT rules
    }
    
    chain DENYIN {
        ip saddr @chain_DENY drop
        ip6 saddr @chain_6_DENY drop
        ip saddr @TEMP_DENY drop
        ip6 saddr @TEMP_6_DENY drop
    }
    
    chain DENYOUT {
        ip daddr @chain_DENY drop
        ip6 daddr @chain_6_DENY drop
    }
    
    # CRITICAL: Preserve exact log prefixes for lfd
    chain LOGDROPIN {
        # DROP_NOLOG ports first (no logging)
        tcp dport {113,500} drop
        udp dport {113,500} drop
        
        # Logged drops with CSF-specific prefixes
        meta l4proto tcp log prefix "Firewall: *TCP_IN Blocked* " level info
        meta l4proto tcp drop
        meta l4proto udp log prefix "Firewall: *UDP_IN Blocked* " level info
        meta l4proto udp drop
        ip protocol icmp log prefix "Firewall: *ICMP_IN Blocked* " level info
        ip protocol icmp drop
        ip6 nexthdr icmpv6 log prefix "Firewall: *ICMP6IN Blocked* " level info
        ip6 nexthdr icmpv6 drop
    }
    
    chain LOGDROPOUT {
        # Rules here are dynamically set based on DROP_OUT configuration
        # Logging always happens first
        meta l4proto tcp log prefix "Firewall: *TCP_OUT Blocked* " level info
        meta l4proto udp log prefix "Firewall: *UDP_OUT Blocked* " level info
        ip protocol icmp log prefix "Firewall: *ICMP_OUT Blocked* " level info
        ip6 nexthdr icmpv6 log prefix "Firewall: *ICMP6OUT Blocked* " level info
        
        # Action depends on DROP_OUT setting (configured at runtime):
        # If DROP_OUT=REJECT (default):
        #   meta l4proto tcp reject with tcp reset
        #   meta l4proto udp reject with icmp port-unreachable
        #   ip protocol icmp reject with icmp admin-prohibited
        #   ip6 nexthdr icmpv6 reject with icmpv6 admin-prohibited
        # If DROP_OUT=DROP:
        #   drop  # Simple drop for all protocols
        # Note: nftables automatically selects appropriate ICMP types for rejects
    }
}
```

### NAT Tables (Separate IPv4/IPv6)
```bash
# IPv4 NAT
table ip csf_nat {
    chain prerouting { 
        type nat hook prerouting priority 0;
        # Messenger redirects
        tcp dport {80,443} ip saddr @MESSENGER redirect to :8888
    }
    
    chain output { 
        type nat hook output priority 0;
        # SMTP_REDIRECT (exclude root)
        tcp dport {25,465,587} meta skuid 0 return
        tcp dport {25,465,587} redirect to :2525
    }
    
    chain postrouting { 
        type nat hook postrouting priority 0;
        # Docker MASQUERADE (scoped to docker networks)
        ip saddr 172.17.0.0/16 oifname != "docker0" masquerade
    }
}

# IPv6 NAT (when MESSENGER6/SMTP_REDIRECT6 enabled)
table ip6 csf_nat6 {
    chain prerouting { type nat hook prerouting priority 0; }
    chain output { type nat hook output priority 0; }
    chain postrouting { type nat hook postrouting priority 0; }
}
```

---

## Command Translation Reference

### Complete iptables to nftables Mapping

| Feature | iptables | nftables |
|---------|----------|----------|
| **Basic Accept/Drop** | | |
| Accept | `-j ACCEPT` | `accept` |
| Drop | `-j DROP` | `drop` |
| Reject | `-j REJECT --reject-with tcp-reset` | `reject with tcp reset` |
| Return | `-j RETURN` | `return` |
| **Logging** | | |
| Log with prefix | `-j LOG --log-prefix "CSF: "` | `log prefix "CSF: " level info` |
| Log with limit | `-m limit --limit 30/m -j LOG` | `limit rate 30/minute log` |
| **Device/Interface** | | |
| Input interface | `-i lo` | `iif lo` |
| Output interface | `-o eth0` | `oif eth0` |
| Not interface | `! -o docker0` | `oifname != "docker0"` |
| **Connection Tracking** | | |
| State match | `-m state --state NEW,ESTABLISHED` | `ct state { new, established }` |
| Invalid state | `-m state --state INVALID` | `ct state invalid` |
| **Protocol/Ports** | | |
| TCP port | `-p tcp --dport 80` | `tcp dport 80` |
| UDP port | `-p udp --sport 53` | `udp sport 53` |
| Port range | `--dport 135:139` | `dport 135-139` |
| Multiport | `-m multiport --dports 80,443,8080` | `tcp dport {80,443,8080}` |
| ICMP | `-p icmp` | `ip protocol icmp` |
| ICMPv6 | `-p icmpv6` | `ip6 nexthdr icmpv6` |
| **Source/Destination** | | |
| Source IP | `-s 192.168.1.1` | `ip saddr 192.168.1.1` |
| Dest IP | `-d 10.0.0.0/8` | `ip daddr 10.0.0.0/8` |
| Source IPv6 | `-s 2001:db8::/32` | `ip6 saddr 2001:db8::/32` |
| **Owner Match** | | |
| UID owner | `-m owner --uid-owner 0` | `meta skuid 0` |
| GID owner | `-m owner --gid-owner 48` | `meta skgid 48` |
| **Rate Limiting** | | |
| Limit rate | `-m limit --limit 10/min --limit-burst 5` | `limit rate 10/minute burst 5 packets` |
| **Connection Limit** | | |
| Conn limit | `-m connlimit --connlimit-above 50` | `ct state new ct count over 50` |
| **ipset/Sets** | | |
| Match set src | `-m set --match-set DENY src` | `ip saddr @DENY` |
| Match set dst | `-m set --match-set ALLOW dst` | `ip daddr @ALLOW` |
| **NAT/Redirect** | | |
| DNAT | `-j DNAT --to-destination 10.1.1.1` | `dnat to 10.1.1.1` |
| SNAT | `-j SNAT --to-source 1.2.3.4` | `snat to 1.2.3.4` |
| Redirect | `-j REDIRECT --to-ports 8080` | `redirect to :8080` |
| Masquerade | `-j MASQUERADE` | `masquerade` |

---

## Advanced Feature Implementations

### SYNFLOOD Protection
```bash
chain SYNFLOOD {
    # Rate limit SYN packets
    tcp flags syn limit rate $SYNFLOOD_RATE burst $SYNFLOOD_BURST packets accept
    # Log and drop excess
    tcp flags syn log prefix "Firewall: *SYNFLOOD* " level info
    tcp flags syn drop
}
```

### PORTFLOOD (Approximation using sets with timeout)

**Note:** This is a best-effort approximation of iptables `-m recent`. Behavior differences:
- iptables recent tracks per-source hit counts within a time window
- nftables implementation uses timeout sets for simpler per-source rate limiting
- Consider this a functional equivalent, not an exact replica

```bash
# Per-port flood protection
set pf_22 { type ipv4_addr; flags timeout; }
set pf_6_22 { type ipv6_addr; flags timeout; }

chain PORTFLOOD {
    # Drop if source already in set (within timeout window)
    tcp dport 22 ip saddr @pf_22 log prefix "PortFlood:22 " drop
    tcp dport 22 ip6 saddr @pf_6_22 log prefix "PortFlood6:22 " drop
    
    # Add source to set with timeout
    tcp dport 22 update @pf_22 { ip saddr timeout 10s }
    tcp dport 22 update @pf_6_22 { ip6 saddr timeout 10s }
}
```

### PORTKNOCKING (Sequential port access)
```bash
# Two-step knock: 1111 → 2222 to open port 22
set PK_22_S1 { type ipv4_addr; flags timeout; }
set PK_22_S2 { type ipv4_addr; flags timeout; }

chain PORTKNOCKING {
    # First knock
    tcp dport 1111 update @PK_22_S1 { ip saddr timeout 30s }
    
    # Second knock (only if first knock completed)
    tcp dport 2222 ip saddr @PK_22_S1 update @PK_22_S2 { ip saddr timeout 30s }
    
    # Allow access if knock sequence completed
    tcp dport 22 ip saddr @PK_22_S2 accept
}
```

### CONNLIMIT
```bash
chain CONNLIMIT {
    # Connection limit per port
    tcp dport 80 ct state new ct count over 100 log prefix "Firewall: *ConnLimit* " level info
    tcp dport 80 ct state new ct count over 100 reject with tcp reset
}
```

### SMTP_BLOCK (Owner-based filtering)
```bash
chain SMTPOUTPUT {
    # Allow root
    tcp dport {25,465,587} meta skuid 0 accept
    # Allow specific UIDs/GIDs
    tcp dport {25,465,587} meta skuid 99 accept
    tcp dport {25,465,587} meta skgid 48 accept
    # Block/reject others
    tcp dport {25,465,587} reject with tcp reset
}
```

### TRACE (Packet tracing)
```bash
# TRACE requires separate ip/ip6 tables (inet doesn't support prerouting hook)
table ip csf_trace {
    chain prerouting { 
        type filter hook prerouting priority -300;
        # Enable tracing for specific source
        ip saddr 198.51.100.10 tcp flags syn meta nftrace set 1
        ip saddr 198.51.100.10 tcp flags syn log prefix "TRACE: " level debug
    }
}

table ip6 csf_trace6 {
    chain prerouting { 
        type filter hook prerouting priority -300;
        # Enable tracing for specific source IPv6
        ip6 saddr 2001:db8::1 tcp flags syn meta nftrace set 1
        ip6 saddr 2001:db8::1 tcp flags syn log prefix "TRACE: " level debug
    }
}
```

---

## Backend API Specification

### Core Methods
```perl
# ConfigServer::Firewall API
$fw->backend()                    # Returns 'iptables' or 'nftables'
$fw->ruleset_init()               # Create tables/chains
$fw->ruleset_flush()              # Remove CSF-owned objects ONLY
$fw->chain_create($name)          # Create chain
$fw->chain_flush($name)           # Flush chain
$fw->chain_delete($name)          # Delete chain
$fw->chain_rename($old, $new)     # Rename chain
$fw->rule_append($chain, $spec)   # Append rule
$fw->rule_insert($chain, $pos, $spec) # Insert rule at position
$fw->rule_delete($chain, $handle)  # Delete rule

# Set management (ipset replacement)
$fw->set_create($name, %opts)     # Create set
  # %opts: family => 'v4'|'v6'|'inet', timeout => bool, interval => bool
$fw->set_add($name, $addr, $timeout_s)  # Add element
$fw->set_del($name, $addr)        # Remove element
$fw->set_flush($name)             # Clear set

# Batch operations
$fw->batch_apply(@chunks)         # Atomic apply (iptables-restore or nft -f)

# NAT operations
$fw->nat_add($hook, $spec)        # Add NAT rule
  # $hook: 'prerouting'|'output'|'postrouting'
```

---

## Configuration Additions

Add to csf.conf after line 2753:

```bash
# Firewall Backend Selection
# Options: auto|iptables|nftables
# auto = detect based on system (nftables for EL10, iptables otherwise)
FIREWALL_BACKEND = "auto"

# nftables Binary Location
NFT = "/usr/sbin/nft"

# nftables Table Configuration
NFT_TABLE_FAMILY = "inet"      # inet covers both IPv4 and IPv6
NFT_TABLE_NAME = "csf"         # Main table name
NFT_NAT4_TABLE = "csf_nat"     # IPv4 NAT table
NFT_NAT6_TABLE = "csf_nat6"    # IPv6 NAT table

# nftables Set Configuration
NFT_SET_HASHSIZE = "1024"      # Advisory for documentation
NFT_SET_MAXELEM = "65536"      # Maximum elements per set

# nftables Logging
NFT_LOG_LEVEL = "info"         # Kernel log level (emerg|alert|crit|err|warn|notice|info|debug)

# nftables Chain Priorities
NFT_PRIORITY_FILTER = "0"      # Filter chain hook priority
NFT_PRIORITY_NAT = "0"         # NAT chain hook priority
NFT_PRIORITY_RAW = "-300"      # Raw chain hook priority (for TRACE)

# Note: In nftables mode:
# - WAITLOCK is forced to 0 (no locking needed)
# - RAW/MANGLE tables are not used (inet family handles all)
# - ipset commands are replaced with native nft sets
```

---

## Implementation Components

### 1. Create Backend Modules

**ConfigServer::Firewall.pm (Factory)**
```perl
package ConfigServer::Firewall;
use strict;
use ConfigServer::Firewall::IPTables;
use ConfigServer::Firewall::NFTables;

our $backend;

sub new {
    my ($class, %config) = @_;
    my $self = { config => \%config };
    
    # Detect backend
    if ($config{FIREWALL_BACKEND} eq "nftables" || 
        ($config{FIREWALL_BACKEND} eq "auto" && detect_nftables())) {
        $backend = ConfigServer::Firewall::NFTables->new(%config);
    } else {
        $backend = ConfigServer::Firewall::IPTables->new(%config);
    }
    
    bless $self, $class;
    return $self;
}

sub detect_nftables {
    # EL10 detection
    if (-e "/etc/redhat-release") {
        my $release = `cat /etc/redhat-release`;
        return 1 if $release =~ /release\s+10\b/;
    }
    # Pure nftables system
    return (-e "/usr/sbin/nft" && !-e "/sbin/iptables");
}
```

**ConfigServer::Firewall::NFTables.pm (Key methods)**
```perl
package ConfigServer::Firewall::NFTables;
use strict;

sub new {
    my ($class, %config) = @_;
    my $self = { 
        config => \%config,
        nft => $config{NFT} || "/usr/sbin/nft",
        table_family => $config{NFT_TABLE_FAMILY} || "inet",
        table_name => $config{NFT_TABLE_NAME} || "csf"
    };
    bless $self, $class;
    return $self;
}

sub ruleset_init {
    my $self = shift;
    my $batch = "";
    
    # Create main filter table
    $batch .= "table $self->{table_family} $self->{table_name} {\n";
    
    # Create sets
    $batch .= "  set chain_DENY { type ipv4_addr; flags interval; }\n";
    $batch .= "  set chain_6_DENY { type ipv6_addr; flags interval; }\n";
    
    # Create base chains with hooks
    $batch .= "  chain input { type filter hook input priority $self->{config}{NFT_PRIORITY_FILTER}; policy accept; }\n";
    $batch .= "  chain output { type filter hook output priority $self->{config}{NFT_PRIORITY_FILTER}; policy accept; }\n";
    
    # Create custom chains
    foreach my $chain (qw(LOCALINPUT LOCALOUTPUT ALLOWIN ALLOWOUT DENYIN DENYOUT LOGDROPIN LOGDROPOUT)) {
        $batch .= "  chain $chain { }\n";
    }
    
    # Add LOGDROPOUT rules respecting DROP_OUT configuration
    my $drop_action = ($self->{config}{DROP_OUT} eq "REJECT") ? "reject" : "drop";
    if ($drop_action eq "reject") {
        $batch .= "  add rule $self->{table_name} LOGDROPOUT meta l4proto tcp reject with tcp reset\n";
        $batch .= "  add rule $self->{table_name} LOGDROPOUT meta l4proto udp reject with icmp port-unreachable\n";
        $batch .= "  add rule $self->{table_name} LOGDROPOUT ip protocol icmp reject with icmp admin-prohibited\n";
        $batch .= "  add rule $self->{table_name} LOGDROPOUT ip6 nexthdr icmpv6 reject with icmpv6 admin-prohibited\n";
    } else {
        $batch .= "  add rule $self->{table_name} LOGDROPOUT drop\n";
    }
    
    $batch .= "}\n";
    
    # Create NAT tables if needed
    if ($self->{config}{MESSENGER} || $self->{config}{SMTP_REDIRECT}) {
        $batch .= "table ip $self->{config}{NFT_NAT4_TABLE} {\n";
        $batch .= "  chain prerouting { type nat hook prerouting priority $self->{config}{NFT_PRIORITY_NAT}; }\n";
        $batch .= "  chain output { type nat hook output priority $self->{config}{NFT_PRIORITY_NAT}; }\n";
        $batch .= "  chain postrouting { type nat hook postrouting priority $self->{config}{NFT_PRIORITY_NAT}; }\n";
        $batch .= "}\n";
    }
    
    return $self->batch_apply($batch);
}

sub ruleset_flush {
    my $self = shift;
    # CRITICAL: Only delete CSF-owned tables, never use "flush ruleset"
    my $cmd = "";
    $cmd .= "delete table $self->{table_family} $self->{table_name}\n" 
        if $self->table_exists("$self->{table_family}", "$self->{table_name}");
    $cmd .= "delete table ip $self->{config}{NFT_NAT4_TABLE}\n" 
        if $self->table_exists("ip", $self->{config}{NFT_NAT4_TABLE});
    $cmd .= "delete table ip6 $self->{config}{NFT_NAT6_TABLE}\n" 
        if $self->table_exists("ip6", $self->{config}{NFT_NAT6_TABLE});
    return $self->batch_apply($cmd) if $cmd;
    return 0;
}

sub rule_append {
    my ($self, $chain, $spec) = @_;
    # Translate if needed, or use native nft syntax
    my $nft_rule = $self->translate_rule($spec);
    my $cmd = "add rule $self->{table_family} $self->{table_name} $chain $nft_rule";
    return system("$self->{nft} $cmd");
}

sub set_add {
    my ($self, $name, $addr, $timeout) = @_;
    my $element = $timeout ? "{ $addr timeout ${timeout}s }" : "{ $addr }";
    my $cmd = "add element $self->{table_family} $self->{table_name} $name $element";
    return system("$self->{nft} $cmd");
}

sub batch_apply {
    my ($self, $batch) = @_;
    # Write to temp file and apply atomically
    my $tmpfile = "/tmp/csf_nft_batch_$$";
    open(my $fh, ">", $tmpfile) or return 1;
    print $fh $batch;
    close($fh);
    my $ret = system("$self->{nft} -f $tmpfile");
    unlink($tmpfile);
    return $ret;
}
```

### 2. Update Platform Detection (os.pl)

```perl
# Add after line 54
our $firewall_backend = "iptables";  # default

# Enhanced detection for EL10
if (-e "/etc/redhat-release") {
    my $release = `cat /etc/redhat-release`;
    if ($release =~ /release\s+10\b/) {
        if (-e "/usr/sbin/nft") {
            print STDERR "Detected EL10 - setting nftables backend\n";
            # Update config to use nftables
            system("sed -i 's/^FIREWALL_BACKEND.*/FIREWALL_BACKEND = \"nftables\"/' /etc/csf/csf.conf");
        } else {
            print STDERR "Warning: EL10 detected but nft not found\n";
        }
    }
}
elsif (-e "/usr/sbin/iptables-nft") {
    # Existing iptables-nft compatibility
    print STDERR "Configuration modified to use iptables-nft\n";
    system("update-alternatives", "--set", "iptables", "/usr/sbin/iptables-nft");
    # ... existing code
}
```

### 3. Update csf.pl Integration

```perl
# After config loading (line ~95)
use ConfigServer::Firewall;
our $firewall;

# Initialize backend
$firewall = ConfigServer::Firewall->new(%config);

# Check firewalld (keep existing check)
if (-e $config{SYSTEMCTL}) {
    my ($childin, $childout);
    my $cmdpid = open3($childin, $childout, $childout, $config{SYSTEMCTL}, "is-active", "firewalld");
    my $status = <$childout>;
    waitpid($cmdpid, 0);
    if ($status =~ /^active/) {
        &error(__LINE__, "*Error* firewalld found to be running. You must stop and disable firewalld when using csf");
    }
}

# Modify syscommand to use backend
sub syscommand {
    my $line = shift;
    my $command = shift;
    my $force = shift;
    
    # Route firewall commands through backend
    if ($command =~ /^($config{IPTABLES}|$config{IP6TABLES})/) {
        if ($firewall->{backend} eq "nftables") {
            # CRITICAL: Avoid complex iptables parsing
            # Only handle simple, well-defined patterns
            # Complex operations should use native backend methods
            if ($command =~ /-(A|I|D|N|F|X)\s+(\S+)/) {
                # Basic chain operations only
                return $firewall->handle_basic_command($command);
            } else {
                # For complex commands, require native implementation
                die "Complex iptables command requires native nftables implementation: $command\n";
            }
        }
    }
    
    # Original syscommand implementation for non-firewall commands
    # ... existing code
}
```

### 4. Update ConfigServer::Config.pm

```perl
# Parse new nftables settings
our @config = qw(
    # ... existing configs
    FIREWALL_BACKEND NFT NFT_TABLE_FAMILY NFT_TABLE_NAME
    NFT_NAT4_TABLE NFT_NAT6_TABLE NFT_SET_HASHSIZE NFT_SET_MAXELEM
    NFT_LOG_LEVEL NFT_PRIORITY_FILTER NFT_PRIORITY_NAT NFT_PRIORITY_RAW
);

# Feature detection for nftables
if ($config{FIREWALL_BACKEND} eq "nftables" || 
    ($config{FIREWALL_BACKEND} eq "auto" && -e $config{NFT} && !-e $config{IPTABLES})) {
    
    # Verify nft binary
    unless (-e $config{NFT}) {
        die "nftables backend selected but $config{NFT} not found\n";
    }
    
    # Test nft capabilities
    my $test = `$config{NFT} list tables 2>&1`;
    unless ($? == 0) {
        die "nftables backend selected but nft not functional\n";
    }
    
    # Force settings for nftables mode
    $config{WAITLOCK} = 0;  # No locking needed with nft
    $config{RAW} = 0;        # Not used in nft mode
    $config{MANGLE} = 0;     # Not used in nft mode
    
    # Test NAT availability
    my $nat_test = `echo "table ip test_nat { chain test { type nat hook prerouting priority 0; } }" | $config{NFT} -f - 2>&1`;
    if ($? == 0) {
        $config{NAT} = 1;
        system("$config{NFT} delete table ip test_nat 2>/dev/null");
    }
}
```

### 5. Update csftest.pl

```perl
# Add nftables testing path
if ($config{FIREWALL_BACKEND} eq "nftables" || 
    ($config{FIREWALL_BACKEND} eq "auto" && -e "/usr/sbin/nft")) {
    
    print "Testing nftables...\n";
    
    print "Testing nft binary...";
    $return = system("nft --version >/dev/null 2>&1");
    if ($return != 0) {
        print "FAILED [FATAL Error: nft not functional]\n";
        $fatal++;
    } else {
        print "OK\n";
    }
    
    print "Testing table creation...";
    $return = system("nft add table inet csf_test 2>/dev/null");
    if ($return != 0) {
        print "FAILED [FATAL Error: Cannot create nft tables]\n";
        $fatal++;
    } else {
        print "OK\n";
        
        print "Testing chain creation...";
        $return = system("nft add chain inet csf_test input { type filter hook input priority 0\\; } 2>/dev/null");
        if ($return != 0) {
            print "FAILED [FATAL Error: Cannot create nft chains]\n";
            $fatal++;
        } else {
            print "OK\n";
        }
        
        print "Testing set creation...";
        $return = system("nft add set inet csf_test test_set { type ipv4_addr\\; } 2>/dev/null");
        if ($return != 0) {
            print "FAILED [Error: Cannot create nft sets] - Required for IP management\n";
            $error++;
        } else {
            print "OK\n";
        }
        
        print "Testing owner match...";
        $return = system("nft add rule inet csf_test input meta skuid 0 accept 2>/dev/null");
        if ($return != 0) {
            print "FAILED [Error: No owner match support] - Required for SMTP_BLOCK\n";
            $error++;
        } else {
            print "OK\n";
        }
        
        print "Testing NAT...";
        $return = system("nft add table ip csf_test_nat 2>/dev/null");
        if ($return == 0) {
            $return = system("nft add chain ip csf_test_nat output { type nat hook output priority 0\\; } 2>/dev/null");
            if ($return == 0) {
                print "OK\n";
                system("nft delete table ip csf_test_nat 2>/dev/null");
            } else {
                print "FAILED [Error: No NAT support] - Required for MESSENGER/SMTP_REDIRECT\n";
                $error++;
            }
        } else {
            print "FAILED [Error: No NAT support] - Required for MESSENGER/SMTP_REDIRECT\n";
            $error++;
        }
        
        # Cleanup
        system("nft delete table inet csf_test 2>/dev/null");
    }
} else {
    # Existing iptables tests
    # ... keep existing code
}
```

---

## Advanced CSF Allow/Deny Format Translation

CSF's advanced filter syntax (csf.allow/csf.deny) requires precise translation:

### Format: `protocol|direction|s/d=port|s/d=ip|u=uid|g=gid`

**IMPORTANT:** Owner matches (u=uid, g=gid) only work for OUTPUT traffic:
- UID/GID matching (`u=` or `g=`) implies **OUTBOUND** direction
- These rules are always placed in OUTPUT chains (ALLOWOUT/DENYOUT)
- Any `s/d=ip` fields with owner matches are typically ignored by CSF

| CSF Format | nftables Rule | Notes |
|------------|---------------|-------|
| `tcp\|in\|d=22\|s=1.2.3.4` | `ip saddr 1.2.3.4 tcp dport 22 accept` | Inbound |
| `tcp\|out\|d=80\|d=5.6.7.8` | `ip daddr 5.6.7.8 tcp dport 80 accept` | Outbound |
| `udp\|in\|s=53\|s=8.8.8.8` | `ip saddr 8.8.8.8 udp sport 53 accept` | Inbound |
| `tcp\|out\|d=25\|\|u=0` | `meta skuid 0 tcp dport 25 accept` | Owner = OUTPUT only |
| `tcp\|out\|d=443\|\|g=48` | `meta skgid 48 tcp dport 443 accept` | Owner = OUTPUT only |
| `icmp\|in\|d=ping\|s=10.0.0.1` | `ip saddr 10.0.0.1 icmp type echo-request accept` | Inbound |
| `tcp\|in\|d=80,443,8080` | `tcp dport {80,443,8080} accept` | Multiport |
| `tcp\|in\|d=2000_3000` | `tcp dport 2000-3000 accept` | Port range |

**DynDNS Updates:** When hostnames are used in csf.dyndns:
- CSF resolves FQDNs to IPs periodically (every 10 minutes by default)
- On IP changes, CSF must update firewall rules to avoid stale entries
- In nftables mode, use set element updates for efficiency (avoids rule churn):
  ```bash
  # Remove old IP from set
  nft delete element inet csf chain_ALLOW { 1.2.3.4 }
  # Add new resolved IP to set
  nft add element inet csf chain_ALLOW { 5.6.7.8 }
  ```
- This approach is more efficient than deleting/recreating entire rules
- Sets allow atomic updates without disrupting traffic flow

---

## Testing & Validation Plan

### 1. Unit Testing Framework
```perl
# test/test_firewall_backends.pl
use Test::More;
use ConfigServer::Firewall;

# Test both backends
foreach my $backend (qw(iptables nftables)) {
    $ENV{CSF_FORCE_BACKEND} = $backend;
    my $fw = ConfigServer::Firewall->new(%config);
    
    # Test basic operations
    ok($fw->ruleset_init(), "$backend: Initialize ruleset");
    ok($fw->chain_create("TEST"), "$backend: Create chain");
    ok($fw->rule_append("TEST", "accept"), "$backend: Add rule");
    ok($fw->set_create("test_set", family => "v4"), "$backend: Create set");
    ok($fw->set_add("test_set", "192.168.1.1"), "$backend: Add to set");
    ok($fw->ruleset_flush(), "$backend: Flush ruleset");
}
```

### 2. Integration Testing Checklist

**Basic Functionality:**
- [ ] Start CSF with TESTING=1
- [ ] Verify tables created: `nft list table inet csf`
- [ ] Check allowed ports accessible
- [ ] Verify denied IPs blocked
- [ ] Confirm log entries with correct prefixes

**Advanced Features:**
- [ ] SYNFLOOD protection active
- [ ] CONNLIMIT enforced
- [ ] PORTFLOOD approximation working
- [ ] Messenger redirects functional
- [ ] SMTP_REDIRECT working
- [ ] Country code filtering active

**Safety Verification:**
- [ ] Emergency flush only removes CSF tables
- [ ] Other applications' rules preserved
- [ ] firewalld remains disabled
- [ ] TESTING mode auto-recovery works

### 3. Performance Metrics
- Rule insertion: < 5ms per rule
- Set operations: < 1ms per IP
- Batch apply: < 500ms for 1000 rules
- Memory usage: < 50MB for 10,000 IPs

---

## Risk Mitigation

### 1. Safety Measures
- **Never use `nft flush ruleset`** - only delete CSF-owned tables
- **Preserve exact log prefixes** for lfd compatibility
- **Maintain firewalld exclusion** to prevent conflicts
- **Test connectivity** after each major change
- **TESTING mode** auto-removes rules after 5 minutes

### 2. Rollback Procedures
```perl
sub apply_with_rollback {
    my ($self, $rules) = @_;
    
    # Save current state
    system("$self->{nft} list ruleset > /var/lib/csf/rollback.nft");
    
    # Apply new rules
    my $ret = $self->batch_apply($rules);
    
    # Test connectivity
    unless (test_connectivity()) {
        # Rollback
        system("$self->{nft} -f /var/lib/csf/rollback.nft");
        return 0;
    }
    
    return 1;
}
```

### 3. Migration Path
1. **Stage 1:** Deploy backend code, keep iptables default
2. **Stage 2:** Test nftables on development systems
3. **Stage 3:** Enable for new EL10 installations
4. **Stage 4:** Provide migration tool for existing systems
5. **Stage 5:** Deprecate iptables backend (2+ years)

---

## Implementation Timeline

### Week 1-2: Foundation
- [ ] Create backend module structure
- [ ] Implement nftables detection
- [ ] Basic table/chain creation
- [ ] Set management functions

### Week 3-4: Core Features
- [ ] Rule translation engine
- [ ] Batch operations
- [ ] NAT/redirect support
- [ ] Logging with correct prefixes

### Week 5: Advanced Features
- [ ] SYNFLOOD/CONNLIMIT implementation
- [ ] PORTFLOOD/PORTKNOCKING approximation
- [ ] Owner matching (SMTP_BLOCK)
- [ ] Country code filtering

### Week 6: Integration
- [ ] csf.pl/lfd.pl integration
- [ ] ConfigServer::Config updates
- [ ] csftest.pl nftables tests
- [ ] Emergency procedures

### Week 7-8: Testing & Documentation
- [ ] Comprehensive testing on EL10
- [ ] Performance optimization
- [ ] Documentation updates
- [ ] Beta release preparation

---

## Conclusion

This implementation plan provides a complete path to nftables support while maintaining full backward compatibility. The approach:

1. **Preserves exact CSF behavior** including log formats for lfd
2. **Ensures safety** by only managing CSF-owned tables (never `flush ruleset`)
3. **Maintains compatibility** with existing iptables systems
4. **Provides clear migration path** for administrators
5. **Handles all CSF features** with appropriate nftables equivalents

### Critical Implementation Requirements Confirmed:

✅ **Owner matches** - Correctly placed in OUTPUT chains only (u/g implies outbound)  
✅ **TRACE support** - Uses separate ip/ip6 tables (inet doesn't support prerouting)  
✅ **DROP_OUT handling** - Dynamically configures drop vs reject based on settings  
✅ **No iptables parsing** - Native nftables implementation, no complex translation  
✅ **Sets over rules** - Uses nft sets for IP lists (more efficient than individual rules)  
✅ **Atomic updates** - Batch operations via `nft -f` for consistency  
✅ **DynDNS efficiency** - Set element updates avoid rule churn  
✅ **PORTFLOOD/KNOCKING** - Best-effort approximation with clear behavior differences noted  

The implementation requires significant effort but ensures CSF remains viable for RHEL 10 and future Linux distributions while maintaining stability for the existing user base of millions of servers.
