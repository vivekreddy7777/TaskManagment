using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using System.Data;

public class CompareToolService
{
    private readonly DbContext _db;
    private readonly ILogger<CompareToolService> _logger;

    public CompareToolService(DbContext db, ILogger<CompareToolService> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task<CompareToolResultModel> CallCompareToolAsync(
        string bulkInsertKey,
        string requestorId,
        string compareId,
        string fileName,
        int gridType,
        string additional1,
        string overrideType,
        string region,
        CancellationToken ct = default)
    {
        var pValidationErrors = Out("@ValidationErrors", SqlDbType.VarChar, 100);
        var pValTableName     = Out("@ValTableName",     SqlDbType.VarChar, 100);
        var pWaETempTableName = Out("@WaETempTableName", SqlDbType.VarChar, 100);
        var pTempTableName    = Out("@TempTableName",    SqlDbType.VarChar, 100);
        var pTempLog          = Out("@TempLog",          SqlDbType.VarChar, 6000);

        string sql =
            "EXEC [Quality_Control].[dbo].[procCompareTool_ClaimBulkInsert1] " +
            "@Password, @BulkInsertKey, @LANID, @W_ID, @FileName, @GridType, @Additional1, @OverrideType, @Region, " +
            "@ValidationErrors OUTPUT, @ValTableName OUTPUT, @WaETempTableName OUTPUT, @TempTableName OUTPUT, @TempLog OUTPUT";

        _logger.LogInformation("Executing CompareTool SP with BulkInsertKey: {Key}", bulkInsertKey);

        await _db.Database.ExecuteSqlRawAsync(sql, new[]
        {
            In("@Password",      SqlDbType.VarChar, 50, "MHZ-20U"),
            In("@BulkInsertKey", SqlDbType.VarChar, 100, bulkInsertKey),
            In("@LANID",         SqlDbType.VarChar, 100, requestorId),
            In("@W_ID",          SqlDbType.VarChar, 50,  compareId),
            In("@FileName",      SqlDbType.VarChar, 255, fileName),
            In("@GridType",      SqlDbType.Int,          gridType),
            In("@Additional1",   SqlDbType.VarChar, 200, additional1),
            In("@OverrideType",  SqlDbType.VarChar, 40,  overrideType),
            In("@Region",        SqlDbType.VarChar, 40,  region),

            pValidationErrors, pValTableName, pWaETempTableName, pTempTableName, pTempLog
        }, ct);

        var result = new CompareToolResultModel
        {
            ValidationErrors = pValidationErrors.Value?.ToString(),
            ValTableName     = pValTableName.Value?.ToString(),
            WaETempTableName = pWaETempTableName.Value?.ToString(),
            TempTableName    = pTempTableName.Value?.ToString(),
            TempLog          = pTempLog.Value?.ToString(),
        };

        _logger.LogInformation("CompareTool SP completed. Result: {@Result}", result);

        return result;
    }

    private static SqlParameter In(string name, SqlDbType type, int size, object? value) =>
        new(name, type, size)
        {
            Direction = ParameterDirection.Input,
            Value = value ?? DBNull.Value
        };

    private static SqlParameter In(string name, SqlDbType type, object? value) =>
        new(name, type)
        {
            Direction = ParameterDirection.Input,
            Value = value ?? DBNull.Value
        };

    private static SqlParameter Out(string name, SqlDbType type, int size = 0) =>
        size > 0
            ? new SqlParameter(name, type, size) { Direction = ParameterDirection.Output }
            : new SqlParameter(name, type)       { Direction = ParameterDirection.Output };
}
