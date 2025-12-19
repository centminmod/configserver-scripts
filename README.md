# Way to the Web Limited Scripts

Update: December 19, 2025

- The official repo at <https://github.com/waytotheweb/scripts> no longer exists. You can use this forked repo instead  <https://github.com/centminmod/configserver-scripts> which is forked from <https://github.com/waytotheweb/scripts>. GitHub incorrectly says this repo is forked from `mappy9211` repo version - it is **NOT** forked from `mappy9211` repo version. You can inspect this forked repo's commit history at <https://github.com/centminmod/configserver-scripts/commits/main/>.
- The official CSF Firewall GPLv3 open source release version is v15.00 and details, changes and migration/download guide are outlined in [README-gpl-csf.md](README-gpl-csf.md). Jump to migration guide [here](README-gpl-csf.md#-migration-guide).
- The official CSF Firewall v15.00 GPLv3 open source release disables the download and version check routines in code, so there is no more auto-updating functionality. You will be stuck on v15.00 until you manually update to newer versions or forks in the future. Some control panels and providers are self-hosting their own CSF Firewall forks which have restored download and version check routines in code. Control panels such as [DirectAdmin](https://forum.directadmin.com/threads/configserver-shutting-down-as-of-31st-of-august-2025.76678/) and [Centmin Mod](https://blog.centminmod.com/2025/09/04/2921/configserver-csf-firewall-shutdown-on-august-31-2025/) have their own forks of CSF Firewall with their own download and version check routines restored in code. cPanel has [announced](https://features.cpanel.net/c/202-firewall-configuration-tool) they will have their own fork of CSF Firewall sometime in early 2026.
- One leading CSF Firewall fork is contender is Aetherinox CSF Firewall Fork <https://github.com/Aetherinox/csf-firewall> with many changes beyond CSF Firewall v15.00 GPLv3 open source release. I am not using this fork but my own Centmin Mod CSF Firewall mirror fork for now. But I am personally tracking Aetherinox CSF Firewall fork's changes in dedicated GitHub repo at <https://github.com/centminmod/csf-firewall-aetherinox-tracking> to fully understand the changes. Direct link to fork's changes file at <https://github.com/centminmod/csf-firewall-aetherinox-tracking/blob/master/aetherinox-fork-changes.md>.

---

This repo is a fork of the official repo at https://github.com/waytotheweb/scripts. To see differences between this fork and official repo, please check out the link [here](https://github.com/waytotheweb/scripts/compare/main...centminmod:configserver-scripts:main).

## All files within this repository are subject to the [GPL license](LICENSE.txt) as outlined in [COPYING.md](COPYING.md)

These are the final copies of the scripts.

There is no intention to update any of these files, so any PR's or other contact will not receive a response. 

For uninstallation of any of our scripts, go to the [uninstallers](uninstallers) directory.

## ConfigServer Closure Announcement

Note, after August 31, 2025 the site http://configserver.com and download site <https://download.configserver.com>will no longer work. Below notice was taken from https://configserver.com/announcement/. Free scripts open sourced under GPLv3 licensing are now at https://github.com/waytotheweb/scripts and in this forked repo https://github.com/centminmod/configserver-scripts.

For CSF Firewall v15.00 GPLv3 updates read https://github.com/centminmod/configserver-scripts/blob/main/README-gpl-csf.md

### Important Notice

Way to the Web Ltd and [Configserver.com](http://configserver.com) will be closing down permanently on **31 August 2025**. The server software market has changed drastically in the more than 25 years since our company began, and we now find the business is no longer profitable so must come to an end.

This closure affects all of our commercial software including:

- ConfigServer Exploit Scanner (cxs)
- MailScanner Front-End (MSFE)
- Outgoing Spam Monitor (osm)

It also affects our free software including:

- ConfigServer Security and Firewall (csf)
- ConfigServer Mail Queues (cmq)
- ConfigServer Mail Manage (cmm)
- ConfigServer Modsecurity Control (cmc)
- ConfigServer Explorer (cse)

After 31st August, there will be no further support, downloads, or license IP changes available.

**⚠️ CRITICAL: In order to continue using any of our commercial software after the 31st of August, you must update the software to the latest version. If not updated, any of our commercial software products will cease to function and cannot be reactivated once the download and license servers are shut down.**

---

### Frequently Asked Questions

#### Can I keep using the software I have installed on my server after 31 August?

Yes, you can keep using the software as long as you update it to the latest version before the 31st of August. After that date, if you have not upgraded to the latest version, the software will stop working. This applies to ConfigServer eXploit Scanner, Outgoing Spam Monitor, and MailScanner Front-End.

#### Can I move the software to another server after 31 August?

No, because changing the license IP will not be possible after 31 August, and you will not be able to download the software to install it on another server.

#### Can I reinstall the software after 31 August?

No, you will no longer be able to download the software to install it.

#### Will you update the software if a bug is found?

No, no further updates will be provided.

#### How can I contact you after 31 August?

Since our company will have closed, there will be no email or helpdesk available after the 31st of August.

#### Will the knowledgebase be available after the 31st of August?

No, the entirety of the support website will be closed down.

#### I have licenses I am not currently using. Will I be able to install or activate them after the 31st of August?

No, after that date there will be no downloads available or any license manager system to activate or move licenses.

#### I just bought the software recently. Can I get a refund?

If the software has not yet been installed and you have purchased it less than 14 days before your request, we will provide a refund if requested. Otherwise as per our ordering and refund policy ([https://configserver.com/ordering-policy/](https://configserver.com/ordering-policy/)) no refunds can be made.

#### I have licenses I am not currently using. Can I get a refund for them?

If the software has not yet been installed and you have purchased it less than 14 days before your request, we will provide a refund if requested. Otherwise as per our ordering and refund policy ([https://configserver.com/ordering-policy/](https://configserver.com/ordering-policy/)) no refunds can be made.

#### Will the Configserver Community Forum still be available?

No, the forum will be closed immediately.

#### Will you continue to update the cxs fingerprint database?

No, there will be no further updates to the cxs fingerprints after 31st August.

#### Will the cxs reputation system continue to work?

No, the cxs reputation system will no longer work because the server running it will shut down.

#### Will your free scripts continue to work after the 31st of August?

Yes, ConfigServer Mail Manage (csm), CS Mail Queues (cmq), CS Modsecurity Control (cmc), CS Explorer (cse) and ConfigServer Security & Firewall (csf) will continue to work. They will no longer be updated, however, and will not be available for download or install.

#### How can I uninstall the software? (Any of the commercial or free software)

Please see our scripts github repository for uninstall scripts for both the commercial and free scripts.

#### Can I upgrade the software using shell commands without going into the UI for each product?

**For Configserver Exploit Scanner (cxs):**

```bash
cxs -U
```

**For Outgoing Spam Monitor (osm):**

```bash
wget --no-check-certificate https://download.configserver.com/osminstaller.tgz
tar -xzf osminstaller.tgz
perl osminstaller.pl
```

**For MailScanner Front-End (msfe):**

```bash
wget --no-check-certificate https://download.configserver.com/msfeinstaller.tgz
tar -xzf msfeinstaller.tgz
perl msfeinstaller.pl
/usr/mscpanel/mscheck.pl
```

**Configserver Explorer:**

```bash
wget --no-check-certificate https://download.configserver.com/cse.tgz
tar -xzf cse.tgz
cd cse
sh install.sh
```

**Configserver Modsecurity Control:**

```bash
wget --no-check-certificate https://download.configserver.com/cmc.tgz
tar -xzf cmc.tgz
cd cmc
sh install.sh
```

**Configserver Mail Manage:**

```bash
wget --no-check-certificate https://download.configserver.com/cmm.tgz
tar -xzf cmm.tgz
cd cmm
sh install.sh
```

**Configserver Mail Queues:**

```bash
wget --no-check-certificate https://download.configserver.com/cmq.tgz
tar -xzf cmq.tgz
cd cmq
sh install.sh
```

**For Configserver Security & Firewall (csf):**

```bash
csf -u
```

**All script updater:**
If you have multiple configserver commercial scripts installed, there is a single command you can run to update them all:

```bash
curl -s https://download.configserver.com/csupdate | perl
```

#### I use the bulk license management system. Will I be able to add and remove licenses after the 31st of August?

No, the bulk license system will close down at the same time as the general license server.

#### My company has a large number of servers with your products installed. Can we have more time to upgrade all the servers?

We hope that 30 days will be enough time to update all servers. If you are unable to get it done before 31st August please contact us and we may be able to assist.
