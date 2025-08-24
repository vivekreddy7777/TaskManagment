public sealed class CompareToolHostedService : BackgroundService
{
    private readonly IServiceProvider _sp;
    private readonly ILogger<CompareToolHostedService> _log;
    private readonly IConfiguration _cfg;

    // cached module metadata
    private string _module = "CompareTool";
    private string _table  = "";
    private string _queue  = "";
    private int    _timeoutMs = 1000;
    private int    _threads   = 1;

    public CompareToolHostedService(IServiceProvider sp,
                                    ILogger<CompareToolHostedService> log,
                                    IConfiguration cfg)
    {
        _sp = sp; _log = log; _cfg = cfg;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // 1) load module details once
        var connStr = _cfg.GetConnectionString("QCoreDb")
                     ?? throw new InvalidOperationException("Missing ConnectionStrings:QCoreDb");
        await LoadModuleAsync(connStr, stoppingToken);

        // 2) spin up listeners
        var tasks = Enumerable.Range(0, _threads)
            .Select(i => RunListenerAsync(i + 1, connStr, stoppingToken))
            .ToArray();

        await Task.WhenAll(tasks);
    }

    private async Task LoadModuleAsync(string connStr, CancellationToken ct)
    {
        await using var conn = new SqlConnection(connStr);
        await conn.OpenAsync(ct);

        await using var cmd = new SqlCommand("procBTE_System_Module_SELECT", conn)
        { CommandType = CommandType.StoredProcedure };

        cmd.Parameters.AddWithValue("@Password", Constants.Password);
        cmd.Parameters.AddWithValue("@UserID",   Constants.UserID);
        cmd.Parameters.AddWithValue("@Module",   _module);
        cmd.Parameters.AddWithValue("@ProcResult", DBNull.Value);

        await using var r = await cmd.ExecuteReaderAsync(ct);
        if (!await r.ReadAsync(ct))
            throw new InvalidOperationException($"Module not found: '{_module}'.");

        _table     = r["MainRequestTable"]?.ToString() ?? throw new InvalidOperationException("MainRequestTable missing.");
        _queue     = r["QueueName"]?.ToString()        ?? "<unknown>";
        _timeoutMs = r["QueueTimeout"] as int? ?? 1000;

        // if you respect ThreadCount like 4.5 (but clamp locally to 1 if needed)
        _threads   = r["ThreadCount"] as int? ?? 1;
        if (Constants.RunningLocal) _threads = 1;

        _log.LogInformation("Module={Module} Table={Table} Queue={Queue} Threads={Threads} TimeoutMs={Timeout}",
            _module, _table, _queue, _threads, _timeoutMs);
    }

    private async Task RunListenerAsync(int index, string connStr, CancellationToken ct)
    {
        _log.LogInformation("Listener #{Idx} started on {Queue}", index, _queue);

        while (!ct.IsCancellationRequested)
        {
            try
            {
                var msg = await ReceiveAsync(connStr, ct);
                if (msg is null)
                {
                    await Task.Delay(250, ct);
                    continue;
                }

                // Build base job
                Job baseJob = msg.Value.RequestNumber.HasValue
                    ? new Job(msg.Value.RequestNumber.Value, _module, _table, _sp)
                    : new Job(msg.Value.XmlBody ?? string.Empty, _module, _table, _sp);

                // Create and run derived job
                var compareJob = ActivatorUtilities.CreateInstance<BTA_CompareTool_Job>(_sp, baseJob);
                await compareJob.RunAsync(ct);
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                _log.LogError(ex, "Listener #{Idx} error", index);
                await Task.Delay(1000, ct); // brief backoff
            }
        }

        _log.LogInformation("Listener #{Idx} stopping", index);
    }

    // one receive attempt; returns null on timeout/empty
    private async Task<(long? RequestNumber, string? XmlBody)?> ReceiveAsync(string connStr, CancellationToken ct)
    {
        await using var conn = new SqlConnection(connStr);
        await conn.OpenAsync(ct);

        await using var cmd = new SqlCommand("procBTE_ServiceBroker_Receive", conn)
        { CommandType = CommandType.StoredProcedure };

        cmd.Parameters.AddWithValue("@Password",     Constants.Password);
        cmd.Parameters.AddWithValue("@UserID",       Constants.UserID);
        cmd.Parameters.AddWithValue("@Module",       _module);
        cmd.Parameters.AddWithValue("@QueueTimeout", _timeoutMs);
        cmd.Parameters.AddWithValue("@ProcResult",   DBNull.Value);

        await using var r = await cmd.ExecuteReaderAsync(ct);
        if (!await r.ReadAsync(ct))
            return null;

        long? req = r["Request_Number"] as long?;
        string? xml = r["xml_message_body"]?.ToString() ?? r["MessageBody"]?.ToString(); // handle either column name

        return (req, xml);
    }
}
