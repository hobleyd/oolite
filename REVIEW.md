# Reviewing the QuickJS-ng port (#767)

This branch exists only to make #767 easier to review. It is not meant to be
merged.

- `quickjs-review-base` is #767 (`quickjs-poc`) with every file under
  `src/Core/Scripting/QuickJS/` (and `src/Core/Debug/QuickJS/OOJSConsole.m`)
  replaced by the SpiderMonkey file it was copied from. Files with no
  SpiderMonkey counterpart (including `QuickJS/meson.build`, which is not a
  copy of the parent's) are left out. The original `OOJSManifest.h` has CRLF
  line endings; it was converted to LF here so that its diff shows only the
  real changes.
- `quickjs-review` puts the QuickJS versions back.

So [the comparison between them](https://github.com/hobleyd/oolite/compare/quickjs-review-base...quickjs-review)
shows, file by file, how each QuickJS file differs from its SpiderMonkey
original. Files with no original show as added.

To see the same thing locally from #767, without this branch:

```sh
# One file
git diff quickjs-poc:src/Core/Scripting/OOJSSun.m quickjs-poc:src/Core/Scripting/QuickJS/OOJSSun.m

# Whole directory in a visual tool (kdiff3, meld, opendiff...)
git fetch https://github.com/hobleyd/oolite quickjs-review
git difftool -d FETCH_HEAD~1 FETCH_HEAD
```

## Why the files are duplicated

#767 adds QuickJS-ng as an option (`-Djs_engine=quickjs`) while SpiderMonkey
stays the default, so both engines have to build from the same tree. Nearly
every line of the scripting layer calls the engine API (`jsval`/`JSObject*`
versus `JSValue`, `JSClass` versus `JSClassDef`, rooting versus reference
counting). Keeping one copy of each file would have meant an `#if` around most
lines, which is harder to read and to review than two clean files.

`src/Core/Scripting/QuickJS/` comes first on the include path, so a QuickJS
build picks up the files there in place of the ones with the same name in the
parent directory. Files that don't use the engine API (`OOScript.m`,
`OOPListScript.m`, `OOScriptTimer.m`, `OOLegacyScriptWhitelist.m`) are shared
by both engines and are not copied. Code outside `src/Core/Scripting` is shared
too; it uses the engine-neutral names from #780 (`OOJSVal`, `OOJSPropID`,
`OOJSFreeValue()`).

The duplication is temporary. #768 makes QuickJS the only engine: it deletes
the SpiderMonkey files and moves the QuickJS files up into
`src/Core/Scripting/`. Until then, a fix to a scripting file has to be made in
both copies.

## The mechanical changes

Most of the diff in most files is the same handful of API translations. Once
you recognise these you can skim past them:

| SpiderMonkey 1.8.5 | QuickJS-ng |
|---|---|
| `static JSObject *sFooPrototype;` | `static JSClassID sFooClassID;` (the prototype comes from `JS_GetClassProto()`) |
| `JSClass sFooClass = { "Foo", flags, JS_PropertyStub, ... }` | `JSClassDef sFooClass = { .class_name = "Foo", .finalizer = ... }` |
| `JSPropertySpec { "name", kFoo_name, OOJS_PROP_READONLY_CB }` | `OOJS_PROPERTY_RO("name", FooGetProperty, kFoo_name)` (or `OOJS_PROPERTY_RW`) |
| `JSFunctionSpec { "bar", FooBar, 1 }` | `JS_CFUNC_DEF("bar", 1, FooBar)` |
| `JS_InitClass(context, global, parentProto, &sFooClass, ...)` | `OOJSRegisterClass()` + `OOJSInitClass(..., OOJS_COUNT(props), ...)` |
| `OOJSRegisterObjectConverter(&sFooClass, ...)` / `OOJSRegisterSubclass(&sFooClass, ...)` | same functions, taking class IDs |
| `-getJSClass:andPrototype:` | `-oo_jsClassID` |
| `JSBool F(JSContext *context, uintN argc, jsval *vp)` | `JSValue F(OOJS_NATIVE_ARGS)` |
| `JSBool FooGetProperty(JSContext*, JSObject*, jsid propID, jsval *value)`, `switch (JSID_TO_INT(propID))`, `*value = x; return YES;` | `JSValue FooGetProperty(OOJS_GETTER_ARGS)`, `switch (magic)`, `return x;` |
| `return NO;` from a native (stop the script) | `return OOJSNativeFailure(context);` |
| `JSObject *` / `jsval` parameters | `JSValueConst` (borrowed) |
| `JS_NewNumberValue(context, d, &v)`, `STRING_TO_JSVAL()`, `OOJSStringFromFoo()` | `JS_NewNumber()`, `JS_NewString()`, `OOJSValueFromFoo()`, all returning owned values |
| `JS_ValueToInt32()` | `OOJSValueToInt32()` (rounds and rejects NaN, as SpiderMonkey did) |
| `JS_AddNamedValueRoot()` / `JS_RemoveValueRoot()` | an owned reference, freed with `JS_FreeValue()` when done and on engine reset |

