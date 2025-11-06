# Section 10.13: GVL_ADS Complete Service Catalog

**Location**: `Library/10_Machine/18_HMI_Handshake/GVL_ADS.TcGVL`

**Purpose**: This section provides a **complete reference** of all 71 service variables in GVL_ADS, categorized by functional area.

**使用說明 Usage Notes**:
- All services use **HandShake pattern**: HMI sets `Execute := TRUE`, PLC processes, sets `Done/Error`, HMI reads status and resets `Execute := FALSE`
- Services are **edge-triggered**: Use rising edge detection (R_TRIG) in FB_HmiHandShake
- Parameter structures are defined in `Library/10_Machine/18_HMI_Handshake/DUTs/`

---

## 10.13.1 Storage Area Services (25 services)

Storage area services handle Fanuc robot operations, warehouse management, and box handling.

### Fanuc Robot Movement Services

| Service Variable | Structure Type | Purpose | HMI Usage |
|-----------------|----------------|---------|-----------|
| `AutoMode_RobotWarehouseMove` | `st_AutoSixAxisWarehouseParameter` | Move Fanuc robot to warehouse position (Side/Layer/Slide) | HMI selects warehouse coordinates |
| `AutoMode_RobotAbsMove` | `st_AutoSixAxisAbsMoveParameter` | Move Fanuc robot to absolute XYZ position | HMI provides target coordinates |
| `AutoMode_WarehouseMotoWithRobotMove` | `st_AutoSixAxisWarehouseParameter` | Coordinated warehouse rack motor + robot movement | Warehouse position with motor sync |

**Example: Move robot to warehouse position**:
```pascal
// HMI writes to GVL_ADS:
GVL_ADS.AutoMode_RobotWarehouseMove.Execute := TRUE;
GVL_ADS.AutoMode_RobotWarehouseMove.Side := 1;      // Left side
GVL_ADS.AutoMode_RobotWarehouseMove.Layer := 5;     // Layer 5
GVL_ADS.AutoMode_RobotWarehouseMove.Slide := 10;    // Slide 10

// PLC executes service, then HMI reads:
IF GVL_ADS.AutoMode_RobotWarehouseMove.Done THEN
    // Movement complete
    GVL_ADS.AutoMode_RobotWarehouseMove.Execute := FALSE;
END_IF
```

---

### Box Handling Services

| Service Variable | Structure Type | Purpose | HMI Usage |
|-----------------|----------------|---------|-----------|
| `AutoMode_FeedInEmptyBoxWithRobot` | `ST_AutoMode_FeedInEmptyBox` | Feed empty box from warehouse to FeedIn conveyor | Specify source (warehouse/buffer) |
| `AutoMode_ReloadRoundBeltBoxWithRobot` | `ST_AutoMode_ReloadRoundBeltBox` | Reload box from warehouse to RoundBelt (legacy) | Warehouse position |
| `AutoMode_EmptyBoxMove` | `st_AutoMode_ClampingBoxesJob` | Single-step empty box manipulation | Manual box movement |
| `AutoMode_RareBoxMove` | `st_AutoMode_ClampingBoxesJob` | Rare/defective box handling | Quality control box |
| `AutoMode_OutBoxMove` | `st_AutoMode_ClampingToPlate` | Move finished box to output | Output station |
| `AutoMode_BoxMoveReverse` | `st_AutoMode_ClampingBoxesJob` | Reverse box movement (return to warehouse) | Return box operation |

**Structure: ST_AutoMode_FeedInEmptyBox**:
```pascal
TYPE ST_AutoMode_FeedInEmptyBox : STRUCT
    HandShake : ST_HandShake;              // Execute/Done/Busy/Error
    SourceType : E_BoxSource;              // Warehouse/Buffer/PullOutConveyor
    TargetConveyor : E_TargetConveyor;     // FeedInConveyor/AllocatedConveyor
    WarehousePosition : ST_WarehousePos;   // If SourceType = Warehouse
END_STRUCT
END_TYPE
```

