<!-- Source: SAP Clean Core Extensibility White Paper (2024), SAP/abap-cloud-development-landscape-cheat-sheet -->
<!-- Reference: ADT API patterns from aws-solutions-library-samples/guidance-for-deploying-sap-abap-accelerator-for-amazon-q-developer -->
<!-- Created: 2026-03-27 -->

# Clean Core Remediation Patterns

Remediation strategies for ABAP objects with ATC clean core findings.
Organized by priority: always try the highest-ranked approach first.

## Clean Core Levels (from ATC Priority)

| ATC Priority | Level | Description | Cloud Impact |
|---|---|---|---|
| 0 (no findings) | A | Fully Clean — released APIs only | Ready for cloud |
| 3 (information) | B | Pragmatically Clean — classic APIs | Cloud-compatible with caveats |
| 2 (warning) | C | Conditionally Clean — internal objects | Needs review before cloud |
| 1 (error) | D | Not Clean — blocked APIs | Blocks cloud migration |

**Rule**: Object level = worst finding. One priority-1 finding = Level D.

## Remediation Hierarchy

For each Level D finding, work through this hierarchy top-to-bottom.
Stop at the first approach that resolves the finding.

### 1. Released API Replacement (Target: Level A)

The finding references an API that has a released successor.
Replace the non-released call with the released alternative.

**How to find the successor:**
1. Check the ATC finding's `successor` field (from SAP API reference data)
2. Search `objectReleaseInfoLatest.json` for the deprecated object — the `successor` field names the replacement
3. Search `objectReleaseInfoLatest.json` by `applicationComponent` for released alternatives in the same functional area

**Example — Direct table access → CDS view:**
```abap
" BEFORE (Level D): Direct read on SAP table
SELECT * FROM usr02 INTO TABLE @DATA(lt_users)
  WHERE bname LIKE 'Z%'.

" AFTER (Level A): Released CDS entity
SELECT * FROM I_BusinessUserBasic INTO TABLE @DATA(lt_users)
  WHERE UserID LIKE 'Z%'.
```

**Example — Obsolete statement → Modern equivalent:**
```abap
" BEFORE (Level D): DESCRIBE FIELD (obsolete)
DESCRIBE FIELD lv_field LENGTH lv_len IN CHARACTER MODE.

" AFTER (Level A): RTTI
DATA(lo_type) = cl_abap_typedescr=>describe_by_data( lv_field ).
DATA(lv_len) = CAST cl_abap_elemdescr( lo_type )->output_length.
```

### 2. Classic API / BAPI Replacement (Target: Level C)

No released successor exists, but a classic API (BAPI, function module)
provides equivalent functionality. D→C is a valid remediation outcome —
the object moves from "blocks cloud" to "conditionally clean."

**Common patterns:**
- Direct table MODIFY → BAPI (e.g., `MODIFY usr02` → `BAPI_USER_CHANGE`)
- Kernel calls → Function modules (e.g., `CALL 'SYSTEM'` → `cl_abap_syst=>get_*`)
- Dynamic code generation → Factory patterns

**Example — Direct table modify → BAPI:**
```abap
" BEFORE (Level D): Direct table modification
MODIFY usr02 FROM ls_user.

" AFTER (Level C): BAPI replacement
DATA: lt_return TYPE TABLE OF bapiret2,
      lv_bname TYPE bapibname-bapibname VALUE 'TESTUSER'.

CALL FUNCTION 'BAPI_USER_CHANGE'
  EXPORTING
    username = lv_bname
  TABLES
    return   = lt_return.

READ TABLE lt_return WITH KEY type = 'E' TRANSPORTING NO FIELDS.
IF sy-subrc <> 0.
  CALL FUNCTION 'BAPI_TRANSACTION_COMMIT'.
ELSE.
  CALL FUNCTION 'BAPI_TRANSACTION_ROLLBACK'.
ENDIF.
```

**BAPI parameter types matter:**
Always use typed variables matching the BAPI's parameter definitions.
String literals cause `CX_SY_DYN_CALL_ILLEGAL_TYPE` at runtime.
Check the function module signature for the correct DDIC type.

### 3. Wrapper Pattern (Target: Level C or B)

No released or classic successor exists. SAP's official recommendation:
encapsulate the non-released call in a custom wrapper class, expose it
as a released custom API, and replace all direct references.

**When to use:**
- The non-released API has no successor in SAP's Cloudification Repository
- Multiple callers use the same non-released API (consolidate in one place)
- The API is deeply embedded in business logic and needs isolation

**Wrapper pattern steps:**

1. **Create wrapper interface** — Define a clean interface with only the
   operations the callers need (not a 1:1 mirror of the SAP API)
2. **Create wrapper class** — Implement the interface, encapsulate the
   non-released call inside
3. **Set release contract** — Mark the wrapper as C1 (released for use
   in same software component) via `@EndUserText.label` and release settings
4. **Replace all callers** — Change direct API calls to use the wrapper
5. **Document** — Add `"TODO: Replace with released API when available`
   comment in the wrapper implementation

