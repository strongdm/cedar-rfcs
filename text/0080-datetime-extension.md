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

Previously for these types of applications, our suggestion has been to use [Unix time](https://en.wikipedia.org/wiki/Unix_time) (see Alt. A) or to pass the result of time computations in the context (see Alt. B).
But these are not ideal solutions because Cedar policies are intended to _expressive_,  _readable_, and _auditable_.
Unix timestamps fail the readability criteria: an expression like `context.now < 1723000000` is difficult to read and understand without additional tooling to decode the timestamp.
Unix timestamps are also indistinguishable from any other integral value providing no additional help in expressing the abstraction of time.
Passing in pre-computed values in the context (e.g., a field like `context.isWorkingHours`) makes authorization logic difficult to audit because it moves this logic outside of Cedar policies and into the calling application.
It is also difficult for the calling application to predict the necessary pre-computed values that a policy writer requires for their intended purpose. These may change over time, and may also differ depending on the principal, action, and/or resource.

### Additional examples

Using an idealized API, the following examples are targeted for this RFC.

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
  9 <= context.currentTime.hour() &&
  context.currentTime.hour() < 18
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
  9 <= context.currentTime.offset(principal.tzOffset).hour() &&
  context.currentTime.offset(principal.tzOffset).hour() < 18
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
  principal.birthDate.format("MMDD") == context.currenTime.format("MMDD")
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

This RFC proposes supporting two new extension types: `datetime`, which represents a particular instant of time, up to millisecond accuracy, and `duration` which represents a duration of time.
To construct and manipulate these types we will provide the functions listed below.
All of this functionality will be hidden behind a `datetime` feature flag (analogous to the current decimal and IP extensions), allowing users to opt-out if they do not want this functionality.
The proposed types will not allow Cedar to support all possible date / time use cases. For example, it would be difficult to model `isLeapYear` in SMT, which makes it hard to support `datetime.dayOfWeek()`, `datetime.dayOfMonth()` or `datetime.year()`. Cedar applications that wish to define policy based on these ideas should pass pre-computed properties through entities, or through `context`.

- `datetime(string)` constructs a datetime value. Like with our other constructors, strict validation requires `string` to be a string literal, although evaluation/authorization support any string-typed expression. The string must be of one of the forms, and regardless of the timezone offset is always normalized to UTC: 
  - `"YYYY-MM-DDThh:mm:ssZ"` (UTC)
  - `"YYYY-MM-DDThh:mm:ss.SSSZ"` (UTC with millisecond precision)
  - `"YYYY-MM-DDThh:mm:ss(+/-)hhmm"` (With timezone offset in hours and minutes)
  - `"YYYY-MM-DDThh:mm:ss.SSS(+/-)hhmm"` (With timezone offset in hours and minutes and millisecond precision)
- `.add(duration)` returns a new `datetime`, offset by duration.
- `.toDate()` returns a new `datetime`, truncating to the day, such that printing the `datetime` would have `00:00:00` as the time.
- `.toTime()` returns a new `duration`, removing the days, such that only milliseconds since `.toDate()` are left. This is equivalent to `DT - DT.toDate()`

Values of type `datetime` can be used with comparison operators:

- `DT1 < DT2` returns `true` when `DT1` is before `DT2`
- `DT1 <= DT2` returns `true` when `DT1` is before or equal to `DT2`
- `DT1 > DT2` returns `true` when `DT1` is after `DT2`
- `DT1 >= DT2` returns `true` when `DT1` is after or equal to `DT2`
- `DT1 == DT2` returns `true` when `DT1` is equal to `DT2`

Values of type `datetime` can also be used with the `-` operator::

- `DT1 - DT2` returns the difference between `DT1` and `DT2` as a `duration`.

Note that `.add(duration)` is the inverse of `-`, not `+`. 

The `datetime` type is internally represented as a `long` and contains a Unix Time in milliseconds. This is the number of non-leap seconds that have passed since `1970-01-01T00:00:00Z` in milliseconds. Unix Time days are always 86,400 seconds and handle leap seconds by absorbing them at the start of the day. Due to using Unix Time, and not providing a "current time" function, Cedar avoids the complexities of leap second handling, pushing them to the system and application.

- `duration(long, string)` constructs a duration value. The string argument must be one of `"days", "hours", "minutes", "seconds", "milliseconds"`. Strict validation requires `string` to be a string literal, although evaluation/authorization support any string-typed expression. 
- `.toMillis()` returns a `long` describing the number of milliseconds in this duration. (the value as a long, itself)
- `.toSeconds()` returns a `long` describing the number of seconds in this duration. (`.toMillis() / 1000`)
- `.toMinutes()` returns a `long` describing the number of minutes in this duration. (`.toSeconds() / 60`)
- `.toHours()` returns a `long` describing the number of hours in this duration. (`.toMinutes() / 60`)
- `.toDays()` returns a `long` describing the number of days in this duration. (`.toHours() / 24`)

Values with type `duration` duration can also be used with comparison operators:

- `DT1 < DT2` returns `true` when `DT1` is before `DT2`
- `DT1 <= DT2` returns `true` when `DT1` is before or equal to `DT2`
- `DT1 > DT2` returns `true` when `DT1` is after `DT2`
- `DT1 >= DT2` returns `true` when `DT1` is after or equal to `DT2`
- `DT1 == DT2` returns `true` when `DT1` is equal to `DT2`


With this API in mind, we can express our examples from above:


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
  (context.now.timestamp - principal.hireDate) > duration(365, "days")
};
```

**Only allow user "alice" to view JPEG photos after after creation.**

```cedar
permit(
  principal == User::"alice",
  action == PhotoOp::"view",
  resource is Photo
) when {
  resource.fileType == "JPEG" &&
  (context.now.timestamp - resource.creationTime) <= duration(7, "days")
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
  context.workdayStart <= context.now.timestamp &&
  context.now.timestamp < context.workdayEnd
};
```

Note that the localized version of this example must adjust `workdayStart` and `workdayEnd`.

**Prevent employees from accessing work documents on the weekend.**

```cedar
forbid(
  principal,
  action == Action::"access",
  resource is Document
) when {
  [1, 7].context.now.dayOfWeek
};
```

**Permit access to a special opportunity for persons born on Leap Day.**

```
permit(
  principal,
  action == Action::"redeem",
  resource is Prize
) when {
  principal.birthDate.day == 29 &&
  principal.birthDate.month == 2 &&
  context.now.day == 29 && 
  context.now.month == 2
};
```

**Forbid access to EU resources after Brexit**

```cedar
forbid(
  principal,
  action,
  resource
) when {
  context.now.timestamp > datetime("20200131T23:00:00Z") &&
  context.location.countryOfOrigin == 'GB' &&
  resource.owner == 'EU'
}
```

### Out of scope

- **Conversion between UTC and epochs:** this will be particularly difficult to model and verify in Lean (although it's technically possible, see [this paper](https://dl.acm.org/doi/abs/10.1145/3636501.3636958) which does something similar in the Coq proof assistant). Since it will likely require input-dependent loops, it is unlikely that this can be reasoned about efficiently with SMT.

- **Conversion between UTC and other Time Zones:** Time Zones are a notoriously complex system that evolves rapidly. We avoid this complexity by offering `datetime.add(duration).` Policy authors that require "local time" can either provide an additional datetime in `context` or provide a `duration` to the `context` and call `.add()` to shift the time.

- **Leap Seconds:** Cedar does not have a clock, and this proposal does not add one. Instead, we pass Unix Time through `context` or an entity and let the system / application handle leap seconds.

### Support for date/time in other authorization systems

AWS IAM supports [date condition operators](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html#Conditions_Date), which can check relationships between date/time values in the ISO 8601 date format or Unix time. You can find an example of a IAM policy using date/time [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_aws-dates.html).

[Open Policy Agent](https://www.openpolicyagent.org) provides a [Time API](https://www.openpolicyagent.org/docs/latest/policy-reference/#time) with nanosecond precision and extensive time zone support. During policy evaluation, the current timestamp can be returned, and date/time arithmetic can be performed. The `diff` (equivalent to our proposed `since` function) returns an array of positional time unit components, instead of a value typed similarly to our proposed `duration`.

## Alternatives

### Alt. A: Represent Unix time with a Long

This is, effectively, the do nothing alternative. Cedar has long suggested workarounds for date/time functionality by using the comparison and arithmetic operators with `context` provided Unix Timestamps.

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

Another workaround we have suggested is to simply handle date/time logic _outside_ of Cedar, and pass the results of checks in the `context`. For example, you could pass in fields like  `context.isWorkingHours` or `context.dayOfTheWeek`.

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

## Potential extensions

### Ext. A: Provide a `currentTime` function

The examples above expect `currentTime` to be passed in the context during the authorization request. An alternative would be to provide a `currentTime()` extension function that computes the current time at policy evaluation time. This has a drawback, when testing policies, in that the evaluator must be somehow extended to mock the time.

## Unresolved questions

### Should we consider local time at all?

Writing programs that involve dates and time is tricky. Standardizing on UTC representations for all dates and times trades achieving evaluator correctness more quickly at the expense of decreased readability for policy authors and auditors.