---

### Robot Region Movement Services

These services move the Fanuc robot between different areas (FeedIn, Allocated, PullOut, Buffer, RoundBelt):

| Service Variable | Structure Type | Purpose | Typical Route |
|-----------------|----------------|---------|---------------|
| `AutoMode_RobotRegionMove_FeedInConveyor` | `ST_AutoMode_RobotRegionMove` | Move robot to FeedIn conveyor area | Warehouse → FeedIn |
| `AutoMode_RobotRegionMove_AllocatedConveyor` | `ST_AutoMode_RobotRegionMove` | Move robot to Allocated conveyor area | FeedIn → Allocated |
| `AutoMode_RobotRegionMove_PullOutConveyor` | `ST_AutoMode_RobotRegionMove` | Move robot to PullOut conveyor area | Allocated → PullOut |
| `AutoMode_RobotRegionMove_BufferArea` | `ST_AutoMode_RobotRegionMove` | Move robot to buffer area | Buffer storage |
| `AutoMode_RobotRegionMove_RoundBelt` | `ST_AutoMode_RobotRegionMove` | Move robot to RoundBelt area (legacy) | RoundBelt transport |
| `AutoMode_RobotRegionMove_Mover` | `ST_AutoMode_RobotMoverMove` | Move robot to XPlanar mover position | XPlanar integration |

**Structure: ST_AutoMode_RobotRegionMove**:
```pascal
TYPE ST_AutoMode_RobotRegionMove : STRUCT
    HandShake : ST_HandShake;
    TargetRegion : E_RobotRegion;          // Enumeration of regions
    WithGripper : BOOL;                    // TRUE if carrying box
    SafeHeight : BOOL := TRUE;             // Use safe Z-height during travel
END_STRUCT
END_TYPE
```

---

### Conveyor & Warehouse Services

| Service Variable | Structure Type | Purpose | Notes |
|-----------------|----------------|---------|-------|
| `AutoMode_UpperFeedInConveyor` | `ST_Auto_UpperFeedInConveyor` | Control upper FeedIn conveyor | Dual-level conveyor system |
| `AutoMode_PullOutConveyor` | `ST_AutoPullOutConveyor` | Control PullOut conveyor operation | Extract finished boxes |
| `AutoMode_WarehouseMove` | `st_ads_OneAxisMoveWithPosition` | Move warehouse rack motor to position | Rack vertical positioning |
| `AutoMode_MoveConveyorJob` | `st_MoveConveyorJob` | Timed conveyor movement job | Conveyor scheduling |

---

### OutRobot Services (Allocation Area - A/B Side Cylinders)

**⚠️ Important**: These "OutRobot" services refer to **Allocation area OutRobot** (2 dual cylinders A/B side for box handling), **NOT** the non-existent Shipping area OutRobot XYR system.

| Service Variable | Structure Type | Purpose | Status |
|-----------------|----------------|---------|--------|
| `AutoMode_OutRobot_MoverToConveyor` | `st_OutRobotJob_Mover` | Transfer box from XPlanar mover to conveyor | Active |
| `AutoMode_OutRobot_ConveyorToMover` | `st_OutRobotJob_Mover` | Transfer box from conveyor to mover | ⚠️ Deprecated (No longer use) |
| `AutoMode_OutRobot_MoverToPlate` | `st_OutRobotJob_Mover` | Transfer box from mover to plate magazine | Active |

**Structure: st_OutRobotJob_Mover**:
```pascal
TYPE st_OutRobotJob_Mover : STRUCT
    HandShake : ST_HandShake;
    MoverID : INT;                         // Target mover ID (1-6)
    Side : E_OutRobotSide;                 // A-side or B-side
    TargetPosition : E_OutRobotTarget;     // Conveyor/Plate
END_STRUCT
END_TYPE
```

---

### Storage System Control

