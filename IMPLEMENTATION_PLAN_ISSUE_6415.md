# Implementation Plan: Millisecond Granularity for DateField and TimeField

**Issue:** https://github.com/adobe/react-spectrum/issues/6415

## Overview

The goal is to add `granularity="millisecond"` support to `DateField` and `TimeField` components, allowing users to edit millisecond values directly in the date/time picker.

## Key Findings

1. **Underlying data structures already support milliseconds**: `Time`, `CalendarDateTime`, and `ZonedDateTime` in `@internationalized/date` all have a `millisecond` property and support `cycle('millisecond', ...)` operations.

2. **`Intl.DateTimeFormat` supports `fractionalSecondDigits`**: Modern browsers support the `fractionalSecondDigits` option (values 1-3) in `Intl.DateTimeFormat`, which controls fractional second digit display.

3. **The granularity system uses ordered keys**: The `getFormatOptions` function uses the ordered keys `['year', 'month', 'day', 'hour', 'minute', 'second']` to determine which segments to display. We need to add `'millisecond'` to this chain.

4. **Custom segment handling needed**: Since `Intl.DateTimeFormat.formatToParts()` returns fractional seconds as part of the `'fractionalSecond'` type, we need custom handling in the segment processing logic.

---

## Implementation Steps

### Phase 1: Type Definitions

**File: `packages/@react-types/datepicker/src/index.d.ts`**

1. Update the `Granularity` type (line 39):
   ```typescript
   export type Granularity = 'day' | 'hour' | 'minute' | 'second' | 'millisecond';
   ```

2. Update the `TimePickerProps` granularity type (line 164):
   ```typescript
   granularity?: 'hour' | 'minute' | 'second' | 'millisecond',
   ```

---

### Phase 2: Core State Management

**File: `packages/@react-stately/datepicker/src/utils.ts`**

1. Update `FieldOptions` type (line 131) to include `fractionalSecondDigits`:
   ```typescript
   export type FieldOptions = Pick<Intl.DateTimeFormatOptions, 'year' | 'month' | 'day' | 'hour' | 'minute' | 'second'> & {
     fractionalSecondDigits?: 1 | 2 | 3
   };
   ```

2. Update `DEFAULT_FIELD_OPTIONS` and `TWO_DIGIT_FIELD_OPTIONS` to include millisecond:
   ```typescript
   const DEFAULT_FIELD_OPTIONS: FieldOptions = {
     year: 'numeric',
     month: 'numeric',
     day: 'numeric',
     hour: 'numeric',
     minute: '2-digit',
     second: '2-digit',
     fractionalSecondDigits: 3  // for millisecond granularity
   };
   ```

3. Update `getFormatOptions()` function (lines 160-203):
   - Add 'millisecond' to the keys array for granularity lookup
   - When granularity is 'millisecond', add `fractionalSecondDigits: 3` to the options
   - Update the `hasTime` check to include 'millisecond'

4. Update `useDefaultProps()` function to support millisecond granularity detection.

---

**File: `packages/@react-stately/datepicker/src/useDateFieldState.ts`**

1. Update `SegmentType` type (line 22) to include `'fractionalSecond'`:
   ```typescript
   export type SegmentType = 'era' | 'year' | 'month' | 'day' | 'hour' | 'minute' | 'second' | 'fractionalSecond' | 'dayPeriod' | 'literal' | 'timeZoneName';
   ```

2. Update `EDITABLE_SEGMENTS` constant (lines 103-112) to include `fractionalSecond`:
   ```typescript
   const EDITABLE_SEGMENTS = {
     year: true,
     month: true,
     day: true,
     hour: true,
     minute: true,
     second: true,
     fractionalSecond: true,  // New
     dayPeriod: true,
     era: true
   };
   ```

3. Update `PAGE_STEP` constant (lines 114-121):
   ```typescript
   const PAGE_STEP = {
     year: 5,
     month: 2,
     day: 7,
     hour: 2,
     minute: 15,
     second: 15,
     fractionalSecond: 100  // Step by 100ms
   };
   ```

4. Update `TYPE_MAPPING` to map `'fractionalSecond'` if needed.

5. Update `getSegmentLimits()` function (lines 498-567) to handle `fractionalSecond`:
   ```typescript
   case 'fractionalSecond':
     return {
       value: date.millisecond,
       minValue: 0,
       maxValue: 999
     };
   ```

6. Update `addSegment()` function (lines 569-596) to handle `fractionalSecond`:
   ```typescript
   case 'fractionalSecond':
     return value.cycle('millisecond', amount, { round: true });
   ```

7. Update `setSegment()` function (lines 598-638) to handle `fractionalSecond`:
   ```typescript
   case 'fractionalSecond':
     return value.set({ millisecond: segmentValue });
   ```

8. Update `processSegments()` function (lines 430-496):
   - Add `'fractionalSecond'` to the `timeValue` array for RTL handling
   - Handle the granularity check for millisecond at the end of time segments

---

