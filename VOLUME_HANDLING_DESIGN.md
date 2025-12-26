# Safe Volume/Mount Handling in Dash to Dock

## Problem Statement

GNOME Shell crashes with heap corruption (SIGABRT/SIGSEGV) when removable
media (CDs, USB drives) are inserted or ejected. The crash occurs because
the extension accesses disposed GProxyVolume/GProxyMount objects.

### Root Cause

The gvfs/GJS integration has a fundamental design mismatch:

1. **GJS wraps GObjects** with JavaScript proxy objects and manages references
2. **gvfs can "dispose" objects** (mark them invalid) while JS still holds references
3. **The JS wrapper outlives the C object** - it continues to exist with the same identity
4. **No safe way to detect disposal** - there's no `isDisposed()` method in GJS
5. **Calling ANY method corrupts the heap** at the C level, before JavaScript
   exception handling can intervene
6. **Even `Array.includes()` is unsafe for validation** - it compares JS wrapper
   identity, not C object validity

### Crash Scenario

```
1. CD is inserted
2. gvfs creates GProxyVolume, emits volume-added signal
3. Extension stores reference to volume, caches data
4. CD is ejected
5. gvfs disposes GProxyVolume (C object freed)
6. JS wrapper still exists, still passes === comparison
7. Extension calls method on "valid" reference
8. C-level heap corruption → SIGABRT
```

The crash can occur in various code paths:
- Signal handlers calling `mount.get_volume()` (returns disposed object)
- Iterating `VolumeMonitor.get_volumes()` during concurrent disposal
- Calling methods on stored references after disposal
- Any method call on objects returned by transitive access

## Design Rules

These rules apply to any GJS code that interacts with gvfs-managed objects
(GProxyVolume, GProxyMount, and objects returned by their methods).

### Rule 1: Signals Are the Source of Truth

VolumeMonitor signals (`volume-added`, `volume-removed`, `mount-added`, etc.)
are the ONLY safe way to receive GObject references. The object passed to a
signal handler is guaranteed valid for the duration of that synchronous
handler execution.

```javascript
// SAFE: Object from signal handler
this._monitor.connect('volume-added', (_, volume) => {
    const uuid = volume.get_uuid();  // Safe - volume is valid here
    const name = volume.get_name();
    this._createVolumeApp(uuid, name);
});

// UNSAFE: Iterating get_volumes() - any volume could be disposed
this._monitor.get_volumes().forEach(v => {
    const uuid = v.get_uuid();  // May crash if v is disposed
});
```

### Rule 2: Extract Immediately, Store Primitives Only

In signal handlers, immediately extract ALL needed data as primitives
(strings, numbers, booleans). Never store GObject references for later use.

```javascript
// SAFE: Store primitives only
this._volumeId = {
    uuid: volume.get_uuid(),
    device: volume.get_identifier('unix-device'),
};
this._volumeCanMount = volume.can_mount();
this._volumeCanEject = volume.can_eject();
this._cachedName = volume.get_name();

// UNSAFE: Storing GObject reference for later use
this._volume = volume;  // Will become invalid when disposed
```

### Rule 3: No Transitive Object Access

Don't call methods that return other GObjects (like `mount.get_volume()` or
`volume.get_mount()`). The returned object may already be disposed even if
the source object is valid.

```javascript
// SAFE: Use mount directly, match by URI
_onMountAdded(mount) {
    const mountUri = mount.get_default_location()?.get_uri();
    const volumeApp = this._findByUri(mountUri);
}

// UNSAFE: get_volume() returns object that may be disposed
_onMountAdded(mount) {
    const volume = mount.get_volume();  // Returned volume may be disposed
    const uuid = volume.get_uuid();     // CRASH
}
```

**Exception:** `volume.get_mount()` during `volume-added` signal handling is
safe because the volume owns the mount reference and both are valid.

### Rule 4: Identity Comparison Only for Stored References

If you must store a reference (e.g., for identity matching in signal
handlers), you may ONLY use `===` comparison. Never call methods on it.

```javascript
// Store reference at construction (when volume is known valid)
this._volumeRef = volume;

// SAFE: === comparison doesn't call any methods
_onVolumeRemoved(volume) {
    const app = this._apps.find(a => a._volumeRef === volume);
    if (app) this._removeApp(app);
}

// UNSAFE: Calling method on stored reference
_onVolumeRemoved(volume) {
    const app = this._apps.find(a => a._volumeRef === volume);
    console.log('Removing', app._volumeRef.get_name());  // CRASH
}
```

### Rule 5: Use GFile Methods for Actions

For user-initiated actions (mount, unmount, eject), use GFile-based methods
instead of stored volume/mount references. GFile objects are local (not gvfs
proxies) and safe to use.

