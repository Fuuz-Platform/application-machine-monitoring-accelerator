# application-machine-monitoring-accelerator

**Version:** 1.0.5
**Platform:** Fuuz ≥ 2025.1.1
**Enterprise:** MFGx
**Spec Version:** 2.0.0

---

## Overview

The Machine Monitoring Accelerator is a comprehensive, production-ready Fuuz application for real-time shop floor visibility, OEE (Overall Equipment Effectiveness) tracking, and IoT-driven machine data collection. It provides the full operational layer between physical manufacturing equipment and digital production records — capturing machine states, production counts, scrap events, downtime, and alarms in real time.

Built around the ISA-95 manufacturing hierarchy and ISO 22400 OEE standards, this accelerator supports discrete and process manufacturers who need accurate, shift-level machine performance data without custom development. It integrates with PLCs and OPC-UA devices via IoT tags, generates hourly OEE records, aggregates performance across configurable dimensions (product, work order, shift, workcenter group), and surfaces all data through operator-facing dashboards and management-level plant views.

### Core Capabilities

- **Real-time machine state tracking** — Workcenter mode changes (production, idle, maintenance, changeover, unplanned downtime) recorded as an immutable history log with timestamps, duration, and shift assignment
- **Automated OEE calculation** — Hourly Availability × Performance × Quality computation per workcenter per shift; aggregated across configurable time frames (daily, weekly, monthly) and dimensions (product, work order, etc.)
- **PLC-integrated production counting** — Counter accumulator logic tracks raw PLC pulse counts, derives production quantities via delta comparison, supports multi-cavity/multi-output workcenters, and posts production records automatically or on-demand
- **IoT device subscriptions** — Subscribes to OPC-UA device tags, maintains current and historical tag values, routes incoming IoT messages to the correct production or mode-change handler
- **Event and alarm management** — Structured event taxonomy with priority levels, escalation logic (level-1 / level-2 notification timers), alarm lifecycle (new → acknowledged → cleared/canceled), and Gantt-ready downtime history
- **Shift scheduling engine** — Day-of-week shift schedules with configurable start/end times, timezone support, and `creditAtStart` flag for cross-midnight shift date attribution
- **Operator control panel** — Per-workcenter HMI screens for mode selection, production setup, output management, scrap recording, and badge-in/out
- **Plant dashboard** — WorkcenterGroup-based multi-machine views showing live OEE, current mode/event, production rates, and alarm counts

---

## Package Contents

```
machine-monitoring/
├── manifest.json
├── definition.json
├── package-data.json
├── data/                        34 seed data files
├── dataFlows/                   38 automation flows
├── dataModels/                  78 data model definitions
├── savedTransforms/             22 reusable data transforms
└── screens/                     84 UI screen definitions
```

---

## Functional Areas

### 1. Workcenter Management

Workcenters are the atomic unit of machine monitoring. Each physical machine or work cell is represented as a `Workcenter` record with configuration for:

- **Production modes** — Linked to a `ModeGroup` which contains an ordered `ModeList` of valid `Mode` records. Each mode entry can have a `plcCode` for PLC-driven automatic mode switching and an optional associated `EventGroup` for structured event capture.
- **Shift scheduling** — Assigned to a `ShiftGroup` for OEE shift resolution. The engine looks up the active shift at any given moment using the workcenter's shift group assignment.
- **OEE rate configuration** — Default ideal rate stored at workcenter level; overridden at the `ApprovedWorkcenter` (product-operation) level for product-specific takt time calculations.
- **External system sync flags** — `productionHistorySync`, `runSync`, and `workcenterHistorySync` flags control whether production data flows from an external ERP rather than being originated in Fuuz.
- **IoT tag binding** — Many-to-many `WorkcenterIotTag` junction associates OPC-UA tags from connected devices to a specific workcenter for counter reading, mode signals, and scrap signals.
- **Facility assignment** — Workcenters are optionally grouped under a `Facility` (type: Production, Distribution, or Subcontract) following the ISA-95 site hierarchy.
- **WorkcenterGroups** — Cross-cutting logical collections (independent of ERP setup) used to build plant-level dashboards. Groups have `externalId` for ERP mapping and `customData` for extended metadata.
- **Current state pointer** — `currentWorkcenterHistoryId` (unique foreign key) always points to the open `WorkcenterHistory` record, enabling instant current-state lookups without a query scan.

### 2. Production State & Setup

