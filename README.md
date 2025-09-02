private void StageMemberRowsAndMapEntities()
{
    // temp table for DB2 results (matches SP: #DB2_MEMBER_INFO)
    const string createDb2Temp = @"
IF OBJECT_ID('tempdb..#DB2_MEMBER_INFO') IS NOT NULL DROP TABLE #DB2_MEMBER_INFO;
CREATE TABLE #DB2_MEMBER_INFO
(
    MEMBER_ID      varchar(30),
    PERSON_NBR     varchar(30),
    RVSVD_PINT_ACN bigint
);";
    using (var cmd = new SqlCommand(createDb2Temp, _sql))
        cmd.ExecuteNonQuery();

    // loop: pull MEMBER_ID/PERSON_NBR batches from #CLAIM_MEMBER_INFO (unprocessed)
    while (true)
    {
        var batch = GetNextMemberBatch(_sql, 100);
        if (batch.Count == 0) break;

        // query DB2 for the batch and insert results into #DB2_MEMBER_INFO
        InsertDb2MemberInfoForBatch(batch);

        // mark those temp rows as processed
        MarkClaimMemberInfoProcessed(batch);
    }

    // Map DB2 ids back to Addtnl_1_IDs (SP: the UPDATE that joins #DB2_MEMBER_INFO)
    const string updateAddtnl1 = @"
UPDATE t
SET t.RVRSD_PTNT_ACN = d.RVSVD_PINT_ACN
FROM dbo.tblCompareClaimsDetails_Addtnl_1_IDs t
JOIN #DB2_MEMBER_INFO d
  ON d.MEMBER_ID  = t.MEMBER_ID
 AND d.PERSON_NBR = t.PERSON_NBR
WHERE t.BulkInsertKey = @key AND t.WM_ID = @wm;";

    using (var cmd = new SqlCommand(updateAddtnl1, _sql))
    {
        cmd.Parameters.AddWithValue("@key", _bulkInsertKey);
        cmd.Parameters.AddWithValue("@wm", _wmId);
        cmd.ExecuteNonQuery();
    }

    // (SP deletes #DB2_MEMBER_INFO afterward)
    using (var cmd = new SqlCommand("IF OBJECT_ID('tempdb..#DB2_MEMBER_INFO') IS NOT NULL DROP TABLE #DB2_MEMBER_INFO;", _sql))
        cmd.ExecuteNonQuery();
}

// pull next TOP (@take) MEMBER_ID/PERSON_NBR where PROCESSED = 0
private static List<(string MemberId, string PersonNbr)> GetNextMemberBatch(SqlConnection sql, int take)
{
    var list = new List<(string, string)>();

    var text = @"
WITH cte AS (
    SELECT TOP (@take) MEMBER_ID, PERSON_NBR
    FROM #CLAIM_MEMBER_INFO WITH (READPAST)
    WHERE PROCESSED = 0
    ORDER BY MEMBER_ID, PERSON_NBR
)
SELECT MEMBER_ID, PERSON_NBR FROM cte;";
    using (var cmd = new SqlCommand(text, sql))
    {
        cmd.Parameters.AddWithValue("@take", take);
        using var rdr = cmd.ExecuteReader();
        while (rdr.Read())
            list.Add((rdr.GetString(0), rdr.GetString(1)));
    }
    return list;
}

// query DB2 for (MEMBER_ID, PERSON_NBR) batch; land to #DB2_MEMBER_INFO
private void InsertDb2MemberInfoForBatch(List<(string MemberId, string PersonNbr)> batch)
{
    if (batch.Count == 0) return;

    // build parameter list for DB2 query `(?, ?), (?, ?), ...`
    var sbParams = new StringBuilder();
    var parms = new List<DB2Parameter>();
    for (int i = 0; i < batch.Count; i++)
    {
        if (i > 0) sbParams.Append(", ");
        sbParams.Append("(?, ?)");
        parms.Add(new DB2Parameter { DB2Type = DB2Type.VarChar, Value = batch[i].MemberId });
        parms.Add(new DB2Parameter { DB2Type = DB2Type.VarChar, Value = batch[i].PersonNbr });
    }

    // NOTE: replace YOUR_DB2_MEMBER_TABLE with the real DB2 table/view
    var db2Sql = $@"
WITH batch(MEMBER_ID, PERSON_NBR) AS (VALUES {sbParams})
SELECT m.MEMBER_ID, m.PERSON_NBR, m.RVSVD_PINT_ACN
FROM YOUR_DB2_MEMBER_TABLE AS m
JOIN batch b ON b.MEMBER_ID = m.MEMBER_ID AND b.PERSON_NBR = m.PERSON_NBR
WITH UR;";

    using var db2Cmd = _db2.CreateCommand();
    db2Cmd.CommandText = db2Sql;
    foreach (var p in parms) db2Cmd.Parameters.Add(p);

    var dt = new DataTable();
    using (var da = new DB2DataAdapter(db2Cmd))
        da.Fill(dt);

    // bulk copy into SQL temp table #DB2_MEMBER_INFO
    using var bulk = new SqlBulkCopy(_sql) { DestinationTableName = "#DB2_MEMBER_INFO" };
    bulk.ColumnMappings.Add("MEMBER_ID", "MEMBER_ID");
    bulk.ColumnMappings.Add("PERSON_NBR", "PERSON_NBR");
    bulk.ColumnMappings.Add("RVSVD_PINT_ACN", "RVSVD_PINT_ACN");
    bulk.WriteToServer(dt);
}

// mark processed rows in #CLAIM_MEMBER_INFO
private void MarkClaimMemberInfoProcessed(List<(string MemberId, string PersonNbr)> batch)
{
    if (batch.Count == 0) return;

    var sb = new StringBuilder();
    var prms = new List<SqlParameter>();
    for (int i = 0; i < batch.Count; i++)
    {
        if (i > 0) sb.Append(" OR ");
        sb.Append("(MEMBER_ID = @m" + i + " AND PERSON_NBR = @p" + i + ")");
        prms.Add(new SqlParameter("@m" + i, batch[i].MemberId));
        prms.Add(new SqlParameter("@p" + i, batch[i].PersonNbr));
    }

    var sql = $"UPDATE #CLAIM_MEMBER_INFO SET PROCESSED = 1 WHERE {sb};";
    using var cmd = new SqlCommand(sql, _sql);
    cmd.Parameters.AddRange(prms.ToArray());
    cmd.ExecuteNonQuery();
}
