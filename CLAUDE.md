# CLAUDE.md

## Project Overview

**TwinCAT 3 PLC project** implementing PackML framework for XPlanar-based automated manufacturing system.

**Key Files**:
- Solution: `CinPhown_PML_XPlanar/CinPhown_PackML.sln`
- PLC Project: `CinPhown_PML_XPlanar/CinPhown_PackML/DAS_CoreSys/DAS_CoreSys.plcproj`
- TwinCAT Project: `CinPhown_PML_XPlanar/CinPhown_PackML/CinPhown_PackML.tsproj`

## Build System

**Prerequisites**: TwinCAT 3 XAE 3.1.4024.0+, Visual Studio, XPlanar license

**Build**: Open `CinPhown_PackML.sln` in TwinCAT XAE → Build Solution
**Primary Target**: Debug|TwinCAT RT (x64)

**Task Configuration**:
| Task | Priority | Cycle | Purpose |
|------|----------|-------|---------|
| CoreSys | 15 | 10ms | Main logic (FB_MachineControl) |
| MotionSync | 10 | 4ms | XPlanar mover sync |
| ModbusTask | 20 | 100ms | Modbus RTU comm |

**Common Issue**: Task priority conflicts → Check `DAS_CoreSys/ModbusTask.TcTTO`

## Architecture

### Component Hierarchy

```
FB_MachineControl (Main orchestrator, extends FB_ModeBase_V2)
├── FB_HmiHandShake (HMI ↔ PLC via ADS)
├── FB_ErrorHandler (Alarm management)
├── FB_MachineHoming_v2 (3-zone homing)
├── FB_StorageArea (Fanuc robot, warehouse)
├── FB_AllocateAreaWithXPlanar (XPlanar + 8 robot groups)
│   └── FB_XPlanar (mover management, path planning, collision avoidance)
└── FB_ShippingAreaWithXPlanar (Stacking & AGV)
```

**PackML Base**: `Library/00_PackML/FB_ModeBase_V2` (use for new development)

**Data Flow**: HMI → `GVL_ADS.ADS_Command` → FB_HmiHandShake → FB_MachineControl → Unit FBs → `GVL_ADS.ADS_Status` → HMI

### Production Areas

**1. Storage Area** (`Library/20_MachineUnits/21_StorageArea/`):
- FB_FanucRobot_Basic (EtherNet/IP), FB_WareHouse, RackMotor
- Key Services: FB_StorageHoming, FB_FeedInEmptyBoxWithRobot
- Signals: `GVL_Input.bStorage_*`, `GVL_EIP.FanucRobot.*`

**2. Allocation Area** (`Library/20_MachineUnits/22_AllocateArea/`):
- **FB_XPlanar**: Mover management, A* pathfinding, collision avoidance
- **FB_XplanarInitial**: Must complete before any mover movement
- **FB_Station**: 8 workstations (1A/1B/1C/1D, 2A/2B/2C/2D)
- **Mover Lifecycle**: Loader → PathPlanning → Docking → Robot needling → Unloader → Release
- 8 Robot Groups (dual-head): Vertical axis + gripper sensors
- Signals: `bXplanar_Loader/Unloader_*Sensor`, `bXplanar_Station{1A}_DockedSensor`

**3. Shipping Area** (`Library/20_MachineUnits/23_ShippingArea/`):
- ShippingElevator, ShippingHook, ShippingStack (5 layers), OutRobot (XYR 3-axis)
- Key Services: FB_ShippingHoming_XPlanar, FB_ShippingPushToStack
- Flow: Unloader → GoodsArea buffer → Elevator → Hook → Stack → OutRobot → AGV

### XPlanar Critical Rules

1. **Never move mover until `FB_XplanarInitial.Done = TRUE`**
2. **Docking**: ±0.1mm tolerance, wait for `bXplanar_{StationName}_DockedSensor` before robot movement
3. **Collision avoidance**: `FB_CheckMoverPathSafe` checks swept area vs. other movers + obstacles
4. **Queue**: `FB_MoverLineUp_v2` maintains FIFO at Loader station

**Common Issues**:
- Docking timeout → Check station position calibration
- Path stuck → Clear obstacle map via `bClearObstacleMap`
- Mover lost → Use `FB_RecoveryErrorMover`

## Global Variables & I/O

**Key GVLs**:
- `GVL_Input/Output`: I/O points (`Library/10_Machine/11_IOData/`)
- `GVL_EIP`: Fanuc robot EtherNet/IP
- `GVL_ADS`: HMI ↔ PLC interface (`Library/10_Machine/18_HMI_Handshake/`)
- `GVL_Machine`: Setup parameters (Local/Remote via `E_DataFrom`)

