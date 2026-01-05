ct.ThrowIfCancellationRequested();

    if (ids == null || ids.Count == 0)
    {
        _log.LogInformation("[PREP] No ids provided; skipping DB2 member load.");
        return;
    }

    // IMPORTANT: ProcessAsync already has a transaction open.
    // Do NOT start a new transaction here.
    var currentEfTx = _context.Database.CurrentTransaction;
    if (currentEfTx == null)
    {
        throw new InvalidOperationException(
            "[PREP] No active EF transaction found. " +
            "LoadDb2MemberInfoAsync must be called inside the ProcessAsync transaction.");
    }

    // Get the SAME connection + SAME transaction EF is using
    var conn = (SqlConnection)_context.Database.GetDbConnection();
    var tx = (SqlTransaction)currentEfTx.GetDbTransaction();

    // ------------------------------------------------------------
    // 1) Build DB2 query
    // ------------------------------------------------------------
    var idList = string.Join(",", ids.Select(id => $"'{id.Replace("'", "''")}'"));

    var db2Sql = $@"
SELECT MEMBERSHIP_NBR AS MEMBER_ID,
       PERSON_NBR,
       TRANS_INDIV_AGN_ID AS RVRSD_PTNT_AGN
FROM DB2LIB.ELG0_ELGSMK00
WHERE MEMBERSHIP_NBR IN ({idList})
WITH UR";

    _log.LogInformation("[PREP] Executing DB2 query");
    ct.ThrowIfCancellationRequested();

    // ------------------------------------------------------------
    // 2) Fetch from DB2 (your DB2 context)
    // ------------------------------------------------------------
    var db2Members = await _db2Context.Db2MemberInfo
        .FromSqlRaw(db2Sql)
        .ToListAsync(ct);

    _log.LogInformation("[PREP] Retrieved {Count} DB2 member records", db2Members.Count);

    if (db2Members.Count == 0)
    {
        _log.LogInformation("[PREP] No DB2 member records; skipping #DB2_MEMBER_INFO insert.");
        return;
    }

    ct.ThrowIfCancellationRequested();

    // ------------------------------------------------------------
    // 3) Create temp table in SQL Server (SAME conn/tx as ProcessAsync)
    // ------------------------------------------------------------
    var createTempSql = @"
IF OBJECT_ID('tempdb..#DB2_MEMBER_INFO') IS NOT NULL DROP TABLE #DB2_MEMBER_INFO;

CREATE TABLE #DB2_MEMBER_INFO
(
    MEMBER_ID      VARCHAR(30) NOT NULL,
    PERSON_NBR     VARCHAR(30) NULL,
    RVRSD_PTNT_AGN VARCHAR(30) NULL
);";

    _log.LogInformation("[PREP] Creating temp table #DB2_MEMBER_INFO");
    await _context.Database.ExecuteSqlRawAsync(createTempSql, ct);

    ct.ThrowIfCancellationRequested();

    // ------------------------------------------------------------
    // 4) Convert to DataTable for bulk copy
    // ------------------------------------------------------------
    var table = new DataTable();
    table.Columns.Add("MEMBER_ID", typeof(string));
    table.Columns.Add("PERSON_NBR", typeof(string));
    table.Columns.Add("RVRSD_PTNT_AGN", typeof(string));

    foreach (var m in db2Members)
    {
        ct.ThrowIfCancellationRequested();
        table.Rows.Add(m.MEMBER_ID, m.PERSON_NBR, m.RVRSD_PTNT_AGN);
    }

    // ------------------------------------------------------------
    // 5) SqlBulkCopy into #temp using SAME connection + SAME transaction
    // ------------------------------------------------------------
    _log.LogInformation("[PREP] Bulk inserting {Count} rows into #DB2_MEMBER_INFO", table.Rows.Count);

    using (var bulk = new SqlBulkCopy(conn, SqlBulkCopyOptions.TableLock, tx))
    {
        bulk.DestinationTableName = "#DB2_MEMBER_INFO";
        bulk.BulkCopyTimeout = 0;

        // explicit mappings (avoid order bugs)
        bulk.ColumnMappings.Add("MEMBER_ID", "MEMBER_ID");
        bulk.ColumnMappings.Add("PERSON_NBR", "PERSON_NBR");
        bulk.ColumnMappings.Add("RVRSD_PTNT_AGN", "RVRSD_PTNT_AGN");

        await bulk.WriteToServerAsync(table, ct);
    }

    _log.LogInformation("[PREP] Bulk insert complete for #DB2_MEMBER_INFO");
}
