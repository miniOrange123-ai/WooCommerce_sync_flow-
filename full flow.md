@startuml WooCommerce Product Sync Process Flow

!theme plain
skinparam backgroundColor #FFFFFF
skinparam sequenceArrowThickness 2
skinparam roundcorner 20
skinparam maxmessagesize 60

title WooCommerce Product Sync - Complete Process Flow

actor User as "User"
participant "UI Handler" as UI
participant "Cron Manager" as CM
participant "Sync Manager" as SM
participant "Sync Handler\n(Independent/Interdependent)" as SH
participant "Sync Analytics" as SA
participant "Default Apps Factory" as DAF
participant "Sync Products\n(Simple/Variable)" as SP
participant "MO Product" as MP
participant "Sync Utils" as SU
participant "DB Utils" as DB
participant "Debug Logger" as DL
participant "WordPress Cron" as WC
participant "External API" as API

== User Initiates Sync ==

User -> UI: Click "Start Sync" button
activate UI

UI -> UI: mo_wcps_dropdown_action_redirect('connection_name', 'play')
UI -> UI: Redirect to admin.php?page=sync-to-woocommerce&tab=sync-settings&action=mowcpsstartsync&connection_name={key}

UI -> UI: handle_action() - mowcpsstartsync
UI -> DB: get_sync_running_status(connection_name)
DB --> UI: sync_status

alt sync_status === 'off'
    UI -> DB: update_sync_running_status(connection_name, 'on')
    UI -> DB: update_option('mo_wcps_{connection_name}_sync_running_status', 'started')
    UI -> CM: enqueue_product_sync(connection_name)
    UI -> DB: update_option('mo_wcps_message', 'Sync connection {connection_name} process started.')
    UI -> DB: update_option('mo_wcps_message_flag', 1)
    UI -> User: Redirect to sync-analytics page
else sync_status !== 'off'
    UI -> DB: update_option('mo_wcps_message', 'Sync connection {connection_name} is already in process.')
    UI -> DB: update_option('mo_wcps_message_flag', 2)
    UI -> User: Redirect to sync-analytics page
end

deactivate UI

== Cron Manager Setup ==

CM -> CM: enqueue_product_sync(connection_name)
activate CM

CM -> SA: new Sync_Analytics(connection_name)
activate SA
SA -> SA: setup_analytics()
SA -> SA: analytics_type = 'complete_connection'
SA -> SA: save()
deactivate SA

CM -> SM: new Sync_Manager(connection_name)
activate SM

CM -> WC: wp_schedule_single_event(time() + 2, 'mo_wcps_product_sync_cron', [cron_arguments])
CM -> SA: sync_status_message = 'Sync in queue'
CM -> SA: save()

deactivate CM

== Main Sync Process ==

WC -> CM: initiate_sync(cron_arguments)
activate CM

CM -> CM: Check license status
alt License expired
    CM -> CM: return (do nothing)
else License valid
    CM -> SM: start_sync()
end

deactivate CM

SM -> DAF: new Default_Apps_Factory(vendor_name)
activate DAF
DAF -> DAF: Initialize vendor-specific configurations
deactivate DAF

SM -> CM: enqueue_product_sync_backup(sync_manager)
activate CM
CM -> CM: Calculate backup cron interval
CM -> WC: wp_schedule_event(time(), 'mo_wcps_{interval}_sec', 'mo_wcps_product_sync_backup_cron', [cron_arguments])
deactivate CM

SM -> SM: do_action('mo_wcps_perform_action_before_sync', sync_manager)
note right: Hook: mo_wcps_perform_action_before_sync\nAvailable for custom actions before sync

SM -> SH: start_sync_process()
activate SH

SH -> DB: get_sync_running_status(connection_name)
DB --> SH: sync_status

