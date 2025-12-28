using System;
using System.Net;
using System.Threading.Tasks;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;

using CompareTool.Services.Security;   // <-- your existing CyberArk services
using CompareTool.Services;            // <-- your processor services

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
            // Enforce TLS 1.2 (important for CyberArk in .NET Framework / mixed stacks)
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

                Log.Information("Starting CompareTool Console in {Env}", environment);

                // -------------------------------
                // CyberArk – pull credentials
                // -------------------------------

                var cyberArkProvider =
                    new CyberArkCredentialProvider(configuration);

                var db2Creds =
                    await cyberArkProvider.GetCredentialsAsync(CredentialKind.Db2);

                var teradataCreds =
                    await cyberArkProvider.GetCredentialsAsync(CredentialKind.Teradata);

                var sqlCreds =
                    await cyberArkProvider.GetCredentialsAsync(CredentialKind.Sql);

                // Store locally in Program.cs (in-memory only)
                var runtimeContext = new RuntimeContext
                {
                    Db2User = db2Creds.User,
                    Db2Password = db2Creds.Password,

                    TeradataUser = teradataCreds.User,
                    TeradataPassword = teradataCreds.Password,

                    SqlUser = sqlCreds.User,
                    SqlPassword = sqlCreds.Password
                };

                // -------------------------------
                // Host + DI
                // -------------------------------

                using IHost host = Host.CreateDefaultBuilder(args)
                    .UseSerilog()
                    .ConfigureServices((_, services) =>
                    {
                        // expose IConfiguration
                        services.AddSingleton(configuration);

                        // expose CyberArk runtime values to the app
                        services.AddSingleton(runtimeContext);

                        // register your existing services
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

                // -------------------------------
                // Application entry execution
                // -------------------------------

                using var scope = host.Services.CreateScope();

                var jobProcessor =
                    scope.ServiceProvider.GetRequiredService<CompareToolJobProcessorService>();

                await jobProcessor.ExecuteAsync();

                return 0;
            }
            catch (Exception ex)
            {
                Log.Fatal(ex, "CompareTool Console failed.");
                return 1;
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }
    }
}
