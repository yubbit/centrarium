---
layout: post
title: "Reading Metastock Files"
date: 2019-09-21
author: yubbit
categories: ['Programming', 'Finance']
excerpt_separator: <!--more-->
---

I've been setting things up to do some data mining on the Philippine stock
market lately, since I have an account that's remained dormant over the past
few years. Before I can perform any sort of backtesting though, I need to find
a way to read the data.

The data I'm using was downloaded from COLFinancial's "Downloads" section, and
contains the daily values of stocks going back as far as 1986, but it's stored
in some proprietary format that doesn't appear to jive well with the modern day
IEEE standards that everybody uses to represent numbers.

COLFinancial's data is stored inside a file archive with the `.lzh` file
format. A quick Google search tells me that this represents a file compressed
using the Lempel-Ziv-Welch algorithm. In Ubuntu, you can simply download the
`lhasa` package to deal with these files.

With that out of the way, you should end up with a directory containing a bunch
of `.dat` and `.dop`, and a `MASTER`, `EMASTER`, or `XMASTER` file. To extract
information from these files, you can use a utility like 
[`atem`](https://github.com/rudimeier/atem/).

To access the data in a Metastock file, you'd first need to examine one or all
of the `*MASTER` files. For a `MASTER` file in particular, the way records are
stored for that particular file is described in the header of the `MASTER`
file, which comprises the first 53 bytes of the file. The breakdown is as
follows (taken from comments in `atem`):

```
 #0,  1b, unsigned char, count records (dat files), must be >0
 #1,  1b, char, always '\0'
 #2,  1b, unsigned char, max record number (dat file number) must be >0
 #3,  1b, char, always '\0'
 #4, 45b, char*, always '\0' ?
#49,  4b, int, serial number
```

More or less, only the first and the third byte are of any importance for
parsing the data. `MASTER` files have a record length of 53, so each set of 53
bytes following the header contains a record of some sort, and the number of
records for this master file is contained in the first byte.

Each record in the `MASTER` file denotes a security, where it can be found,
its start and end dates, and periodicity. The record itself contains the 
following information (again, taken from `atem`):

```
 #0,  1b, unsigned char, dat file number
 #1,  2b, short, file type, always 101 (error #1005)
 #3,  1b, unsigned char, record length
          must be 4 times record count (error #1006)
 #4,  1b, unsigned char, record count
          must be 4, 5, 6, 7 or 8 (error #1007)
 #5,  1b: char, always '\0' (error #1008)
 #6,  1b: char, always '\0' (error #1009)
 #7, 16b: char*, security name
          only alphanumeric characters (error #1010)
#23,  2b: short, always 0 (error #1011)
#25,  4b: float(ms basic), first date, valid (error #1012)
#29,  4b: float(ms basic), last date, valid (error #1013)
#33,  1b: char, periodicity, must be 'I', 'D', 'W', 'M' (error #1014)
#34,  2b: unsigned short, intraday time frame between 0 and 60 minutes
          (error #1015)
#36, 14b: char*, symbol, space padded,
          not always (or never?) zero terminated
          only alphanumeric characters (error #1016)
#50,  1b: char, always a space ' ' (error #1017)
          note, premium data sets '\0'
#51,  1b: char, chart flag, always ' ' or '*' (error #1018)
          note, premium data sets '\0'
#52,  1b: char, always '\0' (error #1019)
```

For these records, the 1st byte contains the `.dat` file number, and
consequently the filename of each record. The 3rd byte contains the length
of each record within that file, in number of bytes. The 4th byte contains
the total number of records in that file. The 16 bytes following the
7th byte have the long name of the security, while the 14 bytes following
the 36th have its ticker symbol. Its periodicty is listed in the 33rd byte as
intraday, daily, weekly, or monthly.

Finally, the beginning and ending dates for the data are listed as floats in 
the 25th and 29th positions respectively. Note that these floats are stored in 
the Microsoft binary format, and can't be parsed using most language's standard 
libraries, since most of them follow the IEEE standard. You can find conversion
functions [here](https://community.embarcadero.com/index.php/article/
technical-articles/162-programming/14799-converting-between-microsoft-binary-
and-ieee-forma). The differences can be summed up in this comment, taken from
that link:

```
/* MS Binary Format                         */
/* byte order =>    m3 | m2 | m1 | exponent */
/* m1 is most significant byte => sbbb|bbbb */
/* m3 is the least significant byte         */
/*      m = mantissa byte                   */
/*      s = sign bit                        */
/*      b = bit                             */

/* IEEE Single Precision Float Format       */
/*    m3        m2        m1     exponent   */
/* mmmm|mmmm mmmm|mmmm emmm|mmmm seee|eeee  */
/*          s = sign bit                    */
/*          e = exponent bit                */
/*          m = mantissa bit                */
```

For the date formats in particular, it should return an integer that can be
read as a string of the form "YYYYMMDD".

The information we have up to this point should be enough to access each
`.dat` file. Just loop through each of the `.dat` files, and read its data
in chunks of the record length given in the `MASTER` file. Within each record,
their byte offsets are given by the following (once again taken from `atem`).
Note that these numbers are represented by their octal values:

```
D_DAT = 01,     DATE
D_HIG = 02,     HIGH
D_LOW = 04,     LOW
D_CLO = 010,    CLOSE
D_VOL = 020,    VOLUME
D_OPE = 040,    OPEN
D_OPI = 0100,   OPENINT (?)
D_TIM = 0200    TIME
```

All of these values are stored as Microsoft binary format floats, and have to
be parsed further to get their true values. The date field is parsed the same
way that the `MASTER` records are, while the time field is parsed similarly,
formatted as HH:MM:SS. All the remaining numbers are interpreted as floats,
though volume can easily be coerced into an int.

That should cover everything needed to parse the files. Details regarding the
`EMASTER` and `XMASTER` files can also be found in the `atem` code, and I'll
likely update this post to include ways to parse those.
