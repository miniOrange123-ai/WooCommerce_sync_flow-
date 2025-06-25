title WooCommerce Product Sync Flow

User->UI: Click Start Sync
UI->UI: handle_action()
note right of UI: Check sync status\nUpdate to 'started'

UI->Cron: enqueue_product_sync()
note right of Cron: Create analytics\nSchedule cron job

Cron->WordPress: Schedule cron
WordPress->Cron: Trigger cron
Cron->Sync: start_sync()
note right of Sync: Setup backup\nHook: mo_wcps_perform_action_before_sync

Sync->Handler: start_sync_process()
Handler->Handler: handle_api_sync()
note right of Handler: Set memory limits\nMake API call

Handler->API: Get products
API->Handler: Return products
Handler->Products: sync_products()
note right of Products: Loop products\nHook: mo_wcps_data_alteration_filter

Products->Product: set_fields()
Product->Product: save()
note right of Product: Check if exists\nUpdate or create

Product->WC: Create/Update product
WC->Product: Return product ID
Product->Products: Return result
Products->Handler: Return status
Handler->Sync: Return status
Sync->User: Sync completed

note over User, WC
**Available Hooks:**
1. mo_wcps_perform_action_before_sync
2. mo_wcps_data_alteration_filter
3. mo_wcps_product_sync_cron
4. mo_wcps_product_batch_sync_cron
5. mo_wcps_product_sync_backup_cron
end note 