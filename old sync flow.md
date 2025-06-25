title WooCommerce Product Sync - Function Call Flow

User->UI Handler: Click "Start Sync" button
UI Handler->UI Handler: handle_action('mowcpsstartsync')
note right of UI Handler: Check if sync running\nUpdate status to 'started'\nCall enqueue_product_sync()

UI Handler->Cron Manager: enqueue_product_sync(connection_name)
note right of Cron Manager: Create Sync_Analytics\nSetup analytics\nSchedule cron job\nSet status 'Sync in queue'

Cron Manager->WordPress: wp_schedule_single_event()
WordPress->WordPress: Schedule cron job (2 seconds later)

WordPress->Cron Manager: Trigger mo_wcps_product_sync_cron
Cron Manager->Cron Manager: initiate_sync(cron_arguments)
note right of Cron Manager: Check license status\nCall start_sync()

Cron Manager->Sync Manager: start_sync()
note right of Sync Manager: Initialize Default_Apps_Factory\nSetup backup cron\nHook: mo_wcps_perform_action_before_sync\nCall start_sync_process()

Sync Manager->Cron Manager: enqueue_product_sync_backup(sync_manager)
note right of Cron Manager: Calculate backup interval\nSchedule backup monitoring

Sync Manager->Sync Handler: start_sync_process()
note right of Sync Handler: Check sync status\nInitialize analytics\nHandle product discarding\nCall handle_api_sync()

Sync Handler->Sync Handler: handle_api_sync()
note right of Sync Handler: Set memory limits\nCall set_product_data()\nCall filter_product_data()\nCall sync_products()

Sync Handler->Sync Handler: set_product_data()
note right of Sync Handler: Make API call to external supplier\nConvert response to array\nMap response attributes

Sync Handler->External API: retrieve_api_data(headers, body, url)
External API->Sync Handler: API response

Sync Handler->Sync Utils: convert_response_to_array()
Sync Utils->Sync Handler: products_array

Sync Handler->Sync Handler: filter_product_data()
note right of Sync Handler: Apply product filters\nRemove unwanted products

Sync Handler->Sync Handler: sync_products(product_data)
note right of Sync Handler: Determine sync structure\nCreate sync class\nCall sync_products() on class

Sync Handler->Sync Products: sync_products()
note right of Sync Products: Loop through products\nHook: mo_wcps_data_alteration_filter\nCreate product objects

Sync Products->MO Product: set_fields(product_data)
note right of MO Product: Extract API data\nMap to WooCommerce fields\nHook: mo_wcps_data_alteration_filter

MO Product->MO Product: save()
note right of MO Product: Check if product exists\nIf exists: update_product()\nIf new: add_product()

MO Product->Sync Utils: update_product() or add_product()
note right of Sync Utils: Create/update WooCommerce product\nSet all fields\nHandle categories, images\nReturn product ID

Sync Utils->MO Product: product_details
MO Product->Sync Products: product_obj

Sync Products->Sync Analytics: Update counters
note right of Sync Analytics: product_saved_success++\ntotal_product_parsed++\nLog success/failure

Sync Products->Sync Handler: Return sync status
Sync Handler->Sync Manager: Return sync status

Sync Manager->Sync Manager: end_sync(sync_end_status)
note right of Sync Manager: Update final analytics\nSet end time\nUpdate sync status\nClear scheduled crons

Sync Manager->User: Sync completed

== Backup Cron Flow ==

WordPress->Cron Manager: Trigger backup cron
Cron Manager->Cron Manager: initiate_sync_backup(cron_arguments)
note right of Cron Manager: Check license\nCall sync_backup_check()

Cron Manager->Sync Manager: sync_backup_check(cron_arguments)
Sync Manager->Sync Handler: check_sync_process(cron_arguments)
note right of Sync Handler: Check batch analytics\nDetermine next action\nSchedule next batch if needed

Sync Handler->Sync Manager: Return sync status
Sync Manager->Cron Manager: Return sync status