The `ProductionSetup` table functions as a one-per-workcenter "live state" record — the single source of truth for what the machine is currently doing:

- **Current mode** — `modeId` reflects the active `Mode`; changing this creates a new `WorkcenterHistory` record and triggers OEE recalculation for the hour
- **Current event** — `eventId` references a specific `Event` from the workcenter's configured `ModeGroup` event lists; event updates modify the open `WorkcenterHistory` record without creating a new one
- **Current production run** — `productionRunId` links to the active `ProductionRun` when the workcenter is executing a scheduled job
- **Operators on duty** — `operators` (JSON array) tracks who is logged in; used in labor efficiency calculations
- **Static bulletins** — `note` field stores free-text instructions visible on operator dashboards without triggering system events
- **Outputs** — One-to-many `ProductionSetupOutput` records, one per active product/work-order/cavity combination

**ProductionSetupOutput** is where PLC integration and production tracking converge:

- `outputId` — cavity number for multi-cavity dies (enabled by `Workcenter.multiOutCapable`)
- `counterInput` / `counterAccum` / `counterLastProduction` — three-field PLC counter accumulator: `counterInput` = last raw PLC value; `counterAccum` = total accumulated across PLC resets; `counterLastProduction` = counter value at last production post; production quantity = `counterInput − counterLastProduction`
- `autoProduction` — when true, the IoT pipeline automatically posts production records on each counter update instead of waiting for operator confirmation
- `unitPerCycle` — floating-point multiplier for dies that produce multiple parts per machine stroke
- `standardQuantity` / `idealRate` / `setupRate` — rate targets sourced from linked `ApprovedWorkcenter` for the product-operation combination
- `producedQuantity` / `scrappedQuantity` — running totals for the current setup
- `remainingBalance` / `dueAt` / `scheduledStartAt` — work order schedule data for operator guidance
- `distributionStatus` / `workOrderProcessDistributionId` — WMS integration hooks for distribution workflow coupling

### 3. Production History

`ProductionHistory` is the immutable append-only ledger of all production and scrap transactions. Every quantity posted — whether from operator entry, PLC auto-production, or ERP sync — creates a record here.

**Key fields:**
- `quantity` — units produced or scrapped (required)
- `occurAt` — timestamp of the event (defaults to `$moment()` at creation)
- `creditedAt` — adjusted date for shift attribution (controlled by `Shift.creditAtStart` for cross-midnight shifts)
- `transactionTypeId` — links to `TransactionType` defining the nature of the record (productionAdd, scrapAdd, productionSetupAdd, productionSetupClear, modeChange, etc.)
- `shiftId` — resolved shift at time of posting; used for OEE aggregation
- `workcenterId` / `approvedWorkcenterId` / `productionRunId` / `scrapReasonId` — full traceability chain
- `externalId` — NetSuite Work Order Completion or Assembly Build ID for ERP reconciliation

**Traceability fields** (for regulated manufacturing):
- `serialNo` — auto-sequenced from `serialNumbers` sequence
- `lotNo`, `heatNo`, `heatCode`, `masterUnitNo`, `trackingNo` — material genealogy
- `productNumber`, `productNumberRevision`, `productRevision` — revision-controlled product identity
- `operationCode`, `operationNumber`, `workOrderNumber` — routing and shop order linkage

`dataChangeCapture` is enabled and exposed, enabling downstream event streaming for ERP integration.

### 4. OEE Calculation & Reporting

OEE is calculated per ISO 22400: **OEE = Availability × Performance × Quality**

**Hourly OEE (`Oee` model):**
- One record per workcenter per shift-hour per shift
- `shiftHour` — 1-based hour index within the shift (e.g., hour 3 of an 8-hour shift)
- `availability` — productionTime / totalTime; `productionTime` = scheduled time minus unplanned downtime
- `performance` — (totalQuantity × averageCycleTime) / productionTime; uses `idealRate` from ApprovedWorkcenter or Workcenter default
- `quality` — totalProductionQuantity / (totalProductionQuantity + totalScrapQuantity)
- `counter` — total machine cycles from PLC or manual entry for the hour; used when cycle time is more reliable than rate
- `plannedDowntime` / `unplannedDowntime` / `other` — seconds in each state category for the hour
- `creditedAt` — adjusted report date (respects `Shift.creditAtStart` for cross-midnight shifts)
- Indexed on `(workcenterId, workcenterGroupId, shiftId, shiftHour, startAt, endAt)` and `(workcenterId, shiftId, creditedAt)` for fast dashboard queries

