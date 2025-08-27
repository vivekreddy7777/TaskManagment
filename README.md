using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;
using Serilog.Events;

// using BTA;  // <-- keep your own namespaces
// using BTA.CompareTool_Service; 
// using BTA.CompareTool_Service.Data;  // DBServerContext
// using BTA.CompareTool_Service.Services; // CompareToolHostedService
// using BTA.ExternalDBCalls_Service;     // if you need others
// using ReadConfigServer = YourNamespace.ReadConfigServer; // alias if helpful

namespace BTA.CompareTool_Service
{
    public static class Program
    {
        public static async Task Main(string[] args)
        {
            // 1) make sure relative paths are from the app folder
            Directory.SetCurrentDirectory(AppContext.BaseDirectory);

            // 2) bootstrap Serilog early (no EventLog while debugging)
            Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
                .MinimumLevel.Information()
                .Enrich.FromLogContext()
                .WriteTo.Console()
                .WriteTo.File(
                    Path.Combine(AppContext.BaseDirectory, "Logs", "service-boot.log"),
                    rollingInterval: RollingInterval.Day,
                    retainedFileCountLimit: 14)
                .CreateLogger();

            try
            {
                Log.Information("=== Service bootstrap starting ===");

                // 3) base configuration: appsettings + env
                var environment =
                    Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "QA";

                var preConfig = new ConfigurationBuilder()
                    .SetBasePath(AppContext.BaseDirectory)
                    .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                    .AddJsonFile($"appsettings.{environment}.json", optional: true, reloadOnChange: true)
                    .AddEnvironmentVariables()
                    .Build();

                // 4) pull keys from CONFIG SERVER and overlay them
                var configServerUrl = preConfig["ConfigServerPath"];
                Dictionary<string, string> serverValues = new(StringComparer.OrdinalIgnoreCase);

                if (!string.IsNullOrWhiteSpace(configServerUrl))
                {
                    Log.Information("Reading config from server: {Url}", configServerUrl);

                    // EXPECTED: returns something like IEnumerable<dynamic> with .name/.value
                    var response = await ReadConfigServer.ReadConfigServerSettingsAsync(configServerUrl);

                    foreach (var entry in response) // entry.name, entry.value
                    {
                        string key = Convert.ToString(entry.name);
                        string val = Convert.ToString(entry.value);
                        if (!string.IsNullOrWhiteSpace(key))
                            serverValues[key] = val ?? string.Empty;
                    }

                    // also push into ConfigurationManager.AppSettings for legacy callers
                    foreach (var kv in serverValues)
                        System.Configuration.ConfigurationManager.AppSettings.Set(kv.Key, kv.Value);

                    Log.Information("Loaded {Count} keys from config server.", serverValues.Count);
                }
                else
                {
                    Log.Warning("ConfigServerPath is empty; continuing with JSON/env only.");
                }

                // 5) final merged configuration (server overrides JSON/env)
                var mergedConfig = new ConfigurationBuilder()
                    .AddConfiguration(preConfig)
                    .AddInMemoryCollection(serverValues) // last wins
                    .Build();

                // sanity check: required connection string
                var coreCs =
                    mergedConfig.GetConnectionString("QCoreDBConnectionString")
                    ?? mergedConfig["QCoreDBConnectionString"];

                if (string.IsNullOrWhiteSpace(coreCs))
                    throw new InvalidOperationException("QCoreDBConnectionString missing from configuration.");

                // 6) build the Windows Service host
                var host = Host.CreateDefaultBuilder(args)
                    .UseWindowsService()
                    .UseSerilog((ctx, services, cfg) =>
                    {
                        cfg.ReadFrom.Configuration(mergedConfig)
                           .Enrich.FromLogContext()
                           .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
                           .WriteTo.File(
                               Path.Combine(AppContext.BaseDirectory, "Logs", "service.log"),
                               rollingInterval: RollingInterval.Day,
                               retainedFileCountLimit: 14);

                        // EventLog only when running as a service (VS users usually lack rights)
                        if (!Environment.UserInteractive)
                        {
                            // requires Serilog.Sinks.EventLog package
                            cfg.WriteTo.EventLog("BTA_CompareTool_Service",
                                manageEventSource: true,
                                source: "Application");
                        }
                    })
                    .ConfigureServices(services =>
                    {
                        services.AddSingleton<IConfiguration>(mergedConfig);

                        services.AddDbContext<DBServerContext>(opt =>
                            opt.UseSqlServer(coreCs),
                            contextLifetime: ServiceLifetime.Scoped);

                        services.AddHostedService<CompareToolHostedService>();
                    })
                    .Build();

                Log.Information("Service starting...");
                await host.RunAsync();
            }
            catch (Exception ex)
            {
                // last-ditch logging
                try { Log.Fatal(ex, "Service failed to start"); } catch { /* ignore */ }
                throw;
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }
    }
}
