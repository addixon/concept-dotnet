# ErgNet — Agent Instructions

This document provides context for AI coding agents (GitHub Copilot, Cursor, Cline, Windsurf, Aider, etc.) working on the ErgNet repository. Use these instructions to produce changes that are consistent with the project's conventions, architecture, and quality standards.

## Project Overview

**ErgNet** is a .NET library for communicating with Concept2 Performance Monitors (PM3/PM4/PM5) over USB, Bluetooth Low Energy, and ANT+. It is published as the `ErgNet` NuGet package under the MIT license.

- **Target framework**: `net10.0`
- **Solution file**: `ErgNet.slnx`
- **Root namespace**: `ErgNet`
- **NuGet Package ID**: `ErgNet`

## Build, Test, and Pack

```bash
# Restore
dotnet restore

# Build
dotnet build --configuration Release

# Test
dotnet test tests/ErgNet.Tests/ErgNet.Tests.csproj --logger "console;verbosity=detailed"

# Pack
dotnet pack --configuration Release --output ./nupkg
```

All warnings are treated as errors (`TreatWarningsAsErrors` in `Directory.Build.props`). The build must produce zero warnings.

## Repository Structure

```
ErgNet/
├── src/ErgNet/                        # Main library project
│   ├── IPerformanceMonitor.cs         # Primary public interface
│   ├── PerformanceMonitor.cs          # Main implementation
│   ├── PerformanceMonitorDiscovery.cs # Device discovery helpers
│   ├── Models/                        # Data models (RowingData, StrokeData, DeviceInfo, enums)
│   ├── Protocol/
│   │   ├── Ant/                       # ANT+ FE-C data page parsing
│   │   ├── Bluetooth/                 # BLE GATT notification parsing
│   │   └── Csafe/                     # CSAFE frame builder, parser, command registry
│   └── Transport/                     # Transport layer interfaces and implementations
├── tests/ErgNet.Tests/                # xUnit test project
├── .github/workflows/                 # CI/CD pipelines
├── Directory.Build.props              # Shared MSBuild properties
├── GitVersion.yml                     # Automatic versioning config
└── ErgNet.slnx                        # Solution file
```

## Naming Conventions

- **`ErgNet`**: The library/package name. Used for namespaces, assembly name, and NuGet package.
- **`Concept2`**: The hardware manufacturer. Used only when referring to Concept2 hardware (e.g., `IsConcept2Device`, `ManufacturerName: "Concept2"`, "Concept2 Performance Monitor" in comments).
- Never describe Concept2 hardware as "ErgNet hardware" — ErgNet is the software library, Concept2 is the manufacturer.

## Coding Conventions

### Style

- File-scoped namespaces: `namespace ErgNet;`
- Use `var` when the type is apparent
- Expression-bodied members for single-line properties and methods
- 4-space indentation, LF line endings
- Follow `.editorconfig` settings
- Sort `using` directives with `System` namespaces first

### Patterns

- **Async/await everywhere**: All I/O methods are async with `CancellationToken` support
- **`IAsyncEnumerable<T>`**: Used for data streaming with `[EnumeratorCancellation]` attribute
- **`SemaphoreSlim`**: Guards concurrent transport access in `PerformanceMonitor`
- **`Channel<T>`**: Merges multiple BLE notification streams
- **Transport abstraction**: Users implement `IHidDevice`, `IBleDevice`, or `IAntDevice`; the library wraps them in `UsbTransport`, `BluetoothTransport`, or `AntTransport`
- **`ObjectDisposedException.ThrowIf`**: Used at the start of public methods for disposed-check
- **`ArgumentNullException.ThrowIfNull`**: Used for null parameter validation

### XML Documentation

- All public types and members must have XML doc comments
- Use `<inheritdoc />` on concrete implementations of interface methods
- Include `<param>`, `<returns>`, and `<remarks>` tags where helpful

### Error Handling

- Throw `ArgumentException` for invalid configurations (see `WorkoutConfiguration.Validate()`)
- Throw `NotSupportedException` for operations unsupported by a transport (e.g., CSAFE over ANT+)
- Throw `InvalidOperationException` for state errors (e.g., missing transport)
- Use `CsafeFrameException` for CSAFE protocol-level errors

## Testing Conventions

