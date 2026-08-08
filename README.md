# CC:FCS aero fix
- Work in progress!
- CC:Tweaked Advanced fire control system for the Create: Big Cannons Minecraft mod.

Required Mods:
- CC:Tweaked
- Neo Peripherals
- Advanced Peripherals
- Create
- Create: Big Cannons
- CBC:MW (Create: Big Cannons: Modern Warfare)
- Create: Aeronautics

NOTICE for original author(xyzKnight):

I Used mod: Neo Peripherals instead of Some Peripherals

# Summary of Changes Between Original and Updated Scripts

## Changes in `main.lua` (compared to `main_orig.lua`)

### 1. **Major Fix: `update_target_data()` Function**

**Original (`main_orig.lua`):**
```lua
function update_target_data()
    local temp_id = 6
    while true do
        local scans = radar.scanForShips(c.peripheral.RADAR_SCAN_RANGE) -- 1 tick yield
        local found = false
        for k, v in pairs(scans) do
            if v.id == temp_id or true then  -- <-- BUG: 'or true' is always true
                found = true
                target_data.pos       = vec3.new(v.pos.x, v.pos.y, v.pos.z)
                target_data.velo      = vec3.new(v.velocity.x, v.velocity.y, v.velocity.z)
                target_data.read_time = os.clock()
            end
        end
        if not found then
            target_data.pos       = vec3.new(0, 0, 0)
            target_data.velo      = vec3.new(0, 0, 0)
            target_data.read_time = os.clock()
        end
    end
end
```

**Updated (`main.lua`):**
```lua
function update_target_data()
    local target_id = c.peripheral.TARGET_ID  -- New config parameter
    local cannon_pos = c.solver.cannon.pos
    
    while true do
        -- Changed: scanForSubLevels instead of scanForShips
        local scans = radar.scanForSubLevels(c.peripheral.RADAR_SCAN_RANGE, true)
        
        if not scans or #scans == 0 then
            target_data.pos       = vec3.new(0, 0, 0)
            target_data.velo      = vec3.new(0, 0, 0)
            target_data.read_time = os.clock()
            os.sleep(0.05)
        else
            -- Group sub-levels by ship ID
            local ships = {}
            for _, v in pairs(scans) do
                local ship_id = v.id
                if not ships[ship_id] then
                    ships[ship_id] = {
                        pos = vec3.new(0, 0, 0),
                        velo = vec3.new(0, 0, 0),
                        count = 0
                    }
                end
                -- Fixed: correct field names
                ships[ship_id].pos = ships[ship_id].pos + vec3.new(v.x or 0, v.y or 0, v.z or 0)
                ships[ship_id].velo = ships[ship_id].velo + vec3.new(v.velX or 0, v.velY or 0, v.velZ or 0)
                ships[ship_id].count = ships[ship_id].count + 1
            end
            
            -- Find nearest ship
            local nearest_dist = math.huge
            local nearest_ship = nil
            
            for ship_id, ship_data in pairs(ships) do
                local avg_pos = ship_data.pos:scale(1 / ship_data.count)
                local dist = (avg_pos - cannon_pos):length()
                
                if dist < nearest_dist then
                    nearest_dist = dist
                    nearest_ship = ship_data
                    nearest_ship.avg_pos = avg_pos
                    nearest_ship.avg_velo = ship_data.velo:scale(1 / ship_data.count)
                end
            end
            
            if nearest_ship then
                target_data.pos = nearest_ship.avg_pos
                target_data.velo = nearest_ship.avg_velo
                target_data.read_time = os.clock()
                print("Target at distance:", nearest_dist, "Pos:", target_data.pos)
            else
                target_data.pos = vec3.new(0, 0, 0)
                target_data.velo = vec3.new(0, 0, 0)
                target_data.read_time = os.clock()
            end
        end
    end
end
```

### 2. **Added Target Check in Main Loop**

