# `datetime` extension

## Related issues and PRs

- Reference Issues:
- Implementation PR(s):

## Timeline

- Started: 2024-08-08

## Summary

We currently support extension functions for IP addresses and decimal values.
A popular request has been support for dates and times as well, but there are several complexities here (e.g., timezones and leap years) that have delayed work on this.
Recall that for anything we add to Cedar, we want to be able to formally model in Lean using our verification-guided development approach (see [this blog post](https://www.amazon.science/blog/how-we-built-cedar-with-automated-reasoning-and-differential-testing)).
We also want to support a decidable SMT-based analysis (see [this paper](https://dl.acm.org/doi/10.1145/3649835)).
The goal of this RFC is to narrow in on a set of date/time related features that are useful in practice, but still feasible to implement in Cedar given these constraints.

## Basic example

This RFC would support a policy like the following, which allows a user to access materials in the "device_prototypes" folder only if they have a sufficiently high job level and a tenure of more than one year.

```cedar
permit(
  principal is User,
  action == Action::"view",
  resource in Folder::"device_prototypes"
)
when {
  principal.department == "HardwareEngineering" &&
  principal.jobLevel >= 10 &&
  context.currentTime.since(principal.hireDate).days() >= 365
};
```

## Motivation

Here are some of the applications we've seen that have prompted requests for this functionality:

- Apply certain rules after some day in future. For instance, say that some new legislation is going into effect at that date.
- Check whether an action is occurring is in certain time frame, like during working hours (Monday - Friday, 9am - 5pm).

Previously for these types of applications, our suggestion has been to use Unix timestamps (see Alt. A) or to pass the result of time computations in the context (see Alt. B).
But these are not ideal solutions because Cedar policies are intended to _expressive_,  _readable_, and _auditable_.
Unix timestamps fail the readability criteria: an expression like `context.now < 1723000000` is difficult to read and understand without additional tooling to decode the timestamp.
Unix timestamps are also indistinguishable from any other integral value providing no additional help in expressing the abstraction of time.
Passing in pre-computed values in the context (e.g., a field like `context.isWorkingHours`) makes authorization logic difficult to audit because it moves this logic outside of Cedar policies and into the calling application.
It is also difficult for the calling application to predict the necessary pre-computed values that a policy writer requires for their intended purpose. These may change over time, and may also differ depending on the principal, action, and/or resource.

### Additional examples

**Only allow user "alice" to view JPEG photos for one week after creation.**

```cedar
permit(
  principal == User::"alice",
  action == PhotoOp::"view",
  resource is Photo
) when {
  resource.fileType == "JPEG" &&
  context.currentTime.since(resource.creationDate).days() <= 7
};
```

**Allow access from a certain IP address only between 9am and 6pm UTC.**

```cedar
permit(
  principal,
  action == Action::"access",
  resource
) when {
  context.srcIp.isInRange(ip("192.168.1.0/24")) &&
  9 <= context.currentTime.hours() &&
  context.currentTime.hours() < 18
};
```

**Allow access from a certain IP address only between 9am and 6pm localized to their time.**

```cedar
permit(
  principal,
  action == Action::"access",
  resource
) when {
  context.srcIp.isInRange(ip("192.168.1.0/24")) &&
  9 <= context.currentTime.offset(principal.tzOffset).hours() &&
  context.currentTime.offset(principal.tzOffset).hours() < 18
};
```

**Prevent employees from accessing work documents on the weekend.**

```cedar
forbid(
  principal,
  action == Action::"access",
  resource is Document
) when {
  [1, 7].contains(context.currentTime.dayOfWeek())
};
```

**Permit access to a special opportunity for persons born on Leap Day.**

```cedar
permit(
  principal,
  action == Action::"redeem",
  resource is Prize
) when {
  principal.birthDate.day() == 29 &&
  principal.birthDate.month() == 2 &&
  context.currentTime.day() == 29 && 
  context.currentTime.month() == 2
};
```

**Forbid access to EU resources after Brexit**

```cedar
forbid(
  principal,
  action,
  resource
) when {
  context.currentTime > datetime("20200131T23:00:00Z") &&
  context.location.countryOfOrigin == 'GB' &&
  resource.owner == 'EU'
}
```


## Detailed design

This RFC proposes supporting two new extension types: `datetime`, which represents a particular instant of time, up to second accuracy, and `duration` which represents a duration of time.
To construct and manipulate these types we will provide the functions listed below.
All of this functionality will be hidden behind a `datetime` feature flag (analogously to the current decimal and IP extensions), allowing users to opt-out if they do not want this functionality.

- `datetime(s)` constructs a datetime object. Like with our other constructors, strict validation requires `s` to be a string literal, although evaluation/authorization support any string-typed expression. The string must have the form `"YYYY-MM-DDThh:mm:ssZ"` (UTC), or `"YYYY-MM-DDThh:mm:ss(+/-)hhmm"` (with Time Zone Offset).
  - This format is a subset of ISO 8601.
  - Internally, the returned `datetime` is always stored as UTC.
- `d.offset(du)` returns a new date with time offset by duration `du`.
  - This allows for a "lightweight" implementation of local time.
- `d.isBetween(d1, d2)` checks whether date `d` is between dates `d1` and `d2` (inclusive).
- `d.isBefore(d1)` checks whether date `d` is before date `d1` (non-inclusive).
- `d.since(d1)` computes the difference between two `datetime` objects and returns a `duration` object.
- `d.year()`, `d.month()`, `d.day()`, `d.hour()`, `d.minute()`, `d.second()` returns the relevant component of the date.
- `d.dayOfWeek()` returns 1 when the day is Sunday, and 7 when the day is Saturday.

Internally, the datetime object will have a structure such as:

```cedarschema
{
  year: long,
  month: long,
  day: long,
  hours: long,
  minutes: long,
  seconds: long,
}
```

The datetime parsing function will be responsible for ensuring validity of the structure (e.g., that `month` is between 1 and 12, see below for full details).

The `d.since(d1)` function returns a `duration` object.

- `duration(n, unit)` returns a `duration` object. Strict validation requires `unit` to be a string literal of one of the following values: "seconds", "minutes", "hours", "days".
- `du.toSeconds()`, `du.toMinutes()`, `du.toHours()`, `du.toDays()` returns a `long` indicating the number of units represented by this duration, truncated as appropriate. _e.g._
  - A `duration` representing 119 seconds, returns `1` when `toMinutes()` is called.
  - A `duration` representing 10 minutes, returns `0` when `toHours()` is called.

Internally, the `duration` object is a tagged `long` representing seconds. As a record:

```cedarschema
{
  seconds: long,
}
```

### Constraints on dates

Here are the full validity constraints on datetime values:

- `year` must be between 1900 and 2100
- `month` must be between 1 and 12
- `day` must be between 1 and...
  - 30 for `month`=4,6,9,11
  - 31 for `month`=1,3,5,7,8,10,12
  - 28 for `month`=2 if `year` is not divisible by 4
  - 29 for `month`=2 if `year` is divisible by 4
- `hours` must be between 0 and 23
- `minutes` must be between 0 and 60
- `seconds` must be between 0 and 60

For the functions `offset`, `since`, and `dayOfWeek` there are cases where we must account for leap years. We note that from the year 1900 to 2100, the leap year rule is simply "is the year divisible by 4?" For the `dayOfWeek` calculation, we can also make note that January 1, 1900 fell on a Monday. Practically speaking, we see no reason to support datetimes outside of the range of 1900 to 2100.

### Out of scope

- **Conversion between UTC and epochs:** this will be particularly difficult to model and verify in Lean (although it's technically possible, see [this paper](https://dl.acm.org/doi/abs/10.1145/3636501.3636958) which does something similar in the Coq proof assistant). Since it will likely require input-dependent loops, it is unlikely that this can be reasoned about efficiently with SMT.
- **Conversion between UTC and other Time Zones:** Time Zones are a notoriously complex system that evolves constantly. We avoid this complexity by offering `datetime.offset(duration).` Policy authors that require "local time" can use `context` to pass in a `duration` that shifts the current time appropriately.
- **Leap Seconds:** Cedar does not have a clock, and this proposal does not add one. We aim only to provide arithmetic operations over instants of time, at second resolution. We recognize that without leap second support, functions such as `isBefore()` or `offset()` cannot be precise.

### Support for date/time in other authorization systems

AWS IAM supports [date condition operators](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html#Conditions_Date), which can check relationships between date/time values in the ISO 8601 date format or Unix time. You can find an example of a IAM policy using date/time [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_aws-dates.html).

[Open Policy Agent](https://www.openpolicyagent.org) provides a [Time API](https://www.openpolicyagent.org/docs/latest/policy-reference/#time) with nanosecond precision and extensive time zone support. During policy evaluation, the current timestamp can be returned, and date/time arithmetic can be performed. The `diff` (equivalent to our proposed `since` function) returns an array of positional time unit components, instead of a value typed similarly to our proposed `duration`.

## Drawbacks



## Alternatives

### Alt. A: Use Unix time / timestamps / epochs

One of our suggested workarounds for date/time functionality has been to use [Unix time](https://en.wikipedia.org/wiki/Unix_time), which measures the number of non-leap seconds that have elapsed since 00:00:00 UTC on January 1, 1970. Unix time is commonly used in computing systems, and there are many libraries available to convert between more familiar day formats and Unix time.

Here are the previous examples rewritten to use Unix Time.

**Only allow experienced, tenured persons from the Hardware Engineering department to see prototypes.**

```cedar
permit(
  principal is User,
  action == Action::"view",
  resource in Folder::"device_prototypes"
)
when {
  principal.department == "HardwareEngineering" &&
  principal.jobLevel >= 10 &&
  (context.currentTime - (365 * 24 * 60 * 60)) >= principal.hireDate
};
```

**Only allow user "alice" to view JPEG photos for one week after creation.**

```cedar
permit(
  principal == User::"alice",
  action == PhotoOp::"view",
  resource is Photo
) when {
  resource.fileType == "JPEG" &&
  resource.creationDate <= (context.currentTime - (7 * 24 * 60 * 60)) 
};
```

**Allow access from a certain IP address only between 9am and 6pm UTC.**

Cedar _does not currently support_ arithmetic division (`/`) or remainder (`%`), and therefore this example is _not expressible_, today.

```cedar
permit(
  principal,
  action == Action::"access",
  resource
) when {
  context.srcIp.isInRange(ip("192.168.1.0/24")) &&
  9 <= ((context.currentTime / (60 * 60)) % 24) &&
  ((context.currentTime / (60 * 60)) % 24) < 18
};
```

Note that the localized version of this example, with timezone offset, could be supported using the `+` or `-` operators on `context.currentTime`.

**Prevent employees from accessing work documents on the weekend.**

With Unix Time, this requires `/` and `%` operators to compute the `dayOfWeek`, which isn't currently expressible in Cedar.

**Permit access to a special opportunity for persons born on Leap Day.**

With Unix Time, this requires `/` and `%` operators to compute whether or not it is a leap year, which isn't currently expressible in Cedar.

**Forbid access to EU resources after Brexit**

```cedar
forbid(
  principal,
  action,
  resource
) when {
  context.currentTime > 1580511600 &&
  context.location.countryOfOrigin == 'GB' &&
  resource.owner == 'EU'
}
```

### Alt. B: Pass results of time checks in the context

Another workaround we have suggested is to simply handle date/time logic _outside_ of Cedar, and pass the results of checks in the context. For example, you could pass in fields like  `context.isWorkingHours` or `context.dayOfTheWeek`.

Here are the previous examples rewritten to use additional context.

**Only allow experienced, tenured persons from the Hardware Engineering department to see prototypes.**

```cedar
permit(
  principal is User,
  action == Action::"view",
  resource in Folder::"device_prototypes"
)
when {
  principal.department == "HardwareEngineering" &&
  principal.jobLevel >= 10 &&
  principal.hireDate <= context.minimumHiringDateForAccess
};
```

**Only allow user "alice" to view JPEG photos for one week after creation.**

```cedar
permit(
  principal == User::"alice",
  action == PhotoOp::"view",
  resource is Photo
) when {
  resource.fileType == "JPEG" &&
  resource.creationDate >= context.oldestViewableDate
};
```

**Allow access from a certain IP address only between 9am and 6pm UTC.**

```cedar
permit(
  principal,
  action == Action::"access",
  resource
) when {
  context.srcIp.isInRange(ip("192.168.1.0/24")) &&
  context.isWorkingHours
};
```

**Prevent employees from accessing work documents on the weekend.**

```cedar
forbid(
  principal,
  action == Action::"access",
  resource is Document
) when {
  context.isTheWeekend
};
```

**Permit access to a special opportunity for persons born on Leap Day.**

```cedar
permit(
  principal,
  action == Action::"redeem",
  resource is Prize
) when {
  context.isMyBirthday && context.isLeapDay
};
```

**Forbid access to EU resources after Brexit**

```cedar
forbid(
  principal,
  action,
  resource
) when {
  context.afterBrexit &&
  context.location.countryOfOrigin == 'GB' &&
  resource.owner == 'EU'
}
```


### Alt. C: Combine Unix Time with additional Pre-Computed context.

The use of [Unix time](https://en.wikipedia.org/wiki/Unix_time), could be augmented with additional provided context, as in Alt. B.. To solve the readability problem of `context.now < 1723000000`, we could introduce `datetime(s)`, but instead return the Unix Time as a `long`.

Here are the previous examples rewritten to use Unix Time with pre-computed context, and the modified `datetime`.

**Only allow experienced, tenured persons from the Hardware Engineering department to see prototypes.**

```cedar
permit(
  principal is User,
  action == Action::"view",
  resource in Folder::"device_prototypes"
)
when {
  principal.department == "HardwareEngineering" &&
  principal.jobLevel >= 10 &&
  (context.currentTime - (365 * 24 * 60 * 60)) >= principal.hireDate
};
```

_(this is unchanged from Alt. A.)_

**Only allow user "alice" to view JPEG photos for one week after creation.**

```cedar
permit(
  principal == User::"alice",
  action == PhotoOp::"view",
  resource is Photo
) when {
  resource.fileType == "JPEG" &&
  resource.creationDate <= (context.currentTime - (7 * 24 * 60 * 60)) 
};
```

_(this is unchanged from Alt. A.)_

**Allow access from a certain IP address only between 9am and 6pm UTC.**

```cedar
permit(
  principal,
  action == Action::"access",
  resource
) when {
  context.srcIp.isInRange(ip("192.168.1.0/24")) &&
  context.workdayStart <= context.currentTime &&
  context.currentTime < context.workdayEnd
};
```

Note that the localized version of this example, must adjust `workdayStart` and `workdayEnd` as appropriate.

**Prevent employees from accessing work documents on the weekend.**

```cedar
forbid(
  principal,
  action == Action::"access",
  resource is Document
) when {
  context.isTheWeekend
};
```

_(this is unchanged from Alt. B.)_

**Permit access to a special opportunity for persons born on Leap Day.**

```
permit(
  principal,
  action == Action::"redeem",
  resource is Prize
) when {
  context.currentDate == principal.birthDate && 
  context.currentDate == datetime("202402029T00:00:00Z")
};
```

We must provide the `currentDate` as we can't truncate `currentTime` to eliminate the time component. 

**Forbid access to EU resources after Brexit**

```cedar
forbid(
  principal,
  action,
  resource
) when {
  context.currentTime > datetime("20200131T23:00:00Z") &&
  context.location.countryOfOrigin == 'GB' &&
  resource.owner == 'EU'
}
```

## Potential extensions

### Ext. A: Provide a `currentTime` function

The examples above expect `currentTime` to be passed in the context during the authorization request. An alternative would be to provide a `currentTime()` extension function that computes the current time at policy evaluation time. This has a drawback, when testing policies, in that the evaluator must be somehow extended to mock the time.

## Unresolved questions

### Should we consider local time at all?

Writing programs that involve dates and time is tricky. Standardizing on UTC representations for all dates and times trades achieving evaluator correctness more quickly at the expense of decreased readability for policy authors and auditors.

### Does second level granularity make sense?



