# Product Requirements Document: pykiteconnect Enhancements

**Version:** 1.0
**Date:** 2024-07-30 (Please update with the current date)
**Author:** AI Assistant

## 1. Introduction

This document outlines a series of proposed improvements, issue fixes, and optimizations for the `pykiteconnect` library (v5.0.1). The goal is to enhance the library's maintainability, modernize its codebase, improve developer experience, and ensure it adheres to current best practices and State-Of-The-Art (SOTA) Python development standards.

## 2. Current Status

`pykiteconnect` is the official Python client for the Kite Connect API, providing REST and WebSocket interfaces for building trading applications. The current version is 5.0.1, which has dropped Python 2.7 support.

## 3. Identified Issues and Areas for Improvement

### 3.1. Code Structure & Maintainability

*   **Issue M1: Large Module Sizes**
    *   **Description:** `kiteconnect/connect.py` (approx. 946 lines) and `kiteconnect/ticker.py` (approx. 864 lines) significantly exceed common module size recommendations (e.g., <300-500 lines), making them harder to navigate, understand, and maintain.
    *   **Proposed Solution:**
        *   Refactor `connect.py`: Consider splitting by API resource groups (e.g., `orders.py`, `portfolio.py`, `market_data.py`, `user.py`) or by functional concerns (e.g., `session_management.py`, `request_builder.py`, `response_parser.py`).
        *   Refactor `ticker.py`: Separate the `Autobahn/Twisted` protocol and factory implementations (`KiteTickerClientProtocol`, `KiteTickerClientFactory`) into their own module(s). Isolate message parsing logic. This would make the main `KiteTicker` class a cleaner interface.
    *   **Impact:** Improved readability, easier maintenance, better separation of concerns.

