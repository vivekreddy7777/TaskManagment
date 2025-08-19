using System;
using System.Collections.Generic;
using System.Data;
using System.Data.Odbc;
using System.Linq;
using Microsoft.Data.SqlClient;

namespace BTA_CompareTool_Console.Services
{
    public sealed class CBMBPLProcessor
    {
        private readonly SqlConnection _sql;     // local SQL Server
        private readonly OdbcConnection _db2;    // direct DB2
        private readonly string _tempTableName;  // your upstream temp from SP1
        private readonly string _bulkInsertKey;  // for auditing if needed
        private readonly int _wmId;              // MID
        private const int BatchSize = 50;
        private readonly bool _debug = false;

        public CBMBPLProcessor(SqlConnection sql, OdbcConnection db2,
                               string tempTableName, string bulkInsertKey, int wmId)
        {
            _sql           = sql  ?? throw new ArgumentNullException(nameof(sql));
            _db2           = db2  ?? throw new ArgumentNullException(nameof(db2));
            _tempTableName = string.IsNullOrWhiteSpace(tempTableName) ? throw new ArgumentNullException(nameof(tempTableName)) : tempTableName;
            _bulkInsertKey = bulkInsertKey ?? throw new ArgumentNullException(nameof(bulkInsertKey));
            _wmId          = wmId;
        }

        public void Process()
        {
            if (_debug) Console.WriteLine($"[CBM_BPL] START WM_ID={_wmId} Key={_bulkInsertKey} Temp={_tempTableName}");

            CreateWorkingTables();
            SeedEntityTblFromTemp();

            while (true)
            {
                var groups = TakeNextGroupBatch(BatchSize);
                if (groups.Count == 0)
                {
                    if (_debug) Console.WriteLine("[CBM_BPL] No more groups to process.");
                    break;
                }

                if (_debug) Console.WriteLine($"[CBM_BPL] Processing {groups.Count} group(s).");

                var batchRows = QueryDb2ForBatch(groups);     // CLAIM_GROUP, CBM_BPL
                StageBatchIntoSql(batchRows);                 // -> #BatchCBMBPL
                UpdateTempTableFromBatch();                   // A.CBM_BPL = B.CBM_BPL
                MarkGroupsSearched(groups);
            }

            if (_debug) Console.WriteLine("[CBM_BPL] END");
        }

        // ---------- SQL Server side ----------

