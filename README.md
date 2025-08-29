using (var cmd = conn.CreateCommand())
{
    cmd.CommandType = CommandType.StoredProcedure;
    cmd.CommandText = "procCompareTool_ClaimBulkInsert1";

    // ===== INPUTS =====
    cmd.Parameters.Add(new SqlParameter("@Password",     SqlDbType.VarChar, 10)   { Value = Constants.Password });

    // GM_ID, RequestorID are VARCHAR in your proc
    cmd.Parameters.Add(new SqlParameter("@GM_ID",        SqlDbType.VarChar, 50)   { Value = GM_ID ?? (object)DBNull.Value });
    cmd.Parameters.Add(new SqlParameter("@RequestorID",  SqlDbType.VarChar, 50)   { Value = RequestorID ?? (object)DBNull.Value });

    // FileName is NVARCHAR(500)
    cmd.Parameters.Add(new SqlParameter("@FileName",     SqlDbType.NVarChar, 500) { Value = FileName ?? (object)DBNull.Value });

    // GridType is INT in the proc
    cmd.Parameters.Add(new SqlParameter("@GridType",     SqlDbType.Int)           { Value = GridType });

    // Tweak lengths if your proc uses different sizes
    cmd.Parameters.Add(new SqlParameter("@Additional",   SqlDbType.VarChar, 100)  { Value = Additional ?? (object)DBNull.Value });
    cmd.Parameters.Add(new SqlParameter("@OverrideType", SqlDbType.VarChar, 50)   { Value = OverrideType ?? (object)DBNull.Value });
    cmd.Parameters.Add(new SqlParameter("@Region",       SqlDbType.VarChar, 50)   { Value = Region ?? (object)DBNull.Value });

    // ===== OUTPUTS =====
    // BIT
    var pValidationErrors = new SqlParameter("@ValidationErrors", SqlDbType.Bit)
    { Direction = ParameterDirection.Output };
    cmd.Parameters.Add(pValidationErrors);

    // VARCHAR(28)
    var pBulkInsertKey = new SqlParameter("@BulkInsertKey", SqlDbType.VarChar, 28)
    { Direction = ParameterDirection.Output };
    cmd.Parameters.Add(pBulkInsertKey);

    // If your proc returns an error text (e.g., VARCHAR(MAX))
    var pErrorText = new SqlParameter("@ErrorText", SqlDbType.VarChar, -1) // -1 => MAX
    { Direction = ParameterDirection.Output };
    cmd.Parameters.Add(pErrorText);

    // SYSNAME => NVARCHAR(128)
    var pTableName        = new SqlParameter("@TableName",        SqlDbType.NVarChar, 128) { Direction = ParameterDirection.Output };
    var pValTableName     = new SqlParameter("@ValTableName",     SqlDbType.NVarChar, 128) { Direction = ParameterDirection.Output };
    var pEvalTempTable    = new SqlParameter("@EvalTempTableName",SqlDbType.NVarChar, 128) { Direction = ParameterDirection.Output };
    var pTempTableName    = new SqlParameter("@TempTableName",    SqlDbType.NVarChar, 128) { Direction = ParameterDirection.Output };
    var pEnvironment      = new SqlParameter("@Environment",      SqlDbType.NVarChar, 50)  { Direction = ParameterDirection.Output }; // or input if proc expects input

    cmd.Parameters.AddRange(new[]
    {
        pTableName, pValTableName, pEvalTempTable, pTempTableName, pEnvironment
    });

    await cmd.ExecuteNonQueryAsync(ct);

    // read outputs
    bool   validationErrors = pValidationErrors.Value != DBNull.Value && (bool)pValidationErrors.Value;
    string bulkInsertKey    = pBulkInsertKey.Value as string ?? "";
    string tableName        = pTableName.Value as string ?? "";
    string valTableName     = pValTableName.Value as string ?? "";
    string evalTempTable    = pEvalTempTable.Value as string ?? "";
    string tempTableName    = pTempTableName.Value as string ?? "";
    string environmentOut   = pEnvironment.Value as string ?? "";
    string errorText        = pErrorText.Value as string ?? "";

    // ...use these as needed
}
