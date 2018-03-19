---
layout: post
title: Data type coercion vs comparison rules
---

A somewhat unexpected behavior has been observed while investigating different aspects of the data type coercion. By coercion I mean an implicit casting to the "common" data type that happens when a few operands have different data types but the expression requires some predefined data type to be chosen for the resulting value. In Firebird, common data type determination is used in two cases: (a) describing the select list expressions of the union and (b) describing the resulting data type of the CASE/COALESCE functions.

In old versions, different data types couldn't be used as arguments in these cases, throwing the famous error "data type unknown", see this simple test case: 

```
select 1 from rdb$database
union
select '1' from rdb$database
```

Starting with Firebird 2.0, the common data type determination logic has been added, so that heterogeneous expressions are now allowed and the aforementioned test case works as expected. The implementation follows the SQL specification, subclause 9.3 - "Result of data type combinations". In particular, case [3a] declares:

> If any of the data types in DTS is character string, then:
> 
> i) All data types in DTS shall be character string, and all of them shall have the same character
repertoire. The character set of the result is the character set of the data type in DTS that has the character encoding form with the highest precedence.

This means that in the above heterogeneous example, the resulting value will be a character string and the non-conforming expressions will be implicitly casted to become strings. And it looks sensible, as otherwise (if a numeric would be chosen as the common data type) the following example would cause the data type conversion error:

```
select 1 from rdb$database
union
select 'a' from rdb$database
```

However, there's a hidden issue which remained unnoticed for a few recent years.

A union without the ALL clause implies DISTINCT which in turn involves the comparison operation to decide whether values are equal or distinct. The SQL specification doesn't allow heterogeneous comparisons, so the data type coercion perfectly fits the declared behavior by initially casting the values to the common data type and then comparing them being of the same data type. But Firebird does allow heterogeneous comparisons and it brings a few interesting questions.

Look at these examples:

```
select 1 from rdb$database where 1 = '1' 
select 1 from rdb$database where 1 = '+1'
select 1 from rdb$database where 1 = '01'
select 1 from rdb$database where 1 = '1.0'
```

All of them return one row as the result. In order to do so, Firebird involves special comparison priorities, in particular character strings are implicitly casted to numerics and only then compared as numerics. In other words, the common data type becomes numeric. On the negative side, it causes the following example to fail with a conversion error:

```
select 1 from rdb$database where 1 = 'a'
```

On the positive side, non-trivial comparisons like the ones mentioned above return the practically correct results. And let's note this is the originally intended behavior which works this way for the past few decades.

Now let's modify the example with unions a little:

```
select 1 from rdb$database
union
select '1.0' from rdb$database
```

This query returns two resulting rows, thus reporting the values as distinct. But accordingly to the comparison rules, these values are equal! A somewhat tricky situation, isn't it?

Okay, let's look at other database engines:

- **PostgreSQL 9.0** : Mixed-type comparisons are not allowed, mixed-type unions are not allowed. The safest but at the same time the most limited solution.
- **Oracle 10g** : Mixed-type comparisons are allowed and work like in Firebird, mixed-type unions are not allowed. This matches the situation existed in Firebird prior to version 2.0.
- **MSSQL 2008** : Both mixed-type comparisons and unions are allowed and the coercion rules are the same: a character string gets casted to a numeric, like in Firebird. IMHO, this is the most flexible and at the same time absolutely consistent solution, something not really expected from Microsoft ;-) However, beware that (1 union 'a') would throw a data type conversion error.

Now the question is what to do with Firebird. The current behavior is IMHO wrong. Changing the data type comparison rules is not an option. Reverting back to the pre-v2.0 limitations is not desirable and it's likely to affect backward compatibility. I tend to think that the MSSQL solution is the best one, even knowing that it can potentially break some existing queries. That said, this isn't going to happen in point releases, so only Firebird 3.0 is likely to have the rules changed.
