---
title: ovirt-engine升级internal CA证书
description: 
published: true
date: 2023-03-31T01:34:00.141Z
tags: ovirt
editor: markdown
dateCreated: 2023-03-29T01:37:42.298Z
---

报错信息：
```
PKIX path validation failed: java.security.cert.CertPathValidatorException: validity check failed
```

ovirt-engine的证书时间只有一年，一年过后，就会提示证书过期而无法进入web管理界面，这个时候只能给engine更新证书，我们可以使用`engine-setup --offline`去执行更新

```shell
[root@localhost ~]# engine-setup --offline
[ INFO  ] Stage: Initializing
[ INFO  ] Stage: Environment setup
          Configuration files: /etc/ovirt-engine-setup.conf.d/10-packaging-jboss.conf, /etc/ovirt-engine-setup.conf.d/10-packaging.conf, /etc/ovirt-engine-setup.conf.d/20-setup-ovirt-post.conf
          Log file: /var/log/ovirt-engine/setup/ovirt-engine-setup-20230329010911-qawyw6.log
          Version: otopi-1.9.6 (otopi-1.9.6-1.el8)
[ INFO  ] Stage: Environment packages setup
[ INFO  ] Stage: Programs detection
[ INFO  ] Stage: Environment setup (late)
[ INFO  ] Stage: Environment customization

          --== PRODUCT OPTIONS ==--

[ INFO  ] ovirt-provider-ovn already installed, skipping.

          --== PACKAGES ==--


          --== NETWORK CONFIGURATION ==--

[WARNING] Failed to resolve ********* using DNS, it can be resolved only locally

          Setup can automatically configure the firewall on this system.
          Note: automatic configuration of the firewall may overwrite current settings.
          Do you want Setup to configure the firewall? (Yes, No) [Yes]:
[ INFO  ] firewalld will be configured as firewall manager.

          --== DATABASE CONFIGURATION ==--

          The detected DWH database size is 385.6336507797241 MB.
          Setup can backup the existing database. The time and space required for the database backup depend on its size. This process takes time, and in some cases (for instance, when the size is few GBs) may take several hours to complete.
          If you choose to not back up the database, and Setup later fails for some reason, it will not be able to restore the database and all DWH data will be lost.
          Would you like to backup the existing database before upgrading it? (Yes, No) [Yes]:
          Perform full vacuum on the oVirt engine history
          database ovirt_engine_history@localhost?
          This operation may take a while depending on this setup health and the
          configuration of the db vacuum process.
          See https://www.postgresql.org/docs/12/sql-vacuum.html
          (Yes, No) [No]:

          --== OVIRT ENGINE CONFIGURATION ==--

          Perform full vacuum on the engine database engine@localhost?
          This operation may take a while depending on this setup health and the
          configuration of the db vacuum process.
          See https://www.postgresql.org/docs/12/sql-vacuum.html
          (Yes, No) [No]:

          --== STORAGE CONFIGURATION ==--


          --== PKI CONFIGURATION ==--

          One or more of the certificates should be renewed, because they expire soon, or include an invalid expiry date, or they were created with validity period longer than 398 days, or do not include the subjectAltName extension, which can cause them to be rejected by recent browsers and up to date hosts.
          See https://www.ovirt.org/develop/release-management/features/infra/pki-renew/ for more details.
          Renew certificates? (Yes, No) [No]: Yes

          --== APACHE CONFIGURATION ==--


          --== SYSTEM CONFIGURATION ==--


          --== MISC CONFIGURATION ==--


          --== END OF CONFIGURATION ==--

[ INFO  ] Stage: Setup validation
[WARNING] Cannot validate host name settings, reason: resolved host does not match any of the local addresses
          During execution engine service will be stopped (OK, Cancel) [OK]:
[WARNING] Less than 16384MB of memory is available
[ INFO  ] Cleaning stale zombie tasks and commands

          --== CONFIGURATION PREVIEW ==--

          Default SAN wipe after delete           : False
          Host FQDN                               : ovirt-engine-02.******
          Firewall manager                        : firewalld
          Update Firewall                         : True
          Set up Cinderlib integration            : False
          Engine database host                    : localhost
          Engine database port                    : 5432
          Engine database secured connection      : False
          Engine database host name validation    : False
          Engine database name                    : engine
          Engine database user name               : engine
          Engine installation                     : True
          PKI organization                        : ********
          Renew PKI                               : True
          Set up ovirt-provider-ovn               : True
          Grafana integration                     : True
          Grafana database user name              : ovirt_engine_history_grafana
          Configure WebSocket Proxy               : True
          DWH installation                        : True
          DWH database host                       : localhost
          DWH database port                       : 5432
          DWH database secured connection         : False
          DWH database host name validation       : False
          DWH database name                       : ovirt_engine_history
          DWH database user name                  : ovirt_engine_history
          Backup DWH database                     : True
          Configure VMConsole Proxy               : True

          Please confirm installation settings (OK, Cancel) [OK]:
[ INFO  ] Cleaning async tasks and compensations
[ INFO  ] Unlocking existing entities
[ INFO  ] Checking the Engine database consistency
[ INFO  ] Stage: Transaction setup
[ INFO  ] Stopping engine service
[ INFO  ] Stopping ovirt-fence-kdump-listener service
[ INFO  ] Stopping dwh service
[ INFO  ] Stopping vmconsole-proxy service
[ INFO  ] Stopping websocket-proxy service
[ INFO  ] Stopping service: grafana-server
[ INFO  ] Stage: Misc configuration (early)
[ INFO  ] Stage: Package installation
[ INFO  ] Stage: Misc configuration
[ INFO  ] Upgrading CA
[ INFO  ] Renewing engine certificate
[ INFO  ] Renewing jboss certificate
[ INFO  ] Renewing websocket-proxy certificate
[ INFO  ] Renewing apache certificate
[ INFO  ] Renewing reports certificate
[ INFO  ] Updating OVN SSL configuration
[ INFO  ] Updating OVN timeout configuration
[ INFO  ] Backing up database localhost:ovirt_engine_history to '/var/lib/ovirt-engine-dwh/backups/dwh-20230329010951.xv0vna3s.dump'.
[ INFO  ] Creating/refreshing DWH database schem
......
```