**Example — READ REPORT wrapper:**
```abap
" Step 1: Interface
INTERFACE zif_source_reader PUBLIC.
  METHODS read_source
    IMPORTING iv_program    TYPE sy-repid
    RETURNING VALUE(rt_source) TYPE string_table
    RAISING   cx_sy_read_src_line_too_long.
ENDINTERFACE.

" Step 2: Wrapper class
CLASS zcl_source_reader DEFINITION PUBLIC FINAL
  CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES zif_source_reader.
ENDCLASS.

CLASS zcl_source_reader IMPLEMENTATION.
  METHOD zif_source_reader~read_source.
    " Encapsulated non-released call — single point of change
    " TODO: Replace with released API when available
    READ REPORT iv_program INTO rt_source.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE cx_sy_read_src_line_too_long.
    ENDIF.
  ENDMETHOD.
ENDCLASS.

" Step 4: Callers use the wrapper
DATA(lo_reader) = NEW zcl_source_reader( ).
DATA(lt_source) = lo_reader->zif_source_reader~read_source( 'ZTEST_PROG' ).
```

**Example — Dynamic code generation wrapper:**
```abap
INTERFACE zif_code_generator PUBLIC.
  METHODS generate_and_run
    IMPORTING it_source TYPE string_table
    RETURNING VALUE(rv_program) TYPE sy-repid
    RAISING   cx_sy_generate_subpool.
ENDINTERFACE.

CLASS zcl_code_generator DEFINITION PUBLIC FINAL
  CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES zif_code_generator.
ENDCLASS.

CLASS zcl_code_generator IMPLEMENTATION.
  METHOD zif_code_generator~generate_and_run.
    " Encapsulated non-released call
    " TODO: Replace with released API when available
    GENERATE SUBROUTINE POOL it_source NAME rv_program.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE cx_sy_generate_subpool.
    ENDIF.
  ENDMETHOD.
ENDCLASS.
```

**Wrapper creation via ADT API:**
New ABAP classes can be created programmatically through ADT REST endpoints:
- POST to `/sap/bc/adt/oo/classes` with XML body containing name, description, package
- Write source via PUT to `/sap/bc/adt/oo/classes/{name}/source/main`
- Write test classes via PUT to `/sap/bc/adt/oo/classes/{name}/includes/testclasses`
- Lock/write/unlock/activate lifecycle applies to new objects same as existing ones

### 4. Architectural Review (No Automated Fix)

The non-released API serves a function that requires architectural redesign
for cloud compatibility. Examples:

- `CALL TRANSACTION` → Requires UI redesign (Fiori app or RAP BO)
- File system access (`OPEN DATASET`) → Requires cloud storage integration
- RFC destinations → Requires Communication Arrangement setup
- Background jobs (`JOB_OPEN/CLOSE`) → Requires Application Jobs framework

**Agent action:** Document the finding with a `" FIXME: Requires architectural review` comment.
Do not attempt automated remediation — these need human design decisions.

## Common Level D Patterns

Quick reference for the most frequently seen priority-1 findings:

| Pattern | Typical Finding | Remediation | Target Level |
|---|---|---|---|
| Direct SAP table read | `SELECT FROM <sap_table>` | Released CDS entity | A |
| Direct SAP table modify | `MODIFY/UPDATE/INSERT <sap_table>` | BAPI or RAP BO | C |
| Obsolete statements | `DESCRIBE FIELD`, `COMPUTE`, `OCCURS` | Modern ABAP equivalents | A |
| Dynamic code generation | `GENERATE SUBROUTINE POOL`, `INSERT REPORT` | Wrapper or redesign | C or review |
| Kernel calls | `CALL 'SYSTEM'`, `CALL 'ENQUEUE'` | Released class methods | A or C |
| Direct function calls | Non-released FM usage | Released FM or wrapper | A or C |
| File system access | `OPEN DATASET`, `READ DATASET` | Cloud storage API | Review |
| UI statements | `CALL SCREEN`, `CALL TRANSACTION` | Fiori/RAP redesign | Review |

## D→C as Valid Outcome

Some replacements are themselves Level C (classic/internal APIs).
This is expected and acceptable when:

- The BAPI or function module is the **best available** replacement
- A fully released alternative would require **architecture redesign**
  (e.g., direct table modify → BAPI is Level C; Level A requires IAS/SCIM)
- The object moves from **"blocks cloud migration"** to **"needs review"**

D→C removes the migration blocker. The remaining Level C findings
can be addressed in a separate phase or accepted as pragmatic technical debt.

## Fix Verification Tests

After applying a fix, write a targeted test that verifies the replacement works:

```abap
CLASS ltcl_fix_verification DEFINITION FINAL FOR TESTING
  DURATION SHORT
  RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS test_replacement_callable FOR TESTING.
ENDCLASS.

CLASS ltcl_fix_verification IMPLEMENTATION.
  METHOD test_replacement_callable.
    " Call the replacement API with safe test data
    " Verify it returns without runtime exceptions
    " Assert expected error handling (e.g., user not found → error in RETURN table)
    DATA: lt_return TYPE TABLE OF bapiret2.
    CALL FUNCTION 'BAPI_USER_CHANGE'
      EXPORTING username = 'NONEXISTENT_USER'
      TABLES   return   = lt_return.
    CALL FUNCTION 'BAPI_TRANSACTION_ROLLBACK'.
    cl_abap_unit_assert=>assert_not_initial( act = lt_return ).
  ENDMETHOD.
ENDCLASS.
```

**Test purpose:** Catch runtime type mismatches, missing authorizations,
and API behavior differences that only surface at execution time.
The ZTEST_LEVEL_D remediation proved this — `CX_SY_DYN_CALL_ILLEGAL_TYPE`
was caught by the fix-verification test, not by syntax check or ATC.