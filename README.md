private async Task DebugDb2WhichColumnAsync(string sql, CancellationToken ct)
{
    await using var conn = new DB2Connection(_db2Connection.ConnectionString);
    await conn.OpenAsync(ct);

    await using var cmd = conn.CreateCommand();
    cmd.CommandText = sql;
    cmd.CommandTimeout = 0;

    await using var reader = await cmd.ExecuteReaderAsync(ct);

    if (!await reader.ReadAsync(ct))
    {
        _log.LogWarning("[DB2 DEBUG] No rows returned.");
        return;
    }

    // Map model property -> type (case-insensitive)
    var propMap = typeof(CcrSmgRow)
        .GetProperties(BindingFlags.Public | BindingFlags.Instance)
        .Where(p => p.CanWrite)
        .ToDictionary(p => p.Name, p => p.PropertyType, StringComparer.OrdinalIgnoreCase);

    _log.LogInformation("[DB2 DEBUG] Inspecting first row. FieldCount={FieldCount}", reader.FieldCount);

    for (int i = 0; i < reader.FieldCount; i++)
    {
        var colName = reader.GetName(i);
        var db2Type = reader.GetDataTypeName(i);
        var fieldType = reader.GetFieldType(i);

        if (!propMap.TryGetValue(colName, out var expectedType))
        {
            object rawExtra = reader.IsDBNull(i) ? DBNull.Value : reader.GetValue(i);
            _log.LogWarning(
                "[DB2 DEBUG] Column not in model: Col[{Idx}] {Col} DB2Type={Db2Type} FieldType={FieldType} Raw={Raw}",
                i, colName, db2Type, fieldType.Name, rawExtra == DBNull.Value ? "DBNULL" : rawExtra);
            continue;
        }

        try
        {
            object val = ReadAsExpectedType(reader, i, expectedType);

            _log.LogInformation(
                "[DB2 DEBUG] OK Col[{Idx}] {Col} DB2Type={Db2Type} FieldType={FieldType} Expected={Expected} Val={Val}",
                i, colName, db2Type, fieldType.Name, FriendlyType(expectedType),
                val == DBNull.Value ? "DBNULL" : val);
        }
        catch (Exception ex)
        {
            object raw;
            try { raw = reader.GetValue(i); }
            catch { raw = "<unreadable>"; }

            _log.LogError(ex,
                "[DB2 DEBUG] FAIL Col[{Idx}] {Col} DB2Type={Db2Type} FieldType={FieldType} Expected={Expected} Raw={Raw}",
                i, colName, db2Type, fieldType.Name, FriendlyType(expectedType),
                raw == DBNull.Value ? "DBNULL" : raw);

            throw; // stop at first real mismatch
        }
    }
}

private static object ReadAsExpectedType(DB2DataReader reader, int ordinal, Type expectedType)
{
    if (reader.IsDBNull(ordinal))
        return DBNull.Value;

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

    return reader.GetValue(ordinal);
}

private static string FriendlyType(Type t)
{
    var u = Nullable.GetUnderlyingType(t);
    return u == null ? t.Name : $"{u.Name}?";
}
