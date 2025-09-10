# Copilot Instructions for LagCompensation SourceMod Plugin

## Repository Overview
This repository contains a **LagCompensation** SourceMod plugin that provides lag compensation for physics entities in Source engine games (CS:GO and CS:Source). The plugin uses advanced SourceMod features including DHooks, SDKCalls, and GameData to compensate for player latency when interacting with physics objects.

## Technical Environment
- **Language**: SourcePawn (SourceMod scripting language)
- **Platform**: SourceMod 1.11+ (as specified in sourceknight.yaml)
- **Build Tool**: SourceKnight (Python-based SourceMod build system)
- **CI/CD**: GitHub Actions with automated building, testing, and releases
- **Target Games**: Counter-Strike: Global Offensive (CS:GO) and Counter-Strike: Source (CS:S)

## Project Structure
```
/
├── .github/workflows/ci.yml          # CI/CD pipeline configuration
├── addons/sourcemod/
│   ├── scripting/
│   │   └── LagCompensation.sp        # Main plugin source code (~1600 lines)
│   └── gamedata/
│       └── LagCompensation.games.txt # Game signatures and offsets for CS:GO/CS:S
├── sourceknight.yaml                 # Build configuration and dependencies
├── .gitignore                        # Git ignore patterns (excludes .smx, build/, etc.)
└── .github/copilot-instructions.md   # This file
```

## Dependencies
The plugin requires several SourceMod extensions and includes:
- **SourceMod**: Core framework (v1.11.0-git6934 as specified)
- **MultiColors**: Color formatting for chat messages
- **PhysHooks**: Physics simulation hooks extension
- **DHooks**: Dynamic hooks for game functions
- **SDKTools/SDKHooks**: Core SourceMod libraries
- **ClientPrefs**: Client preference storage

## Code Style & Standards

### SourcePawn-Specific Guidelines
- Use `#pragma semicolon 1` and `#pragma newdecls required` (already present)
- Indentation: **4 spaces** (represented as tabs in editor)
- Variable naming:
  - **camelCase** for local variables and function parameters
  - **PascalCase** for function names
  - **g_** prefix for global variables (e.g., `g_bLateLoad`, `g_aEntityLagData`)
  - **UPPERCASE** for constants and defines

### Memory Management
- Use `delete` directly without null checks (SourceMod handles this)
- **Never use `.Clear()`** on StringMap/ArrayList - causes memory leaks
- Use `delete` and recreate containers instead of clearing
- Properly handle Handle cleanup with `delete` operator
- Example pattern in codebase:
  ```sourcepawn
  delete hGameData;  // Direct delete, no null check needed
  ```

### Performance Considerations
- Minimize timer usage where possible
- Cache expensive operations (see `g_aLerpTicks` array)
- Use bitwise operations for entity flags (see `SetBit`, `ClearBit`, `CheckBit` macros)
- Consider server tick rate impact in frequently called functions
- Use efficient data structures (arrays over StringMaps where appropriate)

## Build System

### SourceKnight Configuration
The build system uses SourceKnight with configuration in `sourceknight.yaml`:
- **Dependencies**: Automatically downloads SourceMod, MultiColors, and PhysHooks
- **Build target**: `LagCompensation.sp` → `LagCompensation.smx`
- **Output directory**: `/addons/sourcemod/plugins`

### Build Commands
- **Local build**: `sourceknight build` (requires Python and SourceKnight package)
- **CI build**: Automated via GitHub Actions using `maxime1907/action-sourceknight@v1`

