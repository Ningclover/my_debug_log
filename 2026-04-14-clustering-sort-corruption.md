# Bug: Heap Corruption in `get_strategic_points` due to Invalid `std::sort` Comparator

**Date**: 2026-04-14  
**File**: `toolkit/clus/src/clustering_extend.cxx`  
**Function**: `get_strategic_points`  
**Symptom**: `malloc(): invalid next size (unsorted)` / `malloc(): unaligned tcache chunk detected`  
**Introduced by**: rebase onto upstream `apply-pointcloud` branch (commit replacing `std::set` with `vector + sort + unique`)  
**Reference (working) commit**: `0a3ea3b429845162b5f88b62d9422eec82c876b0`

---

## Symptom

Wire-cell clustering crashes with a heap corruption error during `Find_Closest_Points`, called from `clustering_regular.cxx:75`:

```
malloc(): invalid next size (unsorted)
```

or

```
malloc(): unaligned tcache chunk detected
```

GDB backtrace pointed to `cluster2->get_closest_point_blob(p1)` → KD-tree construction.

---

## Root Cause

### The broken `operator<` on `D3Vector` / `geo_point_t`

`geo_point_t = WireCell::Point = D3Vector<double>`. Its `operator<` is defined in `toolkit/util/inc/WireCellUtil/D3Vector.h:168`:

```cpp
bool operator<(const D3Vector& rhs) const
{
    if (z() < rhs.z()) return true;
    if (y() < rhs.y()) return true;
    if (x() < rhs.x()) return true;
    return false;
}
```

This is **not a strict weak ordering**. It checks each axis independently without tiebreaking on the others. For two distinct points where one has smaller `z` but the other has smaller `y`, **both `a < b` and `b < a` return `true`** simultaneously.

Example: `a=(439,107,167)` vs `b=(328,55,202)`:
- `a < b`? `z: 167 < 202` → **true**
- `b < a`? `z: 202 < 167` → false; `y: 55 < 107` → **true**

Both directions are true — violates **asymmetry**, a required property of strict weak ordering.

> Note: for **exact duplicates** (same x, y, z), both directions return `false`, which is valid. The bug only triggers for distinct points with crossed axis relationships.

### Why `std::sort` corrupts memory

`std::sort` internally assumes strict weak ordering. When both `a < b` and `b < a` are true for distinct elements, the swap logic produces undefined behavior — it corrupts the vector's contents in place. After the sort, the `points` vector contains garbage: denormal float coordinates and tiny integer values masquerading as blob pointers.

**Debug output showing corruption:**

Before sort (40 points, all valid):
```
[0] p=(439.711,107.261,166.785) cm blob=0x55e138708720
[1] p=(328.627,55.7199,201.975) cm blob=0x55e1381d3be0
...
[39] p=(327.972,55.8733,201.709) cm blob=0x55e1381d3dc0
```

After sort+unique (39 points, all corrupted):
```
[0] p=(1.23516e-322,1.23516e-322,1.28457e-322) cm blob=0x106
[1] p=(1.28457e-322,1.28457e-322,1.28457e-322) cm blob=0x10a
...
[20] p=(437.745,106.666,167.284) cm blob=0x55e138708cc0   ← a few survive
```

The corrupted blob pointers (`0x106`, `0x10a`, ...) and denormal coordinates are the result of `std::sort` writing past valid memory due to undefined behavior.

The crash then occurs later when `Find_Closest_Points` uses these garbage coordinates as KD-tree query points, which triggers heap allocator error detection.

---

## Code Change: Before (working) vs After (broken)

### Before — commit `0a3ea3b` (working)

Used `std::set` for deduplication. `std::set` inserts one element at a time and uses `operator<` on `std::pair<geo_point_t, const Blob*>`. Because the `Blob*` pointer provides a valid tiebreaker when the `geo_point_t` comparison is ambiguous, the broken `operator<` on `geo_point_t` only causes wrong ordering in the set — **not memory corruption**.

```cpp
std::vector<std::pair<geo_point_t, const Blob*>> WireCell::Clus::Facade::get_strategic_points(const Cluster& cluster) {
    // Store unique points and their corresponding blobs
    std::set<std::pair<geo_point_t, const Blob*>> unique_points;

    // 1. Get extreme points along main axes
    {
        auto [high_y, low_y] = cluster.get_highest_lowest_points(1);
        unique_points.emplace(high_y, cluster.blob_with_point(cluster.get_closest_point_index(high_y)));
        unique_points.emplace(low_y, cluster.blob_with_point(cluster.get_closest_point_index(low_y)));

        auto [front, back] = cluster.get_front_back_points();
        unique_points.emplace(front, cluster.blob_with_point(cluster.get_closest_point_index(front)));
        unique_points.emplace(back, cluster.blob_with_point(cluster.get_closest_point_index(back)));

        auto [early, late] = cluster.get_earliest_latest_points();
        unique_points.emplace(early, cluster.blob_with_point(cluster.get_closest_point_index(early)));
        unique_points.emplace(late, cluster.blob_with_point(cluster.get_closest_point_index(late)));
    }

    // 4. Add points from convex hull vertices
    {
        auto hull_points = cluster.get_hull();
        for (const auto& p : hull_points) {
            auto [closest_p, blob] = cluster.get_closest_point_blob(p);
            unique_points.emplace(closest_p, blob);
        }
    }

    // Convert set to vector for return
    std::vector<std::pair<geo_point_t, const Blob*>> points;
    points.insert(points.end(), unique_points.begin(), unique_points.end());

    return points;
}
```

### After — current broken version

Replaced `std::set` with `vector` + `std::sort` + `std::unique`. `std::sort` requires strict weak ordering globally — the broken `operator<` causes UB and memory corruption.

```cpp
    // ...collect into std::vector<std::pair<geo_point_t, const Blob*>> points...

    // Sort by geo_point_t only (deterministic, no pointer dependency) then deduplicate
    std::sort(points.begin(), points.end(),
        [](const auto& a, const auto& b) { return a.first < b.first; });
    points.erase(std::unique(points.begin(), points.end(),
        [](const auto& a, const auto& b) {
            return !(a.first < b.first) && !(b.first < a.first);
        }), points.end());

    return points;
```

---

## Fix

Revert `get_strategic_points` to use `std::set` for deduplication, as in commit `0a3ea3b`.

Alternatively, replace the `std::sort` comparator with a correct lexicographic one:

```cpp
auto lex_less = [](const geo_point_t& a, const geo_point_t& b) {
    if (a.x() != b.x()) return a.x() < b.x();
    if (a.y() != b.y()) return a.y() < b.y();
    return a.z() < b.z();
};
std::sort(points.begin(), points.end(),
    [&lex_less](const auto& a, const auto& b) { return lex_less(a.first, b.first); });
```

The preferred fix is to revert to `std::set` since that was the tested, working approach.

---

## Why `D3Vector::operator<` Was Not Fixed

The broken `operator<` exists in `D3Vector.h` and is used in many places across the codebase. Fixing it globally could have unintended side effects on other code that may depend on the current (incorrect) behavior. The fix is applied locally at the call site only.

---

## Debugging Path Summary

1. GDB backtrace → crash at `get_closest_point_blob` → KD-tree build
2. Added debug prints in `clustering_regular.cxx`, `Find_Closest_Points`, `get_strategic_points`, `get_hull`
3. `get_hull` output: indices and coordinates all valid
4. `get_strategic_points` output: 40 valid points **before** sort, 39 corrupted points **after** sort
5. Identified `std::sort` with broken `operator<` as the cause
6. Compared to reference commit `0a3ea3b` → original used `std::set`, no `std::sort`
