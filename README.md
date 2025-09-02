// region 4) stage member rows -> batch DB2 pulls -> map entity ids
private void StageMemberRowsAndMapEntities()
{
    // a) temps equivalent to CLAIM_MEMBER_INFO / DB2_MEMBER_INFO in the SP
    const string createTemps = @"
IF OBJECT_ID('tempdb..#CLAIM_MEMBER_INFO') IS NULL
    CREATE TABLE #CLAIM_MEMBER_INFO (MEMBER_ID varchar(30), PERSON_NBR varchar(30), PROCESSED bit);

IF OBJECT_ID('tempdb..#DB2_MEMBER_INFO') IS NOT NULL DROP TABLE #DB2_MEMBER_INFO;
CREATE TABLE #DB2_MEMBER_INFO (MEMBER_ID varchar(30), PERSON_NBR varchar(30), RVSVD_PINT_ACN bigint);";
    using (var cmd = new SqlCommand(createTemps, _sql)) cmd.ExecuteNonQuery();

    // b) load #CLAIM_MEMBER_INFO from the source temp table you already created in StageClaimRows()
    string src = $"[{_tempTableName}]"; // this is in [Quality_Control].[dbo] per your code; fully qualify if needed
    string insertMembersSql = $@"
INSERT INTO #CLAIM_MEMBER_INFO (MEMBER_ID, PERSON_NBR, PROCESSED)
SELECT DISTINCT CAST([MEMBER #] AS varchar(30)), CAST([PERSON NO] AS varchar(30)), 0
FROM {src}
WHERE [MEMBER #] IS NOT NULL;";
    using (var cmd = new SqlCommand(insertMembersSql, _sql)) cmd.ExecuteNonQuery();

    // c) loop batches (<= 100) -> query DB2 -> insert results into #DB2_MEMBER_INFO -> mark processed
    while (true)
    {
        var batch = GetNextMemberBatch(_sql, 100);
        if (batch.Count == 0) break;

        InsertDb2MemberInfoList(batch);      // fills #DB2_MEMBER_INFO for this batch
        MarkClaimMemberInfoProcessed(batch); // sets PROCESSED = 1 in #CLAIM_MEMBER_INFO for this batch
    }

    // d) map DB2 agent into final table (by MEMBER_ID; scoped by key + wm)
    const string updateAddtnl = @"
UPDATE  t
SET     t.RVRSD_PTNT_ACN = d.RVSVD_PINT_ACN
FROM    dbo.tblCompareClaimsDetails_Addtnl_1_IDs t
JOIN    #DB2_MEMBER_INFO d ON d.MEMBER_ID = t.MEMBER_ID
WHERE   t.BulkInsertKey = @key AND t.WM_ID = @wm;";
    using (var cmd = new SqlCommand(updateAddtnl, _sql))
    {
        cmd.Parameters.AddWithValue("@key", (object)_bulkInsertKey ?? DBNull.Value);
        cmd.Parameters.AddWithValue("@wm",  (object)_wmId        ?? DBNull.Value);
        cmd.ExecuteNonQuery();
    }
}

// --- helpers for the loop (kept minimal) ---

private static List<(string MemberId,string PersonNbr)> GetNextMemberBatch(SqlConnection sql, int take)
{
    var list = new List<(string,string)>();
    const string sqlText = @"
WITH cte AS (
  SELECT TOP (@take) MEMBER_ID, PERSON_NBR
  FROM #CLAIM_MEMBER_INFO WITH (READPAST)
  WHERE PROCESSED = 0
)
SELECT MEMBER_ID, PERSON_NBR FROM cte;";
    using (var cmd = new SqlCommand(sqlText, sql))
    {
        cmd.Parameters.AddWithValue("@take", take);
        using var rdr = cmd.ExecuteReader();
        while (rdr.Read()) list.Add((rdr.GetString(0), rdr.GetString(1)));
    }
    return list;
}

// Query DB2 for the batch and bulk-insert into #DB2_MEMBER_INFO
private void InsertDb2MemberInfoList(List<(string MemberId,string PersonNbr)> batch)
{
    if (batch.Count == 0) return;

    // TODO: replace with your real DB2 query. Build an IN list for MEMBER_IDs, fetch
    // (MEMBER_ID, PERSON_NBR, RVSVD_PINT_ACN) into a DataTable, then bulk-copy.

    var tbl = new DataTable();
    tbl.Columns.Add("MEMBER_ID", typeof(string));
    tbl.Columns.Add("PERSON_NBR", typeof(string));
    tbl.Columns.Add("RVSVD_PINT_ACN", typeof(long));

    // --- sample fill from your existing DB2 code ---
    // foreach (var (mid, pn) in batch) { /* query DB2 and add rows to tbl */ }

    // write to #DB2_MEMBER_INFO
    using var bulk = new SqlBulkCopy(_sql) { DestinationTableName = "#DB2_MEMBER_INFO" };
    bulk.ColumnMappings.Add("MEMBER_ID",      "MEMBER_ID");
    bulk.ColumnMappings.Add("PERSON_NBR",     "PERSON_NBR");
    bulk.ColumnMappings.Add("RVSVD_PINT_ACN", "RVSVD_PINT_ACN");
    bulk.WriteToServer(tbl);
}

private void MarkClaimMemberInfoProcessed(List<(string MemberId,string PersonNbr)> batch)
{
    if (batch.Count == 0) return;

    var sb = new System.Text.StringBuilder("UPDATE #CLAIM_MEMBER_INFO SET PROCESSED = 1 WHERE ");
    var prms = new List<SqlParameter>();
    for (int i = 0; i < batch.Count; i++)
    {
        if (i > 0) sb.Append(" OR ");
        sb.Append($"(MEMBER_ID = @m{i} AND PERSON_NBR = @p{i})");
        prms.Add(new SqlParameter($"@m{i}", batch[i].MemberId));
        prms.Add(new SqlParameter($"@p{i}", batch[i].PersonNbr));
    }
    using var cmd = new SqlCommand(sb.ToString(), _sql);
    cmd.Parameters.AddRange(prms.ToArray());
    cmd.ExecuteNonQuery();
}
