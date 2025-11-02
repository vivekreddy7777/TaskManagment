using System;
using System.Collections.Generic;
using System.Data;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using IBM.Data.DB2.Core;                 // or IBM.Data.DB2 (match your package)
using Microsoft.Data.SqlClient;          // for SqlBulkCopy + SqlParameter
using Microsoft.EntityFrameworkCore;     // EF Core
using Microsoft.Extensions.Logging;

namespace BTA_CompareTool_Console.Processors
{
    /// <summary>
    /// One-stop processor for CCRSMG00:
    /// - Finds DBIDs and entity batches (EF LINQ)
    /// - Pulls batch rows from DB2
    /// - Bulk inserts to SQL Server staging
    /// - Marks rows complete / updates log (EF)
    /// - Calls CompareTool SP via EF (OUTPUT params)
    /// </summary>
    public sealed class CcrSmg00Processor
    {
        // ——— injected dependencies ———
        private readonly DbContext      _db;     // your CompareToolDbContext
        private readonly SqlConnection  _sql;    // for SqlBulkCopy only
        private readonly DB2Connection  _db2;    // DB2 batch pull
        private readonly ILogger<CcrSmg00Processor> _log;

        // ——— run params ———
        private readonly string _bulkInsertKey;
        private readonly string _wmId;
        private readonly string _lanId;
        private readonly string _fileName;
        private readonly int    _gridType;
        private readonly string? _additional1;
        private readonly string? _overrideType;
        private readonly string? _region;

        public CcrSmg00Processor(
            DbContext db,
            SqlConnection sql,
            DB2Connection db2,
            ILogger<CcrSmg00Processor> log,
            string bulkInsertKey,
            string wmId,
            string lanId,
            string fileName,
            int gridType,
            string? additional1 = null,
            string? overrideType = null,
            string? region = null)
        {
            _db = db;
            _sql = sql;
            _db2 = db2;
            _log = log;

            _bulkInsertKey = bulkInsertKey;
            _wmId          = wmId;
            _lanId         = lanId;
            _fileName      = fileName;
            _gridType      = gridType;
            _additional1   = additional1;
            _overrideType  = overrideType;
            _region        = region;
        }

        // ————————————————————————————————————————————————————————————————————
        // ENTRY POINT
        // ————————————————————————————————————————————————————————————————————
        public async Task RunAsync(CancellationToken ct = default)
        {
            _log.LogInformation("[CCRSMG00] START key={Key} wmId={WM}", _bulkInsertKey, _wmId);
            var started = DateTime.UtcNow;

            // 1) find participating DBIDs (EF)
            var dbIds = await GetDbIdsAsync(ct);
            _log.LogInformation("[CCRSMG00] DBIDs: {Count}", dbIds.Count);

            // 2) for each DB, loop batches until done
            foreach (var dbid in dbIds)
            {
                _log.LogInformation("[CCRSMG00] DBID={DBID}: begin", dbid);
                while (true)
                {
                    var ids = await GetEntityBatchAsync(dbid, top: 100, ct);
                    if (ids.Count == 0)
                    {
                        _log.LogInformation("[CCRSMG00] DBID={DBID}: no more entities", dbid);
                        break;
                    }

                    _log.LogInformation("[CCRSMG00] DBID={DBID}: batch size={Size}", dbid, ids.Count);

                    // 3) fetch DB2 rows for batch
                    var dt = await FetchDb2BatchAsync(ids, ct);

                    // inject routing columns expected by your destination (WM_ID, BulkInsertKey)
                    if (!dt.Columns.Contains("WM_ID")) dt.Columns.Add("WM_ID", typeof(string));
                    if (!dt.Columns.Contains("BulkInsertKey")) dt.Columns.Add("BulkInsertKey", typeof(string));
                    foreach (DataRow r in dt.Rows)
                    {
                        r["WM_ID"] = _wmId;
                        r["BulkInsertKey"] = _bulkInsertKey;
                    }

                    // 4) bulk insert to SQL Server
                    await BulkInsertAsync(dt, ct);

                    // 5) mark batch complete + check remaining
                    await MarkBatchCompleteAsync(ids, dbid, ct);

                    var hasMore = await HasRemainingEntitiesAsync(dbid, ct);
                    if (!hasMore) break;
                }

                _log.LogInformation("[CCRSMG00] DBID={DBID}: done", dbid);
            }

            // 6) summary/log row (EF)
            var completed = await CountCompletedAsync(ct);
            await UpdateAddtnlLogAsync(started, DateTime.UtcNow, completed, ct);

            // 7) call CompareTool SP (EF only) to continue pipeline
            var spResult = await ExecCompareToolClaimBulkInsertAsync(ct);
            _log.LogInformation("[CCRSMG00] SP result: {Result}", spResult.TempTableName ?? "(no temp)");

            _log.LogInformation("[CCRSMG00] END key={Key} processed={Cnt}", _bulkInsertKey, completed);
        }

