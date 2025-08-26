// Get values from DB command or service variables before building args
string bulkInsertKey   = Convert.ToString(cmd.Parameters["@BulkInsertKey"]?.Value);
string requestorId     = Convert.ToString(cmd.Parameters["@WM_ID"]?.Value);
string fileName        = Convert.ToString(cmd.Parameters["@FileName"]?.Value);
string gridType        = Convert.ToString(cmd.Parameters["@GridType"]?.Value);
string additional      = Convert.ToString(cmd.Parameters["@Additional"]?.Value);
string overrideType    = Convert.ToString(cmd.Parameters["@OverrideType"]?.Value);
string region          = Convert.ToString(cmd.Parameters["@Region"]?.Value);
string tableName       = Convert.ToString(cmd.Parameters["@TableName"]?.Value);
string valTableName    = Convert.ToString(cmd.Parameters["@ValTableName"]?.Value);
string valTempTable    = Convert.ToString(cmd.Parameters["@ValTempTableName"]?.Value);
string tempTableName   = Convert.ToString(cmd.Parameters["@TempTableName"]?.Value);
string environment     = Convert.ToString(cmd.Parameters["@Environment"]?.Value);

// Validate before passing to console
if (new [] { bulkInsertKey, requestorId, fileName, gridType, additional, 
             overrideType, region, tableName, valTableName, valTempTable,
             tempTableName, environment }.Any(string.IsNullOrWhiteSpace))
{
    throw new InvalidOperationException("One or more required arguments for console call are missing!");
}

// Now build the argsList
var argsList = new[]
{
    bulkInsertKey,
    requestorId,
    fileName,
    gridType,
    additional,
    overrideType,
    region,
    tableName,
    valTableName,
    valTempTable,
    tempTableName,
    environment
};