**Aggregated OEE (`OeeAggregate` model):**
- Rolled-up OEE across configured dimensions with weighted averages (`availabilityWeight`, `performanceWeight`, `qualityWeight`)
- Linked to `OeeConfiguration` (which dimensions to apply) and `OeeTimeFrame` (duration in seconds — daily/weekly/monthly)
- Dimension slicing by `productNoRevision`, `workOrderCode`, `workOrderNumber`, `shiftId`, `workcenterId`, `workcenterGroupId`
- `OeeConfiguration.lastRanAt` is automatically set to the start of the configured time frame in America/Detroit timezone when a configuration is created

**OEE Time Frames (`OeeTimeFrame`):**
- `label` — auto-set as the record ID via create trigger
- `duration` — seconds (e.g., 86400 = daily, 604800 = weekly, 2592000 = monthly)
- Used by scheduler to determine when aggregate recalculation runs

**Key data flows:**
- *Calculate OEE* — core system flow; triggered on WorkcenterHistory changes, computes hourly Oee record
- *Get OEEs for shift* — integration flow serving dashboard OEE query by shift date range
- *Get OEEs for shift Production Run* — integration flow filtering OEE by active production run

### 5. Workcenter History & Downtime

`WorkcenterHistory` is the time-series log of every machine state change:

- Each record represents one continuous interval at a single mode/event combination
- `occurAt` — start of the interval ("logDate")
- `endAt` — end of the interval ("time resolved"); null for the currently-open record
- `duration` — `endAt − occurAt` in seconds; used directly in OEE time component calculations
- `disabled` — set from `Event`; marks the interval as "machine down" for OEE runtime calculation
- `includeInOee` — set from `Mode`; determines if interval counts toward OEE scheduled time
- `requireNote` — forces operator to enter a note before the record can be closed
- `modeId` / `eventId` — snapshot of mode and event active during this interval
- `productionRunId` — links to the run active at time of state change
- `shiftId` — resolved shift for time attribution

**Back Fill End Dates** data flow handles edge cases where records are left open across shifts or system restarts, ensuring OEE calculation always has complete time windows.

### 6. Event & Downtime Taxonomy

A three-level event taxonomy provides structured downtime classification:

**EventType** — defines OEE impact flags:
- `running`, `starved`, `blocked`, `idle`, `disabled` (unplanned downtime for OEE), `plannedDowntime`, `unplannedDowntime`

**EventCategory** — 8-way classification:
- `equipment`, `labor`, `material`, `quality`, `schedule`, `ancillary`, `support`, `other`

**EventPriority** — defines escalation thresholds:
- `redMinutes` / `yellowMinutes` — age thresholds for visual alerting
- `critical`, `high`, `low`, `medium` — priority tier flags
- `configurationSchema` — JSON schema for priority-specific metadata capture

**EventGroup / EventList** — groups events for assignment to Mode entries:
- `EventList` associates an `Event` to an `EventGroup` with display `order`, `default` flag, `plcCode` for IoT-driven selection, `requireNote`, and optional `EventPriority`
- `alarm` flag on EventList triggers automatic alarm creation when that event is selected (see *Create Alarm on Event* flow)

**Event** — individual downtime/event entries:
- `externalId` — PLC integer code for automatic event assignment from IoT signals
- `eventCategoryId`, `eventTypeId` — taxonomy linkage
- `customData` — extensible metadata

### 7. Alarm Management

Alarms provide escalating notification when workcenters enter abnormal states:

- `Alarm` records are auto-created by the *Create Alarm on Event* flow when an EventList entry has `alarm: true`
- `alarmStatusId` → `AlarmStatus` tracks lifecycle state with Boolean flags: `new`, `active`, `acknowledged`, `cleared`, `canceled`
- `workcenterHistoryId` (unique) — one alarm per open workcenter history record; cleared automatically when the interval closes
- `levelOneSent` / `levelTwoSent` (DateTime) — escalation timestamps; populated when notifications fire at configured priority thresholds
- `assignedRoleId` / `assignedUserId` — enables alarm routing to specific personnel or roles
- `acknowledgedAt`, `assignedAt`, `canceledAt`, `clearedAt` — full lifecycle audit trail
- TTL on `updatedAt`: ~32 million seconds (~1 year) — old closed alarms auto-expire

