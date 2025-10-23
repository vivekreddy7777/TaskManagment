using System.Diagnostics;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace BTA_CompareTool_Service;

public sealed class CompareToolOptions
{
    public bool   EnableDetailedLogging { get; set; } = true;
    public int    MaxRecordsToProcessPerInstance { get; set; } = 100;
    public int    MaxConcurrentInstances { get; set; } = 0; // optional cap
    public string? CompareToolConsolePath { get; set; }
}

public sealed class CompareToolDispatcher : IAsyncDisposable
{
    private readonly ILogger<CompareToolDispatcher> _logger;
    private readonly CompareToolOptions _opts;
    private SemaphoreSlim _semaphore;

    // self-tuning constants (adjust if needed)
    private const double UsableMemoryFraction = 0.60;
    private const double PayloadSafetyFactor  = 1.5;
    private const double DefaultAvgPayloadMB  = 0.25;
    private const double BaseOverheadMB       = 128;
    private const int    EstPerInstanceMinMB  = 256;
    private const int    EstPerInstanceMaxMB  = 2048;
    private const int    AbsoluteHardCap      = 32;

    public CompareToolDispatcher(ILogger<CompareToolDispatcher> logger, IOptions<CompareToolOptions> opts)
    {
        _logger = logger;
        _opts   = opts.Value;
        _semaphore = new SemaphoreSlim(Math.Max(1, Environment.ProcessorCount)); // rebuilt per dispatch
        _logger.LogInformation("Options => Logging={log}, ChunkSize={chunk}, Cap={cap}, Path={path}",
            _opts.EnableDetailedLogging, _opts.MaxRecordsToProcessPerInstance, _opts.MaxConcurrentInstances, _opts.CompareToolConsolePath);
    }

    private int CalculateAutoMaxConcurrency(int chunkSize, IReadOnlyList<string> xmlPayloads)
    {
        chunkSize = Math.Max(1, chunkSize);
        int cpuCap = Math.Max(1, Environment.ProcessorCount - 1);

        var info = GC.GetGCMemoryInfo();
        double availableMB = Math.Max(0, (info.TotalAvailableMemoryBytes - info.MemoryLoadBytes) / (1024.0 * 1024.0));
        double usableMB = availableMB * UsableMemoryFraction;

        double avgPayloadMB =
            (xmlPayloads == null || xmlPayloads.Count == 0)
                ? DefaultAvgPayloadMB
                : xmlPayloads.Average(x => (x?.Length ?? 0) * 2 / (1024.0 * 1024.0));

        double estPerInstanceMB = BaseOverheadMB + (avgPayloadMB * chunkSize * PayloadSafetyFactor);
        estPerInstanceMB = Math.Clamp(estPerInstanceMB, EstPerInstanceMinMB, EstPerInstanceMaxMB);

        int byMemory = (int)Math.Max(1, Math.Floor(usableMB / estPerInstanceMB));
        int byCpu    = cpuCap;
        int cfgCap   = _opts.MaxConcurrentInstances > 0 ? _opts.MaxConcurrentInstances : int.MaxValue;

        int chosen = Math.Max(1, Math.Min(byMemory, Math.Min(byCpu, Math.Min(cfgCap, AbsoluteHardCap))));
        _logger.LogInformation(
            "AutoConcurrency => availMB={avail:F0}, usableMB={usable:F0}, estPerInstMB={per:F0}, byMem={byMem}, byCPU={byCpu}, cfgCap={cfgCap}, chosen={chosen}",
            availableMB, usableMB, estPerInstanceMB, byMemory, byCpu, (cfgCap == int.MaxValue ? 0 : cfgCap), chosen);

        return chosen;
    }

    public async Task DispatchAsync(List<string> xmlPayloads, CancellationToken ct = default)
    {
        int total = xmlPayloads?.Count ?? 0;
        if (total == 0)
        {
            _logger.LogInformation("No payloads to dispatch.");
            return;
        }

        int chunkSize = Math.Max(1, _opts.MaxRecordsToProcessPerInstance);
        int autoMax   = CalculateAutoMaxConcurrency(chunkSize, xmlPayloads);

        _semaphore.Dispose();
        _semaphore = new SemaphoreSlim(autoMax);

        int needed = (int)Math.Ceiling(total / (double)chunkSize);
        int instances = Math.Max(1, Math.Min(needed, autoMax));
        _logger.LogInformation("Dispatch plan => total={total}, chunkSize={chunk}, instances={inst}", total, chunkSize, instances);

        for (int i = 0; i < instances; i++)
        {
            if (ct.IsCancellationRequested) break;

            var payloadChunk = xmlPayloads.Skip(i * chunkSize).Take(chunkSize).ToList();
            string combinedXml = string.Join(";", payloadChunk);

            await _semaphore.WaitAsync(ct);
            _ = Task.Run(async () =>
            {
                try
                {
                    string exePath = _opts.CompareToolConsolePath ?? string.Empty;
                    if (string.IsNullOrWhiteSpace(exePath))
                        throw new InvalidOperationException("CompareToolConsolePath is not configured.");

                    string instanceName = $"CompareTool-{i + 1}-{DateTime.Now:yyyyMMddHHmmss}";
                    var psi = new ProcessStartInfo
                    {
                        FileName = exePath,
                        Arguments = $"--xml \"{combinedXml}\" --instance \"{instanceName}\"",
                        RedirectStandardOutput = true,
                        RedirectStandardError  = true,
                        UseShellExecute = false,
                        CreateNoWindow = true
                    };

                    if (_opts.EnableDetailedLogging)
                        _logger.LogInformation("Starting {name} with args: {args}", instanceName, psi.Arguments);

                    using var process = Process.Start(psi) ?? throw new InvalidOperationException("Failed to start CompareTool process.");
                    string stdout = await process.StandardOutput.ReadToEndAsync();
                    string stderr = await process.StandardError.ReadToEndAsync();
                    await process.WaitForExitAsync(ct); // .NET 6+ API; fine on .NET 8

                    if (_opts.EnableDetailedLogging)
                    {
                        if (!string.IsNullOrWhiteSpace(stdout)) _logger.LogInformation("OUT[{name}]:\n{out}", instanceName, stdout);
                        if (!string.IsNullOrWhiteSpace(stderr)) _logger.LogWarning("ERR[{name}]:\n{err}", instanceName, stderr);
                    }
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error dispatching instance {idx}", i + 1);
                }
                finally
                {
                    _semaphore.Release();
                }
            }, ct);
        }
    }

    public ValueTask DisposeAsync()
    {
        _semaphore.Dispose();
        return ValueTask.CompletedTask;
    }
}
