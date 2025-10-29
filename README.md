using BTA_CompareTool_Console.Data;
using BTA_CompareTool_Console.Services;
using CompareTool.Models;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;
using System;
using System.Collections.Generic;
using System.IO;
using System.Threading.Tasks;

namespace CompareTool.ConsoleApp
{
    public static class Program
    {
        public static async Task Main(string[] args)
        {
            Directory.SetCurrentDirectory(AppContext.BaseDirectory);

            Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Information()
                .WriteTo.Console()
                .WriteTo.File(
                    Path.Combine(AppContext.BaseDirectory, "Logs", "console-boot.log"),
                    rollingInterval: RollingInterval.Day,
                    retainedFileCountLimit: 14)
                .CreateLogger();

            try
            {
                Log.Information("=== CompareTool Console Bootstrap ===");

                // Load base JSON config
                var localConfig = new ConfigurationBuilder()
                    .SetBasePath(AppContext.BaseDirectory)
                    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                    .AddEnvironmentVariables()
                    .Build();

                string connectionString = localConfig.GetConnectionString("BTEDBConnectionString");

                if (string.IsNullOrWhiteSpace(connectionString))
                    throw new Exception("Connection string missing.");

                Log.Information("DB Connection string found ✅");

                // Host with service registration
                var host = Host.CreateDefaultBuilder(args)
                    .UseSerilog()
                    .ConfigureServices(services =>
                    {
                        services.AddDbContext<DBServerContext>(opt =>
                            opt.UseSqlServer(connectionString),
                            contextLifetime: ServiceLifetime.Scoped);

                        services.AddScoped<Job>();
                        services.AddScoped<XmlConsumer>();
                        services.AddScoped<CompareToolJobProcessorService>();
                        services.AddScoped<JobDetailsService>();
                    })
                    .Build();

                // Execute logic
                using var scope = host.Services.CreateScope();
                var consumer = scope.ServiceProvider.GetRequiredService<XmlConsumer>();

                string xml = "(14993836, <message><Request_Number>14993836</Request_Number><Sender>BTE System</Sender></message>)";
                string tableName = "BTE_CompareTool_Main_Request";

                Log.Information("Executing XML Consumer...");
                await consumer.ProcessXmlAsync(xml, tableName);

                Log.Information("✅ CompareTool Console Completed Successfully ✅");
            }
            catch (Exception ex)
            {
                Log.Fatal(ex, "❌ Console crashed with unhandled exception.");
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }
    }
}
