using System;
using System.Data.Odbc;
using Microsoft.Data.SqlClient;
using Microsoft.Extensions.Configuration;
using Serilog;

namespace BTA_CompareTool_Console
{
    internal class Program
    {
        static int Main(string[] args)
        {
            // 🔹 Initialize logger first
            Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Debug()
                .WriteTo.Console()
                .WriteTo.File("logs\\CompareTool.log",
                              rollingInterval: RollingInterval.Day,
                              rollOnFileSizeLimit: true,
                              retainedFileCountLimit: 7)
                .CreateLogger();

            try
            {
                Log.Information("=== CompareTool started ===");

                if (args.Length < 12)
                {
                    Log.Error("Invalid number of args. Expected 12, got {Count}", args.Length);
                    return 2;
                }

                // Map args
                string bulkInsertKey = args[0];
                int wmId             = int.Parse(args[1]);
                string fileName      = args[2];
                int gridType         = int.Parse(args[3]);
                int additional1      = int.Parse(args[4]);
                string overrideType  = args[5];
                string region        = args[6];
                string tableName     = args[7];
                string valTableName  = args[8];
                string valTempTable  = args[9];
                string tempTable     = args[10];
                string environment   = args[11];

                // 🔹 Load configuration
                var config = new ConfigurationBuilder()
                    .SetBasePath(AppContext.BaseDirectory)
                    .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                    .AddEnvironmentVariables()
                    .Build();

                string sqlConnStr  = config.GetConnectionString("SqlConnection");
                string odbcConnStr = config.GetConnectionString("OdbcConnection");

                using var sql = new SqlConnection(sqlConnStr);
                using var db2 = new OdbcConnection(odbcConnStr);

                sql.Open();
                db2.Open();

                Log.Information("Connections established. SQL: {Sql}, DB2: {Db2}", sqlConnStr, odbcConnStr);

                // 🔹 Run processor
                var processor = new FullSequentialProcessor(
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
                    environment
                );

                processor.RunAll(continueOnFailure: false);

                Log.Information("=== CompareTool completed successfully ===");
                return 0;
            }
            catch (Exception ex)
            {
                Log.Fatal(ex, "❌ CompareTool failed with error");
                return 1;
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }
    }
}