alt sync_status !== 'off'
    SH -> DB: get_all_analytics(connection_name, true)
    DB --> SH: analytics_array
    
    SH -> SA: sync_status_message = 'Initializing sync'
    SH -> SA: sync_start_time = analytics_array[0]->sync_start_time || time()
    SH -> SA: initiate_sync = false
    SH -> SA: sync_end_time = 0
    SH -> SA: save()
    
    SH -> SH: Check discard_products_from setting
    alt discard_products_from !== 'no_products' && page_number === 1 && batch_number === 1
        SH -> SU: update_existing_product_status([filter_case, filter_connection])
        activate SU
        SU -> DB: get_website_products(args)
        DB --> SU: product_ids
        SU -> SU: update_product_meta(product_ids)
        deactivate SU
    end
    
    SH -> SH: handle_api_sync()
    
else sync_status === 'off'
    SH -> SH: return 'Sync Aborted'
end

deactivate SH

SM -> SM: end_sync(sync_end_status)
activate SM

SM -> SA: update_analytics()
SM -> SA: sync_status_message = sync_end_status

alt sync_end_status contains 'Sync Completed'
    SM -> SA: sync_end_time = time()
    SM -> SA: initiate_sync = true
    SM -> DB: update_sync_running_status(connection_name, 'off')
    SM -> SU: create_cron_data_array(connection_name, sync_end_time, 'SCHEDULED_SYNC_EXECUTED')
    SM -> SU: update_scheduled_sync(status_array, cron_data)
else sync_end_status contains 'Sync Aborted'
    SM -> SA: sync_end_time = time()
    SM -> SA: initiate_sync = false
    SM -> DB: update_sync_running_status(connection_name, 'off')
    SM -> SU: create_cron_data_array(connection_name, sync_end_time, 'SCHEDULED_SYNC_ABORTED')
    SM -> SU: update_scheduled_sync(status_array, cron_data)
else sync_end_status contains 'Sync Paused'
    SM -> SA: sync_start_time = 0
    SM -> SA: sync_end_time = 0
    SM -> SA: initiate_sync = false
    SM -> DB: update_sync_running_status(connection_name, 'off')
    SM -> SU: create_cron_data_array(connection_name, sync_end_time, 'SCHEDULED_SYNC_PAUSED')
    SM -> SU: update_scheduled_sync(status_array, cron_data)
end

SM -> SA: save()

SM -> DB: get_all_analytics(connection_name)
DB --> SM: analytics_data

alt analytics_data && analytics_data->sync_status_message === 'Sync Completed'
    SM -> DB: update_option('mo_wcps_{connection_name}_sync_running_status', 'Sync Completed')
end

deactivate SM

deactivate SM

== Sync Handler API Sync Process ==

SH -> SH: handle_api_sync()
activate SH

SH -> SH: set_memory_and_time_limits()
SH -> SH: sync_status_message = 'Initiating API call'
SH -> SA: save()

SH -> SH: set_product_data()
activate SH

SH -> SH: get_incoming_sync_product_data()
activate SH
SH -> SH: handle_incoming_sync_request_alteration()
SH -> API: retrieve_api_data(headers, body, url)
activate API
API --> SH: api_response
deactivate API

SH -> SU: convert_response_to_array(connection_name, config, api_response)
activate SU
SU -> SU: prepare_product_object(data, response_attr)
SU -> SU: map_particular_attr(response_attr, data)
SU --> SH: products_array
deactivate SU

SH -> SH: product_data_from_api = products_array
deactivate SH

SH -> SH: filter_product_data()
SH -> SH: sync_products(filtered_data)

deactivate SH

== Product Sync Process ==

SH -> SH: sync_products(product_data)
activate SH

SH -> SH: Determine sync structure (basic_variable_sync, variable_sync_with_parent_attribute, simple_product_sync)
SH -> SP: new Sync_Simple_Products(connection_name) OR new Sync_Variable_Products(connection_name)
activate SP

SP -> SP: product_data_from_api = product_data
SP -> SA: get_current_batch_analytics()
SP -> SA: additional_sync_data['product_per_page'] = count(product_data_from_api)
SP -> SA: save()

SH -> SA: sync_status_message = 'Sync in Progress'
SH -> SA: save()

SP -> SP: sync_products()
activate SP

alt has_filter('mo_wcps_data_alteration_filter')
    SP -> SP: apply_filters('mo_wcps_data_alteration_filter', [event_specifier: 'simple_products_data_array_initial', products_array])
    note right: Hook: mo_wcps_data_alteration_filter\nAvailable for data modification before sync
