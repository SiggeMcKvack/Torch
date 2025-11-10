# Torch Dependency Upgrade Checklist

## Quick Reference - Upgrade Priority

### CRITICAL (Do Immediately)
- [ ] TinyXML2: 10.0.0 → 11.0.0 (Fixes CVE-2024-50614, CVE-2024-50615)

### HIGH (Next 2-4 weeks)
- [ ] spdlog: 1.12.0 → 1.16.0 (Fixes CVE-2025-6140)
- [ ] CMake: 3.12 → 3.20+ (Better C++20 support)

### MEDIUM (Next 1-2 months)
- [ ] StormLib: 9.22.0 → 9.30 (If BUILD_STORMLIB enabled)
- [ ] CLI11: 2.3.2 → 2.6.0 (Optional, stable version)
- [ ] libgfxd: Investigate version/tagging

### NO ACTION NEEDED
- [ ] yaml-cpp: 0.8.0 (Latest stable, no CVEs)

---

## Phase 1: Critical Fixes (Week 1-2)

### TinyXML2 11.0.0 Upgrade
**File:** `/home/user/Torch/CMakeLists.txt` (Line 269)

**Current:**
```cmake
GIT_TAG 10.0.0
```

**Change To:**
```cmake
GIT_TAG 11.0.0
```

**Testing Checklist:**
- [ ] Search codebase for `int` return types from tinyxml2 calls
- [ ] Identify XMLDocument method return value usages
- [ ] Compile and run existing tests
- [ ] Test XML parsing with valid files
- [ ] Test XML parsing with edge cases/malformed files
- [ ] Verify size_t conversions are handled correctly
- [ ] Run full test suite

**Files to Review:**
- [ ] Grep for "tinyxml2" usage: `grep -r "tinyxml2" src/`
- [ ] Check for XMLDocument function calls returning int
- [ ] Look for string length operations

---

## Phase 2: High Priority (Week 2-4)

### spdlog 1.16.0 Upgrade
**File:** `/home/user/Torch/CMakeLists.txt` (Line 245 & 256)

**Current:**
```cmake
GIT_TAG 7e635fca68d014934b4af8a1cf874f63989352b7
```

**Change To:**
```cmake
GIT_TAG v1.16.0
```

**Testing Checklist:**
- [ ] Compile with updated spdlog
- [ ] Run logging tests across all modules
- [ ] Check async logger behavior
- [ ] Verify no performance regressions
- [ ] Test on multiple platforms (Windows, Linux, macOS)

---

### CMake Requirement Upgrade
**File:** `/home/user/Torch/CMakeLists.txt` (Line 1)

**Current:**
```cmake
cmake_minimum_required(VERSION 3.12)
```

**Change To:**
```cmake
cmake_minimum_required(VERSION 3.20)
```

**Additional Updates:**
- [ ] Update README.md CMake requirement
- [ ] Update CI/CD workflows (if applicable)
- [ ] Test on all target platforms

**Testing Checklist:**
- [ ] Test on Windows (MSVC 2022)
- [ ] Test on Linux (GCC/Clang)
- [ ] Test on macOS (Xcode)
- [ ] Test Emscripten build (if used)
- [ ] Verify C++20 compilation
- [ ] Check all compiler warnings

---

## Phase 3: Medium Priority (Month 2)

### StormLib 9.30 Upgrade (If Enabled)
**File:** `/home/user/Torch/lib/StormLib/CMakeLists.txt` (Lines 346-350)

**Current:**
```cmake
SET(VERSION_MAJOR "9")
SET(VERSION_MINOR "22")
SET(VERSION_PATCH "0")
```

**Change To:**
```cmake
SET(VERSION_MAJOR "9")
SET(VERSION_MINOR "30")
SET(VERSION_PATCH "0")
```

**Testing Checklist:**
- [ ] Only do if BUILD_STORMLIB=ON is used
- [ ] Test MPQ archive operations
- [ ] Test archive creation/extraction
- [ ] Test on Windows (primary StormLib platform)

---

### CLI11 2.6.0 Upgrade (Optional)
**Note:** Current version (2.3.2) is stable - only upgrade if needed

**If Deciding to Upgrade:**
- [ ] Update CLI11.hpp to version 2.6.0
- [ ] Review ExtraValidators.hpp changes
- [ ] Update CMake requirement to 3.14+ (if upgrading from 3.20, already satisfied)
- [ ] Test command-line parsing

---

### libgfxd Investigation
**Action Items:**
- [ ] Visit https://github.com/glankk/libgfxd/releases
- [ ] Identify latest release version
- [ ] Review changelog for breaking changes
- [ ] Consider switching from commit hash to release tag:

**File:** `/home/user/Torch/CMakeLists.txt` (Line 50)

**Current:**
```cmake
GIT_TAG 96fd3b849f38b3a7c7b7f3ff03c5921d328e6cdf
```

**Should Be (when version identified):**
```cmake
GIT_TAG v<version>  # e.g., v1.2.3
```

---

## Documentation Updates

- [ ] Update `/home/user/Torch/README.md` with new CMake requirement
- [ ] Update any CONTRIBUTING.md files
- [ ] Add notes about dependency versions to CMakeLists.txt
- [ ] Create CHANGELOG entry documenting all updates

---

## Pre-Commit Verification

**Before committing any changes:**
```bash
# Full build test
cmake -B build-test -DCMAKE_BUILD_TYPE=Release
cmake --build build-test

# Run all tests
cd build-test
ctest

# Check for compilation warnings
cd ..
cmake -B build-warn -DCMAKE_BUILD_TYPE=Debug
cmake --build build-warn 2>&1 | grep -i "warning\|error"
```

---

## Rollback Plan

If issues are discovered:

1. **TinyXML2 11.0.0 Issues:**
   - Revert to 10.0.0 and patch locally if needed
   - Check for type mismatch compilation errors
   - File issue on tinyxml2 GitHub if bug found

2. **spdlog 1.16.0 Issues:**
   - Revert to 1.15.2 (first CVE-fixed version)
   - Very backward compatible, unlikely to cause issues

3. **CMake 3.20 Issues:**
   - Revert to 3.12 and investigate
   - File issue with CMake project if toolchain issue

---

## Sign-Off Checklist

**Before considering upgrades complete:**

- [ ] All critical (CRITICAL) upgrades merged
- [ ] All high priority (HIGH) upgrades merged
- [ ] Full test suite passes on all platforms
- [ ] Documentation updated
- [ ] PR reviewed and approved
- [ ] No new CVEs identified in upgraded versions
- [ ] Performance benchmarks run (if applicable)

---

**Generated:** November 10, 2025  
**Analysis Tool:** Claude Code Dependency Analyzer  
**Full Report:** See DEPENDENCY_ANALYSIS.md
