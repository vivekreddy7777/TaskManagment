private async Task DebugDb2WhichColumnAsync(string sql, CancellationToken ct)
{
    // Use your DB2 connection string variable; adjust name if needed.
    // If you currently store a DB2Connection object, use .ConnectionString.
    var connStr = _db2Connection.ConnectionString;

    await using var conn = new DB2Connection(connStr);
    await conn.OpenAsync(ct);

    await using var cmd = conn.CreateCommand();
    cmd.CommandText = sql;
    cmd.CommandTimeout = 0;

    await using var reader = await cmd.ExecuteReaderAsync(ct);

    if (!await reader.ReadAsync(ct))
    {
        _log.LogWarning("[DB2 DEBUG] No rows returned for diagnostic query.");
        return;
    }

    // Build property map from your EF keyless DTO
    var propMap = typeof(CcrSmgRow)
        .GetProperties(BindingFlags.Public | BindingFlags.Instance)
        .Where(p => p.CanWrite)
        .ToDictionary(p => p.Name, p => p.PropertyType, StringComparer.OrdinalIgnoreCase);

    _log.LogInformation("[DB2 DEBUG] Inspecting first row. FieldCount={FieldCount}", reader.FieldCount);

    for (int i = 0; i < reader.FieldCount; i++)
    {
        string colName = reader.GetName(i);
        string db2TypeName = Safe(() => reader.GetDataTypeName(i), "<unknown>");
        Type actualDotNetType = Safe(() => reader.GetFieldType(i), typeof(object));

        // If your SELECT returns a column that doesn't exist on the model, log it.
        if (!propMap.TryGetValue(colName, out var expectedType))
        {
            object rawExtra = Safe(() => reader.GetValue(i), "<unreadable>");
            _log.LogWarning(
                "[DB2 DEBUG] Column not in model: Col[{Idx}] {Col} DB2Type={Db2Type} FieldType={FieldType} Raw={Raw}",
                i, colName, db2TypeName, actualDotNetType.Name, FormatRaw(rawExtra));
            continue;
        }

        try
        {
            // IMPORTANT: mimic EF getter choice based on the model property type
            object value = ReadAsExpectedType(reader, i, expectedType);

            _log.LogInformation(
                "[DB2 DEBUG] OK Col[{Idx}] {Col} DB2Type={Db2Type} FieldType={FieldType} Expected={Expected} Value={Val}",
                i, colName, db2TypeName, actualDotNetType.Name, FriendlyType(expectedType), FormatRaw(value));
        }
        catch (Exception ex)
        {
            object raw = Safe(() => reader.GetValue(i), "<unreadable>");

            _log.LogError(ex,
                "[DB2 DEBUG] FAIL Col[{Idx}] {Col} DB2Type={Db2Type} FieldType={FieldType} Expected={Expected} Raw={Raw}",
                i, colName, db2TypeName, actualDotNetType.Name, FriendlyType(expectedType), FormatRaw(raw));

            // Stop at the first failing column so you can fix it quickly
            throw;
        }
    }
}

/// <summary>
/// Reads a column using the getter implied by the expected .NET type (like EF would).
/// Handles nulls and nullable types.
/// </summary>
private static object ReadAsExpectedType(DB2DataReader reader, int ordinal, Type expectedType)
{
    if (reader.IsDBNull(ordinal))
        return DBNull.Value;

    // unwrap Nullable<T>
    var t = Nullable.GetUnderlyingType(expectedType) ?? expectedType;

    if (t == typeof(string)) return reader.GetString(ordinal);
    if (t == typeof(int)) return reader.GetInt32(ordinal);
    if (t == typeof(long)) return reader.GetInt64(ordinal);
    if (t == typeof(short)) return reader.GetInt16(ordinal);
    if (t == typeof(decimal)) return reader.GetDecimal(ordinal);
    if (t == typeof(double)) return reader.GetDouble(ordinal);
    if (t == typeof(float)) return reader.GetFloat(ordinal);
    if (t == typeof(bool)) return reader.GetBoolean(ordinal);
    if (t == typeof(DateTime)) return reader.GetDateTime(ordinal);
    if (t == typeof(Guid)) return reader.GetGuid(ordinal);

    // Fallback: raw object
    return reader.GetValue(ordinal);
}

private static string FriendlyType(Type t)
{
    var u = Nullable.GetUnderlyingType(t);
    return u == null ? t.Name : $"{u.Name}?";
}

private static string FormatRaw(object val)
{
    if (val == null) return "NULL";
    if (val == DBNull.Value) return "DBNULL";
    return val.ToString();
}

private static T Safe<T>(Func<T> func, T fallback)
{
    try { return func(); }
    catch { return fallback; }
}