end

SP -> MP: new MO_Simple_Product(config, [connection_name])
activate MP

loop for each product in product_data_from_api
    SP -> DB: get_sync_running_status(connection_name)
    DB --> SP: sync_status
    
    alt sync_status !== 'off'
        SP -> MP: set_fields(product_data[i])
        activate MP
        
        MP -> MP: get_unique_identifier()
        MP -> MP: get_sku()
        MP -> MP: get_name()
        MP -> MP: get_description()
        MP -> MP: get_short_description()
        MP -> MP: get_regular_price()
        MP -> MP: get_sale_price()
        MP -> MP: get_stock()
        MP -> MP: get_manage_stock()
        MP -> MP: get_in_stock_status()
        MP -> MP: get_image()
        MP -> MP: get_category()
        MP -> MP: get_vendor()
        MP -> MP: get_author()
        MP -> MP: get_brand()
        MP -> MP: get_brand_image()
        MP -> MP: get_custom_fields()
        MP -> MP: get_wc_attributes()
        MP -> MP: get_wc_taxonomies()
        MP -> MP: get_product_status()
        MP -> MP: get_product_status_on_update()
        
        alt has_filter('mo_wcps_data_alteration_filter')
            MP -> MP: apply_filters('mo_wcps_data_alteration_filter', [event_specifier: 'alter_products_category', product_data, categories])
            note right: Hook: mo_wcps_data_alteration_filter\nAvailable for category modification
        end
        
        MP -> MP: save()
        activate MP
        
        MP -> MP: get_product_by_meta() OR wc_get_product_id_by_sku(sku)
        
        alt product_id exists && update_products === 'on'
            MP -> SU: update_product(product_id, mo_product)
            activate SU
            SU -> SU: wc_get_product(product_id)
            SU -> SU: Set product fields (name, description, price, etc.)
            SU -> SU: set_categories(product_id, category)
            SU -> SU: set_image(mo_product, product_id, [connection_name])
            SU -> SU: set_brand(product_id, brand, brand_image)
            SU -> SU: set_custom_fields(custom_fields, product_id)
            SU -> SU: set_wc_attrs(wc_attributes, product_id)
            SU -> SU: add_taxonomy_and_terms(wc_taxonomies, product_id)
            SU --> MP: product_details
            deactivate SU
        else product_id doesn't exist && add_products === 'on'
            MP -> SU: add_product(mo_product)
            activate SU
            SU -> SU: wc_create_product([type, name, description, etc.])
            SU -> SU: Set all product fields
            SU -> SU: Set categories, images, attributes, etc.
            SU --> MP: product_details
            deactivate SU
        end
        
        MP --> SP: product_obj
        deactivate MP
        
        SP -> SP: Extract product_id from product_obj
        
        alt product_id > 0
            SP -> SA: product_saved_success++
            SP -> DL: edit_log('Product saved successfully')
        else
            SP -> SA: product_saved_fail++
            SP -> DL: edit_log('Sync failed for product => ' + print_r(product_data[i], true))
        end
        
        SP -> SA: total_product_parsed++
        SP -> SA: save()
        SP -> DL: edit_log('Current prod ID - ' + product_id)
        
        deactivate MP
        
    else sync_status === 'off'
        alt sync_status === 'paused'
            SP -> SP: return 'Sync Paused'
        else sync_status === 'stopped'
            SP -> SP: return 'Sync Aborted'
        else
            SP -> SP: return 'Sync Completed'
        end
    end
end

SP -> SA: sync_status_message = 'Sync Completed'
SP -> SA: save()

SP --> SH: 'Sync Completed'
deactivate SP

SH -> SA: update_analytics()
SH -> SH: get_detailed_sync_status(sync_end_status, products_sync_type)

SH --> SM: sync_end_status
deactivate SH

== Backup Cron Process ==

WC -> CM: initiate_sync_backup(cron_arguments)
activate CM

CM -> CM: Check license status
alt License expired
    CM -> CM: return (do nothing)
else License valid
    CM -> SM: sync_backup_check(cron_arguments)
end

deactivate CM

SM -> SH: check_sync_process(cron_arguments)
activate SH