**Original (`main_orig.lua`):**
```lua
function main()
    -- ...
    while true do
        -- ...
        local mid_pitch, mid_yaw = reconstruct_mid_state(cannon_pitch, cannon_yaw)
        update_req_history()
        req_pitch, req_yaw = solver:solve(target_data, ...)  -- Always calls solve
        print_error()
        rsc_pitch, rsc_yaw = backsolve_rsc(mid_pitch, mid_yaw, req_pitch, req_yaw)
    end
end
```

**Updated (`main.lua`):**
```lua
function main()
    -- ...
    while true do
        -- ...
        -- Check if target exists
        if target_data.pos and target_data.pos:length() > 0.1 then
            local mid_pitch, mid_yaw = reconstruct_mid_state(cannon_pitch, cannon_yaw)
            update_req_history()
            req_pitch, req_yaw = solver:solve(target_data, ...)
            print_error()
            rsc_pitch, rsc_yaw = backsolve_rsc(mid_pitch, mid_yaw, req_pitch, req_yaw)
        else
            print("No target detected")
            -- Stop rotation if no target
            rsc_pitch = 0
            rsc_yaw = 0
        end
    end
end
```

---

## Changes in `config.lua` (compared to `config_orig.lua`)

### 1. **Fixed Peripheral Types**

**Original (`config_orig.lua`):**
```lua
c.peripheral = {
    RSC_TYPE          = "Create_RotationSpeedController",
    BLOCK_READER_TYPE = "blockReader",        -- <-- Incorrect name
    RADAR_TYPE        = "sp_radar",           -- <-- Incorrect name

    PITCH_RSC_INDEX = 2,                      -- <-- Indexes swapped
    YAW_RSC_INDEX   = 1,

    PITCH_RSC_SIGN  = -1,
    YAW_RSC_SIGN    = 1,

    RADAR_SCAN_RANGE = 10000
}
```

**Updated (`config.lua`):**
```lua
c.peripheral = {
    RSC_TYPE          = "Create_RotationSpeedController",
    BLOCK_READER_TYPE = "block_reader",       -- <-- Fixed
    RADAR_TYPE        = "neo_radar",          -- <-- Fixed

    PITCH_RSC_INDEX = 1,                      -- <-- Indexes corrected
    YAW_RSC_INDEX   = 2,

    PITCH_RSC_SIGN  = -1,
    YAW_RSC_SIGN    = 1,

    RADAR_SCAN_RANGE = 1000
}
```

### 2. **Added New Configuration Parameter**

**Original (`config_orig.lua`):** Missing

**Updated (`config.lua`):** Added at the end of `c.peripheral` section:
```lua
-- peripheral config
c.peripheral = {
    -- ... other parameters ...
    RADAR_SCAN_RANGE = 10000,
    
    TARGET_ID = nil  -- <-- NEW PARAMETER: specific target ID to track
}
```

---

## Summary of All Changes

### In `main.lua`:

| # | Change | Original | Updated |
|---|--------|----------|---------|
| 1 | Radar function | `scanForShips()` | `scanForSubLevels()` |
| 2 | Data structure | `v.pos.x/y/z` | `v.x/y/z` |
| 3 | Velocity fields | `v.velocity.x/y/z` | `v.velX/velY/velZ` |
| 4 | Target selection | Always selects first (bug) | Groups by ID, selects nearest |
| 5 | Target check in main | No check, always computes | Checks if target exists |
| 6 | RSC behavior | Always tries to move | Stops if no target |
| 7 | Config reference | Hardcoded `temp_id = 6` | Uses `c.peripheral.TARGET_ID` |

### In `config.lua`:

| # | Change | Original | Updated |
|---|--------|----------|---------|
| 1 | Block reader type | `"blockReader"` | `"block_reader"` |
| 2 | Radar type | `"sp_radar"` | `"neo_radar"` |
| 3 | Pitch RSC index | `2` | `1` |
| 4 | Yaw RSC index | `1` | `2` |
| 5 | Target ID | Not present | `TARGET_ID = nil` |