- **Framework**: xUnit 2.9.3
- **Naming**: `MethodName_Scenario_ExpectedBehavior` (e.g., `ParseRowerData_TooShort_Throws`)
- **Fakes over mocks**: Use hand-written fake implementations (`FakeUsbTransport`, `FakeBleTransport`, `FakeAntTransport`, `FakeAntDevice`) instead of mocking frameworks
- **Global using**: `using Xunit;` is provided via the test `.csproj` — do not add it to test files
- **Protocol tests**: Include byte-level comments explaining the data structure being tested
- Test file location mirrors source file location (e.g., `Protocol/Csafe/` → `Protocol/Csafe/`)

## Architecture

### Layer Overview

```
┌────────────────────────────────────────────┐
│         IPerformanceMonitor API            │  ← Unified public API
├────────────────────────────────────────────┤
│         PerformanceMonitor                 │  ← Orchestrator
├──────────┬─────────────┬───────────────────┤
│  CSAFE   │   BLE       │   ANT+           │  ← Protocol parsers
│  Builder │   Parser    │   Parser          │
│  Parser  │             │                   │
│  Registry│             │                   │
├──────────┴─────────────┴───────────────────┤
│    ITransport / IBluetoothTransport /      │  ← Transport abstraction
│    IAntTransport                           │
├──────────┬─────────────┬───────────────────┤
│ UsbTrans │ BleTrans    │ AntTrans          │  ← Concrete transports
├──────────┼─────────────┼───────────────────┤
│ IHidDev  │ IBleDev     │ IAntDev           │  ← User-provided devices
└──────────┴─────────────┴───────────────────┘
```

### Key Design Decisions

- **Transport polymorphism**: `PerformanceMonitor` detects transport type via `as` checks and selects the appropriate streaming strategy (polling, BLE notifications, or ANT+ broadcasts).
- **ANT+ is receive-only**: `AntTransport.SendAsync()`/`ReceiveAsync()` throw `NotSupportedException`. Only `SubscribeToDataPagesAsync()` is supported.
- **CSAFE command wrapping**: PM-specific commands are wrapped inside `SetUserCfg1` (0x1A). Multiple PM commands in a single frame are coalesced under a single wrapper.
- **8-bit rollover counters**: ANT+ stroke/stride counts use 8-bit rollover with `AntConstants.RolloverCounterMax = 256`.

## Versioning

- **GitVersion 6.x** with `workflow: GitHubFlow/v1` and Conventional Commits
- `main` branch: `alpha` pre-release label (e.g., `0.2.0-alpha.3`)
- `release/*` branches: `rc` label (e.g., `0.2.0-rc.1`)
- Pull requests: `alpha` label
- Production: stable versions via manual promote workflow (overrides label to empty)
- Version tags prefixed with `v` (e.g., `v0.2.0`)
- No hardcoded version in `.csproj` — version is passed via `-p:Version=` and `-p:PackageVersion=` in CI

## CI/CD

- **ci.yml**: Triggers on push/PR to `main` and `release/*`. Builds, tests, scans for vulnerabilities, packs, and publishes unlisted pre-release to NuGet.org.
- **promote.yml**: Manual trigger. Builds, tests, packs, publishes stable to NuGet.org and GitHub Packages, creates a git tag.
- **OIDC trusted publishing**: Uses `NuGet/login@v1` — no API keys stored as secrets.
- **Dependabot**: Weekly updates for NuGet packages and GitHub Actions.

## Protocol Reference

### CSAFE

- Frame format: `[StartFlag] [Status] [Payload...] [Checksum] [StopFlag]`
- Start flags: `0xF0` (extended) or `0xF1` (standard)
- Stop flag: `0xF2`
- Byte stuffing: `0xF3` + `(original - 0xF0)` for bytes `0xF0`–`0xF3`
- Checksum: XOR of all payload bytes
- Max frame size: 96 bytes
- Short commands: `≥ 0x80`; Long commands: `< 0x80`

### BLE

- Base UUID: `CE060000-43E5-11E4-916C-0800200C9A66`
- Control service: `CE060020` (write to `CE060021`, notify on `CE060022`)
- Data characteristics: `CE060031` (General Status), `CE060032` (Additional Status)
- Data is little-endian; time in centiseconds, distance in tenths of meters

### ANT+

- Device type: 17 (Fitness Equipment)
- Channel period: 8192 (~4 Hz)
- RF frequency: 57 (2457 MHz)
- Data pages: `0x10` (General FE), `0x12` (Metabolic), `0x16` (Rower), `0x18` (Nordic Skier)
- 8-byte payloads with equipment state in bits 4–6 of the final byte
