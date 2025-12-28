using System;
using System.Diagnostics;
using System.Net;
using System.Threading.Tasks;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;

using CompareTool.Services.Security;
using CompareTool.Services;

namespace CompareTool.ConsoleApp
{
    public sealed class RuntimeContext
    {
        public string Db2User { get; init; } = string.Empty;
        public string Db2Password { get; init; } = string.Empty;

        public string TeradataUser { get; init; } = string.Empty;
        public string TeradataPassword { get; init; } = string.Empty;

        public string SqlUser { get; init; } = string.Empty;
        public string SqlPassword { get; init; } = string.Empty;
    }

    public static class Program
    {
        public static async Task<int> Main(string[] args)
        {
            ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;

            Log.Logger = new LoggerConfiguration()
                .WriteTo.Console()
                .CreateLogger();

            try
            {
                var environment =
                    Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "QA";

                IConfiguration configuration = new ConfigurationBuilder()
                    .SetBasePath(AppContext.BaseDirectory)
                    .AddJsonFile("appsettings.json", optional: false)
                    .AddJsonFile($"appsettings.{environment}.json", optional: true)
                    .AddEnvironmentVariables()
                    .Build();

                Log.Information("Starting CompareTool Console | Environment: {Env}", environment);

                var cyberArkProvider =
                    new CyberArkCredentialProvider(configuration);

                Log.Information("Retrieving credentials from CyberArk...");

                var sw = Stopwatch.StartNew();

                var db2Creds = await cyberArkProvider.GetCredentialsAsync(CredentialKind.Db2);
                Log.Information("CyberArk credentials retrieved for DB2");

                var teradataCreds = await cyberArkProvider.GetCredentialsAsync(CredentialKind.Teradata);
                Log.Information("CyberArk credentials retrieved for Teradata");

                var sqlCreds = await cyberArkProvider.GetCredentialsAsync(CredentialKind.Sql);
                Log.Information("CyberArk credentials retrieved for SQL Server");

                sw.Stop();
                Log.Information(
                    "CyberArk credential retrieval completed in {ElapsedMs} ms",
                    sw.ElapsedMilliseconds);

                var runtimeContext = new RuntimeContext
                {
                    Db2User = db2Creds.User,
                    Db2Password = db2Creds.Password,
                    TeradataUser = teradataCreds.User,
                    TeradataPassword = teradataCreds.Password,
                    SqlUser = sqlCreds.User,
                    SqlPassword = sqlCreds.Password
                };

                Log.Information("Runtime credential context initialized successfully");

                using IHost host = Host.CreateDefaultBuilder(args)
                    .UseSerilog()
                    .ConfigureServices((_, services) =>
                    {
                        services.AddSingleton(configuration);
                        services.AddSingleton(runtimeContext);

                        services.AddScoped<CallCompareToolService>();
                        services.AddScoped<CompareToolJobProcessorService>();
                        services.AddScoped<CompareToolExportService>();

                        services.AddScoped<CBMBPLProcessorService>();
                        services.AddScoped<CCRCL100ProcessorService>();
                        services.AddScoped<CCRCL200ProcessorService>();
                        services.AddScoped<CCRCPR00ProcessorService>();
                        services.AddScoped<CCRDC000ProcessorService>();
                        services.AddScoped<CCRSMG00ProcessorService>();
                        services.AddScoped<CDRRUL00ProcessorService>();
                    })
                    .Build();

                using var scope = host.Services.CreateScope();

                var jobProcessor =
                    scope.ServiceProvider.GetRequiredService<CompareToolJobProcessorService>();

                Log.Information("Starting CompareTool job execution");
                await jobProcessor.ExecuteAsync();
                Log.Information("CompareTool job execution completed successfully");

                return 0;
            }
            catch (Exception ex)
            {
                Log.Fatal(ex, "CompareTool Console failed");
                return 1;
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }
    }
}
