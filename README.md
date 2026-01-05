tx
    // =====================================================================
    private async Task LoadDb2MemberInfoAsync(List<string> ids, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();

        if (ids == null || ids.Count == 0)
        {
            _log.LogInformation("[PREP] No ids provided; skipping DB2 member load.");
            return;
        }

        // -----------------------------------------------------------------
        // 1) Build DB2 query
        // -----------------------------------------------------------------
        var idList = string.Join(",", ids.Select(id => $"'{id.Replace("'", "''")}'"));

        var db2Sql = $@"
SELECT MEMBERSHIP_NBR AS MEMBER_ID, PERSON_NBR, TRANS_INDIV_AGN_ID AS RVRSD_PTNT_AGN
FROM DB2LIB.ELG0_ELGSMK00
WHERE MEMBERSHIP_NBR IN ({idList})
WITH UR";

        _log.LogInformation("[PREP] Executing DB2 query");
        ct.ThrowIfCancellationRequested();

        // -----------------------------------------------------------------
        // 2) Fetch from DB2 (EF Core)
        // -----------------------------------------------------------------
        var db2Members = await _db2Context.Db2MemberInfo
            .FromSqlRaw(db2Sql)
            .ToListAsync(ct);

        _log.LogInformation("[PREP] Retrieved {Count} DB2 member records", db2Members.Count);

        if (db2Members.Count == 0)
        {
            _log.LogInformation("[PREP] No DB2 member records to insert into #DB2_MEMBER_INFO");
            return;
        }

        ct.ThrowIfCancellationRequested();

        // -----------------------------------------------------------------
        // 3) Convert to DataTable for SqlBulkCopy
        // -----------------------------------------------------------------
        var table = new DataTable();
        table.Columns.Add("MEMBER_ID", typeof(string));
        table.Columns.Add("PERSON_NBR", typeof(string));
        table.Columns.Add("RVRSD_PTNT_AGN", typeof(string));

        foreach (var m in db2Members)
        {
            ct.ThrowIfCancellationRequested();
            table.Rows.Add(m.MEMBER_ID, m.PERSON_NBR, m.RVRSD_PTNT_AGN);
        }

        // -----------------------------------------------------------------
        // 4) IMPORTANT: Force EF to use ONE connection and ONE transaction
        //    so the temp table (#...) and SqlBulkCopy are in the same scope.
        // -----------------------------------------------------------------
        await _context.Database.OpenConnectionAsync(ct);

        await using var efTx = await _context.Database.BeginTransactionAsync(ct);

        try
        {
            var conn = (SqlConnection)_context.Database.GetDbConnection();
            var tx = (SqlTransaction)efTx.GetDbTransaction();

            // -----------------------------------------------------------------
            // 5) Create the temp table on the SAME connection/transaction
            // -----------------------------------------------------------------
            var createTempSql = @"
IF OBJECT_ID('tempdb..#DB2_MEMBER_INFO') IS NOT NULL DROP TABLE #DB2_MEMBER_INFO;

CREATE TABLE #DB2_MEMBER_INFO
(
    MEMBER_ID       VARCHAR(30) NOT NULL,
    PERSON_NBR      VARCHAR(30) NULL,
    RVRSD_PTNT_AGN  VARCHAR(30) NULL
);";

            _log.LogInformation("[PREP] Creating temp table #DB2_MEMBER_INFO");
            await _context.Database.ExecuteSqlRawAsync(createTempSql, ct);

            ct.ThrowIfCancellationRequested();

            // -----------------------------------------------------------------
            // 6) Bulk copy into the temp table using SAME connection + SAME tx
            // -----------------------------------------------------------------
            _log.LogInformation("[PREP] Bulk inserting into #DB2_MEMBER_INFO");

            using (var bulkCopy = new SqlBulkCopy(conn, SqlBulkCopyOptions.TableLock, tx))
            {
                bulkCopy.DestinationTableName = "#DB2_MEMBER_INFO";
                bulkCopy.BulkCopyTimeout = 0;

                // explicit mappings (prevents column order bugs)
                bulkCopy.ColumnMappings.Add("MEMBER_ID", "MEMBER_ID");
                bulkCopy.ColumnMappings.Add("PERSON_NBR", "PERSON_NBR");
                bulkCopy.ColumnMappings.Add("RVRSD_PTNT_AGN", "RVRSD_PTNT_AGN");

                await bulkCopy.WriteToServerAsync(table, ct);
            }

            _log.LogInformation("[PREP] Bulk inserted {Count} rows into #DB2_MEMBER_INFO", table.Rows.Count);

            // -----------------------------------------------------------------
            // 7) Commit transaction (temp table still exists until connection closes)
            // -----------------------------------------------------------------
            await efTx.CommitAsync(ct);
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "[PREP] Error during DB2 member load/bulk insert");
            try { await efTx.RollbackAsync(ct); } catch { /* ignore rollback errors */ }
            throw;
        }
        finally
        {
            // IMPORTANT:
            // If later code needs to query #DB2_MEMBER_INFO, you MUST keep the
            // connection open until that later query completes.
            //
            // If this method ends here and you close the connection, the temp
            // table is gone.
            //
            // If you need the temp table in later methods, DO NOT close here;
            // instead manage OpenConnection/CloseConnection around the whole PREP flow.
            await _context.Database.CloseConnectionAsync();
        }
    }
}