        // ————————————————————————————————————————————————————————————————————
        // EF QUERIES (SQL SERVER)
        // ————————————————————————————————————————————————————————————————————

        // Distinct DBID list from dbo.tblCompareClaimsDetails_Addtnl_1_IDs
        private async Task<List<int>> GetDbIdsAsync(CancellationToken ct)
        {
            return await _db.Set<CompareClaimsDetailsAddtnl1Ids>()
                .Where(x => x.BulkInsertKey == _bulkInsertKey && x.WM_ID == _wmId)
                .Select(x => x.CCRDB)
                .Distinct()
                .ToListAsync(ct);
        }

        // Batch of entities for a DBID, still not completed
        private async Task<List<long>> GetEntityBatchAsync(int dbId, int top, CancellationToken ct)
        {
            return await _db.Set<CompareClaimsDetailsAddtnl1Ids>()
                .Where(x => x.BulkInsertKey == _bulkInsertKey
                         && x.WM_ID == _wmId
                         && x.CCRDB == dbId
                         && !x.CCRSMG_Comp
                         && x.RVSD_PNT_AGN != "0")
                .OrderBy(x => x.RVSD_PNT_AGN)
                .Select(x => x.RVSD_PNT_AGN)      // NOTE: change type to long if your entity is numeric; otherwise map appropriately
                .Take(top)
                .Select(s => Convert.ToInt64(s))  // keep compatible with DB2 IN list
                .ToListAsync(ct);
        }

        private async Task<bool> HasRemainingEntitiesAsync(int dbId, CancellationToken ct)
        {
            return await _db.Set<CompareClaimsDetailsAddtnl1Ids>()
                .AnyAsync(x => x.BulkInsertKey == _bulkInsertKey
                            && x.WM_ID == _wmId
                            && x.CCRDB == dbId
                            && !x.CCRSMG_Comp
                            && x.RVSD_PNT_AGN != "0", ct);
        }

        private async Task<int> CountCompletedAsync(CancellationToken ct)
        {
            return await _db.Set<CompareClaimsDetailsAddtnl1Ids>()
                .CountAsync(x => x.BulkInsertKey == _bulkInsertKey
                              && x.WM_ID == _wmId
                              && x.CCRSMG_Comp, ct);
        }

        private async Task UpdateAddtnlLogAsync(DateTime start, DateTime end, int claimCount, CancellationToken ct)
        {
            var rows = await _db.Set<CompareClaimsDetailsAddtnlLog>()
                .Where(x => x.BulkInsertKey == _bulkInsertKey && x.WM_ID == _wmId)
                .ToListAsync(ct);

            foreach (var r in rows)
            {
                r.CCRSMG_StartTime = start;
                r.CCRSMG_EndTime   = end;
                r.CCRSMG_ClaimCount = claimCount;
            }
            await _db.SaveChangesAsync(ct);
        }

