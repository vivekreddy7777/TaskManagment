using System.Data;
using System.Diagnostics;
using System.Text;
using System.Text.Json;
using System.Threading.Channels;
using System.Xml;
using System.Xml.XPath;
using Microsoft.Data.SqlClient;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using Polly;
using Polly.Contrib.WaitAndRetry;
using Polly.Retry;

namespace BTA_CompareTool_Service;

// -------------------- Options --------------------
public sealed class DatabaseOptions
{
    public string ConnectionString  { get; set; } = string.Empty;
    public string Module            { get; set; } = "CompareTool";
    public string UserId            { get; set; } = string.Empty;
    public string Password          { get; set; } = string.Empty;
    public int    CommandTimeoutSec { get; set; } = 60;
    public int    ReceiveTimeoutSec { get; set; } = 10;
}

public sealed class ServiceOptions
{
    public int MaxThreads          { get; set; } = 4;    // SQL listeners
    public int ChannelCapacity     { get; set; } = 200;  // in-memory buffer
    public int ConsoleWriters      { get; set; } = 1;    // writer tasks (stdout) when ConsoleExePath is empty

    // Retry
    public int MaxTransientRetries { get; set; } = 8;
    public int FirstBackoffMs      { get; set; } = 200;
    public int MaxBackoffMs        { get; set; } = 8000;
}

public sealed class ConsoleOptions
{
    public string ConsoleExePath { get; set; } = string.Empty; // if empty => write to this process stdout
    public string ConsoleArgs    { get; set; } = string.Empty;
    public int    ProcessCount   { get; set; } = 0;            // 0 -> auto = Environment.ProcessorCount/2
}

// -------------------- Models --------------------
public sealed record ReceivedMessage(long ReqId, string Xml);

// -------------------- Hosted Service --------------------
public sealed class CompareToolHostedService : BackgroundService
{
    private readonly ILogger<CompareToolHostedService> _log;
    private readonly DatabaseOptions _db;
    private readonly ServiceOptions _svc;
    private readonly ConsoleOptions _console;

    private string _module = string.Empty;
    private string _queue  = string.Empty;
    private int    _threads = 1;

    private readonly Channel<ReceivedMessage> _outbox;
    private AsyncRetryPolicy _sqlRetryPolicy = null!;

    // console process pool (external workers)
    private ConsoleProcess[] _procPool = Array.Empty<ConsoleProcess>();

    public CompareToolHostedService(
        ILogger<CompareToolHostedService> log,
        IOptions<DatabaseOptions> db,
        IOptions<ServiceOptions> svc,
        IOptions<ConsoleOptions> console)
    {
        _log = log;
        _db = db.Value;
        _svc = svc.Value;
        _console = console.Value;

        _outbox = Channel.CreateBounded<ReceivedMessage>(new BoundedChannelOptions(Math.Max(50, _svc.ChannelCapacity))
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = false
        });
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        BuildSqlRetryPolicy();
        await _sqlRetryPolicy.ExecuteAsync(ct => LoadModuleAsync(ct), stoppingToken);

        // Start console pool if configured
        var useExternalConsoles = !string.IsNullOrWhiteSpace(_console.ConsoleExePath);
        if (useExternalConsoles)
            await StartConsolePoolAsync(stoppingToken);

        // Writers: either multiple stdout writers OR a single dispatcher to the console pool
        Task[] writerTasks;
        if (useExternalConsoles)
        {
            writerTasks = new[] { Task.Run(() => ConsolePoolDispatcherAsync(stoppingToken), stoppingToken) };
        }
        else
        {
            var writerCount = Math.Max(1, _svc.ConsoleWriters);
            writerTasks = Enumerable.Range(0, writerCount)
                .Select(i => Task.Run(() => StdoutWriterAsync(i + 1, stoppingToken), stoppingToken))
                .ToArray();
        }

        // SQL listeners
        var listenerCount = Math.Clamp(_threads, 1, Math.Max(1, _svc.MaxThreads));
        var listenerTasks = Enumerable.Range(0, listenerCount)
            .Select(i => ListenerAsync(i + 1, stoppingToken))
            .ToArray();

        await Task.WhenAll(Task.WhenAll(listenerTasks), Task.WhenAll(writerTasks));