*Alarm State Change Screen Flow* — handles operator interactions from the alarm management screen, driving status transitions.

### 8. Mode Management

**Mode** — named machine states with OEE impact flags:
- `idle` — machine is not producing but is healthy
- `includeProductionCounts` — whether production posted in this mode counts toward OEE quality numerator
- `usable` — controls availability in operator mode selection lists

**ModeType** — classifies modes into 6 operational categories:
- `production`, `idle`, `changeover`, `maintenance` (planned downtime), `disabled` (unplanned downtime), `other`
- Each ModeType has a `color` for visual differentiation in Gantt and dashboard views

**ModeGroup / ModeList** — per-workcenter configuration of which modes are available:
- `ModeList` entries carry display `order`, `default` flag, `plcCode` for PLC-driven switching, and optional `EventGroup` for event prompting when that mode is selected
- *WorkcenterModeChangeLaunch* screen flow — initiates mode change from operator HMI
- *Change Workcenter Mode* integration flow — applies the mode change, creates WorkcenterHistory record, triggers OEE recalculation
- *No Mode Group Configured* screen flow — error handler for workcenters missing mode group assignment

### 9. IoT Integration

Full OPC-UA device integration pipeline:

**Device → Tag → Workcenter pipeline:**
1. Devices (OPC-UA servers, PLCs) are registered in the platform
2. `IotTag` records define individual tag subscriptions: `name`, `configuration` (JSON — endpoint, node ID, polling interval), `iotTagTypeId`, `iotTagUseCaseId`, `storeHistory` flag, `currentValue` (JSON — last received value), `currentValueRecordedAt`
3. `WorkcenterIotTag` junction maps each tag to one or more workcenters
4. *Create Device Subscription From Iot Tags* — system flow that generates OPC-UA subscriptions on the device from the IotTag configuration
5. *Update Iot Tag by Device Subscriptions* — receives subscription update events, updates `IotTag.currentValue`
6. *Create Historical IOT Tag Values* — when `storeHistory = true`, appends each update to `IotTagHistoricalValue` for trend analysis
7. *IOT Tag Handler* — the main routing flow; reads `iotTagUseCaseId` to determine if incoming value triggers production posting, mode change, scrap recording, or counter update

**IotTagUseCase** controls dispatch routing:
- Use cases map to specific handler behaviors (e.g., "Production Counter", "Mode Signal", "Scrap Signal", "Generic")
- `configurationSchema` per use case defines the expected tag configuration structure

**7-day data change capture** on IotTag (exposed) enables real-time streaming of tag value updates to connected systems.

### 10. Shift & Scheduling

**Shift** — defines a single named work period:
- `startHour`/`startMinute`, `endHour`/`endMinute` — local time bounds
- `timeZone` — IANA timezone string for shift resolution
- Day-of-week flags (`monday`…`sunday`) — all default true; set false to exclude days
- `creditAtStart` — when true, production from a cross-midnight shift (e.g., 11 PM–7 AM) credits to the start date rather than the end date; critical for accurate daily production reporting for third-shift operations

**ShiftGroup** — named collection of shifts assigned to workcenters:
- Multiple workcenters share the same shift group for consistent OEE shift resolution
- `usable` flag controls active assignment

### 11. Product Rate Configuration

**ApprovedWorkcenter** — product-operation-workcenter rate configuration:
- `code` — auto-sequenced identifier
- `idealRate` — takt time target (the denominator in OEE performance calculation when product-specific rates are configured)
- `crewSize`, `setupCrewSize`, `setupTime` — labor standard data
- `standardProductionRate`, `standardQuantity`, `targetRate`, `unitsPerCycle` (Int) — production standard references
- `operationCode`, `operationNo`, `productOperationId` — routing linkage
- `productId`, `productNo`, `productNoRevision`, `productRevision` — product identity at the revision level
- `active` — enables/disables without deleting historical records

**ScrapReason** — configurable scrap classification:
- `code` — unique identifier (also the display label)
- `plcCode` — IoT signal value that auto-posts a scrap record for this reason
- `externalId` — ERP scrap reason code for two-way sync
- `usable` — controls availability in operator scrap entry screens

---

## Data Flows

