// 1) call the proc and capture OUT params
string bulkInsertKey, fileName, gridType, additional, overrideType,
       region, tableName, valTableName, valTempTableName, tempTableName, environment;

using (var cmd = conn.CreateCommand())
{
    cmd.CommandType = CommandType.StoredProcedure;
    cmd.CommandText = "procCompareTool_ClaimBulkInsert";

    Add(cmd, "@Password",        DbType.String,  "HIZ-ZOU");
    Add(cmd, "@WM_ID",           DbType.Int64,   RequestorID);
    Add(cmd, "@Compare_ID",      DbType.String,  Compare_ID);
    Add(cmd, "@FileName",        DbType.String,  FileName);
    Add(cmd, "@GridType",        DbType.String,  GridType);
    Add(cmd, "@Additional",      DbType.String,  Additional);
    Add(cmd, "@OverrideType",    DbType.String,  OverrideType);
    Add(cmd, "@Region",          DbType.String,  Region);

    // OUT params
    Add(cmd, "@ValidationErrors", DbType.Boolean, null, ParameterDirection.Output);
    Add(cmd, "@BulkInsertKey",    DbType.String,  null, ParameterDirection.Output);
    Add(cmd, "@TableName",        DbType.String,  null, ParameterDirection.Output);
    Add(cmd, "@valTableName",     DbType.String,  null, ParameterDirection.Output);
    Add(cmd, "@valTempTableName", DbType.String,  null, ParameterDirection.Output);
    Add(cmd, "@tempTableName",    DbType.String,  null, ParameterDirection.Output);
    Add(cmd, "@Environment",      DbType.String,  null, ParameterDirection.Output);

    await cmd.ExecuteNonQueryAsync(ct);

    // ✅ copy values while cmd is still in scope
    bool   validationErrors = ToBool( cmd.Parameters["@ValidationErrors"]?.Value );
    bulkInsertKey           = ToStr(  cmd.Parameters["@BulkInsertKey"]?.Value );
    tableName               = ToStr(  cmd.Parameters["@TableName"]?.Value );
    valTableName            = ToStr(  cmd.Parameters["@valTableName"]?.Value );
    valTempTableName        = ToStr(  cmd.Parameters["@valTempTableName"]?.Value );
    tempTableName           = ToStr(  cmd.Parameters["@tempTableName"]?.Value );
    environment             = ToStr(  cmd.Parameters["@Environment"]?.Value );

    // if you also want these inputs normalized as strings:
    fileName    = ToStr(cmd.Parameters["@FileName"]?.Value);
    gridType    = ToStr(cmd.Parameters["@GridType"]?.Value);
    additional  = ToStr(cmd.Parameters["@Additional"]?.Value);
    overrideType= ToStr(cmd.Parameters["@OverrideType"]?.Value);
    region      = ToStr(cmd.Parameters["@Region"]?.Value);
} // ← after this, DO NOT reference cmd

// helpers
static string ToStr(object v) => v == null || v == DBNull.Value ? string.Empty : Convert.ToString(v);
static bool   ToBool(object v) => v != null && v != DBNull.Value && Convert.ToBoolean(v);
