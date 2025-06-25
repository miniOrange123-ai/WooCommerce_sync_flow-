@startuml
title Supplier Sync Sequence Diagram

actor "Scheduler" as Scheduler
participant "Supplier Sync Service" as SyncService
participant "Connection1 (Priority 1)" as Conn1
participant "Connection2 (Priority 2)" as Conn2
participant "Connection3 (Priority 3)" as Conn3

Scheduler -> SyncService : Start Supplier Sync
activate SyncService

SyncService -> SyncService : Get Connections by Priority

SyncService -> Conn1 : Start Connection1
activate Conn1
Conn1 --> SyncService : Connection1 Complete
deactivate Conn1

SyncService -> Conn2 : Start Connection2
activate Conn2
Conn2 --> SyncService : Connection2 Complete
deactivate Conn2

SyncService -> Conn3 : Start Connection3
activate Conn3
Conn3 --> SyncService : Connection3 Complete
deactivate Conn3

SyncService -> Scheduler : Supplier Sync Complete
deactivate SyncService

@enduml