### Build Verification
Always verify builds by checking:
1. No compilation errors/warnings
2. Plugin loads without errors in SourceMod
3. No memory leaks (use SourceMod's built-in profiler)
4. GameData signatures are valid for target games

## GameData Management

### File: `addons/sourcemod/gamedata/LagCompensation.games.txt`
Contains critical game signatures and offsets for:
- **CS:GO and CS:Source**: Different signature patterns for each game
- **Key functions**: UTIL_Remove, RestartRound, CalcAbsolutePosition, etc.
- **Offsets**: Memory offsets for entity properties

### Signature Updates
When updating signatures:
1. Test on both CS:GO and CS:Source if changes affect both
2. Use hex patterns for Windows, symbol names for Linux where possible
3. Validate with game updates - signatures may break with engine updates

## DHooks and SDKCalls Usage

### Critical Patterns in Codebase
The plugin uses extensive DHooks detours and SDKCalls:

```sourcepawn
// SDKCall pattern
Handle g_hCalcAbsolutePosition;
StartPrepSDKCall(SDKCall_Entity);
PrepSDKCall_SetFromConf(hGameData, SDKConf_Signature, "CalcAbsolutePosition");
g_hCalcAbsolutePosition = EndPrepSDKCall();

// DHooks detour pattern  
Handle g_hUTIL_Remove;
g_hUTIL_Remove = DHookCreateFromConf(hGameData, "UTIL_Remove");
DHookEnableDetour(g_hUTIL_Remove, false, Detour_UTIL_Remove);
```

### Error Handling
Always check return values:
- SDKCall preparations: Check if `EndPrepSDKCall()` returns valid handle
- DHooks: Verify `DHookCreateFromConf()` success
- Use `SetFailState()` for critical failures that should prevent plugin loading

## Entity Lag Compensation Implementation

### Core Data Structures
```sourcepawn
// Maximum entities and records
#define MAX_ENTITIES 128
#define MAX_RECORDS 64

// Entity lag data tracking
LagRecord g_aaLagRecords[MAX_ENTITIES][MAX_RECORDS];
EntityLagData g_aEntityLagData[MAX_ENTITIES];
```

### Key Implementation Details
- **Entity tracking**: Limited to 128 entities max for performance
- **Record history**: Up to 64 lag records per entity
- **Bitwise flags**: Used extensively for entity state management
- **Coordinate frames**: 14-byte coordinate frame handling

## Client Preferences (Cookies)
The plugin supports client-side settings:
- **g_hCookie_LagCompSettings**: Stores player lag compensation preferences  
- **Settings tracked**: Disable lag comp, laser compensation, messages
- Handle cookie loading/saving properly in `OnClientCookiesCached`

## Translation Support
- Load translations with: `LoadTranslations("common.phrases")`
- Use MultiColors plugin for colored chat messages
- Prefix: `{green}[LagCompensation]{default}`

## Testing & Validation

### Required Tests
1. **Plugin loading**: Verify no errors on plugin load
2. **Entity compensation**: Test lag compensation accuracy with various ping levels
3. **Performance**: Monitor server performance with plugin active
4. **Game compatibility**: Test on both CS:GO and CS:Source
5. **Memory usage**: Check for memory leaks during extended play

### Debug Features
- ConVars for testing: `sm_lagcomp_entityclass`, `sm_lagcomp_checkuserping`
- Client command support for preference testing
- Built-in performance monitoring

## Common Patterns & Best Practices

### Entity Management
```sourcepawn
// Safe entity validation
if (IsValidEntity(entity) && entity > MaxClients) {
    // Entity operations
}

// Bit manipulation for flags
SetBit(g_aLagCompensated, entity);
ClearBit(g_aBlockTriggerTouchPlayers, entity);
if (CheckBit(g_aBlacklisted, entity)) {
    // Handle blacklisted entity
}
```

### Error Handling Pattern
```sourcepawn
Handle hGameData = LoadGameConfigFile("LagCompensation.games");
if (!hGameData) {
    SetFailState("Failed to load LagCompensation gamedata.");
}
// Use gamedata...
delete hGameData;  // Always cleanup
```

## Deployment & Release

### Release Process (Automated)
1. **Push to main/master**: Triggers CI build
2. **Successful build**: Creates artifacts  
3. **Release creation**: Automated with version tagging
4. **Package contents**: Plugin (.smx), gamedata, and documentation

### Manual Deployment
1. Place `LagCompensation.smx` in `addons/sourcemod/plugins/`
2. Place `LagCompensation.games.txt` in `addons/sourcemod/gamedata/`
3. Restart server or use `sm plugins reload LagCompensation`

## Performance & Optimization

### Critical Performance Areas
- **Entity enumeration**: Limited to 128 entities max
- **Lag record management**: Circular buffer for efficiency  
- **Frequent function calls**: `GameFrame` hooks optimized
- **Memory allocation**: Pre-allocated arrays vs dynamic allocation

### Monitoring
- Use SourceMod's `sm_profiler` for performance analysis
- Monitor server tick rate impact
- Check memory usage trends during gameplay

## Troubleshooting

### Common Issues
1. **GameData outdated**: Update signatures for new game builds
2. **Extension missing**: Ensure PhysHooks extension is loaded
3. **Performance issues**: Check entity count limits and record history size
4. **Compilation errors**: Verify all include files are present

### Debug Commands
- `sm_lagcomp_entityclass`: Control which entities to compensate
- `sm_lagcomp_checkuserping`: Enable/disable ping checking
- Server console: Monitor for lag compensation activity

## Contributing Guidelines
When modifying this plugin:
1. **Minimal changes**: Make surgical modifications only
2. **Test thoroughly**: Verify on both supported games
3. **Performance impact**: Monitor server performance
4. **Documentation**: Update relevant comments for complex logic
5. **GameData**: Keep signatures current with game updates