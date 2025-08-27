using System.Diagnostics;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Serilog;
using Serilog.Events;
using Serilog.Sinks.EventLog;

namespace BTA_CompareTool_Service;

public static class Program
{
    public static async Task Main(string[] args)
    {
        // 1) Always pin the working directory to the app folder
        Directory.SetCurrentDirectory(AppContext.BaseDirectory);

        // 2) Bootstrap Serilog early + write to EventLog and file
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
            .MinimumLevel.Information()
            .Enrich.FromLogContext()
            .WriteTo.EventLog(
                source: "BTA_CompareTool_Service",
                logName: "Application",
                manageEventSource: true)
            .WriteTo.File(
                Path.Combine(AppContext.BaseDirectory, "Logs", "service-boot.log"),
                rollingInterval: RollingInterval.Day,
                retainedFileCountLimit: 14)
            .CreateLogger();

        try
        {
            Log.Information("=== Service bootstrap starting ===");

            // 3) Build configuration (appsettings + environment + your config server)
            var environment = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "QA";

            var preConfig = new ConfigurationBuilder()
                .SetBasePath(AppContext.BaseDirectory)
                .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                .AddJsonFile($"appsettings.{environment}.json", optional: true, reloadOnChange: true)
                .AddEnvironmentVariables()
                .Build();

            // Pull keys from your config server into in-memory AppSettings
            var configServerUrl = preConfig["ConfigServerPath"];
            if (string.IsNullOrWhiteSpace(configServerUrl))
                throw new InvalidOperationException("ConfigServerPath missing from configuration.");

            var kv = await ReadConfigServer.ReadConfigServerSettingsAsync(configServerUrl);
            foreach (dynamic e in kv.entries) // your helper returns { entries: [{name,value},...] }
                System.Configuration.ConfigurationManager.AppSettings.Set((string)e.name, (string)e.value);

            // Sanity check: required string
            var coreCs = System.Configuration.ConfigurationManager.AppSettings["QCoreDBConnectionString"];
            if (string.IsNullOrWhiteSpace(coreCs))
                throw new InvalidOperationException("QCoreDBConnectionString missing from configuration.");

            // 4) Proper Windows Service host  (this is the biggie)
            var builder = Host.CreateDefaultBuilder(args)
                .UseWindowsService()                 // REQUIRED on server
                .UseSerilog((ctx, services, cfg) =>  // Full Serilog after Host config exists
                {
                    cfg.ReadFrom.Configuration(preConfig)
                       .Enrich.FromLogContext()
                       .WriteTo.EventLog(
                           source: "BTA_CompareTool_Service",
                           logName: "Application",
                           manageEventSource: true)
                       .WriteTo.File(
                           Path.Combine(AppContext.BaseDirectory, "Logs", "service.log"),
                           rollingInterval: RollingInterval.Day,
                           retainedFileCountLimit: 14);
                })
                .ConfigureServices((ctx, services) =>
                {
                    // Your DI
                    services.AddDbContext<DBServerContext>(opt => opt.UseSqlServer(coreCs),
                                                           contextLifetime: ServiceLifetime.Scoped);

                    services.AddHostedService<CompareToolHostedService>();
                    // register any other singletons/options here
                });

            using var host = builder.Build();

            Log.Information("Service starting...");
            await host.RunAsync();
        }
        catch (Exception ex)
        {
            // If we failed before Serilog pipeline is up, at least log boot failure
            try { Log.Fatal(ex, "Service failed to start"); }
            finally { Log.CloseAndFlush(); }
            // Ensure the SCM sees this as a crash
            throw;
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }
}