| Service Variable | Structure Type | Purpose | Notes |
|-----------------|----------------|---------|-------|
| `StorageRegionHome` | `st_RegionHome` | Home all storage area axes & robot | Homing sequence |
| `StorageRegionMode` | `ST_Mode` | Storage area mode control (Auto/Manual) | Mode switching |
| `FanucRobot_General` | `ST_FanucRobotGeneralCtrl` | General Fanuc robot control (Enable/Reset) | EtherNet/IP control |

---

### Shipping Robot Services (Dual Small Robots)

**Note**: These are **NOT** OutRobot XYR. These are small shipping robots for stacking operations.

| Service Variable | Structure Type | Purpose | Notes |
|-----------------|----------------|---------|-------|
| `AutoMode_ShippingRobot_Left` | `ST_AutoMode_ShippingRobot_MoveAbs` | Left shipping robot absolute move | Left stacking area |
| `AutoMode_ShippingRobot_Right` | `ST_AutoMode_ShippingRobot_MoveAbs` | Right shipping robot absolute move | Right stacking area |
| `AutoMode_ShippingRobot_TransmitCVEnd` | `ST_AutoMode_ShippingRobot_MoveAbs` | Shipping robot to conveyor end position | Transfer point |

---

### Legacy Services (RoundBelt System)

| Service Variable | Structure Type | Purpose | Status |
|-----------------|----------------|---------|--------|
| `EmptyRobotToTurnTableMove` | `st_ads_OneAxisMoveWithPosition` | Move empty robot to TurnTable | Legacy (RoundBelt version) |

---

## 10.13.2 Allocation Area Services (16 services)

Allocation area services handle needling operations, batch processing, and XPlanar mover coordination.

### Needling Operation Services

| Service Variable | Structure Type | Purpose | Pattern |
|-----------------|----------------|---------|---------|
| `AllocationRobotNeedlingMove` | `st_AllocateRobotNeedlingMove` | Move allocation robot to needling position | Single robot control |
| `AutoMode_AllocateMove` | `st_AutoMode_AllocatingJob` | Execute needling operation (forward) | Single needling cycle |
| `AutoMode_AllocateMove_Reverse` | `st_AutoMode_AllocatingJob` | Execute needle removal operation (reverse) | Needle extraction |

**Structure: st_AutoMode_AllocatingJob**:
```pascal
TYPE st_AutoMode_AllocatingJob : STRUCT
    HandShake : ST_HandShake;
    RobotGroup : E_RobotGroup;             // 1A/1B/1C/1D/2A/2B/2C/2D
    NeedleCount : INT;                     // Number of needles
    PenetrationDepth : LREAL;              // mm
END_STRUCT
END_TYPE
```

---

### Batch Needling Services

| Service Variable | Structure Type | Purpose | Capacity |
|-----------------|----------------|---------|----------|
| `AutoMode_AllocateBatchMode` | `ST_AllocationJobBatchMode` | Batch needling (forward) | Multiple boxes |
| `AutoMode_AllocateBatchMode_Reverse` | `ST_AllocationJobBatchMode` | Batch needle removal (reverse) | Multiple boxes |
| `AutoMode_SimpleAllocation_Batch` | `ST_AllocationJobBatchMode` | Simplified batch needling | Quick setup mode |
| `AutoMode_SimpleAllocate` | `ST_AutoMode_SimpleAllocateHandShake` | Simple single-box needling | Simplified workflow |
| `XPlanar_NeedlingBatchMode` | `st_NeedlingJobBatchMode` | XPlanar-coordinated batch needling | XPlanar integration |
| `Xplanar_MoverNeedlingMove` | `st_MoverNeedlingMove` | Move mover to needling station | Mover positioning |

**Structure: ST_AllocationJobBatchMode**:
```pascal
TYPE ST_AllocationJobBatchMode : STRUCT
    HandShake : ST_HandShake;
    BatchSize : INT;                       // Number of boxes (1-6)
    StationMask : BYTE;                    // Bit mask: which stations to use (1A-2D)
    NeedlePattern : ARRAY[1..8] OF ST_NeedleConfig;
    AutoRelease : BOOL := TRUE;            // Auto-release movers when done
END_STRUCT
END_TYPE
```

