# health_sync — Technical Specification

Status: **Draft v0.1** · Last updated: 2026-10-21

`health_sync` is a Flutter package family that gives an app one API to:

1. check whether a health data source is available and which permissions are granted,
2. request permissions, and
3. read health data into the app, both as one-off queries and as incremental syncs.

v1 targets **Apple HealthKit (iOS)** and **Health Connect (Android)**. The architecture is built so that
cloud providers (**Strava**, **Oura**, later Garmin, Whoop, Fitbit, Withings, and others) can be added
without breaking the core API.

---

## 1. Goals and non-goals

### Goals (v1)

- One typed Dart API over HealthKit and Health Connect for availability, permissions, and reads.
- Permission status that is **honest** about what each platform can report (see §6.2). No faking
  "granted" when the OS won't tell us.
- Two read modes:
  - **Raw records**: the individual samples or sessions, with source and device metadata.
  - **Aggregates**: deduplicated totals or averages per time bucket (for example, steps per day).
- **Incremental sync** through opaque cursors, so an app can pull "what changed since last time",
  including deletions.
- A provider abstraction that later cloud providers (OAuth plus REST) plug into without changes to the core.
- Good testability: a fake provider for app unit tests, and native unit tests in each platform package.

### Non-goals (v1)

- Writing data to HealthKit or Health Connect. The permission model includes `write` access so this can
  be added later without breaking changes.
- Background sync or delivery (planned for M5, see §11). The cursor design is chosen so it can be added later.
- Clinical or medical records (HealthKit clinical records, Health Connect medical data).
- Deduplication *across* providers (for example, Oura sleep that is also mirrored into Apple Health).
  v1 exposes enough source metadata for apps to handle this themselves.
- A hosted backend. The package never sends health data off the device.

---

## 2. Prior art

The `health` package on pub.dev (cph-cachet) is the most popular existing option. We are building our own,
rather than wrapping it, because we want:

- **Multiple providers at once.** `health` is a single-platform-at-a-time plugin and has no concept of
  cloud providers.
- **Typed, sealed models** instead of a flat `HealthDataPoint` with a generic `value`.
- **Incremental sync with deletions**, built on HealthKit anchored queries and Health Connect changes tokens.
- **Explicit permission fidelity**, so apps can't misread HealthKit's opaque read status.

---

## 3. Package architecture

A Dart pub workspace monorepo, managed with `melos` for scripts:

```
health_sync/
├── packages/
│   ├── health_sync/                  # Core: pure Dart. Models, provider interface, HealthSync facade, errors.
│   ├── health_sync_apple/            # HealthKit provider (Flutter plugin, iOS, Swift).
│   ├── health_sync_health_connect/   # Health Connect provider (Flutter plugin, Android, Kotlin).
│   ├── health_sync_oauth/            # (M3) Shared OAuth2/PKCE, token storage, HTTP plus rate limiting.
│   ├── health_sync_strava/           # (M3) Strava provider (pure Dart).
│   ├── health_sync_oura/             # (M4) Oura provider (pure Dart).
│   └── health_sync_testing/          # FakeHealthProvider, fixtures, matchers for app tests.
├── example/                          # Example app that exercises every provider.
└── docs/
```

### Why not a standard federated plugin?

A federated plugin assumes **one** implementation per platform behind one interface. Our model is
different: an iOS user may connect Apple Health **and** Oura **and** Strava at the same time. So each
provider is its own package implementing a shared `HealthProvider` interface, and apps depend only on
the providers they use. This keeps binary size, manifest permissions, and App Store or Play review
surface small.

### Native bridge

- **Pigeon** for type-safe Dart↔native messaging (generated Swift and Kotlin).
- iOS: Swift, async/await HealthKit APIs.
- Android: Kotlin, coroutines, `androidx.health.connect:connect-client`.
- Native layers stay thin: they translate platform records into Pigeon DTOs, and Dart maps DTOs into
  the core models. Mapping logic lives in Dart wherever possible so it is unit-testable without a device.

### Platform minimums (proposed)