SH -> DB: get_all_analytics(connection_name + '_batch', false)
DB --> SH: analytics_array

alt !empty(analytics_array)
    SH -> SH: Check each batch analytics
    loop for each analytics_object
        alt analytics_object->sync_status_message !== 'Sync Completed'
            SH -> DB: get_option(analytics_object->connection_name, 0)
            DB --> SH: last_product_number
            
            alt last_product_number === analytics_object->total_product_parsed
                alt analytics_object->analytics_type === 'vendor_unavailable_products'
                    SH -> CM: enqueue_product_batch_sync(connection_name, [batch_number, page_number, cron_initiator: 'vendor_unavailable_products'])
                else batch_flag === 'on'
                    SH -> SH: filter_product_data()
                    SH -> CM: enqueue_product_batch_sync(connection_name, sync_arguments)
                else page_flag === 'on'
                    SH -> SH: filter_product_data()
                    SH -> CM: enqueue_product_batch_sync(connection_name, sync_arguments)
                end
                
                SH -> DB: update_option(analytics_object->connection_name, analytics_object->total_product_parsed)
            end
        else
            SH -> DB: delete_option(sync_analytics->connection_name)
        end
    end
    
    alt no_sync_remaining
        alt last_batch_analytics->analytics_type !== 'vendor_unavailable_products'
            SH -> CM: enqueue_product_batch_sync(connection_name, [cron_initiator: 'vendor_unavailable_products'])
            SH -> SH: sync_end_status = ''
        else
            SH -> SH: sync_end_status = 'Sync Completed'
            SH -> WC: wp_clear_scheduled_hook('mo_wcps_product_sync_backup_cron', [cron_arguments])
        end
    end
    
else empty(analytics_array)
    SH -> SA: get_sync_analytics_object(connection_name)
    SA --> SH: sync_analytics
    
    alt sync_analytics->sync_status_message === 'Sync in Progress'
        SH -> WC: wp_clear_scheduled_hook('mo_wcps_product_sync_backup_cron', [cron_arguments])
        SH -> SH: sync_end_status = 'Sync analytics cleared!'
    else
        SH -> SH: sync_end_status = ''
    end
end

SH --> SM: sync_end_status
deactivate SH

SM -> SM: end_sync(sync_end_status)
activate SM

SM -> SA: update_analytics()
SM -> SA: sync_status_message = sync_end_status

alt sync_end_status contains 'Sync Completed'
    SM -> SA: sync_end_time = time()
    SM -> SA: initiate_sync = true
    SM -> DB: update_sync_running_status(connection_name, 'off')
    SM -> SU: create_cron_data_array(connection_name, sync_end_time, 'SCHEDULED_SYNC_EXECUTED')
    SM -> SU: update_scheduled_sync(status_array, cron_data)
else sync_end_status contains 'Sync Aborted'
    SM -> SA: sync_end_time = time()
    SM -> SA: initiate_sync = false
    SM -> DB: update_sync_running_status(connection_name, 'off')
    SM -> SU: create_cron_data_array(connection_name, sync_end_time, 'SCHEDULED_SYNC_ABORTED')
    SM -> SU: update_scheduled_sync(status_array, cron_data)
end

SM -> SA: save()

SM -> DB: get_all_analytics(connection_name)
DB --> SM: analytics_data

alt analytics_data && analytics_data->sync_status_message === 'Sync Completed'
    SM -> DB: update_option('mo_wcps_{connection_name}_sync_running_status', 'Sync Completed')
end

deactivate SM

== Batch Sync Process ==

WC -> CM: initiate_batch_sync(cron_arguments)
activate CM

CM -> CM: Check license status
alt License expired
    CM -> CM: return (do nothing)
else License valid
    CM -> SM: start_sync_for_batch() OR start_sync_for_discarding_products()
end

deactivate CM

SM -> DAF: new Default_Apps_Factory(vendor_name)
activate DAF
DAF -> DAF: Initialize vendor-specific configurations
deactivate DAF

SM -> SH: start_sync_process()
activate SH

SH -> SH: handle_api_sync() OR set_leftover_product_status()
activate SH