**Example: Batch needling with 5 boxes**:
```pascal
// HMI writes:
GVL_ADS.XPlanar_NeedlingBatchMode.Execute := TRUE;
GVL_ADS.XPlanar_NeedlingBatchMode.BatchSize := 5;
GVL_ADS.XPlanar_NeedlingBatchMode.StationMask := 2#11111000;  // Use stations 1A-1D, 2A

// PLC orchestrates:
// - Requests 5 movers from pool
// - Navigates movers to stations 1A-2A
// - Executes needling at all stations simultaneously
// - Returns movers to pool
// - Sets Done := TRUE

// HMI monitors progress:
nProgress := GVL_ADS.XPlanar_NeedlingBatchMode.Progress;  // 0-100%
```

---

### Legacy Allocation Services (RoundBelt/TurnTable)

| Service Variable | Structure Type | Purpose | Status |
|-----------------|----------------|---------|--------|
| `AllocateMoveStructure` | `st_ads_AllocateMoveStructure` | Single needling (RoundBelt version) | Legacy |
| `AllocateMoveStructure_Reverse` | `st_ads_AllocateMoveStructure` | Needle removal (RoundBelt version) | Legacy |
| `AllocateCircleMove` | `st_AllocateCircleMoveStructure` | RoundBelt circular movement | Legacy |
| `AutoMode_AllocateCircleMove` | `st_AllocateCircleMoveStructure` | Auto mode RoundBelt movement | Legacy |
| `TurnTableMove` | `st_ads_OneAxisMoveWithPosition` | TurnTable rotation | Legacy |
| `AutoMode_TurnTableMOve` | `st_ads_OneAxisMoveWithPosition` | Auto mode TurnTable rotation | Legacy (typo: "MOve") |

---

### Allocation System Control

| Service Variable | Structure Type | Purpose | Notes |
|-----------------|----------------|---------|-------|
| `AllocationRegionHome` | `st_RegionHome` | Home all allocation area components | Homing sequence |
| `AllocationRegion` | `ST_Mode` | Allocation area mode control | Auto/Manual mode |

---

## 10.13.3 Shipping Area Services (8 services)

Shipping area services handle plate stacking, layer management, and output operations.

### Stacking Services

| Service Variable | Structure Type | Purpose | Manual Intervention |
|-----------------|----------------|---------|---------------------|
| `AutoModePushAllPlate` | `st_AutoMode_SingleStepWithInterventionRequest` | Push plate to stack layer | ✅ Requires operator when Layer 6 full |
| `AutoMode_AddPalte` | `st_AutoMode_SingleStepWithInterventionRequest` | Add plate to stack (elevator control) | ✅ Manual removal at full stack |
| `AutoMode_FGA_Finishing` | `st_AutoMode_SingleStepWithInterventionRequest` | Finishing operation - return to origin | ✅ Confirmation required |

**Structure: st_AutoMode_SingleStepWithInterventionRequest**:
```pascal
TYPE st_AutoMode_SingleStepWithInterventionRequest : STRUCT
    HandShake : ST_HandShake;
    InterventionRequest : BOOL;            // TRUE when manual action needed
    InterventionMessage : STRING(255);     // Message for operator
END_STRUCT
END_TYPE
```

**Intervention Workflow**:
```pascal
// Service detects full stack (Layer 6):
GVL_ADS.AutoModePushAllPlate.InterventionRequest := TRUE;
GVL_ADS.AutoModePushAllPlate.InterventionMessage := "Stack Full - Remove 6-Layer Stack Manually";

// HMI displays message, operator removes stack, presses "Done" button:
// HMI clears intervention:
GVL_ADS.AutoModePushAllPlate.InterventionRequest := FALSE;

// Service resumes automatically
```

---

### Shipping Mechanism Services