        private void CreateWorkingTables()
        {
            var sql = @"
IF OBJECT_ID('tempdb..#EntityTbl') IS NOT NULL DROP TABLE #EntityTbl;
CREATE TABLE #EntityTbl (
    CLAIM_GROUP VARCHAR(32) NOT NULL,
    SEARCHED BIT NOT NULL DEFAULT(0)
);

IF OBJECT_ID('tempdb..#BatchCBMBPL') IS NOT NULL DROP TABLE #BatchCBMBPL;
CREATE TABLE #BatchCBMBPL (
    CLAIM_GROUP VARCHAR(32) NOT NULL,
    CBM_BPL     VARCHAR(32) NULL
);";
            using var cmd = new SqlCommand(sql, _sql) { CommandTimeout = 0 };
            cmd.ExecuteNonQuery();
        }

        private void SeedEntityTblFromTemp()
        {
            // Your SP does: SELECT DISTINCT LTRIM(RTRIM([GROUP])) FROM @TempTableName
            var sql = $@"
INSERT INTO #EntityTbl(CLAIM_GROUP)
SELECT DISTINCT LTRIM(RTRIM([GROUP]))
FROM {_tempTableName} WITH (NOLOCK);";
            using var cmd = new SqlCommand(sql, _sql) { CommandTimeout = 0 };
            cmd.ExecuteNonQuery();
        }

        private List<string> TakeNextGroupBatch(int take)
        {
            var list = new List<string>();
            var sql = @"
WITH cte AS (
    SELECT TOP (@take) CLAIM_GROUP
    FROM #EntityTbl WITH (READPAST, UPDLOCK, ROWLOCK)
    WHERE SEARCHED = 0
    ORDER BY CLAIM_GROUP
)
UPDATE cte SET SEARCHED = 2 -- 2 = reserved/in-progress
OUTPUT inserted.CLAIM_GROUP;";
            using var cmd = new SqlCommand(sql, _sql);
            cmd.Parameters.AddWithValue("@take", take);
            using var rd = cmd.ExecuteReader();
            while (rd.Read()) list.Add(rd.GetString(0));
            return list;
        }

        private void StageBatchIntoSql(DataTable rows)
        {
            // Clear staging
            using (var clr = new SqlCommand("TRUNCATE TABLE #BatchCBMBPL;", _sql) { CommandTimeout = 0 })
                clr.ExecuteNonQuery();

            if (rows.Rows.Count == 0) return;

            using var bulk = new SqlBulkCopy(_sql) {
                DestinationTableName = "tempdb..#BatchCBMBPL", BulkCopyTimeout = 0
            };
            bulk.ColumnMappings.Add("CLAIM_GROUP", "CLAIM_GROUP");
            bulk.ColumnMappings.Add("CBM_BPL",     "CBM_BPL");
            bulk.WriteToServer(rows);
        }

        private void UpdateTempTableFromBatch()
        {
            // Your SP (final step): UPDATE A SET A.CBM_BPL = B.CBM_BPL FROM @TempTableName A JOIN #EntityTbl/Batch B
            var sql = $@"
UPDATE A
   SET A.CBM_BPL = B.CBM_BPL
FROM {_tempTableName} AS A
JOIN #BatchCBMBPL      AS B ON A.[GROUP] = B.CLAIM_GROUP;";
            using var cmd = new SqlCommand(sql, _sql) { CommandTimeout = 0 };
            cmd.ExecuteNonQuery();
        }

        private void MarkGroupsSearched(IEnumerable<string> groups)
        {
            const string upd = "UPDATE #EntityTbl SET SEARCHED = 1 WHERE CLAIM_GROUP = @g;"; // 1 = done
            foreach (var g in groups)
            {
                using var cmd = new SqlCommand(upd, _sql);
                cmd.Parameters.AddWithValue("@g", g);
                cmd.ExecuteNonQuery();
            }
        }

        // ---------- DB2 side (OPENQUERY equivalent) ----------

        private DataTable QueryDb2ForBatch(List<string> groups)
        {
            var dt = new DataTable();
            if (groups == null || groups.Count == 0) return dt;

            // Build positional placeholders ?, ?, ...
            var inPlaceholders = string.Join(",", Enumerable.Repeat("?", groups.Count));

            // === DB2 SELECT ===
            // This is the OPENQUERY body from your SP (screens 3–5), converted to native DB2.
            // Keep the same columns/aliases your SP returned (we only need CLAIM_GROUP, CBM_BPL for the update).
            var db2Sql = $@"
SELECT
    LMR.CLAIM_GRP          AS CLAIM_GROUP,
    /* replicate your CASE from the SP for CBM_BPL here */
    CASE 
        WHEN CAR.CBM_BPL IS NOT NULL THEN CAR.CBM_BPL
        /* WHEN BEST_BPL_8BYTE IS NOT NULL THEN BEST_BPL_8BYTE  -- from your later branch */
        ELSE NULL
    END                    AS CBM_BPL
FROM   CMBP.CMBCINCOME CAR           -- <-- replace with exact schema.table used in SP
JOIN   GENV.LMR_CLIENT_AGN LMR       -- <-- replace names to match your DB2
       ON LMR.CLIENT_AGN_ID = CAR.CLIENT_AGN_ID
WHERE  LMR.CLAIM_GRP IN ({inPlaceholders})
  AND  CAR.ROW_DEL_TMS = '2999-12-31-00.00.00.000000'
  AND  LMR.ROW_DEL_TMS = '2999-12-31-00.00.00.000000'
  /* Add other filters from the SP (CLIENT_TYPE filters, GROUP LEVEL, etc.) */
WITH UR";

            using var cmd = new OdbcCommand(db2Sql, _db2) { CommandTimeout = 0 };
            foreach (var g in groups)
                cmd.Parameters.AddWithValue("?", g); // ODBC: positional

            using var adapter = new OdbcDataAdapter(cmd);
            adapter.Fill(dt);

            // Normalize columns that will be bulk-copied
            if (!dt.Columns.Contains("CLAIM_GROUP")) dt.Columns.Add("CLAIM_GROUP", typeof(string));
            if (!dt.Columns.Contains("CBM_BPL"))     dt.Columns.Add("CBM_BPL",     typeof(string));

            return dt;
        }
    }
}
