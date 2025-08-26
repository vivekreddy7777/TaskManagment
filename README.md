using System.Data;
using System.Data.Common;
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

// inside your Job class
protected async Task<DataRow> GetMainRequestAsync(string procName, CancellationToken ct = default)
{
    using var scope     = _scopeFactory.CreateScope();
    var dbContext       = scope.ServiceProvider.GetRequiredService<DBServerContext>();
    var conn            = dbContext.Database.GetDbConnection();

    var mustOpen = conn.State == ConnectionState.Closed;
    if (mustOpen) await conn.OpenAsync(ct);

    try
    {
        // 1) run SELECT proc -> DataTable
        using var cmd = conn.CreateCommand();
        cmd.CommandType = CommandType.StoredProcedure;
        cmd.CommandText = procName;                                  // e.g. procBTE_CompareTool_MainRequest_SELECT
        Add(cmd, "@Password",        DbType.String,   Constants.Password);
        Add(cmd, "@Request_Number",  DbType.Int64,    Request_Number);
        Add(cmd, "@UserID",          DbType.String,   UserID);
        Add(cmd, "@ProcResult",      DbType.Int32,    DBNull.Value, ParameterDirection.Output); // if you need it

        using var rdr = await cmd.ExecuteReaderAsync(ct);
        var dt = new DataTable();
        dt.Load(rdr);

        if (dt.Rows.Count == 0)
            throw new JobFailException($"The Job Details could not be retrieved for RequestNumber: {Request_Number}");

        // 2) map whatever you need (column names same as 4.5)
        var row = dt.Rows[0];
        if (dt.Columns.Contains("User_ID"))           UserID         = Convert.ToString(row["User_ID"]);
        if (dt.Columns.Contains("FK_Frame_Request_ID"))  Frame_Request = ToInt64(row["FK_Frame_Request_ID"]);
        if (dt.Columns.Contains("FK_WorkTranx_ID"))      WorkTranx     = ToInt64(row["FK_WorkTranx_ID"]);
        if (dt.Columns.Contains("FK_Priority_ID"))       Priority      = ToInt32(row["FK_Priority_ID"]);
        if (dt.Columns.Contains("FK_Program_ID"))        ProgramID     = Convert.ToString(row["FK_Program_ID"]);

        // 3) (optional) fire tracking UPDATE proc (mirror of 4.5)
        using var upd = conn.CreateCommand();
        upd.CommandType = CommandType.StoredProcedure;
        upd.CommandText = "procBTE_System_MainRequest_Tracking_UPDATE";
        Add(upd, "@Password",    DbType.String, Constants.Password);
        Add(upd, "@UserID",      DbType.String, UserID);
        Add(upd, "@Module",      DbType.String, Module);
        Add(upd, "@Request_Number", DbType.Int64, Request_Number);
        Add(upd, "@MachineName", DbType.String, Environment.MachineName);
        Add(upd, "@ProcResult",  DbType.Int32,  DBNull.Value, ParameterDirection.Output);
        await upd.ExecuteNonQueryAsync(ct);

        return row; // return row so the rest of your code stays the same
    }
    finally
    {
        if (mustOpen) await conn.CloseAsync();
    }

    static void Add(DbCommand c, string name, DbType type, object value, ParameterDirection dir = ParameterDirection.Input)
    {
        var p = c.CreateParameter();
        p.ParameterName = name;
        p.DbType        = type;
        p.Direction     = dir;
        p.Value         = value ?? DBNull.Value;
        c.Parameters.Add(p);
    }

    static long ToInt64(object v) => v == null || v == DBNull.Value ? 0L : Convert.ToInt64(v);
    static int  ToInt32(object v) => v == null || v == DBNull.Value ? 0  : Convert.ToInt32(v);
}
