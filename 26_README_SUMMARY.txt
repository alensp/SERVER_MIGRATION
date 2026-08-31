VPI-SERVER ADCS / CA READ-ONLY AUDIT
====================================
Computer: VPI-SERVER
Date:     08/31/2026 17:48:33

This audit DOES NOT:
- stop or restart CertSvc
- modify CA configuration
- revoke certificates
- approve/deny requests
- modify certificate templates
- modify CRL/AIA
- uninstall ADCS/IIS
- export a private key
- change GPOs
- restart the server

The purpose is to decide whether the existing CA must be migrated or can be formally decommissioned.

