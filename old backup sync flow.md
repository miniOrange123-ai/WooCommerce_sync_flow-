title WooCommerce Product Sync - Simple Backup Sync Flow

== Backup Setup ==

Sync Manager->Cron Manager: Setup backup monitoring
Cron Manager->WordPress: Schedule backup cron
note right of WordPress: Runs every few minutes\nMonitors sync progress

== Backup Monitoring ==

WordPress->Cron Manager: Trigger backup cron
Cron Manager->Sync Manager: Check sync status
Sync Manager->Sync Handler: check_sync_process()

Sync Handler->DB: Get batch analytics
DB->Sync Handler: Return analytics data

alt Sync is stuck or needs attention
    Sync Handler->Sync Handler: Detect issue
    note right of Sync Handler: Sync not progressing\nNeed to resume or restart
    
    Sync Handler->Connection Manager: Resume sync
    Connection Manager->Cron Manager: Schedule next batch
    Cron Manager->WordPress: Schedule batch cron
    
else Sync is progressing normally
    Sync Handler->Sync Handler: Continue monitoring
    note right of Sync Handler: Sync is working fine\nNo action needed
end

Sync Handler->Sync Manager: Return status
Sync Manager->Sync Manager: Update analytics

alt Sync completed
    Sync Manager->WordPress: Clear backup cron
    note right of WordPress: Stop monitoring\nSync finished
else Sync still in progress
    WordPress->WordPress: Continue backup cron
    note right of WordPress: Keep monitoring\nNext check in few minutes
end

== Available Hooks ==

note over User, WC
**Backup Sync Hooks:**

1. mo_wcps_product_sync_backup_cron
   - Backup monitoring cron hook
   - Parameters: cron_arguments array

2. mo_wcps_sync_backup_check
   - Called during backup check
   - Parameters: sync_manager object
end note 