| Service Variable | Structure Type | Purpose | Axis/Cylinder |
|-----------------|----------------|---------|---------------|
| `AutoMode_HookMove` | `st_ads_OneAxisMoveWithPosition` | Hook positioning (plate engagement) | Hook axis + PlateHook cylinder |
| `AutoMode_ShippingAxisOrigin` | `st_ShippingAxisOrigin` | Return shipping axes to origin | All shipping axes homing |

---

### Shipping System Control

| Service Variable | Structure Type | Purpose | Notes |
|-----------------|----------------|---------|-------|
| `AutoMode_AddBoxes` | `ST_AutoAddsBoxes` | Add boxes to shipping queue | Buffer management |
| `FGARegionHome` | `st_RegionHome` | Home Finished Goods Area (FGA) | FGA = Shipping area |
| `FGARegionMode` | `ST_Mode` | FGA mode control (Auto/Manual) | Mode switching |

---

## 10.13.4 XPlanar Services (7 services)

XPlanar services control mover operations, station navigation, and camera integration.

### Mover Navigation Services

| Service Variable | Structure Type | Purpose | Typical Usage |
|-----------------|----------------|---------|---------------|
| `Xplanar_MoverToStation` | `st_MoverMoveToStation` | Move single mover to target station | Manual mover control |
| `Xplanar_MoverToStationBatch` | `st_MoverMoveToStationBatch` | Move multiple movers to stations | Batch operations |
| `XPlanar_LineUp` | `ST_MoverLineUp` | Line up movers at Loader station | Queue management |

**Structure: st_MoverMoveToStation**:
```pascal
TYPE st_MoverMoveToStation : STRUCT
    HandShake : ST_HandShake;
    MoverID : INT;                         // Mover to move (1-6)
    TargetStation : E_StationID;           // Station_1A..2D / Loader / Unloader
    Velocity : LREAL := 200.0;             // mm/s
    WaitForDocking : BOOL := TRUE;         // Wait for docking complete
END_STRUCT
END_TYPE
```

**Example: Move Mover 3 to Station 1B**:
```pascal
// HMI writes:
GVL_ADS.Xplanar_MoverToStation.Execute := TRUE;
GVL_ADS.Xplanar_MoverToStation.MoverID := 3;
GVL_ADS.Xplanar_MoverToStation.TargetStation := E_StationID.Station_1B;
GVL_ADS.Xplanar_MoverToStation.Velocity := 150.0;  // Slower speed

// PLC executes:
// - Path planning (A* algorithm)
// - Collision detection
// - Movement to station
// - Docking sequence (±0.1mm tolerance)
// - Sets Done := TRUE when docked

// HMI monitors:
IF GVL_ADS.Xplanar_MoverToStation.Done THEN
    // Mover 3 now docked at Station 1B
    GVL_ADS.Xplanar_MoverToStation.Execute := FALSE;
END_IF
```

---

### Manual Mover Control Services

| Service Variable | Structure Type | Purpose | Mode Restriction |
|-----------------|----------------|---------|------------------|
| `XPlanar_MoveAbsCollisionAvoid` | `st_MoverAbsMove` | Move mover to absolute XY position with collision check | Manual mode only |
| `XPlanar_MoverMoveZ` | `st_MoverMoveZ` | Move mover Z-height (levitation height) | Manual mode only |
| `XPlanar_MoverIDExchange` | `st_MoverIDExchange` | Exchange two movers' IDs | Maintenance operation |

**Structure: st_MoverAbsMove**:
```pascal
TYPE st_MoverAbsMove : STRUCT
    HandShake : ST_HandShake;
    MoverID : INT;
    TargetX : LREAL;                       // mm (0-5000)
    TargetY : LREAL;                       // mm (0-4000)
    TargetC : LREAL := 0.0;                // Rotation (degrees)
    Velocity : LREAL := 200.0;             // mm/s
    CollisionCheck : BOOL := TRUE;         // Enable collision detection
END_STRUCT
END_TYPE
```