        // mark processed flag for current batch
        private async Task MarkBatchCompleteAsync(List<long> ids, int dbId, CancellationToken ct)
        {
            // chunk big IN lists
            const int chunk = 900;
            foreach (var slice in ids.Chunk(chunk))
            {
                var inList = string.Join(",", slice);
                var sql =
                    $"UPDATE dbo.tblCompareClaimsDetails_Addtnl_1_IDs " +
                    $"SET CCRSMG_Comp = 1 " +
                    $"WHERE BulkInsertKey = @B AND WM_ID = @W AND CCRDB = @D AND RVSD_PNT_AGN IN ({inList})";

                await _db.Database.ExecuteSqlRawAsync(sql, new SqlParameter("@B", _bulkInsertKey),
                                                           new SqlParameter("@W", _wmId),
                                                           new SqlParameter("@D", dbId));
            }
        }

        // ————————————————————————————————————————————————————————————————————
        // DB2 FETCH + BULKCOPY (internal to class)
        // ————————————————————————————————————————————————————————————————————

        private async Task<DataTable> FetchDb2BatchAsync(IReadOnlyCollection<long> ids, CancellationToken ct)
        {
            var dt = new DataTable();
            if (ids.Count == 0) return dt;

            if (_db2.State != ConnectionState.Open)
                await _db2.OpenAsync(ct);

            var inList = string.Join(",", ids);
            // TODO: replace schema/table & columns with your real ones
            var sql = $@"
SELECT
    SMG_RVSD_PNT_AGN,
    SMG_RQST_TYP_CDE,
    SMG_RQST_VRSN_SEQ,
    SMG_CONFLICT_CDE,
    SMG_CONFLICT_DTL,
    SMG_CONV_FILL_QTY,
    SMG_PROD_SVC_ID,
    SMG_INT_TYPE_CDE,
    SMG_INTRVNTN_CDE,
    SMG_MESSAGE_SEQ,
    SMG_OUTCOME_CDE,
    SMG_SEVERITY_CDE
FROM  DB2LIB.CCRSMG00 WITH UR
WHERE SMG_RVSD_PNT_AGN IN ({inList})";

            using var cmd = _db2.CreateCommand();
            cmd.CommandText = sql;

            using var da = new DB2DataAdapter((DB2Command)cmd);
            da.Fill(dt);

            _log.LogInformation("[CCRSMG00] DB2 pull rows={Rows}", dt.Rows.Count);
            return dt;
        }

        private async Task BulkInsertAsync(DataTable dt, CancellationToken ct)
        {
            if (dt.Rows.Count == 0) return;

            if (_sql.State != ConnectionState.Open)
                await _sql.OpenAsync(ct);

            using var bulk = new SqlBulkCopy(_sql, SqlBulkCopyOptions.FireTriggers | SqlBulkCopyOptions.CheckConstraints, null)
            {
                DestinationTableName = "[dbo].[tblCompareClaimsDetails_Addtnl_CCRSMG00]",
                BatchSize            = 5000,
                BulkCopyTimeout      = 0
            };

            // map DB2 -> SQL columns (keep in sync with your table)
            Map("WM_ID", "WM_ID");
            Map("BulkInsertKey", "BulkInsertKey");
            Map("SMG_RVSD_PNT_AGN", "SMG_RVSD_PNT_AGN");
            Map("SMG_RQST_TYP_CDE", "SMG_RQST_TYP_CDE");
            Map("SMG_RQST_VRSN_SEQ", "SMG_RQST_VRSN_SEQ");
            Map("SMG_CONFLICT_CDE", "SMG_CONFLICT_CDE");
            Map("SMG_CONFLICT_DTL", "SMG_CONFLICT_DTL");
            Map("SMG_CONV_FILL_QTY", "SMG_CONV_FILL_QTY");
            Map("SMG_PROD_SVC_ID", "SMG_PROD_SVC_ID");
            Map("SMG_INT_TYPE_CDE", "SMG_INT_TYPE_CDE");
            Map("SMG_INTRVNTN_CDE", "SMG_INTRVNTN_CDE");
            Map("SMG_MESSAGE_SEQ", "SMG_MESSAGE_SEQ");
            Map("SMG_OUTCOME_CDE", "SMG_OUTCOME_CDE");
            Map("SMG_SEVERITY_CDE", "SMG_SEVERITY_CDE");
            // …add remaining mappings from your screenshots…

            await bulk.WriteToServerAsync(dt, ct);
            _log.LogInformation("[CCRSMG00] Bulk copied {Rows} rows.", dt.Rows.Count);

            void Map(string src, string dest) => bulk.ColumnMappings.Add(src, dest);
        }