```javascript
// SAFE: GFile methods
async mount() {
    await this.location.mount_enclosing_volume(
        Gio.MountMountFlags.NONE, mountOp, cancellable);
}

async unmount() {
    const mount = await this.location.find_enclosing_mount(cancellable);
    await mount.unmount_with_operation(flags, mountOp, cancellable);
}

async ejectUnmounted() {
    // Use CLI with cached device path for unmounted volumes
    const proc = Gio.Subprocess.new(['gio', 'mount', '-e', this._devicePath], ...);
    await proc.wait_async(cancellable);
}

// UNSAFE: Stored volume reference
async mount() {
    await this._volume.mount(...);  // _volume may be disposed
}
```

### Rule 6: Initialization is a Special Case

At startup, iterating `get_volumes()` is relatively safe because no volumes
are being concurrently disposed. This is acceptable ONLY in the constructor,
before any removable media activity can occur.

```javascript
constructor() {
    // Safe at startup - no concurrent disposals yet
    this._monitor.get_volumes().forEach(v => this._onVolumeAdded(v));
}

// UNSAFE: Called when settings change (concurrent disposal possible)
_onSettingsChanged() {
    this._monitor.get_volumes().forEach(v => ...);  // May crash
}
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     VolumeMonitor Signals                        │
│         (volume-added, volume-removed, mount-added, etc.)       │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Signal Handler (synchronous)                  │
│                                                                  │
│  • Object passed is VALID for this execution only               │
│  • Extract ALL needed data as primitives immediately            │
│  • Use === for identity matching with stored refs               │
│  • NO transitive access (get_volume, get_mount)                 │
│  • volume.get_mount() OK during volume-added only               │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Pure Data Store                             │
│                                                                  │
│  • UUIDs, device paths, URIs (strings)                          │
│  • Capabilities: canMount, canEject, canUnmount (booleans)      │
│  • State: isMounted, isNetworkVolume (booleans)                 │
│  • Display: name, icon (strings or safe GLib types)             │
│  • Location: GFile reference (safe - not a gvfs proxy)          │
│  • _volumeRef: for identity comparison only (never call methods)│
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         UI / Actions                             │
│                                                                  │
│  • Render from pure data only                                   │
│  • getApps() filters based on settings                          │
│  • Actions use GFile methods (mount_enclosing_volume, etc.)     │
│  • Eject unmounted: use 'gio mount -e' with cached device path  │
└─────────────────────────────────────────────────────────────────┘
```

## Implementation Summary

### MountableVolumeAppInfo

Caches all volume/mount data as primitives at construction time:

| Field | Type | Source | Purpose |
|-------|------|--------|---------|
| `_volumeId` | `{uuid, device}` | `volume.get_uuid()`, `get_identifier()` | Identification |
| `_cachedId` | string | Computed from UUID | Stable ID for comparisons |
| `_volumeRef` | GProxyVolume | Stored at construction | Identity comparison only |
| `_isNetworkVolume` | boolean | `get_identifier('class')` | Filtering |
| `_volumeCanMount` | boolean | `volume.can_mount()` | Action availability |
| `_volumeCanEject` | boolean | `volume.can_eject()` | Action availability |
| `_isMounted` | boolean | Updated via signals | State tracking |
| `_canUnmount` | boolean | Updated via signals | Action availability |
| `_canEject` | boolean | Updated via signals | Action availability |
| `name` | string | `get_name()` | Display |
| `icon` | Gio.Icon | `get_icon()` | Display |
| `location` | GFile | `get_activation_root()` | Actions via GFile methods |

### Signal Handlers

| Signal | Handler | Key Safety Measures |
|--------|---------|---------------------|
| `volume-added` | `_onVolumeAdded` | Extract all data immediately; `get_mount()` OK here |
| `volume-removed` | `_onVolumeRemoved` | Match by `===` identity only |
| `volume-changed` | `_onVolumeChanged` | Match by `===`, then call `refresh()` with signal's volume |
| `mount-added` | `_onMountAdded` | Match by URI string; NO `get_volume()` call |
| `mount-removed` | `_onMountRemoved` | Match by URI string; update cached state |

### Settings Changes

| Setting | Handler | Approach |
|---------|---------|----------|
| `show-mounts-only-mounted` | Emit 'changed' | `getApps()` filters dynamically |
| `show-mounts-network` | `_onShowMountsNetworkChanged` | Filter existing volumeApps |

### User Actions

| Action | Method | Implementation |
|--------|--------|----------------|
| Mount | `location.mount_enclosing_volume()` | GFile method, no volume ref |
| Unmount | `location.find_enclosing_mount()` → `unmount()` | Fresh mount from GFile |
| Eject (mounted) | `location.find_enclosing_mount()` → `eject()` | Fresh mount from GFile |
| Eject (unmounted) | `gio mount -e <device>` | CLI with cached device path |

## Testing

To reproduce the original crash:
1. Insert an audio CD
2. Wait for it to mount
3. Eject the CD (or let a ripping tool finish and eject)
4. Observe crash (may take 1-2 cycles)

With the fix applied, the CD can be inserted and ejected repeatedly without
crashes. The dock updates correctly to show/hide the volume icon.

## Related Issues

- https://bugs.launchpad.net/ubuntu/+source/gnome-shell-extension-ubuntu-dock/+bug/2137078
- https://github.com/micheleg/dash-to-dock/issues/2255 (different but related)