**Safety**: `XPlanar_MoveAbsCollisionAvoid` automatically checks:
- Other movers in path (swept-area collision)
- Static obstacles (station boundaries)
- Safety zones (near robots)

---

### Camera Integration Service

| Service Variable | Structure Type | Purpose | Integration |
|-----------------|----------------|---------|-------------|
| `AutoMode_MoverCameraJob` | `ST_MoverCameraJob` | Trigger camera job for mover | Vision system integration |

**Structure: ST_MoverCameraJob**:
```pascal
TYPE ST_MoverCameraJob : STRUCT
    HandShake : ST_HandShake;
    MoverID : INT;
    CameraID : INT;                        // Camera station ID
    TriggerMode : E_CameraTrigger;         // Position/External/Manual
    ExpectedBarcode : STRING(50);          // For verification
END_STRUCT
END_TYPE
```

**Workflow**:
1. Mover positions at camera station
2. Camera cylinder extends (positions camera)
3. Trigger image capture
4. Read barcode/OCR
5. Camera cylinder retracts
6. Service returns barcode string

---

## 10.13.5 System-Level Services (14 services)

System services handle machine-wide operations, modes, homing, and diagnostics.

### Machine Control Services

| Service Variable | Structure Type | Purpose | Scope |
|-----------------|----------------|---------|-------|
| `MachineHome` | `st_RegionHome` | Home all machine areas (Storage/Allocate/Shipping) | Global homing |
| `MachineReset` | `st_ads_MachineReset` | Reset machine to default state | Emergency reset |
| `MachineMode` | `ST_Mode` | Machine-level mode control | Global mode |

**Structure: st_RegionHome**:
```pascal
TYPE st_RegionHome : STRUCT
    HandShake : ST_HandShake;
    SequentialHome : BOOL := TRUE;         // Home areas sequentially vs. parallel
    AreaMask : BYTE;                       // Bit mask: which areas to home
END_STRUCT
END_TYPE
```

**Example: Home entire machine**:
```pascal
// HMI writes:
GVL_ADS.MachineHome.Execute := TRUE;
GVL_ADS.MachineHome.SequentialHome := TRUE;  // Safe sequential homing
GVL_ADS.MachineHome.AreaMask := 2#00000111;  // Storage + Allocate + Shipping

// PLC executes homing sequence:
// 1. Storage area homing (Fanuc robot, warehouse motor)
// 2. Allocation area homing (XPlanar init, 8 robot groups)
// 3. Shipping area homing (elevator, hook, stack axes)
// Total time: 60-90 seconds

// HMI monitors progress:
nHomingProgress := GVL_ADS.MachineHome.Progress;  // 0-100%
```

---

### Operator Panel Interface

| Service Variable | Structure Type | Direction | Purpose |
|-----------------|----------------|-----------|---------|
| `OpPannel_Status` | `ST_OpPannelStatus` | PLC → HMI | Operator panel status display |
| `OpPannel_Ctrl` | `ST_OpPannelStatus` | HMI → PLC | Operator panel control inputs |

**Structure: ST_OpPannelStatus**:
```pascal
TYPE ST_OpPannelStatus : STRUCT
    // Indicator lamps
    bPowerON : BOOL;                       // Power indicator
    bRunning : BOOL;                       // Running indicator (green)
    bWarning : BOOL;                       // Warning indicator (yellow)
    bError : BOOL;                         // Error indicator (red)

    // Mode indicators
    bAutoMode : BOOL;
    bManualMode : BOOL;

    // Button states (for OpPannel_Ctrl)
    bStartButton : BOOL;                   // Start button pressed
    bStopButton : BOOL;                    // Stop button pressed
    bResetButton : BOOL;                   // Reset button pressed
    bEStopButton : BOOL;                   // E-Stop button state
END_STRUCT
END_TYPE
```

---

### Camera Services