| # | Name | Type | Module | Description |
|---|------|------|--------|-------------|
| 0 | Alarm State Change Screen Flow | Screen | manufacturing | Handles alarm lifecycle transitions from operator screens |
| 1 | Back Fill End Dates of Workcenter History | System | integration | Closes open WorkcenterHistory records across shift boundaries |
| 2 | Calculate OEE | System | manufacturing | Computes hourly Availability × Performance × Quality per workcenter/shift |
| 3 | Change Workcenter Mode | Integration | manufacturing | Applies mode change, creates WorkcenterHistory record, triggers OEE |
| 4 | Control Panel Router | System | manufacturing | Routes operator control panel actions to appropriate handlers |
| 5 | Create Device Subscription From Iot Tags | System | internetOfThings | Generates OPC-UA subscriptions from IotTag configuration |
| 6 | Create Historical IOT Tag Values | System | internetOfThings | Persists IotTag value history when storeHistory is enabled |
| 7 | IOT Tag Handler | System | internetOfThings | Routes incoming IoT messages by use case to production/mode handlers |
| 8 | Load Workcenter Outputs | System | manufacturing | Loads ProductionSetupOutput records for active workcenter |
| 9 | Mode Event Gantt Data from Workcenter History | System | dashboard | Transforms WorkcenterHistory into Gantt chart format for dashboards |
| 10 | No Mode Group Configured | Screen | manufacturing | Error handler for workcenters without mode group assignment |
| 11 | Publish on Production Setup Mode Change | System | — | Event publisher for mode change notifications to subscribers |
| 12 | Record Production | System | manufacturing | Posts ProductionHistory records from operator entry or PLC counter delta |
| 13 | Start Stop Run | System | manufacturing | Manages ProductionRun start/stop lifecycle at workcenter |
| 14 | Update Iot Tag by Device Subscriptions | System | internetOfThings | Processes device subscription updates, persists current tag value |
| 15 | WorkcenterModeChangeLaunch | Screen | manufacturing | Initiates mode change dialog from operator HMI |
| 16 | Get OEEs for shift | Integration | dashboard | Query flow returning OEE records for a shift date range |
| 17 | Create Alarm on Event | System | manufacturing | Auto-creates Alarm record when an EventList entry with alarm=true is selected |
| 18 | Get OEEs for shift Production Run | Integration | dashboard | Query flow filtering OEE by active production run |
| 19–37 | Additional Flows (19) | Various | Various | Production run management, plant dashboard data, advanced OEE, scheduling integrations, and reporting flows |

---

## Data Models

### Core Machine & Workcenter

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Workcenter` | code!, name!, active!, autoProduction, multiOutCapable, shiftGroupId, modeGroupId, workcenterGroupId | Central machine entity; 75 schema versions |
| `WorkcenterGroup` | name!, usable!, externalId | Plant-level dashboard groupings independent of ERP |
| `WorkcenterHistory` | occurAt, endAt, duration, disabled, includeInOee, modeId, eventId, workcenterId | Immutable machine state time-series; 39 schema versions |
| `WorkcenterMode` | number (auto), plcCode, modeId | Workcenter-level mode assignment with PLC code |
| `WorkcenterIotTag` | workcenterId, iotTagId | Many-to-many IoT tag binding; dataChangeCapture enabled |
| `Facility` | code!, name!, type (Production/Distribution/Subcontract), active | ISA-95 site hierarchy node; 120-day data capture |

### Production

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `ProductionSetup` | workcenterId (unique!), modeId, eventId, productionRunId, operators | Live workcenter state record; 53 schema versions |
| `ProductionSetupOutput` | outputId, productNumber!, counterInput/Accum/LastProduction, autoProduction, unitPerCycle, workcenterId! | Per-output PLC integration and quantity tracking; 23 schema versions |
| `ProductionHistory` | quantity!, occurAt!, creditedAt, shiftId!, transactionTypeId!, workcenterId, serialNo (auto-seq) | Immutable production/scrap ledger; 47 schema versions |
| `ApprovedWorkcenter` | code (auto-seq), idealRate, crewSize, productId!, workcenterId! | Product-operation rate standards per workcenter |
| `ScrapReason` | code! (unique), plcCode, externalId, usable | Configurable scrap classification with IoT and ERP binding |
| `TransactionType` | name! (=id), productionAdd!, scrapAdd!, modeChange!, 28 additional Boolean flags, color! | Transaction classification; dataChangeCapture enabled |

### OEE

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Oee` | shiftHour, availability, performance, quality, oee!, counter, creditedAt!, workcenterId | Hourly OEE record per workcenter/shift; 23 schema versions |
| `OeeAggregate` | All Oee fields + weighted averages, productNoRevision, workOrderCode, oeeConfigurationId, oeeTimeFrameId | Dimensional OEE rollup; 9 schema versions |
| `OeeConfiguration` | name, lastRanAt (auto-set, Detroit tz), oeeTimeFrameId! | Defines aggregation dimensions and schedule; 13 schema versions |
| `OeeDimension` | label, field (grouping key), description | Defines one aggregation dimension (e.g., by product) |
| `OeeDimensionOeeConfiguration` | oeeConfigurationId, oeeDimensionId (unique composite) | M:M junction linking dimensions to configurations |
| `OeeTimeFrame` | label (=id auto), duration (Int seconds), description | Named time windows for OEE rollup (daily/weekly/monthly) |