**Critical Signals**:
- Safety: `bSafety_Door[1..20]`, `Machine_Estop`, `Allocate_EStop`, `Storage_EStop`
- XPlanar: `bXplanar_Loader/Unloader_*Sensor`, `bXplanar_Station{1A}_DockedSensor`
- Robots: `bAllocateRobot_{1A}_Vert_{High/Low}Sensor`, `bAllocateRobot_{1A}_GripperSensor`
- Shipping: `bStackCheck_Layer1~5`, `bGoodsArea_{First/Fifth}Detect_Sensor`

## HMI/ADS Interface

**Commands** (HMI → PLC): `GVL_ADS.ADS_Command.{PackML, Storage, Allocate, Shipping}.ServiceControl`
**Status** (PLC → HMI): `GVL_ADS.ADS_Status.{CurrentState, CurrentMode, Alarms[1..50], *ServiceStatus}`

**Service Pattern**: Execute/ServiceID/Parameters/Abort → Busy/Done/Error/ErrorID/Progress

## Error Handling

**FB_ErrorHandler** (`Library/10_Machine/13_ErrorHandling/`):
- Input: `AlarmList[1..50]`, Output: `HMI_AlarmMonitorList[0..999]`
- Alarm: ID, Trigger, Severity (Critical→Abort, Warning→Hold), Message, Category

**Error Categories**:
1. Safety (Critical→Abort): Door open, E-Stop, light curtain
2. Motion (Critical→Abort): Axis limits, motor overload, encoder error
3. XPlanar (Critical/Warning): Docking failure (retry 3x→Abort), collision, mover lost
4. Sensor-IO (Warning→Hold): Timeout, sensor mismatch
5. Communication (Warning→Hold): Modbus timeout, EIP lost
6. Process (Warning→Hold): Quality failure, recipe mismatch

**Reset**: 1.5s delay confirmation, checks alarm sources cleared before removing from list

**Resume Requirements**:
- All alarms cleared, safety doors closed, no axis errors, XPlanar movers healthy
- Storage robot at safe position, no movers mid-docking, elevator/hook safe
- **Common failures**: Use FB_RecoveryErrorMover, manually jog robot, check NeedlingBatchMode.CurrentStep

## Configuration & Parameters

**Parameters**: `GVL_Machine.{RemoteData, LocalData}` (selector: `FB_MachineControl.DataDirect`)

**Key Adjustments**:
1. **Axis Motion** (`ST_AxisPara_Auto`): Velocity, Acceleration, Jerk, Position
   - Access: `GVL_Machine.RemoteData.Axes.{AxisName}.Auto.*`
   - Tuning: Too fast→reduce Accel/Jerk 20-30%, Timeout→increase Velocity, Overshoot→reduce Velocity