*   **Issue M2: Python 2 Compatibility Remnants**
    *   **Description:** Despite dropping Python 2.7 support in v5, several Python 2 compatibility constructs remain:
        *   `from __future__ import unicode_literals` in `kiteconnect/__init__.py`.
        *   Usage of the `six` library (e.g., `six.StringIO`, `six.PY2`, `six.moves.urllib.parse`).
        *   Python 2-style `super()` calls: `super(ClassName, self).__init__(...)`.
        *   Comments like `# decode to string for Python 3` in `_parse_instruments` (`connect.py`) suggest legacy logic.
    *   **Proposed Solution:**
        *   Remove `unicode_literals` import (it's default behavior in Py3).
        *   Remove the `six` dependency and replace its usages with Python 3 equivalents (e.g., `io.StringIO`, `urllib.parse`).
        *   Update `super()` calls to the modern Python 3 syntax: `super().__init__(...)`.
        *   Review and refactor any logic specifically tied to Python 2 compatibility.
    *   **Impact:** Reduced dependency footprint, cleaner codebase, alignment with current Python 3 practices.

*   **Issue M3: Outdated String Formatting**
    *   **Description:** Use of old `%` style string formatting (e.g., in `KiteConnect.login_url()`).
    *   **Proposed Solution:** Migrate to f-strings for improved readability, conciseness, and generally better performance.
    *   **Impact:** Modernized code, improved developer ergonomics.

### 3.2. Dependencies

*   **Issue D1: Pinned `autobahn[twisted]==19.11.2`**
    *   **Description:** The WebSocket client relies on an older pinned version of `autobahn` and its `Twisted` dependency. `Twisted` is a substantial framework.
    *   **Proposed Solution/Investigation:**
        *   Investigate reasons for pinning to this specific version.
        *   Explore updating to a more recent stable version of `autobahn[twisted]`.
        *   For a future major version, evaluate the feasibility of migrating to a lighter, `asyncio`-native WebSocket library like `websockets` to potentially reduce dependency overhead and align with modern Python async practices. This would be a significant undertaking.
    *   **Impact:** Potentially newer features, security fixes from updated dependencies. A switch to `asyncio`-based websockets could simplify the async story for users.

*   **Issue D2: Pinned `flake8 <= 4.0.1`**
    *   **Description:** `flake8` is pinned to an older major version (v4.0.1), while v6+ is available.
    *   **Proposed Solution:** Upgrade `flake8` to the latest stable version in `dev_requirements.txt`. Address any new linting errors/warnings.
    *   **Impact:** Improved code quality checks, leveraging latest linting rules.

*   **Issue D3: `mock` Dependency**
    *   **Description:** `mock>=3.0.5` is listed in `dev_requirements.txt`. Python 3.3+ includes `unittest.mock` in the standard library.
    *   **Proposed Solution:** Since the library targets Python 3.5+, remove the `mock` dependency and refactor tests to use `unittest.mock`.
    *   **Impact:** Reduced development dependency.

### 3.3. Code Modernization & SOTA Practices

*   **Issue S1: Lack of Type Hinting**
    *   **Description:** The codebase currently lacks Python type hints.
    *   **Proposed Solution:** Incrementally introduce type hints across the library, starting with public APIs and then internal functions/methods. Utilize tools like MyPy for static type checking in the CI/development process.
    *   **Impact:** Significantly improved code clarity, early bug detection, enhanced IDE support, better maintainability. This is a standard SOTA practice.

*   **Issue S2: Synchronous REST Client (`KiteConnect`)**
    *   **Description:** The `KiteConnect` client for REST APIs is synchronous (uses `requests`). This can be a bottleneck in modern `asyncio`-based applications, especially when used alongside the async-capable `KiteTicker`.
    *   **Proposed Solution (Major Enhancement / SOTA):**
        *   Develop an asynchronous version of `KiteConnect`, possibly named `AsyncKiteConnect`, using libraries like `aiohttp` or `httpx`.
        *   This would provide a non-blocking API client, improving performance and integration in async environments.
        *   This would be a significant feature, likely for a new major version or as an explicitly separate offering.
    *   **Impact:** Enables fully asynchronous trading applications, better performance in I/O-bound operations, alignment with modern Python async paradigms.

*   **Issue S3: Broad Exception Suppression for Warnings**
    *   **Description:** `requests.packages.urllib3.disable_warnings()` in `connect.py` globally suppresses `urllib3` warnings.
    *   **Proposed Solution:**
        *   Investigate the specific warnings being suppressed.
        *   If they relate to SSL and are intentional due to `disable_ssl=True`, consider more targeted suppression using `warnings.catch_warnings` or `warnings.filterwarnings` for specific warning types or modules.
    *   **Impact:** More robust error/warning handling, prevents masking potentially useful warnings.

*   **Issue S4: Usage of String Constants for Enumerated Values**
    *   **Description:** Many string constants are defined in `KiteConnect` for things like product types, order types, exchanges (e.g., `KiteConnect.PRODUCT_MIS = "MIS"`).
    *   **Proposed Solution:** Convert these groups of related constants to Python `Enum` types (e.g., `class ProductType(str, Enum): MIS = "MIS"; CNC = "CNC"`). Using `str, Enum` allows them to still be used as strings where needed.
    *   **Impact:** Improved type safety, better IDE autocompletion, reduced risk of typos, more expressive code.

*   **Issue S5: Deprecated `e.message` in Examples**
    *   **Description:** Example code in README and `__init__.py` docstrings use `e.message` for exception details, which is deprecated.
    *   **Proposed Solution:** Update examples to use `str(e)` to get the exception message.
    *   **Impact:** Adherence to current Python practices, ensures examples work correctly in future Python versions.

### 3.4. Documentation & User Experience

*   **Issue UX1: Potentially Outdated Documentation Links**
    *   **Description:** README links to `pykiteconnect/v4` documentation, while the library is v5.0.1. The main API docs link to `/v3`.
    *   **Proposed Solution:** Verify all documentation links and update them to point to the correct and latest versions.
    *   **Impact:** Ensures users access up-to-date documentation.

*   **Issue UX2: Windows Installation Instructions**
    *   **Description:** README provides specific MSVC++ compiler details for older Python versions. This section could be simplified for current Python 3 versions.
    *   **Proposed Solution:** Review and update the Windows installation guidance to be more relevant for Python 3.5+, potentially recommending Python installers that bundle build tools or direct users to general Python C extension build advice for Windows.
    *   **Impact:** Clearer and more helpful installation instructions for Windows users.

## 4. Proposed Enhancements (Summary)

1.  **Refactor Large Modules:** Split `connect.py` and `ticker.py`.
2.  **Remove Python 2 Relics:** Eliminate `six`, update `super()`, remove outdated `__future__` imports.
3.  **Modernize String Formatting:** Use f-strings.
4.  **Dependency Updates:**
    *   Investigate/update `autobahn[twisted]`.
    *   Upgrade `flake8`.
    *   Remove `mock` for `unittest.mock`.
5.  **Introduce Type Hinting:** Add type annotations throughout the codebase.
6.  **Consider Async REST Client:** Plan/develop an `async` version of `KiteConnect`.
7.  **Refine Warning Suppression:** Make `urllib3` warning suppression more specific.
8.  **Use Enums for Constants:** Convert string constants to `Enum` types.
9.  **Update Example Exception Handling:** Use `str(e)` instead of `e.message`.
10. **Verify/Update Documentation Links.**
11. **Simplify Windows Installation Guide.**

## 5. Prioritization (Suggested)

*   **High Priority (Core Maintainability & Modernization):**
    *   M2: Remove Python 2 Relics
    *   M3: Modernize String Formatting
    *   D2: Upgrade `flake8`
    *   D3: Remove `mock`
    *   S5: Update Example Exception Handling
    *   UX1: Verify/Update Documentation Links
*   **Medium Priority (Significant Improvements & SOTA):**
    *   M1: Refactor Large Modules (can be iterative)
    *   S1: Introduce Type Hinting (can be iterative)
    *   S3: Refine Warning Suppression
    *   S4: Use Enums for Constants
    *   UX2: Simplify Windows Installation Guide
*   **Low Priority / Future Consideration (Major Enhancements):**
    *   D1: `autobahn` update/alternative investigation
    *   S2: Async REST Client (`AsyncKiteConnect`)

## 6. Expected Impact

Implementing these changes will result in:
*   A more maintainable, readable, and robust codebase.
*   Reduced technical debt.
*   Better developer experience for users of the library.
*   Alignment with modern Python development practices and SOTA standards.
*   Improved performance and scalability, especially with the introduction of async capabilities.

## 7. Next Steps

*   Review and approve this PRD.
*   Create issues on the GitHub repository for trackable tasks based on this PRD.
*   Begin implementation, starting with high-priority items.

