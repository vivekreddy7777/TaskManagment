using System;
using System.Threading.Tasks;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Microsoft.EntityFrameworkCore;
using Serilog;
using Serilog.Events;
using System.Configuration; // for ConfigurationManager.AppSettings
using Microsoft.Data.SqlClient;

namespace BTA_CompareTool_Service
{
    public static class Program
    {
        public static async Task Main(string[] args)
        {
            // --- logging first ---
            Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Information()
                .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
                .WriteTo.File(AppDomain.CurrentDomain.BaseDirectory + @"\logs\log.txt",
                              rollingInterval: RollingInterval.Day,
                              rollOnFileSizeLimit: true)
                .CreateLogger();

            try
            {
                Log.Information("== Service bootstrap ==");
                
                // env (optional; default QA per your screenshot flow)
                var env = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "QA";

                // --- base config (appsettings + env) ---
                IConfigurationRoot config = new ConfigurationBuilder()
                    .SetBasePath(AppDomain.CurrentDomain.BaseDirectory)
                    .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                    .AddJsonFile($"appsettings.{env}.json", optional: true, reloadOnChange: true)
                    .AddEnvironmentVariables()
                    .Build();

                // --- pull settings from Config Server and push into AppSettings (as per screenshot) ---
                var configServerUrl = config["ConfigServerPath"];
                if (!string.IsNullOrWhiteSpace(configServerUrl))
                {
                    // your existing helper; keep the name you already use
                    var configServerSettings = await ReadConfigServer.ReadConfigServerSettingsAsync(configServerUrl);

                    foreach (dynamic obj in configServerSettings.Entries)
                    {
                        string key = obj.Name;
                        string value = obj.Value;
                        System.Configuration.ConfigurationManager.AppSettings.Set(key, value);
                        Log.Debug("ConfigServer: {Key} = {Value}", key, value);
                    }
                }
                else
                {
                    Log.Warning("ConfigServerPath not set; skipping Config Server pull.");
                }

                // --- values used below (match your screenshot keys) ---
                string sqlConnStr   = System.Configuration.ConfigurationManager.AppSettings.Get("QCoreDBConnectionString") ?? "";
                string consoleAppPath = System.Configuration.ConfigurationManager.AppSettings.Get("consoleAppPath") ?? "";

                if (string.IsNullOrWhiteSpace(sqlConnStr))
                    throw new InvalidOperationException("QCoreDBConnectionString is missing after configuration load.");
                if (string.IsNullOrWhiteSpace(consoleAppPath))
                    Log.Warning("consoleAppPath is empty; console launcher features may fail.");

                // (optional) quick sanity open so failures surface early
                using (var test = new SqlConnection(sqlConnStr))
                {
                    await test.OpenAsync();
                    Log.Information("SQL connectivity OK (QCore).");
                }

                // --- host / DI setup (Windows Service friendly) ---
                HostApplicationBuilder builder = Host.CreateApplicationBuilder(args);
                builder.Services.AddWindowsService(options => options.ServiceName = "BTA_CompareTool_Service");

                // event log + console logging if you want both
                builder.Logging.ClearProviders();
                builder.Logging.AddEventLog(settings =>
                {
                    settings.SourceName = "BTA_CompareTool_Service";
                });
                builder.Logging.AddSerilog(Log.Logger, dispose: false);

                // EF DbContext (the type name must match yours)
                builder.Services.AddDbContext<DBServerContext>(options =>
                {
                    options.UseSqlServer(sqlConnStr, sql =>
                    {
                        sql.EnableRetryOnFailure();
                    });
                }, contextLifetime: ServiceLifetime.Transient, optionsLifetime: ServiceLifetime.Transient);

                // register your job + hosted service
                builder.Services.AddScoped<BTA_CompareTool_Job>();
                builder.Services.AddHostedService<CompareToolHostedService>();

                IHost host = builder.Build();
                Log.Information("Service starting...");
                await host.RunAsync();
            }
            catch (Exception ex)
            {
                Log.Fatal(ex, "Service failed to start");
                // rethrow so VS shows the failure too
                throw;
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }
    }
}