| Platform | Minimum | Reason |
|---|---|---|
| iOS | 15.0 | async HealthKit query descriptors, sleep stages (16.0+ for core, deep, and REM; degrade gracefully on 15) |
| Android | minSdk 26, compileSdk latest | Health Connect client requirement |
| Dart / Flutter | Dart 3.5+ / current stable Flutter | sealed classes, pub workspaces |

---

## 4. Core concepts

| Concept | Description |
|---|---|
| **Provider** | A source of health data: `apple_health`, `health_connect`, `strava`, `oura`. Implements `HealthProvider`. |
| **Data type** | A kind of data, such as `steps`, `heartRate`, `sleep`, or `workout`. Extensible by provider packages. |
| **Permission** | `(dataType, access)` where access is `read` or `write`. For OAuth providers this maps to scopes. |
| **Record** | One item of data returned by a provider: a sample, a session, or a workout. Sealed class hierarchy. |
| **Aggregate** | A deduplicated statistic over a time bucket, computed by the platform where possible. |
| **Cursor** | An opaque, serializable token marking a sync position for `(provider, dataType)`. |
| **Connection** | (Cloud providers) An authenticated link between the app user and a provider account. |

---

## 5. Public API (sketch)

### 5.1 Setup

```dart
final healthSync = HealthSync(
  providers: [
    AppleHealthProvider(),          // no-ops on non-iOS: isSupportedOnThisPlatform == false
    HealthConnectProvider(),
    // (M3) StravaProvider(oauth: StravaOAuthConfig(...)),
  ],
);

// Convenience: the on-device provider for the current platform, or null.
final HealthProvider? device = healthSync.deviceProvider;
```

Providers are fully usable on their own. `HealthSync` is an optional facade that adds a registry,
fan-out helpers (for example, "request these permissions from every connected provider"), and a single place
for logging and config.

### 5.2 `HealthProvider` interface

```dart
abstract interface class HealthProvider {
  ProviderId get id;                       // e.g. ProviderId.appleHealth
  ProviderCapabilities get capabilities;
  Set<HealthDataType> get supportedDataTypes;

  // Availability
  Future<ProviderAvailability> checkAvailability();
  Future<void> openInstallOrUpdate();      // Health Connect: Play Store; others: no-op / throws

  // Permissions
  Future<Map<HealthPermission, PermissionStatus>> getPermissionStatus(
      Set<HealthPermission> permissions);
  Future<PermissionRequestResult> requestPermissions(Set<HealthPermission> permissions);
  Future<void> revokeAllPermissions();     // Health Connect supports this; HealthKit throws Unsupported

  // Reads
  Future<RecordPage<T>> readRecords<T extends HealthRecord>(
    HealthDataType<T> type,
    TimeRange range, {
    int? pageSize,
    String? pageToken,
    Set<String>? sourceFilter,             // app bundle IDs / package names
    SortOrder order = SortOrder.ascending,
  });

  Stream<T> readAllRecords<T extends HealthRecord>(HealthDataType<T> type, TimeRange range);

  Future<List<AggregateBucket>> aggregate(
    HealthDataType type,
    TimeRange range, {
    required AggregateBucketSize bucket,   // duration (e.g. 1h) or calendar period (day/week/month)
    Set<AggregateMetric> metrics = const {AggregateMetric.total},
  });

  // Incremental sync
  Future<SyncCursor> getInitialCursor(HealthDataType type, {DateTime? since});
  Future<ChangeSet> getChanges(HealthDataType type, SyncCursor cursor);
}
```

### 5.3 Typical usage

```dart
final provider = healthSync.deviceProvider!;

switch (await provider.checkAvailability()) {
  case ProviderAvailability.available: break;
  case ProviderAvailability.updateRequired:
  case ProviderAvailability.notInstalled:
    await provider.openInstallOrUpdate();
    return;
  case ProviderAvailability.notSupported:
    return;
}

final perms = {
  HealthPermission.read(HealthDataType.steps),
  HealthPermission.read(HealthDataType.heartRate),
  HealthPermission.read(HealthDataType.sleep),
};

final result = await provider.requestPermissions(perms);

final range = TimeRange.lastDays(7);
final dailySteps = await provider.aggregate(
  HealthDataType.steps, range, bucket: AggregateBucketSize.day);

final heartRate = await provider.readAllRecords(HealthDataType.heartRate, range).toList();

// Incremental sync
var cursor = await store.loadCursor() ?? await provider.getInitialCursor(HealthDataType.sleep);
ChangeSet changes;
do {
  changes = await provider.getChanges(HealthDataType.sleep, cursor);
  await db.upsert(changes.upserts);
  await db.delete(changes.deletedIds);
  cursor = changes.nextCursor;
} while (changes.hasMore);
await store.saveCursor(cursor);
```