### Events & Alarms

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Alarm` | number (auto-seq), alarmStatusId!, workcenterId!, workcenterHistoryId! (unique), levelOneSent, levelTwoSent | Alarm record with escalation tracking; TTL ~1 year |
| `AlarmStatus` | name! (unique), active, new, acknowledged, cleared, canceled, usable! | Alarm lifecycle state definitions |
| `Event` | name! (unique), externalId (PLC int), eventCategoryId, eventTypeId, eventListId, usable | Named event/downtime entry |
| `EventType` | name! (unique), disabled, plannedDowntime, unplannedDowntime, running, starved, blocked, idle, usable! | OEE impact classification per event type |
| `EventCategory` | equipment, labor, material, quality, schedule, ancillary, support, other | 8-way categorical classification |
| `EventGroup` | name! (unique), usable | Container for EventList entries |
| `EventList` | order, alarm, default, requireNote, plcCode, eventId!, eventGroupId, eventPriorityId | Event-to-group mapping with IoT code and alarm trigger |
| `EventPriority` | name! (unique), redMinutes, yellowMinutes, critical, high, low, medium, usable | Priority tiers with escalation time thresholds |

### Modes

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Mode` | name!, idle!, includeProductionCounts, usable!, modeTypeId | Named machine state |
| `ModeType` | name! (unique), production, idle, changeover, maintenance, disabled, other, color, usable | OEE category classification for modes |
| `ModeGroup` | name! (unique), usable | Container for ModeList entries assigned to workcenters |
| `ModeList` | order, default, plcCode, modeId!, modeGroupId!, eventGroupId | Mode-to-group mapping with PLC code |

