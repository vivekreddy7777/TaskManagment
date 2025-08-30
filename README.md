// after await cmd.ExecuteNonQueryAsync(ct);

// normalize helper
static string S(object v) => v == null || v == DBNull.Value ? string.Empty : v.ToString();
static string I(int? v)    => v.HasValue ? v.Value.ToString() : string.Empty;

// pull outputs
bool validationErrors = pValidationErrors.Value != DBNull.Value && (bool)pValidationErrors.Value;
string bulkInsertKey  = S(pBulkInsertKey.Value);
string tableName      = S(pTableName.Value);
string valTableName   = S(pValTableName.Value);
string evalTempTable  = S(pEvalTempTable.Value);
string tempTableName  = S(pTempTableName.Value);
string environment    = S(pEnvironment.Value);

// normalize inputs
string requestorId   = S(RequestorID);
string fileName      = S(FileName);
string gridTypeStr   = I(GridType);
string additionalStr = I(Additional);
string overrideType  = S(OverrideType);
string region        = S(Region);

// build args list in correct order (12 total)
var argsList = new[]
{
    bulkInsertKey,   // 1 OUT
    requestorId,     // 2 IN
    fileName,        // 3 IN
    gridTypeStr,     // 4 IN (int)
    additionalStr,   // 5 IN (int)
    overrideType,    // 6 IN
    region,          // 7 IN
    tableName,       // 8 OUT
    valTableName,    // 9 OUT
    evalTempTable,   // 10 OUT
    tempTableName,   // 11 OUT
    environment      // 12 OUT
};

// sanity check logging
_logger.LogInformation("ArgsList (Count={Count}): {Args}",
    argsList.Length, string.Join("|", argsList.Select(a => $"[{a}]")));

// guard
if (argsList.Length != 12)
    throw new InvalidOperationException("Console expects 12 arguments.");