---

## 6. Availability and permissions

### 6.1 Availability

```dart
enum ProviderAvailability {
  available,
  notInstalled,     // Health Connect APK missing (Android ≤ 13)
  updateRequired,   // Health Connect provider needs update
  notSupported,     // e.g. iPad without HealthKit, wrong platform, device policy
}
```

| Platform | Source |
|---|---|
| iOS | `HKHealthStore.isHealthDataAvailable()` |
| Android | `HealthConnectClient.getSdkStatus()` → `SDK_AVAILABLE` / `SDK_UNAVAILABLE_PROVIDER_UPDATE_REQUIRED` / `SDK_UNAVAILABLE` (plus a check for whether the Play Store intent can resolve) |
| Cloud | `available` if configured; connection state is separate (§9) |

### 6.2 Permission status: the HealthKit problem

**HealthKit deliberately hides read authorization.** An app can't learn whether the user granted or
denied *read* access. A denied read just looks like "no data". `authorizationStatus(for:)` reports
*write* (share) status only. The best signal available is
`getRequestStatusForAuthorization(toShare:read:)`, which says whether the system *would show* the prompt
again (`shouldRequest`) or not (`unnecessary`). It doesn't say what the user chose.

Health Connect, by contrast, reports exact granted permissions via `getGrantedPermissions()`.

So the status model distinguishes these cases explicitly:

```dart
enum PermissionStatus {
  granted,          // definitively granted (HC read/write, HK write)
  denied,           // definitively denied (HC, HK write)
  notDetermined,    // user has never been asked
  requested,        // asked, but platform won't reveal the outcome (HK read)
}

class ProviderCapabilities {
  final PermissionFidelity readPermissionFidelity;  // exact | requestStateOnly
  final bool supportsAggregation;
  final bool supportsChanges;           // anchored / changes-token sync
  final bool supportsDeletionsInChanges;
  final bool supportsRevoke;
  final bool supportsWrite;             // false for all providers in v1
}
```

Guidance in docs: on iOS, treat `requested` as "proceed and handle empty data gracefully". Never block
the UI on a granted check for HealthKit reads.

`PermissionRequestResult` contains the post-request status map plus `userCancelled` where the platform
reports it.

### 6.3 Platform setup requirements (documented, validated at runtime)

The plugin can't, and shouldn't, inject every possible permission. Apps declare only what they use, which
reviewers check.

**iOS**
- HealthKit capability plus entitlement `com.apple.developer.healthkit`.
- `Info.plist`: `NSHealthShareUsageDescription` (read), and `NSHealthUpdateUsageDescription` (write, future).
- Background delivery (M5): `com.apple.developer.healthkit.background-delivery`.

**Android**
- `AndroidManifest.xml`: one `<uses-permission android:name="android.permission.health.READ_*"/>` per data type.
- Optional: `READ_HEALTH_DATA_HISTORY` (reads older than 30 days before first grant), and
  `READ_HEALTH_DATA_IN_BACKGROUND` (M5).
- A privacy-policy rationale activity: an intent filter for `androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE`,
  plus an `activity-alias` for `android.intent.action.VIEW_PERMISSION_USAGE` with
  `android.intent.category.HEALTH_PERMISSIONS` on Android 14+.
- `<queries>` for the Health Connect package (Android ≤ 13).
- A Play Console health permissions declaration.

**Runtime validation:** in debug builds, `requestPermissions` checks the manifest (Android) or Info.plist
and entitlements (iOS) for the requested types. It throws `MisconfigurationException` with an exact fix
message, for example "Add `<uses-permission android:name="android.permission.health.READ_STEPS"/>`".
Failing silently is the top source of confusion with health plugins.

### 6.4 Android activity handling