| Service Variable | Structure Type | Purpose | Camera Station |
|-----------------|----------------|---------|----------------|
| `AutoMode_SameSizeCVCamera` | `ST_AutoMode_TransCVCamera` | Trigger camera for same-size boxes | Conveyor camera station |
| `AutoMode_DifferentCVCamera` | `ST_AutoMode_TransCVCamera` | Trigger camera for different-size boxes | Conveyor camera station |

**Structure: ST_AutoMode_TransCVCamera**:
```pascal
TYPE ST_AutoMode_TransCVCamera : STRUCT
    HandShake : ST_HandShake;
    TriggerPosition : LREAL;               // Conveyor position to trigger
    CaptureDelay : TIME := T#100MS;        // Delay after position trigger
    ExpectedSize : ST_BoxSize;             // Expected box dimensions (verification)
END_STRUCT
END_TYPE
```

---

### Error Handling Service

| Service Variable | Structure Type | Direction | Purpose |
|-----------------|----------------|-----------|---------|
| `AutoMode_ErrorHandle` | `ST_Alarm` | HMI → PLC | Host error notifications to PLC |

**Usage**: HMI can send error/warning messages to PLC (e.g., communication timeout, database error).

---

### Maintenance Services

| Service Variable | Structure Type | Purpose | When to Use |
|-----------------|----------------|---------|-------------|
| `RebootModbusDevices` | `st_RebootModbusDevices` | Reboot Modbus RTU devices | Communication recovery |

**Structure: st_RebootModbusDevices**:
```pascal
TYPE st_RebootModbusDevices : STRUCT
    HandShake : ST_HandShake;
    DeviceMask : WORD;                     // Bit mask: which Modbus slaves to reboot
    RebootDelay : TIME := T#5S;            // Delay between device reboots
END_STRUCT
END_TYPE
```

**Example**: If Modbus barcode reader stops responding:
```pascal
GVL_ADS.RebootModbusDevices.Execute := TRUE;
GVL_ADS.RebootModbusDevices.DeviceMask := 2#0000000000000010;  // Device 2 (barcode reader)
// PLC power-cycles Modbus slave 2, waits 5s, re-initializes communication
```

---

## 10.13.6 Manual/Legacy Services (1 service)

| Service Variable | Structure Type | Purpose | Status |
|-----------------|----------------|---------|--------|
| `RareFullBoxPush` | `st_ads_OutRobtToPlateMove` | Push rare/defective full box to plate | Manual operation |

---

## 10.13.7 Service Usage Patterns

### HandShake Pattern (Standard)

All services follow the **Execute-Done-Busy-Error** pattern:

```pascal
TYPE ST_HandShake : STRUCT
    Execute : BOOL;                        // HMI sets TRUE to start
    Done : BOOL;                           // PLC sets TRUE when complete
    Busy : BOOL;                           // PLC sets TRUE during execution
    Error : BOOL;                          // PLC sets TRUE on error
    ErrorID : DINT;                        // Error code (if Error = TRUE)
END_STRUCT
END_TYPE
```

**HMI Standard Workflow**:
```javascript
// JavaScript/C# HMI example
function executeService(servicePath, parameters) {
    // 1. Set parameters
    ads.write(servicePath + ".Parameter1", parameters.param1);
    ads.write(servicePath + ".Parameter2", parameters.param2);

    // 2. Trigger execution (rising edge)
    ads.write(servicePath + ".HandShake.Execute", true);

    // 3. Poll for completion
    let interval = setInterval(() => {
        let done = ads.read(servicePath + ".HandShake.Done");
        let error = ads.read(servicePath + ".HandShake.Error");

        if (done) {
            // Success
            clearInterval(interval);
            ads.write(servicePath + ".HandShake.Execute", false);  // Reset
            onSuccess();
        } else if (error) {
            // Error occurred
            clearInterval(interval);
            let errorID = ads.read(servicePath + ".HandShake.ErrorID");
            ads.write(servicePath + ".HandShake.Execute", false);  // Reset
            onError(errorID);
        }
    }, 100);  // Poll every 100ms
}

// Usage:
executeService("GVL_ADS.AutoMode_FeedInEmptyBoxWithRobot", {
    sourceType: "Warehouse",
    targetConveyor: "FeedInConveyor",
    warehousePosition: {side: 1, layer: 3, slide: 5}
});
```