The new thing to check on every line is **reference counting**. In
SpiderMonkey the garbage collector found everything; in QuickJS every `JSValue`
you're given is owned and must be freed or handed on, and `JSValueConst`
parameters are borrowed and must not be freed. The rules are in the comment at
the top of `QuickJS/OOJavaScriptEngine.h`. Debug builds log
`script.javaScript.reset.liveObjects` after every context teardown, and
anything other than zero is a leak, so the leak check is automated, but
use-after-free and double-free still need a reader's eye.

## Where the real changes are

These files contain changes in behaviour, not only translations. They're listed
in a suggested reading order, with what to look for. The numbers are lines in
the SpiderMonkey original and lines added and removed.

### 1. Engine core

**`OOJavaScriptEngine.h`** (692, +486 −387). Read the comment at the top first:
the ownership rules, error handling and the one-runtime-per-process model. Then
the macros (`OOJS_NATIVE_ARGS`, `OOJS_RETURN*`, `OOJS_PROPERTY_RO/RW`,
`DEFINE_JS_OBJECT_GETTER`) that make the mechanical changes above possible.

**`OOJavaScriptEngine.m`** (2322, +851 −1236). Context creation and `-reset`
(the runtime lives for the whole process; reset replaces only the context), the
time limiter built on QuickJS's interrupt handler (`InterruptHandler`), error
reporting (QuickJS has error locations only in the `stack` property, so
`GetLocationFromStack()` parses it), `OOJSWrapScriptSource()`, loading
`oolite-quickjs-compat.js`, and the native ⇄ JS object converters.

**`OOJSClasses.m`** (new). Class registration, and the emulation of
SpiderMonkey's "shared permanent" properties: SpiderMonkey reported a
prototype's accessors as *own* properties of each instance, so
`Object.keys(ship)` and `JSON.stringify(vector)` saw them. This is done with
exotic methods, only for classes that need them, so Script objects (the
`with()` scope of every script) pay nothing extra.

**`OOJSEngineTimeManagement.m`** (923, +26 −429). Mostly deletion: the watchdog
thread and SpiderMonkey's operation callback are gone, because QuickJS polls the
interrupt handler. What's left is the native profiler, which is the
SpiderMonkey one without `MOZ_TRACE_JSCALLS`.

### 2. Loading and running scripts

**`OOJSScript.m`** (761, +386 −224). The compatibility layer for OXP scripts:
- `RunScriptWithName()`: the scope-chain emulation. Scripts are wrapped in
  `with(this){(function(){ ... }).call(this)}` so bare names resolve to script
  properties as they did in SpiderMonkey. The wrapper is on the first line, so
  line numbers don't change.
- `HasModernScopeDirective()`: scripts marked `"use oolite modern"` skip the
  emulation (see `tools/oxp-upgrade`).
- `RewriteLegacyConst()`: `const` → `var`, because SpiderMonkey 1.8.5's `const`
  was function-scoped and some OXPs rely on that.
- `CachedCompiledScript()` / `CacheCompiledScript()`: the bytecode cache that
  replaces the XDR cache, keyed by a hash of the exact compiled text and the
  QuickJS version.

**`OOJSFunction.m`** (241, +144 −58). QuickJS can't compile a function from a
body and a list of argument names, so it evaluates a function expression
instead, again with the wrapper on the first line.