**File: `packages/@react-stately/datepicker/src/useTimeFieldState.ts`**

1. Update the default granularity logic (line 75) if needed - the default remains 'minute', but 'millisecond' should be a valid option.

---

### Phase 3: Placeholder Support

**File: `packages/@react-stately/datepicker/src/placeholders.ts`**

1. Update `getPlaceholder()` function to handle `fractionalSecond`:
   ```typescript
   export function getPlaceholder(field: string, value: string, locale: string): string {
     if (field === 'era' || field === 'dayPeriod') {
       return value;
     }
     if (field === 'year' || field === 'month' || field === 'day') {
       return placeholders.getStringForLocale(field, locale);
     }
     // For time fields including fractionalSecond, use dashes
     // For milliseconds, use three dashes to indicate 3-digit field
     if (field === 'fractionalSecond') {
       return '–––';
     }
     return '––';
   }
   ```

---

### Phase 4: ARIA and Accessibility

**File: `packages/@react-aria/datepicker/src/useDateSegment.ts`**

1. Review and update to ensure the `fractionalSecond` segment is properly labeled for screen readers
2. Add appropriate aria-label for the millisecond segment (e.g., "millisecond, 0 to 999")

**Files: Internationalization messages**
- `packages/@react-stately/datepicker/intl/*.json` - Add translations for "millisecond" field label if required

---

### Phase 5: Documentation Updates

**Files to update:**
- `packages/@react-aria/datepicker/docs/useDateField.mdx`
- `packages/@react-aria/datepicker/docs/useTimeField.mdx`
- `packages/@react-spectrum/datepicker/docs/DateField.mdx`
- `packages/@react-spectrum/datepicker/docs/TimeField.mdx`

Add documentation for:
- The new `granularity="millisecond"` option
- Example usage with millisecond precision
- Browser compatibility notes for `fractionalSecondDigits`

---

### Phase 6: Testing

**Files to create/update:**
- `packages/@react-spectrum/datepicker/test/TimeField.test.js`
- `packages/@react-spectrum/datepicker/test/DateField.test.js`
- `packages/@react-aria/datepicker/test/useDatePicker.test.tsx`

Add tests for:
1. Rendering with `granularity="millisecond"`
2. Incrementing/decrementing millisecond segment
3. Setting millisecond values directly
4. Page step (incrementing by 100ms)
5. Keyboard navigation in millisecond segment
6. Placeholder display for millisecond segment
7. RTL text direction handling with milliseconds
8. Value validation with millisecond precision
9. Integration with `CalendarDateTime` and `ZonedDateTime` values

---

## Technical Considerations

1. **Browser Compatibility**: `fractionalSecondDigits` is supported in Chrome 84+, Firefox 84+, Safari 14.1+, Edge 84+. Consider adding a fallback for older browsers.

2. **Segment Type Mapping**: The `Intl.DateTimeFormat.formatToParts()` returns `'fractionalSecond'` type for milliseconds (not `'millisecond'`), so we need to map this correctly.

3. **RTL Languages**: The existing RTL handling wraps time segments in LRI/PDI Unicode markers. We need to extend this to include the millisecond segment when `granularity="millisecond"`.

4. **Value Coercion**: When setting a value with `Time(12, 30, 45)` (no millisecond), the component should default milliseconds to 0.

5. **Display Format**: Milliseconds should display with leading zeros (e.g., "005" for 5ms) to maintain consistent 3-digit width.

---

## File Change Summary

| File | Changes |
|------|---------|
| `@react-types/datepicker/src/index.d.ts` | Update `Granularity` and `TimePickerProps` types |
| `@react-stately/datepicker/src/utils.ts` | Update `FieldOptions`, `getFormatOptions()`, `useDefaultProps()` |
| `@react-stately/datepicker/src/useDateFieldState.ts` | Update `SegmentType`, `EDITABLE_SEGMENTS`, `PAGE_STEP`, `getSegmentLimits()`, `addSegment()`, `setSegment()`, `processSegments()` |
| `@react-stately/datepicker/src/useTimeFieldState.ts` | Verify millisecond granularity support |
| `@react-stately/datepicker/src/placeholders.ts` | Update `getPlaceholder()` for fractionalSecond |
| `@react-aria/datepicker/src/useDateSegment.ts` | Update ARIA labels for millisecond segment |
| `@react-stately/datepicker/intl/*.json` | Add millisecond translations (if needed) |
| Documentation files | Add millisecond granularity documentation |
| Test files | Add comprehensive tests |

---

## Estimated Complexity

- **Type definitions**: Low complexity
- **Core state management**: Medium complexity (main changes)
- **Placeholder support**: Low complexity
- **ARIA/Accessibility**: Low complexity
- **Documentation**: Low complexity
- **Testing**: Medium complexity

The main complexity lies in correctly integrating with `Intl.DateTimeFormat`'s `fractionalSecondDigits` option and ensuring the segment type mapping works correctly between the browser's `formatToParts()` output and the internal segment handling.
