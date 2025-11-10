# Torch Project - Dependency Analysis & Upgrade Recommendations

**Analysis Date:** November 10, 2025  
**Project:** Torch - A generic asset processor for games  
**Current Build System:** CMake 3.12+  
**Current C++ Standard:** C++20 (Main), C++11 (StormLib)

---

## Executive Summary

The Torch project has several outdated dependencies with known security vulnerabilities, particularly **TinyXML2** which has two CVEs affecting the current version. Additionally, **spdlog** has one vulnerability in the current version that's fixed in newer releases. Other dependencies are relatively stable but would benefit from updates for bug fixes, performance improvements, and security patches.

**Overall Assessment:** Multiple dependencies require attention, with 1 **CRITICAL**, 2 **HIGH**, and 3 **MEDIUM** priority upgrades recommended.

---

## Detailed Dependency Analysis

### 1. **TinyXML2** - XML Parser

**Current Version:** 10.0.0 (GIT_TAG)  
**Latest Stable:** 11.0.0 (as of March 2025)  
**Last Updated:** March 8, 2025  

**Priority:** CRITICAL

**Current Issues:**
- **CVE-2024-50614:** Reachable assertion for UINT_MAX/16 in XMLUtil::GetCharacterRef that may lead to application exit
- **CVE-2024-50615:** Reachable assertion for UINT_MAX/digit in XMLUtil::GetCharacterRef that may lead to application exit
- Both vulnerabilities allow for Denial of Service attacks through specially crafted XML files

**Fixed In:** Version 10.1.0+ and 11.0.0

**Upgrade Path:**
- **Recommended:** Upgrade to 11.0.0 (Latest)
- **Note:** Version 11.0.0 includes a minor API breaking change in return size of strings (int -> size_t)

**Benefits of Upgrading:**
- Fixes both CVE-2024-50614 and CVE-2024-50615
- Improved type safety with int to size_t conversions
- Build fixes and typo corrections

**Migration Effort:** Low - Main change is handling the return type change from int to size_t

---

### 2. **spdlog** - Logging Library

**Current Version:** 1.12.0 (GIT commit 7e635fca68d014934b4af8a1cf874f63989352b7)  
**Latest Stable:** 1.16.0 (October 11, 2024)  
**Previous Versions:** 1.15.3, 1.15.2, 1.15.1, 1.15.0, 1.14.1, 1.14.0

**Priority:** HIGH

**Current Issues:**
- **CVE-2025-6140:** Resource exhaustion vulnerability in scopedpadder affecting spdlog up to 1.15.1
  - This vulnerability would affect version 1.12.0
  - Causes excessive resource consumption

**Fixed In:** Version 1.15.2+

**Recommended Versions:**
- **Immediate:** 1.15.2 or later (first version to fix CVE-2025-6140)
- **Current Latest:** 1.16.0 (October 2024)

**What's New in Recent Versions:**
- **v1.16.0** - Ringbuffer sink fixes, TCP timeout support, fmt 12.0.0
- **v1.15.3** - fmt 11.2.0, rotating file sink enhancements
- **v1.15.2** - CVE-2025-6140 fix, fmt 11.1.4
- **v1.15.1** - fmt 11.1.3, wide character support
- **v1.15.0** - fmt 11.0.2, wide character improvements
- **v1.14.1** - C++17/C++11 compatibility fixes
- **v1.14.0** - MDC support, stopwatch milliseconds, async flush improvements

**Benefits of Upgrading:**
- Fixes CVE-2025-6140 resource exhaustion vulnerability
- Better fmt library integration and compatibility
- Improved async logger handling with synchronous flush
- Wide character formatting support
- Rotating file sink improvements

**Migration Effort:** Very Low - Mostly backward compatible

---

### 3. **yaml-cpp** - YAML Parser

**Current Version:** 0.8.0 (GIT commit 2f86d13775d119edbb69af52e5f566fd65c6953b)  
**Latest Stable:** 0.8.0  
**Released:** August 10, 2023

