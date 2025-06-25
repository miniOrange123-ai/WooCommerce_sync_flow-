title WooCommerce Supplier Sync - Simple Backup Sync Flow

== Supplier Backup Setup ==

Supplier Manager->Cron Manager: Setup supplier backup monitoring
Cron Manager->WordPress: Schedule supplier backup cron
note right of WordPress: Runs every few minutes\nMonitors supplier sync progress

== Supplier Backup Monitoring ==

WordPress->Cron Manager: Trigger supplier backup cron
Cron Manager->Supplier Manager: Check supplier sync status
Supplier Manager->Connection Manager: check_supplier_progress()

Connection Manager->DB: Get all connection analytics
DB->Connection Manager: Return connection statuses
note right of Connection Manager: Check each connection:\nConnection1, Connection2, Connection3

alt Current connection is stuck
    Connection Manager->Connection Manager: Detect stuck connection
    note right of Connection Manager: Connection not progressing\nNeed to resume or restart
    
    Connection Manager->Sync Manager: Resume connection sync
    Sync Manager->Cron Manager: Schedule connection cron
    Cron Manager->WordPress: Schedule connection batch cron
    
else Current connection completed
    Connection Manager->Connection Manager: Check next priority connection
    note right of Connection Manager: Move to next connection\nConnection1 → Connection2 → Connection3
    
    alt More connections available
        Connection Manager->Supplier Manager: Start next connection
        Supplier Manager->Connection Manager: start_connection_sync(next_connection)
        Connection Manager->Cron Manager: Schedule next connection cron
        
    else All connections completed
        Connection Manager->Supplier Manager: All connections done
        Supplier Manager->Supplier Manager: Mark supplier sync completed
    end
    
else Connection progressing normally
    Connection Manager->Connection Manager: Continue monitoring
    note right of Connection Manager: Connection working fine\nNo action needed
end

Connection Manager->Supplier Manager: Return supplier status
Supplier Manager->Supplier Manager: Update supplier analytics

alt Supplier sync completed
    Supplier Manager->WordPress: Clear supplier backup cron
    note right of WordPress: Stop monitoring\nSupplier sync finished
else Supplier sync still in progress
    WordPress->WordPress: Continue supplier backup cron
    note right of WordPress: Keep monitoring\nNext check in few minutes
end

== Connection-Level Backup ==

WordPress->Cron Manager: Trigger connection backup cron
Cron Manager->Sync Manager: Check connection sync status
Sync Manager->Sync Handler: check_sync_process(connection)

Sync Handler->DB: Get batch analytics for connection
DB->Sync Handler: Return connection batch data

alt Connection batch stuck
    Sync Handler->Sync Handler: Resume connection batch
    Sync Handler->Connection Manager: Schedule next batch
    Connection Manager->Cron Manager: Schedule batch cron
    
else Connection batch completed
    Sync Handler->Connection Manager: Batch completed
    Connection Manager->Supplier Manager: Update connection progress
end

== Available Hooks ==

note over User, WC
**Supplier Backup Sync Hooks:**

1. mo_wcps_supplier_sync_backup_cron
   - Supplier backup monitoring cron hook
   - Parameters: supplier_arguments array

2. mo_wcps_connection_sync_backup_cron
   - Connection backup monitoring cron hook
   - Parameters: connection_arguments array

3. mo_wcps_connection_completed
   - Called when connection sync completes
   - Parameters: connection_name, supplier_name

4. mo_wcps_supplier_sync_completed
   - Called when all connections complete
   - Parameters: supplier_name
end note 