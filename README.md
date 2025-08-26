using System.Data;
using Microsoft.Data.SqlClient;            // <- ensure this is referenced
using Microsoft.EntityFrameworkCore;

protected async Task<DataTable> GetMainRequestsAsync(
    string procName,
    CancellationToken ct = default)
{
    // 1) Create a scope and get the EF Core DbContext
    using var scope = _scopeFactory.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<DBServerContext>();

    // 2) Get the raw ADO.NET connection from EF Core
    await using var conn = dbContext.Database.GetDbConnection();
    var mustClose = conn.State != ConnectionState.Open;
    if (mustClose) await conn.OpenAsync(ct);

    // 3) Build the stored-proc command + parameters
    await using var cmd = conn.CreateCommand();
    cmd.CommandText = procName;                         // e.g., "procBTE_CompareTool_MainRequest_SELECT"
    cmd.CommandType = CommandType.StoredProcedure;

    // parameters
    var pPwd   = cmd.CreateParameter(); pPwd.ParameterName = "@Password";       pPwd.DbType = DbType.String; pPwd.Value = Constants.Password; cmd.Parameters.Add(pPwd);
    var pReq   = cmd.CreateParameter(); pReq.ParameterName = "@Request_Number"; pReq.DbType = DbType.Int64;  pReq.Value = Request_Number;     cmd.Parameters.Add(pReq);
    var pUser  = cmd.CreateParameter(); pUser.ParameterName = "@UserID";        pUser.DbType = DbType.String;pUser.Value = UserID;            cmd.Parameters.Add(pUser);

    // optional output param if your proc supports it
    var pOut   = cmd.CreateParameter(); pOut.ParameterName = "@ProcResult";     pOut.DbType = DbType.Int32;  pOut.Direction = ParameterDirection.Output; cmd.Parameters.Add(pOut);

    // 4) Execute and load into a DataTable
    var table = new DataTable();
    await using (var reader = await cmd.ExecuteReaderAsync(ct))
    {
        table.Load(reader);
    }

    if (mustClose) await conn.CloseAsync();

    // 5) Validate and use the row(s)
    if (table.Rows.Count == 0)
        throw new JobFailException($"The Job Details could not be retrieved for RequestNumber: {Request_Number}");

    var row = table.Rows[0];

    // Helpers to read safely
    static T Get<T>(DataRow r, string col, T def = default!)
        => r.Table.Columns.Contains(col) && r[col] != DBNull.Value
           ? (T)Convert.ChangeType(r[col], typeof(T))
           : def;

    // Example mappings (adjust column names as needed)
    UserID        = Get<string>(row, "User_ID", UserID);
    Frame_Request = Get<long?>(row, "FK_Frame_Request_ID");
    WorkTranx     = Get<long?>(row, "FK_WorkTranx_ID");
    Priority      = Get<int?>(row, "FK_Priority_ID");
    ProgramID     = Get<string>(row, "FK_Program_ID");

    // 6) (Optional) Call your tracking UPDATE proc, still using the EF connection
    try
    {
        await using var conn2 = dbContext.Database.GetDbConnection();
        if (conn2.State != ConnectionState.Open) await conn2.OpenAsync(ct);

        await using var upd = conn2.CreateCommand();
        upd.CommandText = "procBTE_System_MainRequest_Tracking_UPDATE";
        upd.CommandType = CommandType.StoredProcedure;

        void Add(DbCommand c, string name, object value, DbType type = DbType.String)
        {
            var p = c.CreateParameter();
            p.ParameterName = name; p.DbType = type; p.Value = value ?? DBNull.Value;
            c.Parameters.Add(p);
        }

        Add(upd, "@Password",   Constants.Password);
        Add(upd, "@UserID",     UserID);
        Add(upd, "@Module",     Module);
        Add(upd, "@Request_Number", Request_Number, DbType.Int64);
        Add(upd, "@MachineName", Environment.MachineName);
        var out2 = upd.CreateParameter(); out2.ParameterName="@ProcResult"; out2.DbType=DbType.Int32; out2.Direction=ParameterDirection.Output; upd.Parameters.Add(out2);

        await upd.ExecuteNonQueryAsync(ct);
    }
    catch
    {
        // swallow on purpose like your previous code; logging here is fine
    }

    return table; // if callers still need the table
}