**`OOJSTimer.m`** (419, +200 −122). Read the "Ownership" comment at the top.
Timers outlive their JS objects, and the function and `this` references
deliberately aren't reported to QuickJS's cycle collector, because it would
free objects that a scheduled timer still needs. Check the order of operations
in `-deleteJSPointers`, `-dealloc` and the callback, where the callback may drop
the last reference to the timer.

### 3. Objects with dynamic properties

SpiderMonkey let a class hook every property get, set, delete and enumeration.
QuickJS has no such hooks, so these four classes use exotic methods
(`get_own_property`, `get_own_property_names`, `define_own_property`,
`get_property`, `set_property`, `delete_property`) to give scripts the same
view. Each file starts with a comment describing the mapping. Check that the
corner cases (names starting with `_`, integer-like names, `Object.prototype`
names such as `toString`, enumeration order) behave as before.

- **`OOJSMissionVariables.m`** (205, +205 −104): `missionVariables.foo`.
- **`OOJSSystemInfo.m`** (738, +569 −265): per-system data keys. Note that
  QuickJS doesn't tell a setter whether the caller is in strict mode, so
  assigning to a read-only key only throws when QuickJS asks for it
  (`JS_PROP_THROW`).
- **`OOJSManifest.m`** (292, +204 −109): `player.ship.manifest`.
- **`OOJSWorldScripts.m`** (110, +93 −63): `worldScripts["name"]`.

### 4. Debug console

