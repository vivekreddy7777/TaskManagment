using System.Data;
using System.Threading;
using Microsoft.EntityFrameworkCore;

private async Task<string?> GetSystemVariableAsync(
    string variable,
    string module,
    CancellationToken ct = default)
{
    // Create a scoped DB context
    using var scope = _scopeFactory.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<DBServerContext>();

    // Reuse the EF connection
    var conn = dbContext.Database.GetDbConnection();
    var mustOpen = conn.State != ConnectionState.Open;
    if (mustOpen) await conn.OpenAsync(ct);

    try
    {
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT [dbo].[fn_SystemVariable](@var, @mod) AS [Value]";
        cmd.CommandType = CommandType.Text;

        // @var
        var pVar = cmd.CreateParameter();
        pVar.ParameterName = "@var";
        pVar.DbType = DbType.String;
        pVar.Value = variable ?? (object)DBNull.Value;
        cmd.Parameters.Add(pVar);

        // @mod
        var pMod = cmd.CreateParameter();
        pMod.ParameterName = "@mod";
        pMod.DbType = DbType.String;
        pMod.Value = module ?? (object)DBNull.Value;
        cmd.Parameters.Add(pMod);

        var result = await cmd.ExecuteScalarAsync(ct);
        return result == DBNull.Value ? null : Convert.ToString(result);
    }
    finally
    {
        if (mustOpen) await conn.CloseAsync();
    }
}
