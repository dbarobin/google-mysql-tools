# Introduction #

Math is hard. Floating point math is even harder.

Alas, MySQL has changed things in MySQL 5.0 that make lossless conversions impossible. See [Bug36829](http://bugs.mysql.com/?id=36829) for details. This will not be fixed until 5.1.

# Details #

To use this feature, invoke mysqldump with the **--lossless-fp** option. When this is used, columns with type float and double are converted to a decimal string with 17 digits of precision. This is done because conversions from double to decimal and then back to double are lossless (the initial double value is equal to the final double value) when the double to decimal conversion generates 17 digits of precision. This is generally true. There are some values for float and double for which this is not true (subnormal).

```
mysqldump --lossless-fp my_database my_table > my_table.dump
```