**`src/Core/Debug/QuickJS/OOJSConsole.m`** (1053, +424 −423). Mostly
mechanical. The known differences (`pedanticMode` has no effect,
`dumpNamedRoots()` is gone, `dumpHeap()` prints QuickJS's memory summary) are
listed at the end of `QuickJS/README.md`.

### 5. Everything else: mostly mechanical

These files are long, but almost all of their diff is the translation table
above. Skim them for reference counting (a `JSValue` from a call that isn't
freed or returned) and for any line that isn't a straight translation:

`OOJSShip.m`, `OOJSPlayerShip.m`, `OOJSStation.m`, `OOJSSystem.m`,
`OOJSPlayer.m`, `OOJSMission.m`, `OOJSGlobal.m`, `OOJSVisualEffect.m`,
`OOJSEquipmentInfo.m`, `OOJSFrameCallbacks.m`, `OOJSEntity.m`,
`OOJSShipGroup.m`, `OOJSSound.m`, `OOJSSoundSource.m`, `OOJSOolite.m`,
`OOJSPlanet.m`, `OOJSSun.m`, `OOJSDock.m`, `OOJSWaypoint.m`, `OOJSWormhole.m`,
`OOJSFlasher.m`, `OOJSExhaustPlume.m`, `OOJSClock.m`, `OOJSFont.m`,
`OOJSSpecialFunctions.m`, `OOJSGuiScreenKeyDefinition.m`,
`OOJSInterfaceDefinition.m`, `OOJSPopulatorDefinition.m`, `OOJSCall.m`,
`EntityOOJavaScriptExtensions.m`, `OOConstToJSString.m`,
`OOJSEngineDebuggerHelpers.m`, and the headers.

`OOJSVector.m` (1225, +235 −528) and `OOJSQuaternion.m` (925, +203 −386) shrink
because QuickJS's function-list macros are shorter than SpiderMonkey's
declarations; `poc/tests/vector.js` and `poc/tests/quaternion.js` exercise them
outside the game.

## Functions that are gone

Every function and method defined in a SpiderMonkey original but in none of the
QuickJS files, with the reason. Almost all were renamed or replaced by a
QuickJS mechanism; the few real losses are marked **lost** and are all
debug-only.

`Vector3D.toCoordinateSystem()` and `fromCoordinateSystem()`
(`VectorToCoordinateSystem`, `VectorFromCoordinateSystem`) were on this list
until review found them; they had not been ported, and now are.

### Renamed for the QuickJS API

| SpiderMonkey | QuickJS | Notes |
|---|---|---|
| `-getJSClass:andPrototype:` (entities, dock, planet, station, sun, wormhole, waypoint, visual effect, flasher, exhaust plume) | `-oo_jsClassID` / `-oo_jsClassName` | QuickJS identifies a class by `JSClassID`; the prototype comes from `JS_GetClassProto()` |
| `JSShipClass()`/`JSShipPrototype()`, `JSPlayerClass()`/`JSPlayerPrototype()`, `JSPlayerShipClass()`/`JSPlayerShipPrototype()` | `JSShipClassID()`, `JSPlayerClassID()`, `JSPlayerShipClassID()` | as above |
| `OOJSGetClass()` | `JS_GetClassID()` / `OOJSClassName()` | |
| `OOJSStringFromScanClass()` and the other `OOJSStringFrom…()` in `OOConstToJSString.h`; `OOScanClassFromJSString()` and the other `…FromJSString()` | `OOJSValueFrom…()`, `…FromJSValue()` | same tables; they take and return `JSValue`s |
| `OOJSStringFromConstantPRIVATE()`, `OOConstantFromJSStringPRIVATE()` | `OOJSValueFromConstantPRIVATE()`, `OOConstantFromJSValuePRIVATE()` | the helpers behind the above |
| `CreateOOJSGlobal()` | `SetUpOOJSGlobal()` | QuickJS creates the global object with the context; Oolite only adds its properties |
| `+[OOJSValue valueWithJSObject:inContext:]`, `-initWithJSObject:inContext:` | `+valueWithJSValue:inContext:`, `-initWithJSValue:inContext:` | |
| `OOStringFromJSString()` | `OOStringFromJSValue()` | QuickJS strings are `JSValue`s |
| `OOStringFromJSPropertyIDAndSpec()` | `OOStringFromJSPropertyMagic()` | property names come from the `JSCFunctionListEntry` table, keyed by magic number |
| `OOJSInitJSIDCachePRIVATE()` (behind `OOJSID()`) | `OOJSInternAtomPRIVATE()` | jsids become atoms |
| `OOJSStrLiteralCachePRIVATE()` (behind `OOJSSTR()`) | none | `OOJSSTR()` now makes a string from the cached `OOJSID()` atom |
| `JSNewNSArrayValue()`, `JSNewNSDictionaryValue()` | `JSArrayFromNSArray()`, `JSObjectFromNSDictionary()` | the wrappers only existed to manage GC root scopes |

### SpiderMonkey class hooks, replaced by property tables or exotic methods

QuickJS has no per-class get/set/add/enumerate/delete hooks (see "Objects with
dynamic properties" above).

| SpiderMonkey | QuickJS |
|---|---|
| `WormholeSetProperty` | All Wormhole properties are read-only, so the setter only ever reported an error. They are getter-only (`OOJS_PROPERTY_RO`), and QuickJS reports the assignment itself |
| `MissionVariablesEnumerate`, `KeyForPropertyID` | exotic methods in `OOJSMissionVariables.m`; the `mission_` prefixing is now `KeyForPropertyName` |
| `SystemInfoSetProperty`, `SystemInfoDeleteProperty`, `SystemInfoEnumerate` | exotic methods in `OOJSSystemInfo.m` (`SystemInfoSetPropertyMethod` is the unrelated `setProperty()` JS method, which is still there) |
| `WorldScriptsGetProperty`, `WorldScriptsEnumerate` | `WorldScriptsGetOwnProperty`, `WorldScriptsGetOwnPropertyNames` |
| `ScriptAddProperty` (warned about a `tickle` handler) | the warning is now given after the script runs, by checking for a `tickle` property (`OOJSScript.m`) |

### Time limiter and profiler

| SpiderMonkey | QuickJS |
|---|---|
| `-[OOJSTimeManagementLimiter watchdogTimerThread]`, `OperationCallback` | QuickJS polls an interrupt handler (`InterruptHandler` in `OOJavaScriptEngine.m`), so no watchdog thread is needed |
| `OOJSCopyTimeLimiterNominalStartTime()` | nothing called it |
| `TraceEnterJSFunction`, `FunctionCallback`, `ContextCallback`, `CleanUpJSFrame`, `-[OOJSProfileStackFrame initWithJSFunction:context:]`, `-[OOTimeProfile setJavaScriptTime:]` | all inside `#ifdef MOZ_TRACE_JSCALLS` (the `oo_javascript_trace` build option, off by default), which profiled JavaScript functions as well as natives. QuickJS has no function-call hook, so a QuickJS build profiles natives only, as a default SpiderMonkey build did |

### Garbage collector roots

| SpiderMonkey | QuickJS |
|---|---|
| `-removeGCObjectRoot:`, `-removeGCValueRoot:` | QuickJS uses reference counting: the owner holds a reference and frees it with `JS_FreeValue()` (for example `-[OODebugMonitor dealloc]`) |
| `ConsoleDumpNamedRoots`, `DumpCallback` (`console.dumpNamedRoots()`) | **lost**: QuickJS has no named roots. Listed in the README's known differences |

### Engine and console internals

| SpiderMonkey | QuickJS |
|---|---|
| `-lookUpStandardClassPointers`, `-objectClass`, `-stringClass`, `-arrayClass`, `-numberClass`, `-booleanClass`, `-registerStandardObjectConverters` | the converters test values directly (`JS_IsArray()`, `JS_IsString()`, the value tag) instead of looking up SpiderMonkey's standard `JSClass` pointers |
| `UnregisterObjectConverters`, `UnregisterSubclasses` | the converter and superclass tables are fixed arrays indexed by class ID (`OOJavaScriptEngine.m`, `OOJSClasses.m`), so there are no map tables to free |
| `ReportJSError`, `GetLocationNameAndLine`, `-sendMonitorError:withMessage:inContext:` | QuickJS has no error reporter callback. Callers that see an exception call `OOJSReportPendingException()`, which logs it with `LogProblem()`, reading the location from the error's `stack`; the debug monitor gets it through `-jsEngine:context:problemOfSeverity:…` |
| `OOJSDescribeLocation()`, `OOJSMarkConsoleEvalLocation()` | locations come from the `stack` property. **Lost** (debug console only): locations in code typed into the console were shown as `<console input>`; they are now reported like any other location (see `OOJSSpecialFunctions.m`) |
| `DumpVariable` | stack dumps without local variables: QuickJS has no frame inspection API. Listed in the PR under "Not yet done" |
| `DebuggerHook` | **lost**: in debug builds a `debugger;` statement used to log a stack dump. QuickJS has no debugger hook, so it does nothing |
| `OOJSUnreachable()` | it trapped the unreachable end of `OOJS_PROFILE_EXIT_VAL`. That macro is gone (functions return inside the `@try`), so there is nothing left to trap |
| `LoadScriptWithName`, `CompiledScriptData`, `ScriptWithCompiledData` | `RunScriptWithName`, `CachedCompiledScript`, `CacheCompiledScript`: bytecode from `JS_WriteObject()`/`JS_ReadObject()` instead of XDR |
| `ConsoleFinalize` | `OOJSObjectWrapperFinalize`, as for other wrapped objects |
| `OOJSConsoleDestroy()` | it cleared the cached Console prototype; QuickJS keeps prototypes by class ID |

### Debug statistics

| SpiderMonkey | QuickJS |
|---|---|
| `-reportJSVectorStatistics`, `-clearJSVectorStatistics`, `-reportJSQuaternionStatistics`, `-clearJSQuaternionStatistics` | **lost**: `OO_DEBUG`-only counters of how vectors and quaternions were converted, read with `PS.callObjC()` from the console. They counted SpiderMonkey conversion paths that don't exist in the port |

## Not in the copied files

The rest of #767's diff is outside these directories:

- `Resources/Scripts/oolite-quickjs-compat.js`: `toSource()`, `uneval()` and
  the `const` rewriter.
- `subprojects/quickjs-ng.wrap` and
  `subprojects/packagefiles/quickjs-ng/0001-oolite-primitive-property-assignment.patch`:
  QuickJS-ng v0.17.0 built from source, with the patch that makes
  `"str".foo = 1` silently do nothing, as SpiderMonkey did.
- `src/Core/Scripting/QuickJS/poc/`: a standalone test harness for the engine
  core (`-Dquickjs_poc=true`).
- `tools/js-parity/`: runs the game headless with a probe OXP and compares
  what scripts see with a SpiderMonkey baseline.
- `tools/oxp-upgrade/`: rewrites OXP scripts so they don't need the scope
  emulation.
- Build and CI: `meson.options`, the `meson.build` files and
  `.github/workflows/build-all.yaml`.

`src/Core/Scripting/QuickJS/README.md` covers the design, the emulated
behaviour and the known differences in more detail.