### Shifts

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Shift` | name!, startHour/Minute, endHour/Minute, timeZone, creditAtStart!, day-of-week flags, shiftGroupId! | Named shift schedule; 27 schema versions |
| `ShiftGroup` | name!, usable | Grouping of shifts assigned to workcenters |

### IoT

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `IotTag` | name! (unique), active!, storeHistory!, currentValue (JSON), configuration (JSON), deviceId, iotTagTypeId, iotTagUseCaseId | OPC-UA/IoT data point; 7-day dataChangeCapture |
| `IotTagHistoricalValue` | recordedAt!, value (JSON!), iotTagId | Historical tag value time-series |
| `IotTagType` | name! (unique, =id auto), usable!, configurationSchema | Tag type classification with schema definition |
| `IotTagUseCase` | name! (unique, =id auto), usable!, configurationSchema | Routing classification (Production Counter, Mode Signal, etc.) |

---

## Screens (84 total)

Screens provide operator-facing HMI panels, management dashboards, and configuration interfaces:

**Operator HMI**
- Workcenter control panel with mode selection, event logging, production entry, and scrap recording
- Multi-output production setup for multi-cavity workcenters
- Badge-in / badge-out operator tracking
- Alarm acknowledgment and escalation views

**Plant / Line Dashboards**
- WorkcenterGroup overview — live OEE, current mode/event, production vs. target
- Mode/Event Gantt chart — time-axis view of machine state history per shift
- OEE trend views — hourly, shift, daily, weekly, monthly breakdowns

**Configuration Screens**
- Workcenter setup — mode group, shift group, IoT tag assignment, rate configuration
- Shift and ShiftGroup management
- Mode, ModeGroup, ModeList configuration
- Event, EventGroup, EventPriority configuration
- AlarmStatus configuration
- OeeConfiguration and OeeTimeFrame management
- ScrapReason and TransactionType management

---

## Seed Data (34 records)

Pre-configured reference data included on install:

- **ModeTypes** — production, idle, changeover, maintenance, disabled, other (with default colors)
- **TransactionTypes** — full set of 31 named transaction classifications with color assignments
- **AlarmStatuses** — new, active, acknowledged, cleared, canceled with Boolean flag presets
- **EventTypes** — running, blocked, starved, idle, planned downtime, unplanned downtime
- **EventCategories** — all 8 categories (equipment, labor, material, quality, schedule, ancillary, support, other)
- **EventPriorities** — default priority levels with escalation minute thresholds
- **IotTagTypes** and **IotTagUseCases** — standard OPC-UA tag type and use-case classifications
- **OeeTimeFrames** — daily (86400s), weekly (604800s), monthly (2592000s)

---

## Key Integrations

### ERP / External Systems
- `ProductionHistory.externalId` — NetSuite Work Order Completion / Assembly Build ID
- `ScrapReason.externalId` — ERP scrap reason mapping
- `WorkcenterGroup.externalId` — ERP workcenter group mapping
- `Workcenter.externalId` — external system machine ID
- Sync flags: `productionHistorySync`, `runSync`, `workcenterHistorySync` — enable pull-from-ERP mode per workcenter

### PLC / OPC-UA
- `WorkcenterMode.plcCode` — integer code sent by PLC to trigger mode changes
- `ModeList.plcCode` — PLC code for mode signal routing
- `EventList.plcCode` — PLC code for event signal routing
- `ScrapReason.plcCode` — PLC code for scrap signal routing
- `IotTag.configuration` — full OPC-UA subscription configuration (endpoint, node ID, interval)
- `ProductionSetupOutput.counterInput/Accum/counterLastProduction` — three-register PLC pulse counter accumulator

### Printing / Labeling
- `Workcenter.printerId` → `Device` — label printer assigned to each workcenter for production record labeling

---

## Installation

1. Ensure Fuuz platform version ≥ 2025.1.1 is deployed in the target environment
2. Import the package via Fuuz Package Manager using `manifest.json` (specVersion 2.0.0)
3. Seed data (34 records) will be applied automatically on first import — ModeTypes, TransactionTypes, AlarmStatuses, EventTypes, EventCategories, EventPriorities, IoT reference data, and OEE time frames
4. Configure `ShiftGroup` and `Shift` records for your facility's schedule (start/end times, timezone, day-of-week flags, `creditAtStart` for third shift)
5. Create `ModeGroup` and `ModeList` entries defining valid modes per workcenter type, with PLC codes if PLC integration is in scope
6. Create `EventGroup` and `EventList` entries for structured downtime classification; assign priority levels and enable `alarm` flag on critical events
7. Create `Facility` and `WorkcenterGroup` records following your ISA-95 site hierarchy and plant dashboard layout
8. Create `Workcenter` records and assign `ShiftGroup`, `ModeGroup`, `Facility`, `WorkcenterGroup`, and optional `screenId`
9. Configure `ApprovedWorkcenter` records for each product-operation-workcenter combination requiring product-specific OEE rate standards
10. For IoT/PLC integration: create `IotTag` records with device and OPC-UA node configuration; run *Create Device Subscription From Iot Tags* flow; map tags to workcenters via `WorkcenterIotTag`; configure `iotTagUseCaseId` to route signals to the correct handler
11. Configure `OeeConfiguration` and link to `OeeTimeFrame` and `OeeDimension` records for automated OEE aggregation reporting
12. Activate operator screens and plant dashboards from the Screens library; assign `screenId` to Workcenter records for per-machine control panel routing

---

## Dependencies

- **Fuuz Platform** ≥ 2025.1.1 — required for specVersion 2.0.0 package support
- **Device / OPC-UA module** — required for IoT tag subscription pipeline (flows 5, 6, 7, 14)
- **Scheduler module** — required for automated OEE aggregate recalculation and back-fill flows
- **Notification module** (optional) — required for alarm escalation level-1/level-2 email/SMS delivery
- **ERP Connector** (optional) — required when `productionHistorySync`, `runSync`, or `workcenterHistorySync` flags are enabled on any Workcenter

---

*Built on the [Fuuz Industrial Operations Platform](https://fuuz.com) — [Documentation](https://help.fuuz.com)*

## Service levels

No service level agreement applies to anything published here. It becomes a supported
deliverable only once it has been implemented by a Fuuz services professional or an
approved Fuuz partner.
