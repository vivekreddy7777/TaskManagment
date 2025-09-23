DECLARE @Schema SYSNAME = 'dbo';
DECLARE @Table  SYSNAME = 'YourTableName';

DECLARE @sql NVARCHAR(MAX) = N'';

;WITH cols AS (
  SELECT
      s.name  AS SchemaName,
      t.name  AS TableName,
      c.name  AS ColumnName,
      TYPE_NAME(c.user_type_id) AS DataType,
      c.max_length AS DeclaredMaxBytes,
      c.column_id
  FROM sys.schemas s
  JOIN sys.tables  t ON t.schema_id = s.schema_id
  JOIN sys.columns c ON c.object_id = t.object_id
  WHERE s.name=@Schema AND t.name=@Table
)
SELECT @sql = STRING_AGG(CONVERT(NVARCHAR(MAX),
N'SELECT ' +
  'N''' + ColumnName + ''' AS ColumnName,' +
  'N''' + DataType + '''  AS DataType,' +
  CAST(DeclaredMaxBytes AS NVARCHAR(10)) + ' AS DeclaredMaxBytes,' +
  -- Max characters (divide by 2 for n* types)
  'MAX(CASE WHEN ''' + DataType + ''' IN (''nvarchar'',''nchar'') ' +
      'THEN DATALENGTH(' + QUOTENAME(ColumnName) + ')/2 ' +
      'ELSE DATALENGTH(' + QUOTENAME(ColumnName) + ') END) AS MaxChars,' +
  -- Max bytes on disk for the column
  'MAX(DATALENGTH(' + QUOTENAME(ColumnName) + ')) AS MaxBytes,' +
  -- Null count
  'SUM(CASE WHEN ' + QUOTENAME(ColumnName) + ' IS NULL THEN 1 ELSE 0 END) AS NullCount ' +
'FROM ' + QUOTENAME(@Schema) + '.' + QUOTENAME(@Table)
), ' UNION ALL ')
FROM cols
ORDER BY column_id;

EXEC sys.sp_executesql @sql;