**Priority:** MEDIUM

**Current Status:**
- Version 0.8.0 is the latest stable release
- No unpatched CVEs affecting this version
- Previous vulnerabilities (CVE-2017-5950, CVE-2017-11692, CVE-2018-20573, CVE-2018-20574, CVE-2019-6285, CVE-2019-6292) were fixed in 0.6.3-1 or later
- 0.8.0 includes fixes from much newer versions and is secure

**Why No Upgrade Needed:**
- Already using the latest stable version
- No security vulnerabilities in 0.8.0
- Stable and well-maintained library

**Monitoring:**
- Watch for 0.9.0 release if announced
- Check GitHub releases periodically

---

### 4. **CLI11** - Command Line Parser

**Current Version:** 2.3.2 (header-only)  
**Latest Stable:** 2.6.0 (available on Conan)  
**Released:** Via maintenance updates

**Priority:** MEDIUM

**Current Status:**
- Version 2.3.2 is stable and widely used
- No known CVEs or security vulnerabilities
- Maintenance release with various bug fixes
- Header-only single-file library for easy inclusion

**Recent Updates in 2.x Series:**
- Better ADL support for lexical_cast
- Flag parsing improvements
- Out-of-bounds handling improvements
- Help message formatting fixes

**Upgrade Options:**
- **Current:** 2.3.2 (Working well, no critical issues)
- **Latest:** 2.6.0 (Additional validators in ExtraValidators.hpp, CMake 3.14+ required)

**Migration Effort:** Very Low - Mostly backward compatible

**Recommendation:**
- Current version (2.3.2) is acceptable and stable
- Consider upgrading to 2.6.0 if new features are needed
- No urgent security reason to upgrade

---

### 5. **StormLib** - MPQ Archive Library

**Current Version:** 9.22.0 (per CMakeLists.txt VERSION specification)  
**Latest Stable:** 9.30 (November 2, 2024)

**Priority:** MEDIUM

**Current Status:**
- Version 9.22.0 was released November 10, 2017
- No publicly documented CVEs for this version
- Latest version 9.30 was released November 2, 2024
- Well-maintained project with active development

**What's New in 9.30:**
- 7 years of bug fixes and improvements
- Performance enhancements
- Platform support improvements
- Better compression handling

**Current Usage:**
- Optional dependency (BUILD_STORMLIB OFF by default)
- If enabled, provides MPQ archive processing

**Recommendation:**
- Upgrade to 9.30 when StormLib support is enabled
- For projects not using StormLib (BUILD_STORMLIB=OFF), this is not critical

**Migration Effort:** Low - Generally backward compatible

---

### 6. **tinyxml2** Secondary Note - Already Addressed Above

See **TinyXML2** section - This is critical.

---

### 7. **Bundled/Internal Dependencies**

#### **n64graphics** - N64 Graphics Library
- **Version:** Not explicitly versioned
- **Status:** Internal/custom library
- **Action:** No upgrade needed (project-specific)
- **Priority:** N/A

#### **BinaryTools** - Binary Manipulation Library
- **Version:** Not explicitly versioned
- **Status:** Internal/custom library
- **Action:** No upgrade needed (project-specific)
- **Priority:** N/A

#### **miniz** - Compression Library (zip_file.hpp)
- **Version:** Not explicitly versioned (appears to be integrated)
- **Status:** Header-only utility
- **Action:** No upgrade needed (stable implementation)
- **Priority:** N/A

#### **libgfxd** - Display List Decompiler

**Current Commit:** 96fd3b849f38b3a7c7b7f3ff03c5921d328e6cdf  
**Status:** Fetched via GitHub FetchContent

**Note:**
- Specific released version cannot be determined from commit hash alone
- Repository: https://github.com/glankk/libgfxd
- Consider pinning to a specific release tag when available
- Check GitHub releases page for version information