Permission requests use `PermissionController.createRequestPermissionResultContract()`. The plugin
implements `ActivityAware` and handles results through `ActivityPluginBinding.addActivityResultListener`,
so apps **don't** need to switch to `FlutterFragmentActivity`.

---

## 7. Data model

### 7.1 Data types

`HealthDataType<T extends HealthRecord>` is a **class with const instances, not an enum**. This lets
provider packages define their own types (for example, `OuraDataType.readiness`) while core types stay
shared. Each type declares its record class, canonical unit, and aggregation support.

**MVP types (M1)**

| Type | Record | Canonical unit | HealthKit | Health Connect |
|---|---|---|---|---|
| `steps` | `QuantitySample` | count | `stepCount` | `StepsRecord` |
| `distance` | `QuantitySample` | meter | `distanceWalkingRunning` (+ cycling, swimming…) | `DistanceRecord` |
| `activeEnergy` | `QuantitySample` | kilocalorie | `activeEnergyBurned` | `ActiveCaloriesBurnedRecord` |
| `heartRate` | `QuantitySample` | bpm | `heartRate` | `HeartRateRecord` (series, flattened) |
| `restingHeartRate` | `QuantitySample` | bpm | `restingHeartRate` | `RestingHeartRateRecord` |
| `weight` | `QuantitySample` | kilogram | `bodyMass` | `WeightRecord` |
| `sleep` | `SleepSession` | — | `sleepAnalysis` (grouped into sessions) | `SleepSessionRecord` |
| `workout` | `Workout` | — | `HKWorkout` | `ExerciseSessionRecord` |

**M2 additions:** `heartRateVariability` (note: HK reports SDNN, HC reports RMSSD, so they are kept as
**distinct types** and never silently merged), `oxygenSaturation`, `respiratoryRate`, `vo2Max`, `bodyFat`,
`height`, `bodyTemperature`, `basalEnergy`, `bloodPressure`, `bloodGlucose`, `hydration`, `floorsClimbed`.

### 7.2 Records (sealed hierarchy)

```dart
sealed class HealthRecord {
  String get id;                 // provider-native, stable
  ProviderId get provider;
  HealthDataType get type;
  DateTime get start;            // UTC
  DateTime get end;              // UTC (== start for instantaneous samples)
  Duration? get startZoneOffset; // when known (HC always; HK via HKMetadataKeyTimeZone)
  RecordSource get source;       // app bundle / package, device, recording method
  DateTime? get lastModified;
  Map<String, Object?> get extras; // provider-specific raw fields, namespaced
}

final class QuantitySample extends HealthRecord { double value; HealthUnit unit; String? parentId; }
final class SleepSession   extends HealthRecord { List<SleepStage> stages; String? title; }
final class Workout        extends HealthRecord {
  WorkoutActivity activity;      // unified enum + `other` with raw value preserved
  String rawActivityType;
  double? totalDistanceMeters;
  double? totalEnergyKcal;
  WorkoutRoute? route;           // M2+, requires separate route permission
}
final class BloodPressureSample extends HealthRecord { double systolic; double diastolic; } // M2

class RecordSource {
  String? appId;                 // bundle id / package name
  String? appName;
  DeviceInfo? device;            // manufacturer, model, type (watch, phone, ring…)
  RecordingMethod method;        // automatic | activelyRecorded | manual | unknown
}
```

### 7.3 Mapping decisions

- **Units:** values are always returned in the type's canonical unit. `HealthUnit` offers conversion
  helpers, for example `sample.valueIn(HealthUnit.mile)`.
- **Series flattening:** Health Connect series records (heart rate, speed, power) are flattened into
  `QuantitySample`s with `id = "<recordId>#<index>"` and `parentId = recordId`. This keeps one shape
  across platforms.
- **Sleep sessions on iOS:** HealthKit stores sleep as individual category samples. The Apple provider
  groups contiguous samples from the same source into a `SleepSession`. A gap of ≥ 30 min starts a new
  session, and the threshold is configurable.
- **Sleep stages** (unified): `awake`, `inBed`, `asleepUnspecified`, `light`, `deep`, `rem`, `outOfBed`, `unknown`.
  HK `asleepCore` maps to `light`. HC `awakeInBed` maps to `awake`. Raw values are preserved in `extras`.
