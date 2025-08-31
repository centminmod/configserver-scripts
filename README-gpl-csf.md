# CSF Firewall v15.00 - GPLv3 Release

## 🔄 Major Changes Overview

### License Transition

**CSF is now Free and Open Source Software under GPLv3**

- **Previous**: Proprietary "Way to the Web Product License" (restrictive commercial)
- **Current**: GNU General Public License v3 (copyleft open-source)
- **Version**: 14.24 → **15.00**
- **Copyright**: (C) 2006-2025 Jonathan Michaelson
- **Repository**: <https://github.com/waytotheweb/scripts>

### Change Statistics

- **Files Modified**: 121
- **Lines Added**: ~2,448
- **Lines Removed**: ~766
- **Files Deleted**: 3
- **Release Date**: August 29, 2025

---

## ⚠️ Breaking Changes - Action Required

### 1. Auto-Update System Disabled

The built-in update mechanism that connected to ConfigServer's servers has been disabled:

**What's affected:**

- `csget.pl` - Download server array emptied
- `downloadservers` - All entries commented out
- `ConfigServer/Config.pm` - Fallback server removed

### 2. Version Check Functionality

- Automatic version checking against central servers no longer functions
- Manual version checking against GitHub releases recommended

---

## 📝 Detailed Changes

### Removed Components

| File | Purpose | Impact |
|------|---------|--------|
| `.gitattributes` | Git line ending configuration | None - cosmetic |
| `ConfigServer/Security.pm` | Stub GPG/SHA256 verification (never implemented) | None - was non-functional |
| `regex.txt` | 143 lines of log parsing examples | None - test data only |

### Modified Components

#### Core Infrastructure

- **Download System**: All references to `download.configserver.com` and `download2.configserver.com` removed or commented
- **Version Checking**: `csget.pl` modified to prevent automatic version checks
- **Attribution**: UI footers updated from "Way to the Web Limited" to "Jonathan Michaelson"

```diff
diff --git a/csf/downloadservers b/csf/downloadservers
index ef8376d..e7b430f 100644
--- a/csf/downloadservers
+++ b/csf/downloadservers
@@ -1,2 +1,2 @@
-download.configserver.com
-download2.configserver.com
+#download.configserver.com
+#download2.configserver.com
```

```diff
diff --git a/csf/csget.pl b/csf/csget.pl
index a3e8876..016ecb6 100755
--- a/csf/csget.pl
+++ b/csf/csget.pl
@@ -1,8 +1,21 @@
 #!/usr/bin/perl
 ###############################################################################
-# Copyright 2006-2023, Way to the Web Limited
-# URL: http://www.configserver.com
-# Email: sales@waytotheweb.com
+# Copyright (C) 2006-2025 Jonathan Michaelson
+#
+# https://github.com/waytotheweb/scripts
+#
+# This program is free software; you can redistribute it and/or modify it under
+# the terms of the GNU General Public License as published by the Free Software
+# Foundation; either version 3 of the License, or (at your option) any later
+# version.
+#
+# This program is distributed in the hope that it will be useful, but WITHOUT
+# ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
+# FOR A PARTICULAR PURPOSE. See the GNU General Public License for more
+# details.
+#
+# You should have received a copy of the GNU General Public License along with
+# this program; if not, see <https://www.gnu.org/licenses>.
 ###############################################################################
 use strict;
 use warnings;
@@ -25,7 +38,7 @@
 
 $0 = "ConfigServer Version Check";
 
-my @downloadservers = ("https://download.configserver.com", "https://download2.configserver.com");
+my @downloadservers = ""; # ("https://download.configserver.com", "https://download2.configserver.com");
 
 system("mkdir -p /var/lib/configserver/");
 system("rm -f /var/lib/configserver/*.txt /var/lib/configserver/*error");
```

```diff
diff --git a/csf/ConfigServer/Config.pm b/csf/ConfigServer/Config.pm
index f854b29..f7a4648 100644
--- a/csf/ConfigServer/Config.pm
+++ b/csf/ConfigServer/Config.pm
@@ -1,7 +1,20 @@
 ###############################################################################
-# Copyright 2006-2023, Way to the Web Limited
-# URL: http://www.configserver.com
-# Email: sales@waytotheweb.com
+# Copyright (C) 2006-2025 Jonathan Michaelson
+#
+# https://github.com/waytotheweb/scripts
+#
+# This program is free software; you can redistribute it and/or modify it under
+# the terms of the GNU General Public License as published by the Free Software
+# Foundation; either version 3 of the License, or (at your option) any later
+# version.
+#
+# This program is distributed in the hope that it will be useful, but WITHOUT
+# ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
+# FOR A PARTICULAR PURPOSE. See the GNU General Public License for more
+# details.
+#
+# You should have received a copy of the GNU General Public License along with
+# this program; if not, see <https://www.gnu.org/licenses>.
 ###############################################################################
 ## no critic (RequireUseWarnings, ProhibitExplicitReturnUndef, ProhibitMixedBooleanOperators, RequireBriefOpen)
 # start main
@@ -429,21 +442,13 @@ sub getdownloadserver {
  my $downloadservers = "/etc/csf/downloadservers";
  my $chosen;
  if (-e $downloadservers) {
-##  open (my $DOWNLOAD, "<", $downloadservers);
-##  flock ($DOWNLOAD, LOCK_SH);
-##  my @data = <$DOWNLOAD>;
-##  close ($DOWNLOAD);
-##  chomp @data;
-##  foreach my $line (@data) {
-##   if ($line =~ /^download/) {push @servers, $line}
-##  }
   foreach my $line (slurp($downloadservers)) {
    $line =~ s/$cleanreg//g;
    if ($line =~ /^download/) {push @servers, $line}
   }
   $chosen = $servers[rand @servers];
  }
- if ($chosen eq "") {$chosen = "download.configserver.com"}
+## if ($chosen eq "") {$chosen = "download.configserver.com"}
  return $chosen;
 }
 ## end getdownloadserver
```