**Recommendation:**
- Review GitHub for versioning/tagging scheme
- Consider updating to latest stable release once identified

---

### 8. **CMake Build System**

**Current Requirement:** 3.12 (minimum)  
**Recommendation:** 3.20 or higher

**Priority:** MEDIUM

**Issues with CMake 3.12:**
- Basic C++20 support only
- Limited feature enumeration
- No full awareness of all C++20 features
- May cause compilation issues with some C++20 code

**Benefits of CMake 3.20+:**
- Enhanced C++20 support
- Better C++20 feature detection
- Improved compiler compatibility
- Better module system support

**Recommendation:**
- Update CMakeLists.txt to require CMake 3.20+ instead of 3.12
- Ensures better C++20 support matching the project's C++20 standard requirement

**Migration Path:**
```cmake
# Change from:
cmake_minimum_required(VERSION 3.12)

# To:
cmake_minimum_required(VERSION 3.20)
```

---

## Summary Table

| Dependency | Current | Latest | Priority | Action | CVE | Breaking |
|---|---|---|---|---|---|---|
| TinyXML2 | 10.0.0 | 11.0.0 | CRITICAL | Upgrade | 2024-50614, 50615 | Minor (return type) |
| spdlog | 1.12.0 | 1.16.0 | HIGH | Upgrade | CVE-2025-6140 | No |
| yaml-cpp | 0.8.0 | 0.8.0 | LOW | None | None | N/A |
| CLI11 | 2.3.2 | 2.6.0 | MEDIUM | Optional | None | No |
| StormLib | 9.22.0 | 9.30 | MEDIUM | Upgrade if used | None | No |
| libgfxd | (commit) | (unclear) | MEDIUM | Review | None | Unknown |
| CMake | 3.12 | 3.20+ | MEDIUM | Upgrade | N/A | No |

---

## Upgrade Strategy & Recommendations

### Phase 1: Critical (Immediate - 1-2 weeks)

1. **Upgrade TinyXML2 from 10.0.0 to 11.0.0**
   ```cmake
   GIT_TAG 11.0.0  # from: 10.0.0
   ```
   - Fixes CVE-2024-50614 and CVE-2024-50615
   - Search codebase for `size_t` vs `int` return value handling
   - Test all XML parsing functionality

### Phase 2: High Priority (2-4 weeks)

2. **Upgrade spdlog from 1.12.0 to 1.16.0**
   ```cmake
   GIT_TAG v1.16.0  # from: 7e635fca68d014934b4af8a1cf874f63989352b7
   ```
   - Fixes CVE-2025-6140 resource exhaustion vulnerability
   - Minimal code changes required
   - Test logging functionality across all modules

3. **Upgrade CMake requirement from 3.12 to 3.20**
   ```cmake
   cmake_minimum_required(VERSION 3.20)  # from: 3.12
   ```
   - Better C++20 support alignment
   - Update CI/CD pipelines accordingly
   - Update README.md with new CMake requirement

### Phase 3: Medium Priority (1-2 months)

4. **Upgrade StormLib from 9.22.0 to 9.30 (if BUILD_STORMLIB is enabled)**
   ```cmake
   SET(VERSION_MAJOR "9")
   SET(VERSION_MINOR "30")
   ```
   - Benefits from 7 years of improvements
   - Optional: only if MPQ support is actively used

5. **Evaluate CLI11 upgrade to 2.6.0 (optional)**
   - Current version is stable
   - Upgrade only if new features are needed
   - CMake requirement changes to 3.14+

6. **Investigate libgfxd versioning**
   - Contact library maintainer or check releases
   - Pin to specific release version when identified
   - Add documentation to CMakeLists.txt explaining version choice

---

## Testing Recommendations

### TinyXML2 Upgrade Tests
```bash
# Test all XML parsing functionality
# Verify return type changes from int to size_t
# Test with malformed XML files (edge cases)
# Regression testing on all XML parsing code paths
```

