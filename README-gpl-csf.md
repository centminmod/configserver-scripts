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

**Migration Steps:**

```bash
# Option 1: Configure custom download mirrors
echo "your.mirror.server" > /etc/csf/downloadservers

# Option 2: Use manual updates from GitHub
cd /tmp
wget https://github.com/waytotheweb/scripts/raw/refs/heads/main/csf.tgz
# Extract and run install.sh

# Option 3: Use your distribution's package manager (when available)
```

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
   cp /usr/local/csf/version.txt /root/csf_version_backup.txt
   ```

2. **Update CSF**

   ```bash
   # Download from GitHub
   cd /tmp
   wget https://github.com/waytotheweb/scripts/raw/refs/heads/main/csf.tgz
   tar -xzf csf.tgz
   cd csf
   sh install.sh
   ```

3. **Verify Installation**

   ```bash
   csf -v  # Should show v15.00
   csf -r  # Restart CSF
   ```

4. **Configure Update Source** (if desired)
   - Set up your own mirror in `/etc/csf/downloadservers`
   - OR rely on manual updates from GitHub

### For New Users

Simply clone and install:

```bash
git clone https://github.com/waytotheweb/scripts.git
cd scripts/csf
sh install.sh
```

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
