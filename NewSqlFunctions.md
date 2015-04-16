# Notes #

New SQL functions have been added:
  * IEEE754\_TO\_STRING
  * UNORDERED\_CHECKSUM
  * ORDERED\_CHECKSUM
  * HASH
  * LAST\_VALUE
  * NEW\_PASSWORD

# IEEE754\_TO\_STRING #

Converts a float or double decimal value with type string. This generates 17 digits of precision so that conversion of the string back to a double does not lose precision (the original double should be equal to the final double for all but a few special cases.

# UNORDERED\_CHECKSUM #

This is a SQL aggregate function that accepts one or more arguments. It returns the hash of its input arguments per group. The function is order independent. The result from each row in a group is combined by XOR.

```
select unordered_checksum(c1) from foo group by c2;
select unordered_checksum(c1, c2) from foo group by c3;
```

# ORDERED\_CHECKSUM #

This is a SQL aggregate function that accepts one or more arguments. It returns the hash of its input arguments per group. The function is order dependent. The output of this from the first row in a group is used as the seed for the hash on the next row.

```
select ordered_checksum(c1) from foo group by c2;
select ordered_checksum(c1, c2) from foo group by c3;
```

# HASH #

This is a SQL function. It returns the hash of its input argument. It is not an aggregate function and produces one value per row.

```
select hash(column) from foo
```

# LAST\_VALUE #

This is a SQL aggregate function. It returns the last value read per group. Thus this depends on the input order to aggregation. See OnlineDataDrift for a use case.

# NEW\_PASSWORD #

Computes the new-style password hash regardless of the value for the my.cnf parameter **old\_passwords**.