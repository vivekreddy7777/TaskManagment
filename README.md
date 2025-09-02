private (List<string> ids, string csv) BuildEntityList(SqlConnection sql)
{
    const string getIdsSql = @"
        SELECT TOP (150) MEMBER_ID
        FROM #CLAIM_MEMBER_INFO
        WHERE PROCESSED = 0
        ORDER BY MEMBER_ID;";

    var ids = new List<string>();
    using (var cmd = new SqlCommand(getIdsSql, sql))
    using (var rdr = cmd.ExecuteReader())
        while (rdr.Read())
            ids.Add(rdr.GetString(0));

    // mimic: SELECT STUFF((SELECT ',' + '''' + MEMBER_ID + '''' ...),1,1,'')
    // -> "'A','B','C'"
    var quoted = ids.Select(id => $"'{id.Replace("'", "''")}'");
    var csv = string.Join(",", quoted);

    Console.WriteLine($"[DEBUG] @EntityList: {csv}");
    return (ids, csv);
}

// 2) OPENQUERY equivalent against DB2 and INSERT into #DB2_MEMBER_INFO
//    SP columns: MEMBERSHIP_NBR, PERSON_NBR, TRANS_INDIV_AGN_ID
//    Target temp table: #DB2_MEMBER_INFO (MEMBER_ID, PERSON_NBR, RVRSD_PTNT_ACN)
private void OpenQueryIntoDb2MemberInfo(SqlConnection sql, DB2Connection db2)
{
    // a) get the next 150 unprocessed MEMBER_IDs (as the SP does each loop)
    var (ids, csv) = BuildEntityList(sql);
    if (ids.Count == 0)
    {
        Console.WriteLine("[DEBUG] No unprocessed MEMBER_ID rows left.");
        return;
    }

    // b) Build a parameterized DB2 IN (...) rather than string-splicing
    //    (DB2 .NET supports positional '?' parameters)
    var placeholders = string.Join(",", Enumerable.Range(0, ids.Count).Select(_ => "?"));

    // IMPORTANT: swap in your real DB2 schema/table name used in the SP
    var db2Sql = $@"
        SELECT
            MEMBERSHIP_NBR AS MEMBER_ID,
            PERSON_NBR     AS PERSON_NBR,
            TRANS_INDIV_AGN_ID AS RVRSD_PTNT_ACN
        FROM ELGO.ELGSMK00               -- <== your DB2 object from the SP
        WHERE MEMBERSHIP_NBR IN ({placeholders})
        WITH UR";

    using var db2Cmd = db2.CreateCommand();
    db2Cmd.CommandText = db2Sql;

    // add one DB2 parameter per id (positional)
    foreach (var id in ids)
    {
        var p = db2Cmd.CreateParameter();
        p.DB2Type = DB2Type.VarChar;
        p.Value = id;
        db2Cmd.Parameters.Add(p);
    }

    // c) fetch from DB2
    var db2Rows = new DataTable();
    using (var da = new DB2DataAdapter((DB2Command)db2Cmd))
        da.Fill(db2Rows);

    // d) land into the SQL temp table exactly like: INSERT INTO #DB2_MEMBER_INFO (...) EXEC(@SQL_Final)
    using (var bulk = new SqlBulkCopy(sql) { DestinationTableName = "#DB2_MEMBER_INFO" })
    {
        bulk.ColumnMappings.Add("MEMBER_ID",       "MEMBER_ID");
        bulk.ColumnMappings.Add("PERSON_NBR",      "PERSON_NBR");
        bulk.ColumnMappings.Add("RVRSD_PTNT_ACN",  "RVSVD_PINT_ACN"); // matches your temp table column name
        bulk.WriteToServer(db2Rows);
    }

    // e) mark those 150 as processed (SP sets PROCESSED = 1 for the same set)
    if (ids.Count > 0)
    {
        var sb = new StringBuilder();
        var prms = new List<SqlParameter>();
        for (int i = 0; i < ids.Count; i++)
        {
            if (i > 0) sb.Append(",");
            sb.Append($"@m{i}");
            prms.Add(new SqlParameter($"@m{i}", ids[i]));
        }

        var markSql = $"UPDATE #CLAIM_MEMBER_INFO SET PROCESSED = 1 WHERE MEMBER_ID IN ({sb});";
        using var markCmd = new SqlCommand(markSql, sql);
        markCmd.Parameters.AddRange(prms.ToArray());
        markCmd.ExecuteNonQuery();
    }
}
