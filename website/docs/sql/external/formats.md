# External Formats and Options

Hyper supports various external data formats.

Some external formats carry schema information (e.g., Apache
Parquet) while others do not (e.g., CSV). While Hyper can infer
the schema of formats possessing schema information automatically,
the schema has to be given explicitly for formats without schema
information (or is taken from the target table in a
[COPY FROM](/docs/sql/command/copy_from) statement).

The format of an external source or target is set through the
`FORMAT` option. If the `FORMAT` option is not
specified, Hyper will try to infer the format from the file extension.
If multiple files are read or written, all have to possess the same
extension for the inference to succeed.

The following formats are supported:

Format |`format` Option Value |Recognized File Extensions |Schema Inference? |Description
----|----|----|----|----
[Text](#external-format-text) |`'text'` |`.txt`, `.txt.gz` |No |Text format; as in PostgreSQL. Optionally, gzip compressed.
[CSV](#external-format-csv) |`'csv'` |`.csv`, `.csv.gz` |No |Comma Separated Value format; as in PostgreSQL. Optionally, gzip compressed.
[Apache Parquet](#external-format-parquet) |`'parquet'` |`.parquet` |Yes |The [Apache Parquet format](https://parquet.apache.org/); both version 1 and 2 supported
[Apache Iceberg](#external-format-iceberg) |`'iceberg'` |Specified path must point to table directory |Yes |The [Apache Iceberg format](https://iceberg.apache.org/); version 1 and 2 are supported; version 3 is not supported
[Apache Arrow](#external-format-arrow) | `'arrowfile'`, `'arrowstream'` | `arrow`, `arrows` | No | The [Apache Arrow format](https://arrow.apache.org/) version 1.3 [with restrictions](#external-format-arrow)

## Format Options

Format options allow customizing the way the external format is read and
interpreted. The available options depend on the format. Syntactically,
options are specified using the syntax for named parameters, that is,
`option => value`, and separated by commas. In the [COPY FROM](../command/copy_from),
[COPY TO](../command/copy_to) and [CREATE EXTERNAL TABLE](../command/create_external_table)
statement, options are specified in the `WITH` clause. For the
[`external`](../setreturning#external) function, options are specified
as function arguments after the initial argument describing the source
or target. For example, the following statements all read from a CSV
file `products.csv`, using field delimiter `'|'`:

```
COPY products FROM 'products.csv' WITH (FORMAT => 'csv', DELIMITER => '|');

CREATE TEMPORARY EXTERNAL TABLE products (name text, price int)
    FOR 'products.csv'
    WITH (FORMAT => 'csv', DELIMITER => '|');

SELECT * FROM external('products.csv',
    FORMAT => 'csv', DELIMITER => '|',
    COLUMNS => DESCRIPTOR(name text, price int));
```

Depending on the format, various format-specific options are available, which alter the way in which the external format will be processed.
The available options are described in detail below.

## Common Format Options

The following options are available for all or multiple external formats:

`FORMAT => 'format_name'`

: The format, as specified in the table above.
  This parameter can be omitted, if the name(s) of the file(s) to be
  scanned end in the respective recognized file extension.

`COLUMNS => DESCRIPTOR(<column_def> [, ... ])`

: The schema of the file.  A `column_def` consists of a column name followed by a
  column type. Optionally, it can also include a `COLLATE` clause or a `NOT NULL`
  restriction.
  The `COLUMNS` option is o nly necessary when using the `external`
  function to read the data. `COPY` and `CREATE TEMPORARY EXTERNAL TABLE`
  will assume the file to have the same schema as the table.

`SANITIZE => boolean`

: Hyper requires any input text to be valid UTF-8 and does verify this
  when reading external data. If `sanitize` is `false` (default), then
  Hyper will raise an error if invalid UTF-8 data is encountered. If
  `sanitize` is set to `true`, then Hyper will replace invalid UTF-8
  sequences in text fields with the replacement character "�".

`COMPRESSION => { 'auto' | 'none' | 'gzip' }`

: If set to `'auto'` (default) then any input file having an extension
  of `.gz` is assumed to be gzipped, while any file without this
  extension is assumed to be uncompressed. If set to `'none'`, all
  input files are treated as uncompressed. If set to `'gzip'`, all
  input files are expected to be gzip compressed.
: Not available for Apache Parquet, Apache Arrow or Iceberg, as those formats handle
  compression internally, so gzipping a Parquet file is not advisable
  and therefore not used in practice.

`MAX_FILE_SIZE => integer`

: If set, the `COPY TO` statement will write to multiple files, each of
  size specified by `MAX_FILE_SIZE`, with the exception of the last
  file.

## Text Format {#external-format-text}

When the `FORMAT => 'text'` option or a file extension of `.txt` is
used, the data is read or written as a text file with one line per table
row. Optionally, gzip compressed text files can also be read, with the
extension `.txt.gz`.

Columns in a row are separated by the delimiter character.
The column values are string representations of the
values, as if the values were casted to the `text` type. The specified
null string represents a SQL null value. An error will be raised if any
line of the input file contains more or fewer columns than expected.

Backslash characters (`\`) can be used in the data to quote data
characters that might otherwise be taken as row or column delimiters. In
particular, the following characters *must* be preceded by a backslash
if they appear as part of a column value: backslash itself, newline,
carriage return, and the current delimiter character.

The input is matched against the null string before removing
backslashes. Therefore, a null string such as `\N` cannot be confused
with the actual data value `\N` (which would be represented as `\\N`).

The following special backslash sequences are recognized:

Sequence |Represents
-------------|--------------
`\b` |Backspace (ASCII 8)
`\f` |Form feed (ASCII 12)
`\n` |Newline (ASCII 10)
`\r` |Carriage return (ASCII 13)
`\t` |Tab (ASCII 9)
`\v` |Vertical tab (ASCII 11)
`\<digits>` |Backslash followed by one to three octal digits specifies the character with that numeric code
`\x<digits>` |Backslash `x` followed by one or two hex digits specifies the character with that numeric code

It is strongly recommended that applications generating data convert
data newlines and carriage returns to the `\n` and `\r` sequences
respectively. At present it is possible to represent a data carriage
return by a backslash and carriage return, and to represent a data
newline by a backslash and newline. However, these representations might
not be accepted in future releases. They are also highly vulnerable to
corruption if the file is transferred across different machines (for
example, from Unix to Windows or vice versa).

Besides the [common format options](#common-format-options), the text
format supports the following options:

`DELIMITER => 'delimiter'`

: Specifies the character that separates columns within each row
 (line) of the file. The default is a tab character in text format, a
 comma in `CSV` format. This must be a single one-byte character.

`NULL => 'null_string'`

: Specifies the string that represents a null value. The default is
 `\N` (backslash-N) in text format, and an unquoted empty string in
 `CSV` format. You might prefer an empty string even in text format
 for cases where you don't want to distinguish nulls from empty
 strings.

`ENCODING => { 'utf-8' | 'utf-16' | 'utf-16-le' | 'utf-16-be' }`

: Specifies the encoding of the file read. `utf-16` automatically
 infers the endianness of the file from its byte order mark. If no
 byte order mark is present, little endian is used. `utf-16-le`
 (little endian) and `utf-16-be` (big endian) can be used to
 explicitly specify the endianness. The default is `utf-8`.

`ON_CAST_FAILURE => { 'error' | 'set_null' }`

: Specifies the behavior when a cast failure occurs, that is, when the
 value is not a correct textual representation of the data type.
 `error` raises an error. `set_null` sets the value with cast failure
 to NULL, if the field is nullable. If the field is not nullable,
 then an error is raised. The default is `error`.

## CSV Format {#external-format-csv}

The `FORMAT => 'csv'` format option or a file extension of `.csv` is
used for reading and writing the Comma Separated Value (CSV) file format
used by many other programs, such as spreadsheets. Instead of the
escaping rules used by Hyper's standard text format, it produces and
recognizes the common CSV escaping mechanism. Optionally, gzip
compressed CSV files can also be read, with the extension `.csv.gz`.

The values in each record are separated by the `DELIMITER` character. If
the value contains the delimiter character, the `QUOTE` character, the
`NULL` string, a carriage return, or line feed character, then the whole
value is prefixed and suffixed by the `QUOTE` character, and any
occurrence within the value of a `QUOTE` character or the `ESCAPE`
character is preceded by the escape character.

The CSV format has no standard way to distinguish a `NULL` value from
an empty string. Hyper handles this by quoting. A `NULL` is represented
by the `NULL` option string and is not quoted, while a non-`NULL` value
matching the `NULL` option string is quoted. For example, with the
default settings, a `NULL` is represented as an unquoted empty string,
while an empty string data value is represented with double quotes
(`""`). You can use `FORCE_NOT_NULL` to prevent `NULL` input comparisons
for specific columns. You can also use `FORCE_NULL` to convert quoted
null string data values to `NULL`.

`FORCE_NULL` and `FORCE_NOT_NULL` can be used simultaneously on the same
column. This results in converting only quoted null strings to null
values while unquoted null strings remain unchanged.

:::note
In `CSV` format, all characters are significant. A quoted value
surrounded by white space, or any characters other than `DELIMITER`,
will include those characters. This can cause errors if you import data
from a system that pads `CSV` lines with white space out to some fixed
width. If such a situation arises you might need to preprocess the `CSV`
file to remove the trailing white space, before importing the data into
Hyper.
:::

:::note
CSV format will recognize CSV files with quoted values containing
embedded carriage returns and line feeds. Thus the files are not
strictly one line per table row like text-format files.
:::

Besides the [common format options](#common-format-options) and the
[text format options](#text-format-options), the CSV format supports the
following options:

`HEADER => boolean`

: Specifies whether the file contains a header line with the names of
 each column in the file. Default is false.

`QUOTE => 'quote_character'`

: Specifies the quoting character to be used when a data value is
 quoted. The default is double-quote. This must be a single one-byte
 character.

`ESCAPE => 'escape_character'`

: Specifies the character that should appear before a data character
 that matches the `QUOTE` value. The default is the same as the
 `QUOTE` value (so that the quoting character is doubled if it
 appears in the data). This must be a single one-byte character.

`FORCE_NOT_NULL`

: Do not match the specified columns' values against the null string.
 In the default case where the null string is empty, this means that
 empty values will be read as zero-length strings rather than nulls,
 even when they are not quoted.

`FORCE_NULL`

: Match the specified columns' values against the null string, even
 if it has been quoted, and if a match is found set the value to
 `NULL`. In the default case where the null string is empty, this
 converts a quoted empty string into NULL.

## Apache Parquet Format {#external-format-parquet}

The `FORMAT => 'parquet'` option or a file extension of `.parquet`
enables reading and writing the [Apache Parquet
format](https://parquet.apache.org/), which is a binary columnar format.
Thus, data stored in Parquet will usually be smaller than in text
format. As data is stored in columnar format, Hyper does not need to
read the whole file if only a subset of columns is selected but can
instead only read the parts of the file in which selected columns are
stored. This will speed up queries selecting only few columns of a file
having a lot of columns, especially when reading from an external
location with limited bandwidth, such as when accessing Amazon S3 from a
non-AWS machine.

The Apache Parquet format stores schema information, so Hyper can infer
the schema for Parquet files.

Hyper supports both versions v1 and v2 of Parquet, including the
DataPageV2 page format. However, some data types, encodings, and
compressions are not supported. The following restrictions apply when
reading Parquet:

- Nested columns and therefore the nested types `MAP` and `LIST` are
 not supported.

- The types `BSON`, `UUID`, and `ENUM` are not supported.

- The physical type `FIXED_LEN_BYTE_ARRAY` without any logical or
 converted type is not supported.

- The types `TIME_MILLIS` and `TIME_NANOS` are not supported. Consider
 using `TIME_MICROS` instead.

- The deprecated `BIT_PACKED` encoding is not supported. No recent
 Parquet files should use this encoding, as it is deprecated for over
 half a decade.

- The `DELTA_LENGTH_BYTE_ARRAY` encoding and the very recent
 `BYTE_STREAM_SPLIT` encoding are not supported, as they are not
 written by any library. If you encounter any Parquet files using
 these encodings, let us know.

- Supported compressions are `SNAPPY`, `GZIP`, `ZSTD`, and `LZ4_RAW`.

:::note
If a Parquet file contains columns with unsupported data types or
encodings, Hyper can still read the other columns in the file, as long
as you do not select any unsupported columns.
:::

Besides the [common format options](#common-format-options), the
following options are available when writing to Parquet files:

`ROWS_PER_ROW_GROUP => integer`

: The number of rows to include in a row group.

`CODEC => { 'uncompressed' | 'snappy' | 'gzip' | 'zstd' | 'lz4_raw' }`

: Options to compress the data.

`WRITE_STATISTICS => { 'none' | 'rowgroup' | 'page' | 'columnindex' }`

: Whether or how the statistics should be written. `rowgroup` means to
  write row group statistics only. `page` means to write row group and
  page statistics. `columnindex` means to write row group, column and
  offset statistics, without page statistics.

## Apache Iceberg Format {#external-format-iceberg}

The `FORMAT => 'iceberg'` option enables reading (currently not possible
to write) the [Apache Iceberg format](https://iceberg.apache.org/),
which is a binary format designed
for huge data analytics tables. Apache Iceberg provides metadata
information like versioning, partitioning and statistics on top of
storage formats like parquet. Hyper can use the metadata information to
efficiently access Iceberg tables and only read the data from the
parquet files that are needed to answer a query.

When using Iceberg in a query, you need to provide the path to the table
root directory, i.e. the directory that has `metadata/` and `data/`
subdirectories. The location of the Iceberg table can be specified as
described in [External Locations](location), however the `ARRAY` syntax is not
supported. Iceberg tables can be queried from local disk and from Amazon
S3. Here is an example how to use iceberg, assuming no credentials need
to be specified:

```
SELECT * FROM EXTERNAL('s3://mybucket/salesorders', FORMAT => 'iceberg');
```

Hyper supports versions v1 and v2 of Iceberg; version 3 is not
supported. Please note the following restrictions and supported features
when using the Iceberg format:

- Hyper only supports Apache Parquet as data storage format. Other
 formats like ORC are currently not supported

- The [restrictions for parquet file
 support](#external-format-parquet) apply

- Merge on Read is supported for positional deletes. Equality deletes
 are not supported.

- Schema changes like added, dropped and renamed columns are supported

:::note
Hyper's support for the Iceberg format is still in an early state and
should be considered experimental. We recommend to not use it in
production workloads yet.
:::

## Apache Arrow Format {#external-format-arrow}

:::warning Experimental Feature
Hyper's support for the Arrow format is still in an early state and
should be considered experimental. We recommend to not use it in
production workloads yet. Set the `experimental_external_format_arrow` [setting](/docs/hyper-api/hyper_process#experimentalsettings)
to `true` to activate it.
:::

The `FORMAT => 'arrowfile'` or `FORMAT => 'arrowstream'` option enables reading the columnar binary format [Apache Arrow](https://arrow.apache.org/).
As data is stored in a columnar format, Hyper does not need to
read the whole file if only a subset of columns is selected. Hyper can read only
the parts of the file in which selected columns are
stored. This will speed up queries selecting only few columns of a file
having a lot of columns. In contrast to [Parquet](#external-format-parquet),
we do not support loading partial Arrow files from S3 - if reading a 
subset of all columns from S3, Parquet is the better option.

We support the [IPC streaming format](https://arrow.apache.org/docs/format/Columnar.html#ipc-streaming-format) (file extension `.arrows`)
with `FORMAT => 'arrowstream'` and the [IPC file format](https://arrow.apache.org/docs/format/Columnar.html#ipc-file-format) (sometimes referred to as Feather V2; file extensions `.arrow` or `.feather`) with `FORMAT => 'arrowfile'`.

The only options supported by Arrow are the 
[common format options](#common-format-options). The `COLUMNS` option 
is required because we do not support schema inference, yet.
We provide a 1-to-1 mapping from Arrow types to SQL types (table below) and expect the user
to choose these SQL types exactly. We do not perform any implicit casts.
The `COMPRESSION` option is not supported.

Arrow Type | Hyper Type | Notes
----|----|----
`Bool` | `Bool`
`Int` signed, 8 bit | `SmallInt` |
`Int` signed, 16 bit | `SmallInt` |
`Int` signed, 32 bit | `Integer` |
`Int` signed, 64 bit | `BigInt` |
`Int` unsigned, 8 bit | `SmallInt` |
`Int` unsigned, 16 bit | `Integer` |
`Int` unsigned, 32 bit | `BigInt` |
`Int` unsigned, 64 bit | `Numeric(20,0)` | Not recommended - processing is slower than other Int types.
`FloatingPoint` with any precision | `DOUBLE PRECISION` | Half floats are not supported.
`Decimal(precision, scale)` | `NUMERIC(precision, scale)` | Hyper does not support precision > 38. For better performance, we recommend using precision <= 18.
`Date` | `Date` |
`Time` | `Time` | Hyper does not support nano second precision and truncates it to microseconds.
`Timestamp` without timezone | `TIMESTAMP` | Hyper does not support nanosecond precision.
`Timestamp` with timezone | `TIMESTAMPTZ` | Hyper does not support nanosecond precision.
`UTF8` | `Text`


Hyper supports version 1.3 of the Arrow specification. The following Arrow features are not supported:

- Data types: Interval, LargeUtf8, Struct, lists, union, map types, Binary, LargeBinary, FixedSizeBinary, HalfFloat

- Run-end or dictionary encoding

- Arrow internal compression, i.e., LZ4 compression, on record batches (see `RecordBatch::compression` in the [Arrow messages schema](https://github.com/apache/arrow/blob/main/format/Message.fbs))

- Big-Endian encoding

- Duplicated, i.e., non-unique, column names

- Schemas with zero columns

:::note
If an Arrow file contains columns with unsupported data types, Hyper can still read the other columns in the file, as long as you do not select any unsupported columns. This is not the case for unsupported encodings, e.g., dictionary encoding, since Hyper cannot read dictionary-encoded Arrow data.
:::

:::note
Since Hyper parallelizes queries on the granularity of record batches,
we recommend batch sizes of a few thousand rows.
:::
