using BTA_CompareTool_Service;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Serilog;
using System.Configuration;

namespace BTA_CompareTool_Service
{
    public static class Program
    {
        public static async Task Main(string[] args)
        {
            // bootstrap logging
            Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Information()
                .WriteTo.Console()
                .WriteTo.File(Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "logs", "service.log"),
                    rollingInterval: RollingInterval.Day)
                .CreateLogger();

            try
            {
                Log.Information("=== Service bootstrap starting ===");

                // load initial settings (env, json, etc.)
                IConfigurationRoot config = new ConfigurationBuilder()
                    .SetBasePath(AppDomain.CurrentDomain.BaseDirectory)
                    .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                    .AddJsonFile($"appsettings.{Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "QA"}.json",
                        optional: true, reloadOnChange: true)
                    .AddEnvironmentVariables()
                    .Build();

                // pull settings from config server into AppSettings
                string? configServerUrl = config["ConfigServerPath"];
                if (!string.IsNullOrWhiteSpace(configServerUrl))
                {
                    var dict = await ReadConfigServer.ReadConfigServerSettingsAsync(configServerUrl);
                    foreach (dynamic kv in dict.Entries)
                    {
                        System.Configuration.ConfigurationManager.AppSettings.Set(kv.Name, kv.Value);
                    }
                }

                // read DB connection string from config
                string? connStr = ConfigurationManager.AppSettings["QCoreDBConnectionString"];
                if (string.IsNullOrWhiteSpace(connStr))
                    throw new InvalidOperationException("QCoreDBConnectionString missing from configuration.");

                // host + DI setup
                var host = Host.CreateApplicationBuilder(args);

                host.Logging.ClearProviders();
                host.Logging.AddSerilog();

                // register EF DbContext
                host.Services.AddDbContext<DBServerContext>(opt =>
                    opt.UseSqlServer(connStr));

                // register jobs & services
                host.Services.AddScoped<BTA_CompareTool_Job>();
                host.Services.AddHostedService<CompareToolHostedService>();

                Log.Information("Service starting...");
                await host.Build().RunAsync();
            }
            catch (Exception ex)
            {
                Log.Fatal(ex, "Service failed to start");
                throw;
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }
    }
}