2. **Timeouts** (embedded in service FBs): Search `.TcPOU` for `TIME :=` or `TON(PT := T#`
   - Examples: FB_NeedlingBatchMode_v2 (T#10S docking, T#15S robot), FB_ShippingPushToStack (T#8S elevator)

3. **Coordinates**: `GVL_Machine.RemoteData.AllocateRobotCoordinates[1..8]` (PickPosition, WorkPosition, SafePosition)

## Debugging Guide

**Call Chains**:
1. **Start**: HMI → `GVL_ADS.ADS_Command.PackML.SC_Start` → FB_HmiHandShake → FB_MachineControl.SUPER^ → M_Starting (check homed/alarms/safety) → M_Execute → Unit FBs
2. **Needling**: FB_NeedlingBatchMode_v2 → State 0 (RequestMover) → 10 (PathPlanning) → 20 (Docking+wait sensor) → 30 (Robot needling) → 40 (Undock+Release) → 50 (Done)
3. **Error→Resume**: Error → FB_ErrorHandler.AlarmList → Severity→Abort/Hold → Reset (1.5s delay, check sources) → Resume (check alarms/safety/positions)

**Fault Diagnosis**:
1. Check `GVL_ADS.ADS_Status.Alarms[1..50]` (ID, Message, Severity, Category)
2. Identify unit: `GVL_ADS.ADS_Status.{Storage/Allocate/Shipping}.ServiceStatus.Error`
3. Inspect state: `FB_NeedlingBatchMode_v2.CurrentStep`, `FB_XPlanar.MoverList[].State/ErrorID`
4. Verify I/O: `GVL_HMI.IOMonitor.Input/Output` (safety doors, XPlanar sensors, robot positions, stack layers)

**XPlanar Troubleshooting**:
- **Docking failure**: Check FB_Station[n].ActualPosition vs SetPosition → Recalibrate; Check MoverList[n].ErrorID
- **Mover lost**: Use FB_RecoveryErrorMover OR HMI "XPlanar Reset" (aborts all work)
- **Path stuck**: Clear obstacle map (bClearObstacleMap := TRUE), check ObstacleMap/StartPos/EndPos valid

**Resume Checklist**:
1. Clear alarm source → Reset (1.5s) → Alarms disappear
2. Safety OK: 20 doors closed, no E-Stop
3. Positions safe: FanucRobot.AtHome, XPlanar movers not in ErrorState, Shipping axes homed
4. Check service FB states (CurrentStep)
5. Resume → If fails: Manual mode → Jog to home → Auto mode → Start fresh

## Development Guidelines

**vs RoundBelt Version**: Transport (RoundBelt→XPlanar), Axes (TurnTable→Shipping), Robots (1→9), Safety (14→20 doors), Shipping (Conveyor→Stacking), Homing (2→3 zones)
**Reusable**: FB_ModeBase_V2, FB_ErrorHandler, Storage FBs
**DO NOT reuse**: FB_RoundBeltService, FB_TurnTableService, FB_Allocate_X/Y, Shipping conveyor FBs

**Service FB Template**: `Library/.../02_ServiceTemplate/FB_ServicesTemplate.TcPOU`
- Structure: EXTENDS FB_ActionBasic, I/O: Execute/Abort/Done/Busy/Error/ErrorID, State machine with timeout protection

**PackML State Actions**:
- M_Resetting: Clear errors, reset service FBs
- M_Starting: Check preconditions (homed/alarms/safety), don't start motion
- M_Execute: Main production logic (state-based service sequencing)
- M_Holding: Save state, abort services (don't reset)
- M_Unholding: Restore state, validate safe to continue
- M_Aborting: Stop all motion, safe all actuators, release XPlanar movers

## Common Tasks

**Adding Sensor**:
1. Add to `GVL_Input.TcGVL`: `bSensorName : BOOL; // AT %IX...`
2. Map in TwinCAT I/O: `.tsproj` → I/O → Devices → Link to GVL_Input
3. Use in service FB with timeout protection
4. Update HMI mapping in FB_HmiHandShake (if needed)

**Adjusting Timeouts**:
- Quick: Change hardcoded value (e.g., `PT := T#20S`)
- Proper: Add to `ST_SetupPara`, use `CurrentData.{ParamName}`, make HMI-editable

**Stuck in Starting State**:
- Check: `FB_MachineControl.{xError, MachineHomed}`, `fb{Storage/Allocate/Shipping}Area.xError`
- Check homing: `fbMachineHoming.{xBusy, xDone}`

## Technology Stack

**Platform**: TwinCAT 3 XAE 3.1.4024.0+, TwinCAT RT (x64)
**Language**: Structured Text (ST) 90%, IEC 61131-3
**Libraries**: Tc2_MC2 (Motion), Tc3_XPlanar (**LICENSE REQUIRED**), Tc2_System, PackML (custom `Library/00_PackML/`)
**Protocols**: ADS (HMI↔PLC), EtherNet/IP (Fanuc robot), Modbus RTU (sensors), EtherCAT (I/O)

## Documentation

**Main Docs**: `docs/PackML_XPlanar.zh-TW.md` (Chinese, detailed) / `.en.md` (English)
- §1: Build/architecture, §2: PackML states, §3: I/O, §4: Units, §7: Error codes (alarm dictionary), §8: PLC↔HMI mapping, §9: XPlanar details

**Handover**: `交接文件/CinPhown_PackML/專案描述.md`, `CinPhown_PackML 專案.md`

**Version**: 3.1 (Git: 619b7aa), Breaking from v2.0: RoundBelt→XPlanar (DO NOT in-place upgrade)

## Claude Code AI Guidelines

**Rules**:
- Use Structured Text (ST) only, not Python/JavaScript/Ladder
- Never modify FB_ModeBase_V2 without explicit request
- For XPlanar changes: Read `docs/PackML_XPlanar.zh-TW.md` §9 first
- Parameters: Use `GVL_Machine.RemoteData`, not hardcoded values
- Errors: Add to `FB_ErrorHandler.AlarmList`, not custom flags
- Service FBs: Follow `FB_ServicesTemplate.TcPOU` pattern (state machine)
- PLC is cycle-based: Use state machines, not async/await
- XPlanar movers: Managed resources with complex state, not simple axes

**Bug Reports**:
1. Ask: Alarm ID/message (`GVL_ADS.ADS_Status.Alarms`), which unit, PackML state
2. Check: Service FB state (CurrentStep/State variable)
3. Suggest: Watch variables in TwinCAT Online View