        // teardown pool
        if (_procPool.Length > 0)
            await StopConsolePoolAsync();
    }

    // -------------------- Listener --------------------
    private async Task ListenerAsync(int index, CancellationToken ct)
    {
        await using var conn = new SqlConnection(_db.ConnectionString);
        await conn.OpenAsync(ct);

        await using var cmd = new SqlCommand("procBTE_ServiceBroker_Receive", conn)
        {
            CommandType = CommandType.StoredProcedure,
            CommandTimeout = _db.CommandTimeoutSec
        };
        cmd.Parameters.AddWithValue("@Password", _db.Password);
        cmd.Parameters.AddWithValue("@UserID",   _db.UserId);
        cmd.Parameters.AddWithValue("@Queue",    _queue);
        cmd.Parameters.AddWithValue("@Timeout",  _db.ReceiveTimeoutSec);

        while (!ct.IsCancellationRequested)
        {
            try
            {
                var (reqId, xml) = await _sqlRetryPolicy.ExecuteAsync(c => ReceiveAsync(cmd, c), ct);
                if (reqId is null || string.IsNullOrWhiteSpace(xml))
                {
                    await Task.Delay(100, ct);
                    continue;
                }

                await _outbox.Writer.WriteAsync(new ReceivedMessage(reqId.Value, xml), ct);
            }
            catch (OperationCanceledException) { break; }
            catch (Exception ex)
            {
                _log.LogError(ex, "Listener {Index} encountered a non-retriable error.", index);
                await Task.Delay(500, ct);
            }
        }
    }

    // -------------------- Writer: STDOUT (in-proc scaling) --------------------
    private async Task StdoutWriterAsync(int writerIndex, CancellationToken ct)
    {
        using var stdout = Console.OpenStandardOutput();
        using var sw = new StreamWriter(stdout, new UTF8Encoding(false), bufferSize: 64 * 1024) { AutoFlush = true };

        await foreach (var msg in _outbox.Reader.ReadAllAsync(ct))
        {
            try
            {
                var (v1, v2) = TryParseValues(msg.Xml);

                var payload = new
                {
                    reqId  = msg.ReqId,
                    module = _module,
                    queue  = _queue,
                    value1 = v1,
                    value2 = v2,
                    writer = writerIndex,
                    xml    = msg.Xml
                };

                await sw.WriteLineAsync(JsonSerializer.Serialize(payload));
            }
            catch (Exception ex)
            {
                _log.LogError(ex, "STDOUT writer {Index} failed for ReqId={ReqId}", writerIndex, msg.ReqId);
            }
        }
    }

    // -------------------- Writer: External Console Pool (multi-process scaling) --------------------
    private async Task ConsolePoolDispatcherAsync(CancellationToken ct)
    {
        // partition by ReqId for stable ordering per partition
        int Partition(long reqId) => (int)(reqId % _procPool.Length);

        await foreach (var msg in _outbox.Reader.ReadAllAsync(ct))
        {
            try
            {
                var (v1, v2) = TryParseValues(msg.Xml);

                var payload = new
                {
                    reqId  = msg.ReqId,
                    module = _module,
                    queue  = _queue,
                    value1 = v1,
                    value2 = v2,
                    xml    = msg.Xml
                };

                var line = JsonSerializer.Serialize(payload);

                var idx = Partition(msg.ReqId);
                var proc = _procPool[idx];
                await proc.SendAsync(line, ct);
            }
            catch (Exception ex)
            {
                _log.LogError(ex, "Dispatcher failed for ReqId={ReqId}", msg.ReqId);
            }
        }
    }

    // -------------------- SQL Helpers --------------------
    private async Task<(long? ReqId, string? Xml)> ReceiveAsync(SqlCommand cmd, CancellationToken ct)
    {
        await using var rd = await cmd.ExecuteReaderAsync(ct);
        if (!await rd.ReadAsync(ct)) return (null, null);

        var body = rd["MessageBody"] as string ?? rd["message_body"] as string;
        if (string.IsNullOrWhiteSpace(body)) return (null, null);

        var id = TryParseRequestNumber(body);
        return (id, body);
    }

    private async Task LoadModuleAsync(CancellationToken ct)
    {
        await using var conn = new SqlConnection(_db.ConnectionString);
        await conn.OpenAsync(ct);

        await using var cmd = new SqlCommand("procBTE_System_Module_SELECT", conn)
        {
            CommandType = CommandType.StoredProcedure,
            CommandTimeout = _db.CommandTimeoutSec
        };
        cmd.Parameters.AddWithValue("@Password", _db.Password);
        cmd.Parameters.AddWithValue("@UserID",   _db.UserId);
        cmd.Parameters.AddWithValue("@Module",   _db.Module);

        await using var r = await cmd.ExecuteReaderAsync(ct);
        if (!await r.ReadAsync(ct))
            throw new InvalidOperationException($"Module not found: {_db.Module}");

        _module  = _db.Module;
        _queue   = r["QueueName"]?.ToString() ?? throw new InvalidOperationException("QueueName missing.");
        _threads = Math.Max(1, ParseInt(r["ThreadCount"]));
    }

    // -------------------- XML Helpers --------------------
    private static long? TryParseRequestNumber(string xml)
    {
        try
        {
            using var sr = new StringReader(xml);
            var xdoc = new XPathDocument(sr);
            var nav  = xdoc.CreateNavigator();
            var node = nav.SelectSingleNode("/message/Request_Number");
            return node != null && long.TryParse(node.Value, out var id) ? id : null;
        }
        catch (XmlException) { return null; }
    }

    private static (long?, long?) TryParseValues(string xml)
    {
        try
        {
            using var sr = new StringReader(xml);
            var xdoc = new XPathDocument(sr);
            var nav = xdoc.CreateNavigator();

            var n1 = nav.SelectSingleNode("/message/Value1");
            var n2 = nav.SelectSingleNode("/message/Value2");

            long? v1 = long.TryParse(n1?.Value, out var a) ? a : null;
            long? v2 = long.TryParse(n2?.Value, out var b) ? b : null;
            return (v1, v2);
        }
        catch { return (null, null); }
    }

    // -------------------- Retry Policy --------------------
    private void BuildSqlRetryPolicy()
    {
        var delays = Backoff.DecorrelatedJitterBackoffV2(
            medianFirstRetryDelay: TimeSpan.FromMilliseconds(Math.Max(50, _svc.FirstBackoffMs)),
            retryCount: Math.Max(1, _svc.MaxTransientRetries),
            fastFirst: true);

        if (_svc.MaxBackoffMs > 0)
            delays = delays.Select(d => d > TimeSpan.FromMilliseconds(_svc.MaxBackoffMs)
                ? TimeSpan.FromMilliseconds(_svc.MaxBackoffMs)
                : d);

        _sqlRetryPolicy = Policy
            .Handle<SqlException>(IsTransient)
            .WaitAndRetryAsync(
                sleepDurations: delays,
                onRetryAsync: async (ex, delay, attempt, ctx) =>
                {
                    _log.LogWarning(ex, "Transient SQL (attempt {Attempt}). Retrying in {Delay} ms.",
                        attempt, (int)delay.TotalMilliseconds);
                    await Task.CompletedTask;
                });
    }

    private static bool IsTransient(SqlException ex) =>
        ex.Number is -2 or 4060 or 40197 or 40501 or 40613 or 10928 or 10929 or 1205;

    private static int ParseInt(object? v) => int.TryParse(v?.ToString(), out var n) ? n : 1;

    // -------------------- External Console Process Pool --------------------
    private async Task StartConsolePoolAsync(CancellationToken ct)
    {
        var count = _console.ProcessCount > 0 ? _console.ProcessCount : Math.Max(1, Environment.ProcessorCount / 2);
        _procPool = new ConsoleProcess[count];

        for (int i = 0; i < count; i++)
        {
            var p = new ConsoleProcess(_log, _console.ConsoleExePath, _console.ConsoleArgs, $"p{i+1}");
            await p.StartAsync(ct);
            _procPool[i] = p;
        }

        _log.LogInformation("Console process pool started: {Count} process(es).", count);
    }

    private async Task StopConsolePoolAsync()
    {
        foreach (var p in _procPool)
            await p.DisposeAsync();

        _procPool = Array.Empty<ConsoleProcess>();
        _log.LogInformation("Console process pool stopped.");
    }
}

