# Java Date and Time API

## Why `java.time` Replaced the Legacy API

The old `java.util.Date` and `Calendar` are notoriously error-prone: they are **mutable** (not thread-safe), have **zero-based months** (January = 0), mix date and time confusingly, and `SimpleDateFormat` is not thread-safe. Java 8 introduced **`java.time`** (JSR-310, inspired by Joda-Time): a comprehensive, **immutable**, thread-safe, clearly-typed API. Use it for all new code.

## The Core Types

Types are split by whether they carry a date, a time, and/or a zone:

| Type | Represents | Example |
| --- | --- | --- |
| `LocalDate` | date, no time/zone | `2026-07-15` |
| `LocalTime` | time, no date/zone | `13:45:30` |
| `LocalDateTime` | date + time, no zone | `2026-07-15T13:45` |
| `Instant` | a point on the UTC timeline | `2026-07-15T11:45:30Z` |
| `ZonedDateTime` | date + time + zone | `...+01:00 Europe/London` |
| `OffsetDateTime` | date + time + UTC offset | `...+05:30` |
| `Duration` | time-based amount (seconds/nanos) | `PT2H30M` |
| `Period` | date-based amount (years/months/days) | `P1Y2M10D` |

```java
LocalDate today = LocalDate.now();
LocalDate release = LocalDate.of(2026, Month.JULY, 15);   // months are 1-based / enum
LocalDateTime meeting = today.atTime(14, 30);
Instant now = Instant.now();                               // machine timestamp (UTC)
```

## Immutability and Fluent Manipulation

Every type is immutable — methods return new instances, so they're inherently thread-safe. Manipulation is fluent and readable:

```java
LocalDate nextFriday = today.plusWeeks(1)
                            .with(TemporalAdjusters.next(DayOfWeek.FRIDAY));
LocalDate firstOfMonth = today.withDayOfMonth(1);
boolean before = release.isBefore(today);
```

`TemporalAdjusters` handles "last day of month", "next Tuesday", etc.

## `Instant`, `ZonedDateTime`, and Time Zones

* **`Instant`** is a timestamp on the UTC timeline — best for logging, storage, and comparing moments across zones.
* **`ZonedDateTime`** applies a `ZoneId` (region rules, including daylight saving) to a moment, for human-facing local times.
* **`OffsetDateTime`** carries a fixed UTC offset without full zone rules.

```java
ZoneId london = ZoneId.of("Europe/London");
ZonedDateTime local = instant.atZone(london);
ZonedDateTime tokyo = local.withZoneSameInstant(ZoneId.of("Asia/Tokyo")); // same moment
```

Zone rules (from the IANA tz database) handle daylight saving automatically — a gap in spring and an overlap in autumn — which manual offset math gets wrong. Store timestamps as `Instant`/UTC and convert to a zone only for display.

## `Duration` vs `Period`

* **`Duration`** measures *time* (hours, minutes, seconds, nanos) — use with `Instant`/`LocalTime`.
* **`Period`** measures *dates* (years, months, days) — use with `LocalDate`.

The distinction matters across daylight-saving boundaries: adding a `Period` of one day keeps the local wall-clock time, whereas adding a `Duration` of 24 hours adds exactly 24 hours of elapsed time (which may land on a different wall-clock time).

```java
Duration elapsed = Duration.between(start, end);   // Instants/times
Period age = Period.between(birthDate, today);      // LocalDates
```

## Formatting and Parsing

`DateTimeFormatter` is **immutable and thread-safe** (unlike `SimpleDateFormat`), so a single instance can be shared and reused.

```java
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
String text = meeting.format(fmt);
LocalDateTime parsed = LocalDateTime.parse("2026-07-15 14:30", fmt);

meeting.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);          // built-in
DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM);          // locale-aware
```

## Interoperating with Legacy Code

Bridge methods convert between old and new:

```java
Instant i = new Date().toInstant();
Date d = Date.from(instant);
LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());
```

When migrating, keep `java.time` at the core and convert only at legacy boundaries.

## Common Pitfalls

* Don't use `LocalDateTime` for a moment in time — it has no zone, so two `LocalDateTime`s in different zones aren't comparable as instants. Use `Instant`/`ZonedDateTime`.
* Reuse a `DateTimeFormatter`; never share a `SimpleDateFormat` across threads.
* Prefer `Instant`/UTC for storage and transport; apply zones only for display.
* Remember `Period` (dates) vs `Duration` (time) semantics across DST.

## Interview Q&A

**Q: Why was `java.time` introduced?**
A: The legacy `Date`/`Calendar` were mutable, not thread-safe, had zero-based months, and `SimpleDateFormat` wasn't thread-safe. `java.time` (JSR-310) is immutable, thread-safe, and clearly typed.

**Q: `LocalDateTime` vs `ZonedDateTime` vs `Instant`?**
A: `LocalDateTime` is date+time with no zone; `ZonedDateTime` adds full zone rules (DST) for human-local times; `Instant` is a point on the UTC timeline for timestamps and storage.

**Q: `Duration` vs `Period`?**
A: `Duration` is time-based (hours/minutes/seconds/nanos); `Period` is date-based (years/months/days). They differ across DST boundaries — 24 hours vs one calendar day.

**Q: Why is `DateTimeFormatter` better than `SimpleDateFormat`?**
A: It's immutable and thread-safe, so one instance can be shared safely; `SimpleDateFormat` is mutable and causes data corruption when shared across threads.

**Q: How do you store and display times correctly?**
A: Store as `Instant`/UTC and convert to a `ZonedDateTime` in the user's zone only for display, letting the tz database handle DST.

**Q: Are `java.time` types thread-safe?**
A: Yes — they're immutable, so every operation returns a new instance and they can be shared freely across threads.

**Q: How do you convert between legacy `Date` and `java.time`?**
A: `date.toInstant()` and `Date.from(instant)`, plus `LocalDateTime.ofInstant(instant, zone)` — convert at boundaries and keep `java.time` internally.

**Q: What handles daylight saving automatically?**
A: `ZonedDateTime` with a region `ZoneId` uses IANA tz rules to adjust for DST gaps and overlaps, which manual offset arithmetic gets wrong.

**Q: Why not use `LocalDateTime` for an event timestamp?**
A: It lacks a zone, so it doesn't identify a unique moment; comparisons across zones are meaningless. Use `Instant` or `ZonedDateTime`.

## Interview Notes

* Explain why the legacy API was replaced (mutability, thread safety, zero-based months).
* Map each core type to what it represents (date/time/zone/instant/amount).
* Distinguish `Duration` vs `Period` and their DST implications.
* Stress `DateTimeFormatter` thread safety vs `SimpleDateFormat`.
* Recommend storing `Instant`/UTC and converting to zones only for display.
* Know the legacy interop conversion methods.
