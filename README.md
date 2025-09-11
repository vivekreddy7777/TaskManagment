private DataTable QueryDb2ForBatch(List<string> groups, string envLetter)
{
    var dt = new DataTable();
    if (groups == null || groups.Count == 0)
        return dt;

    // Build the IN clause with properly quoted values
    string inClause = string.Join(", ", groups.Select(g => $"'{g}'"));

    // DB2 SQL (with DB0 prefix in front of environment)
    string db2Sql = $@"
        SELECT
            LWR.OPERATIONAL_ID AS CLAIM_GROUP,   -- LWR
            CAR.OPERATIONAL_ID AS CBM_BPL        -- CAR
        FROM DB0{envLetter}CBMO.CBMCLT00 CAR
        INNER JOIN DB0{envLetter}CBMO.CBMCPO00 CMP 
            ON CAR.CLIENT_AGN = CMP.ACSTR_CLNT_AGN_ID
        INNER JOIN DB0{envLetter}CBMO.CBMCLT00 LWR 
            ON LWR.CLIENT_AGN = CMP.CLIENT_AGN_ID
        WHERE CAR.ROW_DEL_TMS = '2999-12-31-00.00.00.000000'
          AND LWR.ROW_DEL_TMS = '2999-12-31-00.00.00.000000'
          AND LWR.OPERATIONAL_ID IN ({inClause})
          AND LWR.CLIENT_TYPE_SEQ = 60
          AND (CAR.CLIENT_TYPE_CDE = 'WB' OR CAR.CLIENT_TYPE_CDE = 'BT')
        WITH UR
    ";

    using (var cmd = new DB2Command(db2Sql, _db2))
    using (var adapter = new DB2DataAdapter(cmd))
    {
        adapter.Fill(dt);
    }

    return dt;
}
