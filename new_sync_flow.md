title WooCommerce Supplier Sync Flow - Priority Based Connection Sync

User->UI: Click "Start Supplier Sync"
UI->UI: handle_supplier_sync_action()
note right of UI: Check if supplier sync running\nUpdate supplier status to 'started'\nGet all connections for supplier

UI->Cron: enqueue_supplier_sync(supplier_name)
note right of Cron: Create Supplier_Analytics\nGet connections by priority\nSchedule supplier sync cron\nSet status 'Supplier Sync in queue'

Cron->WordPress: Schedule supplier cron
WordPress->Cron: Trigger mo_wcps_supplier_sync_cron
Cron->Supplier Manager: start_supplier_sync()
note right of Supplier Manager: Initialize supplier\nGet priority connections\nStart with highest priority\nHook: mo_wcps_perform_action_before_supplier_sync

Supplier Manager->Connection Manager: get_connections_by_priority(supplier_name)
Connection Manager->Supplier Manager: priority_connections[]
note right of Supplier Manager: Sort connections by priority\n[Connection1: Priority 1]\n[Connection2: Priority 2]\n[Connection3: Priority 3]

Supplier Manager->Connection Manager: start_connection_sync(connection1)
note right of Connection Manager: Start sync for highest priority connection\nSet connection status 'in_progress'

Connection Manager->Cron: enqueue_connection_sync(connection1)
Cron->WordPress: Schedule connection cron
WordPress->Cron: Trigger mo_wcps_connection_sync_cron
Cron->Sync Manager: start_connection_sync(connection1)
note right of Sync Manager: Initialize connection\nSetup backup cron\nHook: mo_wcps_perform_action_before_connection_sync

Sync Manager->Sync Handler: start_sync_process(connection1)
Sync Handler->Sync Handler: handle_api_sync(connection1)
note right of Sync Handler: Set memory limits\nMake API call for connection1\nProcess products for connection1

Sync Handler->API: Get products for connection1
API->Sync Handler: Return products for connection1
Sync Handler->Products: sync_products(connection1_products)
note right of Products: Loop products for connection1\nHook: mo_wcps_data_alteration_filter

Products->Product: set_fields(connection1_product)
Product->Product: save()
Product->WC: Create/Update product
WC->Product: Return product ID
Product->Products: Return result
Products->Sync Handler: Return connection1 status
Sync Handler->Sync Manager: Return connection1 status

Sync Manager->Sync Manager: end_connection_sync(connection1)
note right of Sync Manager: Update connection1 analytics\nSet connection1 end time\nMark connection1 as completed

Sync Manager->Connection Manager: connection1_completed()
Connection Manager->Connection Manager: check_next_connection()
note right of Connection Manager: Check if more connections\nGet next priority connection

Connection Manager->Supplier Manager: connection_completed(connection1)
Supplier Manager->Supplier Manager: check_supplier_progress()
note right of Supplier Manager: Update supplier progress\nCheck if all connections done

alt More connections available
    Supplier Manager->Connection Manager: start_connection_sync(connection2)
    note right of Connection Manager: Start sync for next priority connection
    
    Connection Manager->Cron: enqueue_connection_sync(connection2)
    Cron->WordPress: Schedule connection cron
    WordPress->Cron: Trigger mo_wcps_connection_sync_cron
    Cron->Sync Manager: start_connection_sync(connection2)
    
    Sync Manager->Sync Handler: start_sync_process(connection2)
    Sync Handler->Sync Handler: handle_api_sync(connection2)
    note right of Sync Handler: Process connection2 products
    
    Sync Handler->API: Get products for connection2
    API->Sync Handler: Return products for connection2
    Sync Handler->Products: sync_products(connection2_products)
    
    Products->Product: set_fields(connection2_product)
    Product->Product: save()
    Product->WC: Create/Update product
    WC->Product: Return product ID
    Product->Products: Return result
    Products->Sync Handler: Return connection2 status
    Sync Handler->Sync Manager: Return connection2 status
    
    Sync Manager->Sync Manager: end_connection_sync(connection2)
    Sync Manager->Connection Manager: connection2_completed()
    Connection Manager->Supplier Manager: connection_completed(connection2)
    
    Supplier Manager->Supplier Manager: check_supplier_progress()
    note right of Supplier Manager: Check if all connections done
    
    alt More connections available
        Supplier Manager->Connection Manager: start_connection_sync(connection3)
        note right of Connection Manager: Continue with remaining connections
        Connection Manager->Supplier Manager: ... (repeat for all connections)
    else All connections completed
        Supplier Manager->Supplier Manager: end_supplier_sync()
        note right of Supplier Manager: All connections completed\nUpdate supplier analytics\nMark supplier sync as completed
    end
else All connections completed
    Supplier Manager->Supplier Manager: end_supplier_sync()
    note right of Supplier Manager: All connections completed\nUpdate supplier analytics\nMark supplier sync as completed
end

Supplier Manager->User: Supplier sync completed

== Backup Cron Flow for Connections ==

WordPress->Cron: Trigger connection backup cron
Cron->Connection Manager: initiate_connection_backup()
Connection Manager->Sync Manager: sync_backup_check(connection)
note right of Sync Manager: Check if connection sync stuck\nResume or complete connection sync

== Available Hooks ==

note over User, WC
**Available WordPress Hooks:**

1. mo_wcps_perform_action_before_supplier_sync
   - Called in start_supplier_sync()
   - Parameters: supplier_manager object

2. mo_wcps_perform_action_before_connection_sync
   - Called in start_connection_sync()
   - Parameters: connection_manager object

3. mo_wcps_connection_completed
   - Called when connection sync completes
   - Parameters: connection_name, supplier_name

4. mo_wcps_supplier_sync_completed
   - Called when all connections complete
   - Parameters: supplier_name

5. mo_wcps_data_alteration_filter
   - Called during product processing
   - Events: 'simple_products_data_array_initial', 'alter_products_category'
   - Parameters: array with event_specifier and data

6. mo_wcps_supplier_sync_cron
   - Supplier sync cron hook
   - Parameters: supplier_arguments array

7. mo_wcps_connection_sync_cron
   - Connection sync cron hook
   - Parameters: connection_arguments array

8. mo_wcps_connection_sync_backup_cron
   - Connection backup/monitoring cron hook
   - Parameters: connection_arguments array
end note 