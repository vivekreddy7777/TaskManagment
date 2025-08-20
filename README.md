
using BTA;
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Serilog;

namespace BTA_CompareTool_Service
{
    public static class Program
    {
        public static async Task Main(string[] args)
        {
            Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Information()
                .WriteTo.File(AppDomain.CurrentDomain.BaseDirectory + "\\logs\\log.txt",
                              rollingInterval: RollingInterval.Day,
                              rollOnFileSizeLimit: true)
                .CreateLogger();

            try
            {
                Log.Information("== Service bootstrap ==");

                var env = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "QA";

                IConfigurationRoot config = new ConfigurationBuilder()
                    .SetBasePath(AppDomain.CurrentDomain.BaseDirectory)
                    .AddJsonFile($"appsettings.{env}.json", optional: true, reloadOnChange: true)
                    .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                    .AddEnvironmentVariables()
                    .Build();

                // (Optional) pull values from Config Server into AppSettings first
                var configServerUrl = config["ConfigServerPath"];
                if (!string.IsNullOrWhiteSpace(configServerUrl))
                {
                    var dict = await ReadConfigServer.ReadConfigServerSettingsAsync(configServerUrl);
                    foreach (dynamic kv in dict.Entries)
                        System.Configuration.ConfigurationManager.AppSettings.Set(kv.Name, kv.Value);
                }

                var host = Host.CreateApplicationBuilder(args);

                host.Logging.ClearProviders();
                host.Logging.AddSerilog(Log.Logger, dispose: false);
                if (OperatingSystem.IsWindows() && !Environment.UserInteractive)
                    host.Logging.AddEventLog();

                var connStr = host.Configuration.GetConnectionString("SqlServerConnection");
                if (string.IsNullOrWhiteSpace(connStr))
                    throw new InvalidOperationException("SqlServerConnection missing.");

                // Register EF context(s)
                host.Services.AddDbContext<DBServerContext>(opt => opt.UseSqlServer(connStr));
                host.Services.AddScoped<DbContext>(sp => sp.GetRequiredService<DBServerContext>());

                // Your services
                host.Services.AddScoped<BTA_CompareTool_Job>();
                host.Services.AddHostedService<CompareToolHostedService>();

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

    public class CompareToolHostedService : BackgroundService
    {
        private readonly IServiceProvider _services;
        private readonly ILogger<CompareToolHostedService> _logger;

        public CompareToolHostedService(IServiceProvider services, ILogger<CompareToolHostedService> logger)
        {
            _services = services;
            _logger = logger;
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation("CompareTool Service Started");

            using var scope = _services.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<DBServerContext>();
            var job = scope.ServiceProvider.GetRequiredService<BTA_CompareTool_Job>();

            await job.RunAsync(stoppingToken);
        }
    }
}