### spdlog Upgrade Tests
```bash
# Verify all logging output still works
# Test async logger behavior
# Verify no performance regressions
# Check wide character support (if used)
```

### CMake Upgrade Tests
```bash
# Test compilation on all target platforms:
#   - Windows (MSVC 2022, Clang)
#   - Linux (GCC, Clang)
#   - macOS (Xcode)
#   - Emscripten (web builds)
#   - Nintendo Switch (if applicable)
# Verify C++20 features compile correctly
```

---

## Compiler Support Notes

### Current Requirements (from README)
- **Windows:** Visual Studio 2022 with C++ feature set, MSVC v142
- **Linux:** GCC or Clang with CMake, Ninja-build, libbz2-dev
- **macOS:** Xcode with CMake, Ninja
- **Web:** Emscripten

### Compatibility with Proposed Updates
- **TinyXML2 11.0.0:** Supports all current compilers
- **spdlog 1.16.0:** Supports all current compilers, enhanced fmt library integration
- **CMake 3.20:** Widely available in all toolchain versions
- **StormLib 9.30:** Maintains broad platform support

---

## Cost-Benefit Analysis

### TinyXML2 Upgrade
- **Cost:** Low (minor return type changes)
- **Benefit:** High (fixes 2 CVEs affecting security)
- **Risk:** Very Low
- **Timeline:** 1-2 weeks

### spdlog Upgrade
- **Cost:** Very Low (backward compatible)
- **Benefit:** High (fixes resource exhaustion CVE)
- **Risk:** Very Low
- **Timeline:** 1 week

### CMake Upgrade
- **Cost:** Low (single line change + CI updates)
- **Benefit:** Medium (better C++20 support)
- **Risk:** Low
- **Timeline:** 1-2 weeks

### StormLib Upgrade
- **Cost:** Low (if using this feature)
- **Benefit:** Medium (years of improvements)
- **Risk:** Very Low
- **Timeline:** 2-4 weeks

---

## Documentation Updates Needed

1. **README.md**
   - Update CMake minimum version requirement
   - Note TinyXML2 security update

2. **CMakeLists.txt**
   - Add comments explaining version choices
   - Document why specific commit hashes are used
   - Consider switching to release tags where possible

3. **CONTRIBUTING.md** (if exists)
   - Update development environment requirements
   - Document new CMake version requirement

4. **CHANGELOG.md** (if exists)
   - Document all dependency updates and their rationale

---

## Risks and Mitigations

| Risk | Severity | Mitigation |
|---|---|---|
| TinyXML2 API change (int→size_t) | Low | Code review, regression testing |
| Breaking changes in spdlog | Very Low | Version is backward compatible, test async logging |
| CMake toolchain issues | Low | Test on all target platforms in CI/CD |
| StormLib compatibility | Very Low | Test MPQ operations if feature enabled |

---

## Timeline & Resource Estimate

| Phase | Task | Duration | Priority |
|---|---|---|---|
| 1 | Assess impact of TinyXML2 return type change | 2 days | CRITICAL |
| 1 | Update TinyXML2, test, merge | 5 days | CRITICAL |
| 2 | Update spdlog, test, merge | 3 days | HIGH |
| 2 | Update CMake requirement, test CI/CD | 5 days | HIGH |
| 3 | StormLib evaluation and optional upgrade | 7 days | MEDIUM |
| 3 | libgfxd investigation and versioning | 3 days | MEDIUM |

**Total Estimated Time:** 4-6 weeks (depending on parallel work)

---

## Conclusion

The Torch project has dependencies with known security vulnerabilities that should be addressed promptly. The most critical issue is **TinyXML2 10.0.0** with two CVEs affecting application availability. Combined with **spdlog 1.12.0**'s resource exhaustion vulnerability, these upgrades should be prioritized.

The migration path is straightforward, with most updates being backward compatible and low-risk. A phased approach is recommended, starting with critical security fixes before moving to optional enhancements.