- **Workout activity:** a unified enum of about 40 common activities, with `other`. The raw platform value
  is always preserved.
- **Timestamps:** always UTC `DateTime` plus an optional zone offset. Callers bucket by local day using
  the aggregate API, or use the offset themselves.

---

## 8. Reading

### 8.1 Raw records

- Paged, with an opaque `pageToken` (HC native page tokens; HK uses a sort-date plus anchor cursor).
- `readAllRecords` is a `Stream` convenience that pages lazily.
- Default page size is 500, and the max is clamped per platform.
- Health Connect reads are **foreground-only** without the background permission, and are rate-limited.
  The plugin surfaces `RateLimitedException` and never retries silently in a loop.

### 8.2 Aggregates

Apps should use aggregates for totals. Summing raw samples double-counts, because iPhone and Watch both
record steps.

| | HealthKit | Health Connect |
|---|---|---|
| Implementation | `HKStatisticsCollectionQuery` (built-in source dedup) | `aggregateGroupByDuration` / `aggregateGroupByPeriod` (dedup by user's app priority) |
| Metrics | total, average, min, max | total, average, min, max (per record type availability) |
| Calendar buckets | anchored to device local calendar | `LocalDateTime` period buckets |

Calendar buckets (`day`, `week`, `month`) use the device's current time zone. Duration buckets (`1h`,
`15m`) are time-zone-agnostic. Unsupported metric and type combinations throw `UnsupportedAggregationException`
up front, based on a static capability table.

### 8.3 Incremental sync (changes)

| | HealthKit | Health Connect | Strava (M3) | Oura (M4) |
|---|---|---|---|---|
| Mechanism | `HKAnchoredObjectQuery` | `getChangesToken` / `getChanges` | `after=` timestamp on activities list | date-range re-fetch with overlap window |
| Deletions | ✅ `deletedObjects` | ✅ `DeletionChange` | ❌ (webhooks only) | ❌ |
| Expiry | none | tokens expire after ~30 days | n/a | n/a |

`SyncCursor` is an opaque, versioned, `String`-serializable value (`{v, provider, type, payload}`), so
apps can persist it anywhere. When a cursor is expired or invalid, `getChanges` throws
`CursorExpiredException`. The app then does a bounded full re-read and calls `getInitialCursor` again.
`ChangeSet.deletionsSupported` tells the app whether a missing deletion means "not deleted" or
"unknown".

---

## 9. Cloud providers (M3+) — designed now, built later

### 9.1 Connection lifecycle

Cloud providers add a connection layer on top of the same `HealthProvider` interface:

```dart
abstract interface class ConnectableHealthProvider implements HealthProvider {
  Future<ConnectionState> getConnectionState();   // disconnected | connected | expired | revoked
  Future<void> connect(Set<HealthPermission> permissions); // launches OAuth (PKCE) in browser / ASWebAuthenticationSession
  Future<void> disconnect();                        // revoke token + clear storage
}
```

For these providers, `requestPermissions` means connecting, or reconnecting with more scopes.
`getPermissionStatus` derives from the granted scopes on the stored token.

### 9.2 OAuth and secrets

- Authorization uses PKCE plus a system browser (`flutter_web_auth_2` or equivalent) with a custom-scheme or
  universal-link redirect.
- **Client secrets must not ship in the app.** Strava's token exchange requires `client_secret`, so the
  OAuth package exposes a `TokenExchanger` interface. The app supplies a call to **its own backend**
  for code exchange and refresh. A direct-exchange implementation exists only for development,
  clearly labeled.
- Token storage goes through a `TokenStore` interface. The default implementation uses `flutter_secure_storage`.
- Refresh is automatic and single-flight (concurrent requests share one refresh).

### 9.3 Provider notes

- **Strava**: scopes `read`, `activity:read`, `activity:read_all`, `profile:read_all`. Activities map to
  `Workout`. Streams (heart rate, power) map to `QuantitySample`s. Rate limits are read from response headers. The
  **API agreement** restricts how Strava data can be shown or used, including no AI/ML training and
  limits on showing data to other users. This will be documented prominently.