// -------------------- External ConsoleProcess --------------------
internal sealed class ConsoleProcess : IAsyncDisposable
{
    private readonly ILogger _log;
    private readonly string _exePath;
    private readonly string _args;
    private readonly string _name;

    private Process? _proc;
    private StreamWriter? _stdin;
    private Task? _stdoutPump;
    private Task? _stderrPump;

    public ConsoleProcess(ILogger log, string exePath, string args, string name)
    {
        _log = log;
        _exePath = exePath;
        _args = args ?? string.Empty;
        _name = name;
    }

    public async Task StartAsync(CancellationToken ct)
    {
        if (!File.Exists(_exePath))
            throw new FileNotFoundException("Console exe not found", _exePath);

        var psi = new ProcessStartInfo
        {
            FileName = _exePath,
            Arguments = _args,
            UseShellExecute = false,
            RedirectStandardInput = true,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            CreateNoWindow = true,
            StandardOutputEncoding = Encoding.UTF8,
            StandardErrorEncoding = Encoding.UTF8
        };

        _proc = new Process { StartInfo = psi, EnableRaisingEvents = true };
        _proc.Start();

        _stdin = new StreamWriter(_proc.StandardInput.BaseStream, new UTF8Encoding(false)) { AutoFlush = true };

        _stdoutPump = Task.Run(async () =>
        {
            string? line;
            while ((_proc is { HasExited: false }) && (line = await _proc.StandardOutput.ReadLineAsync()) is not null)
                _log.LogInformation("[{Name}-out] {Line}", _name, line);
        }, ct);

        _stderrPump = Task.Run(async () =>
        {
            string? line;
            while ((_proc is { HasExited: false }) && (line = await _proc.StandardError.ReadLineAsync()) is not null)
                _log.LogWarning("[{Name}-err] {Line}", _name, line);
        }, ct);
    }

    public Task SendAsync(string jsonLine, CancellationToken ct)
    {
        if (_stdin is null) throw new InvalidOperationException("Console process not started.");
        return _stdin.WriteLineAsync(jsonLine).WaitAsync(ct);
    }

    public async ValueTask DisposeAsync()
    {
        try
        {
            _stdin?.Dispose();
            if (_proc is { HasExited: false })
            {
                try { _proc.Kill(entireProcessTree: true); } catch { /* ignore */ }
                _proc.WaitForExit(3000);
            }
            _proc?.Dispose();
            if (_stdoutPump is not null) await _stdoutPump;
            if (_stderrPump is not null) await _stderrPump;
        }
        catch { /* ignore */ }
    }
}