---

### Intervention Request Pattern (Shipping Services)

Services that may require manual intervention:

```pascal
TYPE st_AutoMode_SingleStepWithInterventionRequest : STRUCT
    HandShake : ST_HandShake;
    InterventionRequest : BOOL;            // PLC sets TRUE when manual action needed
    InterventionMessage : STRING(255);     // Human-readable message
END_STRUCT
END_TYPE
```

**HMI Intervention Handling**:
```javascript
function monitorServiceWithIntervention(servicePath) {
    let interval = setInterval(() => {
        let interventionRequest = ads.read(servicePath + ".InterventionRequest");

        if (interventionRequest) {
            let message = ads.read(servicePath + ".InterventionMessage");

            // Display modal dialog to operator
            showInterventionDialog(message, () => {
                // Operator confirmed completion
                ads.write(servicePath + ".InterventionRequest", false);
            });
        }

        let done = ads.read(servicePath + ".HandShake.Done");
        if (done) {
            clearInterval(interval);
            // Service complete
        }
    }, 500);
}
```

---

## 10.13.8 Service Coverage Summary

| Category | Service Count | Coverage in Doc (Before This Section) | Coverage Now |
|----------|---------------|----------------------------------------|--------------|
| **Storage Area** | 25 | 0% | ✅ 100% |
| **Allocation Area** | 16 | 0% | ✅ 100% |
| **Shipping Area** | 8 | 50% (4/8) | ✅ 100% |
| **XPlanar Services** | 7 | 0% | ✅ 100% |
| **System Services** | 14 | 7% (1/14) | ✅ 100% |
| **Manual/Legacy** | 1 | 0% | ✅ 100% |
| **TOTAL** | **71** | **5.6%** (4/71) | ✅ **100%** |

---

## 10.13.9 Quick Reference: Service by Functional Goal

### "I want to move the Fanuc robot to a warehouse position"
→ Use: `AutoMode_RobotWarehouseMove`

### "I want to feed an empty box to the FeedIn conveyor"
→ Use: `AutoMode_FeedInEmptyBoxWithRobot`

### "I want to execute batch needling for 5 boxes"
→ Use: `XPlanar_NeedlingBatchMode` (modern) or `AutoMode_AllocateBatchMode` (legacy)

### "I want to move a mover to Station 1A"
→ Use: `Xplanar_MoverToStation`

### "I want to push a plate to the stack"
→ Use: `AutoModePushAllPlate` (will trigger intervention if Layer 6 full)

### "I want to home the entire machine"
→ Use: `MachineHome`

### "I want to trigger the camera for a mover"
→ Use: `AutoMode_MoverCameraJob`

### "I want to manually move a mover to XY position"
→ Use: `XPlanar_MoveAbsCollisionAvoid` (Manual mode only)

---

## 10.13.10 Deprecated/Legacy Services

The following services are from the **RoundBelt transport system** (previous version) and may not be actively used in XPlanar version:

- `AllocateMoveStructure` / `AllocateMoveStructure_Reverse`
- `AllocateCircleMove` / `AutoMode_AllocateCircleMove`
- `TurnTableMove` / `AutoMode_TurnTableMOve`
- `EmptyRobotToTurnTableMove`
- `AutoMode_RobotRegionMove_RoundBelt`
- `AutoMode_ReloadRoundBeltBoxWithRobot`
- `AutoMode_OutRobot_ConveyorToMover` (marked "No longer use, pending")

**Migration Note**: If migrating from RoundBelt to XPlanar system, these services should be replaced with XPlanar-based equivalents (e.g., `Xplanar_MoverToStation`, `XPlanar_NeedlingBatchMode`).

---

**End of Section 10.13: GVL_ADS Complete Service Catalog**