- **Oura (API v2)**: sleep maps to `SleepSession` (hypnogram to stages), heart rate maps to `QuantitySample`, workouts
  map to `Workout`. Readiness, activity score, and sleep score are **provider-specific types**
  (`OuraDataType.readiness`) defined in `health_sync_oura`. They are not forced into core.
- **Webhooks** (both providers support them) need a server, so they are out of scope for the package. Docs
  will show how a backend can call `getChanges`-equivalent APIs.

---

## 10. Errors, logging, privacy

```dart
sealed class HealthSyncException implements Exception {}
final class ProviderUnavailableException  extends HealthSyncException {}
final class PermissionDeniedException     extends HealthSyncException {}  // HC / cloud only
final class MisconfigurationException     extends HealthSyncException {}  // missing manifest/plist/entitlement
final class NotConnectedException         extends HealthSyncException {}  // cloud
final class RateLimitedException          extends HealthSyncException { Duration? retryAfter; }
final class CursorExpiredException        extends HealthSyncException {}
final class UnsupportedDataTypeException  extends HealthSyncException {}
final class UnsupportedAggregationException extends HealthSyncException {}
final class NativeHealthException        extends HealthSyncException { String code; String? message; } // wrapped native error
```

- **Logging:** a `HealthSyncLogger` hook, off by default. Logs never include health values, only types,
  counts, and timings.
- **Privacy:** no network calls from the device providers. No analytics. Cloud providers talk only to the
  provider API and the app's own token-exchange endpoint.

---

## 11. Milestones

| Milestone | Scope |
|---|---|
| **M0 — Foundation** | Workspace and melos, CI (analyze, format, test; iOS and Android builds), core models, `HealthProvider` interface, `health_sync_testing` fake provider, Pigeon scaffolding, example app shell. |
| **M1 — Device providers MVP** | HealthKit plus Health Connect: availability, permission status and request, `readRecords` for the 8 MVP types, misconfiguration validation, setup docs. |
| **M2 — Aggregates and sync** | `aggregate`, `getChanges` and cursors, M2 data types, Health Connect history permission, workout routes. |
| **M3 — OAuth and Strava** | `health_sync_oauth`, `ConnectableHealthProvider`, Strava provider. |
| **M4 — Oura** | Oura provider plus provider-specific types. |
| **M5 — Background** | HealthKit background delivery (observer queries), Health Connect background reads via WorkManager. Dart callback dispatch in a headless isolate. |
| **Later** | Write support, Garmin, Whoop, Fitbit or Google Health API, Withings, cross-provider dedup helpers. |

## 12. Testing strategy

- **Core (Dart):** unit tests for models, unit conversion, cursor serialization, mapping from Pigeon DTOs
  (fixture-driven: JSON fixtures of real platform payloads).
- **iOS:** XCTest against the provider using a protocol-wrapped `HKHealthStore`. The HealthKit simulator
  supports real reads and writes for integration tests.
- **Android:** JUnit plus `androidx.health.connect:connect-testing` (`FakeHealthConnectClient`) for
  permissions, reads, aggregates, and changes.
- **Integration:** `integration_test` in the example app on the iOS simulator and an Android emulator with
  Health Connect. CI seeds data, then asserts reads.
- **Cloud:** recorded HTTP fixtures, so there are no live API calls in CI.
- **App developers:** `FakeHealthProvider` with programmable availability, permission states (including
  HK-style `requested`), and seeded records.

---

## 13. Open questions

1. **Minimum iOS version.** 15.0 (proposed) or 16.0 (simpler sleep-stage handling, drops some users)?
2. **Write support timing.** Is writing (for example, logging workouts) needed before cloud providers?
3. **Background sync priority.** Should M5 come before Strava and Oura?
4. **Cloud token exchange.** Do you already have a backend that can host the OAuth token exchange for
   Strava and Oura? (Assumed: yes, or one will be built.)
5. **HRV.** Is it acceptable to expose SDNN (iOS) and RMSSD (Android) as distinct types, or should we
   offer a "best available HRV" convenience?
6. **Package naming and publishing.** `health_sync`, `health_sync_apple`, and `health_sync_health_connect`
   are currently free on pub.dev. Will these be published publicly or kept private?
7. **Data types.** Which data types matter most for your app? This decides which types make the M1 cut.
8. **License.** MIT, BSD-3, or Apache-2.0?
