var sql =
    "UPDATE " + serverDB + ".[dbo].[MCR_GridTracker_SOURCE] " +
    "SET [CopayGridStatus]    = @CopayGridStatus, " +
    "    [GridEffectiveDate] = @GridEffectiveDate, " +
    "    [PlanYear]          = @PlanYear, " +
    "    [AccumPlatform]     = @AccumPlatform, " +
    "    [Comments]          = @Comments " +
    "WHERE [fkMasterID]      = @fkMasterID";

var parameters = new[]
{
    new SqlParameter("@CopayGridStatus",   (object?)response.CopayGridStatus   ?? DBNull.Value),
    new SqlParameter("@GridEffectiveDate", (object?)response.GridEffectiveDate ?? DBNull.Value),
    new SqlParameter("@PlanYear",          (object?)response.PlanYear          ?? DBNull.Value),
    new SqlParameter("@AccumPlatform",     (object?)response.AccumPlatform     ?? DBNull.Value),
    new SqlParameter("@Comments",          (object?)response.Comments          ?? DBNull.Value),
    new SqlParameter("@fkMasterID",        response.fkMasterID)
};

await _linkedServerContext.Database.ExecuteSqlRawAsync(sql, parameters);