        // ————————————————————————————————————————————————————————————————————
        // EF-ONLY stored procedure call with OUTPUT params
        // ————————————————————————————————————————————————————————————————————
        private async Task<(string? ValidationErrors, string? ValTableName, string? WaETempTableName, string? TempTableName, string? TempLog)>
            ExecCompareToolClaimBulkInsertAsync(CancellationToken ct)
        {
            var pValidationErrors = Out("@ValidationErrors", SqlDbType.VarChar, 100);
            var pValTableName     = Out("@ValTableName",     SqlDbType.VarChar, 100);
            var pWaETempTableName = Out("@WaETempTableName", SqlDbType.VarChar, 100);
            var pTempTableName    = Out("@TempTableName",    SqlDbType.VarChar, 100);
            var pTempLog          = Out("@TempLog",          SqlDbType.VarChar, 6000);

            string sql =
                "EXEC [Quality_Control].[dbo].[procCompareTool_ClaimBulkInsert1] " +
                "@Password, @BulkInsertKey, @LANID, @W_ID, @FileName, @GridType, @Additional1, @OverrideType, @Region, " +
                "@ValidationErrors OUTPUT, @ValTableName OUTPUT, @WaETempTableName OUTPUT, @TempTableName OUTPUT, @TempLog OUTPUT";

            await _db.Database.ExecuteSqlRawAsync(sql, new[]
            {
                In("@Password",      SqlDbType.VarChar, 50,  "MHZ-20U"),
                In("@BulkInsertKey", SqlDbType.VarChar, 100, _bulkInsertKey),
                In("@LANID",         SqlDbType.VarChar, 100, _lanId),
                In("@W_ID",          SqlDbType.VarChar, 50,  _wmId),
                In("@FileName",      SqlDbType.VarChar, 255, _fileName),
                In("@GridType",      SqlDbType.Int,         _gridType),
                In("@Additional1",   SqlDbType.VarChar, 200, _additional1),
                In("@OverrideType",  SqlDbType.VarChar, 40,  _overrideType),
                In("@Region",        SqlDbType.VarChar, 40,  _region),
                pValidationErrors, pValTableName, pWaETempTableName, pTempTableName, pTempLog
            }, ct);

            return (
                pValidationErrors.Value as string,
                pValTableName.Value     as string,
                pWaETempTableName.Value as string,
                pTempTableName.Value    as string,
                pTempLog.Value          as string
            );
        }

        // ————————————————————————————————————————————————————————————————————
        // helpers
        // ————————————————————————————————————————————————————————————————————
        private static SqlParameter In(string name, SqlDbType type, int size, object? value) =>
            new(name, type, size){ Direction = ParameterDirection.Input, Value = value ?? DBNull.Value };
        private static SqlParameter In(string name, SqlDbType type, object? value) =>
            new(name, type){ Direction = ParameterDirection.Input, Value = value ?? DBNull.Value };
        private static SqlParameter Out(string name, SqlDbType type, int size = 0) =>
            size > 0 ? new(name, type, size){ Direction = ParameterDirection.Output }
                     : new(name, type){ Direction = ParameterDirection.Output };
    }

    // ————————————————————————————————————————————————————————————————————
    // Minimal entity types (match your EF model)
    // ————————————————————————————————————————————————————————————————————
    public sealed class CompareClaimsDetailsAddtnl1Ids
    {
        public string BulkInsertKey { get; set; } = default!;
        public string WM_ID { get; set; } = default!;
        public int    CCRDB { get; set; }
        public string RVSD_PNT_AGN { get; set; } = default!;
        public bool   CCRSMG_Comp { get; set; }
    }

    public sealed class CompareClaimsDetailsAddtnlLog
    {
        public string BulkInsertKey { get; set; } = default!;
        public string WM_ID { get; set; } = default!;
        public DateTime? CCRSMG_StartTime { get; set; }
        public DateTime? CCRSMG_EndTime   { get; set; }
        public int?      CCRSMG_ClaimCount { get; set; }
    }
}