alt cron_initiator === 'vendor_available_products'
    SH -> SH: handle_api_sync()
    SH -> SH: sync_products(product_data)
    SH -> SH: initiate_next_process(next_process_flag)
    
    alt next_process_flag === 'Initiate batch sync'
        SH -> CM: enqueue_product_batch_sync(connection_name, [batch_number + 1, page_number, cron_initiator: 'vendor_available_products', sync_type: 'batch_sync'])
    else next_process_flag === 'Initiate page sync'
        SH -> CM: enqueue_product_batch_sync(connection_name, [batch_number, page_number + 1, cron_initiator: 'vendor_available_products', sync_type: 'page_sync'])
    else next_process_flag === 'Discard products'
        SH -> CM: enqueue_product_batch_sync(connection_name, [batch_number + 1, page_number + 1, cron_initiator: 'vendor_unavailable_products', sync_type: 'leftover_sync'])
    end
    
else cron_initiator === 'vendor_unavailable_products'
    SH -> SH: set_leftover_product_status()
    SH -> DB: get_website_products(args)
    DB --> SH: product_ids
    
    alt count(product_ids) > 0
        SH -> SU: update_leftover_products(product_ids, discarding_details, sync_analytics)
        activate SU
        SU -> SU: mo_bulk_update_or_delete_product_status(product_ids, set_status, sync_analytics)
        SU --> SH: sync_status
        deactivate SU
    end
    
    SH -> SA: sync_status_message = sync_status
    SH -> SA: save()
    SH -> WC: wp_clear_scheduled_hook('mo_wcps_product_sync_backup_cron', [connection_name])
end

SH --> SM: sync_end_status
deactivate SH

SM -> SM: end_sync(sync_end_status)
activate SM

SM -> SA: update_analytics()
SM -> SA: sync_status_message = sync_end_status

alt sync_end_status contains 'Sync Completed'
    SM -> SA: sync_end_time = time()
    SM -> SA: initiate_sync = true
    SM -> DB: update_sync_running_status(connection_name, 'off')
    SM -> SU: create_cron_data_array(connection_name, sync_end_time, 'SCHEDULED_SYNC_EXECUTED')
    SM -> SU: update_scheduled_sync(status_array, cron_data)
else sync_end_status contains 'Sync Aborted'
    SM -> SA: sync_end_time = time()
    SM -> SA: initiate_sync = false
    SM -> DB: update_sync_running_status(connection_name, 'off')
    SM -> SU: create_cron_data_array(connection_name, sync_end_time, 'SCHEDULED_SYNC_ABORTED')
    SM -> SU: update_scheduled_sync(status_array, cron_data)
end

SM -> SA: save()

SM -> DB: get_all_analytics(connection_name)
DB --> SM: analytics_data

alt analytics_data && analytics_data->sync_status_message === 'Sync Completed'
    SM -> DB: update_option('mo_wcps_{connection_name}_sync_running_status', 'Sync Completed')
end

deactivate SM

deactivate SM

== Available Hooks Summary ==

note over User, API
**Available WordPress Hooks:**

1. **mo_wcps_perform_action_before_sync**
   - Triggered before sync starts
   - Parameters: sync_manager object
   - Use: Custom actions before sync

2. **mo_wcps_data_alteration_filter**
   - Triggered during product data processing
   - Parameters: array with event_specifier and data
   - Events: 'simple_products_data_array_initial', 'alter_products_category'
   - Use: Modify product data before sync

3. **mo_wcps_product_sync_cron**
   - Main sync cron hook
   - Parameters: cron_arguments array
   - Use: Custom sync processing

4. **mo_wcps_product_batch_sync_cron**
   - Batch sync cron hook
   - Parameters: cron_arguments array
   - Use: Custom batch processing

5. **mo_wcps_product_sync_backup_cron**
   - Backup/monitoring cron hook
   - Parameters: cron_arguments array
   - Use: Sync monitoring and recovery

6. **mo_wcps_sync_scheduler**
   - Scheduled sync hook
   - Parameters: connection_name
   - Use: Scheduled sync processing

7. **mo_wcps_delete_product_images_cron**
   - Image cleanup cron hook
   - Parameters: product_ids array
   - Use: Cleanup unused product images
end note

@enduml 