using System;
using System.Data.Odbc;
using Microsoft.Data.SqlClient;
using Microsoft.Extensions.Configuration;
using BTA_CompareTool_Console.Services;

namespace BTA_CompareTool_Console
{
    public static class Program
    {
        // Expected args (exact order the service must send):
        //  0  bulkInsertKey
        //  1  wmId
        //  2  fileName
        //  3  gridType
        //  4  additional1
        //  5  overrideType
        //  6  region
        //  7  tableName
        //  8  valTableName
        //  9  valTempTableName
        // 10  tempTableName
        // 11  environment
        public static int Main(string[] args)
        {
            if (args.Length < 12)
            {
                Console.Error.WriteLine(
                    "Usage: BTA_CompareTool_Console " +
                    "<bulkInsertKey> <wmId> <fileName> <gridType> <additional1> " +
                    "<overrideType> <region> <tableName> <valTableName> <valTempTableName> <tempTableName> <environment>");
                return 2;
            }

            // Parse args
            string bulkInsertKey   = args[0];
            int    wmId            = int.Parse(args[1]);
            string fileName        = args[2];
            int    gridType        = int.Parse(args[3]);
            int    additional1     = int.Parse(args[4]);
            string overrideType    = args[5];
            string region          = args[6];
            string tableName       = args[7];
            string valTableName    = args[8];
            string valTempTable    = args[9];
            string tempTable       = args[10];
            string environment     = args[11];

            // Load configuration (for connection strings)
            IConfiguration config = new ConfigurationBuilder()
                .SetBasePath(AppContext.BaseDirectory)
                .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                .Build();

            string sqlConnStr = config.GetConnectionString("SqlConnection")
                                  ?? throw new InvalidOperationException("Missing ConnectionStrings:SqlConnection.");
            string odbcConnStr = config.GetConnectionString("OdbcConnection")
                                   ?? throw new InvalidOperationException("Missing ConnectionStrings:OdbcConnection.");

            // Open connections and run
            using var sql = new SqlConnection(sqlConnStr);
            using var db2 = new OdbcConnection(odbcConnStr);
            try
            {
                sql.Open();
                db2.Open();

                // NOTE: keep argument order in sync with your FullSequentialProcessor constructor
                var runner = new FullSequentialProcessor(
                    sql,
                    db2,
                    bulkInsertKey,
                    wmId,
                    fileName,
                    gridType,
                    additional1,
                    overrideType,
                    region,
                    tableName,
                    valTableName,
                    valTempTable,
                    tempTable,
                    environment,
                    batchSize: 50 // change if you want a different batch size
                );

                runner.RunAll(continueOnFailure: false);
                Console.WriteLine("Compare console completed successfully.");
                return 0;
            }
            catch (Exception ex)
            {
                Console.Error.WriteLine($"[FATAL] {ex}");
                return 1;
            }
        }
    }
}