#### License Headers (All Files)

Every source file updated with GPLv3 boilerplate:

- Perl modules (`.pm`)
- Perl scripts (`.pl`)
- Shell scripts (`.sh`)
- CGI scripts (`.cgi`)
- PHP files (`.php`)
- Configuration files
- JavaScript files

### Unchanged Functionality

✅ All core security features remain intact:

- Firewall rules management
- Login failure detection (LFD)
- Port scan detection
- DDoS protection
- UI functionality
- Control panel integrations (cPanel, DirectAdmin, etc.)
- Email alerts
- Country blocking
- Rate limiting

---

## 🚀 Migration Guide

### For Existing Users (v14.x → v15.00)

1. **Backup Current Configuration**

```bash
cp -R /etc/csf /etc/csf.backup
cp /usr/local/csf/version.txt /usr/local/csf/version-backup.txt
```

2. **Update CSF**

```bash
# Use manual updates from GitHub
cd /usr/src
wget https://github.com/waytotheweb/scripts/raw/refs/heads/main/csf.tgz
# Extract and run install.sh
tar -xzf csf.tgz
cd csf
sh install.sh
```

3. Disable Auto Updates

In `/etc/csf/csf.conf` disable auto updates variable setting it to `0`. CSF v15.00 disabled auto update and version check in code, but `AUTO_UPDATES` if enabled, will still try to auto update - leading to download error which is just cosmetic and won't impact CSF Firewall operations.

```bash
# Enabling auto updates creates a cron job called /etc/cron.d/csf_update which
# runs once per day to see if there is an update to csf+lfd and upgrades if
# available and restarts csf and lfd
AUTO_UPDATES = "0"
```

4. **Verify Installation**

   ```bash
   csf -v  # Should show v15.00
   csf -r  # Restart CSF
   ```

5. **Configure Update Source** (if desired)
   - Set up your own mirror in `/etc/csf/downloadservers`. Will need to restore CSF v15.00 disabled auto update routines and version checks first.
   - OR rely on manual updates from GitHub

---

## 📋 Post-Upgrade Checklist

- [ ] Verify CSF version shows 15.00: `csf -v`
- [ ] Check firewall rules are intact: `csf -l`
- [ ] Confirm LFD is running: `systemctl status lfd`
- [ ] Test temporary IP blocking: `csf -d 1.2.3.4 "test"`
- [ ] Remove test block: `csf -dr 1.2.3.4`
- [ ] Review UI access (if applicable): Access CSF through your control panel
- [ ] Set up alternative update method (GitHub watch, custom mirror, etc.)
- [ ] Review `/var/log/lfd.log` for any errors

---

## 📄 License Implications

### What GPLv3 Means for You

**You CAN:**

- ✅ Use CSF commercially
- ✅ Modify the source code
- ✅ Distribute modified versions
- ✅ Use CSF in private networks

**You MUST:**

- 📋 Include copyright notice
- 📋 Include license text
- 📋 State changes made
- 📋 Disclose source code (if distributing)
- 📋 Use same GPLv3 license (for derivatives)

**You CANNOT:**

- ❌ Sublicense under different terms
- ❌ Hold liable for damages
- ❌ Use contributors' names for endorsement

---

## 🔧 Technical Details

### File Header Example

```perl
###############################################################################
# Copyright (C) 2006-2025 Jonathan Michaelson
#
# https://github.com/waytotheweb/scripts
#
# This program is free software; you can redistribute it and/or modify it under
# the terms of the GNU General Public License as published by the Free Software
# Foundation; either version 3 of the License, or (at your option) any later
# version.
###############################################################################
```

### Affected File Types

- **115+ source files** updated with GPLv3 headers
- **Configuration files** maintained with new copyright
- **Documentation** updated with new attribution
- **UI components** modified to reflect new ownership

---

## 📞 Support & Community

- **GitHub Issues**: Report bugs and request features
- **Documentation**: Available in `/usr/local/csf/readme.txt`

---

## ⚡ Quick Reference

| Component | Status | Notes |
|-----------|---------|-------|
| Core Firewall | ✅ Unchanged | Full functionality retained |
| LFD | ✅ Unchanged | All detection methods working |
| Auto-Updates | ❌ Disabled | Manual updates required |
| Version Check | ❌ Disabled | Check GitHub for releases |
| UI | ✅ Working | Attribution text updated |
| Panel Integration | ✅ Working | All panels supported |
| Configuration | ✅ Compatible | No changes needed |
