@startuml
title Connection Sync Sequence Diagram

actor "Scheduler" as Scheduler
participant "Connection Sync Service" as SyncService

Scheduler -> SyncService : Start Connection Sync
activate SyncService

SyncService -> SyncService : Sync in Progress

SyncService --> Scheduler : Connection Sync Complete
deactivate SyncService

@enduml
